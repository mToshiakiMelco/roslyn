# Roslyn IOperation と制御フローグラフ（CFG）詳細解説
## ― アナライザー向けの言語共通・公開の意味モデル：`OperationInterfaces.xml` の自動生成から CFG 構築まで ―

対象:
- `src/Compilers/Core/Portable/Operations/`（`OperationInterfaces.xml`、`Operation.cs`、`ControlFlowGraphBuilder*.cs`、`ControlFlowGraph.cs` ほか）
- `src/Compilers/Core/Portable/Generated/`（`Operations.Generated.cs`、`OperationKind.Generated.cs`、`FlowAnalysis.Generated.cs`）
- `src/Tools/CompilerGeneratorTools/Source/IOperationGenerator/`（生成器）
- `src/Compilers/CSharp/Portable/Operations/CSharpOperationFactory*.cs`（BoundTree → IOperation）
- `src/Compilers/Core/Portable/DiagnosticAnalyzer/`（アナライザーとの統合）

基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは `src/Compilers/` からの相対パス（生成器は `src/Tools/…`）。

関連ドキュメント：`csharp-boundtree-boundnodes.ja.md`（変換元の BoundTree）、`csharp-compiler-dataflow-analysis.ja.md`（内部のフロー解析との対比）、`csharp-compilation-semanticmodel-driver.ja.md`（アナライザードライバ）

---

## 0. 要約

- **IOperation** は、アナライザーやリファクタリング向けに公開されている「意味の木」。内部の BoundTree（internal、言語ごと、リリースごとに変わる、Lowering で消える）を、**公開・安定・言語共通（C# と VB で同じ型）**の形にしたもの。
- ノードの定義は `OperationInterfaces.xml`（134 の具象ノード、8 の抽象ノード）で、`IOperationGenerator` が次のものを生成する。
  - 公開インターフェイス（`IInvocationOperation` など）
  - 実装クラス
  - `OperationKind` 列挙
  - Visitor
  - Cloner
  - 子の列挙の状態機械
- C# では `CSharpOperationFactory` が**初期束縛（Lowering 前、null 許容性の書き換え済み）の BoundTree** から作る。`SemanticModel.GetOperation` は**メンバーごとに 1 回だけ**木全体を作ってキャッシュする。
- **ControlFlowGraph（CFG）** は、IOperation の木を基本ブロックと領域（try、catch、ローカル変数の寿命など）のグラフに**低水準化**したもの。`if`/ループ/`using`/`lock`/`?.`/`??`/`switch` を分岐に分解し、評価途中の値は「フローキャプチャ」で表す。内部のフロー解析（グラフを作らない方式）とは別の、公開の仕組み。
- アナライザーは `RegisterOperationAction` などで IOperation を受け取り、`GetControlFlowGraph()` で CFG を得る。CFG はブロックの解析中だけキャッシュされる。

---

## 1. IOperation のモデルと自動生成

### 1.1 `OperationInterfaces.xml`（`Core/Portable/Operations/OperationInterfaces.xml`、3,776 行）

- `<Tree Root="IOperation">`（`:3`）。冒頭のコメント（`:5-21`）：
  - **ノードの並び順が `OperationKind` の値を決める**ので、新しいノードは末尾に追加する。
  - 再生成は `eng/generate-compiler-code.cmd`。
  - テストは `ROSLYN_TEST_IOPERATION=true`。
- `<UnusedOperationKinds>`（`:23-27`）で欠番（0x1D、0x62、0x64）を予約する。
- 要素：`<Node>` 134 個、`<AbstractNode>` 8 個。抽象ノードは `ILoopOperation`、`IMemberReferenceOperation`、`IAssignmentOperation`、`ISymbolInitializerOperation`、`ICaseClauseOperation`、`IInterpolatedStringContentOperation`、`IPatternOperation`、`IMethodBodyBaseOperation`。

```xml
<Node Name="IInvocationOperation" Base="IOperation" ChildrenOrder="Instance,Arguments" HasType="true">   <!-- :699 -->
  <Property Name="TargetMethod" Type="IMethodSymbol"> <Comments>…</Comments> </Property>
  <Property Name="Instance" Type="IOperation?"/>
  <Property Name="Arguments" Type="ImmutableArray&lt;IArgumentOperation&gt;"/>
  ...
</Node>
```

主な属性（モデルは `IOperationGenerator/Model.cs`）：

