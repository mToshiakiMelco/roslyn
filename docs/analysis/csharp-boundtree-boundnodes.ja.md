# Roslyn C# BoundTree と `BoundNodes.xml` 詳細解説
## ― 意味解析の中間表現と、その自動生成の仕組み ―

対象:
- `src/Compilers/CSharp/Portable/BoundTree/`（`BoundNodes.xml` / `BoundNodes.xsd` / 手書き partial 約 80 ファイル）
- `src/Tools/CompilerGeneratorTools/Source/BoundTreeGenerator/`（コード生成器、約 2,450 行）
- `src/Compilers/CSharp/Portable/Generated/BoundNodes.xml.Generated.cs`（生成物、17,746 行）

基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは特記なき限り `src/Compilers/CSharp/Portable/` からの相対パス。

関連ドキュメント:
- 構文側の同じ仕組み → `csharp-syntax-pipeline.ja.md`（`Syntax.xml` と Green/Red ツリー）
- BoundTree を歩く解析 → `csharp-compiler-dataflow-analysis.ja.md`

---

## 0. 要約

- **BoundTree（束縛済みツリー）** は、構文木（Syntax）をバインダが意味解析した結果の **コンパイラ内部の中間表現（IR）**。名前は全てシンボルに解決され、全ての式に型が付き、暗黙の変換は `BoundConversion` ノードとして明示され、オーバーロードは選択済み。
- ノード型（具象 239、抽象 28）は **`BoundNodes.xml` に宣言的に定義**され、`BoundTreeGenerator` が次のものを 1 つのファイル `BoundNodes.xml.Generated.cs` に生成する。
  - `BoundKind` 列挙
  - 全ノードクラス（コンストラクタ・プロパティ・`Update`・`Accept`）
  - 3 種類の Visitor
  - Walker・Rewriter
  - `NullabilityRewriter`
  - ツリーダンパ
  - パイプライン段階の検証器
- 構文木と違って、**full fidelity（元テキストの完全保持）も Red/Green の分離も無い**。BoundTree は「意味」だけを持つ普通の不変オブジェクトの木で、各ノードは元の `SyntaxNode` を 1 つ指す。
- 同じ BoundTree の型が **バインド → フロー解析 → Lowering（数段）→ IL 生成** の全段階で使われる。段階ごとに「この種別はもう存在してはならない」という制約があり、それを XML の `DoesNotSurvive` 属性で宣言し、DEBUG ビルドで検証する。
- 構文側（`Syntax.xml`）がビルド時のソースジェネレータで生成されるのに対し、**`BoundNodes.xml` は `eng/generate-compiler-code` スクリプトで生成し、生成物をコミットする**方式（§2.1）。

---

## 1. BoundTree の位置づけ

### 1.1 コンパイルパイプライン上の役割

```text
SyntaxTree (Red/Green)                                   … 構文: 何が書いてあるか
   │ Binder (Binder_Statements.cs / Binder_Expressions.cs ...)
   ▼
BoundTree［初期束縛］  BoundCall, BoundConversion, BoundLambda, BoundUnconvertedXxx ...  … 意味: それが何を意味するか
   ├─→ NullableWalker / DefiniteAssignment / ControlFlowPass / RefSafetyAnalysis（解析、ツリーは変えない）
   ├─→ SemanticModel（構文→Bound の対応表で GetTypeInfo/GetSymbolInfo を答える）
   ├─→ CSharpOperationFactory → IOperation（アナライザー向け公開 API）
   ▼ PipelinePhaseValidator.AssertAfterInitialBinding        MethodCompiler.cs:1508
LocalRewriter.Rewrite                 … foreach/using/lock/パターン/補間文字列/?. などを基本構文へ
   ▼
（ExtensionMethodReferenceRewriter / AsyncExceptionHandlerRewriter）
   ▼
ClosureConversion.Rewrite             … ラムダ・ローカル関数をクラス/メソッドへ  → AssertAfterClosureConversion (:1596)
   ▼
IteratorRewriter / AsyncRewriter(RuntimeAsyncRewriter) … ステートマシン化   → AssertAfterStateMachineRewriting (:1621)
   ▼
CodeGen.CodeGenerator                 … BoundTree → IL
```

`MethodCompiler.LowerBodyOrInitializer`（`Compiler/MethodCompiler.cs:1495-1630`）がこの順に Rewriter を適用する。どの段階の入出力も同じ `BoundStatement`/`BoundNode` 型の木で、**段階を追うごとに使われるノード種別が減っていく**。

### 1.2 構文木との違い

| | 構文木（Syntax） | BoundTree |
|---|---|---|
| 目的 | ソースの忠実な表現（IDE・整形） | 意味の表現（解析・コード生成） |
| 公開 | 公開 API | **internal**（公開されるのは IOperation） |
| 構造 | Green（共有）＋ Red（親・位置） | 単一層。親参照なし・位置なし（`Syntax` を参照） |
| trivia・括弧 | 全て保持 | 無い（括弧は消える。ただし `Syntax` から辿れる） |
| 名前 | 識別子トークン | `Symbol`（`MethodSymbol`, `LocalSymbol` …） |
| 型 | 無い | `BoundExpression.Type`（`TypeSymbol?`） |
| 暗黙の操作 | 無い | `BoundConversion`、既定引数、params 配列などを明示 |
| 誤り | 診断として保持 | `HasErrors` フラグ、`BoundBadExpression` |
| 定義 | `Syntax.xml` | `BoundNodes.xml` |
| 生成方法 | ソースジェネレータ（ビルド時） | スクリプト生成＋コミット |
| 1 構文に対する数 | 1 | 0〜多（例：`a += b` → 複合代入＋変換＋演算子…） |

