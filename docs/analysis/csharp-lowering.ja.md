# Roslyn C# Lowering 詳細解説
## ― 高水準な BoundTree を IL 生成可能な形へ：LocalRewriter・クロージャ変換・イテレータ/async ステートマシン ―

対象: `src/Compilers/CSharp/Portable/Lowering/`（`LocalRewriter/` 約 70 ファイル、`ClosureConversion/`、`StateMachineRewriter/`、`IteratorRewriter/`、`AsyncRewriter/`、`Instrumentation/` ほか）
基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは特記なき限り `src/Compilers/CSharp/Portable/` からの相対パス。

関連ドキュメント（同ディレクトリ）:
- `csharp-binder.ja.md`：入力となる BoundTree を作るバインダ
- `csharp-boundtree-boundnodes.ja.md`：BoundTree、`DoesNotSurvive`、`PipelinePhaseValidator`
- `csharp-compiler-dataflow-analysis.ja.md`：Lowering の前に走るフロー解析
- `csharp-codegen-emit.ja.md`：Lowering の出力を IL にする段階

---

## 0. 要約

- **Lowering** は、バインダが作った「C# の意味をそのまま表す BoundTree」を、**IL に 1 対 1 に近い単純な BoundTree** へ段階的に書き換える。入出力の型はどちらも `BoundStatement`。
- 段階（`MethodCompiler.LowerBodyOrInitializer`）は次のとおり。

  ```text
  LocalRewriter（＋必要なら SpillSequenceSpiller）
    → 拡張メンバー参照の書き換え
    → AsyncExceptionHandlerRewriter（catch/finally 内の await がある場合）
    → ClosureConversion（ラムダ/ローカル関数がある場合）
    → IteratorRewriter
    → AsyncRewriter または RuntimeAsyncRewriter
  ```

- **LocalRewriter** の処理内容：
  - 制御構文をラベルと `BoundConditionalGoto` に分解する（`if`/`while`/`for`/`foreach`/`switch`/パターン/`using`/`lock`）。
  - 糖衣構文を展開する（文字列連結、補間文字列、`?.`、`??`、複合代入、持ち上げ演算子、タプル、分解、初期化子、コレクション式、`dynamic`、Index/Range）。
  - 定数を畳み込む。
  - シーケンスポイント（デバッグ情報）を挿入する。
- **`await` のように「文を含む式」は `BoundSpillSequence` で表し、`SpillSequenceSpiller` が文のレベルへ持ち上げる**。評価順序を保つため、先に評価される部分式は一時変数へ退避（spill）する。
- **ClosureConversion** は、捕捉される変数をスコープごとの「環境」（display class `<>c__DisplayClassN`、または ref で渡す struct）に移し、ラムダ/ローカル関数を通常のメソッドにする。捕捉の無いラムダは共有の `<>c` シングルトンのインスタンスメソッドにし、デリゲートをキャッシュする。
- **ステートマシン**：
  - イテレータ（`yield`）と async（`await`）のメソッド本体を、合成した型（`<M>d__N`）の `MoveNext` に移す。
  - 状態番号の `switch` で再開位置へ飛ぶ。
  - 中断をまたいで生きている変数をフィールドへ持ち上げる（hoist）。
  - **新しい runtime-async**（`MethodImplAttributes.Async`）では、ステートマシンを作らずにランタイムへ任せる。
- Lowering はほとんどエラーを報告しない（バインダとフロー解析が報告済み）。主な例外は**必要な既知メンバーが見つからない場合**（`ERR_MissingPredefinedMember`）。エラーを含む木は Lowering しない。

---

## 1. パイプラインと共通基盤

### 1.1 段階の順序（`Compiler/MethodCompiler.cs:1481-1631`）

```text
if (body.HasErrors) return body;                                         :1502  エラーのある木は Lowering しない
PipelinePhaseValidator.AssertAfterInitialBinding(body)                   :1508
LocalRewriter.Rewrite(...)  ── 内部で SpillSequenceSpiller（_needsSpilling 時）  :1512  → out sawLambdas / sawLocalFunctions / sawAwaitInExceptionHandler
ExtensionMethodBodyRewriter / ExtensionMethodReferenceRewriter            :1534-1543  C# 14 の拡張メンバーを実装メソッドの呼び出しへ
AsyncExceptionHandlerRewriter.Rewrite  (sawAwaitInExceptionHandler)       :1550-1563
VariableSlotAllocator の準備（Edit and Continue 用）                         :1570
ClosureConversion.Rewrite  (sawLambdas || sawLocalFunctions)             :1573-1589
PipelinePhaseValidator.AssertAfterClosureConversion                      :1596
IteratorRewriter.Rewrite                                                 :1598
RuntimeAsyncRewriter.Rewrite  or  AsyncRewriter.Rewrite                  :1608-1616
PipelinePhaseValidator.AssertAfterStateMachineRewriting                  :1621
（各段階の後、結果が HasErrors なら打ち切り。スタック溢れは BoundBadStatement :1625-1630）
```

**ラムダとローカル関数の本体は 2 周目で処理する**：ClosureConversion が合成したメソッドは `compilationState.SynthesizedMethods` に積まれる。`CompileSynthesizedMethods`（`MethodCompiler.cs:758-798`）が後で同じ Iterator → Async の段階を適用する。async ラムダやイテレータのローカル関数は、クロージャのメソッドへ移された**後で**ステートマシン化される。

Lowering の**前**には、次の 3 つが走る。

- `InitializerRewriter`（フィールド初期化子をコンストラクタの文にする, `:1052`）
- `DiagnosticsPass`（警告と式ツリーの制限, `:1145`）
- `FlowAnalysisPass`（確定代入と暗黙の return の追加, `:1151`）

エラー診断があるメソッドでも、木に `HasErrors` が無ければ Lowering はするが、**IL は出力しない**（`:1254-1256`）。

### 1.2 段階の検証（`Lowering/LocalRewriter/PipelinePhaseValidator.cs`）

