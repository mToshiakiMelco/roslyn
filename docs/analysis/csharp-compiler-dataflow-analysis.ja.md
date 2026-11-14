# Roslyn C# コンパイラ データフロー解析 詳細解説

対象: `src/Compilers/CSharp/Portable/FlowAnalysis/`（約 28,500 行）
基準コミット: `88579e780fe7` (2026-11-13)
※ 行番号はこのコミット時点のもの。パスは特記なき限り `FlowAnalysis/` からの相対パス。

---

## 0. 要約

- Roslyn C# のフロー解析は、**束縛済みツリー（Bound Tree）を直接歩く「構文主導・前向き」の格子（lattice）データフロー解析**である。CFG（制御フローグラフ）を明示的には構築せず、`BoundTreeVisitor` による再帰的な走査で文の構造そのものを制御フローとして解釈する。
- 全解析の共通基盤は汎用クラス `AbstractFlowPass<TLocalState, TLocalFunctionState>`。状態型 `TLocalState` と格子演算 `Join`／`Meet` を差し替えることで、
  - 到達可能性（`ControlFlowPass`、状態 = `bool`）
  - 確定代入（`DefiniteAssignmentPass`、状態 = `BitVector`）
  - Null 許容解析（`NullableWalker`、状態 = スロットごとの `NullableFlowState`）
  - IDE 向け領域解析（`DataFlowsInWalker` など 10 種以上）
  を同じ骨格の上で実装している。
- 後方分岐（ループ・後方 `goto`・ローカル関数の再帰的な状態変化）は「状態が使われた後に変化した」フラグ `stateChangedAfterUse` を立て、**本体全体を不動点に達するまで再走査**することで処理する。
- `try/finally` は例外経路を個別に追跡せず、`Meet` を使った保守的な近似で処理する（言語仕様の要求には十分）。「代入解除」を追跡する必要がある解析（DataFlowsOut、Nullable）のみ `NonMonotonicState` で補正する。

---

## 1. コンパイルパイプライン上の位置

`Compiler/MethodCompiler.cs` の `CompileMethod` における 1 メソッドの処理順:

| 順 | 処理 | 場所 |
|---|---|---|
| 1 | 本体の束縛 `BindMethodBody` | `MethodCompiler.cs:1060`, `:1888` |
| 1a | └ Null 許容解析 `NullableWalker.AnalyzeIfNeeded` | `MethodCompiler.cs:1981`, `:2079`（初期化子は `:646`, `:1027`） |
| 1b | └ ref 安全性解析 `RefSafetyAnalysis.Analyze` | `MethodCompiler.cs:1055` ほか |
| 2 | 雑多な診断 `DiagnosticsPass.IssueDiagnostics` | `MethodCompiler.cs:1145` 付近 |
| 3 | **フロー解析 `FlowAnalysisPass.Rewrite`** | `MethodCompiler.cs:1151` |
| 4 | 下位変換（Lowering: LocalRewriter → クロージャ変換 → 非同期/イテレータのステートマシン化） | `LowerBodyOrInitializer` |

`FlowAnalysisPass.Analyze`（`FlowAnalysisPass.cs:210-221`）は次の 2 パスを順に呼ぶだけである:

```csharp
needsImplicitReturn = ControlFlowPass.Analyze(compilation, method, block, diagnostics);
DefiniteAssignmentPass.Analyze(compilation, method, block, diagnostics,
                               out implicitlyInitializedFieldsOpt, requireOutParamsAssigned: true);
```

その結果を使って `FlowAnalysisPass.Rewrite`（`:31-`）は
- `void`／イテレータ／`Task` 返却 async で終端が到達可能なら **暗黙の `return` を末尾に追加**（`AppendImplicitReturn`）
- C# 11 以降の構造体コンストラクタで未代入フィールドがあれば **既定値初期化を先頭に挿入**（`PrependImplicitInitializations`、自動既定構造体）

という *ツリー書き換え* も行う。つまりフロー解析は診断だけでなく、以降の Lowering／コード生成に影響する。

これとは別に、IDE／アナライザー向け公開 API `SemanticModel.AnalyzeDataFlow` / `AnalyzeControlFlow`（`Compilation/CSharpSemanticModel.cs:2430-2499`）が、同じ基盤上の「領域解析 walker」をオンデマンドで起動する（§7）。

---

## 2. 共通基盤 `AbstractFlowPass<TLocalState, TLocalFunctionState>`

`AbstractFlowPass.cs`（3,903 行）＋ partial ファイル群
（`.ILocalState.cs`, `.PendingBranchesCollection.cs`, `_LocalFunctions.cs`, `_Switch.cs`）。

### 2.1 格子（lattice）の抽象

`AbstractFlowPass.ILocalState.cs` で派生クラスが実装すべき格子演算が定義されている:

| メンバ | 意味 | 満たすべき性質 |
|---|---|---|
| `TopState()` | 到達可能だが情報なし。メソッド入口の状態 | |
| `UnreachableState()` | 絶対的なボトム。`return`/`throw` の後 | C# では到達不能状態は 1 種のみ |
| `ReachableBottomState()` | 到達可能かつ他の状態に Join しても影響しない状態（全変数代入済み／全て非 null） | 「代入解除」を追跡する解析で必須 |
| `Join(ref self, ref other)` | **合流**（if/else 合流、同一ラベルへの goto）。格子を「上」へ | `Join(⊥,X)=X`, `Join(Top,X)=Top` |
| `Meet(ref self, ref other)` | **加算的合成**（try 内 return の状態と finally 終端状態の合成など）。格子を「下」へ | `Meet(⊥,X)=⊥`, `Meet(Top,X)=X` |

`Join`/`Meet` は「`self` が変化したか」を `bool` で返し、これが不動点判定に使われる。

状態インターフェイス `ILocalState` は `Clone()` と `Reachable` のみ。

### 2.2 走査中の状態

