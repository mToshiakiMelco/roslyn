# Roslyn C# バインダ（意味解析）詳細解説
## ― 構文木から BoundTree を作る：スコープ・名前探索・式/文のバインド・変換・オーバーロード解決・型推論 ―

対象: `src/Compilers/CSharp/Portable/Binder/`（約 10.5 万行。うち `Semantics/` 約 2.6 万行）
基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは特記なき限り `src/Compilers/CSharp/Portable/Binder/` からの相対パス。

関連ドキュメント（同ディレクトリ）:
- `csharp-syntax-pipeline.ja.md`：入力となる構文木
- `csharp-boundtree-boundnodes.ja.md`：出力となる BoundTree
- `csharp-compiler-dataflow-analysis.ja.md`：出力を解析するフロー解析

---

## 0. 要約

- **バインダ（Binder）** は、構文木を受け取り、名前をシンボルに解決し、型を決め、暗黙の変換を挿入し、オーバーロードを選び、**BoundTree** を作る。C# の言語仕様（名前探索・型推論・変換・オーバーロード解決・演算子・パターン…）の大半がここに実装されている。
- `Binder` は 1 つの巨大な `partial class`（`Binder.cs` ＋ `Binder_*.cs` 約 40 ファイル）で、**インスタンスは単方向連結リスト（バインダチェーン）の 1 つの環**。各環は 1 つのスコープ（コンパイル単位・名前空間・型・メソッド・ブロック・ループ…）を表す。名前探索や文脈の問い合わせは、内側の環から `Next` をたどって外へ進む。
- チェーンは 2 段階で作られる。
  - メソッド本体の手前（名前空間・型・メソッド）までは **`BinderFactory`** が任意の構文位置に対して作り、キャッシュする。
  - 本体の中は **`ExecutableCodeBinder` ＋ `LocalBinderFactory`** が、最初に必要になった時点で「構文ノード → バインダ」の対応表を一括で作る。
- 式は `BindExpression` → 巨大な `switch` で種類別のメソッドへ振り分ける。**型が文脈で決まる式（`new()`、`?:`、`switch` 式、タプル、コレクション式、補間文字列、`default`）は、いったん「未変換」ノードとして作り**、目的の型が決まった時点で `CreateConversion` が確定させる。目的の型が無ければ `BindToNaturalType` が自然型で確定させる。
- 呼び出しは、メソッドグループ → **オーバーロード解決**（`Semantics/OverloadResolution/`）→ 引数の変換・既定引数・params の補完 → `BoundCall`。ジェネリックの型引数は **`MethodTypeInferrer`**（仕様の 2 段階アルゴリズム）で推論する。
- **ラムダはデリゲート型ごとに何度もバインドし直す**。候補ごとの適用可能性判定、戻り値型推論、最終的な変換、エラー回復の各段階で別々にバインドするため、結果をキャッシュして繰り返しを抑えている。
- 性能面の工夫：
  - 2 パス方式（安く試し、失敗したときだけ診断を集める）
  - 組み込み型同士の変換・演算子を引く EasyOut 表
  - 各種オブジェクトのプール
  - 深い左再帰の反復処理
  - O(n) の暫定首位探索の後に O(n²) の比較
- 診断の連鎖（cascading errors）を抑える規則：既にエラーのある式に対しては追加の診断を出さない。

---

## 1. パイプライン上の位置と入口

```text
SourceMemberMethodSymbol.TryGetBodyBinder()                 Symbols/Source/SourceMemberMethodSymbol.cs:808
   └ BinderFactory.GetBinder(本体) → InMethodBinder
   └ new ExecutableCodeBinder(syntax, method, inMethodBinder)
MethodCompiler.BindMethodBody                              Compiler/MethodCompiler.cs:1888
   importChain = bodyBinder.ImportChain                     （PDB 用 using 情報）
   bodyBinder.BindWithLambdaBindingCountDiagnostics(... bodyBinder.BindMethodBody(syntax, diagnostics))   :1942-1946
      Binder.BindMethodBody                                 Binder_Statements.cs:3711
        ├ 主コンストラクタ / コンストラクタ（初期化子 BindConstructorInitializer）
        ├ メソッド・アクセサ → BindMethodBody(decl, blockBody, expressionBody)   :4135 → BoundNonConstructorMethodBody
        ├ => 式本体 → BindExpressionBodyAsBlock                                   :3635
        └ トップレベル文 → BindSimpleProgram                                       :3740
   → NullableWalker.AnalyzeAndRewrite / AnalyzeIfNeeded → RefSafetyAnalysis → （FlowAnalysisPass → Lowering）
```

SemanticModel の経路も同じバインダを使う。

- `SyntaxTreeSemanticModel` は `BinderFactory` を持ち、メンバーの外では `GetEnclosingBinderInternal`（`Compilation/SyntaxTreeSemanticModel.cs:148-166`）でバインダを得る。
- 本体を持つメンバーは `CreateMemberModel`（`:1070`）で `MethodBodySemanticModel` を作る。
- `MemberSemanticModel.RootBinder` には `BinderFlags.SemanticModel` を付ける（`Compilation/MemberSemanticModel.cs:62`）。
- コンパイル時に作った BoundTree とバインダは `MethodBodySemanticModel.InitialState`（`MethodCompiler.cs:1996`）として渡され、SemanticModel が再利用する。

---

## 2. `Binder` の構造とバインダチェーン

### 2.1 クラスの形（`Binder.cs`）

```csharp
internal partial class Binder                                   // :21
{
    internal CSharpCompilation Compilation { get; }              // :23
    internal readonly BinderFlags Flags;                         // :25
    internal Binder(CSharpCompilation compilation)               // :30 根（BuckStopsHereBinder 専用）
    internal Binder(Binder next, Conversions? conversions = null)// :38 next.Flags を継承
    protected Binder(Binder next, BinderFlags flags)             // :47 フラグを明示
    protected internal Binder? Next { get; }                     // :115
}
```

文脈を返す仮想プロパティ・メソッドは、**既定では `Next` に問い合わせる**だけ。各派生バインダは、自分のスコープが持ち込むものだけをオーバーライドする。

- `GetBinder(SyntaxNode)`（`:196`）、`ContainingMemberOrLambda`（`:271`）、`IsInMethodBody`（`:337`）
- `IsDirectlyInIterator`（`:352`）、`GetBreakLabel`/`GetContinueLabel`（`:395`/`:405`）
- `ImportChain`（`:425`）、`InExecutableBinder`（`:447`）
- `ConstantFieldsInProgress`/`LocalInProgress`（`:498-516`）、`ConditionalReceiverExpression`（`:531`）

`Conversions`（`:541`）と `OverloadResolution`（`:555`）は、バインダごとに遅延生成する（`Interlocked.CompareExchange`）。アクセス可能性など、結果が「どこから見ているか」に依存するため、バインダへの参照を持つ。

デバッグ用に、`Dump()`（`:960-`）はチェーン全体（各環のローカル変数とスコープ）を表示する。

### 2.2 根：`BuckStopsHereBinder`（`BuckStopsHereBinder.cs:15`）

「シンボルを何も知らず、それ以上委譲しない」終端。

- `ImportChain` とローカル探索は `null` を返す。
- 最後のアクセス可能性判定 `IsAccessibleHelper`（`:67`）を、アセンブリを基準に行う。
- `file` ローカル型の判定に使う `AssociatedFileIdentifier`（`:31`）を持つ。

### 2.3 フラグ `BinderFlags`（`BinderFlags.cs:15`、`uint` の 32 ビット全て使用）

`SuppressConstraintChecks`、`SuppressObsoleteChecks`、`ConstructorInitializer`、`FieldInitializer`、`InFixedStatement`、`CollectionInitializerAddMethod`、`AttributeArgument`、`GenericConstraintsClause`、`Cref`、`CrefParameterOrReturnType`、`UnsafeRegion`、`SuppressUnsafeDiagnostics`、**`SemanticModel`**、`EarlyAttributeBinding`、**`CheckedRegion`/`UncheckedRegion`**、`InLockBody`、`InCatchBlock`、`InFinallyBlock`、`InTryBlockOfTryCatch`、`InCatchFilter`、`InNestedFinallyBlock`、**`IgnoreAccessibility`**、`ParameterDefaultValue`、`AllowMoveableAddressOf`、`AllowAwaitInUnsafeContext`、`IgnoreCorLibraryDuplicatedTypes`、`InContextualAttributeBinder`、`InEEMethodBinder`、`SuppressTypeArgumentBinding`、`InExpressionTree`、`CollectionExpressionConversionValidation`。