---

## 2. `BoundNodes.xml` と `BoundNodes.xsd`

### 2.1 ビルドへの組み込み

`Microsoft.CodeAnalysis.CSharp.csproj:34-35` では `BoundTree\BoundNodes.xml` と `.xsd` は `<None>` 扱いで、ビルドには参加しない。生成は `eng/generate-compiler-code.cmd`（実体 `eng/generate-compiler-code.cs:182-185`）が行う。

```csharp
retVal |= BoundTreeGenerator.Program.Generate(
    language,
    Path.Join(languageDir, "BoundTree", "BoundNodes.xml"),
    Path.Join(generatedDir, $"BoundNodes.xml.Generated.{extension}"));
```

出力は `Generated/BoundNodes.xml.Generated.cs` としてコミットされ、通常のソースとしてコンパイルされる。`-test` オプションを付けると一時ディレクトリに生成して差分を比べ、XML を変えたのに再生成し忘れたケースを CI で検出する。同じジェネレータで VB 版（`BoundNodes.xml.Generated.vb`）も作る（`TargetLanguage.VB`）。

### 2.2 要素と属性

ジェネレータが実際に読むのは `Model.cs` の `XmlSerializer` 定義で、**`BoundNodes.xsd` ではない**（xsd はエディタ補完用）。

```text
<Tree Root="BoundNode">
  <ValueType Name/>                               … 値型として扱う型名（null 検査をしない）
  <Enum Name [Flags]> <Field Name Value/> </Enum>  … モデル上は可能（現在の C# XML では未使用）
  <PredefinedNode Name Base/>                      … モデル上は可能（未使用）
  <AbstractNode Name Base [HasValidate] [UpdateMethodModifiers]>
     <Field .../>*
  <Node Name Base [HasValidate] [DoesNotSurvive] [SkipInNullabilityRewriter] [UpdateMethodModifiers]>
     <Field Name Type [Null] [Override] [PropertyOverrides] [SkipInVisitor] [SkipInNullabilityRewriter] [New]/>*
```

`BoundNodes.xml`（2,722 行）での使用数:

| 要素・属性 | 数 | 意味と生成への影響 |
|---|---:|---|
| `<Node>` | 239 | 具象ノード（`sealed`。ただし他ノードの基底になる場合は非 sealed：`CanBeSealed`） |
| `<AbstractNode>` | 28 | 抽象基底（`BoundExpression`, `BoundStatement`, `BoundLoopStatement`, `BoundPattern` …） |
| `<ValueType>` | 20 | `ConversionKind`, `RefKind`, `BinaryOperatorKind`, `TextSpan`, `SafeContext` … |
| `Field/@Type` | — | ノード型、`ImmutableArray<…>`、シンボル型、値型、任意の参照型。末尾 `?` は null 許容 |
| `Null="allow"` | 88 | null（`ImmutableArray` なら default）を許す。既定 `disallow` は DEBUG で `RoslynDebug.Assert` |
| `Null="always"` | 12 | オーバーライドで「このサブクラスでは常に null」 |
| `Null="NotApplicable"` | 13 | 値型なので検査不要 |
| `Override="true"` | 122 | 基底のフィールドを再定義（例：`Type` を非 null の `TypeSymbol` に絞る → `public new TypeSymbol Type => base.Type!;`） |
| `PropertyOverrides="true"` | 50 | 基底クラスの virtual プロパティ（`ResultKind`, `ExpressionSymbol`, `IsEquivalentToThisReference` …）を `public override` で実装 |
| `HasValidate="true"` | 30 | コンストラクタ末尾で `[Conditional("DEBUG")] private partial void Validate();` を呼ぶ。本体は手書き（例：`BoundCall.cs:13`） |
| `DoesNotSurvive="…"` | 36 | そのノードが存在してよい最後のパイプライン段階（§6） |
| `SkipInVisitor="true"` | 45 | Walker/Rewriter がその子を訪問しない（プレースホルダや、同じノードを別経路で持つフィールド） |
| `SkipInVisitor="ExceptNullabilityRewriter"` | 1 | NullabilityRewriter だけは訪問する |
| `SkipInNullabilityRewriter` | 12 | NullabilityRewriter の生成を省略し、手書きする（`BoundBinaryOperator` のような深い左再帰を反復処理するため） |
| `UpdateMethodModifiers` | 2 | `Update` メソッドの修飾子を変更 |

`BoundNodes.xml` 冒頭のコメント（`:7-18`）にも `Null` 属性と `ValueType` の意味が書かれている。

> ⚠ **xsd と実装のずれ**：`BoundNodes.xsd`（55 行）には `DoesNotSurvive`・`UpdateMethodModifiers`・`Enum`・`PredefinedNode` が定義されておらず、`Node/Field/@SkipInVisitor` の型も他と違う。また `BoundNodes.xml:107`（`BoundCapturedReceiverPlaceholder`）と `:158`（`BoundImplicitIndexerReceiverPlaceholder`）の `Receiver` フィールドは **`SkipInVisit="true"`**（正しくは `SkipInVisitor`）と書かれている。`XmlSerializer` は未知の属性を黙って無視するため、生成された Walker/Rewriter はこの `Receiver` を**訪問している**（`BoundNodes.xml.Generated.cs:10064-10068`, `:11135-11140`）。意図どおりかどうかはコメントからは判断できないが、少なくとも属性の綴りは効いていない。

### 2.3 定義例