```csharp
protected TLocalState State;           // 通常状態
protected TLocalState StateWhenTrue;   // 条件状態（真の場合）
protected TLocalState StateWhenFalse;  // 条件状態（偽の場合）
protected bool IsConditionalState;     // 現在条件状態か
```
（`AbstractFlowPass.cs:105-110`）

- **条件状態（conditional state）** が Roslyn のフロー解析の鍵。ブール式を評価した直後は「真の場合の状態」と「偽の場合の状態」を別々に保持する。
  - `Split()`：通常状態を複製して条件状態へ（`:146`）
  - `Unsplit()`：`Join(StateWhenTrue, StateWhenFalse)` して通常状態へ（`:154-160`）
  - `VisitCondition(expr)` は条件状態を作る。`VisitRvalue(expr)` は最後に `Unsplit()` する（`:626-665`）。
- これにより `if (x != null && x.Length > 0)`、`if (int.TryParse(s, out var n) && n > 0)` のような短絡評価での状態の精密な伝搬が可能になる。

`&&`/`||`（`VisitBinaryLogicalOperatorChildren`, `:2427-2540`）:
- 深い左再帰でスタックを溢れさせないよう、左側の連鎖を **明示スタックに積んで反復処理** する。
- 左辺を `VisitCondition` → `&&` なら `StateWhenTrue`、`||` なら `StateWhenFalse` から右辺を評価 → 右辺の条件状態と左辺の残り側を合成して全体の条件状態を作る。

`!` 演算子は `StateWhenTrue` と `StateWhenFalse` を入れ替えるだけ（`VisitUnaryOperator`, `:2797-2810`）。`is` パターン式も同様に条件状態を生成する（`VisitIsPatternExpression`, `:921-`）。
`true`/`false` 定数は `SplitIfBooleanConstant`（`:1682-`）で片側を到達不能にする。これにより `while (true)` の後が到達不能になる。

### 2.3 分岐の扱い：Pending Branches

CFG を持たない代わりに、**ジャンプ文は「保留中の分岐（PendingBranch）」としてキューに積み、ジャンプ先を処理するコードがそれを回収する**。

```csharp
internal sealed class PendingBranch {
    public readonly BoundNode Branch;   // goto/break/continue/return/yield ...
    public TLocalState State;           // ジャンプ時点の状態
    public LabelSymbol Label;           // ジャンプ先（return なら null）
    public bool IsConditionalState; public TLocalState StateWhenTrue, StateWhenFalse;
}
```
（`AbstractFlowPass.cs:392-418`）

- `goto L;` → `PendingBranches.Add(new PendingBranch(node, State, L)); SetUnreachable();`（`:3377-3383`）
- ラベル `L:` 到達時 `VisitLabel` → その時点の状態に、L への保留分岐の状態を全て `Join`（`ResolveBranches`, `:775-806`）。各ラベルの状態は `_labels` 辞書に保存。
- `break`/`continue` はループ／switch の終端で `ResolveBreaks`/`ResolveContinues` が回収（`:722-756`）。
- `return` は最後まで残り、`Scan` 終了時に `RemoveReturns()` で取り出され、呼び出し側（例えば out 引数チェック）に返る。
- `SavePending()`/`RestorePending()`（`:825-880`）でスコープ単位に保留分岐を退避・復元。`try` やラムダ本体への「外からの分岐」を禁止する言語規則もこれで表現される。

### 2.4 ループと不動点反復

```csharp
// while (cond) { body; continue: } break:
LoopHead(node);                 // 前回のループ先頭状態と Join し保存
VisitCondition(node.Condition);
SetState(StateWhenTrue);  VisitStatement(node.Body);
ResolveContinues(node.ContinueLabel);
LoopTail(node);                 // 末尾状態を先頭状態に Join。変化したら stateChangedAfterUse = true
ResolveBreaks(StateWhenFalse, node.BreakLabel);
```
（`VisitWhileStatement`, `:2350-2363`、`LoopHead`/`LoopTail`, `:694-716`）

外側のドライバ `Analyze`（`:436-452`）は:

```csharp
do {
    regionPlace = RegionPlace.Before;
    State = initialState ?? TopState();
    PendingBranches.Clear(); stateChangedAfterUse = false; Diagnostics.Clear();
    returns = Scan(ref badRegion);     // メソッド本体全体を 1 回走査
} while (stateChangedAfterUse);
```

すなわち **ワークリスト法ではなく「本体全体を再走査」する**。ループ先頭・後方ラベル・ローカル関数の状態は単調に変化するので必ず停止する。ソースコメント（`:68-78`）にある通り、`goto` を逆順に連鎖させるような病的コードでは二乗時間になり得るが、実用上は 1～2 回で収束する。
（確定代入のような単調な解析では、ループ本体を 2 回見れば十分なことが多い。）

`_nonMonotonicTransfer`（`:112-123`）：転送関数が格子を上下どちらにも動かす解析（領域解析・Nullable）では `true`。これにより try/finally などで追加の補正が入る（§2.6）。

### 2.5 if 文のスタック化

`VisitIfStatement`（`:1781-1830`）も `else if` の長い連鎖でスタックを溢れさせないよう、`(trueState, node)` を明示スタックに積んで反復処理し、最後にまとめて `Join` する。式の深い再帰に対しては `VisitExpressionOrPatternWithoutStackGuard` と `InsufficientExecutionStack` 検出で `ERR_InsufficientStack` を出す（`:372-388`）。

### 2.6 try / catch / finally（`VisitTryStatement`, `:1832-1912`）

方針（クラス冒頭コメント `:20-35`）: **例外が発生し得る全地点の状態は追跡しない**。その代わり:

1. `initialState` = try 開始時点の状態（catch / finally の入口状態として使う。「try 内のどこで例外が起きても、少なくとも try 開始時点の状態は成立している」という保守的近似）。
2. try ブロック → 各 catch ブロックを `initialState` から開始 → 終端状態を `Join`。
3. finally は `initialState` から解析。その後、
   - try/catch から finally を通って外へ出る全ての保留分岐（return/goto/break）の状態に、finally 終端状態を **`Meet`**（finally で代入されたものは分岐先でも代入済み）。`yield return` は finally を通らないので除外。
   - 終端状態にも `Meet`。