`PipelinePhase { InitialBinding, LocalRewriting, Spilling, ClosureConversion, StateMachineRewriting, Emit }`（`:11-19`）。各段階の後、DEBUG ビルドで木全体を歩き、`BoundNodes.xml` の `DoesNotSurvive` で「この段階を過ぎたら存在してはならない」と宣言されたノードが残っていないかを検査する（BoundTree ドキュメント §6）。例：

- `BoundIfStatement` は LocalRewriting で消える。
- `BoundSpillSequence` は Spilling で消える。
- `BoundLocalFunctionStatement` は ClosureConversion で消える。
- `BoundAwaitExpression`・`BoundYieldReturnStatement` は StateMachineRewriting で消える。

### 1.3 `SyntheticBoundNodeFactory`（`Lowering/SyntheticBoundNodeFactory.cs`、1,917 行）

Lowering で使う合成ノード用のファクトリ（`F`）。作った全てのノードに、現在の `Syntax`（位置の帰属先）と `WasCompilerGenerated = true` を付ける。

| 分類 | メソッド |
|---|---|
| 参照 | `This()`、`Parameter`、`Field`、`Property`、`Local`、`Literal`、`Null` |
| 代入・文 | `Assignment`（文）、`AssignmentExpression`、`ExpressionStatement`、`Block`、`StatementList`、`Return`、`Throw` |
| 演算 | `Binary`、`IntEqual`、`ObjectEqual`、`IsNotNullReference`、`Conditional`、`Convert`（IL レベルの変換） |
| 生成・呼び出し | `New`、`StaticCall`、`Call`、`Array` |
| 制御 | `If`（`ConditionalGoto` とラベルに展開）、`ConditionalGoto`、`Goto`、`Label`、`GenerateLabel`、`HiddenSequencePoint` |
| 式中の文 | `Sequence`、`MakeSequence`（副作用の無い部分を省く）、`SpillSequence` |
| 一時変数 | `SynthesizedLocal(type, kind…)`、**`StoreToTemp(expr, out store, refKind, kind)`**（`:1670`） |

`CurrentFunction`（`:70-84`）は、いま Lowering しているメソッド・ラムダ・ローカル関数を表す。

**既知メンバーが見つからない場合**：`WellKnownMember(wm)` / `SpecialMember(sm)`（`:328-382`）は、例えば `Monitor.Enter` や `String.Concat` が参照アセンブリに無いと、`ERR_MissingPredefinedMember` を持つ `MissingPredefinedMember` 例外を投げる。`LocalRewriter.Rewrite` がこれを捕捉して診断を報告し、`BoundBadStatement` を返す（`LocalRewriter.cs:173-179`）。LocalRewriter 自身は主に、例外を投げない `TryGetWellKnownTypeMember`（`:606`）を使う。

**一時変数の種類**（`SynthesizedLocalKind`）：`LoweringTemp`、`Spill`、`Lock`、`Using`、`Awaiter`、`LambdaDisplayClass`、`StateMachineCachedState` など。Edit and Continue（EnC）で前の世代と変数のスロットを対応付けるために使い、Spiller も退避の要否の判断に使う。

---

## 2. `LocalRewriter`

### 2.1 構造（`Lowering/LocalRewriter/LocalRewriter.cs`、約 70 の partial ファイル）

```csharp
internal sealed partial class LocalRewriter : BoundTreeRewriterWithStackGuard          // :23
{
    private readonly CSharpCompilation _compilation;
    private readonly SyntheticBoundNodeFactory _factory;
    private readonly LoweredDynamicOperationFactory _dynamicFactory;   // dynamic の CallSite
    private bool _sawLambdas; private int _availableLocalFunctionOrdinal;
    private bool _inExpressionLambda;                                  // Expression<T> に変換されるラムダの中
    private bool _sawAwait, _sawAwaitInExceptionHandler, _needsSpilling;
    private Dictionary<BoundValuePlaceholderBase, BoundExpression>? _placeholderReplacementMapDoNotUseDirectly;   // :54
}
public static BoundStatement Rewrite(CSharpCompilation, MethodSymbol method, int methodOrdinal, NamedTypeSymbol containingType,
    BoundStatement statement, TypeCompilationState, SynthesizedSubmissionFields, bool allowOmissionOfConditionalCalls,
    MethodInstrumentation, DebugDocumentProvider, BindingDiagnosticBag,
    out ImmutableArray<SourceSpan> codeCoverageSpans, out bool sawLambdas, out bool sawLocalFunctions, out bool sawAwaitInExceptionHandler)   // :85-100
```

`Rewrite` の流れ：

1. **計装（instrumentation）チェーンを組む**（`:112-141`）。`NoOp` を起点に、有効なものを順に重ねる。
   - `LocalStateTracingInstrumenter`
   - `CodeCoverageInstrumenter`
   - `StackOverflowProbingInstrumenter`
   - `ModuleCancellationInstrumenter`

   最後に**必ず `DebugInfoInjector` で包む**。コメント（`:143-144`）によれば、PDB を出力するかどうかで IL が変わらないよう、シーケンスポイントは常に生成する。
2. `VisitStatement(statement)` で本体を Lowering する（`:147`）。
3. 必要なら **`SpillSequenceSpiller.Rewrite`**（`:160-166`）。Spiller は独立した段階ではなく、`LocalRewriter.Rewrite` の内部で走る。

### 2.2 訪問の規則

- `Visit`・`VisitExpression`・`VisitStatement` は `!node.HasErrors` を assert する（`:214/232/244`）。
- **定数畳み込み**（`VisitExpressionImpl`, `:249-300`）：`ConstantValueOpt` を持つ式（null 許容型以外）は **`BoundLiteral` に置き換え、部分木は Lowering しない**。
- Lowering の前後で式の型が一致することを assert する（dynamic・タプル名・null 許容性の違いは除く, `:285`）。
- **プレースホルダ**（`BoundValuePlaceholder` など）は、`AddPlaceholderReplacement`（`:541`）で実際の式に置き換えてから中身を Lowering する。終了時に対応表が空であることを検査する。

### 2.3 ラムダとローカル関数（この段階では印を付けるだけ）