```xml
<!-- 最も単純な文 -->
<Node Name="BoundIfStatement" Base="BoundStatement" DoesNotSurvive="LocalRewriting">   <!-- :1257 -->
  <Field Name="Condition" Type="BoundExpression"/>
  <Field Name="Consequence" Type="BoundStatement"/>
  <Field Name="AlternativeOpt" Type="BoundStatement?"/>
</Node>

<!-- 抽象ノードの継承（ループ） -->
<AbstractNode Name="BoundLoopStatement" Base="BoundStatement">                         <!-- :1263 -->
  <Field Name="BreakLabel" Type="LabelSymbol"/>
  <Field Name="ContinueLabel" Type="LabelSymbol"/>
</AbstractNode>
<AbstractNode Name="BoundConditionalLoopStatement" Base="BoundLoopStatement">
  <Field Name="Locals" Type="ImmutableArray&lt;LocalSymbol&gt;"/>
  <Field Name="Condition" Type="BoundExpression"/>
  <Field Name="Body" Type="BoundStatement"/>
</AbstractNode>
<Node Name="BoundWhileStatement" Base="BoundConditionalLoopStatement"/>

<!-- 型を非 null に絞り、Validate を持つ式 -->
<Node Name="BoundCall" Base="BoundExpression" HasValidate="true">                         <!-- :1863 -->
  <Field Name="Type" Type="TypeSymbol" Override="true" Null="disallow"/>
  <Field Name="ReceiverOpt" Type="BoundExpression?"/>
  <Field Name="Method" Type="MethodSymbol"/>
  <Field Name="Arguments" Type="ImmutableArray&lt;BoundExpression&gt;"/>
  <Field Name="ArgumentNamesOpt" Type="ImmutableArray&lt;string?&gt;" Null="allow"/>
  <Field Name="Expanded" Type="bool"/>            <!-- params 展開形か -->
  <Field Name="ArgsToParamsOpt" Type="ImmutableArray&lt;int&gt;" Null="allow"/>
  <Field Name="DefaultArguments" Type="BitVector"/>
  ...
</Node>

<!-- まれな情報を別オブジェクトに追い出して省メモリ化 -->
<Node Name="BoundBinaryOperator" Base="BoundBinaryOperatorBase" SkipInNullabilityRewriter="true" HasValidate="true">  <!-- :488 -->
  <Field Name="OperatorKind" Type="BinaryOperatorKind"/>
  <Field Name="Data" Type="BoundBinaryOperator.UncommonData?"/>   <!-- 定数値・ユーザー定義演算子など（BoundBinaryOperator.UncommonData.cs） -->
  <Field Name="ResultKind" PropertyOverrides="true" Type="LookupResultKind"/>
</Node>
```

`BoundBinaryOperator` は最も数の多いノードの 1 つなので、定数値・ユーザー定義演算子メソッド・制約型などの「まれな情報」を `UncommonData` にまとめ、通常は null にしてメモリを節約している。

### 2.4 ジェネレータの検証規則

`Program.ValidateTree`（`BoundTreeGenerator/Program.cs`）：**`Conversion` 型のフィールドを持てるのは `BoundConversion` だけ**。他のノードが変換を表したい場合は、プレースホルダ＋`BoundConversion` ノードで表現しなければならない（エラーメッセージ「Types that are not BoundConversions should represent conversions as actual BoundConversion nodes, with placeholders if necessary.」）。変換を必ずノードとして明示させることで、NullableWalker・Lowering・IOperation が変換を見落とさないようにしている。

---

## 3. ジェネレータ `BoundTreeGenerator`

| ファイル | 行数 | 役割 |
|---|---:|---|
| `Model.cs` | 121 | `Tree`/`TreeType`/`AbstractNode`/`Node`/`Field`/`EnumType`/`ValueType`（`[XmlAttribute]` 付き POCO） |
| `Program.cs` | 97 | `<language> <input> <output>`。`XmlSerializer` で読み込み → `ValidateTree` → `BoundNodeClassWriter.Write` |
| `BoundNodeClassWriter.cs` | 2,229 | C# と VB の両方を出力する本体 |

出力構成（`BoundNodeClassWriter.WriteFile`, `:150-203`）:

```csharp
WriteKinds();                   // enum BoundKind : byte
WriteTypes();                   // 全ノードクラス
WriteVisitor();                 // BoundTreeVisitor<A,R> / BoundTreeVisitor（VisitInternal の switch と VisitXxx）
WriteWalker();                  // BoundTreeWalker
WriteRewriter();                // BoundTreeRewriter
WriteNullabilityRewriter();     // NullabilityRewriter（C# のみ）
WriteTreeDumperNodeProducer();  // BoundTreeDumperNodeProducer（テスト・デバッグ用ダンプ）
WritePipelinePhaseValidator();  // PipelinePhaseValidator.DoesNotSurvive（C#・DEBUG のみ）
```

生成コードの主な規則:

| 規則 | 実装 |
|---|---|
| 子ノードを持つ型はコンストラクタに `bool hasErrors = false` を付け、**子の `HasErrors()` を OR して基底に渡す** | `WriteConstructorWithHasErrors`（`:416-`） |
| 子ノードを持たない型は `hasErrors` 必須版と、`hasErrors` 無し版の 2 つを作る | `WriteConstructor`（`:403-414`） |
| 非 null フィールドに DEBUG の null 検査 | `WriteNullChecks`（`:661-`）。メッセージ「make the type nullable in BoundNodes.xml to remove this check」 |
| `Update` の比較：`TypeSymbol` は `TypeSymbol.Equals(..., ConsiderEverything)`、その他のシンボルは `SymbolEqualityComparer.ConsiderEverything`、それ以外は `!=`（参照・値比較） | `WriteUpdatedMethodCSharp`（`:957-1005`） |
| 変化があれば新しいノードを作り、`CopyAttributes(this)` でフラグを引き継ぐ。変化が無ければ `this` を返す | 同上 |
| 他ノードの基底にならない型は `sealed` | `CanBeSealed`（`:286`） |
| C# のキーワードと衝突する名前は `@` を付ける（`@event` など） | `FixKeyword`（`:1888`） |

