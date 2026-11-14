# Roslyn C# Compilation・SemanticModel・ドライバ 詳細解説
## ― 全体をまとめる層：不変のコンパイル、診断の段階、SemanticModel、csc、ソースジェネレータ、アナライザードライバ ―

対象:
- `src/Compilers/CSharp/Portable/Compilation/`（`CSharpCompilation`、`*SemanticModel`、`SyntaxAndDeclarationManager`）
- `src/Compilers/Core/Portable/Compilation/`（`Compilation`、`SemanticModel`）
- `src/Compilers/Core/Portable/CommandLine/`、`src/Compilers/CSharp/csc/`、`src/Compilers/Shared/`、`src/Compilers/Server/VBCSCompiler/`（コマンドラインとコンパイラサーバー）
- `src/Compilers/Core/Portable/SourceGeneration/`、`src/Compilers/CSharp/Portable/SourceGeneration/`（ソースジェネレータ）
- `src/Compilers/Core/Portable/DiagnosticAnalyzer/`（アナライザードライバ）

基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは `src/Compilers/` からの相対パス。

関連ドキュメント：本シリーズの他の全ての章（索引 `README.ja.md` を参照）

---

## 0. 要約

- **`CSharpCompilation` は不変**：`AddSyntaxTrees`、`ReplaceSyntaxTree`、`WithOptions` などは全て新しいコンパイルを返す。内部の状態を最大限に再利用する。
  - 参照マネージャー（束縛済みの参照）
  - 宣言テーブル（編集中の 1 ファイルだけを差し替える）
  - 遅延計算される全てのシンボル
- **診断は 3 段階**（`CompilationStage`）：
  1. `Parse`：構文木の診断
  2. `Declare`：宣言・シンボルの診断。`ForceComplete` で駆動する
  3. `Compile`：メソッド本体の診断。IL を出力せずに `MethodCompiler` を走らせる
- **SemanticModel** は構文木ごとの `SyntaxTreeSemanticModel` と、その中のメンバーごとの `MemberSemanticModel` の 2 層。
  - メンバーの本体は**必要な文だけを増分的にバインド**する。
  - null 許容性の解析が有効なら、最初にメンバー全体を 1 回だけ解析して書き換えた木を使う。
- **csc**：
  1. コマンドラインを解析する。
  2. コンパイルを作る。
  3. **ソースジェネレータ**を実行する。
  4. **アナライザー**を接続する。
  5. 宣言の診断を出す。
  6. メソッド本体をコンパイルする（IL 生成）。
  7. アナライザーの診断を集める。
  8. Emit する。

  コンパイラサーバー（VBCSCompiler）に名前付きパイプで依頼でき、失敗すればプロセス内でコンパイルする。
- **ソースジェネレータ**は**インクリメンタルなノードグラフ**（Select/Where/Combine/Collect、構文プロバイダ）で表す。各ノードの結果を前回の実行と比較して、変化の無い部分の再計算を省く。
- **アナライザードライバ**は、コンパイルが発行する**イベントのキュー**（コンパイル開始、シンボルの宣言、ファイルの完了、コンパイル完了）を並列に処理する。シンボル、構文、IOperation、SemanticModel のアクションを呼び、例外を `AD0001` 診断に変える。

---

## 1. `CSharpCompilation`（`CSharp/Portable/Compilation/CSharpCompilation.cs`）

### 1.1 不変性と状態の再利用

- `public sealed partial class CSharpCompilation : Compilation`（`:44`）。中核の状態は次の 3 つ。
  - `_options`（`:54`）
  - `_referenceManager`（`:119`）
  - `_syntaxAndDeclarations`（`SyntaxAndDeclarationManager`, `:121`）
