# Roslyn C# 構文解析パイプライン 詳細解説
## ― ソーステキストからレクサー・パーサー・GreenNode を経て公開 Syntax API になるまで ―

対象:
- `src/Compilers/CSharp/Portable/Parser/`（レクサー・パーサー、約 3.3 万行）
- `src/Compilers/Core/Portable/Syntax/`（言語共通の Green/Red 基盤）
- `src/Compilers/CSharp/Portable/Syntax/`（`Syntax.xml` / `Syntax.xsd` / 手書き部分）
- `src/Tools/CompilerGeneratorTools/Source/CSharpSyntaxGenerator/`（コード生成器）
- `src/Compilers/CSharp/Portable/Generated/`（生成物）

基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。

---

## 0. 全体像

```text
                SourceText (string / stream)
                        │
  CSharpSyntaxTree.ParseText(...)                                    [公開 API]
                        │
                        ▼
┌──────────────── Lexer (Parser/Lexer.cs) ───────────────┐
│ SlidingTextWindow → QuickScanner(高速パス) / ScanSyntaxToken      │
│ → LexerCache(識別子・キーワード・trivia を共有)                  │
│ → 前処理ディレクティブは DirectiveParser で即時解析               │
│ 出力: Green SyntaxToken（leading/trailing trivia 付き）           │
└───────────────────────────┬───────────────────────────┘
                            ▼
┌──────────── LanguageParser (Parser/LanguageParser.cs) ──────────┐
│ 再帰下降 + 優先順位上昇法、ResetPoint による投機的解析・後戻り    │
│ 誤り回復: 欠落トークン(Missing) と SkippedTokensTrivia             │
│ 増分解析: Blender が旧ツリーの Green ノードを再利用                │
│ ノード生成: ContextAwareSyntax(_syntaxFactory)   ←─ 自動生成     │
└───────────────────────────┬───────────────────────────┘
                            ▼
      Green ツリー（Syntax.InternalSyntax.*、不変・親なし・位置なし・共有可能）
         GreenNode ← CSharpSyntaxNode ← IfStatementSyntax ...   ←─ 自動生成
                            │  CreateRed(parent, position)（遅延）
                            ▼
      Red ツリー（Microsoft.CodeAnalysis.CSharp.Syntax.*、公開 API）
         SyntaxNode ← CSharpSyntaxNode ← IfStatementSyntax ...  ←─ 自動生成
         SyntaxToken / SyntaxTrivia / SyntaxList<T> は struct ラッパ
                            │
                            ▼
      CSharpSyntaxTree (ParsedSyntaxTree)、SyntaxFactory、SyntaxVisitor/Rewriter ←─ 自動生成
```

設計の要点（いわゆる **Red-Green Tree**）:

| | Green（内部） | Red（公開） |
|---|---|---|
| 名前空間 | `Microsoft.CodeAnalysis.CSharp.Syntax.InternalSyntax` | `Microsoft.CodeAnalysis.CSharp.Syntax` |
| 保持情報 | Kind, 幅（FullWidth）, フラグ, 子（slot） | Green への参照 + **親** + **絶対位置** |
| 親参照 | 無し | 有り |
| 位置 | 相対幅のみ | 絶対位置（`Position`） |
| 生成 | パーサー/ファクトリが一括構築（ボトムアップ） | アクセス時に遅延生成（トップダウン） |
| 共有 | 同一部分木を複数の木・複数箇所で共有可能（キャッシュ・増分解析） | 1 つの木に固有 |
| トークン | クラス（`InternalSyntax.SyntaxToken`） | `readonly struct SyntaxToken` |

---

## 1. コード生成：`Syntax.xml` → `CSharpSyntaxGenerator` → 生成コード

（詳細は §5〜§7。ここではビルドの配線のみ）

### 1.1 入力ファイル

- `CSharp/Portable/Syntax/Syntax.xml`（5,265 行）：**C# 構文木の全ノード定義**（ノード型・基底型・フィールド・許可される SyntaxKind・XML ドキュメントコメント）。
- `CSharp/Portable/Syntax/Syntax.xsd`（241 行）：`Syntax.xml` の XML スキーマ。エディタでの補完・検証用で、**ジェネレータ自身は xsd を読まない**（`XmlSerializer` で `Model/*.cs` に直接デシリアライズする）。

### 1.2 ビルドへの組み込み（C#）

`CSharp/Portable/Microsoft.CodeAnalysis.CSharp.csproj:18-49`:

```xml
<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
<CompilerGeneratedFilesOutputPath>$(MSBuildThisFileDirectory)\Generated</CompilerGeneratedFilesOutputPath>
...
<ProjectReference Include="..\..\..\Tools\CompilerGeneratorTools\Source\CSharpSyntaxGenerator\CSharpSyntaxGenerator.csproj">
  <OutputItemType>Analyzer</OutputItemType>
  <ReferenceOutputAssembly>false</ReferenceOutputAssembly>
  <SetTargetFramework>TargetFramework=netstandard2.0</SetTargetFramework>
</ProjectReference>
...
<AdditionalFiles Include="Syntax\Syntax.xml" />
<None Include="Syntax\Syntax.xsd" />
<Compile Remove="$(CompilerGeneratedFilesOutputPath)/*/**/*.cs" />
```

- C# コンパイラの構文コードは **ビルド時に Roslyn 自身の Source Generator（`IIncrementalGenerator`）で生成**される。
- `CSharpSyntaxGenerator.csproj` は **2 つの TFM** をビルドする：`netstandard2.0` 版がソースジェネレータ、`$(NetRoslyn)` 版がコンソールアプリ。
- `EmitCompilerGeneratedFiles` により、生成物は `Generated/CSharpSyntaxGenerator/CSharpSyntaxGenerator.SourceGenerator/` にも書き出されてリポジトリにコミットされている（レビュー・閲覧用）。ただし `<Compile Remove=...>` で**コンパイル対象からは除外**しており、実際にコンパイルされるのはジェネレータのメモリ上の出力。

| 生成ファイル | 行数 | 中身 |
|---|---:|---|
| `Syntax.xml.Internal.Generated.cs` | 39,522 | Green ノード 291 型、Green Visitor/Rewriter、`ContextAwareSyntax`（パーサー用ファクトリ）、static `InternalSyntax.SyntaxFactory` |
| `Syntax.xml.Syntax.Generated.cs` | 17,069 | Red ノード 291 型（公開 API のプロパティ、`With*`/`Add*`/`Update`） |
| `Syntax.xml.Main.Generated.cs` | 6,714 | 公開 `CSharpSyntaxVisitor`/`<TResult>`、`CSharpSyntaxRewriter`、公開 `SyntaxFactory` のノード生成メソッド群 |

### 1.3 `eng/generate-compiler-code.cmd`（`eng/generate-compiler-code.cs`）

`Syntax.xml` 冒頭のコメントにある再生成スクリプト。単一ファイル C# アプリ（`#:project` で各ジェネレータを参照）で、C# については（`:163-189`）:

- **ソースは生成しない**（`language != CSharp` の時のみ `writeSource: true`。C# はソースジェネレータ担当）
- `/grammar` → `Generated/CSharp.Generated.g4`（ANTLR 形式の文法、1,811 行）
- `/test` → `Test/Syntax/Generated/Syntax.Test.xml.Generated.cs`（20,977 行のファクトリ／Rewriter テスト）
- 同じスクリプトが `BoundNodes.xml` → `BoundNodes.xml.Generated.cs`、`ErrorCode.cs` → `ErrorFacts.Generated.cs`、`OperationInterfaces.xml` も生成
- `-test` を付けると一時ディレクトリに生成して差分を検査（CI で生成物の更新漏れを検出）

VB は現在もスクリプトでソースを生成し（`Syntax.xml.*.Generated.vb`）、ソースジェネレータは使っていない。

---

## 2. レクサー（`Parser/Lexer.cs` ほか）

### 2.1 クラス構成

| ファイル | 行数 | 役割 |
|---|---:|---|
| `AbstractLexer.cs` | — | `SlidingTextWindow TextWindow`（**変更可能な struct をフィールドで直接保持**）、`LexemeStartPosition`、遅延生成されるエラーリスト `_errors` |
| `Lexer.cs` | 4,861 | `internal sealed partial class Lexer : AbstractLexer`（`:71`）。トークン・trivia・ディレクティブ・XML ドキュメントの字句解析 |
| `SlidingTextWindow.cs` | 473 | テキストバッファと文字列インターン |
| `QuickScanner.cs` | 439 | 状態遷移表による高速パス |
| `LexerCache.cs` | 257 | キーワード・トークン・空白 trivia のキャッシュ |
| `CharacterInfo.cs` | 240 | 文字種判定（`SyntaxFacts.IsWhitespace` / `IsNewLine` / `IsIdentifierStartCharacter` 等） |
| `Lexer_StringLiteral.cs` / `Lexer_RawStringLiteral.cs` / `Lexer.Interpolation.cs` | — | 文字列系 |
| `DirectiveParser.cs` / `Directives.cs` | 929 / 432 | 前処理ディレクティブ |
| `DocumentationCommentParser.cs` | 1,680 | XML ドキュメントコメント |

`Lexer` は `LexerCache.GetInstance()` から各種プールを借りて構築され（`:117-132`）、`Dispose()` で返却する。`Reset(position, DirectiveStack)`（`:170`）は増分解析の Blender が使う。

### 2.2 `LexerMode`（`Lexer.cs:19-53`）

下位 16 ビットが主モード、上位ビットは XML ドキュメントコメント用の副状態。

| フラグ | 意味 |
|---|---|
| `Syntax` (0x1) | 通常の C# |
| `DebuggerSyntax` (0x2) | 式エバリュエータ方言（`$` 付き識別子、旧 `123#` オブジェクト ID） |
| `Directive` (0x4) | `#...` 行の中 |
| `XmlDocComment`, `XmlElementTag`, `XmlAttributeText(Double)Quote`, `XmlCref(Double)Quote`, `XmlName(Double)Quote`, `XmlCDataSectionText`, `XmlCommentText`, `XmlProcessingInstructionText`, `XmlCharacter` | XML ドキュメントの各文脈 |
| `XmlDocCommentLocation*`（bit16-19） | `///` や `*` の行頭プレフィクスに対する位置。**レクサー自身が書き換える**（`Lex(ref LexerMode)`） |
| `XmlDocCommentStyle*`（bit20-21） | `///` 形式か `/** */` 形式か |