---

## 4. 生成コードの詳細

### 4.1 `BoundKind`

```csharp
internal enum BoundKind : byte      // BoundNodes.xml.Generated.cs:21
{
    FieldEqualsValue, PropertyEqualsValue, ParameterEqualsValue, GlobalStatementInitializer, ...   // 具象ノード 239 個
}
```

**`byte` なので種別は最大 256**。現在 239 種類で、`SpillSequenceSpiller` が特別な値として `(BoundKind)byte.MaxValue` を使っている（`BoundNode.cs` のコンストラクタの assert）。残りは十数個しかなく、新しいノードを追加する際の隠れた制約になっている。

### 4.2 ノードクラス（`BoundIfStatement`）

```csharp
internal sealed partial class BoundIfStatement : BoundStatement
{
    public BoundIfStatement(SyntaxNode syntax, BoundExpression condition, BoundStatement consequence,
                            BoundStatement? alternativeOpt, bool hasErrors = false)
        : base(BoundKind.IfStatement, syntax,
               hasErrors || condition.HasErrors() || consequence.HasErrors() || alternativeOpt.HasErrors())   // ★ 誤りの上方伝搬
    {
        RoslynDebug.Assert(condition is object, "Field 'condition' cannot be null (make the type nullable in BoundNodes.xml to remove this check)");
        RoslynDebug.Assert(consequence is object, "...");
        this.Condition = condition;
        this.Consequence = consequence;
        this.AlternativeOpt = alternativeOpt;
    }

    public BoundExpression Condition { get; }
    public BoundStatement Consequence { get; }
    public BoundStatement? AlternativeOpt { get; }

    [DebuggerStepThrough]
    public override BoundNode? Accept(BoundTreeVisitor visitor) => visitor.VisitIfStatement(this);

    public BoundIfStatement Update(BoundExpression condition, BoundStatement consequence, BoundStatement? alternativeOpt)
    {
        if (condition != this.Condition || consequence != this.Consequence || alternativeOpt != this.AlternativeOpt)
        {
            var result = new BoundIfStatement(this.Syntax, condition, consequence, alternativeOpt, this.HasErrors);
            result.CopyAttributes(this);      // WasCompilerGenerated / IsSuppressed / WasConverted などを引き継ぐ
            return result;
        }
        return this;                          // ★ 変化が無ければ同じインスタンス → 部分木の共有
    }
}
```

### 4.3 `HasValidate` と型の絞り込み（`BoundCall`）

```csharp
public BoundCall(SyntaxNode syntax, BoundExpression? receiverOpt, ..., TypeSymbol type, bool hasErrors = false)
    : base(BoundKind.Call, syntax, type, hasErrors || receiverOpt.HasErrors() || arguments.HasErrors())
{
    RoslynDebug.Assert(!arguments.IsDefault, "Field 'arguments' cannot be null (use Null=\"allow\" in BoundNodes.xml to remove this check)");
    ...
    Validate();
}
[Conditional("DEBUG")] private partial void Validate();   // 手書き: BoundTree/BoundCall.cs:13（ResultKind の整合性など）
public new TypeSymbol Type => base.Type!;                  // Override="true" Null="disallow" の結果
public override LookupResultKind ResultKind { get; }      // PropertyOverrides="true" の結果
```

`Update` では、シンボルは `SymbolEqualityComparer.ConsiderEverything`、型は `TypeSymbol.Equals(..., ConsiderEverything)` で比較する。null 許容注釈の違いだけでも新しいノードになる（NullabilityRewriter が型を更新するため）。

### 4.4 Visitor（3 種類）

```csharp
// (1) 引数・戻り値ジェネリック版
internal abstract partial class BoundTreeVisitor<A, R> {
    [MethodImpl(MethodImplOptions.NoInlining), DebuggerStepThrough]
    internal R VisitInternal(BoundNode node, A arg) {        // Generated.cs:9071  全 239 種の switch
        switch (node.Kind) { case BoundKind.FieldEqualsValue: return VisitFieldEqualsValue((BoundFieldEqualsValue)node, arg); ... }
    }
    public virtual R VisitIfStatement(BoundIfStatement node, A arg) => this.DefaultVisit(node, arg);   // :9662
}
// (2) 非ジェネリック版（BoundNode? を返す）。Walker/Rewriter と全解析の基底
internal abstract partial class BoundTreeVisitor {
    public virtual BoundNode? VisitIfStatement(BoundIfStatement node) => this.DefaultVisit(node);      // :9903
}
```

手書きの `BoundTreeVisitor<A,R>.Visit`（`BoundTree/BoundTreeVisitors.cs:23-`）は、**頻出の 50 種類弱だけを先に `switch` で振り分け**（`TypeExpression`、`UnaryOperator`、`BinaryOperator`、`AssignmentOperator`、`Call`、`Local` …）、残りを生成された `VisitInternal` に回す。巨大な switch が JIT されてインライン化を妨げないよう、`VisitInternal` は `NoInlining` にしてある。非ジェネリック版は `node.Accept(this)`（仮想呼び出し）でディスパッチする。

### 4.5 `BoundTreeWalker`（読むだけ）