- 全ての変更は `Update(...)`（`:659-675`）で新しいコンパイルを作る。
  - **`AddSyntaxTrees`**（`:931-989`）：`#r`/`#load` を含まない構文木なら、参照マネージャーを再利用する（`reuseReferenceManager &= !tree.HasReferenceOrLoadDirectives`）。
  - **`ReplaceSyntaxTree`**（`:1069-1121`）：同じ構文木なら `this` を返す。新旧どちらにも `#r`/`#load` が無ければ、参照マネージャーを再利用する。
  - **`WithAssemblyName`**：名前が同じ場合だけ再利用する（InternalsVisibleTo などの解決が名前に依存するため, `:680-697`）。
- **宣言テーブルの増分更新**（`SyntaxAndDeclarationManager.cs`）：
  - 状態がまだ計算されていなければ、構文木の一覧を差し替えるだけ。
  - 計算済みなら、影響を受けた構文木の宣言だけを差し替える（`:100-110`, `:470-502`）。
  - 宣言とシンボル編 §1.3 の「古いファイル群＋最新の 1 ファイル」の仕組みと組み合わさり、**IDE での 1 回のキー入力が安く済む**。

### 1.2 遅延するフィールド

- `_lazyAssemblySymbol`：`GetBoundReferenceManager()`（`:1229-1240`）が初回に参照を束縛して `SourceAssemblySymbol` を作る。`_lazyGlobalNamespace`（`:1500-1519`）なども同様。
- **`BinderFactory` の弱参照キャッシュ**（`:2655-2709`）：構文木の番号ごとに `WeakReference<BinderFactory>` を持ち、ロック無しの CAS で埋める（バインダ編 §2.5）。

### 1.3 診断の段階（`Core/Portable/Compilation/CompilationStage.cs:13`）

```csharp
GetParseDiagnostics()       => GetDiagnostics(CompilationStage.Parse, …)     // :2992
GetDeclarationDiagnostics() => GetDiagnostics(CompilationStage.Declare, …)   // :3001
GetMethodBodyDiagnostics()  => GetDiagnostics(CompilationStage.Compile, …)   // :3009
```

| 段階 | 内容 |
|---|---|
| Parse | 構文木ごとの診断（`ConcurrentBuild` なら `RoslynParallel.For` で並列, `:3053-3065`）、オプションの誤り |
| Declare | アセンブリ名、オプション、参照マネージャーの診断、**`Assembly.ForceComplete` による宣言の診断**（宣言とシンボル編 §3.4） |
| Compile | `MethodCompiler.CompileMethodBodies(…, moduleBeingBuiltOpt: null, emittingPdb: false, emitMethodBodies: false, …)`（**バインド、フロー解析、ref 安全性の解析まで行い、IL は作らない**）＋ XML ドキュメントコメント ＋ 未使用の using（`:3138-3190`） |

- 最後に `FilterAndAppendDiagnostics` で `/nowarn`、`/warnaserror`、`#pragma` を適用する（補足トピック編 A.6）。
- `MethodCompiler` は `ConcurrentBuild` なら名前空間と型ごとにタスクを積み、全てを待つ（`MethodCompiler.cs:153-156`）。

### 1.4 イベントキュー（アナライザー用）

コンパイルは、設定されていれば `AsyncQueue<CompilationEvent>` にイベントを積む。

| イベント | 発行のタイミング |
|---|---|
| `CompilationStartedEvent` | コンストラクタ（`:606`） |
| `SymbolDeclaredCompilationEvent` | 各シンボルの宣言の診断が完了したとき（`SourceNamespaceSymbol.cs:239`、`SourceMemberContainerSymbol.cs:672`、`SourceOrdinaryMethodSymbol.cs:294` など） |
| `CompilationUnitCompletedEvent` | そのファイルの未使用 using の報告が終わったとき（`CompleteTree`, `:2722-2738`） |
| `CompilationCompletedEvent` | 全てのファイルが完了したとき（`Core/Portable/Compilation/Compilation.cs:1858-1871`） |

**コンパイルの進行（宣言の完成、本体のコンパイル）そのものがイベントを生み**、アナライザーはそれを追いかけて並行して実行される。

### 1.5 `GetSemanticModel`