4. `_nonMonotonicTransfer` な解析では、try/catch 内で一時的に「上へ動いた」状態（例: 代入解除、null になった）を `NonMonotonicState` に蓄積し、catch/finally 入口に `Join` で反映（`VisitTryBlockWithAnyTransferFunction`, `:1924-1950`）。これにより「try 内で一瞬 null が代入された後、例外で catch に来る」ケースも健全に扱う。

### 2.7 switch（`AbstractFlowPass_Switch.cs`）

- 各セクションへの入口は、パターンマッチングの **決定 DAG（`ReachabilityDecisionDag`）** で到達可能と判定されたラベルなら switch 式評価後の状態、そうでなければ到達不能（`VisitSwitchStatementDispatch`, `:34-60`）。
- セクション終端が到達可能なら（エラーだが）暗黙の break として扱い `Join`。
- `ControlFlowPass` がここで `ERR_SwitchFallThrough` / `ERR_SwitchFallOut` を報告（`ControlFlowPass.cs:399`）。

### 2.8 ローカル関数（`AbstractFlowPass_LocalFunctions.cs`）

ローカル関数は「宣言位置」でなく「呼び出し位置」で効果が生じ、前方呼び出し・相互再帰もあるため特別扱い。各ローカル関数ごとに **要約（summary）** を保持する:

```csharp
internal abstract class AbstractLocalFunctionState {
    public TLocalState StateFromBottom; // 呼び出し時に現在状態と Join（状態を弱める効果。例: 捕捉変数が null になり得る）
    public TLocalState StateFromTop;    // 呼び出し時に現在状態と Meet（状態を強める効果。例: 捕捉変数に代入する）
    public bool Visited;                // 要約が既に呼び出し側で使われたか
}
```

- 本体解析（`VisitLocalFunctionStatement`, `:62-140`）は `TopState()` から開始し、全 return の状態を `Join` した `stateAtReturn` で要約を更新（`RecordStateChange`）。
- **要約が変化し、かつ既に `Visited`（誰かが古い要約を使った）なら `stateChangedAfterUse = true`** → 本体全体を再走査。これがローカル関数の不動点計算。
- 確定代入版（`DefiniteAssignment.LocalFunctions.cs`）は更に `ReadVars`（ローカル関数内で、代入前に読まれる捕捉変数）を記録し、**呼び出し地点で** それらが確定代入済みかを検査する（`CheckIfAssignedDuringLocalFunctionReplay`）。

```csharp
void M() {
    int x;
    Local();      // ← ここで CS0165 (x 未代入) が報告される
    x = 1;
    Local();      // OK
    void Local() => Console.WriteLine(x);
}
```

### 2.9 ラムダ

基底の `VisitLambda` は何もしない（`AbstractFlowPass.cs:1280`、ラムダ本体は実行タイミング不明なので外側の状態に影響させない）。`DefiniteAssignmentPass.VisitLambda`（`DefiniteAssignment.cs:2420-2464`）は:
- ラムダ宣言時点の状態をコピーして本体を解析（捕捉変数は宣言時点で確定代入されている必要がある）
- 終了後、状態は `stateAfterLambda` に戻す（`Join` は「領域解析以外では no-op」とコメントされている）。

### 2.10 領域（region）追跡

`trackRegions: true` で構築すると、`firstInRegion`/`lastInRegion` のノードに入る／出る時に `EnterRegion()`/`LeaveRegion()` が呼ばれ、`regionPlace` が `Before → Inside → After` と遷移する（`:258-292`）。領域解析 walker はこれをフックして「領域内外での読み書き」を分類する（§7）。走査終了時に `After` に達していなければ `badRegion = true`。

---

## 3. 到達可能性解析 `ControlFlowPass`

`ControlFlowPass.cs`（437 行）。最も単純な具体例。

```csharp
internal struct LocalState : ILocalState {
    internal bool Alive;     // 到達可能か
    internal bool Reported;  // 到達不能コード警告を既に出したか（同じ到達不能区間で警告を重複させない）
}
Join: Alive |= other.Alive; Reported &= other.Reported;
Meet: Alive &= other.Alive; Reported &= other.Reported;
TopState = (true, false); UnreachableState = (false, State.Reported)
```
（`:39-110`）

報告する主な診断:

| 診断 | 内容 | 場所 |
|---|---|---|
| `WRN_UnreachableCode` (CS0162) | 到達不能コード | `CheckReachable`, `:264-275` |
| `ERR_LabelNotFound` (CS0159) | 解決されない goto | `:219` |
| `ERR_BadDelegateLeave` (CS1632) | ラムダから外へ break/continue/goto | `:232` |
| `WRN_UnreferencedLabel` (CS0164) | 未使用ラベル | `:142` |
| `ERR_SwitchFallThrough`/`ERR_SwitchFallOut` (CS0163/CS8070) | switch セクションの落下 | `:399` |

`ControlFlowPass.Analyze` の戻り値「終端が到達可能か」が `FlowAnalysisPass` の暗黙 return 挿入と、非 void メソッドの `ERR_ReturnExpected`（CS0161, `FlowAnalysisPass.cs:106`）判定に使われる。ラムダについては Binder 側でも直接呼ばれる（`Binder/Binder_Statements.cs:620`）。

`Reported` フラグは「到達不能文の連続ブロックの先頭で 1 回だけ警告」するための工夫。`LabelState` では `Reported = false` にリセットし、再走査パス間で抑止が持ち越されないようにしている（`:112-118`）。

---

## 4. 変数スロットモデル `LocalDataFlowPass`

`LocalDataFlowPass.cs`, `DefiniteAssignment.VariableIdentifier.cs`。確定代入と Nullable 解析が共有する「変数 → 整数スロット」の写像。

```csharp
internal readonly struct VariableIdentifier {
    public readonly Symbol Symbol;        // Local / Parameter / Field / Property / Event / LocalFunction
    public readonly int ContainingSlot;   // 0 = ルート。>0 なら親（構造体変数など）のスロット
}
```