- `VisitLambda`（`:331`）：`_sawLambdas = true`。`_factory.CurrentFunction` を切り替え、本体をその場で Lowering する。
- `VisitLocalFunctionStatement`（`:385`）：序数を振る（`_availableLocalFunctionOrdinal++`）。ジェネリックなローカル関数では、dynamic の CallSite の容器が型パラメータを引き継ぐ必要があるので、専用の `LoweredDynamicOperationFactory` を作る。
- **式ツリー**：`Expression<T>` への変換では `_inExpressionLambda` を立て（`LocalRewriter_Conversion.cs:70-77`）、**計装を止め**、木の形を保つ別経路の Lowering を使う。`System.Linq.Expressions` のファクトリ呼び出しへの変換は、後で ClosureConversion が `ExpressionLambdaRewriter` に任せる（`ClosureConversion.cs:1642`）。

### 2.4 計装：`Lowering/Instrumentation/`

| クラス | 役割 |
|---|---|
| `Instrumenter`（`Instrumenter.cs:31` `NoOp`） | 既定は何もしない仮想フック群（`InstrumentExpressionStatement`、`InstrumentIfStatementConditionalGoto`、`InstrumentForEachStatement*`、`InstrumentCall` …） |
| `CompoundInstrumenter` | 前段（`Previous`）へ委譲するチェーン |
| **`DebugInfoInjector`** | 常に最も外側。多くのフックで `new BoundSequencePoint(node.Syntax, node)` を足す（`DebugInfoInjector_SequencePoints.cs:16-19`）。宣言子・プロパティ・`using` などは専用の範囲を使う |
| `CodeCoverageInstrumenter` | `--instrument TestCoverage`。カバレッジ用ペイロードの更新を挿入する |
| `LocalStateTracingInstrumenter` | ローカル変数とパラメータへの書き込みを追跡する API 呼び出しを挿入する（`try/finally` で包む） |

呼び出し側の決まった書き方：`if (this.Instrument && !node.WasCompilerGenerated) result = Instrumenter.InstrumentX(node, result);`。

### 2.5 制御構文：全てラベルと条件付きジャンプへ

| 構文 | Lowering 後の形 | 場所 |
|---|---|---|
| `if (c) S` | `GotoIfFalse c afterif; S; afterif:` | `LocalRewriter_IfStatement.cs:46-58` |
| `if (c) S1 else S2` | `GotoIfFalse c alt; S1; goto afterif; alt: S2; afterif:` | `:61-83` |
| `while (c) S` | `goto continue; start: { S  continue: GotoIfTrue c start; } break:` | `LocalRewriter_WhileStatement.cs:50-61` |
| `while` がローカルを宣言する場合 | `continue: { GotoIfFalse c break; S; goto continue; } break:`（スコープブロックに毎回上から入る。display class を毎回作り直すため） | `:119-150` |
| `do S while (c)` | `start: { S  continue: GotoIfTrue c start; } break:` | `LocalRewriter_DoStatement.cs:40-53` |
| `for (I; c; N) S` | `{ I; goto end; start: S; continue: N; end: GotoIfTrue c start; break: }` | `LocalRewriter_ForStatement.cs:82-97` |
| `try` | Release で try ブロックに副作用が無ければ catch を、finally に副作用が無ければ finally を削除する | `LocalRewriter_TryStatement.cs:15-40` |

`while` の条件を末尾に置くのは、ループ 1 回あたりの分岐を 1 つにするため。分岐先にはデバッグ用の隠しシーケンスポイントを置く。`else if` の連鎖は明示スタックで反復処理する。

### 2.6 `foreach`（`LocalRewriter_ForEachStatement.cs`）

`VisitForEachStatement`（`:34-69`）が対象の種類で分岐する。

1. **1 次元配列**（`:873`）：`for (A[] a = x, int i = 0; i < a.Length; i = i + 1) { V v = (V)a[i]; body }`
2. **多次元配列**：`GetLowerBound`/`GetUpperBound` による入れ子の `for`
3. **インライン配列**（`:734`）
4. **`string`、`Span<T>`、`ReadOnlySpan<T>`**（`CanRewriteForEachAsFor`, `:76-101`）：`Length` とインデクサによる `for`。列挙子オブジェクトを作らない。
5. **パターン／インターフェイス**（`RewriteEnumeratorForEachStatement`, `:119`）：

   ```csharp
   E e = ((C)x).GetEnumerator();
   try { while (e.MoveNext()) { V v = (V)(T)e.Current; body } }
   finally { ((IDisposable)e).Dispose(); }   // null 判定付き / await e.DisposeAsync() / Dispose 不要なら try 無し
   ```

   `await foreach` では、`MoveNextAsync` と `DisposeAsync` の await に固定の ID（0 と 1）を振る。finally の中の await なので `_sawAwaitInExceptionHandler` を立てる（`:390`）。

### 2.7 `using`・`lock`

- **`using`**（`LocalRewriter_UsingStatement.cs`）：`try { S } finally { … }`。値型ならそのまま `Dispose()`、参照型なら `if (r != null)`、null 許容型なら `if (r.HasValue)`（`:292-343`）。資源が null と分かっていれば try/finally を省く。`await using` は `DisposeAsync()` を待機する。
- **`lock`**（`LocalRewriter_LockStatement.cs`）：
  - 対象が `System.Threading.Lock` 型なら `using (x.EnterScope()) { body }`（`:51`）。
  - それ以外（.NET 4.0 以降の `Monitor.Enter(object, ref bool)` がある場合, `:124-139`）：

    ```csharp
    L $lock = arg; bool $lockTaken = false;
    try { Monitor.Enter($lock, ref $lockTaken); body }
    finally { if ($lockTaken) Monitor.Exit($lock); }
    ```

    旧 API しか無ければ `Enter` を try の前に置く。型パラメータの値は、`Enter` と `Exit` が同じオブジェクトを見るよう先にボックス化する。

### 2.8 パターン・`switch`（決定 DAG の Lowering）