- **フラグは値としてコピーされる**。変更したいときは、フラグだけを変えた軽い環を 1 つ挟む（`Binder_Flags.cs`）。
  - `WithAdditionalFlags`（`:71`）。既に含まれていれば `this` を返す。
  - `WithContainingMemberOrLambda`（`:78`）
  - `WithCheckedOrUncheckedRegion`（`:122`）。片方を立て、もう片方を消す。
  - `SetOrClearUnsafeRegionIfNecessary`（`:94`）。C# 13 以降は、unsafe 文脈の中でもイテレータ本体を安全な文脈にする。
- `ExecutableCodeBinder` は生成時に `AllClearedAtExecutableCodeBoundary`（lock/catch/finally/try/fixed 系、`BinderFlags.cs:123`）を消す（`ExecutableCodeBinder.cs:39`）。「lock の中」といった状態がラムダやローカル関数の本体に漏れないようにするため。

### 2.4 典型的なチェーン

`namespace N { class C<U> { void M<T>(int p) { … ここ … } } }` の式をバインドするときのチェーン（外 → 内）：

```text
BuckStopsHereBinder                               根（BinderFactory ごとに 1 つ）
 └ WithUsingNamespacesAndTypesBinder              コンパイル単位の using
   └ WithExternAndUsingAliasesBinder              コンパイル単位の using エイリアス / extern alias
     └ InContainerBinder(グローバル名前空間)
       └ (N の using / エイリアス)
         └ InContainerBinder(N)                   名前空間 N のメンバー
           └ WithPrimaryConstructorParametersBinder(C)
             └ InContainerBinder(C)               型 C のメンバー（継承を含む）
               └ WithClassTypeParametersBinder(C) 型パラメータ U
                 └ WithMethodTypeParametersBinder(M)  型パラメータ T
                   └ InMethodBinder(M)            パラメータ p、イテレータ情報
                     └ ExecutableCodeBinder(M)    ここから下は本体用の対応表
                       └ BlockBinder({…})         ローカル変数・ローカル関数・ラベル
                         └ ForLoopBinder / ForEachLoopBinder / WhileBinder / UsingStatementBinder / LockBinder /
                           SwitchBinder / CatchClauseBinder / EmbeddedStatementBinder / ExpressionVariableBinder /
                           SwitchExpressionArmBinder / WithLambdaParametersBinder / NameofBinder ...
```

主なバインダ:

| バインダ | 役割 |
|---|---|
| `InContainerBinder`（`InContainerBinder.cs:21`） | 名前空間・型のメンバーを探索する（`LookupMembersInternal`）。拡張メソッドのスコープになる（`SupportsExtensions => true`）。型が容器なら、その型を基準にアクセス可能性を判定する |
| `WithUsingNamespacesAndTypesBinder` / `WithExternAndUsingAliasesBinder` | `using`・`using static`・エイリアス・`extern alias`。使われた using を記録する（§3.6） |
| `InMethodBinder`（`InMethodBinder.cs:23`） | メソッドのパラメータ（遅延構築の `MultiDictionary`）、イテレータの要素型、ローカル関数か |
| `LocalScopeBinder`（`LocalScopeBinder.cs:17`） | ローカル変数・ローカル関数・ラベルを持つ全バインダの基底。`BuildLocals`/`BuildLocalFunctions`/`BuildLabels` で**遅延構築**し、名前の対応表を作る。名前の衝突も診断する（`EnsureSingleDefinition`, `:541`） |
| `ExpressionVariableBinder` | `out var` やパターン変数（`is int n`）のスコープ。`ExpressionVariableFinder` が該当する宣言を集める（ラムダとクエリの中には入らない） |
| `EmbeddedStatementBinder` | `if (c) var x = …;` のような埋め込み文が宣言する変数を、その文の中に閉じ込める |

### 2.5 `BinderFactory`：メソッド本体の手前まで（`BinderFactory.cs`）

```csharp
internal readonly struct BinderCacheKey { CSharpSyntaxNode syntaxNode; NodeUsage usage; }   // :20
private readonly ConcurrentCache<BinderCacheKey, Binder> _binderCache;                    // :49（容量 50, :77）
internal Binder GetBinder(SyntaxNode node, ...)                                            // :109
```

- 構文木ごとに 1 つ。`CSharpCompilation.GetBinderFactory`（`Compilation/CSharpCompilation.cs:2658`）が `WeakReference` で保持するので、寿命は短い。
- `GetBinder(node)` は、プールされた `BinderFactoryVisitor` を使い、`node.Parent` から上へ構文をたどる。宣言ノードは、位置（`_position`）がそのノードのどの部分にあるかで **`NodeUsage`** を決める（`BinderFactory.NodeUsage.cs`）。同じメソッド宣言でも、本体の中、型パラメータの範囲、それ以外で別のバインダになる。
- 例：`VisitMethodDeclaration`（`BinderFactoryVisitor.cs:135`）は、キャッシュを確認し、無ければ親（型宣言）のバインダの上に `WithMethodTypeParametersBinder` と `InMethodBinder` を積んで `TryAdd` する。
- コンパイル単位（`:943`）：通常のコードでは「using 名前空間と型 → エイリアス → グローバル名前空間」の順に積む。スクリプトでは、グローバルインポート、前のサブミッションのインポート、ホストオブジェクト、サブミッションクラスの順。
- **ファクトリは `InMethodBinder` までしか作らない**。本体の中は次節の `LocalBinderFactory` の担当。

### 2.6 本体の中：`ExecutableCodeBinder` と `LocalBinderFactory`

```csharp
internal ExecutableCodeBinder(SyntaxNode root, Symbol memberSymbol, Binder next, ...)   // ExecutableCodeBinder.cs:32
private SmallDictionary<SyntaxNode, Binder> _lazyBinderMap;                             // :30
internal override Binder GetBinder(SyntaxNode node)
    => BinderMap.TryGetValue(node, out binder) ? binder : Next.GetBinder(node);         // :58
```

`LocalBinderFactory`（`LocalBinderFactory.cs:32`）は `CSharpSyntaxWalker` の派生クラスで、**本体全体を 1 回歩いて、スコープを作る全ての構文ノードについて「ノード → バインダ」の対応表を作る**（`BuildMap`, `:71`）。

| 構文 | バインダ |
|---|---|
| ブロック（`:540`） | `BlockBinder` |
| `for`（`:601-637`） | `ForLoopBinder`。条件は `ExpressionVariableBinder`、増分式は `ExpressionListVariableBinder` |
| `foreach`（`:650`） | コレクション式は外側の `ExpressionVariableBinder`、本体は `ForEachLoopBinder`。反復変数がコレクション式から見えないようにするため |
| `checked`/`unchecked`/`unsafe` | フラグだけを変える環 |
| `lock`（`:726-740`） | `LockBinder`。本体には `InLockBody` |
| `switch` 文 / 式 | `SwitchBinder` と各セクション、`SwitchExpressionBinder` と各アーム |
| `try`/`catch`/`finally` | `InTryBlockOfTryCatch`、`CatchClauseBinder`（フィルタには `InCatchFilter`）、`InFinallyBlock` |
| ローカル関数（`:404-421`）／ラムダ | `WithMethodTypeParametersBinder` ＋ `InMethodBinder`／`WithLambdaParametersBinder` |
| `nameof(…)` かもしれない呼び出し（`:213-235`） | `NameofBinder` |

文のバインドは「**このノード用のスコープバインダを対応表から取り出し、そのバインダで中身をバインドする**」形を一貫して使う。例：`BindBlock`（`Binder_Statements.cs:1909`）→ `GetBinder(node).BindBlockParts(node, …)`。

---

## 3. 名前探索

### 3.1 チェーンをたどる（`Binder_Lookup.cs`）

```csharp
private Binder LookupSymbolsInternal(LookupResult result, string name, int arity, ..., LookupOptions options, bool diagnose, ...)   // :87
{
    Binder binder = null;
    for (var scope = this; scope != null && !result.IsMultiViable; scope = scope.Next)
    {
        if (binder != null) { var tmp = LookupResult.GetInstance();
                              scope.LookupSymbolsInSingleBinder(tmp, name, arity, …, originalBinder: this, …);
                              result.MergeEqual(tmp); tmp.Free(); }
        else { scope.LookupSymbolsInSingleBinder(result, …, this, …); if (!result.IsClear) binder = scope; }
        ...
    }
}
```