- **スロット 0 は予約**：確定代入では「到達不能フラグ」（`Assigned[0] == true` ⇔ 到達不能。`DefiniteAssignment.cs:2979-2985`）。
- 構造体のフィールドは **(フィールドシンボル, 親スロット)** の組で別スロットを持つ。よって `s.a.b` のような入れ子も追跡でき、同じフィールドシンボルでも親が違えば別スロット。
- `GetOrCreateSlot`（`LocalDataFlowPass.cs:100-145`）：
  - 範囲変数は追跡しない、**空構造体（データメンバを持たない構造体）は -1**（追跡不要。`EmptyStructTypeCache`）。
  - 新しいスロットを作った後 `Normalize(ref State)` で既存の状態ベクトルを拡張（新変数の初期値を決める）。
  - タプルの 8 要素目以降（`Rest` フィールド）は `DescendThroughTupleRestFields` で実フィールドまで降りる。
- `MakeSlot(BoundExpression)`（`:200-230`）：`this`/`base`/ローカル/パラメータ/フィールド/プロパティ/イベントアクセスからスロットを求める。レシーバが追跡されていないメンバは -1。

`EmptyStructTypeCache`（272 行）は「構造体が（再帰的に）インスタンスフィールドを持つか」をキャッシュ。参照アセンブリ由来の構造体の private フィールドの扱いの差異が、後述の strict / compat 二重解析の理由の一つ。

---

## 5. 確定代入解析 `DefiniteAssignmentPass`

`DefiniteAssignment.cs`（2,988 行）＋ `DefiniteAssignment.LocalFunctions.cs`。C# 仕様 §9.4「確定代入」の実装。

### 5.1 状態と格子

```csharp
internal struct LocalState : ILocalDataFlowState {
    internal BitVector Assigned;          // bit i = スロット i が確定代入済み（bit 0 は到達不能フラグ）
    public bool NormalizeToBottom { get; }// 新規変数を「代入済み」で初期化するか（ローカル関数要約用）
}
```
（`DefiniteAssignment.cs:2937-2990`）

| 演算 | 実装 | 意味 |
|---|---|---|
| `Join` | 両方到達可能なら `Assigned.IntersectWith(other)`（積集合）。片方が到達不能ならもう片方を採用 | 全経路で代入済みのものだけが代入済み（`:2910-2932`） |
| `Meet` | 他方で代入済みのスロットを `SetSlotAssigned`（和集合）。他方が到達不能なら到達不能 | finally の代入を反映（`:2884-2908`） |
| `TopState` | 空集合＋（外側メソッドの非 out 引数・`this` などは代入済み） | `:1763-1840` |
| `UnreachableState` | 現状態の複製で bit 0 を立てる | 到達不能では全変数を代入済みとみなす（`:1943-1949`）→ 到達不能コードで誤検出しない |
| `ReachableBottomState` | 全ビット 1、bit 0 のみ 0 | `:1843-1848` |

### 5.2 構造体フィールドの階層的代入

`SetSlotAssigned(slot, ref state)`（`:1686-1716`）:
1. 自スロットを代入済みに。
2. 構造体なら **全子フィールドを再帰的に代入済み**に。
3. **親方向に遡り**、親構造体の全フィールドが代入済み（`FieldsAllSet`）になったら親も代入済みに。

これにより
```csharp
S s; s.x = 1; s.y = 2; Use(s);  // s 全体が代入済みとみなされる
```
が正しく扱われる。インライン配列（`[InlineArray]`, 長さ > 1）はフィールドだけでは完全初期化を判定できないので常に `false`（`FieldsAllSet`, `:1648-1670`）。

`SetSlotUnassigned` は逆に子孫と祖先を未代入にする（代入解除を追跡する解析用）。`NonMonotonicState` があればそちらにも反映（`:1751-1761`）。

### 5.3 読み取り検査と診断

- 変数の読み取り → `CheckAssigned(symbol, node)`（`:1113-1130`）→ 未代入なら `ReportUnassigned`（`:1149-1330`）。
- `_alreadyReported` ビットベクトルで同一変数の重複報告を抑止。
- フィールドアクセス式は `IsAssigned(expr, out unassignedSlot)`（`:1340-1440`）でレシーバ連鎖を辿る。

| 診断 | 内容 |
|---|---|
| `ERR_UseDefViolation` (CS0165) | 未代入ローカルの使用 |
| `ERR_UseDefViolationOut` (CS0269) | 未代入 out 引数の使用 |
| `ERR_ParamUnassigned` (CS0177) | out 引数を代入せずに return |
| `ERR_UseDefViolationField` / `...Property` / `ERR_UnassignedThis...` | 構造体コンストラクタでの未代入フィールド/`this` |
| `WRN_UnreferencedVar` (CS0168) / `WRN_UnreferencedVarAssg` (CS0219) | 未使用変数／代入のみで未使用（`ReportIfUnused`, `:2287-2300`） |
| `ERR_UnsafeUnassignedAddressOf` | 未代入変数のアドレス取得 |

また `_sourceAssembly.NoteFieldAccess` でフィールドの読み書きをアセンブリに記録し、未使用フィールド警告（CS0169/CS0414/CS0649）の材料にしている（`:733-961`）。

### 5.4 構造体コンストラクタと「自動既定構造体」(C# 11)

コンストラクタ終了時（`LeaveParameters` 経由）に `this` の各フィールドを検査（`:430-500`）。言語バージョンが C# 11 以上なら、エラーの代わりに **未代入フィールドのリスト `implicitlyInitializedFieldsOpt` を返し**、`FlowAnalysisPass.PrependImplicitInitializations` がコンストラクタ先頭に `this.f = default;` を挿入する。古い言語バージョンでは `...UnsupportedVersion` 系エラー。

### 5.5 strict / compat の二重解析（`Analyze`, `:507-600`）