- クラス階層：`PatternLocalRewriter` → `DecisionDagRewriter` → `BaseSwitchLocalRewriter` → `SwitchStatementLocalRewriter` / `SwitchExpressionLocalRewriter`。
- バインダが作った**決定 DAG**（バインダドキュメント §12）の各ノードにラベルを振り、テストを条件付きジャンプにする（`LowerDecisionDagCore`, `LocalRewriter.DecisionDagRewriter.cs:369`）。
  - 連続する値テストは `ValueDispatchNode` の木にまとめ、IL の `switch` 命令や二分探索にする（`:498-742`）。
  - 文字列の `switch` では、`<PrivateImplementationDetails>` に合成したハッシュ関数を使える（`EnsureStringHashFunction`, `:902`）。
- **`switch` 式**：結果を一時変数に入れる文の列になるので `BoundSpillSequence` を返し、`_needsSpilling = true` を立てる。どのアームにも一致しなければ `SwitchExpressionException` を投げる（`LocalRewriter_SwitchExpression.cs:130-157`）。
- **`is` パターン**（`LocalRewriter_IsPatternOperator.cs`）：DAG が直線なら `e1 && e2 && …` という純粋な式にする（`:186`）。そうでなければラベルと結果の一時変数を使う SpillSequence にする（`:133-176`）。

### 2.9 文字列

- **連結**（`LocalRewriter_StringConcat.cs`）：
  - `a + b + c + …` の左再帰を平坦化する（`:208`）。
  - `null` と `""` を除き、隣り合う定数をまとめる。入れ子の `Concat` を展開する。`s ?? ""` は `s` にする（`:287`）。
  - 4 引数以下は `String.Concat(string, …)`、それ以上は `Concat(string[])`。`char` を含む場合は `ReadOnlySpan<char>` 版。object の引数はボックス化を避けるため `.ToString()` を呼ぶ（`:72-85`）。
- **補間文字列**（`LocalRewriter_StringInterpolation.cs`）：
  - 穴が全て書式指定なしの文字列なら連結にする。
  - そうでなければ `DefaultInterpolatedStringHandler` ＋ `ToStringAndClear()` を使う。それも無ければ `String.Format`。
  - ハンドラパターンは `var h = new H(literalLength, formattedCount, …, out bool ok); if (ok) { h.AppendLiteral(…); h.AppendFormatted(…); }` の形にする（`:29-130`）。

### 2.10 演算子・アクセス

| 対象 | Lowering | 場所 |
|---|---|---|
| 複合代入 `a.P += v`、`c[i] += v` | 受け手とインデックスを一時変数に退避し、**1 回だけ評価する**：`t = a; t.P = t.P + v` | `LocalRewriter_CompoundAssignmentOperator.cs:724-745` |
| `?.` | 通常は `BoundLoweredConditionalAccess` として IL 生成に任せる（`dup` と分岐）。dynamic なら `(object)t != null ? access : default` | `LocalRewriter_ConditionalAccess.cs:38-190` |
| `??` | `T?` で右辺が `default(T)` なら `GetValueOrDefault()`。一般には `var t = left; t != null ? t : right` | `LocalRewriter_NullCoalescingOperator.cs:138-169` |
| `??=` | `lhs ?? (lhs = right)`。null 許容値型は `HasValue` で分岐 | `LocalRewriter_NullCoalescingAssignmentOperator.cs:34-87` |
| 持ち上げ演算子 `int? + int?` | `x.HasValue & y.HasValue ? new R?(x.GetValueOrDefault() + y.GetValueOrDefault()) : default(R?)`。null と分かっている／値を持つと分かっている側の検査は省く | `LocalRewriter_BinaryOperator.cs:1421-1456` |
| タプル | 入れ子の `ValueTuple` コンストラクタ（8 要素以上は `TRest`）。`==` は要素ごとの `&&` | `LocalRewriter_TupleCreationExpression.cs:31-50` |
| 分解 `(a, b) = e` | 初期化 → `Deconstruct` 呼び出し → 変換 → 代入の 4 段の `BoundSequence`。`(_, _) = (1, 2)` は何も生成しない | `LocalRewriter_DeconstructionAssignmentOperator.cs:17-134` |
| オブジェクト／コレクション初期化子 | `t = new C(); t.P = …; t.Add(…); t`（`BoundSequence`）。`[Conditional]` な `Add` は省略される | `LocalRewriter_ObjectCreationExpression.cs:260-271` |
| コレクション式 | 目的の型の種類（配列／Span／`[CollectionBuilder]`／`IEnumerable` 実装型／`IList<T>` など）で分岐。インライン配列を使った Span や `CollectionsMarshal` による `List<T>` の直接構築などの最適化がある | `LocalRewriter_CollectionExpression.cs:29-90` |
| 呼び出し | 名前付き引数を仮引数の順に並べ替える。**元の評価順序を保つため**、副作用のある引数は一時変数に退避する（`MakeArguments`, `:1192`）。空の params 配列は `Array.Empty<T>()` にする（`:1325`）。インターセプタ（`:136`） | `LocalRewriter_Call.cs` |
| イベント `+=` | `add_E` / `remove_E` の呼び出し | `LocalRewriter_Event.cs` |
| `dynamic` | 型ごとの静的クラスに `CallSite<T>` フィールドを遅延生成する：`if (site == null) site = CallSite<T>.Create(binder); site.Target(site, args)` | `LoweredDynamicOperationFactory.cs:599` |
| `^e`、`a..b` | `new Index(e, fromEnd: true)`、`Range` | `LocalRewriter_Index.cs`、`_Range.cs` |
| `await` | **残す**（AsyncRewriter が処理する）。値を使う場合は Spill 用の一時変数に入れる SpillSequence にする | `LocalRewriter_Await.cs:34-55` |
| `yield` | シーケンスポイントを足すだけで残す | `LocalRewriter_Yield.cs` |

`checked`/`unchecked` 用のノードは無い。checked かどうかは、バインダが演算子と変換の種類（`BinaryOperatorKind.Checked`）に焼き込んでいる。

---

## 3. `SpillSequenceSpiller`：式の中の文を外へ出す（`Lowering/SpillSequenceSpiller.cs`、1,611 行）

**目的**：IL の評価スタックは、`await` による中断（メソッドからの return）をまたいで保存できない。また、ラベルを含むような式（`switch` 式）も評価スタックが空の位置にしか置けない。そこで、**`BoundSpillSequence`（文を含む式）を全て消し、それらが文のレベル（評価スタックが空の位置）にだけ現れるようにする**。

