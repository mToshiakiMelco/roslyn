# Roslyn C# 宣言とシンボル 詳細解説
## ― 構文木からシンボル表へ：宣言テーブル・ソース/メタデータシンボル・遅延完了 ―

対象: `src/Compilers/CSharp/Portable/Declarations/`、`Symbols/`（`Source/`、`Metadata/PE/`、`Retargeting/`、`Synthesized/`、`PublicModel/`）、`Compilation/`（`SyntaxAndDeclarationManager`、`ReferenceManager`）
基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは特記なき限り `src/Compilers/CSharp/Portable/` からの相対パス。`Core/` は `src/Compilers/Core/Portable/`。

関連ドキュメント：`csharp-syntax-pipeline.ja.md`（入力の構文木）、`csharp-binder.ja.md`（シンボルを使うバインダ）、`csharp-compilation-semanticmodel-driver.ja.md`（コンパイル全体）

---

## 0. 要約

パイプライン上で構文解析とバインダの間にある層。

```text
SyntaxTree ──(DeclarationTreeBuilder, 遅延)──▶ RootSingleNamespaceDeclaration（1 ファイル 1 つ）
                    全ファイル分を DeclarationTable で併合 ──▶ MergedNamespaceDeclaration（partial 型を統合）
                                                                    │
CSharpCompilation ─ ReferenceManager（参照の解決・統合）─▶ SourceAssemblySymbol ─▶ SourceModuleSymbol
                                                                    └▶ SourceNamespaceSymbol ─▶ SourceNamedTypeSymbol ─▶ メンバー
                    参照アセンブリ ─▶ PEAssemblySymbol / PEModuleSymbol / PENamedTypeSymbol（メタデータから遅延読み込み）
```

- **宣言（Declaration）**：構文木から型と名前空間の**形だけ**を安く抜き出した要約。メンバーの本体は見ない。
  - 型の名前、種類、型パラメータの数。
  - メンバー名の集合。
  - 「属性があるか」「拡張メソッドがあるか」などのフラグ。
- **シンボル（Symbol）**：宣言から作るコンパイラ内部の意味モデル。全てが**遅延して部分的に計算される**。
  - 基底型、メンバー、シグネチャ、制約、属性は、必要になったときに、必要な部分だけ作る。
  - メソッド本体は、シンボルの構築では一切バインドしない（MethodCompiler か SemanticModel がバインドする）。
- **`CompletionPart` による段階的な完成**：シンボルごとのビットフラグで「どこまで計算済みか」を管理する。複数スレッドが同じシンボルを同時に完成させても、**各部分の計算と診断はちょうど 1 回だけ**になる。
- **IDE での編集に強い**：宣言テーブルは「多数の古いファイル＋いま編集中の 1 ファイル」に最適化されている。メンバー名の集合や参照の解決結果も、前のコンパイルから再利用する。
- **公開 API**（`ISymbol`）は、内部シンボルを包む別のオブジェクト（`PublicModel`）。null 許容性の注釈を持たせるためと、内部 API を自由に変えられるようにするため。

---

## 1. 宣言（`Declarations/`）

### 1.1 クラス階層

```text
Declaration（Declaration.cs:42）
├ SingleNamespaceOrTypeDeclaration          … 1 つのファイルの 1 つの宣言
│   ├ SingleNamespaceDeclaration ─ RootSingleNamespaceDeclaration（1 ファイルの根）
│   └ SingleTypeDeclaration
└ MergedNamespaceOrTypeDeclaration          … 全ファイルを併合した結果
    ├ MergedNamespaceDeclaration
    └ MergedTypeDeclaration                 … partial 型の全ての部分をまとめる
```

### 1.2 `DeclarationTreeBuilder`（`Declarations/DeclarationTreeBuilder.cs`）

```csharp
internal sealed class DeclarationTreeBuilder : CSharpSyntaxVisitor<SingleNamespaceOrTypeDeclaration>   // :24
public static RootSingleNamespaceDeclaration ForTree(SyntaxTree syntaxTree, string scriptClassName, bool isSubmission,
    OneOrMany<WeakReference<BoxedMemberNames>>? previousMemberNames = null)                            // :86
```