```text
strict 解析（参照アセンブリ由来構造体の private フィールドも追跡）
  ├─ エラー無し → そのまま報告して終了
  └─ エラー有り → compat 解析（従来互換。インポート構造体を空とみなす）も実行
        ├─ 件数が同じ → strict の結果を報告
        └─ strict にしか無いエラー → 警告に格下げ（ERR_UseDefViolation → WRN_UseDefViolation 等）
```

歴史的に旧コンパイラが見逃していたケースを、互換性を壊さず警告として表面化させるための仕組み。`ERR_InsufficientStack` が compat 側で起きた場合は compat を優先する、といったフォールバックもある。

### 5.6 その他の転送関数の要点

- 宣言 `int x;` → スロット生成・未代入。到達不能地点の宣言は代入済み扱い（`AssignImpl`, `:1495-1530`）。
- パターン変数（`is int n`）は `VisitPattern` で **`StateWhenTrue` 側だけ** 代入済み（`:1953-1960`）。`if (o is int n) ... else ...` の else で `n` を使うと CS0165。
- `out var` 引数・`ref` 引数は `WriteArgument` で代入扱い（`AbstractFlowPass.cs:1589`）。
- `[DoesNotReturn]` 等の Nullable 属性は確定代入には影響しない（Nullable 側で扱う）。

---

## 6. 解析間の関係（クラス階層）

```text
BoundTreeVisitor
└─ AbstractFlowPass<TLocalState, TLocalFunctionState>
   ├─ ControlFlowPass                          … 到達可能性
   │   └─ AbstractRegionControlFlowPass
   │       ├─ EntryPointsWalker / ExitPointsWalker / RegionReachableWalker
   └─ LocalDataFlowPass<,>                     … スロット管理
       ├─ DefiniteAssignmentPass               … 確定代入
       │   ├─ UnassignedVariablesWalker / UnassignedAddressTakenVariablesWalker
       │   ├─ VariablesDeclaredWalker
       │   └─ AbstractRegionDataFlowPass
       │       ├─ DataFlowsInWalker / DataFlowsOutWalker
       │       ├─ DefinitelyAssignedWalker / AlwaysAssignedWalker
       │       └─ ReadWriteWalker
       └─ NullableWalker                       … Null 許容フロー解析
```

---

## 7. 領域データフロー解析（IDE / Analyzer API）

`SemanticModel.AnalyzeDataFlow(...)` は `CSharpDataFlowAnalysis`（`CSharpDataFlowAnalysis.cs`）を返す。各プロパティは **遅延評価** で、初回アクセス時に対応する walker を走らせてキャッシュする。「メソッドの抽出」リファクタリング等の基盤。

`RegionAnalysisContext`（`RegionAnalysisContext.cs`）が (Compilation, Member, 束縛ノード, 領域の先頭/末尾ノード) を保持し、先頭 > 末尾や型式のみの領域などは `Failed` とする。

| プロパティ | walker | アルゴリズムの要点 |
|---|---|---|
| `VariablesDeclared` | `VariablesDeclaredWalker` | 領域内の宣言を収集 |
| `DataFlowsIn` | `DataFlowsInWalker` | **領域入口で状態を `TopState`（全未代入）にリセット**し、領域内で「未代入使用」として報告された変数＝外から値が流れ込む変数。外から領域内ラベルへの goto の状態もリセット（`NoteBranch`） |
| `DataFlowsOut` | `DataFlowsOutWalker` | **領域内の代入を「代入しなかった」ことにして**（`AssignImpl` で `written = false`）解析し、領域外で未代入エラーになる変数＝値が外へ流れ出る変数。`trackUnassignments: true`（非単調）。ループ対策として、DataFlowsIn の変数が入口で未代入なら追加。領域外の ref/out 引数への到達可能な代入は例外経路でも見えるので常に含める |
| `DefinitelyAssignedOnEntry/OnExit` | `DefinitelyAssignedWalker` | 領域の入口／出口での確定代入集合 |
| `AlwaysAssigned` | `AlwaysAssignedWalker` | 領域内で必ず代入される変数 |
| `ReadInside/WrittenInside/ReadOutside/WrittenOutside` | `ReadWriteWalker` | `NoteRead`/`NoteWrite` を `regionPlace` で分類 |
| `Captured/CapturedInside/CapturedOutside` | （ReadWrite と同時） | ラムダ/ローカル関数に捕捉される変数 |
| `UnsafeAddressTaken` | `UnassignedAddressTakenVariablesWalker` 等 | `&x` された変数 |
| `UsedLocalFunctions` | | 領域内で使われたローカル関数 |

`AbstractRegionDataFlowPass`（`AbstractRegionDataFlowPass.cs`）は全パラメータ・`this`・拡張パラメータ・ラムダ/ローカル関数パラメータに事前にスロットを作り（領域外で宣言された変数も追跡するため）、`nameof` の中身は解析しない。

`AnalyzeControlFlow(...)` は `ControlFlowAnalysis`（`ControlFlowAnalysis.cs`）を返し、`EntryPoints`（外から領域内への goto 先ラベル）、`ExitPoints`（領域から外へ出る break/continue/goto/return）、`StartPointIsReachable`/`EndPointIsReachable`（`RegionReachableWalker`）、`ReturnStatements` を提供する。

---

## 8. Null 許容フロー解析 `NullableWalker`

`NullableWalker.cs`（14,527 行）＋ `NullableWalker_Patterns.cs`（1,275）, `.Variables.cs`（436）, `.SnapshotManager.cs`（276）, `.DebugVerifier.cs`（464）, `.PlaceholderLocal.cs`（76）。
`internal sealed partial class NullableWalker : LocalDataFlowPass<NullableWalker.LocalState, NullableWalker.LocalFunctionState>`（`:26-27`）。確定代入と同じ基盤の上に乗っているが、**型推論・変換・属性解釈までやり直す「第 2 の意味解析器」**とでも言うべき規模を持つ。

### 8.1 格子と状態表現

フロー状態 `NullableFlowState`（`Symbols/NullableFlowState.cs`）は 3 値:

| 値 | ビット値 | 意味 |
|---|---|---|
| `NotNull` | `0b00` | null でない |
| `MaybeNull` | `0b01` | null かもしれない（`T?` の null） |
| `MaybeDefault` | `0b11` | `default(T)` かもしれない（制約なし型パラメータ `T` 用） |

`Join` = 大きい方（より弱い情報）、`Meet` = 小さい方（`NullableFlowStateExtensions.cs:21,27`）。

`LocalState`（`NullableWalker.cs:14059-14460`）:

```csharp
internal struct LocalState : ILocalDataFlowState {
    internal readonly int Id;            // Variables.Id（メソッド/ラムダのスコープ ID）
    private readonly Boxed? _container;  // 外側スコープの状態（ラムダ/ローカル関数用）
    private BitVector _state;            // スロットあたり 2 ビット。bit 0 = 到達可能
    public bool Reachable => _state[0];  // :14097
}
```

- 実際のビット符号化は `GetValue`/`SetValue`（`:14287`, `:14316`）: `(true,true)=NotNull`, `(true,false)=MaybeNull`, `(false,true)=MaybeDefault`。
  ⚠ `:14074` のコメントは別の符号化を記述しており、**コードと食い違っている**（コメントが古い）。
- **確定代入とは到達可能ビットの極性が逆**（DA は bit0=1 が到達不能、Nullable は bit0=1 が到達可能）。
- 到達不能状態は 1 つだけ：`GetValue` は全スロット `NotNull` を返し、`SetValue` は無視する（`:14308`）。到達不能コードで警告が出ないのはこのため。
- 新しいスロットの初期値は宣言型・注釈から `GetDefaultState`（`:2076`）で決まる（`string?` → MaybeNull、`string` → NotNull など）。
- `TopState()`（`:3021`）= 全スロット宣言型由来の既定状態、`ReachableBottomState()`（`:3033`）= 全スロット NotNull。

### 8.2 スロットモデル `Variables`（`NullableWalker.Variables.cs`）

確定代入の平坦なスロット表と異なり、**スコープ階層を持つ**:

- スロット番号 = `(id << 16) | index`（`ConstructSlot :413` / `DeconstructSlot :423`）。ラムダ/ローカル関数ごとに子 `Variables` を作り新しい ID を割り当てる（`CreateNestedMethodScope :213`）。`LocalState` は ID が違うスロットを `_container` に委譲するので、**ラムダ内の状態が外側の状態を包含**できる。
- `VariableIdentifier(symbol, ContainingSlot)` で `a.b.c` のような**メンバパスを追跡**（クラス型のフィールド/プロパティも追跡対象。DA は構造体フィールドのみ）。
- **`MaxSlotDepth = 5`**（`:84`）：メンバ連鎖の深さ 5 以上、またはインデックス上限を超えるとスロットを作らない（-1）。`class Node { Node? Next; }` で `a.Next.Next.Next...` が無限に増えるのを防ぐ。
- `_variableTypes`（`:128`）に `var` ローカル等の**推論された null 許容型**を保持。
- `PlaceholderLocal`：生成中オブジェクト（`new C { P = ... }`）やパターン入力に仮のスロットを与えるための合成ローカル。
- `MakeSlot`（`:2174`）は `?.` の受け側（`BoundConditionalReceiver`）も `_lastConditionalAccessSlot` 経由で解決。代入時は `InheritNullableStateOfMember`（`:2825`）でメンバの状態をコピーする（`a = b;` の後 `a.P` は `b.P` の状態を引き継ぐ）。

### 8.3 エントリポイントと呼び出し元

| メソッド | 行 | 用途 |
|---|---|---|
| `AnalyzeIfNeeded(compilation, method, ...)` | 1500 | コンパイル時診断。言語バージョンと `#nullable` 文脈で有効な場合のみ実行 |
| `AnalyzeAndRewrite` | 1653 | SemanticModel 用。解析＋ null 許容情報を埋め込んだ Bound Tree へ書き換え |
| `AnalyzeAndRewriteSpeculation` | 1729 | 投機的 SemanticModel。スナップショットから途中状態を復元 |
| `AnalyzeWithoutRewrite` | 1638 | nullable 無効時の検証用 |
| `Analyze(compilation, BoundLambda, ...)` | 1850 | ラムダ戻り値型推論 |
| `AnalyzeIfNeeded(Binder, ...)` | 1809 | 属性引数・既定値・`field` 等の単発式 |

呼び出し元: `MethodCompiler.cs:1966`（`AnalyzeAndRewrite`, `createSnapshots: true`）, `:1981`, `:646`, `:1027`, `:2079`、各 `*SemanticModel.cs`、`BoundTree/UnboundLambda.cs:157` など。

**IDE 連携**：解析中 `SetAnalyzedNullability`（`:388`）で各式の結果を `_analyzedNullabilityMapOpt` に記録し、`NullabilityRewriter`（`BoundTree/NullabilityRewriter.cs`）がそれを Bound Tree に反映。`SnapshotManager` は文の位置ごとに状態スナップショット（`TakeIncrementalSnapshot`, `:208`）を保存し、任意位置の投機的解析を途中から再開できる（ループ再走査で後の安定した状態が上書き）。`GetTypeInfo(...).Nullability` の値はここから来る。

### 8.4 式の結果と条件状態

```csharp
readonly struct VisitResult {           // :116-140
    TypeWithState RValueType;           // 値の型＋フロー状態（読む側）
    TypeWithAnnotations LValueType;     // 代入可能な型＋注釈（書く側）
    Optional<LocalState> StateForLambda;// ラムダ式を見た時点の状態
    VisitResult[]? NestedVisitResults;
}
```
直前に訪問した式の結果は `_visitResult`（`:303`）に置かれる。**注釈（annotation, 宣言の `?`）と状態（state, フロー上の現在値）を区別**するのが Nullable 設計の中核。

null テストからの学習:

| 構文 | 処理 |
|---|---|
| `x == null` / `x != null` | 二項演算子解析（`:5880-5915`）で `SplitAndLearnFromNonNullTest`（`:5946`）/ `LearnFromNullTest`（`:6100`）。`a?.b?.c == null` なら a, b, c 全てに学習 |
| `x ?? y`, `x ??= y` | `VisitNullCoalescingOperator`（`:6249`）/ `...AssignmentOperator`（`:6212`） |
| `x?.M()` | `VisitConditionalAccess`（`:6416`）。null 側の状態を保存し、アクセス中は受け側を NotNull に |
| `x!` | 専用 Visit は無く、`VisitExpressionWithoutStackGuardEpilogue`（`:3828-3846`）で `IsSuppressed` なら結果を NotNull に強制。各診断箇所も `IsSuppressed` を確認 |
| `is` / `switch` | 決定 DAG を歩く（§8.6） |

**属性（`FlowAnalysisAnnotations`）の解釈**:

| 属性 | 処理 |
|---|---|
| `[AllowNull]`/`[DisallowNull]`（事前条件） | `VisitArgumentConversionAndInboundAssignmentsAndPreConditions`（`:8293`）, `CheckDisallowedNullAssignment`（`:8382`） |
| `[NotNull]`/`[MaybeNull]`/`[NotNullWhen]`/`[MaybeNullWhen]`（事後条件） | `LearnFromPostConditions`（`:8659`）。`When` 系は条件状態を分割 |
| 戻り値の `[MaybeNull]`/`[NotNull]` | `ApplyUnconditionalAnnotations`（`:7535`） |
| `[NotNullIfNotNull]` | `:7770` ほか |
| `[MemberNotNull(When)]` | 呼び出し側 `ApplyMemberPostConditions`（`:8011`）、メソッド内では終了時に強制 |
| `[DoesNotReturn]` | 呼び出し後 `SetUnreachable()`（`:7882`） |
| `[DoesNotReturnIf(b)]` | 片側の条件状態のみ残す（`:8242-8270`） |

`Equals`/`ReferenceEquals`/`Interlocked.CompareExchange` 等は `LearnFromEqualsMethod`（`:7185`）等で特別扱い。

### 8.5 型推論のやり直し・変換・ラムダ

- **ジェネリックメソッドの型引数を null 許容性込みで再推論**：`VisitCall`（`:6843`）→ `ReInferMethodAndVisitArguments`（`:7043`）→ `VisitArguments`（`:7615`）→ `InferMemberTypeArguments`（`:8786`）が `MethodTypeInferrer.Infer` を引数の null 許容型で再実行（`:8827`）。例えば `Identity(s)` の `T` は `s` が MaybeNull なら `string?` になる。その後 `CheckMethodConstraints`（`:9036`）で `notnull` 制約などを検査。演算子（`ReInferBinaryOperator :5765`）、Deconstruct、デリゲート生成も同様。
- **変換**：`VisitConversion`（ワーカー `:9900`）が全ての暗黙/明示変換で状態を伝搬・検査（アンボクシング `WRN_UnboxPossibleNull` 等）。
- **ラムダ**：`VisitLambda`（`:11230`）は状態を `StateForLambda` に保存するだけ。本体は **ターゲットデリゲート型が判明する変換時に解析**（`:9997-10008` → `AnalyzeLocalFunctionOrLambda :3491`）。ネストした `Variables`/`LocalState` を作り、捕捉変数は宣言時点の状態を引き継ぐ。ラムダはオーバーロード解決・戻り値推論・最終変換で複数回解析され得る（`UnboundLambda.cs:148-152` に指数的コストの注記）。
- **ローカル関数**（`:3412`）：まだ呼び出し地点が無ければ捕捉変数を NotNull、あれば呼び出し地点の状態を Join した `StartingState` から開始。
- **ターゲット型付き式**は `_targetTypedAnalysisCompletionOpt`（`:267`）に「後で型が分かったら完了する」関数を保存する遅延評価。

### 8.6 パターン / switch（`NullableWalker_Patterns.cs`）

1. `LearnFromAnyNullPatterns`（`:25`）：パターン中に `null` 定数パターンがあれば、入力を「null かもしれない」とみなす（`if (s is null)` と書いた時点で開発者は null の可能性を想定している、という学習）。
2. `LearnFromDecisionDag`（`:377`）：**バインダが生成した決定 DAG のノードをトポロジカル順に**訪問（`:423`）し、ノードごとの状態 `nodeStateMap`（`:418`）を `Join` で合成:
   - 評価ノード（`:439`）：プロパティ/フィールド/Deconstruct/インデクサ評価の一時値にスロットを割当（元変数のスロットを再利用するので、学んだ事実が元の変数に戻る）。
   - テストノード（`:539`）：型テスト・非 null テストの真側で NotNull、明示 null テストの真側で MaybeNull／偽側で NotNull。
   - `when` 節ノード（`:620`）、リーフノード（`:615`）→ ラベルごとの状態へ。
3. `is` 式（`:1247`）は真/偽ラベルの状態から条件状態を作り、switch 文/式は各セクション/アームの状態として使う。switch 式で null が網羅されていなければ `WRN_SwitchExpressionNotExhaustiveForNull`（`:1096`）。

### 8.7 メソッド終了時の検査（`Scan`, `:632-680`）

`base.Scan` の後、本体末尾と全 `return` の保留分岐について:
- `EnforceDoesNotReturn` → `WRN_ShouldNotReturn`
- `enforceMemberNotNull`（ローカル関数）→ コンストラクタ終了時の未初期化非 null フィールド `WRN_UninitializedNonNullableField`（CS8618, `:850`）、`[MemberNotNull]` 違反 `WRN_MemberNotNull`
- `EnforceParameterNotNullOnExit`（`:1351`）→ `[NotNull]` 引数
- `EnforceNotNullWhenForPendingReturn`（`:1308`）→ `return true/false` や保留分岐の条件状態から `[NotNullWhen]` を検査

`Scan` 冒頭で `Diagnostics.Clear()`（`:638`）するので、**不動点に達した最終パスの診断だけが残る**。

### 8.8 主な診断