- **最初に有効（Viable）な結果が見つかった環で止まる**。内側のスコープが外側を隠すという C# の規則そのもの。
- 有効でない結果（アクセス不可、引数の数違いなど）では止まらず、優先度で合成しながら外へ進む。
- `originalBinder`（探索を始めた環）を渡すことで、**アクセス可能性などは、シンボルが見つかった場所ではなく探索を始めた場所から判定**する。
- `LookupSymbolsWithFallback`（`:67`）は、まず `diagnose: false` で安く探索する。結果が Viable でも Empty でもない場合だけ、`diagnose: true` で探索し直す。診断情報（`DiagnosticInfo`）を作るのは失敗したときだけ。

### 3.2 `LookupResult` と優先順位

- `LookupResult`（`LookupResult.cs:62`）：プールされる。`_kind` と `ArrayBuilder<Symbol>`、エラーを持つ。
- `LookupResultKind`（`LookupResultKind.cs:38`）は**順序付き**で、値の大きいものが優先する。

```text
Empty < NotATypeOrNamespace < NotAnAttributeType < WrongArity < NotCreatable < Inaccessible < NotReferencable
      < NotAValue < NotAVariable < NotInvocable < NotLabel < StaticInstanceMismatch < OverloadResolutionFailure
      < Ambiguous < MemberGroup < Viable
```

- `MergeEqual`（`:256`）：種類が高い方が勝つ。両方 Viable ならシンボルを連結する（メソッドグループやあいまいさの材料になる）。両方 WrongArity ならジェネリックな候補を優先する。
- これにより「見つからない」より「見つかったがアクセスできない」の方が良い診断になる。また、どこかに Viable な結果があれば、アクセス不可の候補はそれに負ける。

### 3.3 `LookupOptions` と適格性検査

- `LookupOptions`（`LookupOptions.cs`）：`NamespacesOrTypesOnly`、`MustBeInvocableIfMember`、`MustBeInstance`/`MustNotBeInstance`、`AllMethodsOnArityZero`、`LabelsOnly`、`AttributeTypeOnly`、`MustNotBeMethodTypeParameter`、`MustBeOperator` など。
- `CheckViability`（`Binder_Lookup.cs:1487`）は、エイリアスを外したうえで次の順に検査する。
  1. オプション
  2. `file` ローカル型のスコープ（`IsInScopeOfAssociatedSyntaxTree`, `:1439`。**範囲外なら「アクセス不可」ではなく「存在しない」扱い**）
  3. `[Embedded]`
  4. 引数の数（WrongArity）
  5. 名前で参照可能か
  6. 型か名前空間か
  7. 呼び出し可能か
  8. **アクセス可能性**
  9. static と instance の不一致

### 3.4 型の中のメンバー探索と隠蔽

`LookupMembersInType`（`:280`）は型の種類で分岐する。

- クラスなど → `LookupMembersInClass`（`:880`）
- インターフェイス → `LookupMembersInInterface`（`:1252`）
- 型パラメータ → `LookupMembersInTypeParameter`（`:1270`）
- エラー型 → `LookupMembersInErrorType`（`:321`）

クラスでは基底型をたどり、各段で `MergeHidingLookupResults`（`:1326`）を適用する。規則は「**メソッドはメソッド以外を隠し、メソッド以外は全てを隠す**」。シグネチャによる隠蔽は、オーバーロード解決側で扱う。基底型の循環は `ERR_CircularBase`。

型や名前空間の Viable な結果が複数ある場合は、`ResultSymbol`（`Binder_Symbols.cs:1996`）が決める。ソースのシンボルが参照アセンブリのものに勝つ（`WRN_SameFullNameThisAggAgg` など）。それ以外は `ERR_AmbigContext` などのエラーにする。

### 3.5 アクセス可能性（`Semantics/AccessCheck.cs`）

- `IsAccessible`（`Binder_Lookup.cs:1773-1810`）：`IgnoreAccessibility` フラグがあれば常に真。そうでなければ仮想メソッド `IsAccessibleHelper` をチェーンの外へたどり（型の `InContainerBinder` → … → `BuckStopsHereBinder`）、最後に `AccessCheck.IsSymbolAccessible`（`:24/37/52`）を呼ぶ。
- 本体の `IsSymbolAccessibleCore`（`:133`）は**割り当てもイテレータも使わない**実装（頻繁に呼ばれるため）。
  - 配列・ポインタは要素型の判定に帰着させる。型パラメータ・ローカル・名前空間は常に可。
  - 型 → `IsNamedTypeAccessible`（`:223`）。型引数も全て判定する。
  - メンバー → `IsMemberAccessible`（`:297`）→ `IsNonPublicMemberAccessible`（`:342`）。
    - `private` → 同じ型の中にあるか。
    - `internal` → `HasInternalAccessTo`（同じアセンブリ、`InternalsVisibleTo`、対話型）。
    - `private protected` → internal **かつ** protected。
    - `protected internal` → internal **または** protected。
    - `protected` → `IsProtectedSymbolAccessible`（`:416`）。経由する型（through type）が現在の型を継承していなければ失敗し、`ERR_BadProtectedAccess` になる。
- 失敗すると `LookupResult.Inaccessible(symbol, diag)` になる（`ERR_BadAccess` など）。

### 3.6 using・`ImportChain`・未使用 using の検出

- `Imports`（`Imports.cs:25`）：エイリアスの辞書、using の配列、extern alias の配列。データの出どころは `SourceNamespaceSymbol.AliasesAndUsings`。`global using` は全ての構文木から集める。
- `WithUsingNamespacesAndTypesBinder.LookupSymbolsInSingleBinder`（`:117`）は using の各対象を探索する。名前空間の中の名前空間は対象にしない。`using static` は static メンバーと入れ子の型だけを対象にする。Viable なものが見つかったら `MarkImportDirective` を呼ぶ。
- `MarkImportDirective`（`Binder_Lookup.cs:490`）は `Compilation.MarkImportDirectiveAsUsed` を呼ぶ。ただし**呼び出し元が SemanticModel 用のバインダのときは記録しない**。IDE の問い合わせで結果が変わらないようにするため。最後に `ReportUnusedImports`（`CSharpCompilation.cs:2741`）が `HDN_UnusedUsingDirective` を報告する。
- `ImportChain`（`ImportChain.cs:17`）：PDB に書き出す using の情報。`MethodCompiler` が `bodyBinder.ImportChain` から取得する。

### 3.7 拡張メンバー探索

- `ExtensionScopes`（`ExtensionScope.cs:28`）は、チェーンの中で `SupportsExtensions` が真の環（`InContainerBinder` と `WithUsingNamespacesAndTypesBinder`）だけを内側から列挙する。
- 呼び出し側は**スコープを 1 つずつ試し、最初に成功したスコープで止まる**。例：`ResolveExtension`（`Binder_Expressions.cs:8760` / `:8802`）、拡張インデクサ（`:9218`）、拡張演算子（`Binder_Operators.cs:596, 2111`）。オーバーロード解決自体はスコープを意識しない（`Options.IsExtensionMethodResolution` を受け取るだけ）。
- C# 14 の `extension` ブロック（拡張プロパティ・演算子など）も同じスコープの仕組みで探索する（`LookupAllExtensionMembersInSingleBinder`, `:186`）。

### 3.8 型名のバインド（`Binder_Symbols.cs`）

- `BindType`（`:305`）→ `BindNamespaceOrTypeOrAliasSymbol`（`:429`）が構文の種類で分岐する（`?`、組み込み型、識別子、ジェネリック名、`::`、修飾名、配列、ポインタ、関数ポインタ）。
- **`var`**：`BindTypeOrAliasOrVarKeyword`（`:109`）は、まず `var` という名前の型を探す。**Viable な型が見つかればそれを使い**、見つからなければキーワードとして扱う。
- **`dynamic`・`nint`・`nuint`** も同様に、その名前の型が見つからなかった場合だけ特別な型になる（`:896-1012`）。
- **Color Color 規則**：`Color Color;` のように、型名と同じ名前のメンバーがある場合、`Color.X` の左辺が型か値かは X を探索するまで決まらない。`BindLeftOfPotentialColorColorMemberAccess`（`Binder_Expressions.cs:7694`）は `BoundTypeOrValueExpression`（`:7783`）を作って判断を保留し、メンバー探索の後で `ReplaceTypeOrValueReceiver` が確定させる。

---

## 4. 式のバインド（`Binder_Expressions.cs`、12,565 行）

### 4.1 入口

| メソッド | 行 | 役割 |
|---|---|---|
| `BindExpression(node, diagnostics)` | 568 | 公開の入口 → `BindExpressionInternal` |
| `BindExpressionInternal` | 611 | `switch (node.Kind())`（`:645-880`）で種類別に振り分け |
| `BindValue(node, diagnostics, BindValueKind)` | 260 | `BindExpression` ＋ `CheckValue`（左辺値・参照などの要件） |
| `BindRValueWithoutTargetType` | 266 | `BindToNaturalType(BindValue(…, RValue))` |
| `BindToNaturalType` | 311 | 未変換の式を自然型で確定させる唯一の場所（§5） |
| `BindToTypeForErrorRecovery` | 288 | エラー回復用。未変換の式を必ず確定させる |