- **遅延実行**：`SyntaxAndDeclarationManager`（`Compilation/SyntaxAndDeclarationManager.cs:299`）が `new Lazy<RootSingleNamespaceDeclaration>(() => DeclarationTreeBuilder.ForTree(...))` で包む。宣言が必要になるまで構文木を歩かない。
- **訪問するもの**：コンパイル単位、名前空間（ファイルスコープを含む）、class/struct/interface/record/union/extension、delegate、enum。**メンバーの本体には入らない**。
- **トップレベル文**（`VisitNamespaceChildren` `:125`、`CreateSimpleProgram` `:234`）：
  - 通常のファイルの `GlobalStatementSyntax` を集め、`await`/`yield`/値付き `return` の有無を調べる（`SyntaxFacts.HasAwaitOperations` など）。
  - `Program` という名前の暗黙のクラス（`ImplicitClass`）の宣言を作り、`IsSimpleProgram | HasAwaitExpressions | …` のフラグを付ける。
- **フラグ**（`SingleTypeDeclaration.TypeDeclarationFlags`, `SingleTypeDeclaration.cs:41-75`）：`AnyMemberHasExtensionMethodSyntax`、`HasAnyAttributes`、`HasBaseDeclarations`、`AnyMemberHasAttributes`、`HasAnyNontypeMembers`、`HasAwaitExpressions`、`IsIterator`、`IsSimpleProgram`、`HasRequiredMembers`、`HasPrimaryConstructor` など。後段は**型を丸ごと飛ばす判定**に使う。
  - 例：型でないメンバーを持たない宣言は、メンバー構築でスキップする。
  - 例：どの部分にも属性が無ければ、`[Obsolete]` の判定を即座に「無し」とする。
- **メンバー名の集合**（`GetOrComputeMemberNames`, `:1054-1100`）：
  - Green ノードをキーにしたプロセス全体の `ConditionalWeakTable<GreenNode, BoxedMemberNames>`（`:36`）にキャッシュする。
  - **前のコンパイルの同じ型の集合と内容が同じなら、それを再利用する**（`WeakReference`, `:1076-1085`）。編集のたびに、メンバー名が変わらない型の集合を作り直さずに済む。

### 1.3 `DeclarationTable`：編集に強い併合（`Declarations/DeclarationTable.cs`）

クラスのコメント（`:15-22`）：「一定の宣言の集合と、繰り返し追加と削除をされる 1 つの構文木（編集中のファイル）」に最適化されている。

```csharp
private readonly ImmutableSetWithInsertionOrder<Lazy<RootSingleNamespaceDeclaration>> _allOlderRootDeclarations;  // :32 古いファイル群
private readonly Lazy<RootSingleNamespaceDeclaration>? _latestLazyRootDeclaration;                             // :33 最新の 1 ファイル
private readonly Cache _cache;                                                                                 // :36 古いファイル群の併合結果
```

- `CalculateMergedRoot`（`:83-106`）は、キャッシュ済みの「古いファイル群の併合結果」に最新の 1 ファイルを足して併合する。**1 ファイルの編集では、他の全ファイルの併合をやり直さない**。
- 型名と名前空間名の集合も、`UnionCollection.Create(キャッシュ, 最新ファイルの名前)` で、大きな集合を作り直さずに表す（`:127-139`）。
- `DeclarationTable.Builder`（`DeclarationTable.Builder.cs`）：
  - 最新ファイルだけを追加または削除する場合はキャッシュを保つ（`RealizeAdds` `:69-105`、`RealizeRemoves` `:107-143`）。
  - それ以外の変更では、前の最新ファイルを古い集合へ移すか、キャッシュを捨てる。
- `CSharpCompilation.ContainsSymbolsWithName`（`Compilation/CSharpCompilation.cs:4818`）は、**宣言のメンバー名の集合だけで答え、シンボルを作らない**。

### 1.4 partial 型の併合

- `MergedNamespaceDeclaration.MakeChildren`（`MergedNamespaceDeclaration.cs:90-146`）：子を名前空間と型に分け、名前空間は名前、型は「同一性」でまとめる。全て同じ名前なら辞書を作らない高速パスがある。
- **型の同一性**（`TypeDeclarationIdentity.Equals`, `SingleTypeDeclaration.cs:249-283`）：
  - 名前、型パラメータの数、種類が一致すれば併合する。
  - `file` 型は同じファイルの中だけで併合する。enum、delegate、extension は併合しない。
  - **`partial` キーワードの有無に関係なく併合し**、`partial` の欠落は後でシンボル側が診断する。