```csharp
public override BoundNode? VisitIfStatement(BoundIfStatement node)    // Generated.cs:10431
{
    this.Visit(node.Condition);
    this.Visit(node.Consequence);
    this.Visit(node.AlternativeOpt);
    return null;
}
```

**子ノードのフィールドだけを XML の宣言順に訪問**する。シンボルや値は訪問しない。`SkipInVisitor` の付いたフィールドは除外する。

### 4.6 `BoundTreeRewriter`（書き換え）

```csharp
public override BoundNode? VisitIfStatement(BoundIfStatement node)    // Generated.cs:11735
{
    BoundExpression condition = (BoundExpression)this.Visit(node.Condition);
    BoundStatement consequence = (BoundStatement)this.Visit(node.Consequence);
    BoundStatement? alternativeOpt = (BoundStatement?)this.Visit(node.AlternativeOpt);
    return node.Update(condition, consequence, alternativeOpt);
}
public override BoundNode? VisitCall(BoundCall node)                   // :12167
{
    MethodSymbol method = this.VisitMethodSymbol(node.Method);                       // ★ シンボルも「訪問」する
    ImmutableArray<MethodSymbol> originalMethodsOpt = this.VisitSymbols<MethodSymbol>(node.OriginalMethodsOpt);
    BoundExpression? receiverOpt = (BoundExpression?)this.Visit(node.ReceiverOpt);
    ImmutableArray<BoundExpression> arguments = this.VisitList(node.Arguments);
    TypeSymbol? type = this.VisitType(node.Type);                                    // ★ 型も
    return node.Update(receiverOpt, ..., method, arguments, ..., originalMethodsOpt, type);
}
```

- **シンボルと型も仮想メソッドで訪問する**（`VisitType`、`VisitMethodSymbol`、`VisitSymbols` など。既定実装は恒等写像。`BoundTree/BoundTreeRewriter.cs:17-160`）。
  - これにより、`ClosureConversion` やステートマシン生成で使う `MethodToClassRewriter`（`Lowering/MethodToClassRewriter.cs`）は、型パラメータの置換（メソッドのジェネリック型パラメータ → 合成クラスの型パラメータ）とローカル変数・パラメータの付け替えを、**ツリー全体に対して機械的に**行える。
- `VisitList`（`BoundTreeRewriter.cs:22-60`）は、要素が 1 つも変わらなければ**元の `ImmutableArray` をそのまま返す**。変化が最初に見つかった時点で初めて `ArrayBuilder` を確保する。
- `Update` が `this` を返す性質と合わせて、**変化の無い部分木は割り当てゼロで共有される**。

### 4.7 深い木への対策

`a + b + c + ... `（数千項）や長い `else if` は木が左に深くなり、再帰ではスタックが溢れる。

- `BoundTreeVisitor.VisitExpressionOrPatternWithStackGuard`（`BoundTreeVisitors.cs:200-255`）は、再帰の深さを数えて `StackGuard.EnsureSufficientExecutionStack` を呼ぶ。限界に達したら `InsufficientExecutionStackException` を `CancelledByStackGuardException` に包んで投げ、呼び出し側が `ERR_InsufficientStack` として報告する。
- `BoundTreeWalkerWithStackGuard` / `BoundTreeRewriterWithStackGuard`：上記を組み込んだ基底クラス。
- `...WithoutRecursionOnTheLeftOfBinaryOperator`（`BoundTreeWalker.cs:96-`, `BoundTreeRewriter.cs:237-300`）：`VisitBinaryOperator` を `sealed override` し、**左側の連鎖を明示スタックに積んで反復処理**する。XML で `SkipInNullabilityRewriter="true"` が付いた二項演算子系は、同じ理由で NullabilityRewriter も手書きしている。

### 4.8 `NullabilityRewriter`

```csharp
internal sealed partial class NullabilityRewriter : BoundTreeRewriter {            // Generated.cs:12705
    private readonly ImmutableDictionary<BoundExpression, (NullabilityInfo Info, TypeSymbol? Type)> _updatedNullabilities;
    private readonly NullableWalker.SnapshotManager? _snapshotManager;
    private readonly ImmutableDictionary<Symbol, Symbol>.Builder _remappedSymbols;

    public override BoundNode? VisitCall(BoundCall node) {                           // :14425
        MethodSymbol method = GetUpdatedSymbol(node, node.Method);   // 再推論された型引数付きメソッドへ差し替え
        ...
        if (_updatedNullabilities.TryGetValue(node, out var infoAndType)) {
            updatedNode = node.Update(..., infoAndType.Type!);
            updatedNode.TopLevelNullability = infoAndType.Info;      // ★ BoundNode の属性ビットに null 許容状態を書き込む
        } else { updatedNode = node.Update(..., node.Type); }
        return updatedNode;
    }
}
```

`NullableWalker.AnalyzeAndRewrite` が解析結果（式ごとの null 許容状態・再推論された型・更新されたシンボル）をこの Rewriter で BoundTree に反映し、**SemanticModel 用の BoundTree** を作る。`GetTypeInfo(expr).Nullability` はここで書き込まれた `TopLevelNullability` から来る（データフロー解析ドキュメント §8.3 を参照）。コンパイル（Emit）用の BoundTree には反映しない。

### 4.9 ダンパと `PipelinePhaseValidator`

- `BoundTreeDumperNodeProducer`（`Generated.cs:15474`）：全ノードを `TreeDumperNode("ifStatement", …)` の木に変換する。テストの期待値（`BoundNode.Dump()`、`BoundNode.cs:419`）やデバッガ表示に使う。`BoundNode_Source.cs` の `DumpSource()` は、C# 風の擬似ソースとして出力する。
- `PipelinePhaseValidator.DoesNotSurvive(BoundKind)`（`Generated.cs:17699`）：XML の `DoesNotSurvive` から作られる switch（§6）。