`switch` の主な振り分け：

- 識別子 → `BindIdentifier`（`:1643`）
- `.` → `BindMemberAccess`（`:7636`）
- 呼び出し → `BindInvocationExpression`（`Binder_Invocation.cs:183`）
- `new T(…)` → `BindObjectCreationExpression`（`:5289`）、`new(…)` → `BindImplicitObjectCreationExpression`（`:5272`）
- 二項演算子 → `BindSimpleBinaryOperator`、`&&`/`||` → `BindConditionalLogicalOperator`
- `?:` → `BindConditionalOperator`、`switch` 式 → `BindSwitchExpression`
- リテラル → `BindLiteralConstant`（`:7539`）
- タプル → `BindTupleExpression`（`:1037`）、コレクション式 → `BindCollectionExpression`（`:5349`）
- ラムダ → `BindAnonymousFunction`、`await` → `BindAwait`、クエリ → `BindQuery`
- その他 → `BadExpression`

### 4.2 名前とメンバーアクセス

- **`BindIdentifier`**：`LookupIdentifier`（`:1897`）で探索する。
  - 呼び出される名前には `MustBeInvocableIfMember` を付ける。
  - メソッドなら `BoundMethodGroup`。暗黙の `this` が受け手になる場合は `HasImplicitReceiver` を立てる。
  - それ以外は `BindNonMethod`（`:2058`）でローカル、パラメータ、フィールド、プロパティ、イベント、型などのノードにする。
- **`BindMemberAccessWithBoundLeft`**（`:7904`）は左辺の種類で分岐する。
  - 名前空間 → 名前空間のメンバー
  - 型 → static メンバー（`BindMemberOfType`, `:8637`）
  - 値（または型か値か未定） → インスタンスメンバー（`BindInstanceMemberAccess`, `:8313`）
  - `dynamic` → `BindDynamicMemberAccess`
  - 値の場合はメソッドグループに `SearchExtensions` を付け、後で拡張メソッドも探す。
- 要素アクセス `BindElementAccess`（`:10292`）：配列、ポインタ、インデクサ、インライン配列、Index/Range パターン。

### 4.3 値の種類の検査：`BindValueKind` と `CheckValue`（`Binder.ValueChecks.cs`）

```csharp
[Flags] internal enum BindValueKind : ushort     // :471
{ RValue, Assignable, RefersToLocation, RefAssignable,   // 4 つの能力ビット
  CompoundAssignment = RValue | Assignable, RefOrOut = RefersToLocation | RValue | Assignable, ReadonlyRef, AddressOf, RefReturn, ... }
```

- `CheckValue`（`:729`）の流れ：
  - プロパティグループをインデックス付きプロパティとして解決する。
  - **未変換ノードは RValue 要求なら素通しする**（`:816-825`）。
  - メソッドグループは `RValueOrMethodGroup` 以外なら `ResolveMethodGroup` で解決する。
  - 最後に `CheckValueKind`（`:921`）で、代入可能か、参照を取れるか、プロパティの getter/setter があるかを検査する。
- 参照の安全性（ref safety）：エスケープ範囲（`SafeContext`：`CallingMethod`/`ReturnOnly`/`CurrentMethod`/`Empty`）の計算は `GetRefEscape`/`GetValEscape`（`:3675`/`:4323`）にある。検査の多くは、現在は別のパス `RefSafetyAnalysis.cs` に移っている。

### 4.4 エラー回復と連鎖エラーの抑止

- `BadExpression(…)`（`:118-196`）は `BoundBadExpression` を作る。子ノードは `BindToTypeForErrorRecovery` で確定させるので、**未変換ノードが木に漏れない**。型は `CreateErrorType()`（エラー型）。
- 「既にエラーを含む式には追加の診断を出さない」規則があちこちにある。
  - `GenerateConversionForAssignment` は、式が `HasAnyErrors` なら診断を捨てる（`Binder_Statements.cs:1979-1982`）。**ただしラムダは除外**して、IDE の補完が `x => x.` を部分的にでもバインドできるようにする。
  - `BindSimpleBinaryOperator` は、オペランドにエラーがあれば演算子のオーバーロード解決を行わない（`Binder_Operators.cs:945`）。

---

## 5. 未変換（target-typed）の式と変換の適用

### 5.1 二段構え

- **型が文脈で決まる式は、まず未変換ノードとして作る**。`BoundExpression.NeedsToBeConverted()`（`BoundTree/BoundExpression.cs:77`）が真になるのは次のもの。
  - `TupleLiteral`、`UnconvertedSwitchExpression`、`UnconvertedObjectCreationExpression`（`new()`）
  - `UnconvertedConditionalOperator`、`DefaultLiteral`、`UnconvertedInterpolatedString`、`UnconvertedCollectionExpression`
  - 型未定の `stackalloc`、補間文字列の未変換 `+`
- 例：`BindImplicitObjectCreationExpression` は引数だけをバインドして `BoundUnconvertedObjectCreationExpression` を返す。**コンストラクタのオーバーロード解決は、型が決まった時点（`ConvertObjectCreationExpression`, `Binder_Conversions.cs:818`）まで延期**する。
- `?:` は、2 つの分岐に最良の共通型（`BestTypeInferrer.InferBestTypeForConditionalOperator`）があれば、その場で両方を変換する。無ければ `BoundUnconvertedConditionalOperator` を作り、`ERR_AmbigQM` などの診断を後で使えるように保存しておく（`Binder_Operators.cs:5975`）。

### 5.2 確定させる 2 つの道

1. **目的の型がある**（代入、引数、戻り値、キャスト…）：`ClassifyConversionFromExpression` が特別な変換の種類（`SwitchExpression`、`ConditionalExpression`、`ObjectCreation`、`CollectionExpression`、`InterpolatedStringHandler`、`DefaultLiteral` など）を返し、`CreateConversion` がノードを目的の型に合わせて作り直す。
2. **目的の型が無い**（`var x = …;`、式文…）：`BindToNaturalType`（`:311`）が自然型で確定させる。自然型が無ければ `ERR_DefaultLiteralNoTargetType`、`ERR_ImplicitObjectCreationNoTargetType`、`ERR_CollectionExpressionNoTargetType` などを報告する。

DEBUG ビルドでは、**全ての右辺値が「変換」か「自然型への確定」のどちらかを必ず 1 回通る**ことを `WasConverted` フラグで検査する（`BindToNaturalType` の最後が `result?.WithWasConverted()`）。

### 5.3 `GenerateConversionForAssignment` と `CreateConversion`

- `GenerateConversionForAssignment`（`Binder_Statements.cs:1963`）：代入、戻り値、引数などで使う標準の暗黙変換。
  1. 変換を分類する。
  2. ref 代入なら恒等変換を要求する。
  3. 暗黙変換でなければ `GenerateImplicitConversionError` を報告する。
  4. `CreateConversion` を呼ぶ。
- `CreateConversion`（`Binder_Conversions.cs:54`）は変換の種類で分岐する。
  - 恒等 → 自然型で確定するだけ
  - メソッドグループ → `CreateMethodGroupConversion`（`:2929`）
  - ラムダ → `CreateAnonymousFunctionConversion`（`:2898`。ここで `unboundLambda.Bind(デリゲート型)` が呼ばれる）
  - 関数型 → `CreateFunctionTypeConversion`
  - stackalloc → `CreateStackAllocConversion`
  - タプル → `CreateTupleLiteralConversion`
  - `switch` 式 / `?:` → 各アームを目的の型へ変換（`ConvertSwitchExpression`/`ConvertConditionalExpression`）
  - 補間文字列 → `FormattableString` 化、またはハンドラ化
  - `new()` → `ConvertObjectCreationExpression`
  - コレクション式 → `ConvertCollectionExpression`（`:885`。配列、Span、`[CollectionBuilder]`、`IEnumerable` 実装型などで分岐）
  - ユーザー定義変換 → `CreateUserDefinedConversion`（`:2518`）
  - それ以外 → `new BoundConversion(…)`
- ユーザー定義変換は「元の型 → 演算子の引数型」「演算子の呼び出し」「戻り値型 → 目的の型」の 3 段の `BoundConversion` を入れ子にし、1 つの `ConversionGroup` を共有させる。

---

## 6. 変換の分類（`Semantics/Conversions/`）

### 6.1 クラス構成