`GetSemanticModel(tree, options)`（`:2621-2640`）は、`SemanticModelProvider` が設定されていればそれに任せ、無ければ `new SyntaxTreeSemanticModel(this, tree, options)`。アナライザードライバは `CachingSemanticModelProvider` を設定して、**コンパイラ自身とアナライザーが同じ SemanticModel を共有する**ようにする（§2.5）。

---

## 2. SemanticModel

### 2.1 階層

```text
SemanticModel（Core, 公開）
└ CSharpSemanticModel（抽象。GetSymbolInfo/GetTypeInfo/LookupSymbols/AnalyzeDataFlow/GetOperation …）   CSharpSemanticModel.cs:46
   ├ PublicSemanticModel
   │   ├ SyntaxTreeSemanticModel（構文木ごと）                SyntaxTreeSemanticModel.cs:25
   │   │   └ SpeculativeSyntaxTreeSemanticModel（型・cref の投機的解析）
   │   └ SpeculativeSemanticModelWithMemberModel
   └ MemberSemanticModel（internal、メンバーの本体ごと）      MemberSemanticModel.cs:27
       ├ MethodBodySemanticModel / InitializerSemanticModel / AttributeSemanticModel
```

BoundNode から API の結果への写像は、`GetSymbolInfoForNode`（`:1893`）と `GetTypeInfoForNode`（`:2029`）が行う。

- 「最も低い」BoundNode（変換の前）から `Type` を求める。
- 「最も高い」BoundNode（変換の後）から `ConvertedType` を求める。

### 2.2 `SyntaxTreeSemanticModel`（構文木ごと）

- `_memberModels`：`ImmutableDictionary<CSharpSyntaxNode, MemberSemanticModel>` を **`ImmutableInterlocked.GetOrAdd`** で更新する（ロック無し。競争すると 2 つ作られうるが、公開されるのは 1 つ, `:35`, `:1054-1065`）。
- `GetSymbolInfoWorker`（`:202`）：
  - ノードがメンバーの本体（メソッド本体、式本体、初期化子）の中なら、`GetMemberModel(node)`（`:835`）で得た `MemberSemanticModel` に任せる。
  - そうでなければ（型の宣言部分など）、`BinderFactory` のバインダで直接バインドする。
- `CreateMemberModel`（`:1070-1250`）：`symbol.TryGetBodyBinder(...)` を使って `MethodBodySemanticModel` などを作る（バインダ編 §1）。

### 2.3 `MemberSemanticModel`（メンバーの本体ごと）

```csharp
private readonly ReaderWriterLockSlim _nodeMapLock;                                       // :31
private readonly Dictionary<SyntaxNode, OneOrMany<BoundNode>> _guardedBoundNodeMap;      // :33 構文 → BoundNode
private readonly Dictionary<SyntaxNode, IOperation> _guardedIOperationNodeMap;           // :34 構文 → IOperation
private NullableWalker.SnapshotManager _lazySnapshotManager;                             // :36
```

- **増分バインド**（`GetBoundNodes`, `:2113-2204`）：
  1. 対応表を読み取りロックで引く。
  2. 無ければ、そのノードを含む**文**（バインドの根）だけを、`IncrementalBinder` を使って書き込みロックの下でバインドし、`NodeMapBuilder`（BoundTree 編 §7.3）で対応表に追加する。
- **`IncrementalBinder`**（`:2442`）：`BindStatement`（`:2486`）をオーバーライドし、**既にバインド済みの文は対応表から再利用する**。後の文を問い合わせても、前の文をバインドし直さない。
- **null 許容性**（`EnsureNullabilityAnalysisPerformedIfNecessary`, `:1957-2053`）：有効なら、最初の問い合わせでメンバー全体をバインドし、`NullableWalker` で解析・書き換えた木（スナップショット付き）を対応表に入れる（データフロー解析編 §8.3）。
- 投機的なモデル（`TryGetSpeculativeSemanticModel`）は、元のメソッドのスナップショットから null 許容性の状態を引き継ぐ（`MethodBodySemanticModel.cs:215`）。