**仕組み**：

- 内部用の疑似ノード `BoundSpillSequenceBuilder`（種別 `(BoundKind)byte.MaxValue`, `:19`, `:37`）に、一時変数と文を溜めながら上へ伝える。最後に `UpdateStatement`（`:282`）が、溜まった文を元の文の前に置く。
- **評価順序の保持**（`VisitExpressionList`, `:631-700`）：Spill を含む最後のオペランドより**前に評価されるオペランドは、全て一時変数へ退避する**。

  ```csharp
  F(a(), await b(), c())
  // ↓
  var t1 = a();            // a() は await より前に評価済みでなければならない
  var t2 = await b();      // 文のレベル
  F(t1, t2, c());
  ```

  リテラル、`this`、Spill 済みのローカル、readonly/static なフィールドは退避しなくてよい（`:378-425`）。
- `x && await y` は `tmp = x; if (tmp) { …; tmp = await y; }`、`?:` は一時変数付きの `if` にする（`:1048-1230`）。

LocalRewriter が Spill を要する形を作ったとき（使われる `await`、`switch` 式、直線でない `is` パターン、コレクション式の展開、入れ子の `stackalloc` など）にだけ `_needsSpilling` が立ち、Spiller が走る。

---

## 4. クロージャ変換（`Lowering/ClosureConversion/`）

### 4.1 全体の流れ

```csharp
internal sealed partial class ClosureConversion : MethodToClassRewriter           // ClosureConversion.cs:73
public static BoundStatement Rewrite(BoundStatement loweredBody, NamedTypeSymbol thisType, ParameterSymbol? thisParameter,
    MethodSymbol method, int methodOrdinal, MethodSymbol substitutedSourceMethod,
    ArrayBuilder<EncLambdaInfo> lambdaDebugInfoBuilder, ..., ArrayBuilder<EncClosureInfo> closureDebugInfoBuilder,
    VariableSlotAllocator? slotAllocator, TypeCompilationState compilationState, BindingDiagnosticBag diagnostics, ...)   // :231
```

1. `Analysis.Analyze`：スコープの木を作り、捕捉を解析する（`:251`）。
2. `SynthesizeClosureEnvironments`：環境ごとに display class を作る（`:274`）。
3. `SynthesizeClosureMethods`：ラムダ・ローカル関数ごとにメソッドを作る（`:275`）。
4. 本体を書き換える（`:277`）。
5. 合成したメソッドを `compilationState.SynthesizedMethods` に積む（`:291-302`）。

### 4.2 解析：スコープの木と捕捉（`ClosureConversion.Analysis*.cs`）

- データモデル（`Analysis.Tree.cs`）：
  - `Scope`：親、子スコープ、入れ子の関数、宣言された変数。
  - `NestedFunction`：捕捉する変数、捕捉する環境、`this` を（推移的に）捕捉するか。
  - `ClosureEnvironment`：捕捉される変数、親環境、`IsStruct`。
- `ScopeTreeBuilder`（`:288`）が本体を歩いて木を作る。`AddIfCaptured`（`:606`）は、変数を宣言した関数と使った関数が違えば、間にある全ての関数の「捕捉変数」に加える。後方への `goto` があるスコープは親と併合できない。
- `Analysis.Analyze`（`Analysis.cs:70-103`）：

  ```csharp
  analysis.MakeAndAssignEnvironments();          // 捕捉変数を宣言するスコープごとに環境を 1 つ
  analysis.ComputeLambdaScopesAndFrameCaptures(); // 関数をどの環境（クラス）に置くか、環境の親子関係
  if (OptimizationLevel == Release)
      analysis.MergeEnvironments();              // 同じ関数群からしか参照されない環境を親に併合（デバッグ時の変数の見え方が変わるので Release のみ）
  analysis.InlineThisOnlyEnvironments();         // this だけを持つ環境は作らず、元の型のインスタンスメソッドにする
  ```

- **struct の環境**：捕捉する関数が全て「デリゲートに変換されない、async でもイテレータでもないローカル関数」の場合だけ、環境を **struct** にして `ref` 引数で渡す（`CanTakeRefParameters`, `Analysis.cs:38-41`）。**ヒープ割り当てが発生しない**。

### 4.3 合成されるメソッドの置き場所（`ClosureKind`）

| 種類 | 条件 | 置き場所 |
|---|---|---|
| `Static` | 捕捉なし、デリゲートに変換されない | 元の型の static メソッド |
| `Singleton` | 捕捉なし、デリゲートに変換される | 共有の **`<>c`** クラスのインスタンスメソッド |
| `ThisOnly` | `this` だけを捕捉 | 元の型のインスタンスメソッド |
| `General` | 変数を捕捉 | display class **`<>c__DisplayClass{メソッド}_{クロージャ}`** のメソッド |

捕捉の無いラムダを static メソッドではなくシングルトンのインスタンスメソッドにするのは、**デリゲート呼び出しはインスタンスメソッドのほうが速い**から（コメント `ClosureConversion.cs:1671-1674`）。

名前の例（`Symbols/Synthesized/GeneratedNames.cs`）：

- ラムダ `<M>b__0_0`、ローカル関数 `<M>g__Local|0_0`
- display class `<>c__DisplayClass0_0`、シングルトンのインスタンス `<>c.<>9`、デリゲートのキャッシュ `<>c.<>9__0_0`
- 捕捉された `this` は `<>4__this`、外側の環境への参照は `CS$<>8__locals1`
- EnC の世代は `#N` の接尾辞で表す

### 4.4 書き換え

- **環境の導入**（`IntroduceFrame`, `:674-757`）：スコープの入口で `var <>8__locals = new <>c__DisplayClass0_0();` を作り、外側の環境への参照とパラメータの値をコピーする。以降、そのスコープ内で捕捉変数を使う箇所は、全て `<>8__locals.x` というフィールドアクセスになる。
- **基底クラス `MethodToClassRewriter`**（`Lowering/MethodToClassRewriter.cs`）：
  - `proxies`（変数 → 置き換え）に従い、`VisitLocal`/`VisitParameter` をフィールドアクセスに置き換える（`:184-227`）。
  - `base.M()` を別の型へ移すと非仮想呼び出しができなくなるので、ラッパ `<>n__N` を合成する。
  - 型パラメータの置換は `BoundTreeToDifferentEnclosingContextRewriter` が行う。`VisitType` / `VisitMethodSymbol` を `TypeMap` で置換する（BoundTree ドキュメント §4.6 の「シンボルと型も訪問する Rewriter」の実例）。