- `ConversionsBase`（4,067 行）：言語仕様の変換規則の本体。
- `Conversions : ConversionsBase`：バインダに依存する部分。メソッドグループ変換（中でオーバーロード解決を行う）、補間文字列、コレクション式。
- `TypeConversions`：バインダに依存しない、シンボルの検査用。
- `WithNullability(bool)` は、null 許容性を考慮する版／しない版の対になるインスタンスを 1 回だけ作ってキャッシュする（`_lazyOtherNullability`）。

### 6.2 「式からの変換」と「型からの変換」

| メソッド | 行 | 意味 |
|---|---|---|
| `ClassifyImplicitConversionFromExpression` | 98 | 恒等の高速判定 → 式に依存する組み込み変換（定数、`null`、ラムダ、メソッドグループ、タプルリテラル、補間文字列、コレクション式、`default`、`new()`） → EasyOut 表 → 標準変換の詳細判定 → 型の無い式なら関数型 → ユーザー定義 → `switch` 式 / `?:`（最も低い優先度） |
| `ClassifyImplicitConversionFromType` | 169 | 型だけで判定（恒等 → EasyOut → 標準 → ユーザー定義） |
| `ClassifyConversionFromExpressionType` | 234 | 合成した式用。式からの暗黙の dynamic 変換を加える |
| `ClassifyStandardConversion` | 529 | 標準変換（ユーザー定義変換の前後に使える変換） |

**定数（`int` 定数 → `byte`）、ラムダ、`null` のように、式でなければ決まらない変換がある**ため、2 つの系統がある。標準の暗黙変換は、恒等、数値、null 許容、参照（配列、デリゲート、分散 `HasVariantConversion`）、ボックス化、ポインタ、タプル、Span の順に判定する（`ClassifyStandardImplicitConversion`, `:695`）。

### 6.3 ユーザー定義変換（`UserDefinedImplicitConversions.cs`、仕様 §10.5.4）

1. 対象となる型の集合 D を作る：元の型とその基底型、目的の型（`ComputeUserDefinedImplicitConversionTypeSet`, `:118`）。
2. 適用可能な演算子の集合 U を作る：持ち上げ（lifted）形も含める（`:142`）。
3. U が空なら変換は無い。
4. 最も特殊な元の型 SX（`:365`）と目的の型 TX（`:381`）を決める。
5. `MostSpecificConversionOperator`（`:428`）で演算子を 1 つに絞る。持ち上げの段数で同点を崩す。

結果は `Valid`、`Ambiguous`、`NoBestSourceType`、`NoBestTargetType` のいずれか。**あいまいなユーザー定義変換は「存在する（`Exists`）が有効でない（`IsValid` が偽）」**として扱う。組み込みの暗黙変換は、呼び出し側が先に調べるので、ユーザー定義変換より優先される。

### 6.4 `Conversion` 構造体（`Conversion.cs:19`）

```csharp
public readonly struct Conversion { private readonly ConversionKind _kind; private readonly UncommonData? _uncommonData; }
```

普通は enum 1 つと null だけ。ユーザー定義演算子、入れ子の変換（null 許容、タプル）、分解、コレクション式などの付加情報は `UncommonData` の派生クラスに追い出し、省メモリにしている。

### 6.5 EasyOut 表（`ConversionEasyOut.cs`）

`object`、`string`、`bool`、`char`、整数型全般、`nint`/`nuint`、`float`、`double`、`decimal` と、それぞれの null 許容型（計 32 種）について、**32×32 の `byte[,]` に変換の種類を事前に持つ**。組み込み型同士の変換は表を引くだけで済む（`FastClassifyConversion`, `ConversionsBase.cs:488`）。

---

## 7. 呼び出しとオーバーロード解決

### 7.1 呼び出しのバインド（`Binder_Invocation.cs`）

```text
BindInvocationExpression(syntax)                          :183
  ├ nameof なら TryBindNameofOperator                       :2396
  ├ a.M().N().O() の連鎖は明示スタックで反復（深い再帰を避ける）
  └ BindArgumentsAndNames → BindInvocationExpression(core)  :327
       ├ dynamic → BindDynamicInvocation                     :399
       ├ メソッドグループ → BindMethodGroupInvocation          :698
       │    ResolveMethodGroup (Binder_Expressions.cs:11594)
       │      ├ ResolveDefaultMethodGroup → OverloadResolution.MethodInvocationOverloadResolution
       │      └ 適用可能なものが無く SearchExtensions なら ResolveExtension（スコープを内→外）  :8760
       ├ デリゲート → BindDelegateInvocation                  :629
       └ 関数ポインタ → BindFunctionPointerInvocation          :2570
  → BindInvocationExpressionContinued                        :1163
       ReplaceTypeOrValueReceiver（Color Color の確定）
       CheckAndCoerceArguments（各引数を最も基底のメンバーの引数型へ CreateConversion、params の収集）  Binder_Expressions.cs:3501
       BindDefaultArguments（省略された省略可能引数・Caller 情報属性・空の params）                     :1566
       ReportDiagnosticsIfObsolete、UnmanagedCallersOnly、Finalize の検査
       → new BoundCall(...)                                   :1390
```

- 適用可能な候補が `dynamic` 引数を取る場合は `BoundDynamicInvocation` にし、実行時のバインドに回す。
- 失敗したときは `CreateBadCall`（`:2076`）と `BuildArgumentsForErrorRecovery`（`:2155-2353`）で、引数を「最もそれらしい引数型」に変換する。これにより、引数の中のラムダも IDE 向けにバインドされる。

### 7.2 オーバーロード解決（`Semantics/OverloadResolution/OverloadResolution.cs`、4,796 行）

**入口**：`MethodInvocationOverloadResolution`（`:137`）、`ObjectCreationOverloadResolution`（`:101`）、`PropertyOverloadResolution`（`:164`）、`FunctionPointerOverloadResolution`（`:556`）。共通の処理は `MethodOrPropertyOverloadResolution<TMember>`（`:185`）。

**2 パス方式**：まず `completeResults: false` で解決する。成功しなかった場合（`OverloadResolutionResultIsValid`, `:320`）だけ、結果を消して `completeResults: true` で解決し直し、診断のために全候補の失敗理由を集める（`:206-221`）。

**手順**（`PerformMemberOverloadResolution`, `:427`）：

1. 候補が 50 個を超える場合は、宣言している型ごとに分けて、オーバーライドと隠蔽の検査を O(n²) にしないようにする（`:371`）。
2. `AddMemberToCandidateSet`（`:1058`）：
   - より派生したオーバーライドがあれば除外する（**最も派生したメンバーを残す、意図的な仕様違反**。IDE とコード生成がオーバーライド先を見られるようにするため。`:1074-1101` のコメント）。
   - 名前による隠蔽も除外する。
   - 通常形で適用可能か（`IsMemberApplicableInNormalForm`, `:4109`）を調べ、駄目なら params の展開形（`IsMemberApplicableInExpandedForm`, `:4187`）を調べる。
3. 推論された型引数のアクセス可能性（`RemoveInaccessibleTypeArguments`）。
4. 基底型のメンバーを劣位として印を付ける（`RemoveLessDerivedMembers`）。
5. 新しい言語バージョンでは、さらに static/instance の不一致、制約違反、メソッドグループ変換での戻り値型の不一致を除外する。
6. `[OverloadResolutionPriority]` による絞り込み（`RemoveLowerPriorityMembers`, `:1858`。属性が無ければ割り当てなしで素通りする）。
7. **より良い関数メンバー**の選択（`RemoveWorseMembers`, `:1950`）：まず O(n) で暫定首位を探して検証し、決まらなければ O(n²) の比較表で `Worse`/`Worst` を付ける。

**適用可能性**（`IsApplicable`, `:4250` / `:4491`）：

- ジェネリックで型引数が明示されていなければ `MethodTypeInferrer.Infer`（§8）で推論し、構築した後、**全ての**仮引数型について制約を検査する。
- 引数ごとに `CheckArgumentForApplicability`（`:4696`）を呼ぶ。
  - ref の種類が一致しなければ不可。
  - 値引数は `ClassifyImplicitConversionFromExpression`。`ref`/`out` は恒等変換を要求する。
  - `completeResults` でなければ、最初の不適合で打ち切る。
- 引数と仮引数の対応（名前付き引数、位置、省略）は `OverloadResolution_ArgsToParameters.cs` の `AnalyzeArguments`（`:65`）が決める。

**より良い関数メンバー**（`BetterFunctionMember`, `:2097`）：