- `MergedTypeDeclaration`（`MergedTypeDeclaration.cs:22`）は全ての部分（`ImmutableArray<SingleTypeDeclaration>`）を持つ。メンバー名は各部分の和集合（遅延, `:256-268`）。

---

## 2. コンパイルからシンボルへ

1. `CSharpCompilation.GetBoundReferenceManager()`（`Compilation/CSharpCompilation.cs:1229-1240`）：初回に `ReferenceManager.CreateSourceAssemblyForCompilation(this)` を呼び、参照を解決して `SourceAssemblySymbol` を作る（§4.4）。
2. `SourceAssemblySymbol`（`Symbols/Source/SourceAssemblySymbol.cs:29`）が `SourceModuleSymbol` を作る。追加のネットモジュールは `PEModuleSymbol` になる。
3. `SourceModuleSymbol.GlobalNamespace`（`SourceModuleSymbol.cs:193-212`）が、`MergedRootDeclaration` から `SourceNamespaceSymbol` を遅延生成する（`Interlocked.CompareExchange`）。**競争に勝ったスレッドだけが診断を追加する**。

---

## 3. ソースのシンボル（`Symbols/Source/`）

### 3.1 `SourceNamespaceSymbol`

- メンバーの辞書 `Dictionary<ReadOnlyMemory<char>, ImmutableArray<NamespaceOrTypeSymbol>>`（`:24-48`）。`ReadOnlyMemory<char>` をキーにして、名前の一部で引くときに文字列を割り当てない。
- `GetNameToMembersMap()`（`:226-248`）：辞書を CAS で 1 回だけ作る。勝ったスレッドだけが次を行う。
  - 診断を追加する。
  - corlib をコンパイル中なら `System.Object` などの特殊型を登録する。
  - **`SymbolDeclaredEvent`** を発行する（アナライザー向け）。
  - `CompletionPart.NameToMembersMap` を完了にする。
- `BuildSymbol`（`:404-430`）：宣言の種類に応じて、`SourceNamespaceSymbol`、`SourceNamedTypeSymbol`、`ImplicitNamedTypeSymbol`（スクリプト、トップレベル文）を作る。型だけのグループは `NamedTypeSymbol[]` として確保し、型だけを求められたときにコピーせず使い回す（`:268-275`）。

### 3.2 `SourceMemberContainerTypeSymbol` / `SourceNamedTypeSymbol`

`SourceMemberContainerSymbol.cs:26` / `SourceNamedTypeSymbol.cs:28`。遅延する状態（`SourceMemberContainerSymbol.cs:195-237`）：

```csharp
protected SymbolCompletionState state;                  // 完成の進捗（§3.3）
protected readonly MergedTypeDeclaration declaration;   // 全ての partial 部分
private DeclaredMembersAndInitializers? _lazyDeclaredMembersAndInitializers;
private MembersAndInitializers? _lazyMembersAndInitializers;
private Dictionary<ReadOnlyMemory<char>, ImmutableArray<Symbol>>? _lazyMembersDictionary;
private Dictionary<ReadOnlyMemory<char>, ImmutableArray<NamedTypeSymbol>>? _lazyTypeMembers;
```

**メンバーの 2 段階構築**（`BuildMembersAndInitializers`, `:3669-3760`）：

1. **宣言されたメンバー**（`AddDeclaredNontypeMembers`, `:4061`）：構文から次のシンボルを作る（`AddNonTypeMembers`, `:5806-6070`）。
   - フィールド、メソッド（`SourceOrdinaryMethodSymbol`）、コンストラクタ、プロパティ、インデクサ、イベント、演算子
   - partial メソッドとプロパティは併合する（`:3762`, `:4169`）。
   - コメント（`:3720-3723`）にあるとおり、この段階ではパラメータをバインドしない。最終的なメンバーの一覧ができる前に、バインダのキャッシュにメンバーを取り込んでしまうため。