### 2.4 SemanticModel とコンパイル結果の分離

- SemanticModel 用のバインダには `BinderFlags.SemanticModel` を付け、未使用の using の記録などに**影響を与えない**（バインダ編 §3.6）。
- `IgnoresAccessibility` オプションでは、アクセス可能性を無視してバインドする（IDE の補完などで使う）。

### 2.5 `CachingSemanticModelProvider`（`Core/Portable/DiagnosticAnalyzer/CachingSemanticModelProvider.cs:24`）

- コンパイルごとに、構文木 → SemanticModel のキャッシュを `ConditionalWeakTable` で持つ。
- アナライザードライバが、ファイルの完了時とコンパイルの完了時にキャッシュを消す。
- **コンパイラのメソッド本体のバインドと、アナライザーの問い合わせが同じモデル（とその中の BoundTree）を共有する**ので、二重にバインドしない。

---

## 3. コマンドライン（csc）

### 3.1 入口

```text
csc/Program.cs Main → BuildClient.Run(args, CSharpCompile, Csc.Run, GetCompileOnServerFunc, logger)   :14/:37
   ├ サーバー（VBCSCompiler）に名前付きパイプで依頼（Shared/BuildClient.cs:151）
   └ 失敗したらプロセス内で Csc.Run（Shared/Csc.cs:22）→ CSharpCompiler → CommonCompiler.Run
```

`CommonCompiler` のコンストラクタが `CSharpCommandLineParser`（`CSharp/Portable/CommandLine/CSharpCommandLineParser.cs:23`）で引数を解析する（`Core/Portable/CommandLine/CommonCompiler.cs:129`）。

### 3.2 `Run` / `RunCore`（`CommonCompiler.cs:748`, `:869`）

1. `/errorlog` があれば SARIF のロガーを作る（v1/v2）。
2. `/version`、`/help` などを処理し、引数のエラーを報告する。
3. `.editorconfig` とグローバル設定を読む（`AnalyzerConfigSet`）。
4. **`CreateCompilation`**（`CSharpCompiler.cs:36`）：ソースを（並列に）解析し、`CSharpCompilation.Create`。
5. アナライザーとジェネレータを解決する。
6. **`CompileAndEmit`**（`:1103`）。

### 3.3 `CompileAndEmit`

| 段階 | 内容 | 場所 |
|---|---|---|
| ① ソースジェネレータ | `driver.RunGeneratorsAndUpdateCompilation`。生成されたファイルは PDB に埋め込み、`/generatedfilesout` なら書き出す。コマンドラインではホスト出力（`RegisterHostOutput`）を無効にする | `:1155-1220`, `:803-865` |
| ② アナライザー | `AnalyzerDriver.CreateAndAttachToCompilation`：イベントキュー付きのコンパイルに差し替える | `:1533-1558` |
| ③ 宣言の診断 | `GetDiagnostics(Declare)`。抑止できないエラーがあれば、ここで打ち切る | `:1230-1243` |
| ④ 本体のコンパイル | `CheckOptionsAndCreateModuleBuilder` → `compilation.CompileMethods`（IL 生成まで） | `:1317-1333` |
| ⑤ アナライザーの診断 | `analyzerDriver.GetDiagnosticsAsync` → **診断サプレッサー**の適用 | `:1425-1431` |
| ⑥ Emit | PE、PDB、参照アセンブリのストリームへ `SerializeToPeStream` | `:1453-1495` |
| ⑦ touched files | 読んだファイルと書いたファイルの記録（`/touchedfiles`） | `:1569-1610` |

④の本体のコンパイルが進むにつれてイベントが発行され、②で接続したアナライザーが**並行して**走る。

### 3.4 コンパイラサーバー（VBCSCompiler）