1. 関数型（ラムダの自然型）を必要としない候補を優先する。
2. 引数ごとに `BetterConversionFromExpression`（`:2886`）で比べる。
   - 恒等なら引き分け。
   - 非定数の補間文字列ではハンドラ変換が勝つ。
   - 関数型以外の変換が関数型に勝つ。
   - `ExpressionMatchExactly`（`:3267`）。
   - コレクション式の規則。
   - 暗黙の Span 変換は負ける。
   - それ以外は `BetterConversionTarget`（`:3468`。`ReadOnlySpan` が `Span` に勝つ、暗黙変換の方向、`Task<T>` の中身、デリゲートの戻り値）。
3. 引数型の並びが同じ場合の同点決着：
   - 非ジェネリックがジェネリックに勝つ。
   - 通常形が展開形に勝つ。
   - 既定値に頼らない方が勝つ。
   - より特殊な仮引数型が勝つ。
   - カスタム修飾子が少ない方が勝つ。
   - params コレクションの優劣。

**結果と診断**（`OverloadResolutionResult.cs`）：`ReportDiagnostics`（`:205`）は、優先度の高い順に 1 つの原因だけを報告する。

1. あいまい（`ERR_AmbigCall`）
2. static/instance の不一致、戻り値の不一致、制約違反
3. 引数の型の不一致（`ERR_BadArgType`）
4. 型推論の失敗（`ERR_CantInferMethTypeArgs`）
5. 使用場所のエラー
6. 引数の数・名前の誤り（`ERR_BadArgCount`、`ERR_BadNamedArgument` …）

---

## 8. 型推論（`Semantics/OverloadResolution/MethodTypeInference.cs`、3,534 行）

```csharp
internal struct MethodTypeInferrer                                            // :91
{
    private readonly (TypeWithAnnotations Type, bool FromFunctionType)[] _fixedResults;
    private readonly HashSet<TypeWithAnnotations>[] _exactBounds, _upperBounds, _lowerBounds;
    private Dependency[,] _dependencies;   // 型パラメータ間の依存関係（遅延構築）
    public static MethodTypeInferenceResult Infer(Binder, ConversionsBase, ImmutableArray<TypeParameterSymbol> methodTypeParameters,
        NamedTypeSymbol constructedContainingTypeOfMethod, ImmutableArray<TypeWithAnnotations> formalParameterTypes,
        ImmutableArray<RefKind>, ImmutableArray<BoundExpression> arguments, ref useSiteInfo, Extensions extensions = null, ...)   // :172
}
```

仕様の 2 段階アルゴリズム：

- **第 1 段階**（`InferTypeArgsFirstPhase`, `:601`）：引数ごとに「明示的な引数型推論」を行う。
  - `ref`/`out`/`in` は完全一致（Exact）、それ以外は下限（LowerBound）。
  - ラムダは、パラメータ型が明示されていればそこから推論する（`ExplicitParameterTypeInference`）。
  - コレクション式は要素ごと、タプルリテラルは要素ごとに推論する。
  - 型の無い `null` は、null 許容注釈の下限だけに寄与する。
- **第 2 段階**（`InferTypeArgsSecondPhase`, `:765`）：依存関係のグラフを作り、反復する（`DoSecondPhase`, `:815`）。
  1. 入力型が確定していて、出力型に未確定の型パラメータを含む引数（主にラムダとメソッドグループ）について「出力型推論」を行う（`MakeOutputTypeInferences`, `:863`）。
     - ラムダは、確定したパラメータ型でバインドして戻り値型を推論する（`InferredReturnTypeInference`）。
     - メソッドグループは、確定したデリゲートの引数型でオーバーロード解決して戻り値型を得る（`MethodGroupReturnTypeInference`）。
  2. 依存されていない型パラメータを確定させる（`FixNondependentParameters`）。無ければ依存している側を確定させる。
  3. 進展が無ければ推論は失敗。
- **確定（Fix）**（`:2827`）：
  - 下限・上限・完全一致の候補の中から、全ての候補が暗黙変換できる唯一の候補を選ぶ。
  - 分散を考慮して候補をまとめる（`AddOrMergeCandidate`）。関数型以外の候補があれば、関数型の候補は捨てる。
  - ソースにも「このメソッドは大量のゴミを作る（UNDONE: This method makes a lot of garbage）」というコメントがある（`:2869`）。
- **NullableWalker との連携**：`Extensions` 抽象クラス（`:93-113`）を差し替えると、**フロー状態を反映した型で推論をやり直せる**。`NullableWalker.MethodInferenceExtensions` がこれを使う（データフロー解析ドキュメント §8.5）。

`BestTypeInferrer`（`Semantics/BestTypeInferrer.cs`）は「最良の共通型」を求める。暗黙型付け配列、ラムダの複数の return、`?:` で使う。優劣が推移的でなくても、O(n) の暫定首位探索と検証で唯一の最良型を見つける（`GetBestType`, `:177`）。

---

## 9. 演算子（`Binder_Operators.cs` ＋ `Semantics/Operators/`）

### 9.1 バインド

- **`BindSimpleBinaryOperator`**（`Binder_Operators.cs:883`）：`a + b + c + …` の左再帰を**明示スタックで反復処理**する。機械生成コードの数千項の連結でもスタックが溢れない。`&&`/`||` も同様。
- 処理本体（`:936`）：
  - オペランドにエラーがあれば打ち切る。
  - `dynamic` → `BindDynamicBinaryOperator`。
  - それ以外は `BinaryOperatorOverloadResolution`。
  - 結果に変換を適用して `BoundBinaryOperator` を作る。
- **定数畳み込み**（`FoldBinaryOperator`, `:2753`）：
  - 0 除算 → `ERR_IntDivByZero`。
  - checked 文脈で桁あふれ → `ERR_CheckedOverflow`。
  - unchecked なら桁あふれを許して計算する（`FoldUncheckedIntegralBinaryOperator`）。
  - enum と null 許容型の比較は専用の処理。

### 9.2 演算子の種類のエンコード（`Semantics/Operators/OperatorKind.cs`）

```text
BinaryOperatorKind = 型(下位8bit: Int=5, Long, ..., String, Enum, Pointer, UserDefined, Dynamic, ...)
                   | 演算(0xFF00: Multiplication=0x1000, Addition=0x1100, ..., Equal=0x1700, ...)
                   | Lifted(0x10000) | Logical(0x20000) | Checked(0x40000)
例: IntAddition = Int | Addition
```

1 つの enum 値で「どの型のどの演算か、持ち上げ・checked か」を表す。BoundTree の `BoundBinaryOperator.OperatorKind` にそのまま入り、Lowering と IL 生成もこの値で分岐する。

### 9.3 演算子のオーバーロード解決（`BinaryOperatorOverloadResolution.cs`）

1. **EasyOut**（`BinaryOperatorEasyOut.cs:286`）：両オペランドが組み込み型なら、`s_opkind[演算][左, 右]` の表を引くだけで決まる。`&&` は `&` として解決する。
2. 決まらなければ：
   - **クラス・構造体のユーザー定義演算子**を、左オペランドの型から基底型へ探す。最初に候補が見つかった型で止める。組み込み型にはユーザー定義演算子が無いことを利用して探索を省く（`OperatorFacts.DefinitelyHasNoUserDefinedOperators`）。
   - 見つからなかった場合**だけ**、インターフェイスの演算子を探す。
   - **最後に組み込み演算子**を集める。
   - `BetterOperator`（`:1223`）で最良のものを選ぶ。
3. 持ち上げ（`int?` の `+` など）は `UserDefinedBinaryOperatorCanBeLifted`（`:1055`）で判定する。
4. C# 14 の拡張演算子は `ExtensionScopes` で 1 スコープずつ解決する（`:1431`）。インスタンスの複合代入演算子（`op_AdditionAssignment` など）があれば、そちらを先に試す（`Binder_Operators.cs:348, 593-599`）。

組み込み演算子の表は、コンパイルごとに 1 つの `BuiltInOperators`（`Compilation/BuiltInOperators.cs`）が遅延構築してキャッシュする。

---

## 10. ラムダ（`Binder_Lambda.cs` ＋ `BoundTree/UnboundLambda.cs`）

- `BindAnonymousFunction`（`Binder_Lambda.cs:365`）は、パラメータ・修飾子・属性・既定値・明示の戻り値型だけを解析して **`UnboundLambda` を返す。本体はまだバインドしない**。本体の意味（選ばれるオーバーロード、変換、エラー）はパラメータ型によって変わり、パラメータ型は目的のデリゲート型が決まるまで分からないため。
- `UnboundLambdaState`（`UnboundLambda.cs:536`）には 3 つのキャッシュがある。