2. **合成メンバー**（`AddSynthesizedMembers`, `:4013-4041`）：
   - トップレベル文のエントリポイント
   - record のメンバー（位置プロパティ、`Deconstruct`、コピーコンストラクタ、`Clone`、`EqualityContract`、`Equals`、`GetHashCode`、`PrintMembers`、`ToString`、`==`/`!=`, `:5094-5670`）。**ユーザーが同じシグネチャで宣言していれば合成しない**。
   - 既定のコンストラクタと static コンストラクタ
   - 主コンストラクタ（`SynthesizedPrimaryConstructor`, `:4139`）

**名前で引く**：

- `GetMembers(string)`（`:1634`）は辞書を引くだけ。
- `GetSimpleNonTypeMembers`（`:1653-1661`）は、**宣言のメンバー名の集合に名前が無ければ、メンバーを 1 つも作らずに空を返す**。
- `MemberNames`（`:1391-1396`）は宣言の集合をそのまま返す（シンボルを作らない）。

### 3.3 `CompletionPart` と `SymbolCompletionState`：段階的な完成

- `[Flags] enum CompletionPart`（`Symbols/CompletionPart.cs:24-120`）。**ビットの意味はシンボルの種類ごとに使い回す**（例：`1 << 11` は、型では `Members`、名前空間では `NameToMembersMap`、フィールドでは `FixedSize`）。
  - 型の部分：`Attributes`、`StartBaseType`/`FinishBaseType`、`StartInterfaces`/`FinishInterfaces`、`TypeParameters`、`Members`、`TypeMembers`、`StartMemberChecks`/`FinishMemberChecks`、`MembersCompleted` など。
- `struct SymbolCompletionState`（`Symbols/SymbolCompletionState.cs:18`）：

  ```csharp
  private volatile int _completeParts;                                    // :30  完了ビットの集合
  internal bool HasComplete(CompletionPart part)                          // :76
  internal bool NotePartComplete(CompletionPart part)                     // :84  Interlocked でビットを立て、立てたスレッドだけ true
  internal CompletionPart NextIncompletePart                              // :95  最下位の未完了ビット
  internal void SpinWaitComplete(CompletionPart part, CancellationToken)  // :120 他スレッドが完了させるまで待つ
  ```

- **`Start*`/`Finish*` の対**で「作業を宣言（claim）してから、結果を公開する」。`StartBaseType` を立てられたスレッドだけが基底型を検査し、診断を追加し、`FinishBaseType` を立てる。他のスレッドは `SpinWaitComplete` で待つ。

### 3.4 `ForceComplete` とコンパイルによる駆動

```csharp
// SourceMemberContainerSymbol.ForceComplete (:583-750) の骨格
while (true) {
    var incompletePart = state.NextIncompletePart;
    switch (incompletePart) {
        case CompletionPart.StartBaseType: case CompletionPart.FinishBaseType:
            if (state.NotePartComplete(CompletionPart.StartBaseType)) {
                var diagnostics = BindingDiagnosticBag.GetInstance();
                CheckBase(diagnostics); AddDeclarationDiagnostics(diagnostics);
                state.NotePartComplete(CompletionPart.FinishBaseType); }
            break;
        case CompletionPart.Members: GetMembersByName(); break;
        case CompletionPart.MembersCompleted: /* 全メンバーの ForceComplete */ ...
        ...
    }
    state.SpinWaitComplete(incompletePart, cancellationToken);
}
```

**コンパイルの宣言診断がこれを駆動する**：

```text
GetDeclarationDiagnostics()                       CSharpCompilation.cs:3001
→ GetSourceDeclarationDiagnostics → Assembly.ForceComplete(location, filter, ct)   :3407/:3420
→ SourceAssemblySymbol.ForceComplete（属性・モジュール）                            SourceAssemblySymbol.cs:906-960
→ SourceModuleSymbol.ForceComplete → GlobalNamespace.ForceComplete                  SourceModuleSymbol.cs:226-300
→ 名前空間 → 型 → メンバー（ConcurrentBuild なら RoslynParallel.For で並列）
```

- 1 ファイルだけの診断（SemanticModel）では、`location` で範囲を絞り、そのファイルに関係する部分だけを完成させる（`NamedTypeSymbolWithLocationAll`）。
- 各部分は 1 つのスレッドだけが担当するので、**どの診断もちょうど 1 回だけ報告される**。

### 3.5 基底型・インターフェイスと循環の検出（`SourceNamedTypeSymbol_Bases.cs`）

2 段階で計算する。