| 属性 | 意味 |
|---|---|
| `ChildrenOrder` | 子の列挙順（**ソース上の評価順**）。IOperation 型のプロパティが 2 つ以上あるノードでは必須 |
| `HasType` / `HasConstantValue` | `Type` / `ConstantValue` を保持するか |
| `Namespace` | 生成先のファイル。`FlowAnalysis` なら CFG 専用のノード（§3.1） |
| `Internal` | 内部用のノードまたはプロパティ（例：`IForEachLoopOperation.Info`、`ILockOperation.LockTakenSymbol`。CFG の構築にだけ使うデータ） |
| `SkipClassGeneration` | 実装クラスを手書きする（`IInvalidOperation`、dynamic 系など。`Operations/OperationNodes.cs`） |
| `VisitorName`、`SkipInVisitor`、`SkipInCloner`、`SkipChildrenGeneration` | 生成の調整 |
| `<OperationKind Include="false">` | 具象型が抽象親の種類を共有する（例：各ループは全て `OperationKind.Loop`） |

`BoundNodes.xml`（BoundTree 編）や `Syntax.xml`（構文編）と同じく、**XML の宣言から大量の定型コードを生成する**方式。

### 1.2 生成器（`src/Tools/CompilerGeneratorTools/Source/IOperationGenerator/`）

- `Program.cs`：`XmlSerializer` で読み、`IOperationClassWriter.Write`。`eng/generate-compiler-code.cs:59` から呼ばれ、**生成物はコミットする**。
- `IOperationClassWriter.Verifier.cs`：
  - 名前は `I…Operation` でなければならない。
  - 全ての型とプロパティに `<summary>` が必要。
  - IOperation 型のプロパティが 2 つ以上なら `ChildrenOrder` が完全かつ一貫していなければならない（`:102-135`）。
- `IOperationClassWriter.cs`（1,263 行）が出力するもの：

  | 出力 | 生成メソッド |
  |---|---|
  | 公開インターフェイス（ドキュメントコメント付き） | `WriteInterface`, `:205` |
  | 実装クラス：`abstract BaseXxxOperation` / `sealed XxxOperation` | `WriteClasses`, `:400` |
  | 実装クラスのコンストラクタ（子ごとに `SetParentOperation`） | 同上 |
  | 子の列挙の状態機械 `MoveNext(previousSlot, previousIndex)` / `MoveNextReversed` | `:712-856` |
  | `OperationKind` 列挙 | `WriteOperationKind`, `:305` |
  | `OperationCloner` | `WriteCloner`, `:939` |
  | `OperationVisitor` / `OperationVisitor<TArgument, TResult>` | `WriteVisitors`, `:1026` |

### 1.3 生成物

| ファイル | 規模 | 内容 |
|---|---|---|
| `Core/Portable/Generated/Operations.Generated.cs` | 11,766 行 | インターフェイス（`:17-`）、実装（`:4036-`）、Cloner（`:10854-`）、Visitor（`:11484-`） |
| `Generated/FlowAnalysis.Generated.cs` | 148 行 | CFG 専用の 6 インターフェイス |
| `Generated/OperationKind.Generated.cs` | 292 行 | `public enum OperationKind`（`None = 0` 〜 `0x81`。互換用の別名あり） |

```csharp
public interface IInvocationOperation : IOperation {                     // Operations.Generated.cs:771
    IMethodSymbol TargetMethod { get; }  ITypeSymbol? ConstrainedToType { get; }
    IOperation? Instance { get; }  bool IsVirtual { get; }  ImmutableArray<IArgumentOperation> Arguments { get; } }

internal sealed partial class InvocationOperation : Operation, IInvocationOperation {   // :5190
    internal InvocationOperation(..., IOperation? instance, ..., ImmutableArray<IArgumentOperation> arguments,
                                 SemanticModel? semanticModel, SyntaxNode syntax, ITypeSymbol? type, bool isImplicit)
        : base(semanticModel, syntax, isImplicit)
    { Instance = SetParentOperation(instance, this); Arguments = SetParentOperation(arguments, this); ... }
    internal override (bool hasNext, int nextSlot, int nextIndex) MoveNext(int previousSlot, int previousIndex) {
        switch (previousSlot) {
            case -1: if (Instance != null) return (true, 0, 0); else goto case 0;
            case 0:  if (!Arguments.IsEmpty) return (true, 1, 0); else goto case 1;
            case 1 when previousIndex + 1 < Arguments.Length: return (true, 1, previousIndex + 1);
            case 1: case 2: return (false, 2, 0); ... } }
    public override OperationKind Kind => OperationKind.Invocation;
}
```