```csharp
ImmutableDictionary<(NamedTypeSymbol Type, bool IsExpressionLambda), BoundLambda>? _bindingCache;   // :544 最終バインド（デリゲート型ごと）
ImmutableDictionary<ReturnInferenceCacheKey, BoundLambda>? _returnInferenceCache;                  // :549 戻り値型推論用
BoundLambda? _errorBinding;                                                                         // :551 エラー回復用
```

- **何度もバインドし直す理由**：
  - オーバーロード解決では、候補ごとにデリゲートの引数型が違う。
  - 型推論では、部分的に分かったデリゲート型で戻り値型を推論する必要がある。
  - 最終的な変換では、選ばれたデリゲート型でバインドする。
  - IDE のエラー回復用にも別のバインドが要る。
- `ReturnInferenceCacheKey` は引数型・ref 種別・戻り値の Task 型だけをキーにする。シグネチャが同じなら、デリゲート型が違っても 1 つのバインド結果を共有する。最終バインドでも、戻り値推論で式本体をバインドした結果を再利用する（`ReallyBind`, `:818-849`）。
- **自然な関数型**：`var f = (int x) => x;` のための `FunctionTypeSymbol`（`Symbols/FunctionTypeSymbol.cs:35`）。関数のシグネチャを型として扱える仮の `TypeSymbol` で、変換、`BestTypeInferrer`、型推論の中だけで使い、シンボルモデルには漏らさない。最終的に `Func<>`/`Action<>` か匿名デリゲート型になる（`InferDelegateType`, `:738`）。
- `BindWithLambdaBindingCountDiagnostics`（`Binder_Lambda.cs:426`。`MethodCompiler.cs:1942` から呼ばれる）は、`[ThreadStatic]` の辞書でラムダごとのバインド回数を数え、**100 回を超えたラムダに情報診断 `INF_TooManyBoundLambdas` を出す**（`:447-452`）。適用可能なオーバーロードの数と、入れ子になったラムダのオーバーロードの数の積で回数が増える、という問題をユーザーに知らせるため。

---

## 11. 文のバインド（`Binder_Statements.cs`、4,293 行）

| 文 | メソッド | 要点 |
|---|---|---|
| 振り分け | `BindStatement`（`:40`） | 約 25 種類の `switch`。ローカル関数とブロック以外に付いた属性は `ERR_AttributesNotAllowed` |
| ブロック | `BindBlock`（`:1909`） | 対応表の `BlockBinder` で `BindBlockParts` |
| ローカル変数宣言 | `BindVariableDeclaration`（`:954`） | `var` なら `BindInferredVariableInitializer`（`:878`）で自然型にする。`null`、`void`、自然型の無いラムダやメソッドグループは `ERR_ImplicitlyTypedVariableAssignedBadValue`。型が明示されていれば `GenerateConversionForAssignment`。`out var` と分解の変数は `VariablePendingInference` で型が決まるまで待つ |
| 式文 | `BindExpressionStatement`（`:644`） | 呼び出し・代入・増減・`await`・オブジェクト生成以外は `ERR_IllegalStatement`。待機されない awaitable は `WRN_UnobservedAwaitableExpression` |
| `if` | `BindIfStatement`（`:2580`） | `else if` の連鎖は明示スタック。条件は `BindBooleanExpression`（`:2650`。`bool` への暗黙変換、無ければ `operator true`） |
| `while`/`do`/`for` | 仮想メソッド `BindWhileParts` など（`:2814-2839`） | `WhileBinder`/`ForLoopBinder` がオーバーライドする。break/continue のラベルはループバインダが持つ |
| `foreach` | `ForEachLoopBinder.BindForEachPartsWorker`（`ForEachLoopBinder.cs:216`） | §11.1 |
| `using`/`lock` | `UsingStatementBinder`/`LockBinder` | Dispose パターンか `IDisposable`/`IAsyncDisposable`。`System.Threading.Lock` 型の特別扱い（`TryFindLockTypeInfo`） |
| `return` | `BindReturn`（`:3061`） | 戻り値型の取得、イテレータの中なら `ERR_ReturnInIterator`、ref の一致、`void`/`Task` で値を返すとエラー。`CreateReturnConversion`（`:3200`）は async なら `Task<T>` を `T` に剥がす。戻り値を推論中のラムダでは検査を最小限にする |
| `throw`/`try` | `BindThrow`（`:434`）/ `BindTryStatement`（`:3290`） | `throw;` の位置検査、例外フィルタ |
| `switch` 文 | `SwitchBinder_Patterns.BindSwitchStatementCore` | ラベル、`goto case`、決定 DAG（§12） |
| `yield` | `BindYieldReturnStatement`（`:234`） | 要素型への変換。finally・catch・unsafe の中なら禁止 |

### 11.1 `foreach` の列挙子の決定

`GetEnumeratorInfoCore`（`ForEachLoopBinder.cs:846`）は次の順に試す。

1. インライン配列（Span として列挙）
2. 配列と文字列（`GetDefaultEnumeratorInfo`）
3. **パターン**：`GetEnumerator()`、`Current`、`MoveNext()` を名前で探す（`SatisfiesGetEnumeratorPattern`, `:1354`）
4. `IEnumerable<T>`/`IEnumerable` インターフェイス
5. 拡張メソッドの `GetEnumerator`

Dispose の要否も含めて `ForEachEnumeratorInfo`（`ForEachEnumeratorInfo.cs:15`）にまとめ、`BoundForEachStatement` に持たせる。Lowering はこの情報を使って `foreach` を展開する。

---

## 12. パターンと決定 DAG

- `Binder_Patterns.cs`：`BindPattern`（`:220`）がパターンの種類ごとに振り分け、`BoundPattern` を作る。
  - 定数か型名かのあいまいさは `BindConstantPatternWithFallbackToTypePattern`（`:479`）で解く。
  - その他：宣言パターン、`var`、再帰パターン（位置パターンは `Deconstruct` か `ITuple`、プロパティパターン）、関係パターン、`not`/`and`/`or`、リストパターン、スライスパターン。
- `DecisionDagBuilder.cs`（5,044 行）：全ての case を 1 つの**決定 DAG**（有限オートマトン）にまとめる。
  1. 各パターンをテスト（null 判定、型判定、値判定）と評価（プロパティ、`Deconstruct`、インデクサ）の木に分解する（`MakeTestsForPattern`, `:188`）。
  2. 「部分的に一致した case の集合」を状態とし、状態を重複排除しながらワークリストで DAG を作る（`MakeDecisionDag`, `:1292`）。
  3. `CheckConsistentDecision`（`:1959`）で「真の側・偽の側で分かること」を推論し、到達不能な case（包含されている case）や網羅性を判定する。
- 診断：
  - 到達不能なラベル → `ERR_SwitchCaseSubsumed`（`SwitchBinder_Patterns.cs:77-125`）。
  - `switch` 式の到達不能なアーム → `ERR_SwitchArmSubsumed`。
  - 網羅されていない → `WRN_SwitchExpressionNotExhaustive`。`PatternExplainer.cs` が一致しない値の例を作る。
- この DAG は、Lowering（`switch` の展開）、NullableWalker（データフロー解析ドキュメント §8.6）、到達可能性の判定でも使われる。

---

## 13. その他の構文

| ファイル | 内容 |
|---|---|
| `Binder_Await.cs` | `await` の awaitable パターン（`GetAwaiter` → `IsCompleted` → `GetResult`）を `BoundAwaitableInfo` にまとめる。オペランドは `BoundAwaitableValuePlaceholder` で表す。async の外、lock の中などでの禁止を診断する |
| `Binder_Query.cs` | クエリ式を仕様どおりメソッド呼び出しに**翻訳**する。`where` → `Where`、`from…from` → `SelectMany`（透過識別子）、`let`、`join`、`orderby`、`group`。各句の本体は `UnboundLambda` として作り、通常の呼び出しのバインドとラムダの推論に任せる |
| `Binder_Deconstruct.cs` | `(a, b) = e;` を、タプルの要素ごと、または `Deconstruct` メソッドの呼び出しの入れ子になった分解変換としてバインドする |
| `Binder_InterpolatedString.cs` | 補間文字列を、`string`、`FormattableString`、ハンドラパターン（`(literalLength, formattedCount)` のコンストラクタ ＋ `AppendLiteral`/`AppendFormatted`）のいずれかに変換する |
| `Binder_Attributes.cs` | 属性のコンストラクタ呼び出しと名前付き引数をバインドし、定数であることを検査して `CSharpAttributeData` にする。`EarlyWellKnownAttributeBinder` は宣言の早い段階で必要な属性（`Obsolete` など）を簡易にバインドする |
| `Binder_Crefs.cs` | XML ドキュメントの `cref` の解決 |
| `RefSafetyAnalysis.cs` | `ref struct`/`scoped` の参照の安全性を、バインドの後に別パスで検査する |