1. **宣言された基底**（`MakeDeclaredBases`, `:280`）：全ての partial 部分の基底リストをバインドする。このとき `basesBeingResolved = basesBeingResolved.Prepend(this)`（不変の `ConsList`）をバインダへ渡す。**基底リストの中の名前探索が、解決中の型の基底を再び解決しようとする**のを防ぐ（無限再帰の防止）。
2. **循環の無い基底**（`MakeAcyclicBaseType`, `:770`）：`BaseTypeAnalysis.TypeDependsOn(declaredBase, this)`（`:825`）で推移閉包を調べる（宣言された基底と、同じコンパイル内の外側の型を含む）。循環があれば `ERR_CircularBase`（インターフェイスなら `ERR_CycleInInterfaceInheritance`）を報告し、エラー型に置き換える。

型パラメータの制約も同様に遅延計算する（`SourceTypeParameterSymbol.GetBounds(inProgress)`, `:222-245`）。`inProgress` の列で循環を検出する。種類（class/struct/new()）は早く必要になるので、型より先に別に計算する。

### 3.6 属性：早期と通常の 2 段階（`Symbols/Symbol_Attributes.cs:303` `LoadAndValidateAttributes`）

1. 属性の**型だけ**をバインドする（`Binder.BindAttributeTypes`）。
2. **早期デコード**（`EarlyDecodeWellKnownAttributes`, `:363`）：`[Obsolete]`、`[AttributeUsage]`、`[Conditional]` など、**バインダがバインド中に必要とする**属性を、引数を完全にはバインドせずに読む。引数までバインドすると循環を起こしうるため。
3. コンストラクタと引数をバインドする。
4. 既知の属性をデコードする（`DecodeWellKnownAttribute`）。
5. `SetAttributes` と後処理（`PostDecodeWellKnownAttributes`）。

`CustomAttributesBag<T>`（`Core/Symbols/Attributes/CustomAttributesBag.cs:19`）は、上の各段階の結果をスレッドセーフに段階的に保存する入れ物。`SourceNamedTypeSymbol.ObsoleteAttributeData`（`:1091-1110`）は次の順で答える。

1. 早期デコードの結果があれば、それを返す。
2. どの部分にも属性が無ければ（宣言のフラグ）、null。
3. それ以外は `Uninitialized`。この場合バインダは Obsolete の診断を後回しにする（`LazyObsoleteDiagnosticInfo`、補足トピック編 A.4）。

### 3.7 メンバーのシグネチャも遅延してバインドする

| シンボル | 方法 |
|---|---|
| メソッド | `LazyMethodChecks()`（`SourceMemberMethodSymbol.cs:462-511`）が**ロック**を取って `MethodChecks` を実行する。オーバーライドの解決が同じスレッドで仮のシグネチャを必要とするため、スピン待ちではなく**再入可能なロック**を使う。シグネチャは `BinderFactory.GetBinder(syntax.ReturnType, …)` に `SuppressConstraintChecks` を付けてバインドする（制約の検査はメンバー一覧ができた後で, `SourceOrdinaryMethodSymbol.cs:114-123`） |
| フィールド | `GetFieldType(fieldsBeingBound)`（`SourceMemberFieldSymbol.cs:476`）。`fieldsBeingBound` で `var` フィールドの型推論の循環を防ぐ |
| プロパティ | `EnsureSignature()`（`SourcePropertySymbolBase.cs:379-430`）。`Start/FinishPropertyEnsureSignature` で保護 |
| ローカル変数 | バインダが作る `SourceLocalSymbol`。`var` の型は初期化子から遅延推論する（`SourceLocalSymbol.cs:39`, `:500`） |

---

## 4. 参照アセンブリのシンボル（`Symbols/Metadata/PE/`）と参照の管理

### 4.1 PE シンボル