**子の列挙**（`IOperation.OperationList`, `Operations/IOperation.OperationList.cs:24`）：`readonly struct` の列挙子が `(slot, index)` を持ち、生成された `MoveNext` に委譲する。**子の配列を作らずに**、評価順で（逆順も可）列挙できる。

### 1.4 `Operation` 基底クラス（`Operations/Operation.cs`）

```csharp
internal abstract partial class Operation : IOperation {                                 // :18
    protected Operation(SemanticModel? semanticModel, SyntaxNode syntax, bool isImplicit) // :27
    public IOperation? Parent { get; }                                                   // :48
    public Optional<object?> ConstantValue                                               // :94
    SemanticModel? IOperation.SemanticModel => _owningSemanticModelOpt?.ContainingPublicModelOrSelf;   // :123
    public static T? SetParentOperation<T>(T? operation, IOperation? parent)             // :148
}
```

- **`Parent` は遅延ではない**：木は下から上へ作られ、子を親のコンストラクタに渡した時点で `SetParentOperation` が親を設定する（`:137-145`）。根は `SetParentOperation(op, null)`（C# では `MemberSemanticModel.cs:1248`）。
- `IsImplicit`：コンパイラが合成したノード（ソースに直接対応しない）。**明示的なノードは構文ノードと 1 対 1 に対応する**。
- 手書きの `OperationWalker`（`Operations/OperationWalker.cs`）は、スタックガード付きで子を再帰的に歩く。

---

## 2. C#：BoundTree から IOperation へ

### 2.1 `CSharpOperationFactory`（`CSharp/Portable/Operations/CSharpOperationFactory.cs`、3,045 行）

```csharp
internal sealed partial class CSharpOperationFactory {
    public IOperation? Create(BoundNode? boundNode)        // :27  switch (boundNode.Kind) に 157 の case
}
```

| BoundNode | IOperation | 場所 |
|---|---|---|
| `BoundCall` | `InvocationOperation`。引数は `DeriveArguments` で**評価順の `IArgumentOperation`** にし、既定引数と params も明示する（`_Methods.cs:194-359`）。エラーのある呼び出しは `InvalidOperation` | `:447-464` |
| `BoundConversion` | `ConversionOperation`（公開の `CommonConversion` 構造体を持つ）。メソッドグループやラムダからデリゲートへの変換は `DelegateCreationOperation`。`as` は `isTryCast: true`。構文の無い変換やタプルの人工的な変換は省く | `:1030-1207` |
| `BoundLambda` / `UnboundLambda` | `AnonymousFunctionOperation`。未変換のラムダは `BindForErrorRecovery()` でバインドしてから変換し、同じラムダに 2 つのノードができないようにする | `:999-1009` |
| `BoundIsPatternExpression` | `IsPatternOperation` | `:2856` |
| `BoundBadExpression` | `InvalidOperation` | `:1495` |
| IOperation に対応する形が無いもの（`MethodGroup`、`Sequence`、`TypeExpression` …） | `NoneOperation`（子だけを持つ） | `:299-324` |

`IsImplicit` は、ほぼ `boundNode.WasCompilerGenerated`。ただし 2 つのノードが同じ構文ノードを主張しないよう、各所で調整している。

### 2.2 どの BoundTree を使うか

- **Lowering 前の初期束縛の木**を使う（`MemberSemanticModel` がキャッシュしているもの）。
- null 許容性の解析が有効なら、**`NullableWalker` で書き換えた木**（null 許容注釈付き）。`EnsureNullabilityAnalysisPerformedIfNecessary()`（`CSharp/Portable/Compilation/MemberSemanticModel.cs:1957`）がこれを準備する。

### 2.3 `SemanticModel.GetOperation` の仕組み

```text
SemanticModel.GetOperation(node)            Core/Portable/Compilation/SemanticModel.cs:76
→ CSharpSemanticModel.GetOperationWorker    :478-486
→ SyntaxTreeSemanticModel: メンバーのモデルを選ぶ    :169-199
→ MemberSemanticModel.GetOperationWorker    :1202-1228
```

```csharp
using (_nodeMapLock.DisposableRead()) { if (_guardedIOperationNodeMap.Count != 0) return guardedGetIOperation(); }
IOperation rootOperation = GetRootOperation();          // メンバー全体の BoundNode → IOperation（1 回だけ）
using var _ = _nodeMapLock.DisposableWrite();
if (_guardedIOperationNodeMap.Count != 0) return guardedGetIOperation();
OperationMapBuilder.AddToMap(rootOperation, _guardedIOperationNodeMap);
return guardedGetIOperation();
```