---

## 5. 手書きの基盤（`BoundTree/*.cs`）

### 5.1 `BoundNode`（`BoundTree/BoundNode.cs`）

```csharp
internal abstract partial class BoundNode {
    private readonly BoundKind _kind;               // 1 バイト
    private BoundNodeAttributes _attributes;        // short のビットフラグ
    public readonly SyntaxNode Syntax;              // 対応する構文ノード（SequencePoint 以外は非 null）
}
```

`BoundNodeAttributes`（`:24-64`）:

| ビット | 意味 |
|---|---|
| `HasErrors` | このノードか子孫にエラーがある。**Lowering/Emit を打ち切る判定**に使う（コンストラクタで子から OR される） |
| `CompilerGenerated` | 合成ノード（ソースに対応しない）。シーケンスポイント・IOperation の `IsImplicit`・診断位置に影響 |
| `IsSuppressed` | `!`（null 免除）演算子が適用された |
| `TopLevelFlowStateMaybeNull` / `TopLevelNotAnnotated` / `TopLevelAnnotated` | NullabilityRewriter が書き込む null 許容情報（3 ビット） |
| `WasCompilerGeneratedIsChecked` / `WasTopLevelNullabilityChecked` | 「既に読まれた後に書き換えていないか」を DEBUG で検出するための番兵 |
| `WasConverted` | 「全ての右辺値は、目的の型への変換か自然型への変換を必ず 1 回通る」という不変条件の検査用。タプルリテラル・switch 式・`new()` など、**型が文脈で決まる式（unconverted）**の取りこぼしを防ぐ |
| `ParamsArrayOrCollection` | params 展開で合成された配列・コレクション |
| `WasPropertyBackingFieldAccessChecked` | `field` キーワード関連の検査済みフラグ |

`AttributesPreservedInClone` に含まれるビットだけが `Update`/`MemberwiseClone` で引き継がれる（`CopyAttributes`, `:153-170`）。

`HasErrors` と `HasAnyErrors` の違い（`:96-140`）：

- `HasErrors`：ビットを読むだけで O(1)。
- `HasAnyErrors`：`HasErrors` に加えて、構文側のエラーと式の型がエラー型かどうかも調べる。バインダが連鎖エラー（cascading errors）を抑えるために使う。

### 5.2 `BoundExpression`（`BoundTree/BoundExpression.cs`）

| メンバー | 意味 |
|---|---|
| `Type`（生成） | 式の型。`null` は型を持たない式（メソッドグループ、`null` リテラル、ラムダの未変換状態など） |
| `ConstantValueOpt` | 定数畳み込みの結果（`BoundLiteral`、定数のローカル・フィールド、畳み込まれた演算など） |
| `ExpressionSymbol` | SemanticModel の `GetSymbolInfo` が返すシンボル |
| `ResultKind` | `LookupResultKind`（`Viable` 以外ならエラーノード） |
| `NeedsToBeConverted()` | 未変換の式か（§5.4） |
| `IsEquivalentToThisReference` | プレースホルダが `this` 相当か |

### 5.3 部分クラスによる拡張

`BoundCall.cs`、`BoundConversion.cs`、`BoundBinaryOperator.UncommonData.cs`、`BoundDecisionDag.cs` など約 80 ファイルが生成クラスの `partial` で、`Validate()` の本体、派生プロパティ、ヘルパーを追加している。`Constructors.cs` には、よく使う組み合わせの便宜コンストラクタ（21 個。例 `BoundObjectCreationExpression(syntax, constructor, params BoundExpression[] arguments)`）がある。

### 5.4 特殊なノード群

| 種類 | 例 | 役割 |
|---|---|---|
| **未変換ノード** | `BoundUnconvertedConditionalOperator`, `...SwitchExpression`, `...ObjectCreationExpression`（`new()`）, `...CollectionExpression`, `...InterpolatedString`, `...AddressOfOperator` | 型が文脈で決まる式。変換時に通常のノードへ置き換えられる。全て `DoesNotSurvive="InitialBinding"` |
| **プレースホルダ** | `BoundValuePlaceholder`, `BoundDeconstructValuePlaceholder`, `BoundAwaitableValuePlaceholder`, `BoundListPatternReceiverPlaceholder` … | 「ここに何らかの値が入る」という穴。`foreach` の要素変換、`await` の GetAwaiter 呼び出し、分解代入など、**変換や呼び出しを独立したノードとして先にバインドしておく**ために使う（§2.4 の規則と対になる）。Lowering で実際の式に置き換える |
| **エラーノード** | `BoundBadExpression`, `BoundBadStatement`（`IBoundInvalidNode`） | バインドに失敗した箇所。子ノード（候補）を保持し、IDE が部分的な情報を出せるようにする |
| **パターンの決定 DAG** | `BoundDecisionDag`, `BoundTestDecisionDagNode`, `BoundEvaluationDecisionDagNode`, `BoundDagTemp` … | switch やパターンを判定 DAG に変換したもの。到達可能性・網羅性の判定、NullableWalker、Lowering の全てが使う |
| **Lowering 専用** | `BoundSequence`, `BoundSpillSequence`, `BoundStatementList`, `BoundConditionalGoto`, `BoundLabelStatement`, `BoundSequencePoint` … | Lowering 後の低水準の制御フロー、評価順序の固定、デバッグ情報 |
| **ラムダ** | `BoundLambda`, `UnboundLambda` | `UnboundLambda` はデリゲート型ごとに何度もバインドし直せる（`BoundTree/UnboundLambda.cs`）。結果をキャッシュする |