- **サーバー**（`Server/VBCSCompiler/ServerDispatcher.cs:23`）：名前付きパイプで接続を待ち受け、既定では 10 分間アイドルが続くと終了する。
- **プロセス間で共有するもの**：
  - メタデータのキャッシュ（`MetadataCache`）
  - ジェネレータドライバのキャッシュ
  - 決定的キーによるコンパイルのキャッシュ
  - 非ロックのアナライザーローダー（`CompilerRequestHandler.cs:74`）
- **起動のコストと参照の読み込みを省く**のが目的。ディスク上のアナライザーと読み込み済みのものが食い違う要求は拒否する（`AnalyzerConsistencyChecker`）。

---

## 4. ソースジェネレータ（`Core/Portable/SourceGeneration/`）

### 4.1 ドライバ（不変）

- `GeneratorDriver`（`GeneratorDriver.cs:27`）は `readonly struct GeneratorDriverState` を 1 つ持ち、全ての操作が新しいドライバを返す。
- **`RunGeneratorsAndUpdateCompilation`**（`:66`）：各ジェネレータの出力（PostInit、PreCompilation、通常）を集め、`compilation.AddSyntaxTrees(trees)` を**1 回だけ**呼ぶ（`:78-85`）。宣言テーブルの増分更新（§1.1）がそのまま効く。
- `CSharpGeneratorDriver`（`CSharp/Portable/SourceGeneration/CSharpGeneratorDriver.cs:18`）：拡張子 `.cs`、C# の構文ヘルパー。

### 4.2 `ISourceGenerator` と `IIncrementalGenerator`

- `ISourceGenerator`：`Initialize` ＋ `Execute`（旧 API）。
- `IIncrementalGenerator`：`Initialize(IncrementalGeneratorInitializationContext)` だけ。
- **内部は全てインクリメンタル**で、旧 API のジェネレータは `SourceGeneratorAdaptor`（`GeneratorAdaptor.cs:34`）で包む。

### 4.3 実行の流れ（`RunGeneratorsCore`, `:231-438`）

1. 初回のみ、各ジェネレータの `Initialize` でパイプライン（ノードグラフ）を登録する。PostInit の出力を先にコンパイルへ加える。
2. **PreCompilation の段階**：コンパイルを参照しない出力。
3. **通常の段階**：`SetCompilation(compilation)` の後で、Source、Implementation、Host の出力を更新する。
4. 生成ファイルのパスは `baseDirectory/アセンブリ名/ジェネレータの型名/ヒント名`。

### 4.4 インクリメンタルなノードグラフ（`Nodes/`）

```csharp
internal interface IIncrementalGeneratorNode<T> {                            // Nodes/IIncrementalGeneratorNode.cs:14
    NodeStateTable<T> UpdateStateTable(DriverStateTable.Builder graphState, NodeStateTable<T>? previousTable, CancellationToken);
    IIncrementalGeneratorNode<T> WithComparer(IEqualityComparer<T> comparer);
    IIncrementalGeneratorNode<T> WithTrackingName(string name); }
```

| ノード | API | 役割 |
|---|---|---|
| `InputNode<T>` | `CompilationProvider`、`AdditionalTextsProvider`、`AnalyzerConfigOptionsProvider`、`ParseOptionsProvider`、`MetadataReferencesProvider` | 入力を取り出し、前回と比較する |
| `TransformNode` | `Select`、`SelectMany`、`Where` | 入力が Cached なら前回の結果をそのまま使い、変化した入力だけを再計算する。出力が前回と等しければ Cached にする |
| `BatchNode` | `Collect` | 全要素を 1 つの配列にまとめる |
| `CombineNode` | `Combine` | 2 つの入力を組み合わせる |
| `SyntaxInputNode` | `SyntaxProvider.CreateSyntaxProvider(predicate, transform)` | 構文木ごとに述語を走らせる。全ての構文ノードを 1 回の走査でまとめて調べる |
| — | `ForAttributeWithMetadataName` | まず**構文だけで**属性の単純名を絞り込み（エイリアスを考慮）、候補だけを SemanticModel で確かめる（`SyntaxValueProvider_ForAttributeWithMetadataName.cs:83-130`） |