- **ラムダ → デリゲート**（`RewriteLambdaConversion`, `:1632-1750`）：
  - 結果は `new BoundDelegateCreationExpression(receiver, 合成メソッド)`。受け手は、シングルトンなら `<>c.<>9`、static なら型、それ以外は環境。
  - **キャッシュ**：シングルトンのラムダ、またはループやラムダの中で毎回作られるラムダは、`cache ?? (cache = new D(...))` にする（`<>c.<>9__0_0` など）。デリゲート型がメソッドの型パラメータを含む場合はキャッシュしない。
  - **式ツリー**は、メソッドに取り出さず `ExpressionLambdaRewriter` で `Expression.Lambda(...)` などの呼び出しにする（`:1635-1645`）。
- **ローカル関数の呼び出し**（`RemapLocalFunction`, `:883-955`）：struct 環境を `ref` 引数として追加で渡す。

### 4.5 Edit and Continue への配慮

ラムダとクロージャの ID（序数と世代）は、`VariableSlotAllocator` を通じて**前の世代の ID を再利用**する（`GetLambdaId` `:1453`、`GetClosureId` `Analysis.cs:528`）。結果は `EncLambdaInfo` / `EncClosureInfo` として PDB に記録され、次の世代の EnC でソースのラムダと合成メソッドを対応付けるのに使われる。

---

## 5. ステートマシンの共通基盤（`Lowering/StateMachineRewriter/`）

### 5.1 `StateMachineRewriter`（抽象基底）

`Rewrite()`（`StateMachineRewriter.cs:93-138`）：

1. 型を開く。
2. 制御フィールド（状態など）を作る。
3. **`IteratorAndAsyncCaptureWalker` で持ち上げる変数を決める**。
4. 変数ごとのフィールドを作る。
5. `MoveNext` などのメソッドを実装する。
6. 元のメソッド（キックオフメソッド）の本体を「ステートマシンを作って開始する」コードに置き換える。

**持ち上げる変数の決定**（`IteratorAndAsyncCaptureWalker.cs`）：

- **確定代入解析（`DefiniteAssignmentPass`）を流用する**。`await` / `yield return` の位置で全変数を「未代入」に戻し、その後で「未代入の読み取り」と報告された変数＝**中断をまたいで生きている変数**を持ち上げる（`:158`, `:193-205`, `:277`）。フロー解析ドキュメントの DataFlowsOut と同じ発想。
- `this`、yield を含む try の finally で使う変数も持ち上げる。
- `ref` ローカルや ref struct が中断をまたぐ場合はエラー（`ERR_ByRefTypeAndAwait`、`ERR_RefLocalAcrossAwait`）。
- **Debug ビルドでは、デバッガで値を見られるよう、ユーザーのローカル変数とパラメータを全て持ち上げる**（`:116-155`）。

**フィールド名**：

- `<>1__state`（状態）、`<>2__current`（現在の要素）、`<>4__this`
- `<x>5__1`（持ち上げたユーザー変数）、`<>s__N`（合成変数）、`<>7__wrapN`（再利用されるフィールド）
- `<>3__p`（列挙可能型でのパラメータの初期値）、`<>l__initialThreadId`
- `<>t__builder`（async のビルダー）、`<>u__N`（awaiter）

Release ではユーザー変数のフィールドを**スコープ単位で型ごとに再利用**する（`GetOrAllocateReusableHoistedField`, `MethodToStateMachineRewriter.cs:466-494`）。

### 5.2 状態番号（`Core/Portable/CodeGen/StateMachineState.cs`）

| 値 | 意味 |
|---|---|
| ≥ 0 | 再開可能な位置（async は 0 から、イテレータは 1 から増加） |
| -1 | 実行中、または未開始 |
| -2 | 終了 |
| ≤ -3 | イテレータの finally 用の状態（減少）。async イテレータの yield 用（-4 から減少） |

`ResumableStateMachineStateAllocator` は、EnC では前の世代の状態番号を再利用する。前の世代にあって削除された状態に来たら、ホットリロードの例外を投げるコードを生成する。

### 5.3 `MethodToStateMachineRewriter`：MoveNext の骨格

- `cachedState = this.<>1__state;` で状態をローカルに読み、**`switch (cachedState) { case n: goto resume_n; … }`** で再開位置に飛ぶ（`Dispatch`, `:250-274`）。
- **try の中の再開位置**：IL では try の外から中へ直接ジャンプできない。そのため、外側の dispatch は「try の直前のラベル」へ飛び、try ブロックの先頭に**入れ子の dispatch** を置いて中のラベルへ飛ぶ（`VisitTryStatement`, `:729-773`）。
- **finally の制御**：中断（return）のときは finally を実行してはならない。そこで finally を `if (cachedState < 0) { finally }` で包む（`ShouldEnterFinallyBlock`, `:780-783`）。再開可能な状態（≥ 0）で抜けるときは実行されない。

---

## 6. イテレータ（`Lowering/IteratorRewriter/`）

- `IteratorRewriter.Rewrite`（`IteratorRewriter.cs:49-95`）。ステートマシンは常に **class**（`IteratorStateMachine.cs:46-49`）。戻り値型が `IEnumerable<T>` か `IEnumerator<T>` かで、実装するインターフェイスが変わる。
- 生成されるメンバー：
  - `MoveNext`、`IDisposable.Dispose`
  - `Current`（`<>2__current` を返す）
  - `Reset`（`NotSupportedException` を投げる, `:238-242`）
  - `.ctor(int state)`
  - 列挙可能型なら `GetEnumerator`：**生成したスレッドが最初に呼んだ場合は自身を再利用し**（`state == -2 && initialThreadId == 現在のスレッド`）、そうでなければ新しいインスタンスを作る（`StateMachineRewriter.cs:406-`）。