- `PEAssemblySymbol`（`PEAssemblySymbol.cs:22`）、`PEModuleSymbol`（`PEModuleSymbol.cs:26`）。`PEModuleSymbol` は `PEModule`（`Core/MetadataReader/PEModule.cs:33`。System.Reflection.Metadata の読み取りを包む）を持ち、型ハンドル → シンボルのキャッシュ（`ConcurrentDictionary`, `:67`, `:76`）を持つ。
- `PENamedTypeSymbol`（`PENamedTypeSymbol.cs:31`）：
  - **`MemberNames`**（`:1063-1150`）：メタデータからメンバーの**名前だけ**を列挙する。シンボルは作らない。
  - **`LoadMembers`**（`:1711-1935`）：初めて全メンバーが必要になったとき、フィールド、メソッド、プロパティ、イベントのシンボルを作って名前で分類する。その後、名前の集合を辞書のキーに差し替え、名前の情報を 1 つにする。
  - 基底型は、`[Dynamic]`、`[NativeInteger]`、`[TupleElementNames]`、`[Nullable]` 属性を読むデコーダを通して、C# の型（`dynamic`、`nint`、タプル名、null 許容注釈）に戻す（`:770-773`）。
- `MetadataDecoder`（`Symbols/Metadata/PE/MetadataDecoder.cs:21`。汎用部分は `Core/MetadataReader/MetadataDecoder.cs:111`）：シグネチャの blob を読んで型を作る。

### 4.2 使用場所の診断（use-site diagnostics）

- 参照アセンブリのシンボルに問題があっても、**使われるまで報告しない**（例：さらに別のアセンブリが無い、サポートされないメタデータ、`[RequiredAttribute]`、`CompilerFeatureRequired`）。
- `Symbol.GetUseSiteInfo()`（`Symbols/Symbol.cs:1078`）。`PENamedTypeSymbol` は結果をキャッシュする（`:2709-2720`）。バインダは `CompoundUseSiteInfo` に溜めて、使った場所で報告する（バインダ編 §14）。
- サポートされない型は `UnsupportedMetadataTypeSymbol`（`ERR_BogusType`）。

### 4.3 リターゲット（`Symbols/Retargeting/`）

- **なぜ必要か**（`RetargetingAssemblySymbol.cs:23-39`）：コンパイル C1 のソースアセンブリを、別のコンパイル C2 が参照する場合、C2 では C1 の依存先（corlib を含む）が**別のアセンブリに解決される**ことがある（Lib v1 と v2 など）。この場合 C1 の `SourceAssemblySymbol` はそのままでは使えない。
- `RetargetingAssemblySymbol` が元のソースアセンブリを包み、`RetargetingSymbolTranslator`（`RetargetingSymbolTranslator.cs:104`）が、型の参照を新しい依存先へ付け替えたシンボル（`RetargetingNamedTypeSymbol`、`RetargetingMethodSymbol` …）を 1 対 1 に遅延生成する。IDE で、プロジェクト間の参照がターゲットフレームワークの違いをまたぐときに使われる。

### 4.4 `ReferenceManager`（`Symbols/ReferenceManager.cs:55`）

`CreateSourceAssemblyForCompilation`（`:167-200`）は次の 3 つのどれかを行う。

- **完全な束縛**（`:342`）：
  1. メタデータ参照を解決する。
  2. `Bind`（`Core/ReferenceManager/CommonReferenceManager.Binding.cs:93`）で、各アセンブリの参照を `AssemblyIdentityComparer` で定義に結び付ける。**バージョンを統合（unification）**し、循環参照と corlib を判定する。
  3. キャッシュからアセンブリシンボルを再利用し、無ければ作る。
  4. `SourceAssemblySymbol` を作り、corlib を決める（corlib が無ければ `MissingCorLibrarySymbol`）。
- **前のコンパイルの結果の再利用**（`:297`）：`AddSyntaxTrees` などで派生したコンパイルでは、参照が変わっていなければ束縛結果をそのまま使う。
- 循環参照を見つけた場合の新しいマネージャー。

---

## 5. シンボルモデル

### 5.1 基底クラス（`Symbols/Symbol.cs:31`）

- `Kind`、`ContainingSymbol`、`ContainingAssembly`、`DeclaringCompilation`、`Locations`、`DeclaringSyntaxReferences`。
- `OriginalDefinition`（`:362`）：構築型やメンバーの元の定義。`IsDefinition => this == OriginalDefinition`。
- `RequiresCompletion` / `ForceComplete` / `HasComplete`（`:45-62`）：既定では何もしない（ソースのシンボルだけがオーバーライドする）。

### 5.2 構築型と置換シンボル