### 2.3 1 トークンの字句解析

```csharp
public SyntaxToken Lex(LexerMode mode)          // Lexer.cs:248
    // Syntax / DebuggerSyntax:  QuickScanSyntaxToken() ?? LexSyntaxToken()   (:258)
    // Directive:                LexDirectiveToken()
    // XML 系:                   LexXmlToken / LexXmlElementTagToken / ...

private SyntaxToken LexSyntaxToken()             // :292-309（遅いが完全なパス）
{
    _leadingTriviaCache.Clear();
    LexSyntaxTrivia(isFollowingToken: TextWindow.Position > 0, isTrailing: false, ref _leadingTriviaCache);
    var tokenInfo = default(TokenInfo);
    Start(); ScanSyntaxToken(ref tokenInfo); var errors = GetErrors();
    _trailingTriviaCache.Clear();
    LexSyntaxTrivia(isFollowingToken: true, isTrailing: true, ref _trailingTriviaCache);
    return Create(in tokenInfo, leading, trailing, errors);
}
```

- **`TokenInfo`**（`:96-115`）はスタック上の作業用 struct。`Kind`/`ContextualKind`/`Text`/`StringValue` に加え、`IntValue`・`DoubleValue`・`DecimalValue` など**値型ごとのフィールド**を持つので、走査中にボックス化が起きない。
- **`ScanSyntaxToken`**（`:425-734`）は `TextWindow.PeekChar()` に対する巨大な `switch`:
  - 記号は `AdvanceChar`＋`TryAdvance` の連鎖で複合演算子（`==`, `=>`, `/=` …）を判定する。
  - `.` は `.5`（実数）と `..`（範囲）を区別する。
  - `@` は逐語的文字列（`@"`、`@$"`）または逐語的識別子。`$` は補間文字列。
  - `_`、`a-z`、`A-Z` は `case` に直接列挙して `ScanIdentifierOrKeyword` へ送る（`IsIdentifierChar()` の呼び出しが高価なため）。
  - EOF は番兵文字 `InvalidCharacter` で検出する。ここで未閉の `#if`/`#region` を報告する。
  - 不正文字は `ERR_UnexpectedCharacter`。**不正トークンが 200 個を超えたら残り全体を 1 つの不正トークンにする**（病的入力への保護）。
- **識別子とキーワード**:
  - `ScanIdentifier_FastPath`（`:1321-1419`）は `CurrentWindowSpan`（`ReadOnlySpan<char>`）上で `[A-Za-z0-9_]` を走査し、そのまま `TextWindow.Intern(span)` する。Unicode・`\u` エスケープ・`@` などが来たら低速パスへ移る。
  - `ScanIdentifierOrKeyword`（`:1815-1870`）は `_cache.TryGetKeywordKind`（`SyntaxFacts.GetKeywordKind` をキャッシュしたもの。10 文字超はキーワードでない）でキーワードを判定する。
  - **文脈キーワード（`var`, `async`, `await`, `record`, `field` …）は `Kind = IdentifierToken`、`ContextualKind = 該当キーワード` として出力**し、キーワード扱いするかはパーサーが決める（`EatContextualToken` → `ConvertToKeyword`）。
- **数値リテラル**（`ScanNumericLiteral`, `:844-`）：`0x`/`0b` 接頭辞、`_` 区切り（位置検査付き）、小数点、指数、`f/d/m/L/U/UL` 接尾辞を処理し、`ValueKind` に応じた型付きフィールドに値を格納する。

### 2.4 Trivia の規則（`LexSyntaxTrivia`, `:1872-2057`）

- **後続 trivia（trailing）は最初の改行まで（改行を含む）**。`\r`/`\n` を読んだ時点で `isTrailing` なら戻る（`:1963-1974`）。それ以降の空白・コメント・改行は**次のトークンの先行 trivia（leading）**になる。
  ```text
  int x = 1;   // comment⏎
      return x;
  → ';' の trailing = [Whitespace "   ", SingleLineComment "// comment", EndOfLine]
  → 'return' の leading = [Whitespace "    "]
  ```
- `///` と `/**` はドキュメントコメント。trailing 側にいる場合はそこで打ち切り、**次トークンの leading として構造化 trivia**（`DocumentationCommentTriviaSyntax`）にする。
- `#` は**行頭（空白のみの後）かつ leading 側**の場合だけディレクティブとして扱う。それ以外は `ERR_BadDirectivePlacement` を付けて `SkippedTokensTrivia` にする。
- `<<<<<<<` / `=======` / `>>>>>>>` は Git のマージ衝突マーカーとして専用 trivia にする（`:2063-2110`）。
- 改行は `SyntaxFactory.CarriageReturnLineFeed` / `LineFeed` / `CarriageReturn` の**共有シングルトン**を返す。幅 1 の空白は `SyntaxFactory.Space`、42 文字未満の空白は `LexerCache` でハッシュ検索する（`ScanWhitespace`, `:2275-2334`）。

### 2.5 Green トークンの生成（`Create`, `:327-423`）

| 種別 | 生成 |
|---|---|
| 識別子 | `SyntaxFactory.Identifier(contextualKind, leading, text, valueText, trailing)` |
| 数値 | `SyntaxFactory.Literal(leading, text, <型付き値>, trailing)` → `SyntaxTokenWithValue<T>` |
| 文字列・文字 | `Literal(..., StringValue / CharValue, ...)` |
| 補間文字列 | `Literal(leading, Text, InterpolatedStringToken, Text, trailing)`（値は持たない。後でパーサーが再走査する） |
| 記号・キーワード | `SyntaxFactory.Token(leading, kind, trailing)` → **共有トークンキャッシュ**（§2.8） |
| 不正 | `BadToken` |

エラーは `token.WithDiagnosticsGreen(errors)` で付ける。§4.1 で述べたとおり、診断付きトークンは別インスタンスになる。

### 2.6 テキストバッファ `SlidingTextWindow`

- `[NonCopyable] internal struct`（`:24-25`）。`SourceText` から 4,096 文字ずつ `char[]`（プール品）に読み込む（`DefaultWindowLength = 4096`、「字句の 97〜98% が収まる」とコメントにある）。
- `InvalidCharacter = char.MaxValue`（U+FFFF）が番兵。本物の U+FFFF との区別は `IsReallyAtEnd()` で行う。
- **文字列インターン**：`Intern(ReadOnlySpan<char>)` などは `StringTable` を使う。ローカル表（2^11）の背後にプロセス全体の共有表（2^16）があり、**同じ識別子文字列はファイルをまたいで同一インスタンス**になる（`Core/.../InternalUtilities/StringTable.cs`）。
- `GetText` は `""`、`" "`、`"\n"`、`"\r\n"`、`"//"` などをリテラル定数で返し、割り当てしない。

### 2.7 高速パス `QuickScanner` と `LexerCache`