---

## 6. パイプライン段階と `DoesNotSurvive`

```csharp
internal enum PipelinePhase {         // Lowering/LocalRewriter/PipelinePhaseValidator.cs:11（DEBUG）
    InitialBinding, LocalRewriting, Spilling, ClosureConversion, StateMachineRewriting, Emit }
```

`DoesNotSurvive="X"` は、**段階 X を過ぎたら存在してはならない**という宣言。

| 段階 | 数 | 該当する主なノード |
|---|---:|---|
| `InitialBinding` | 13 | 未変換ノード全般、`BoundTypeOrValueExpression`、`BoundTupleOperandPlaceholder`、`BoundDisposableValuePlaceholder` |
| `LocalRewriting` | 16 | `BoundIfStatement`、`BoundValuePlaceholder`、`BoundDeconstructValuePlaceholder`、リストパターン用プレースホルダ、`BoundPointerElementAccess` … |
| `Spilling` | 1 | `BoundSpillSequence` |
| `ClosureConversion` | 1 | `BoundLocalFunctionStatement` |
| `StateMachineRewriting` | 5 | `BoundAwaitExpression`、`BoundAwaitableInfo`、`BoundAwaitableValuePlaceholder`、`BoundYieldReturnStatement`、`BoundYieldBreakStatement` |
| 未指定 | 203 | IL 生成まで残ってよい（`Emit`） |

`MethodCompiler` は各段階の後で `PipelinePhaseValidator.AssertAfterInitialBinding` / `AssertAfterClosureConversion` / `AssertAfterStateMachineRewriting` を呼び（`MethodCompiler.cs:798, 1508, 1596, 1621`）、DEBUG ビルドでツリー全体を歩いて違反を検出する。`PipelinePhaseValidator` 自身も `BoundTreeWalkerWithStackGuardWithoutRecursionOnTheLeftOfBinaryOperator` の派生クラス。

例えば `BoundIfStatement` が `LocalRewriting` で消えるのは、`LocalRewriter` が `if` を `BoundConditionalGoto` と `BoundLabelStatement` に展開するから（`Lowering/LocalRewriter/LocalRewriter_IfStatement.cs:56`, `:78`）。したがって IL 生成器（`CodeGen`）は `if` 文を知らなくてよい。

---

## 7. BoundTree の生成と消費

### 7.1 生成：バインダ

- `Binder.BindStatement` / `BindExpression`（`Binder/Binder_Statements.cs`、`Binder_Expressions.cs` ほか）が、構文ノードごとに対応する BoundNode を返す。例えば `IfStatementSyntax` → `BindIfStatement` → `new BoundIfStatement(node, condition, consequence, alternative)`。条件式は `bool` への変換（`BoundConversion`）で包まれる。
- バインド中に出た診断は BoundTree ではなく `BindingDiagnosticBag` に集める。エラーがあればノードに `hasErrors: true` を渡し、それが親に伝搬する。
- 未変換の式は、目的の型が決まった時点で `CreateConversion` / `BindToNaturalType` によって通常のノードへ置き換えられ、`WasConverted` が立つ。

### 7.2 生成：Lowering の `SyntheticBoundNodeFactory`

`Lowering/SyntheticBoundNodeFactory.cs` は Lowering 用の便宜ファクトリ（`public Bound… Xxx(...)` 形式のメソッドが約 130 個）。`F.Call(receiver, method, args)`、`F.Local(local)`、`F.Assignment(l, r)`、`F.If(cond, then)`、`F.Goto(label)` のように、合成ノード（`WasCompilerGenerated = true`）を簡潔に書けるようにしている。現在の `Syntax` と `CurrentFunction` を保持し、合成コードの位置情報も一緒に設定する。

### 7.3 消費者

| 消費者 | 基底クラス | 用途 |
|---|---|---|
| `ControlFlowPass` / `DefiniteAssignmentPass` / `NullableWalker` | `AbstractFlowPass`（`BoundTreeVisitor` の派生） | フロー解析（データフロー解析ドキュメント） |
| `RefSafetyAnalysis`, `DiagnosticsPass` | Walker | 追加の検査 |
| `MemberSemanticModel.NodeMapBuilder`（`Compilation/MemberSemanticModel.NodeMapBuilder.cs:19`） | `BoundTreeWalkerWithStackGuard` | **構文ノード → BoundNode の対応表** `Dictionary<SyntaxNode, OneOrMany<BoundNode>>`（`MemberSemanticModel.cs:33`）を作る。`GetTypeInfo`/`GetSymbolInfo` はこの表から答える |
| `CSharpOperationFactory`（`Operations/CSharpOperationFactory.cs:34`） | `switch (boundNode.Kind)` | 初期束縛ツリー（Lowering 前）を IOperation に変換する |
| `LocalRewriter`, `ClosureConversion`, `SpillSequenceSpiller`, `IteratorRewriter`, `AsyncRewriter`, `MethodToClassRewriter` | `BoundTreeRewriter` 系 | Lowering |
| `CodeGen.CodeGenerator` | 独自の switch | IL 生成 |

---

## 8. 新しいノードを追加するときの手順

1. `BoundTree/BoundNodes.xml` に `<Node>` を追加する。
   - フィールドの null 許容性は `Type="…?"` か `Null="allow"` で指定する。
   - 変換は `Conversion` 型のフィールドにせず、`BoundConversion` ＋プレースホルダで表す（§2.4）。
   - Lowering で消えるなら `DoesNotSurvive` を指定する。
   - 不変条件を DEBUG で検査するなら `HasValidate="true"` を付け、`partial void Validate()` を手書きする。