- `List<int>` は `NamedTypeSymbol.Construct`（`NamedTypeSymbol.cs:1418-1541`）→ `ConstructedNamedTypeSymbol`（`ConstructedNamedTypeSymbol.cs:57`）。
- `SubstitutedNamedTypeSymbol`（`SubstitutedNamedTypeSymbol.cs:25`）は `TypeMap`（型パラメータ → 型引数の写像）を持ち、メンバーを**元の定義のメンバーの置換版**（`SubstitutedMethodSymbol` など）として遅延生成する（`GetMembers`, `:240-272`）。戻り値型やパラメータも、要求されたときに置換する（`SubstitutedMethodSymbol.cs:30-33`）。
- `TypeMap.WithAlphaRename`（`TypeMap.cs:102`）：新しい型パラメータを作って名前の衝突を避ける。Lowering の型パラメータの置換（Lowering 編 §4.4）も同じ仕組みを使う。

### 5.3 `TypeWithAnnotations`（`TypeWithAnnotations.cs:22`）

型と `NullableAnnotation`（と修飾子）の組。`T?` で `T` が値型か参照型かまだ分からない段階のための遅延形（`LazyNullableTypeParameter`, `:1059`）がある。

### 5.4 エラー型

- `ErrorTypeSymbol`（`ErrorTypeSymbol.cs:20`）：エラー情報と候補シンボルを持つ。`ErrorTypeSymbol.UnknownResultType`（`:22`）は**遅延初期化の番兵**として全体で使われる。
- `ExtendedErrorTypeSymbol`：候補と結果の種類を記録する（バインダと循環の検出が使う）。
- `MissingMetadataTypeSymbol`：参照に無い型（`ERR_NoTypeDef`）。

### 5.5 合成シンボル・匿名型・タプル

- `Symbols/Synthesized/`：コンパイラが作るシンボル。
  - エントリポイント、既定コンストラクタ、バッキングフィールド
  - 埋め込み属性（`NullableAttribute` など）
  - インライン配列、`<PrivateImplementationDetails>`、匿名デリゲート、record のメンバー
  - 名前の生成規則（`GeneratedNames`）
- **匿名型**（`AnonymousTypes/AnonymousTypeManager.cs:15`）：バインド中は「公開用の匿名型シンボル」を使う。実際のクラス（テンプレート）は、プロパティ名の並びをキーにした辞書で共有する（`AnonymousTypeManager.Templates.cs:30`）。名前は Emit 時に付ける（`AssignTemplatesNamesAndCompile`, `:452`）。
- **タプルは別のクラスではない**：`System.ValueTuple<…>` の `NamedTypeSymbol` に、要素名などの付加情報（`TupleExtraData`）を持たせたもの（`Tuples/TupleTypeSymbol.cs`）。8 要素以上は `Rest` で入れ子にする（`ValueTupleRestPosition = 8`）。

### 5.6 公開 API のラッパ（`Symbols/PublicModel/`）

- 内部のシンボルは `ISymbol` を実装しない。`Symbol.CreateISymbol()`（`Symbol.cs:1848-1860`）が、必要になったときに `PublicModel.NonErrorNamedTypeSymbol` などのラッパを作る（CAS で 1 つに）。
- **分けている理由**：
  - 内部 API（`TypeWithAnnotations`、完成の仕組み、バインダ専用のメンバー）を自由に変えられる。
  - **公開 API の型には、トップレベルの null 許容注釈を持たせる**必要がある。同じ内部型に対して、注釈の違うラッパを別に作る（`CreateITypeSymbol(nullableAnnotation)`, `NamedTypeSymbol.cs:2961`）。
  - 例えば `string` と `string?` は、内部では同じ `TypeSymbol` だが、公開 API では `NullableAnnotation` の違う別の `ITypeSymbol` になる。

---

## 6. 既知の型と特殊型

- **特殊型**（`System.Object`、`System.Int32` など）：`CSharpCompilation.GetSpecialType`（`:1748-1768`）→ corlib のアセンブリシンボルがスロットごとにキャッシュする（`MetadataOrSourceAssemblySymbol.cs:27`, `:54-98`）。
- **既知の型**（`System.Threading.Tasks.Task` など）：`GetWellKnownType`（`Symbols/Compilation_WellKnownMembers.cs:124-199`）が、参照を含めてメタデータ名で探す。
  - C# 7 以前からある型は、あいまいでも警告で済ませる。それ以降の型は、あいまいなら「見つからない」として扱う。
  - スロットごとに CAS でキャッシュする。