- `QuickScanSyntaxToken()`（`QuickScanner.cs:192-259`）は**状態遷移表**（`s_stateTransitions`, `byte[,]`）と**文字クラス表**（`CharProperties`、先頭 0x180 文字分の `ReadOnlySpan<byte>`）で、最大 42 文字を「先行空白 → トークン → 後続空白と改行 1 つ」まで一気に読み、同時に FNV ハッシュを計算する。
  - 状態：`Initial`、`FollowingWhite`、`FollowingCR`、`Ident`、`Number`、`Punctuation`、`Dot`、`CompoundPunctStart`、`DoneAfterNext`、`Done`、`Bad`。
  - コメント（`/`）、`#`、`"`、`@`、`$`、`\` などの複雑な文字、接尾辞付きの数値、`+=` のような複合記号では `Bad` になり、低速パスへ戻る。
  - `Done` なら `_cache.LookupToken(span, hash, CreateQuickToken)` を呼ぶ。**trivia を含むトークン全体を、元テキストをキーにして memo 化**している。ミス時は低速パスで生成してキャッシュに登録する。
- `LexerCache`（プール品）：
  - `TextKeyedCache<SyntaxToken>` / `TextKeyedCache<SyntaxTrivia>`。ローカル表とスレッド間で共有される静的表の 2 段構成。Green は不変なので**レクサー間・スレッド間で共有しても安全**。
  - キーワード種別の `CachingIdentityFactory<string, SyntaxKind>`。
  - `StringBuilder`、識別子バッファ、trivia 用 `SyntaxListBuilder`。

`x = y + 1;` のような典型的なコードでは、大半のトークンが QuickScanner とキャッシュで処理され、新しい Green オブジェクトの割り当てがほとんど発生しない。

### 2.8 Green トークンのクラス階層（`Syntax/InternalSyntax/SyntaxToken*.cs`）

トークンは最も数が多い Green ノードなので、**必要なフィールドだけを持つサブクラスに細かく分けて**メモリを節約している。

| クラス | 追加フィールド | 用途 |
|---|---|---|
| `SyntaxToken` | なし（Kind・フラグ・幅のみ。`Text => SyntaxFacts.GetText(Kind)`） | trivia の無い記号・キーワード |
| `SyntaxTokenWithTrivia` | `LeadingField`, `TrailingField` | trivia 付きの記号・キーワード |
| `MissingTokenWithTrivia` | （`IsNotMissing` を落とす） | 欠落トークン |
| `SyntaxIdentifier` | `TextField` | trivia 無しの識別子 |
| `SyntaxIdentifierWithTrailingTrivia` | `_trailing` | **最頻出**（識別子＋後続空白） |
| `SyntaxIdentifierExtended` | `contextualKind`, `valueText` | 文脈キーワード、`@` 付き識別子など |
| `SyntaxIdentifierWithTrivia` | `_leading`, `_trailing` | |
| `SyntaxTokenWithValue<T>` | `TextField`, `ValueField`（型付き） | リテラル（値は `Value` を読むまでボックス化しない） |
| `SyntaxTokenWithValueAndTrivia<T>` | `_leading`, `_trailing` | |

**共有トークン表**（`SyntaxToken.cs:140-162`）：`TildeToken`〜`EndOfFileToken`（記号とキーワード全種）について、静的コンストラクタが次の 4 種を事前に作る。

- trivia 無し
- 後続 trivia が `Space` 1 個
- 後続 trivia が `CRLF` 1 個
- 前後が `ElasticZeroSpace`

これに欠落トークン用の表も加わる。`SyntaxToken.Create(kind, leading, trailing)`（`:85-119`）は、trivia が上のいずれかに**参照一致**すれば共有インスタンスを返す。つまり行末の `;⏎` や `{`、`int ` のようなトークンは**割り当てゼロ**である。

`SyntaxTrivia`（`InternalSyntax/SyntaxTrivia.cs`）は `Text` 文字列を 1 つ持つ Green ノード。ディレクティブ・ドキュメントコメント・スキップされたトークンは `StructuredTriviaSyntax`（子を持つ Green ノード）で表す。

### 2.9 前処理ディレクティブ

- trivia 中の `#` → `LexDirectiveAndExcludedTrivia`（`Lexer.cs:2336`）→ `LexSingleDirective`。**同じレクサーを `Directive` モードで使う `DirectiveParser`**（`DirectiveParser.cs:14`、`SyntaxParser` の派生）がその場で構文解析し、結果の `DirectiveTriviaSyntax` を**構造化 trivia として leading trivia に入れる**。
- 同時に `_directives = directive.ApplyDirectives(_directives)` で、**不変の `DirectiveStack`**（`Directives.cs:118`。`ConsList<Directive>`）を更新する。
  - `#if` は `branchTaken = isActive && EvaluateBool(expr)`（`DirectiveParser.cs:147-154`）。式評価は `!`、`&&`、`||`、`==`、`!=`、括弧、`true`/`false`、シンボル（`IsDefined`：スタック上の `#define`/`#undef` → `Options.PreprocessorSymbols` の順に調べる）。
  - 偽の分岐は `LexDisabledText`（`:2415-2464`）が、**行頭が `#` の行に当たるまでの行をまとめて 1 つの `DisabledTextTrivia`** にする。中の入れ子ディレクティブは `isActive: false` で解析し、入れ子構造だけを追跡する。
  - `#endif` で `CompleteIf` がそのグループを畳み、採用された分岐のディレクティブだけを残す。
- `#region`/`#error`/`#warning` の残りのテキストは `PreprocessingMessageTrivia` になる。`#nullable`、`#pragma warning`、`#line`、`#!`（shebang）、`#:`（ファイルベースアプリ用、無視される）も処理する。
- 解析が終わったときの `DirectiveStack` は `ParsedSyntaxTree` に渡され（`CSharpSyntaxTree.cs:504-514`）、`#define` の判定などに使われる。増分解析では、旧ツリーと新ツリーでディレクティブ状態が `IncrementallyEquivalent` でなければ、ディレクティブを含むノードを再利用しない。

### 2.10 文字列

- 通常・逐語的文字列：`ScanStringLiteral`（`Lexer_StringLiteral.cs:14`）がエスケープを解釈し、値を `_builder` に溜める。`u8` 接尾辞は UTF-8 リテラル。
- **生文字列（raw string, `"""`）**：レクサーは範囲を決めるだけで、値の計算とインデント除去・診断は行わない（`Lexer_RawStringLiteral.cs:94-98` の assert）。後で `LanguageParser.ParseRawStringToken` が処理する。
- **補間文字列**：レクサーは `ref struct InterpolatedOrRawStringScanner` で穴（`{...}`）・入れ子の文字列・コメント・書式指定の対応を取り、**文字列全体を 1 つの `InterpolatedStringToken`** として出力する。レクサーをトークン間で状態を持たない設計に保つためで、分解は §3.7 のとおりパーサーが行う。

### 2.11 ドキュメントコメント

`///` または `/**` → `LexXmlDocComment`（`Lexer.cs:2794`）→ `DocumentationCommentParser`（XML モードでレクサーを再利用）が XML 要素・属性・`cref`/`name`（中身は C# として解析）・CDATA などを構文解析し、`SingleLine/MultiLineDocumentationCommentTrivia` という 1 つの構造化 trivia にする。`DocumentationMode < Parse` の場合は通常のコメント trivia として扱う。

---

## 3. パーサー（`Parser/SyntaxParser.cs`, `Parser/LanguageParser*.cs`）

### 3.1 `SyntaxParser`：トークンバッファと先読み

`internal abstract partial class SyntaxParser : IDisposable`（`SyntaxParser.cs:19`）。派生クラスは `LanguageParser`・`DirectiveParser`・`DocumentationCommentParser`。

```csharp
private ArrayElement<SyntaxToken>[] _lexedTokens;   // 非増分時のトークン配列（プール、4096）
private BlendedNode[] _blendedTokens;               // 増分時（ノードまたはトークン）
private int _firstToken;   // 配列[0] のトークン番号
private int _tokenOffset;  // 現在位置
private int _tokenCount;
private int _resetCount, _resetStart;               // 未解放の ResetPoint 数と最も古い位置
```

- **`CurrentToken`**（`:316`）は遅延取得。`AddNewToken`（`:341`）は非増分なら `lexer.Lex(_mode)`、増分なら直前スロットの `Blender.ReadToken(_mode)` を呼ぶ。
- **`PeekToken(n)`**（`:466`）は必要な分だけ字句解析して任意の距離を先読みできる。曖昧性の解消は大半がこれに頼っている。
- バッファが満杯になったとき、未解放の ResetPoint が古い部分を固定していなければ配列をスライドし、固定されていれば倍に拡張する（`:381-463`）。
- 非増分かつキャンセル不可の解析では、コンストラクタで最大 `min(4096, 文字数/2)` トークンを先に字句解析する（`PreLex`, `:138-156`）。
- `IsMakingProgress`（`:1178`）が、ループが前進しなくなった場合を検知する安全装置になっている。

### 3.2 トークンの消費と欠落トークン

| メソッド | 動作 |
|---|---|
| `EatToken()` | 無条件に消費 |
| `EatToken(kind)` | 一致すれば消費。**不一致なら消費せず、幅 0 の欠落トークン**（`SyntaxFactory.MissingToken`）に「X expected」診断を付けて返す（`CreateMissingToken`, `:552`） |
| `TryEatToken(kind)` | 不一致なら null |
| `EatTokenAsKind(kind)` | 欠落トークンを作り、実際のトークンをその trailing のスキップ構文にする |
| `EatTokenEvenWithIncorrectKind(kind)` | 種別違いでも消費し、「X expected」を付ける |
| `EatContextualToken(kind)` | `ContextualKind` を比較し、一致すれば `ConvertToKeyword` でキーワード種別のトークンに作り直す |

欠落トークンの診断コード（`GetExpectedTokenErrorCode`, `:700-730`）：`;` → `ERR_SemicolonExpected`（CS1002）、`)` → `ERR_CloseParenExpected`（CS1026）、`{` → `ERR_LbraceExpected`、`}` → `ERR_RbraceExpected`、識別子 → `ERR_IdentifierExpected`（予約語なら `ERR_IdentifierExpectedKW`）、その他 → `ERR_SyntaxError`。

**診断の位置**（`GetDiagnosticSpanForMissingNodeOrToken`, `:783-896`）：直前のトークンの trailing trivia に改行があれば「前の行の末尾に幅 0」、無ければ「次のトークン」に波線を引く。診断はノード先頭からの**相対オフセット**で Green に保存されるので、ノードが移動しても無効にならない。

### 3.3 スキップされたトークン

`AddLeadingSkippedSyntax` / `AddTrailingSkippedSyntax`（`:958-1102`）は、消費できなかったトークンを隣接トークンの trivia に移す。各トークンを「leading trivia ＋ `SkippedTokensTrivia(トークン本体)` ＋ trailing trivia」に分解し、`SyntaxFirstTokenReplacer` / `SyntaxLastTokenReplacer` で対象ノードの最初または最後のトークンを差し替える。**どんなに壊れた入力でも、全文字がいずれかのトークンか trivia に収まる**（full fidelity の保証）。

### 3.4 ResetPoint（投機的解析と後戻り）

```csharp
protected readonly struct ResetPoint {          // SyntaxParser.ResetPoint.cs:11-25
    internal readonly int ResetCount; internal readonly LexerMode Mode;
    internal readonly int Position;   internal readonly GreenNode PrevTokenTrailingTrivia; }
GetResetPoint()   // :158  位置を記録し、_resetStart でバッファの解放を止める
Reset(ref point)  // :170  _tokenOffset を戻す（トークン配列はそのまま再利用）
Release(ref point)// :209
```

`LanguageParser` はこれを包む `new struct ResetPoint`（`LanguageParser.cs:14671`）を定義し、`_termState`・`IsInAsync`・`IsInQuery`・`IsInFieldKeywordContext` も保存する。`using` で使える `ref struct DisposableResetPoint`（`:14646`）もある。例：`using (GetDisposableResetPoint(resetOnDispose: true)) { kind = ScanTypeArgumentList(options); }`。

**字句解析をやり直さず、トークン配列の添字を戻すだけ**なので、後戻りは安い。

### 3.5 `LanguageParser`：構造

```csharp
internal sealed partial class LanguageParser : SyntaxParser {      // :20
    private readonly SyntaxListPool _pool = new SyntaxListPool();
    private readonly SyntaxFactoryContext _syntaxFactoryContext;
    private readonly ContextAwareSyntax _syntaxFactory;            // ← 自動生成された Green ファクトリ
    private int _recursionDepth;  private TerminatorState _termState;
}
```

- **入口** `ParseCompilationUnit()`（`:168`）→ `ParseWithStackGuard(ParseCompilationUnitCore, …)`。
  - `NamespaceBodyBuilder`（extern/using/属性/メンバーの 4 つのプール済みリスト）を確保して `ParseNamespaceBody`（`:410` → `ParseNamespaceBodyWorker` `:562`）を呼ぶ。
  - EOF を食べて `_syntaxFactory.CompilationUnit(...)` を返す。
  - extern → using → グローバル属性 → メンバーの順序は `enum NamespaceParts` で検査し、順序違反はエラーにする。
- **メンバー / トップレベル文** `ParseMemberDeclarationOrStatement`（`:2574`）。トップレベル文は暗黙に async なので、`ParserSyntaxContextResetter(isInAsyncContext: …)` で文脈を切り替える（`:2629`）。
- **文** `ParseStatementCore`（`:8285`）：
  1. 増分解析での再利用を試す（`TryReuseStatement`, `:8367`）。
  2. ResetPoint を取る。
  3. キーワードで大きく `switch` する（`if`/`for`/`foreach`/`try`/`switch`/`using`/`return`/`{`/`;` …）。
  4. 残りは `IsPossibleLocalDeclarationStatement`（`:8507`）で宣言か式文かを判定する。
  - async 文脈外で `await` から始まる宣言にエラーが出た場合は、**async 文脈として解析し直す**（`:8409-8420`）。
- `[...]` は属性かコレクション式かを、コレクション式として投機的に解析し、後続トークンを見て判定する（`ParseStatementAttributeDeclarations`, `:8197`）。

### 3.6 式：優先順位上昇法（precedence climbing）

`enum Precedence`（`:11200-11224`、緩い順）：

`Expression = Assignment = Lambda` < `Conditional` < `Coalescing` < `ConditionalOr` < `ConditionalAnd` < `LogicalOr` < `LogicalXor` < `LogicalAnd` < `Equality` < `Relational` < `Shift` < `Additive` < `Multiplicative` < `Switch` < `Range` < `Unary` < `Cast` < `PointerIndirection` < `AddressOf` < `Primary`

```text
ParseExpression() → ParseSubExpression(Precedence.Expression)          // :11062, :11435
  ParseSubExpressionCore(precedence)                                     // :11450
    left = 単項/一次式 (前置演算子は演算子の優先度で再帰, await, throw, クエリ, ParsePrimaryExpression)
    ParseExpressionContinued(left, precedence)                            // :11535
      loop:
        (opToken, exprKind) = GetExpressionOperatorTokenKindAndExpressionKind()
        newPrec = GetPrecedence(exprKind)
        if newPrec < precedence または (同順位かつ右結合でない) → 終了
        EatExpressionOperatorToken()        // '>' '>' を '>>' に合成（ジェネリクスとの両立のため lexer は '>' を単独で出す）
        right = ParseSubExpression(newPrec)  // 右辺は同じ優先度で再帰
        left = _syntaxFactory.BinaryExpression(exprKind, left, op, right)
      '?' があり precedence <= Conditional なら条件演算子
```

`GetPrecedence` は**トークンではなく式の SyntaxKind**に対して定義されている（`:11226`）。DEBUG ビルドでは、生成した全ての式についてこの表を引いて漏れを検出する。

### 3.7 曖昧性解消のヒューリスティクス

C# の文法は LL(k) で解けない曖昧性を含むため、**`Scan*` 系メソッドでツリーを作らずに先読みし、ResetPoint で戻る**のが基本の型。

| 曖昧性 | 処理 |
|---|---|
| `A < B > C` はジェネリクスか比較か | `ScanTypeArgumentList`（`:6230`）。式の文脈では `>` の**直後のトークン**で決める（仕様 §7.5.4.2）。`( ) ] } : ; , . ? == != \| ^` なら型引数。C# 7 で `&& \|\| & [ < <= >= is as`、C# 8 で `{` が加わった |
| `(T)x` はキャストか括弧式か、タプルか | `ParseCastOrParenExpressionOrTuple`（`:12831`）→ `ScanCast`（`:12919`）。`ScanTypeFlags`（`MustBeType`, `NullableType`, `GenericTypeOrMethod`, `TupleType` …）と `CanFollowCast` で判定する |
| `(a, b) => ...` はラムダか | `IsPossibleLambdaExpression`（`:13057`）→ `ScanParenthesizedLambda`（`:12714`）→ `TryParseLambdaExpression`（`:13879`、三項演算子の `?` と衝突したら戻る） |
| `b ? [c] : d` | コレクション式か条件付きアクセスかを投機的に解析し直す（`ForceConditionalAccessExpression`） |
| `x is T` / パターン | `ParseTypeOrPatternForIsOperator`（`LanguageParser_Patterns.cs:21`）はパターンとして解析し、型として表せるものは `IsExpression` に戻す |
| 文脈キーワード | `IsTrueIdentifier`（`:6022`）は、クエリ内のクエリキーワードと async 内の `await` を識別子として扱わない。`record`/`union`/`field`/`extension` などは `IsFeatureEnabled` に応じて解釈を変える |

パターン（`LanguageParser_Patterns.cs`）：`ParseDisjunctivePattern`（`or`）→ `ParseConjunctivePattern`（`and`）→ `ParseNegatedPattern`（`not`）→ `ParsePrimaryPattern`（関係・括弧・リスト・`var`・型・再帰パターン）。`switch` 式は `ParseSwitchExpression`（`:593`）。

### 3.8 補間文字列の再解析（`LanguageParser_InterpolatedString.cs`）

1. レクサーは文字列全体を 1 つの `InterpolatedStringToken` として出力する（§2.10）。
2. `ParseInterpolatedStringToken`（`:90`）→ `ParseInterpolatedOrRawStringToken`（`:126`）が、**一時的な `Lexer` をトークンの文字列上に作り直し**、`ScanInterpolatedOrRawStringLiteralTop` で `{` `:` `}` の範囲（`Lexer.Interpolation`）を集める。
3. 穴ごとに `ParseInterpolation`（`:486`）が、さらに**一時 `Lexer` と一時 `LanguageParser`** を作って式・`,alignment`・`:format` を解析する。外側の `IsInAsync` などの文脈は引き継ぐ。
4. 文字列部分は `MakeInterpolatedStringTextToken`（`:579`）で生成し、診断のオフセットを補正する。DEBUG では `originalToken.ToFullString() == result.ToFullString()` を検証する。
5. 増分解析では、補間内の変更があると Blender が範囲を行頭まで広げ、文字列全体を字句解析し直す（`Blender.cs:146-153`）。

### 3.9 誤り回復：`TerminatorState`

`[Flags] enum TerminatorState`（`:58-90`、29 ビット）。各規則は入るときに自分の終端条件を OR し、出るときに元へ戻す。

```csharp
var saveTerm = _termState; _termState |= TerminatorState.IsEndOfParameterList; ...; _termState = saveTerm;
```

`IsTerminator()`（`:94-137`）は、**有効なフラグのいずれかの述語が現在のトークンに当てはまれば真**を返す。例えば引数リストの中で `}` や `class` が来た場合、外側の規則が再同期できるトークンなので、内側の規則はそこで消費をやめる。

`SkipBadTokensWithExpectedKind`（`:4582`）は、期待しないトークンを読み飛ばしてスキップ trivia にまとめる。終端トークンに当たれば `PostSkipAction.Abort`。

まとめると、パーサーの誤り回復は次の 3 段構え。

1. 欠落トークンを挿入する
2. 余分なトークンを trivia として飛ばす
3. 外側の規則の終端トークンで再同期する

**パーサーは決して失敗せず、必ず 1 つのツリーを返す**。

### 3.10 スタック深度・キャンセル・言語バージョン

- 再帰的な規則（名前空間、メンバー、文、部分式）は `_recursionDepth++` と `StackGuard.EnsureSufficientExecutionStack(_recursionDepth)` で深さを数える。`ParseWithStackGuard`（`:207-234`）は `InsufficientExecutionStackException` を捕捉すると、**入力全体を 1 つの BadToken としてスキップ trivia にし**、`ERR_InsufficientStack` を付ける。
- メンバー・文のループで `cancellationToken.ThrowIfCancellationRequested()` を呼ぶ。
- **言語バージョン検査の大半は現在バインダ側にある**。パーサーの `CheckFeatureAvailability` 呼び出しは `IDS_FeatureGenerics`（`:6553`）の 1 箇所だけ（`DirectiveParser` に別途 2 箇所）。パーサーは `IsFeatureEnabled` を**解析を分岐させるため**（`record`/`field`/`extension` を文脈キーワードとして扱うかどうか等）に使う。

### 3.11 ノード生成とリストのプール

- 全ての Green ノードは `_syntaxFactory`（自動生成された `ContextAwareSyntax`、§7.2）で作る。
- `SyntaxFactoryContext`（`Parser/SyntaxFactoryContext.cs`）の `IsInAsync`/`IsInQuery`/`IsInFieldKeywordContext` が、ノードの `FactoryContextIsIn*` フラグに焼き込まれる。`ref struct ParserSyntaxContextResetter`（`:4322`）が文脈の一時的な切り替えと復元を受け持つ。
- 可変長リストは `_pool.Allocate<T>()` / `AllocateSeparated<T>()` で借り、`_pool.ToListAndFree(builder)` で Green リストに確定する。「ファクトリはビルダへの参照を保持しない」（`:22`）ので再利用は安全。

### 3.12 増分解析（Blender）

```text
旧ツリー + TextChangeRange[]
   │ Blender (readonly struct, 不変スナップショット)                  Blender.cs:15
   │   ExtendToAffectedRange: 変更範囲を先読み分(1トークン)・補間文字列なら行頭まで拡張
   │   Cursor: 旧 Red ツリーを幅 0 要素を飛ばしつつ巡回                 Blender.Cursor.cs
   ▼
Reader.ReadNodeOrToken(mode)                                        Blender.Reader.cs:38-104
   changeDelta < 0 → 旧トークンを捨てる / > 0 → 新テキストを字句解析
   == 0 → TryTakeOldNodeOrToken: CanReuse なら旧 Green を丸ごと返す、ダメなら子へ降りる
   ▼
BlendedNode { Node, Token, Blender(読み終えた後の状態) }               BlendedNode.cs
   → SyntaxParser._blendedTokens[] に格納（後戻りは添字を戻すだけ）
```

- **`CanReuse`**（`Blender.Reader.cs:215-275`）が再利用を拒否する場合：
  - 幅 0
  - 注釈を含む
  - 次の変更範囲と交差する
  - **診断を含む**
  - **パーサーが合成した種別**（`IsFabricatedToken`, `:299`）。`>>`、`>>=`、`>>>`、`>>>=`、`..` と**全ての文脈キーワード**。これらはレクサーではなくパーサーが作った種別なので、そのまま流用できない
  - 欠落トークン、または未完成のノード（最後の終端記号が欠落）
  - ディレクティブを含み、かつ新旧の `DirectiveStack` が等価でない
- **パーサー側の再利用**：`SyntaxParser.CurrentNode`（`:250`）で旧ノードを覗き、`EatNode()`（`:292`）で Green ノードを丸ごと取り込む。`LanguageParser.IsIncrementalAndFactoryContextMatches`（`:14416`）は、**旧ノードを解析したときの async/クエリ/field 文脈が現在と一致する**ことを要求する（`MatchesFactoryContext`, `:14430`。§4.1 のフラグはこのためにある）。
- 再利用する箇所：using、属性、メンバー宣言（`CanReuseMemberDeclaration`, `:2513`）、アクセサ、パラメータリスト、変数宣言子、enum メンバー、識別子名、文（`TryReuseStatement`）、ブロック。

**位置を持たない Green ツリー**（§4.1）は、この増分解析を前提にした設計でもある。編集位置より後ろのノードも、オフセットを書き換えずにそのまま新しいツリーへ移せる。

---

## 4. Green ノードの実装（`Core/Portable/Syntax/GreenNode.cs` ほか）

### 4.1 `GreenNode` のメモリレイアウト

```csharp
internal abstract partial class GreenNode            // GreenNode.cs:21
{
    internal const int ListKind = 1;
    protected const int SlotCountTooLarge = 0b1111;   // :38
    private readonly ushort _kind;                     // SyntaxKind (16bit)
    private NodeFlagsAndSlotCount _nodeFlagsAndSlotCount; // 16bit = Count(4) + Flags(12)
    private int _fullWidth;                            // trivia 込みの文字数
    private static readonly ConditionalWeakTable<GreenNode, DiagnosticInfo[]> s_diagnosticsTable;  // :44
    private static readonly ConditionalWeakTable<GreenNode, SyntaxAnnotation[]> s_annotationsTable; // :47
}
```

- インスタンス固有データは **Kind(2B) + フラグとスロット数(2B) + 幅(4B) = 8 バイト**だけ（`GreenNode.cs:30-42` のコメント「Pack the kind, node-flags, slot-count, and full-width into 64bits」）。これにオブジェクトヘッダと子フィールドが加わる。
- **診断（エラー）と注釈は本体に持たず**、`ConditionalWeakTable` という外部の弱参照テーブルに置く（`:44-47`, `:69-118`）。大半のノードは診断も注釈も持たないため、全ノードにフィールドを持たせるより省メモリ。持っているかどうかはフラグ `ContainsDiagnostics`/`HasAnnotationsDirectly` で O(1) 判定。
- **位置（オフセット）を持たない**。子の相対オフセットは `GetSlotOffset(i)`（左の兄弟の `FullWidth` を合計、`:190-204`）で計算する。位置を持たないから、同じ部分木を別の位置・別のツリーで共有できる。

`NodeFlagsAndSlotCount`（`GreenNode.NodeFlagsAndSlotCount.cs`）:

```text
 ビット: CCCC FFFF FFFF FFFF
         └┬─┘ └──────┬─────┘
   SlotCount(0-14)   NodeFlags(12bit)
   15(=0b1111) なら「大きすぎる」→ 仮想 GetSlotCount() で取得（要素数の多いリストのみ）
```

`NodeFlags`（`GreenNode.cs:258-296`）:

| フラグ | 上方継承 | 意味 |
|---|:-:|---|
| `IsNotMissing` | ✔ | 欠落トークン/ノードでない。**「非欠落」を 1 にしてあるのは OR で上へ伝搬させるため**（子の 1 つでも非欠落なら親も非欠落） |
| `HasAnnotationsDirectly` | | 自身が注釈を持つ |
| `FactoryContextIsInAsync` / `IsInQuery` / `IsInFieldKeywordContext` | | パース時の文脈（`await`/クエリ/`field` が文脈キーワードだったか）。**増分解析で再利用してよいかの判定に使う** |
| `ContainsAnnotations` / `ContainsAttributes` / `ContainsDiagnostics` / `ContainsDirectives` / `ContainsSkippedText` / `ContainsStructuredTrivia` | ✔ | 子孫のどこかに存在する |

`InheritMask` に含まれるフラグは、子を設定する度に `AdjustFlagsAndWidth(child)`（`:120-125`）で親に OR され、同時に `_fullWidth += child.FullWidth` される。これによりボトムアップ構築だけで、全ノードの幅と「子孫に〜を含むか」が求まる。例えば `ContainsDiagnostics` が偽の部分木は診断収集で丸ごとスキップできる。

### 4.2 スロットと主な仮想メンバ

| メンバ | 説明 |
|---|---|
| `GetSlot(int)` | i 番目の子（トークン・ノード・リスト、無ければ null）。生成コードは `index switch` で実装 |
| `SlotCount` | 子の数（null も含む固定長） |
| `FullWidth` / `Width` / `GetLeadingTriviaWidth()` / `GetTrailingTriviaWidth()` | 幅の計算（Width = FullWidth − 前後 trivia） |
| `IsToken` / `IsTrivia` / `IsList` / `IsMissing` / `IsStructuredTrivia` / `IsDirective` | 種別判定 |
| `CreateRed(SyntaxNode? parent, int position)` | **対応する Red ノードを生成**（`:978`）。生成コードが `new CSharp.Syntax.XxxSyntax(this, parent, position)` を返す |
| `SetDiagnostics` / `SetAnnotations` | 診断・注釈を付けた**新しい**ノードを返す（不変） |
| `AddError(DiagnosticInfo)` | 既存の診断に追加した新しいノード（`:990-1010`）。パーサーの誤り報告に使われる |
| `IsEquivalentTo` | 構造的等価性（`:843-904`） |
| `WriteTo` / `ToFullString` | テキスト復元。スタックオーバーフローを避けるため明示スタックで反復（`:649-700`） |

**Green ツリーは元テキストを完全に保持する（full fidelity）**。全トークンと全 trivia（空白・コメント・ディレクティブ・無効化されたテキスト・スキップされたトークン）を連結すると、元のソース文字列に 1 文字の狂いもなく戻る。

### 4.3 リスト（`Core/Portable/Syntax/InternalSyntax/SyntaxList*.cs`）

リストも Kind = `ListKind`(1) の GreenNode。要素数で実装クラスを使い分けて省メモリ化:

| 要素数 | 表現 |
|---|---|
| 0 | `null`（空リストはオブジェクトを作らない） |
| 1 | **要素そのもの**（リストノードを作らない） |
| 2 | `SyntaxList.WithTwoChildren` |
| 3 | `SyntaxList.WithThreeChildren` |
| 4〜 | `WithManyChildren`（配列）、さらに多いと `WithLotsOfChildren`（子のオフセットを事前計算して二分探索可能にする） |

（`GreenNode.CreateList`, `GreenNode.cs:921-975`）。区切り付きリスト（`a, b, c`）は要素と区切りトークンを交互に並べた同じ構造で表す。

パーサーは `SyntaxListPool`（`InternalSyntax/SyntaxListPool.cs`）から `SyntaxListBuilder` を借りて要素を溜め、`ToList()` で確定したら返却する。これでパース中の一時配列の割り当てを避けている。

### 4.4 Green ノードキャッシュ `SyntaxNodeCache`（`InternalSyntax/SyntaxNodeCache.cs`）

**同じ形の小さなノードを重複排除（hash-consing）**する機構:

- `s_cache`：サイズ 2^16 の **単純な配列**（`:125-136`）。衝突解決はせず、同じインデックスなら後から来たもので上書きする「機会主義的」キャッシュ。読み出し時に Kind・フラグ・子の参照一致を検証する。
- 対象は **子が 3 つ以下**のノードだけ。子は**参照の等価**で比較するので、子自身が既に重複排除済みでないと意味が無い（`AllChildrenInCache`）。トークンはレクサー側で共有済みとみなす。
- 診断・注釈付きのノードは対象外。それらは常にコンストラクタ（`SetDiagnostics` 等）経由で作られ、ファクトリを通らないためキャッシュに入らない（`:43-50` の remarks）。
- C# 版の `CSharpSyntaxNodeCache`（`CSharp/Portable/Syntax/InternalSyntax/CSharpSyntaxNodeCache.cs`）は `SyntaxFactoryContext`（async/クエリ文脈）をフラグとしてハッシュに含める。文脈が違えば別ノードになる。

例えば `x` という `IdentifierNameSyntax` や `i++`、`a.b` のような頻出の小さな式は、ファイル全体（さらにはプロセス全体）で同じ Green インスタンスを指すことになる。

### 4.5 C# 固有の Green 基底 `InternalSyntax.CSharpSyntaxNode`

`CSharp/Portable/Syntax/InternalSyntax/CSharpSyntaxNode.cs`：`GreenNode` を継承し、`Kind`（`SyntaxKind` へのキャスト）、`Language = "C#"`、`SetFactoryContext(SyntaxFactoryContext)`（`:223`、文脈フラグを設定）、トークン/trivia の共通処理、`Accept(CSharpSyntaxVisitor)` を提供する。手書きの partial（`InternalSyntax/IdentifierNameSyntax.cs`, `TypeSyntax.cs` など）が生成コードに補助メンバを足している。

---

## 5. `Syntax.xml` と `Syntax.xsd` の構造

### 5.1 要素一覧（`Syntax.xsd`）

```text
<Tree Root="SyntaxNode">
  <PredefinedNode Name Base [ExperimentalUrl]/>           … 手書き済みの基底型（生成しない）
  <AbstractNode Name Base [ExperimentalUrl]>             … 抽象ノード
     <TypeComment/> <Field .../> <Choice/> ...
  <Node Name Base [SkipConvenienceFactories] [ExperimentalUrl]> … 具象ノード
     <Kind Name/>+                                        … このノード型が取り得る SyntaxKind
     <Field Name Type [Optional] [Override] [MinCount] [AllowTrailingSeparator] [ExperimentalUrl]>
        <Kind Name/>*                                      … このトークンが取り得る SyntaxKind
        <PropertyComment/>
     </Field>
     <Choice [Optional]> Field/Sequence ... </Choice>     … いずれか 1 つ
     <Sequence> Field/Choice ... </Sequence>              … 連続する組
     <TypeComment/> <FactoryComment/>
```

`Syntax.xml` の規模：`PredefinedNode` 3（`CSharpSyntaxNode`, `SyntaxToken`, `StructuredTriviaSyntax`）、`AbstractNode` 41、`Node` 252、`<Kind>` 1,019、`Optional="true"` 151、`Override="true"` 318、`<Choice>` 18、`<Sequence>` 11、`SkipConvenienceFactories` 9。

### 5.2 実例：`IfStatementSyntax`（`Syntax.xml:2780-2836`）

```xml
<Node Name="IfStatementSyntax" Base="StatementSyntax">
  <Kind Name="IfStatement"/>
  <Field Name="AttributeLists" Type="SyntaxList&lt;AttributeListSyntax&gt;" Override="true"/>
  <Field Name="IfKeyword" Type="SyntaxToken"> <Kind Name="IfKeyword"/> ... </Field>
  <Field Name="OpenParenToken" Type="SyntaxToken"> <Kind Name="OpenParenToken"/> </Field>
  <Field Name="Condition" Type="ExpressionSyntax"/>
  <Field Name="CloseParenToken" Type="SyntaxToken"> <Kind Name="CloseParenToken"/> </Field>
  <Field Name="Statement" Type="StatementSyntax"/>
  <Field Name="Else" Type="ElseClauseSyntax" Optional="true"/>
  <TypeComment>...</TypeComment>
  <FactoryComment>...</FactoryComment>
</Node>
```

属性の意味:

| 属性・要素 | 生成コードへの影響 |
|---|---|
| `Type` | `SyntaxToken` / ノード型 / `SyntaxList<T>` / `SeparatedSyntaxList<T>` / `SyntaxTokenList` / 値型（`bool` 等の「値フィールド」） |
| `Field/<Kind>` | ファクトリでのトークン種別検査（Green は `#if DEBUG` のみ、公開 API は常時）。種別が 1 つなら**自動生成可能なトークン**として簡易ファクトリの引数から省かれる |
| `Node/<Kind>` が複数 | ファクトリの第 1 引数に `SyntaxKind kind` が要る（例 `BinaryExpressionSyntax` は `AddExpression` など多数） |
| `Optional="true"` | null 許容フィールド（`ElseClauseSyntax?`）。簡易ファクトリでは省略可 |
| `Override="true"` | 基底の抽象プロパティのオーバーライド（`public override SyntaxList<AttributeListSyntax> AttributeLists`）。`WithXxxCore` の橋渡しも生成 |
| `MinCount` / `AllowTrailingSeparator` | リストの最小要素数・末尾区切り許可（テストと文法生成で使用） |
| `Choice` / `Sequence` | 「どれか 1 つが非 null」「組で現れる」という**文法上の関係**を表す。ソース生成では平坦化され（`TreeFlattening.cs`）、`Choice` 内のフィールドは自動的に `Optional` になる。主に `.g4` 文法生成のための情報 |
| `ExperimentalUrl` | 生成型/メンバに `[Experimental]` 属性を付ける |
| `SkipConvenienceFactories` | 簡易ファクトリのオーバーロードを生成しない |
| `TypeComment` / `PropertyComment` / `FactoryComment` | 生成コードの XML ドキュメントコメント |

---

## 6. ジェネレータ `CSharpSyntaxGenerator` の実装

`src/Tools/CompilerGeneratorTools/Source/CSharpSyntaxGenerator/`（約 4,000 行）

| ファイル | 役割 |
|---|---|
| `Model/Tree.cs`, `TreeType.cs`, `Node.cs`, `AbstractNode.cs`, `PredefinedNode.cs`, `Field.cs`（`Choice`/`Sequence` 含む）, `Kind.cs`, `Comment.cs` | `[XmlRoot]`/`[XmlElement]`/`[XmlAttribute]` 付きの POCO。`XmlSerializer` でそのままデシリアライズ |
| `SourceGenerator.cs`（`#if NETSTANDARD`） | `IIncrementalGenerator`。`AdditionalTextsProvider` から `Syntax.xml` を拾う |
| `Program.cs`（`#if NET`） | コンソール版。`input output [/test|/grammar]` または `input /sig` |
| `TreeFlattening.cs` | `Choice`/`Sequence` を平坦な `Fields` リストに展開 |
| `AbstractFileWriter.cs` | インデント付き書き出し、型判定ヘルパ（`IsNodeOrNodeList`, `IsAnyList`, `IsOptional`, `GetNode` …） |
| `SourceWriter.cs`（1,931 行） | **本体**。3 つのファイルを書き出す |
| `TestWriter.cs` | `Syntax.Test.xml.Generated.cs` |
| `SignatureWriter.cs` | `/sig`：型シグネチャ一覧 |
| `Grammar/GrammarGenerator.cs` | `.g4` 文法。`SyntaxKind.cs`/`SyntaxKindFacts.cs` をリンクして、Kind からトークン文字列（`'if'` など）を得る |

### 6.1 Source Generator の流れ（`SourceGenerator.cs`）

```csharp
var syntaxXmlFiles = context.AdditionalTextsProvider
    .Where(at => Path.GetFileName(at.Path) == "Syntax.xml").Collect();
context.RegisterSourceOutput(syntaxXmlFiles, (context, files) => {
    // 無ければ CSSG1001（警告）、読めなければ CSSG1002、XML 構文エラーは CSSG1003（行・列付き）
    var reader = XmlReader.Create(new SourceTextReader(inputText),
                                  new XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit });
    tree = (Tree)new XmlSerializer(typeof(Tree)).Deserialize(reader);
    DoGeneration(tree, context, context.CancellationToken);
});
// DoGeneration:
TreeFlattening.FlattenChildren(tree);
addResult(w => SourceWriter.WriteMain(w, tree, ct),     "Syntax.xml.Main.Generated.cs");
addResult(w => SourceWriter.WriteInternal(w, tree, ct), "Syntax.xml.Internal.Generated.cs");
addResult(w => SourceWriter.WriteSyntax(w, tree, ct),   "Syntax.xml.Syntax.Generated.cs");
```

巨大な出力（約 4 万行）を 1 本の `string` にしないよう、`StringBuilder` をラップする独自の `StringBuilderText : SourceText` で `AddSource` している。`SourceTextReader` も同じ理由で、`SourceText` を `string` にせずに `XmlReader` へ流す。

### 6.2 `SourceWriter` の出力構成

```csharp
WriteInternal(): namespace ...Syntax.InternalSyntax;
    WriteGreenTypes();             // Green ノード型（AbstractNode と Node）
    WriteGreenVisitors();          // InternalSyntax.CSharpSyntaxVisitor / <TResult>
    WriteGreenRewriter();          // InternalSyntax.CSharpSyntaxRewriter
    WriteContextualGreenFactories();  // ContextAwareSyntax（パーサー用、SyntaxFactoryContext 付き）
    WriteStaticGreenFactories();      // static InternalSyntax.SyntaxFactory
WriteSyntax():   namespace ...CSharp.Syntax;
    WriteRedTypes();               // Red ノード型
WriteMain():     namespace ...CSharp;
    WriteRedVisitors();            // 公開 CSharpSyntaxVisitor / <TResult>
    WriteRedRewriter();            // 公開 CSharpSyntaxRewriter
    WriteRedFactories();           // 公開 SyntaxFactory
```
（`SourceWriter.cs:116-145`）

### 6.3 生成ルール

- **Green ファクトリのキャッシュ判定**（`:658-663`）：値フィールドとノードフィールドの合計が **3 以下**なら `SyntaxNodeCache` を使うコードを生成。ただし `SkippedTokensTriviaSyntax`・`DocumentationCommentTriviaSyntax`・`IncompleteMemberSyntax`・`AttributeSyntax` は除外する（同一性が文脈に依存するため）。
- **自動生成可能なトークン** `IsAutoCreatableToken`（`:1461-1466`）：`SyntaxToken` 型で、
  - 許可 Kind が 1 つだけで、それが `IdentifierToken` でも `*LiteralToken` でもない（例 `IfKeyword`, `OpenParenToken`）、または
  - トークンの Kind 数とノードの Kind 数が等しい（ノードの Kind から一意に決まる。例 `BinaryExpression` の演算子トークン。`Get{Node}{Field}Kind(SyntaxKind)` 変換関数も生成する `WriteKindConverters`, `:1667-1690`）。
- **自動生成可能なノード** `IsAutoCreatableNode`：必須引数が 0 のノード型（例：空でよい `ParameterList`）。
- **必須フィールド** `IsRequiredFactoryField`（`:1474-1477`）＝ 省略不可・リストでない・自動生成不可、または値フィールド。
- **公開ファクトリのオーバーロード**（`WriteRedFactories`）：
  1. 全フィールドを取る完全版（`WriteRedFactory`）
  2. 自動生成可能なトークンを省いた版（`WriteRedFactoryWithNoAutoCreatableTokens`）
  3. 必須フィールドだけの最小版（`WriteRedMinimalFactory`）
  4. `IdentifierToken`/`IdentifierNameSyntax` を `string` で受ける版（`withStringNames: true`）

`IfStatement` の例では 2 と 3 が生成される（§7.3）。

---

## 7. 生成コードの詳細（`IfStatementSyntax` を例に）

### 7.1 Green 型（`Syntax.xml.Internal.Generated.cs`）

```csharp
internal sealed partial class IfStatementSyntax : StatementSyntax
{
    internal readonly GreenNode? attributeLists;      // リストは GreenNode として保持
    internal readonly SyntaxToken ifKeyword;
    internal readonly SyntaxToken openParenToken;
    internal readonly ExpressionSyntax condition;
    internal readonly SyntaxToken closeParenToken;
    internal readonly StatementSyntax statement;
    internal readonly ElseClauseSyntax? @else;

    // ① 診断・注釈付き（SetDiagnostics/SetAnnotations から使用）
    internal IfStatementSyntax(SyntaxKind kind, ..., DiagnosticInfo[]? diagnostics, SyntaxAnnotation[]? annotations)
      : base(kind, diagnostics, annotations)
    {
        this.SlotCount = 7;
        if (attributeLists != null) { this.AdjustFlagsAndWidth(attributeLists); this.attributeLists = attributeLists; }
        this.AdjustFlagsAndWidth(ifKeyword);  this.ifKeyword = ifKeyword;
        ...                                   // 各子で幅とフラグを畳み込む
    }
    // ② パーサー用（SyntaxFactoryContext 付き → async/クエリ文脈フラグを記録）
    internal IfStatementSyntax(..., SyntaxFactoryContext context) : base(kind) { this.SetFactoryContext(context); ... }
    // ③ 文脈なし（static SyntaxFactory 用）
    internal IfStatementSyntax(...) : base(kind) { ... }

    public SyntaxToken IfKeyword => this.ifKeyword;   // 型付きアクセサ
    ...
    internal override GreenNode? GetSlot(int index) => index switch {
        0 => this.attributeLists, 1 => this.ifKeyword, 2 => this.openParenToken,
        3 => this.condition, 4 => this.closeParenToken, 5 => this.statement, 6 => this.@else, _ => null };

    internal override SyntaxNode CreateRed(SyntaxNode? parent, int position)
        => new CSharp.Syntax.IfStatementSyntax(this, parent, position);      // ★ Green→Red の接点

    public override void Accept(CSharpSyntaxVisitor visitor) => visitor.VisitIfStatement(this);

    public IfStatementSyntax Update(...)   // 子が 1 つでも変われば新ノード（診断・注釈は引き継ぐ）、同じなら this
    internal override GreenNode SetDiagnostics(DiagnosticInfo[]? d) => new IfStatementSyntax(Kind, ..., d, GetAnnotations());
    internal override GreenNode SetAnnotations(SyntaxAnnotation[]? a) => new IfStatementSyntax(Kind, ..., GetDiagnostics(), a);
}
```

ポイント:
- 子はフィールドに**型付きで直接**保持する（配列ではない）。`GetSlot` は switch 式でフィールドを返すだけ。
- **3 種類のコンストラクタ**は「診断・注釈を付けるか」「パース文脈を記録するか」の違い。
- `SetDiagnostics` は新しいインスタンスを作る。不変性を保つ設計で、§4.4 のキャッシュから診断付きノードが除外される理由もここにある。

### 7.2 Green ファクトリ（パーサー用 `ContextAwareSyntax` と static `SyntaxFactory`）

```csharp
internal partial class ContextAwareSyntax {          // Internal.Generated.cs:28675
    private SyntaxFactoryContext context;
    public IfStatementSyntax IfStatement(...) {      // :31649
#if DEBUG
        if (ifKeyword.Kind != SyntaxKind.IfKeyword) throw new ArgumentException(nameof(ifKeyword));
        ...                                           // 検査は DEBUG のみ（リリースのパーサーは無検査で高速）
#endif
        return new IfStatementSyntax(SyntaxKind.IfStatement, attributeLists.Node, ..., this.context);  // 子 7 個→キャッシュ無し
    }
    public ElseClauseSyntax ElseClause(SyntaxToken elseKeyword, StatementSyntax statement) {   // :31665
        int hash;
        var cached = CSharpSyntaxNodeCache.TryGetNode((int)SyntaxKind.ElseClause, elseKeyword, statement, this.context, out hash);
        if (cached != null) return (ElseClauseSyntax)cached;                  // 子 2 個→キャッシュ有り
        var result = new ElseClauseSyntax(SyntaxKind.ElseClause, elseKeyword, statement, this.context);
        if (hash >= 0) SyntaxNodeCache.AddNode(result, hash);
        return result;
    }
}
```

パーサーは `_syntaxFactory`（`ContextAwareSyntax` のインスタンス）経由でノードを作る。`SyntaxFactoryContext`（`Parser/SyntaxFactoryContext.cs`）はパーサーが `async` メソッドやクエリ式の中にいるかを保持していて、ノードのフラグに焼き込まれる。static `InternalSyntax.SyntaxFactory`（`:34102-`）は文脈なしの同等品で、公開 `SyntaxFactory` から呼ばれる。

### 7.3 Red 型（`Syntax.xml.Syntax.Generated.cs`）

```csharp
public sealed partial class IfStatementSyntax : StatementSyntax
{
    private SyntaxNode? attributeLists;     // 子「ノード」の Red キャッシュ（遅延生成）
    private ExpressionSyntax? condition;
    private StatementSyntax? statement;
    private ElseClauseSyntax? @else;
    // ※ トークン用フィールドは無い（SyntaxToken は struct で毎回作る）

    internal IfStatementSyntax(InternalSyntax.CSharpSyntaxNode green, SyntaxNode? parent, int position)
      : base(green, parent, position) { }

    public SyntaxToken IfKeyword => new SyntaxToken(this,
        ((InternalSyntax.IfStatementSyntax)this.Green).ifKeyword, GetChildPosition(1), GetChildIndex(1));
    public ExpressionSyntax Condition => GetRed(ref this.condition, 3)!;
    public ElseClauseSyntax? Else => GetRed(ref this.@else, 6);
    public override SyntaxList<AttributeListSyntax> AttributeLists => new(GetRed(ref this.attributeLists, 0));

    internal override SyntaxNode? GetNodeSlot(int index) => index switch {
        0 => GetRedAtZero(ref this.attributeLists)!, 3 => GetRed(ref this.condition, 3)!,
        5 => GetRed(ref this.statement, 5)!, 6 => GetRed(ref this.@else, 6), _ => null };
    internal override SyntaxNode? GetCachedSlot(int index) => ...;   // 生成済みのものだけ返す（位置計算の高速化）

    public IfStatementSyntax Update(...) {   // 変更があれば公開 SyntaxFactory で新ノードを作り、注釈を引き継ぐ
        ...
    }
    public IfStatementSyntax WithCondition(ExpressionSyntax condition) => Update(this.AttributeLists, this.IfKeyword, ..., condition, ...);
    public new IfStatementSyntax AddAttributeLists(params AttributeListSyntax[] items) => WithAttributeLists(this.AttributeLists.AddRange(items));
    internal override StatementSyntax WithAttributeListsCore(...) => WithAttributeLists(...);   // Override="true" の橋渡し
}
```

### 7.4 公開 API（`Syntax.xml.Main.Generated.cs`）

```csharp
public static partial class SyntaxFactory {
    // 完全版：Red の引数から Green を取り出し、Green ファクトリで作り、CreateRed() で新しいルートとして返す
    public static IfStatementSyntax IfStatement(SyntaxList<AttributeListSyntax> attributeLists, SyntaxToken ifKeyword,
        SyntaxToken openParenToken, ExpressionSyntax condition, SyntaxToken closeParenToken, StatementSyntax statement, ElseClauseSyntax? @else)
    {
        if (ifKeyword.Kind() != SyntaxKind.IfKeyword) throw new ArgumentException(nameof(ifKeyword));   // 常時検査
        ...
        return (IfStatementSyntax)Syntax.InternalSyntax.SyntaxFactory.IfStatement(
            attributeLists.Node.ToGreenList<...>(), (InternalSyntax.SyntaxToken)ifKeyword.Node!, ...,
            (InternalSyntax.ExpressionSyntax)condition.Green, ...).CreateRed();
    }
    // 自動生成可能トークンを省いた版
    public static IfStatementSyntax IfStatement(SyntaxList<AttributeListSyntax> attributeLists, ExpressionSyntax condition, StatementSyntax statement, ElseClauseSyntax? @else)
        => SyntaxFactory.IfStatement(attributeLists, SyntaxFactory.Token(SyntaxKind.IfKeyword), SyntaxFactory.Token(SyntaxKind.OpenParenToken),
                                     condition, SyntaxFactory.Token(SyntaxKind.CloseParenToken), statement, @else);
    // 最小版
    public static IfStatementSyntax IfStatement(ExpressionSyntax condition, StatementSyntax statement) => ...;
}

public partial class CSharpSyntaxRewriter : CSharpSyntaxVisitor<SyntaxNode?> {
    public override SyntaxNode? VisitIfStatement(IfStatementSyntax node)
        => node.Update(VisitList(node.AttributeLists), VisitToken(node.IfKeyword), VisitToken(node.OpenParenToken),
                       (ExpressionSyntax?)Visit(node.Condition) ?? throw new ArgumentNullException("condition"), ...,
                       (ElseClauseSyntax?)Visit(node.Else));   // 子が何も変わらなければ Update が this を返す → 部分木を共有
}
```

`CSharpSyntaxVisitor`/`<TResult>` には全ノード型の `VisitXxx` が `DefaultVisit` 呼び出しとして生成される（Main.Generated.cs:17, :771）。

### 7.5 生成量（集計）

| 項目 | 数 |
|---|---:|
| Green ノード型（`Internal.Generated.cs`） | 291（うち抽象 41） |
| Red ノード型（`Syntax.Generated.cs`） | 291 |
| 公開 `SyntaxFactory` のノード生成 static メソッド | 575 |
| ANTLR 文法 `CSharp.Generated.g4` | 1,811 行 |
| 生成テスト `Syntax.Test.xml.Generated.cs` | 20,977 行 |

**手書きと生成の分担**：`Syntax/` 直下の `IfStatementSyntax.cs` などの手書き partial は、生成型に便宜メソッドや互換用の旧オーバーロードを追加している。`SyntaxKind.cs`（列挙）と `SyntaxFacts`（キーワード表）は手書き。

---

## 8. Red ツリー（`Core/Portable/Syntax/SyntaxNode.cs` ほか）

### 8.1 `SyntaxNode`

```csharp
public abstract partial class SyntaxNode {           // SyntaxNode.cs:27
    private readonly SyntaxNode? _parent;
    internal SyntaxTree? _syntaxTree;
    internal GreenNode Green { get; }
    internal int Position { get; }                   // 絶対位置（leading trivia の先頭）
    public TextSpan FullSpan => new(Position, Green.FullWidth);
    public TextSpan Span { get { /* Position + LeadingTriviaWidth, FullWidth - 前後trivia */ } }
}
```

- Red ノード = **Green + 親 + 絶対位置** の薄いラッパ。Span は都度 Green の幅から計算する。
- **子の遅延生成**（`GetRed`, `:172-187`）：
  ```csharp
  var green = this.Green.GetSlot(slot);
  Interlocked.CompareExchange(ref field, (T)green.CreateRed(this, this.GetChildPosition(slot)), null);
  ```
  初回アクセス時に Red を作ってフィールドにキャッシュする。ロックを使わない CAS で、**複数スレッドから同時にアクセスしても同じインスタンス**が返る（負けた方は捨てる）。`GetRedAtZero` はスロット 0 用で、位置計算が不要（親と同じ位置）。
- **子の位置** `GetChildPosition(index)`（`:617-645`）：既に生成済みの兄弟（`GetCachedSlot`）があればその `EndPosition` から、無ければ Green の幅を左から足して求める。
- **SyntaxTree の遅延決定**（`CSharp/Portable/Syntax/CSharpSyntaxNode.cs:45-80`）：`_syntaxTree` が null なら親をたどり、ルートまで行っても無ければ `CSharpSyntaxTree.CreateWithoutClone(root, ...)` を CAS で設定する。`SyntaxFactory` で作ったノードも、問い合わせた時点で自動的にツリーを持つ。

### 8.2 トークン・trivia・リストは struct

- `public readonly struct SyntaxToken`（`SyntaxToken.cs:22-40`）= `(Parent, Node(Green), Position, Index)`。**トークンごとに Red オブジェクトを作らない**ので、ツリーをたどってもトークン分のヒープ割り当てが発生しない。
- `SyntaxTrivia`、`SyntaxTokenList`、`SyntaxTriviaList`、`SyntaxList<T>`、`SeparatedSyntaxList<T>`、`ChildSyntaxList` も struct。
- `SyntaxList<T>` の実体は Red の `SyntaxList` ノード（`WithTwoChildren` など）。

### 8.3 弱参照による Red キャッシュ（巨大メソッド本体対策）

`InternalSyntax.SyntaxList.WithManyChildren.CreateRed`（`:61-70`）は、親の `ShouldCreateWeakList()` が真なら `WithManyWeakChildren`／`SeparatedWithManyWeakChildren` を作り、子の Red を **`WeakReference`** で保持する（`SyntaxNode.GetWeakRedElement`, `:252-284`）。

C# の `ShouldCreateWeakList`（`CSharp/Portable/Syntax/CSharpSyntaxNode.cs:518-530`）が真になるのは、**メンバ宣言またはアクセサの直下にある `Block` の文リスト**。IDE で巨大なファイルを開いても、一度たどったメソッド本体の Red ノードは GC で回収でき、必要になれば Green から再生成される（Green は常に強参照で保持）。Red ノードを使い捨てられる、という Red-Green 設計の利点が活きている。

### 8.4 不変性と編集

- `WithXxx` / `Update` / `ReplaceNode` / `SyntaxRewriter` はいずれも、**変更されたノードからルートまでの経路上の Green だけを作り直し**、それ以外の部分木（Green）は共有する。Red は新しいルートから改めて遅延生成される。
- `ReplaceCore` などは `.AsRootOfNewTreeWithOptionsFrom(this.SyntaxTree)` で、元のパースオプションを引き継いだ新しいツリーを作る（`CSharpSyntaxNode.cs:462-476`）。
- `SyntaxAnnotation` は Green 側（外部テーブル）に付くため、書き換えをまたいでノードを追跡できる。

---

## 9. 公開 API としての入口

### 9.1 `CSharpSyntaxTree.ParseText`（`Syntax/CSharpSyntaxTree.cs:489-521`）

```csharp
using var lexer  = new InternalSyntax.Lexer(text, options);
using var parser = new InternalSyntax.LanguageParser(lexer, oldTree: null, changes: null, cancellationToken: cancellationToken);
var compilationUnit = (CompilationUnitSyntax)parser.ParseCompilationUnit().CreateRed();   // Green ルート → Red ルート
var tree = new ParsedSyntaxTree(text, text.Encoding, text.ChecksumAlgorithm, path, options,
                                compilationUnit, parser.Directives, diagnosticOptions, cloneRoot: false);
compilationUnit._syntaxTree = tree;
tree.VerifySource();
```

- **Green ルートに対して `CreateRed()` を 1 回呼ぶだけ**で、Red は `CompilationUnitSyntax` 1 個しか作られない。その下は §8.1 のとおり、アクセスされた時点で生成される。
- `ParseTextLazy`（`:414`）は `LazySyntaxTree` を返し、`GetRoot()` の初回呼び出しで解析する。
- `ParsedSyntaxTree`（`CSharpSyntaxTree.ParsedSyntaxTree.cs:20`）は `SourceText`・オプション・ルート・`DirectiveStack` を保持する。

### 9.2 増分解析 `WithChangedText`（`:535-590`）

`newText.GetChangeRanges(oldText)` で変更範囲を取り、変化が無ければ `this` を返す。そうでなければ `new LanguageParser(lexer, oldTree.GetRoot(), changes)` で §3.12 の Blender による増分解析を行い、`ParsedSyntaxTree(cloneRoot: true)` を作る。テキスト全体が置き換わった場合は通常の全体解析にする。

### 9.3 `SyntaxFactory.ParseXxx`（`Syntax/SyntaxFactory.cs:1718-1962`）

`ParseExpression`・`ParseStatement`・`ParseTypeName`・`ParseName`・`ParseMemberDeclaration`・`ParseCompilationUnit`・`ParseParameterList`・`ParseArgumentList` などは、共通して次のように動く。

```csharp
using (var lexer = MakeLexer(text, offset, options))
using (var parser = MakeParser(lexer)) {
    var node = parser.ParseExpression();
    if (consumeFullText) node = parser.ConsumeUnexpectedTokens(node);   // 余ったトークンは ERR_UnexpectedToken 付きスキップ trivia
    return CreateRed<ExpressionSyntax>(node, lexer.Options);             // CreateRed + CreateWithoutClone で木を付与
}
```

### 9.4 一連の流れ（`if (a) b();` を例に）

```text
"if (a) b();"
 Lexer:   [if·][(][a][)·][b][(][)][;]          （· = trailing Space。if·, )· は共有トークン表から）
          IdentifierToken "a","b" は StringTable でインターン済み文字列を持つ SyntaxIdentifier
 Parser:  ParseStatementCore → case IfKeyword (:8324) → ParseIfStatement (:10016)
            EatToken(IfKeyword), EatToken(OpenParenToken), ParseExpressionCore() → IdentifierName(a)  [キャッシュ対象]
            EatToken(CloseParenToken), ParseEmbeddedStatement() → ExpressionStatement(InvocationExpression(...))
          _syntaxFactory.IfStatement(attrs, if, (, cond, ), stmt, else: null)   [子 7 個 → キャッシュ無し]
 Green:   IfStatementSyntax(FullWidth=11, SlotCount=7, Flags=IsNotMissing)
 Red:     tree.GetRoot() → CompilationUnitSyntax（Green.CreateRed）
          .Members[0] → GlobalStatementSyntax → .Statement → IfStatementSyntax（GetRed で遅延生成・CAS でキャッシュ）
          .IfKeyword → new SyntaxToken(this, green.ifKeyword, GetChildPosition(1), 1)（struct、割り当て無し）
```

---

## 10. 設計上の要点まとめ

1. **Red-Green Tree**：不変・位置なし・親なしの Green（共有と増分再利用のため）と、遅延生成される親・位置付きの Red（使いやすい API のため）を分けている。Red は使い捨てで、メソッド本体のリストは弱参照で持つ。
2. **完全な忠実度（full fidelity）**：全ての文字がトークンか trivia（空白・コメント・ディレクティブ・無効化テキスト・スキップされたトークン）に属し、`ToFullString()` で元のテキストに戻る。IDE の整形やリファクタリングの土台になる。
3. **徹底したメモリ共有**：Green のヘッダは 8 バイト、診断と注釈は外部の弱参照テーブル。共有トークン表、トークンのサブクラス分割、空白とトークンのテキスト鍵キャッシュ（スレッド間共有）、文字列インターン（プロセス共有）、子 3 個以下のノードキャッシュ、要素数 0・1 のリストはオブジェクトを作らない。
4. **宣言的なノード定義と自動生成**：約 290 のノード型の Green／Red 型、3 種類のファクトリ、Visitor、Rewriter、`With*`／`Update`、テスト、ANTLR 文法までを `Syntax.xml` 1 ファイルから生成する。C# は**ビルド時のソースジェネレータ**で生成し、確認用に生成物もコミットする。新しい構文を追加するときは、`Syntax.xml` にノードを足し、`SyntaxKind` を足し、パーサーに解析コードを書く。
5. **パーサーは失敗しない**：欠落トークン、スキップ trivia、`TerminatorState` による再同期、ResetPoint による投機的解析で、どんな入力からも 1 つのツリーを作る。
6. **文脈依存性の管理**：文脈キーワードはレクサーでは識別子とし、パーサーが決める。async/クエリ/field の文脈はノードのフラグに記録し、増分再利用の可否判定に使う。
7. **増分解析**：Blender が旧ツリーの Green ノードを再利用する。診断を含むノード、パーサーが合成したトークン、ディレクティブ状態の変化は、安全側に倒して再利用しない。
8. **深い入力への保護**：明示スタックによる反復、`StackGuard`、不正トークン 200 個の上限、全体失敗時に入力全体をスキップ trivia にするフォールバック。