- キックオフメソッド：`return new <M>d__0(-2 /*列挙可能型*/ or 0) { <>4__this = this, <>3__p = p };`
- **`yield return e`**（`IteratorMethodToStateMachineRewriter.cs:332-354`）：

  ```csharp
  this.<>2__current = e; this.<>1__state = n; return true;
  resume_n: this.<>1__state = <現在の finally フレームの状態 or -1>;
  ```

  `yield break` は `return false` へ。
- **yield を含む try/finally**（`:371-467`）：finally の本体を別メソッド **`<>m__Finally1`** に移す。状態を -3 から減少させて割り当て、try の本体は `state = -3; body; this.<>m__Finally1();` にする。`Dispose()` は状態で `switch` し、該当する finally メソッドを（入れ子なら外側も）呼ぶ（`:162-293`）。yield を含む try がある場合、MoveNext 全体を `try { } fault { Dispose(); }` で包む（`:131-152`）。

---

## 7. async（`Lowering/AsyncRewriter/`）

### 7.1 `AsyncRewriter`

- **struct か class か**（`AsyncRewriter.cs:74`）：

  ```csharp
  // CLR は struct へのフィールド追加をサポートしないので、EnC を有効にするには class にする必要がある
  var typeKind = (Options.EnableEditAndContinue || method.IsIterator) ? TypeKind.Class : TypeKind.Struct;
  ```

  通常は Debug（EnC 有効）で class、Release で struct になる。**struct のステートマシンは、同期的に完了した場合にヒープ割り当てが発生しない**。最初に中断したとき、ビルダーが箱詰めする。
- **ビルダーの選択**（`AsyncMethodBuilderMemberCollection.TryCreate`, `:112`）：
  - `void` → `AsyncVoidMethodBuilder`
  - `Task` → `AsyncTaskMethodBuilder`
  - `Task<T>` → `AsyncTaskMethodBuilder<T>`
  - それ以外は `[AsyncMethodBuilder]` 属性（型またはメソッド）で指定されたカスタムビルダー（`ValueTask` もこの経路）
  - async イテレータ → `AsyncIteratorMethodBuilder`
- キックオフメソッド（`:201-276`）：

  ```csharp
  var sm = new <M>d__0();               // class の場合のみ new
  sm.<>t__builder = AsyncTaskMethodBuilder<T>.Create(); sm.<>4__this = this; sm.p = p; sm.<>1__state = -1;
  sm.<>t__builder.Start(ref sm);
  return sm.<>t__builder.Task;
  ```

### 7.2 `MoveNext` と `await` の展開（`AsyncMethodToStateMachineRewriter.cs`）

```csharp
void MoveNext() {
    int cachedState = this.<>1__state;
    try {
        switch (cachedState) { case 0: goto resume_0; ... }
        <body>
    } catch (Exception ex) { this.<>1__state = -2; <持ち上げた変数の後始末>; this.<>t__builder.SetException(ex); return; }
    exprReturnLabel: this.<>1__state = -2; <後始末>; this.<>t__builder.SetResult(result);
}
```

`return e;` は結果を変数に保存して `exprReturnLabel` へ飛ぶ。

**`[r =] await e;`**（`VisitAwaitExpression` `:348-412`、`GenerateAwaitForIncompleteTask` `:467-555`）：

```csharp
$awaiter = e.GetAwaiter();
if (!$awaiter.IsCompleted) {
    this.<>1__state = cachedState = 0;                          // 再開位置の状態番号
    this.<>u__1 = $awaiter;                                     // awaiter をフィールドへ（awaiter の型ごとに共有）
    this.<>t__builder.AwaitUnsafeOnCompleted(ref $awaiter, ref this);   // ICriticalNotifyCompletion なら Unsafe 版
    return;                                                     // ← 中断。完了時にビルダーが MoveNext を再度呼ぶ
resume_0:
    $awaiter = this.<>u__1; this.<>u__1 = default; this.<>1__state = cachedState = -1;
}
[r =] $awaiter.GetResult();
```

**完了済みのタスクなら `IsCompleted` が真になり、中断もフィールドへの保存も無しに進む**（高速パス）。中断の前後には、デバッガ用の yield point と resume point の印（`NoOp`）を置く。

### 7.3 `AsyncExceptionHandlerRewriter`：catch/finally の中の await

IL では、catch/finally ブロックの中から外へ出て（中断して）戻ってくることができない。そこで ClosureConversion より前（例外変数が捕捉される前）に、ハンドラを通常のブロックへ移す（`AsyncExceptionHandlerRewriter.cs:44-108`）。

```csharp
// finally { await X; }  →
object pendingException = null;
try { body; goto finallyLabel; } catch (object ex) { pendingException = ex; }
finallyLabel: { await X; if (pendingException != null) ExceptionDispatchInfo.Capture(pendingException).Throw(); <保留した分岐の再開> }
// catch (E e) { await X; }  →  catch (E t) { pendingException = t; pendingCatch = 1; }  …  switch (pendingCatch) { case 1: await X; }
```

try の中の `return` や `break` は「保留した分岐」として番号で記録し、finally の後で再開する。例外は `ExceptionDispatchInfo` で再スローして、元のスタックトレースを保つ。

### 7.4 async イテレータ（`IAsyncEnumerable<T>`）

- `AsyncIteratorRewriter`：`ManualResetValueTaskSourceCore<bool>` 型の `<>v__promiseOfValueOrEnd` を使い、`MoveNextAsync()` は `new ValueTask<bool>(this, version)` を返す。`IValueTaskSource<bool>` も自身で実装し、割り当てなしで結果を返す。
- `yield return e`：`current = e; state = n; promise.SetResult(true); return;` → 再開時に `if (disposeMode) goto 後始末;`。
- `yield break` と `DisposeAsync` は `<>w__disposeMode` を立てて finally を実行させる。`[EnumeratorCancellation]` 用の `CancellationTokenSource`（`<>x__combinedTokens`）も合成する。

### 7.5 runtime-async（`AsyncRewriter/RuntimeAsyncRewriter.cs`）