- **既知のメンバー**（`GetWellKnownTypeMember`, `:81-117`）：`WellKnownMembers` の記述子の表（`Core/WellKnownMembers.cs`。バイト列からデコードする）に基づいて、型からメンバーを探す。Lowering の `ERR_MissingPredefinedMember`（Lowering 編 §1.3）はここで見つからなかった場合。

---

## 7. スレッド安全性と性能のパターン

1. **1 回だけ公開する遅延初期化**：`if (_lazy == null) Interlocked.CompareExchange(ref _lazy, Compute(), null);`。null の代わりに番兵を使うことも多い（`ErrorTypeSymbol.UnknownResultType`、`TypeParameterBounds.Unset`、`DeclaredMembersAndInitializers.UninitializedSentinel`）。**診断を追加するのは CAS に勝ったスレッドだけ**。負けたスレッドの計算は無駄になるが、診断は重複しない。
2. **`ImmutableArray`** は `ImmutableInterlocked.InterlockedInitialize` で初期化する（struct なので `IsDefault` を番兵にする）。
3. **完成のビット**：シンボルごとに `volatile int` 1 つ。`Start`/`Finish` の対と `SpinWaitComplete` で、多数のスレッドが同じシンボルのグラフを同時に完成させられる。範囲を絞った完成で、1 ファイルの診断にも対応する。
4. **名前で引く辞書**：`ReadOnlyMemory<char>` をキーにして割り当てを避ける。配列はコピーせずに使い回す。
5. **名前だけの判定を先に行う**：宣言のメンバー名、`ContainsName`、PE の `MemberNames` で、シンボルを作らずに「その名前のメンバーがあるか」に答える。
6. **本体をバインドしない**：宣言は本体を見ず（トップレベル文の await/yield のフラグだけ）、シグネチャは専用のバインダで制約の検査を後回しにしてバインドする。
7. **編集をまたぐ再利用**：遅延のルート宣言、宣言テーブルの古い集合と最新ファイルの分離、メンバー名の集合の再利用、参照の束縛結果の再利用、PE とリターゲットのアセンブリシンボルの再利用。
8. **再入**：メソッドのシグネチャ計算は、同じスレッドで自身の仮のシグネチャを必要としうるので、スピン待ちではなくロックを使う。

---

## 付録: 主要ファイル早見表

| ファイル | 役割 |
|---|---|
| `Declarations/DeclarationTreeBuilder.cs` | 構文木 → 宣言 |
| `Declarations/DeclarationTable*.cs` | 宣言の併合・編集に強いキャッシュ |
| `Declarations/MergedNamespaceDeclaration.cs` / `MergedTypeDeclaration.cs` / `SingleTypeDeclaration.cs` | 併合と型の同一性 |
| `Compilation/SyntaxAndDeclarationManager.cs` | 構文木と宣言の対応・増分更新 |
| `Symbols/Source/SourceAssemblySymbol.cs` / `SourceModuleSymbol.cs` / `SourceNamespaceSymbol*.cs` | アセンブリ・モジュール・名前空間 |
| `Symbols/Source/SourceMemberContainerSymbol.cs` / `SourceNamedTypeSymbol*.cs` | 型・メンバーの構築・基底・属性 |
| `Symbols/CompletionPart.cs` / `SymbolCompletionState.cs` | 段階的な完成 |
| `Symbols/Symbol_Attributes.cs`、`Core/Symbols/Attributes/CustomAttributesBag.cs` | 属性の 2 段階デコード |
| `Symbols/BaseTypeAnalysis.cs` | 基底型の循環の検出 |
| `Symbols/Metadata/PE/*`、`Core/MetadataReader/*` | 参照アセンブリのシンボル |
| `Symbols/ReferenceManager.cs`、`Core/ReferenceManager/*` | 参照の解決・統合 |
| `Symbols/Retargeting/*` | リターゲット |
| `Symbols/SubstitutedNamedTypeSymbol.cs` / `TypeMap.cs` | 構築型・置換 |
| `Symbols/PublicModel/*` | 公開 API のラッパ |
| `Symbols/Compilation_WellKnownMembers.cs`、`Core/WellKnownMembers.cs` | 既知の型・メンバー |