### 4.5 キャッシュの単位：`NodeStateTable`（`Nodes/NodeStateTable.cs`）

- 各エントリの状態は `EntryState { Added, Removed, Modified, Cached }`（`:46`）。実行後に `AsCached()`（`:150-173`）で全てを Cached にし、次回はそこから比較する。
- `TryModifyEntries`（`:345`）：要素ごとに、等しければ Cached、違えば Modified、多ければ Removed/Added にする。
- **比較子（`WithComparer`）が等しいと判定すれば、下流のノードは再計算されない**。ジェネレータの性能は、比較可能で小さな値をパイプラインに流せるかで決まる。
- 公開の実行理由 `IncrementalStepRunReason`（New/Modified/Unchanged/Cached/Removed）で、テストやツールから各段階の再計算の有無を観察できる（`TrackIncrementalSteps`）。

### 4.6 例外とキャンセル

- ユーザーのラムダは `WrapUserFunction`（`UserFunction.cs:63`）で包み、キャンセル以外の例外を `UserFunctionException` に変える。
- 失敗したジェネレータは出力を出さず、**警告** CS8784（初期化中）または CS8785（生成中）を報告する。他のジェネレータは続行する（`GeneratorDriver.cs:578-620`）。
- 例外を捕捉しない設定（`CatchAnalyzerExceptions` が偽）なら FailFast する。

---

## 5. アナライザードライバ（`Core/Portable/DiagnosticAnalyzer/`）

### 5.1 接続（`AnalyzerDriver.CreateAndAttachToCompilation`, `AnalyzerDriver.cs:842-867`）

```csharp
newCompilation = compilation.WithSemanticModelProvider(CachingSemanticModelProvider.Instance)
                            .WithEventQueue(new AsyncQueue<CompilationEvent>());
analyzerDriver.Initialize(newCompilation, …);                         // 別タスクで各アナライザーの Initialize とコンパイル開始アクション
analyzerDriver.AttachQueueAndStartProcessingEvents(newCompilation.EventQueue!, …);
```

### 5.2 イベントの処理と並列性

- `ProcessCompilationEventsAsync`（`:1523`）：並列解析なら**プロセッサ数のワーカー**がキューからイベントを取り出す。`CompilationCompletedEvent` だけは、他の全てのイベントの後で処理する。
- イベントごとの処理（`TryProcessEventCoreAsync`, `:1733`）：

  | イベント | 実行するアクション |
  |---|---|
  | SymbolDeclared | シンボルのアクション。その宣言の構文ノード、IOperation（IOperation 編 §4）、コードブロックのアクション。SymbolStart/End |
  | CompilationUnitCompleted | SemanticModel のアクション。その後、そのファイルの SemanticModel のキャッシュを消す |
  | CompilationCompleted | コンパイル終了のアクション |

- **アナライザーごとの並行性**：`EnableConcurrentExecution()` を呼ばなかったアナライザーには、ロック（gate）を付けて逐次実行にする（`:529-538`）。
- **例外**：アナライザーの例外は `AD0001`、ドライバの例外は `AD0002` の診断に変える（`AnalyzerExecutor.cs:1335-1369`）。実行時間はアナライザーごとに集計する（`/reportanalyzer`）。

### 5.3 `CompilationWithAnalyzers`（IDE 向け、`CompilationWithAnalyzers.cs:24`）

- コンパイルを複製して、診断用のイベントキューと SemanticModel のキャッシュを付ける。結果は `AnalysisResultBuilder` に蓄える（旧 `AnalysisState` は廃止されている）。
- **ファイル単位・範囲単位の解析**（`GetAnalyzerSemanticDiagnosticsAsync(model, filterSpan, …)`）：そのファイルの SemanticModel の診断から必要なイベントだけを事前に生成し（`:991`）、それだけを処理する。エディタで開いているファイルだけを解析できる。