---

## 14. 診断と依存関係：`BindingDiagnosticBag`

- `BindingDiagnosticBag`（`BindingDiagnosticBag.cs:13`。基底は `Core/Portable/Binding/BindingDiagnosticBag.cs`）は、**診断に加えて「使ったアセンブリ（依存関係）」も集める**。
- `CompoundUseSiteInfo<AssemblySymbol>`（`Core/Portable/Binding/UseSiteInfo.cs:106`）は構造体で、探索や変換の分類に `ref` で渡す。使用場所の診断（見つからない参照アセンブリなど）と依存関係を溜め、最後に `diagnostics.Add(node, useSiteInfo)` でまとめて移す。投機的な処理（オーバーロード解決の候補など）では `Discarded` を使って捨てる。
- 集めた依存関係は `CSharpCompilation.GetCompleteSetOfUsedAssemblies`（`Symbols/Compilation_UsedAssemblies.cs:63`）が使い、「実際に使われた参照アセンブリ」の判定（未使用参照の報告など）に役立てる。

---

## 15. 性能とスケーラビリティの工夫

| 手法 | 例 |
|---|---|
| 2 パス方式 | 名前探索（`diagnose: false` → 失敗時のみ `true`）、オーバーロード解決（`completeResults: false` → 失敗時のみ `true`） |
| 高速表 | `ConversionEasyOut`（32×32）、`BinaryOperatorEasyOut`、`UnaryOperatorEasyOut`、全ての分類での恒等の先行判定 |
| プール | `LookupResult`、`AnalyzedArguments`、`OverloadResolutionResult<T>`、`ArrayBuilder`、`PooledHashSet`/`PooledDictionary`、`BinderFactoryVisitor` |
| キャッシュ | `BinderFactory` の `ConcurrentCache`、`ExecutableCodeBinder` の対応表、`LocalScopeBinder` の遅延構築、ラムダの 3 つのキャッシュ、`BuiltInOperators` の表、`Conversions` の null 許容性の対 |
| 計算量の上限 | 50 候補を超えたら型ごとに分割、O(n) の暫定首位 → O(n²)、分散変換の再帰深さ 50、`BetterConversionTarget` の再帰上限 100 |
| 深い木 | 二項演算子・`&&`/`||`・`else if`・メソッド呼び出しの連鎖の反復処理 |
| 探索の省略 | 組み込み型のユーザー定義演算子は探さない、インターフェイスの演算子はクラスで見つからない場合だけ、拡張メソッドは通常のメソッドが適用不可の場合だけ探す |
| 割り当ての抑制 | `Conversion`/`BoundBinaryOperator` のまれな情報を外に出す、変換配列は最初の非恒等変換が出てから確保、`AccessCheck` の割り当てなし実装 |

---

## 16. 設計上の要点まとめ

1. **スコープ＝バインダの連結リスト**：名前探索は内から外へたどり、最初に有効な結果が見つかったところで止まる。文脈（unsafe、checked、lock の中、SemanticModel かどうか）もフラグとしてチェーンに乗る。
2. **バインダは事前に作ってキャッシュする**：宣言の部分は `BinderFactory`（構文位置 × 用途のキャッシュ）、本体は `LocalBinderFactory`（1 回の走査で全スコープの対応表）。コンパイラと SemanticModel が同じ仕組みを使う。
3. **型が文脈で決まる式の遅延確定**：未変換ノード → `CreateConversion`/`BindToNaturalType`。`WasConverted` の不変条件で漏れを検出する。
4. **変換は必ずノードとして明示する**：暗黙変換、ユーザー定義変換、メソッドグループ変換、ラムダ変換を全て `BoundConversion` として木に入れる（BoundTree ドキュメント §2.4 の規則と対応）。
5. **仕様に忠実なアルゴリズムと、意図的な逸脱**：オーバーロード解決、型推論、ユーザー定義変換は仕様の手順をほぼそのまま実装している。ただし「最も派生したオーバーライドを残す」「native コンパイラとの互換のための優先度」など、理由付きの逸脱がコメントで明記されている。
6. **ラムダは目的の型ごとにバインドし直す**：キャッシュと自然な関数型でコストを抑える。
7. **IDE を前提にしたエラー回復**：連鎖エラーの抑止、`BoundBadExpression` による部分情報の保持、引数をエラー回復用に変換してラムダもバインドしておく、SemanticModel の問い合わせはコンパイル結果（未使用 using の判定など）に影響させない。

---

## 付録 A: 主要ファイル早見表

| ファイル | 行数 | 役割 |
|---|---:|---|
| `Binder.cs` / `Binder_Flags.cs` / `BinderFlags.cs` | 1,023 / — / — | 基底・チェーン・フラグ |
| `BinderFactory*.cs` | 約 1,700 | 宣言側のバインダ構築とキャッシュ |
| `ExecutableCodeBinder.cs` / `LocalBinderFactory.cs` | — / 1,128 | 本体内のバインダ対応表 |
| `LocalScopeBinder.cs` ほか `*Binder.cs` | — | 各スコープ |
| `Binder_Lookup.cs` / `LookupResult*.cs` / `LookupOptions.cs` | 2,156 / — / — | 名前探索 |
| `Binder_Symbols.cs` | 2,873 | 型名・名前空間のバインド |
| `Binder_Expressions.cs` | 12,565 | 式 |
| `Binder.ValueChecks.cs` | 5,960 | 値の種類・参照の安全性 |
| `Binder_Conversions.cs` | 4,378 | 変換の適用 |
| `Binder_Invocation.cs` | 2,641 | 呼び出し |
| `Binder_Operators.cs` | 6,142 | 演算子・定数畳み込み |
| `Binder_Lambda.cs` / `BoundTree/UnboundLambda.cs` | 483 / 1,668 | ラムダ |
| `Binder_Statements.cs` / `ForEachLoopBinder.cs` | 4,293 / 2,002 | 文 |
| `Binder_Patterns.cs` / `DecisionDagBuilder*.cs` | 2,289 / 約 6,500 | パターン・決定 DAG |
| `Semantics/OverloadResolution/OverloadResolution.cs` / `OverloadResolutionResult.cs` | 4,796 / 1,644 | オーバーロード解決 |
| `Semantics/OverloadResolution/MethodTypeInference.cs` | 3,534 | 型推論 |
| `Semantics/Conversions/ConversionsBase.cs` / `Conversion.cs` | 4,067 / 1,328 | 変換の分類 |
| `Semantics/Operators/BinaryOperatorOverloadResolution.cs` / `OperatorKind.cs` | 1,634 / 773 | 演算子の解決 |
| `Semantics/AccessCheck.cs` | 704 | アクセス可能性 |

## 付録 B: `var s = list.Select(x => x * 2);` のバインドの流れ

```text
BindStatement → BindLocalDeclarationStatement → BindVariableDeclaration (isVar)
  BindInferredVariableInitializer
    BindInvocationExpression
      BindMemberAccess(list.Select)
        BindIdentifier(list) → LookupSymbolsWithFallback: BlockBinder でローカル list を発見 → BoundLocal(List<int>)
        BindInstanceMemberAccess: List<int> に Select は無い → BoundMethodGroup(SearchExtensions)
      BindArgumentsAndNames: x => x * 2 → UnboundLambda（本体はまだバインドしない）
      BindMethodGroupInvocation → ResolveMethodGroup
        通常のメソッドで適用可能なものは無い → ResolveExtension: ExtensionScopes を内→外
          using System.Linq のスコープで Enumerable.Select<TSource,TResult> の 2 つのオーバーロードを発見
          OverloadResolution.MethodInvocationOverloadResolution
            MethodTypeInferrer.Infer
              第1段階: list → TSource の下限 int
              第2段階: TSource=int を確定 → ラムダを Func<int, ?> で戻り値型推論
                       （BindForReturnTypeInference、x*2 → int）→ TResult の下限 int → 確定
            Select(IEnumerable<TSource>, Func<TSource,int,TResult>)（インデックス付き）はラムダの引数の数が合わず適用不可 → 1 つに決まる
      BindInvocationExpressionContinued
        CheckAndCoerceArguments: ラムダ → CreateAnonymousFunctionConversion → UnboundLambda.Bind(Func<int,int>)
          （_returnInferenceCache の式本体を再利用）
        → BoundCall(Enumerable.Select<int,int>, receiver=list（this 引数）, InvokedAsExtensionMethod=true)
    BindToNaturalType → 型 IEnumerable<int>
  local s の型を IEnumerable<int> に設定 → BoundLocalDeclaration
```