- **メンバーの木全体を 1 回だけ作り**、構文ノード → IOperation の辞書に入れる（`ReaderWriterLockSlim` で保護）。以降の `GetOperation` は辞書を引くだけなので、**同じモデルからは常に同じオブジェクト**が返る。
- `OperationMapBuilder`（`Core/Portable/Operations/OperationMapBuilder.cs`）は**明示的なノードだけ**を登録し、「1 つの構文ノードに明示的なノードは 1 つ」を assert する。二項演算や条件演算の深い連鎖は反復処理する。

---

## 3. 制御フローグラフ（CFG）

### 3.1 公開型（名前空間 `Microsoft.CodeAnalysis.FlowAnalysis`）

| 型 | 主なメンバー |
|---|---|
| `ControlFlowGraph`（`Operations/ControlFlowGraph.cs:24`） | `OriginalOperation`、`Blocks`、`Root`（最上位の領域）、`LocalFunctions`、`GetLocalFunctionControlFlowGraph`、`GetAnonymousFunctionControlFlowGraph` |
| `BasicBlock`（`BasicBlock.cs:17`） | `Kind`（Entry/Exit/Block）、`Operations`（文の列）、`BranchValue`（分岐の条件または戻り値）、`ConditionKind`（WhenTrue/WhenFalse）、`FallThroughSuccessor`、`ConditionalSuccessor`、`Predecessors`、`IsReachable`、`EnclosingRegion` |
| `ControlFlowBranch` | `Source`、`Destination`、`Semantics`（Regular/Return/Throw/Rethrow/StructuredExceptionHandling/ProgramTermination/Error）、出入りする領域と通過する finally |
| `ControlFlowRegion` | `Kind`、入れ子の領域、`Locals`、`LocalFunctions`、`CaptureIds`、最初と最後のブロック |
| `ControlFlowRegionKind` | `Root`、**`LocalLifetime`**（ローカル変数の寿命）、`Try`、`Catch`、`Filter`、`FilterAndHandler`、`TryAndCatch`、`Finally`、`TryAndFinally`、`StaticLocalInitializer`（VB）、`ErroneousBody` |
| CFG 専用のノード | `IFlowCaptureOperation`（値を一時的に捕捉）、`IFlowCaptureReferenceOperation`、`IIsNullOperation`、`ICaughtExceptionOperation`、`IStaticLocalInitializationSemaphoreOperation`、`IFlowAnonymousFunctionOperation` |

### 3.2 入口（`ControlFlowGraph.cs`）

```csharp
public static ControlFlowGraph? Create(SyntaxNode node, SemanticModel semanticModel, CancellationToken ct = default)   // :83
public static ControlFlowGraph Create(IBlockOperation body, …)            // :105
public static ControlFlowGraph Create(IMethodBodyOperation methodBody, …) // :165  ほか初期化子・属性・コンストラクタ本体
internal static ControlFlowGraph CreateCore(IOperation operation, …)      // :171
```

`CreateCore` は、根のノード（`Parent == null`）で、SemanticModel を持つことを要求してから `ControlFlowGraphBuilder.Create(operation)` を呼ぶ。**CFG のブロックの中のノードは、元の IOperation の木とは別の（書き換えられた）ノード**（`:196-199`）。

### 3.3 `ControlFlowGraphBuilder`（`Operations/ControlFlowGraphBuilder.cs`、8,142 行＋ partial ファイル群）

```csharp
internal sealed partial class ControlFlowGraphBuilder : OperationVisitor<int?, IOperation>     // :27
public static ControlFlowGraph Create(IOperation body, ControlFlowGraph? parent = null, ControlFlowRegion? enclosing = null,
                                      CaptureIdDispenser? captureIdDispenser = null, in Context context = default)   // :87
```

IOperation を**書き換えながら**、文を現在の基本ブロックへ追加していく Visitor。引数の `int?` は「結果を入れる捕捉 ID」。

`Create`（`:113-167`）の手順：

1. Root 領域に入り、Entry ブロックを置く。
2. `LocalLifetime` 領域に入る。
3. 本体を訪問する。
4. Exit ブロックを置く。
5. 未解決の分岐を検査する。
6. **`Pack`**（`:502`）で空のブロックや不要な領域を併合・簡約する。
7. 到達可能性に印を付ける。
8. ブロック → 後続（`ControlFlowBranch`）→ 先行の 3 段階で不変の結果を作る。