| 診断 | 例 | 場所 |
|---|---|---|
| `WRN_NullReferenceReceiver` (CS8602) | null かもしれない参照の逆参照 | `CheckPossibleNullReceiver` `:13559` |
| `WRN_NullAsNonNullable` / `WRN_NullReferenceAssignment` / `...Return` / `...Argument` (CS8625/8601/8603/8604) | null を非 null へ | `ReportNullableAssignmentIfNecessary` `:2492` |
| `WRN_NullabilityMismatchIn*` (CS8619/8620 等) | 型引数レベルの不一致 | `:2701`, `:12167` |
| `WRN_UninitializedNonNullableField` (CS8618) | コンストラクタ終了時 | `:850` |
| `WRN_NullableValueTypeMayBeNull` (CS8629) | `int?` の `.Value` | `:10182` |
| `WRN_ThrowPossibleNull` (CS8597) | `throw null` 可能性 | `:13755` |

全ての nullable 警告は `ReportDiagnostic`（`:2773`）を通り、**到達可能かつ `_disableDiagnostics == false` の時だけ**報告される。引数の 2 回目の訪問などでは `_disableDiagnostics` を立てて重複報告を防ぐ。

### 8.9 安全・性能機構

- `MaxSlotDepth`、到達不能状態の単一化、二項演算子連鎖の反復処理（`:5476`）。
- 深い木では `CancelledByStackGuardException` を捕捉してエラー診断化（`:564`, `:1970`）。
- `DebugVerifier`（DEBUG ビルドのみ）：全 `BoundExpression` が解析済み null 許容マップに登録されているか検証し、SemanticModel の情報欠落を防ぐ（`:44-63`）。
- ループ・goto・ローカル関数は基底の `stateChangedAfterUse` 再走査で不動点へ。格子が有限（3 値）かつ Join が単調なので停止する。

---

## 9. 設計上の特徴と注意点まとめ

1. **Bound Tree 直接走査 + 条件状態**：CFG を作らない代わりに、条件状態（真/偽の 2 状態）と PendingBranch キューで制御フローを表現。構造化された C# の文に対しては単純かつ高速。CFG が必要な用途（アナライザー向け）には別途 `Microsoft.CodeAnalysis.FlowAnalysis.ControlFlowGraph`（IOperation ベース）が存在し、本稿のコンパイラ内部解析とは独立している点に注意。
2. **全体再走査による不動点**：後方分岐はワークリストでなく `stateChangedAfterUse` による全体再走査。単純だが、ループの多重ネストや Nullable のように高コストな walker では解析時間に影響する。
3. **保守的な例外モデル**：try 内のどこで例外が起きても「try 開始時点の状態」で catch/finally に入るとみなす。これは仕様準拠かつ健全だが、try 内で代入した変数を catch で使うと CS0165 になる理由でもある。
4. **一つの基盤、多数の解析**：格子の型と `Join`/`Meet`、`NoteRead`/`NoteWrite`/`ReportUnassigned`/`EnterRegion` 等のフックを差し替えるだけで、コンパイラ診断と IDE の領域解析を同じエンジンで実装している（DataFlowsIn/Out は「確定代入解析を細工して再利用する」という巧妙な設計）。
5. **互換性への配慮**：strict/compat の二重解析、C# 11 での構造体自動既定化、言語バージョンごとの診断切替など、既存コードを壊さないための分岐が随所にある。
6. **スタック深度対策**：`&&`/`||` 連鎖・`else if` 連鎖の明示スタック化、`_recursionDepth`、`InsufficientExecutionStack` → `ERR_InsufficientStack`。

---

## 付録 A: 主要ファイル早見表

| ファイル | 行数 | 役割 |
|---|---:|---|
| `AbstractFlowPass.cs` | 3,903 | 共通走査エンジン（文・式の転送関数の既定実装） |
| `AbstractFlowPass.ILocalState.cs` | 78 | 格子演算の抽象定義 |
| `AbstractFlowPass.PendingBranchesCollection.cs` | 153 | ラベル別の保留分岐コレクション |
| `AbstractFlowPass_LocalFunctions.cs` | 197 | ローカル関数の要約と不動点 |
| `AbstractFlowPass_Switch.cs` | 185 | switch 文/式 |
| `ControlFlowPass.cs` | 437 | 到達可能性 |
| `LocalDataFlowPass.cs` | 271 | 変数スロット管理 |
| `DefiniteAssignment.cs` | 2,988 | 確定代入 |
| `DefiniteAssignment.LocalFunctions.cs` | 218 | ローカル関数の読み取り再生 |
| `EmptyStructTypeCache.cs` | 272 | 空構造体判定キャッシュ |
| `FlowAnalysisPass.cs` | 223 | パイプラインへの組込み・暗黙 return/初期化挿入 |
| `CSharpDataFlowAnalysis.cs` / `ControlFlowAnalysis.cs` | 400 / 152 | 公開 API 実装 |
| `*Walker.cs`（領域解析） | 各 50～330 | §7 |
| `NullableWalker*.cs` | 約 17,000 | §8 |

## 付録 B: 例で見る状態遷移（確定代入）

```csharp
int x;                       // slot1 生成, Assigned = {}           （bit0=0: 到達可能）
if (b) {                     // VisitCondition(b) → 条件状態 T={}, F={}
    x = 1;                   //   T 側: {x}
} else {
    return;                  //   F 側: PendingBranch(return, {}) を登録 → 到達不能 (bit0=1)
}
                             // Join({x}, 到達不能) = {x}
Console.WriteLine(x);        // CheckAssigned(x): OK
```

```csharp
int y;
while (cond) {               // LoopHead: 先頭状態 {} を保存
    y = 0;                   //   {y}
}                            // LoopTail: Join({}, {y}) = {} → 変化なし → 再走査不要
Console.WriteLine(y);        // 出口 = StateWhenFalse(cond) = {} → CS0165
```

```csharp
int z;
try { z = 1; }               // initialState = {}
catch { throw; }             // catch は {} から開始 → throw で到達不能
finally { }
Console.WriteLine(z);        // 終端 = Join(try終端 {z}, catch終端 ⊥) = {z} → OK
```