### 5.4 診断サプレッサー・.editorconfig・生成コード

- **`DiagnosticSuppressor`**（`DiagnosticSuppressor.cs:12`）：コンパイラやアナライザーの診断を、条件付きで抑止するアナライザー。csc は複数の時点で適用する。
- **.editorconfig**：`CompilerAnalyzerConfigOptionsProvider` がファイルごとのオプションを提供する。`dotnet_diagnostic.ID.severity` は構文木ごとの重大度として、補足トピック編 A.6 のフィルタに入る。
- **生成コード**：アナライザーは `ConfigureGeneratedCodeAnalysis` で明示的に許可しない限り、生成コードでは呼ばれない。生成コードの判定は次の順で行い、結果をキャッシュする（`AnalyzerDriver.cs:2327-2345`）。
  1. `.editorconfig` の `generated_code`。
  2. ファイル名。
  3. 先頭の `<auto-generated>` コメント。
  4. `[GeneratedCode]` 属性。

---

## 6. 設計上の要点まとめ

1. **不変性と構造の共有**：構文木（Red-Green）、宣言テーブル、参照マネージャー、シンボル、ジェネレータの状態が、全て「新しい版を作り、変わらない部分を共有する」形でできている。IDE の頻繁な編集とバックグラウンド解析を、ロックを最小限にして両立させている。
2. **遅延と段階**：コンパイルは「全てを最初に計算する」のではなく、診断の段階（Parse/Declare/Compile）、シンボルの完成の段階、SemanticModel の増分バインドという何段もの遅延で、**問い合わせに必要な分だけ**を計算する。
3. **コンパイラ、IDE、アナライザーで計算を共有する**：SemanticModel のキャッシュ（`CachingSemanticModelProvider`）、メンバーの BoundTree、IOperation の木、CFG のキャッシュ、コンパイルのイベントにより、同じ結果を何度も計算しない。
4. **拡張点の隔離**：ジェネレータとアナライザーの例外は診断に変えてコンパイルを続行し、非並行のアナライザーにはロックを付け、生成コードの扱いを明示的な許可制にする。
5. **インクリメンタルなジェネレータ**：比較可能な値が流れるデータフローグラフとして実行し、前回と等しい部分を省く。ジェネレータの作者にとっては、パイプラインの設計が性能を決める。

---

## 付録: 主要ファイル早見表

| ファイル | 役割 |
|---|---|
| `CSharp/Portable/Compilation/CSharpCompilation.cs` / `Core/Portable/Compilation/Compilation.cs` | コンパイル・診断の段階・イベント・Emit |
| `CSharp/Portable/Compilation/SyntaxAndDeclarationManager.cs` | 構文木と宣言の増分管理 |
| `CSharp/Portable/Compilation/CSharpSemanticModel.cs` / `SyntaxTreeSemanticModel.cs` / `MemberSemanticModel*.cs` | SemanticModel |
| `CSharp/Portable/Compilation/MethodBodySemanticModel.cs` / `InitializerSemanticModel.cs` / `AttributeSemanticModel.cs` / `Speculative*.cs` | 個別のモデル・投機的解析 |
| `Core/Portable/CommandLine/CommonCompiler.cs` / `CSharp/Portable/CommandLine/CSharpCompiler.cs` / `CSharpCommandLineParser.cs` | コマンドライン |
| `CSharp/csc/Program.cs` / `Shared/Csc.cs` / `Shared/BuildClient.cs` / `Server/VBCSCompiler/*` | csc とコンパイラサーバー |
| `Core/Portable/SourceGeneration/GeneratorDriver*.cs` / `IncrementalContexts.cs` / `Nodes/*` | ソースジェネレータ |
| `Core/Portable/DiagnosticAnalyzer/AnalyzerDriver.cs` / `AnalyzerExecutor.cs` / `CompilationWithAnalyzers.cs` / `AnalysisScope.cs` / `DiagnosticSuppressor.cs` / `CachingSemanticModelProvider.cs` | アナライザー |