中核の仕組み：

- **`VisitStatement`**（`:1241`）→ `AddStatement`（`:1278`）：文の親を切り離し（`SetParentOperation(statement, null)`）、現在のブロックへ追加する。
- **評価スタックの退避**（`SpillEvalStack`, `:1822`）：式の途中に分岐が入る場合（`a + (b ?? c)` など）、先に評価済みの値を **`FlowCaptureOperation`** として捕捉する。後で `FlowCaptureReferenceOperation` で参照する。Lowering の `SpillSequenceSpiller`（Lowering 編 §3）と同じ発想を、公開 API の形で行っている。
- 同じ値を 2 回使う必要がある場合（持ち上げ演算子など）は `OperationCloner` で複製する。

### 3.4 高水準の構文の分解

| 構文 | CFG での形 | 場所 |
|---|---|---|
| `if` / `?:` | 文なら `GotoIfFalse cond alt; then; goto after; alt: else`。式なら両方の分岐を**同じ捕捉 ID** に入れ、`FlowCaptureReference` を返す | `VisitConditional` `:1544-1670` |
| `while` / `do` / `for` / `foreach` | 条件分岐とループ。`foreach` は列挙子の捕捉を `LocalLifetime` 領域に入れ、Dispose が必要なら `TryAndFinally` 領域を作る | `:3583`, `:5724`, `:4464` |
| `try/catch/finally` | `TryAndFinally` → `Try` と `TryAndCatch` → `Try`、catch ごとの `Catch`/`FilterAndHandler`、`Finally`。catch 変数には `CaughtExceptionOperation` を代入する | `VisitTry` `:3656-3822` |
| `using` / `await using` | `LocalLifetime` → 資源 → `TryAndFinally` → `Finally`（null 検査 ＋ `Dispose`/`DisposeAsync`） | `:3957-4131` |
| `lock` | `System.Threading.Lock` なら `EnterScope()` を使う `using` と同じ形。それ以外は `Monitor.Enter($lock, ref $lockTaken)` ＋ `TryAndFinally` | `VisitLock` `:4229-4413` |
| `??` / `??=` | 値を捕捉し、`IsNullOperation` で分岐する | `VisitCoalesce` `:3050` |
| `?.` | 受け手ごとに捕捉して null 判定し、共通の `whenNull` ブロックへ飛ぶ | `VisitConditionalAccess` `:3370` |
| `switch` | 値を 1 回だけ捕捉し、セクションごとに条件分岐の連鎖を作る | `VisitSwitch` `:5473` |
| `return` / `break` / `throw` | **文ではなく、ブロックの分岐の種類**（Return、Regular、Throw/Rethrow）になる | `:3837-3922` |

LocalRewriter（Lowering 編 §2）とよく似た分解をするが、**IL ではなく、解析しやすい公開のグラフ**を目的としている。そのため、領域（スコープ、例外ハンドラ）やソースとの対応を保つ。

### 3.5 ローカル関数とラムダ

- **本体はインライン化しない**：
  - ローカル関数は、それを含む領域に登録するだけ（`VisitLocalFunction`, `:6447`）。
  - ラムダは `FlowAnonymousFunctionOperation` に置き換える（`:6470`）。
- 子の CFG は**必要になったときに作り**、親の CFG がキャッシュする（`GetLocalFunctionControlFlowGraph`, `ControlFlowGraph.cs:244-318`。`Interlocked.CompareExchange`）。`CaptureIdDispenser` を共有して、親子の間で捕捉 ID を一意に保つ。
- `ControlFlowGraphExtensions.GetLocalFunctionControlFlowGraphInScope`：親の CFG をたどって、その関数を持つグラフを探す。

### 3.6 公開されているデータフロー解析

CFG の上で動く**公開のデータフロー解析器は `Compilers/` には無い**。公開されているのは次の 2 つ。

- グラフそのもの。
- 別系統の `SemanticModel.AnalyzeDataFlow`/`AnalyzeControlFlow`（フロー解析編 §7。BoundTree ベース）。

到達可能性の計算は、Workspaces 層の解析に複製されているとコメントにある（`ControlFlowGraphBuilder.cs:235`）。

---

## 4. アナライザーとの統合

### 4.1 登録と実行