.NET の新しい「ランタイム実装の async」に対応する経路。**ステートマシンを生成しない**。

- **条件**（`CSharpCompilation.IsRuntimeAsyncEnabledIn`, `Compilation/CSharpCompilation.cs:355-400`）：
  - ランタイムが対応している。
  - `[RuntimeAsyncMethodGeneration]` 属性か `runtime-async=on` 機能フラグがある。
  - 戻り値型が `Task` / `Task<T>` / `ValueTask` / `ValueTask<T>`。
- メソッドに `MethodImplAttributes.Async` を付け（`SourceMethodSymbolWithAttributes.cs:1768-1776`）、ランタイムに中断と再開を任せる。
- `await e` は `System.Runtime.CompilerServices.AsyncHelpers.Await(e)`、カスタム awaiter なら `AwaitAwaiter` / `UnsafeAwaitAwaiter` の呼び出しにする（`:126-226`）。
- ビルダー、`MoveNext`、状態フィールドは無い。持ち上げるのは、await をまたぐ by-ref の値だけ（通常のローカルへ）。最後に `SpillSequenceSpiller` を通す（`:64`）。

---

## 8. 設計上の要点まとめ

1. **段階的な縮退**：各段階が特定の機能を消し、`PipelinePhaseValidator` が DEBUG でそれを保証する。IL 生成器が知るべきノードは、ラベル、条件付きジャンプ、try、呼び出し、代入などの小さな集合に絞られる。
2. **評価順序と単一評価の厳守**：複合代入の受け手、名前付き引数、`await` の前の部分式は一時変数へ退避し、C# の「左から右へ 1 回だけ評価」という意味を保つ。
3. **割り当てを減らす最適化**：
   - struct の環境（ローカル関数）、シングルトンとデリゲートのキャッシュ、環境の併合（Release）
   - struct の async ステートマシン、awaiter フィールドの共有、持ち上げフィールドの再利用
   - `Array.Empty`、Span による `foreach`、`String.Concat` のオーバーロード選択、コレクション式の特殊化
4. **デバッグ体験と最適化の両立**：
   - シーケンスポイントを常に生成し（IL は PDB の有無に依存しない）、Debug では変数を全て持ち上げる。
   - 環境の併合とフィールドの再利用は Release だけで行う。
   - EnC のために、ラムダ、クロージャ、状態番号、awaiter、持ち上げ変数のスロットを前の世代と対応付ける。
5. **既存の解析の流用**：持ち上げる変数の決定に確定代入解析（`DefiniteAssignmentPass`）を、`switch` とパターンの Lowering にバインダの決定 DAG をそのまま使う。

---

## 付録 A: 主要ファイル早見表

| ファイル | 行数 | 役割 |
|---|---:|---|
| `Lowering/LocalRewriter/LocalRewriter.cs` | 1,184 | 入口・訪問規則・定数畳み込み |
| `LocalRewriter_BinaryOperator.cs` / `_Conversion.cs` / `_Call.cs` | 2,495 / 2,010 / 1,889 | 演算子（持ち上げ）・変換・呼び出し |
| `LocalRewriter_CollectionExpression.cs` / `_ForEachStatement.cs` | 1,661 / 1,306 | コレクション式・foreach |
| `LocalRewriter.DecisionDagRewriter.cs` / `.PatternLocalRewriter.cs` | 1,246 / 721 | パターン・switch |
| `LocalRewriter_CompoundAssignmentOperator.cs` / `_StringConcat.cs` | 1,119 / 769 | 複合代入・文字列連結 |
| `LoweredDynamicOperationFactory.cs` | 933 | dynamic |
| `Lowering/SpillSequenceSpiller.cs` | 1,611 | 式中の文の持ち上げ |
| `Lowering/SyntheticBoundNodeFactory.cs` | 1,917 | 合成ノードのファクトリ |
| `Lowering/Instrumentation/*` | — | シーケンスポイント・カバレッジ・状態追跡 |
| `Lowering/ClosureConversion/ClosureConversion*.cs` | — | クロージャ変換・解析 |
| `Lowering/ClosureConversion/ExpressionLambdaRewriter.cs` | — | 式ツリー |
| `Lowering/MethodToClassRewriter.cs` / `BoundTreeToDifferentEnclosingContextRewriter.cs` | — | 変数のフィールド化・型の置換 |
| `Lowering/StateMachineRewriter/*` | — | ステートマシン共通基盤・捕捉解析 |
| `Lowering/IteratorRewriter/*` | — | イテレータ |
| `Lowering/AsyncRewriter/*` | — | async・async イテレータ・catch/finally 内の await・runtime-async |

## 付録 B: 例

```csharp
async Task<int> SumAsync(int[] xs) {
    int sum = 0;
    foreach (var x in xs) sum += await GetAsync(x);
    return sum;
}
```

1. **LocalRewriter**：
   - `foreach` を配列の `for` にする：`int[] a = xs; int i = 0; goto cont; start: { int x = a[i]; sum = sum + await GetAsync(x); i = i + 1; } cont: if (i < a.Length) goto start;`
   - `sum + await …` の `await` は使われる値なので SpillSequence にし、`_needsSpilling` を立てる。
2. **SpillSequenceSpiller**：`var t1 = sum; var t2 = await GetAsync(x); sum = t1 + t2;`。`sum` は await より前に読まれるので退避される。
3. **ClosureConversion**：ラムダが無いのでスキップ。
4. **AsyncRewriter**：
   - 型 `<SumAsync>d__0`（Release では struct）を作る。
   - **中断をまたいで生きている** `xs`（パラメータ）、`sum`、`a`、`i`、`t1` をフィールド（`<sum>5__2`、`<>7__wrap2` …）にする。`x` と `t2` は await の後でしか使わないので、Release ではローカルのまま。
   - `MoveNext` に `switch (state) { case 0: goto resume_0; }` を置き、await を「`IsCompleted` でなければ awaiter を `<>u__1` に保存して `AwaitUnsafeOnCompleted`、`return`、`resume_0:`」に展開する。
   - キックオフメソッドは `Create` → `Start(ref sm)` → `return builder.Task`。