2. `eng/generate-compiler-code.cmd` を実行して `Generated/BoundNodes.xml.Generated.cs` を再生成し、コミットする。
3. 新しい `VisitXxx` は全 Visitor の既定実装（`DefaultVisit`）に入るので、**意味のある処理が必要な既存の Visitor を全て見直す**。少なくとも以下。
   - `AbstractFlowPass` 系（到達可能性・確定代入・Nullable）
   - `LocalRewriter`（または `DoesNotSurvive` で消えることを保証する）
   - `CodeGenerator`（Emit まで残る場合）
   - `CSharpOperationFactory`
   - `NullabilityRewriter`（`SkipInNullabilityRewriter` を付けた場合は手書きする）
4. `BoundKind` は `byte` なので、種類数の上限（256。特殊値 255 は使用済み）に注意する（§4.1）。

---

## 9. 設計上の要点まとめ

1. **意味を明示する IR**：名前解決・型・変換・オーバーロード・既定引数・params 展開を、全てノードかフィールドとして明示する。特に「変換は必ず `BoundConversion` ノード」という規則をジェネレータが強制する。後段の解析が暗黙の意味を推測しなくて済む。
2. **宣言的定義と大量の定型コード生成**：239 ノード分のクラス、Visitor、Walker、Rewriter、NullabilityRewriter、ダンパ、段階検証器を、2,722 行の XML から約 1.8 万行生成する。新しいノードを追加しても、走査コードの書き漏れが起きにくい。
3. **不変性と構造共有**：`Update` は変化が無ければ `this` を返し、`VisitList` は変化が無ければ元の配列を返す。何段もの Rewriter を通しても、変わらない部分は割り当てなしで共有される。
4. **シンボルと型も書き換え可能な Rewriter**：`VisitType`/`VisitMethodSymbol` などのフックにより、クロージャ変換やステートマシン化の型パラメータ置換を汎用的に実装できる。
5. **段階的な縮退と DEBUG 検証**：Lowering の段階ごとに使えるノードが減り、`DoesNotSurvive` → `PipelinePhaseValidator` で各段階の不変条件を検証する。`HasValidate`、null 検査、`WasConverted`、`WasCompilerGeneratedIsChecked` など、DEBUG 専用の検査が多重に仕込まれている。
6. **誤りの伝搬**：`HasErrors` はコンストラクタで子から自動的に OR され、エラーを含む木は Lowering/Emit に進まない。
7. **深さと性能への配慮**：頻出ノードの先行 switch、`NoInlining` の巨大 switch、二項演算子の反復処理、スタックガード、まれな情報の `UncommonData` への分離。

---

## 付録 A: 主要ファイル早見表

| ファイル | 行数 | 役割 |
|---|---:|---|
| `BoundTree/BoundNodes.xml` | 2,722 | ノード定義 |
| `BoundTree/BoundNodes.xsd` | 55 | スキーマ（補完用。実装とずれあり） |
| `Generated/BoundNodes.xml.Generated.cs` | 17,746 | 生成コード |
| `BoundTree/BoundNode.cs` | — | 基底クラス・属性ビット・`HasErrors`・`Dump` |
| `BoundTree/BoundExpression.cs` | — | 式の基底（`ConstantValueOpt`, `ExpressionSymbol`, `ResultKind` …） |
| `BoundTree/BoundTreeVisitors.cs` | — | `Visit` の先行 switch、スタックガード |
| `BoundTree/BoundTreeWalker.cs` / `BoundTreeRewriter.cs` | — | Walker/Rewriter の手書き部分（StackGuard 版、二項演算子の反復処理、`VisitType` など） |
| `BoundTree/NullabilityRewriter.cs` | — | NullabilityRewriter の手書き部分 |
| `BoundTree/Constructors.cs` | — | 便宜コンストラクタ |
| `BoundTree/BoundNode_Source.cs`, `Formatting.cs` | — | `DumpSource()`、表示 |
| `Lowering/LocalRewriter/PipelinePhaseValidator.cs` | — | `PipelinePhase` と検証器本体 |
| `Lowering/SyntheticBoundNodeFactory.cs` | — | Lowering 用ファクトリ |
| `Tools/.../BoundTreeGenerator/BoundNodeClassWriter.cs` | 2,229 | ジェネレータ本体（C#/VB 両対応） |

## 付録 B: `if (x > 0) M(x);` の BoundTree（初期束縛）

```text
BoundIfStatement                                       DoesNotSurvive=LocalRewriting
├─ Condition: BoundBinaryOperator (OperatorKind=IntGreaterThan, Type=bool)
│    ├─ Left:  BoundLocal (LocalSymbol x, Type=int)
│    └─ Right: BoundLiteral (ConstantValue=0, Type=int)
├─ Consequence: BoundExpressionStatement
│    └─ BoundCall (Method=C.M(int), ReceiverOpt=BoundThisReference[CompilerGenerated], Type=void)
│         └─ Arguments[0]: BoundLocal (x)
└─ AlternativeOpt: null
```

LocalRewriter 後（概念図）:

```text
BoundStatementList
├─ BoundConditionalGoto (Condition=x > 0, JumpIfTrue=false, Label=<afterif>)
├─ BoundExpressionStatement (BoundCall M(x))
└─ BoundLabelStatement (<afterif>)
```

`int` の `x > 0` のように変換が不要な場合、`BoundConversion` は挿入されない。`long` と比較するなら、`Left` が `BoundConversion(ImplicitNumeric)` で包まれる。