- 登録：`RegisterOperationAction(action, params OperationKind[])`、`RegisterOperationBlockStartAction`、`RegisterOperationBlockAction`（`Core/Portable/DiagnosticAnalyzer/DiagnosticAnalysisContext.cs:164-199`）。
- `AnalyzerDriver`（`AnalyzerDriver.cs:2745-3062`）が、実行可能なコードを持つ宣言ごとに次のことを行う。
  1. `semanticModel.GetOperation(ブロックの構文)` でブロックの IOperation を得る（§2.3 と同じキャッシュ）。
  2. `DescendantsAndSelf()` で平坦化し、`OperationKind` ごとに登録されたアクションを呼ぶ。
  3. ブロックの開始・終了のアクションには、CFG を得るための関数を渡す。

### 4.2 CFG のキャッシュ（`AnalyzerExecutor.cs`）

```csharp
_lazyControlFlowGraphMap.GetOrAdd(operation, op => ControlFlowGraphBuilder.Create(op));   // :1708
```

- 1 つのブロックの解析中は、全てのアナライザーが同じ CFG を共有する。
- ブロックのアクションが全て終わったら捨てる（`OnOperationBlockActionsExecuted`, `:1724-1740`）。
- 公開の `ControlFlowGraph.Create` で作ったグラフはキャッシュされない。

### 4.3 IOperation と BoundTree の比較

| | BoundTree | IOperation |
|---|---|---|
| 公開 | internal | public（インターフェイスは Roslyn だけが実装する） |
| 言語 | C#／VB で別 | 共通（同じ `Operation` クラス） |
| 安定性 | リリースごとに変わる | `OperationKind` は追加のみ（XML の順序と公開 API アナライザーで保証） |
| 意味の段階 | 初期束縛 → Lowering で縮退 | 初期束縛（Lowering 前、null 許容性の書き換え済み）のみ |
| ソースとの対応 | 0〜多 | 明示的なノードは構文と 1 対 1、合成は `IsImplicit` |
| 制御フロー | フロー解析は木を直接歩く | CFG を明示的に構築する |

### 4.4 VB との共通化

VB の `VisualBasicOperationFactory.vb`（1,704 行）も、C# と**同じ `Core` の `Operation` クラス**（`InvocationOperation`、`ConversionOperation` …）を作る。`MemberSemanticModel` のキャッシュの仕組みも同じ。VB 固有の構文（`With`、static ローカル、`For…To`）は、内部ノードと内部プロパティで CFG まで運ぶ。**1 つの `ControlFlowGraphBuilder` で両言語を扱う**。

---

## 5. 設計上の要点まとめ

1. **公開の安定した意味モデル**：内部の BoundTree を直接公開せず、XML の宣言から生成した安定したインターフェイス群で包む。実装の自由度と、アナライザーの互換性を両立させている。
2. **言語共通**：C# と VB が同じノードを作るので、多くのアナライザーを 1 つのコードで両言語に対応させられる。
3. **割り当てを抑えた木**：親は構築時に固定し、子は配列を作らずに状態機械で列挙する。メンバーごとに 1 回だけ作ってキャッシュする。
4. **CFG は解析しやすさのための別の低水準化**：Lowering と似た分解を、領域、捕捉、分岐の種類という解析向けの語彙で表す。子のグラフは遅延生成し、アナライザーの実行中だけ共有する。

---

## 付録: 主要ファイル早見表

| ファイル | 役割 |
|---|---|
| `Core/Portable/Operations/OperationInterfaces.xml` | ノードの定義 |
| `src/Tools/CompilerGeneratorTools/Source/IOperationGenerator/*` | 生成器 |
| `Core/Portable/Generated/Operations.Generated.cs` / `OperationKind.Generated.cs` / `FlowAnalysis.Generated.cs` | 生成物 |
| `Core/Portable/Operations/Operation.cs` / `IOperation*.cs` / `OperationNodes.cs` / `OperationWalker.cs` | 基底・手書きのノード・Walker |
| `Core/Portable/Operations/OperationMapBuilder.cs` / `OperationExtensions.cs` | 構文との対応・拡張メソッド |
| `CSharp/Portable/Operations/CSharpOperationFactory*.cs` | C# の変換 |
| `Core/Portable/Operations/ControlFlowGraph*.cs` / `BasicBlock.cs` / `ControlFlowBranch.cs` / `ControlFlowRegion*.cs` | CFG の公開型 |
| `Core/Portable/Operations/ControlFlowGraphBuilder*.cs` | CFG の構築 |
| `Core/Portable/DiagnosticAnalyzer/AnalyzerDriver.cs` / `AnalyzerExecutor.cs` | アナライザーとの統合 |
| `VisualBasic/Portable/Operations/VisualBasicOperationFactory*.vb` | VB の変換 |
