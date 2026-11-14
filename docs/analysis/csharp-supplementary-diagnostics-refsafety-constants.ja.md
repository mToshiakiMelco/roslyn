# Roslyn C# 補足トピック：診断の基盤・ref 安全性の解析・定数畳み込み

対象: `src/Compilers/CSharp/Portable/Errors/`、`Compilation/CSharpDiagnosticFilter.cs`、`Binder/RefSafetyAnalysis.cs`・`Binder/Binder.ValueChecks.cs`・`Binder/SafeContext.cs`、`Core/Portable/ConstantValue*.cs`、`Binder/Binder_Operators.cs`・`Binder_Conversions.cs`・`Symbols/ConstantValueUtils.cs` ほか
基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。パスは特記なき限り `src/Compilers/CSharp/Portable/` からの相対パス。

関連ドキュメント：`csharp-binder.ja.md`（バインダ）、`csharp-compiler-dataflow-analysis.ja.md`（フロー解析）、`csharp-lowering.ja.md`・`csharp-codegen-emit.ja.md`（定数の後段での扱い）

---

## A. 診断の基盤

### A.1 `ErrorCode` 列挙（`Errors/ErrorCode.cs`、2,529 行）

- **列挙値がそのまま公開の診断番号**：`ERR_BadBinaryOps = 19` → CS0019、`WRN_UnreferencedField = 169` → CS0169（`:31`, `:146`）。
- **接頭辞が重大度を決める**：`ERR_`（エラー）、`WRN_`（警告）、`FTL_`（致命的。扱いはエラー）、`INF_`（情報）、`HDN_`（非表示）。重大度を記録する属性や表は無く、**名前だけが根拠**になる（A.2 の生成器が名前から表を作る）。
- C# のリリースごとに `#region` で区切られている。廃止したコードもコメントとして残し、番号の再利用を防ぐ（例 `//FTL_InternalError = 1,`, `:13`）。
- 「機能が使えない」エラーは言語バージョンごとに別の番号がある。テレメトリで区別できるようにするため（コメント `:1274`）。
  - 例：`ERR_FeatureNotAvailableInVersion1 = 8022`、`…7_3 = 8370`、`…11 = 9058`、`…14 = 9327`、`…15 = 9399`。
  - プレビュー機能用に `ERR_FeatureInPreview = 8652`。
- 末尾のチェックリスト（`:2518-2527`）：コードを追加したら、次を行う。
  1. `eng/generate-compiler-code.cmd` で再生成する。
  2. `IsBuildOnlyDiagnostic` と `GetWarningLevel` を更新する。
  3. テストを更新する。
  4. `CSharpResources.resx` にメッセージと `_Title` を追加する。

### A.2 重大度の表の自動生成（`ErrorFacts.Generated.cs`）

- 生成器：`src/Tools/CompilerGeneratorTools/Source/CSharpErrorFactsGenerator/Program.cs`。
  - `ErrorCode.cs` を 1 行ずつ読み、`WRN_`/`FTL_`/`INF_`/`HDN_` で始まる行のメンバー名を集める（`:46-61`）。
  - `IsWarning`/`IsFatal`/`IsInfo`/`IsHidden` の 4 つの `switch` メソッドを持つ `partial class ErrorFacts` を出力する（`:64-132`）。
- `eng/generate-compiler-code.cs` から呼ばれ、`Generated/ErrorFacts.Generated.cs` を作る（構文編 §1.3・BoundTree 編 §2.1 と同じスクリプト）。
- `ErrorFacts.GetSeverity`（`Errors/ErrorFacts.cs:106`）は、`IsWarning` → `IsInfo` → `IsHidden` の順に調べ、どれでもなければ `Error`（`FTL_` もここに入る）。

### A.3 `ErrorFacts.cs`（手書き部分）

- **`NullableWarnings`**（`:22`）：null 許容性に関する警告 ID（"CS8600" など）の集合。A.6 のフィルタで `#nullable` の文脈による抑止に使う。
- **メッセージ**：`GetMessage`/`GetTitle`（`_Title`）/`GetDescription`（`_Description`）。リソースのキーは列挙の名前そのもの（`CSharpResources.resx`。例：`WRN_UnreferencedField` → "The field '{0}' is never used"）。
- **`GetWarningLevel`**（`:201`）＝**警告ウェーブ**：
  - 情報と非表示は常に出す（アナライザーが依存するため）。
  - 警告には、追加されたリリースに応じたレベルを割り当てる。

    | レベル | 例 |
    |---|---|
    | 10 | `WRN_UnassignedInternalRefField` |
    | 9 | `WRN_InterceptsLocationAttributeUnsupportedSignature` |
    | 8 | `WRN_AddressOfInAsync` |
    | 7 | `WRN_LowerCaseTypeName`（.NET 7 / C# 11 で追加） |
    | 6 | `WRN_PartialMethodTypeDifference` |
    | 5 | `WRN_PrecedenceInversion` など |
    | 1〜4 | 従来の警告 |
    | 0 | エラー |

  - **既存のコードに新しい警告を出すと互換性を壊す**ため、新しい警告はプロジェクトの警告レベル（`<AnalysisLevel>`/`/warn:N`）を上げたときだけ出る。
- **`IsBuildOnlyDiagnostic`**（`:588`）：後段（Lowering・Emit・エントリポイント判定など）でしか出ず、`SemanticModel.GetDiagnostics` では出ない診断の網羅的な `switch`。
  - 例：`WRN_UnreferencedField`、`ERR_NoEntryPoint`、`ERR_InsufficientStack`、`ERR_TooManyLocals`、`ERR_RefLocalAcrossAwait`。
  - IDE（LSP）側の一覧と同期させる必要がある、というコメントがある。

### A.4 診断オブジェクトの階層

| クラス | 役割 |
|---|---|
| `MessageProvider`（`Errors/MessageProvider.cs:17`、シングルトン） | 言語共通の `CommonMessageProvider` の C# 実装。接頭辞 `"CS"`、`ErrorFacts` への委譲、言語共通のエラー（コマンドライン・メタデータ・PDB）と C# のコードの対応付け |
| `CSDiagnosticInfo`（`Errors/CSDiagnosticInfo.cs:16`） | コードと引数（シンボルを含む）。位置は持たない |
| `CSDiagnostic`（`Errors/CSDiagnostic.cs`） | 位置付きの診断。`WithSeverity`、`WithIsSuppressed` |
| `SyntaxDiagnosticInfo`（`Errors/SyntaxDiagnosticInfo.cs:11`） | パーサー用。Green ノードからの**相対オフセットと幅**を持つ（構文編 §3.2）。位置を持たない Green ノードに付けられる |
| `LazyDiagnosticInfo`（`Errors/LazyDiagnosticInfo.cs`） | 診断を出すかどうか自体を後で決める。`GetResolvedInfo()` が 1 回だけ計算し（`Interlocked.CompareExchange`）、`Void` なら診断を捨てる |
| `LazyObsoleteDiagnosticInfo` | バインド時点で、シンボルの `[Obsolete]` 属性がまだ解析されていない場合に使う。後で属性を完成させて判断する |

`LazyObsoleteDiagnosticInfo` は、シンボルの属性の解析を待たずにバインドを進めるための仕組み（宣言とシンボル編の「遅延完了」と組み合わせて使う）。

### A.5 言語バージョンによる機能の制御（`Errors/MessageID.cs`）

- `enum MessageID`（`:10`）に、機能ごとの ID（例 `IDS_FeatureRefFields`）がある。
- `RequiredVersion()`（`:486`）：機能 → 必要な `LanguageVersion` の巨大な `switch`（例：`IDS_FeatureFieldKeyword` → `CSharp14`、`IDS_FeatureRefFields` → C# 11）。
- `CheckFeatureAvailability`（`:366-464`）：使えなければ診断を追加して false を返す。必要なバージョンが Preview なら `ERR_FeatureInPreview`、それ以外は「**現在の**言語バージョンに対応する」`ERR_FeatureNotAvailableInVersionN`（`LanguageVersionExtensionsInternal.GetErrorCode`, `LanguageVersion.cs:337`）。
- バインダ編でも触れたとおり、**バージョンの検査はほとんどバインダで行い**、パーサーは解析の分岐に使うだけ。

### A.6 `#pragma warning` と診断のフィルタ

- **`#pragma warning`**（`Syntax/CSharpPragmaWarningStateMap.cs:37`）：
  - 全ての `#pragma warning` を順にたどり、「位置 → 全体の状態＋診断 ID ごとの状態」のソート済み配列を作る（`:80-106`）。
  - `disable` → Disabled、`restore` → Default、`enable` → Enabled。番号は `CS` 付きの ID に変換し、識別子（アナライザーの ID）はそのまま使う。
  - `CSharpSyntaxTree.GetPragmaDirectiveWarningState`（`Syntax/CSharpSyntaxTree.cs:703`）が遅延構築して二分探索する。
- **`CSharpDiagnosticFilter`**（`Compilation/CSharpDiagnosticFilter.cs`）：コンパイルオプションに基づいて、各診断の最終的な報告方法（`ReportDiagnostic`）を決める。`GetDiagnosticReport`（`:135`）の順序：
  1. **null 許容性の警告**は、その位置の `#nullable` の状態で抑止する（`:157-177`）。
  2. **警告レベル**：`if (diagnosticWarningLevel > warningLevelOption) return Suppress;`（`:180`。警告ウェーブの実体）。
  3. 個別の `/nowarn:N`、`/warnaserror[+-]:N`。
  4. アナライザーの独自の重大度。
  5. `.editorconfig`（構文木ごと） → グローバルなアナライザー設定。
  6. 指定が無ければ「既定で有効か」。
  7. **`#pragma` の状態**：Disabled なら抑止の印を付ける。**診断自体は残して `IsSuppressed` にする**（`/reportSuppressedDiagnostics` のため, `:114`）。
  8. 全体のオプション：`/warnaserror` は警告だけをエラーに上げる。`/nowarn`（全抑止）は警告と情報だけを下げ、非表示は下げない。
  9. `WRN_Experimental` は、指定が無ければエラーとして扱う（`:322`）。

---

## B. ref 安全性の解析（`RefSafetyAnalysis`）

### B.1 目的と実行のタイミング

- `ref struct`（`Span<T>` など）、`scoped`、ref フィールド、`[UnscopedRef]`、ref 戻り値について、**参照が寿命を超えて逃げないこと（ref safety）**を検査する。C# 11 の「低レベル struct の改善」の仕様に基づく。
- **バインド後の独立したパス**として、完成した BoundTree を歩く。
  - `MethodCompiler.cs:2001`：メソッド本体のバインド直後に `RefSafetyAnalysis.Analyze(compilation, method, methodBody, diagnostics)`。
  - `:1055`：フィールド初期化子を含むコンストラクタ。初期化子を新しい `BoundBlock` で包み、ローカルスコープを正しく作る。
- 入口（`Binder/RefSafetyAnalysis.cs:19`）：

  ```csharp
  var visitor = new RefSafetyAnalysis(compilation, symbol, node,
      inUnsafeRegion: InUnsafeContext(compilation, symbol),
      useUpdatedEscapeRules: symbol.ContainingModule.UseUpdatedEscapeRules, diagnostics);
  visitor.Visit(node);   // 深い木では CancelledByStackGuardException → エラー
  ```

- 本体は `RefSafetyAnalysis.cs`（1,522 行）と、`Binder.ValueChecks.cs` の中にある複数の `partial class RefSafetyAnalysis` の部分。

### B.2 `SafeContext`：寿命を表す数（`Binder/SafeContext.cs`）

```csharp
internal readonly struct SafeContext {               // uint を 1 つ包むだけ
    CallingMethod = 0   // 呼び出し元まで逃げてよい
    ReturnOnly    = 1   // return では返せるが、ref 引数経由では逃がせない
    CurrentMethod = 2   // このメソッドの中だけ
    Empty         = uint.MaxValue   // 記憶域が確保されない
    Narrower() => value + 1;  Wider() => value - 1;   // 入れ子のブロックで深くなる
    IsConvertibleTo(other) => _value <= other._value;  // 小さいほど広い
}
```

**値が小さいほど寿命が長い**。3 以上は入れ子のブロックの深さを表す。兄弟のスコープは交わらないので、単純な深さの数で十分に表せる。`Intersect` は狭い方、`Union` は広い方を返す。

### B.3 ローカル変数の追跡

- `_localScopeDepth`（現在のブロックの深さ、初期値 `CurrentMethod`）と、`_localEscapeScopes: Dictionary<LocalSymbol, (RefEscapeScope, ValEscapeScope)>`。
  - **ref-safe-to-escape**：その変数への参照がどこまで逃げてよいか。
  - **safe-to-escape**：その変数の値（ref struct の中身）がどこまで逃げてよいか。
- ブロックに入るたびに `LocalScope`（`:94`）で深さを 1 つ深くする。ただしメソッドの最上位のブロックは深くしない（最上位のローカルは値渡しのパラメータと同じ寿命）。
- 宣言の規則（`AddLocalScopes`, `:551`。コードに仕様の表が引用されている）：

  | 宣言 | ref-safe-to-escape | safe-to-escape |
  |---|---|---|
  | `Span<int> s` | current method | calling method |
  | `scoped Span<int> s` | current method | current method |
  | `ref Span<int> s` | calling method | calling method |
  | `scoped ref Span<int> s` | current method | calling method |

- `scoped` でない変数は、**初期化子から寿命を推論する**（`VisitLocalDeclaration`, `:594`）。例えば `Span<int> s = stackalloc int[10];` なら、`s` は現在のメソッドの外へ逃がせない。
- 検査する位置：
  - `return`（`ValidateEscape(expr, ReturnOnly, …)`, `:634`）
  - 代入（`:654`）、複合代入、分解、`foreach`、コレクション式、パターン、`await`
- ラムダとローカル関数は、それぞれ別の `RefSafetyAnalysis` で解析する（`:374-390`）。

### B.4 逃げる範囲の計算（`Binder.ValueChecks.cs`）

- `GetRefEscape`（`:3675`）と `GetValEscape`（`:4323`）は、式の種類ごとに「逃げてよい最も広い範囲」を計算する。エラーのある式は `CallingMethod`（何も推論しない）。
- `ValidateEscape`（`:3652`）→ `CheckRefEscape`/`CheckValEscape`（`:3956`/`:5016`）で、逃げてはいけない所へ逃げていないかを検査する。
- **引数の混合規則**（`CheckInvocationArgMixing`, `:3139`）：ref struct の `ref`/`out` 引数がある呼び出しでは、寿命の短い値を、寿命の長い ref struct の中へ書き込めないようにする。旧規則では「書き込み可能な ref struct 引数の safe-to-escape の和より狭い引数があってはならない」（`ERR_CallArgMixing`, `:3211`）。
- **規則の世代は呼び出される側のメソッドごとに選ぶ**（`MethodInfo.UseUpdatedEscapeRules`, `:44`）。古い規則でコンパイルされたアセンブリのメソッドは、古い規則で解析する。

### B.5 新旧の規則の切り替え

- ソースのモジュール：言語バージョンが C# 11 以上、またはランタイムが ref フィールドに対応していれば、新しい規則を使う（`Symbols/Source/SourceModuleSymbol.cs:736`）。その場合、モジュールに `[RefSafetyRules(11)]` を付ける。
- 参照アセンブリ：モジュールの `RefSafetyRulesAttribute` のバージョンが 11 なら新しい規則（`Symbols/Metadata/PE/PEModuleSymbol.cs:724`）。

### B.6 診断と unsafe

- 主な診断：`ERR_RefReturnLocal`、`ERR_EscapeVariable`、`ERR_EscapeStackAlloc`、`ERR_EscapeCall`、`ERR_EscapeOther`、`ERR_CallArgMixing`。
- **unsafe 文脈の中では、多くが警告に下がる**（`inUnsafeRegion ? WRN_EscapeVariable : ERR_EscapeVariable`）。
- 「unsafe の進化」（プレビュー機能 `IDS_FeatureUnsafeEvolution`、`MemorySafetyRulesAttribute`）は、この解析ではエラーか警告かの判定にだけ関わる。逃げる範囲の規則自体は `UseUpdatedEscapeRules` で選ぶ。

---

## C. 定数と定数畳み込み

### C.1 `ConstantValue`（`Core/Portable/ConstantValue.cs`、`ConstantValueSpecialized.cs`）

- 種類（`ConstantValueTypeDiscriminator`, `:14`）：`Null`、`Bad`、整数型全般、`NInt`/`NUInt`、`Char`、`Boolean`、`Single`、`Double`、`String`、`Decimal`、`DateTime`（VB 用）。
- 特別な値：
  - `ConstantValue.Bad`：エラー（定数として扱うと誤りになる）。
  - `ConstantValue.Unset`：「まだ計算していない」の番兵。
  - `True`/`False`。
- **よく使う値はキャッシュする**：`Create(int)` は 0 と 1 なら共有のインスタンスを返す（`:187-191`）。
- `ConstantValueString` は文字列を **`Rope`**（連結を遅延する木）で持ち、`string` は必要になったときに作って `WeakReference` でキャッシュする。巨大な定数文字列の連結を安くするため。
- `ConstantValueSingle` は内部で `double` を持つ。コメントにあるとおり、C# は浮動小数点の定数畳み込みを最大精度で行い、`SingleValue` を読むときに丸める。

### C.2 二項・単項演算の畳み込み（`Binder/Binder_Operators.cs`）

`FoldBinaryOperator`（`:2753`）。戻り値は、定数でなければ null、エラーなら `ConstantValue.Bad`。

1. オペランドにエラーがあれば null。**null 許容型は定数にならない**（例外：`null == null` の持ち上げ比較は畳み込む。Dev10 から引き継いだ仕様違反, `:2910`）。
2. enum は基底の整数型で計算してから戻す（`:2678`）。
3. **0 除算**は unchecked 文脈でも `ERR_IntDivByZero`（`:4732`）。
4. bool、object の `==`、`float`/`double` の算術：桁あふれしない（IEEE の結果）。
5. **文字列連結**（`FoldStringConcatenation`, `:3198`）：

   ```csharp
   Rope leftValue = valueLeft.RopeValue ?? Rope.Empty;    // null の文字列定数は "" として扱う
   long newLength = (long)leftValue.Length + (long)rightValue.Length;
   return newLength > int.MaxValue ? ConstantValue.Bad : ConstantValue.CreateFromRope(Rope.Concat(leftValue, rightValue));
   ```

   長すぎれば `ERR_ConstantStringTooLong`。
6. `decimal`：実行時と同じ `System.Decimal` の演算で計算する。桁あふれは文脈に関係なく `ERR_DecConstError`。
7. `nint`/`nuint`：桁あふれはプラットフォーム依存なので、checked 文脈では `WRN_CompileTimeCheckedOverflow` を出し、**定数にしない**（null を返す）。
8. 整数：
   - **checked 文脈（定数式の既定）**では、桁あふれは `ERR_CheckedOverflow`（「checked モードでコンパイル時に桁あふれ」）。
   - 明示的な `unchecked` の中では桁あふれを許す（`:2532`/`:2588`）。仕様 7.5.12 により、定数式は明示的に `unchecked` で囲まない限り checked。

単項演算は `FoldUnaryOperator`（`:4473`）、条件演算子は `FoldConditionalOperator`（`:6077`）が同じ構造で畳み込む。

### C.3 変換の畳み込み（`Binder/Binder_Conversions.cs`）

- `FoldConstantConversion`（`:3799`）：
  - `default` リテラル → 目的の型の既定値。
  - `float`/`double` への恒等変換 → 目的の精度で丸め直す。
  - 数値・enum の変換 → `FoldConstantNumericConversion`。null 許容型が目的なら定数にしない。
  - 参照変換 → `null` 定数だけを保つ。
- `FoldConstantNumericConversion`（`:3901`）：
  - `decimal` の範囲外は常に `ERR_ConstOutOfRange`。
  - checked 文脈では範囲外を `ERR_ConstOutOfRangeChecked` にする。実行時には成功し得る場合は `WRN_ConstOutOfRangeChecked` を出し、定数にしない。
  - unchecked 文脈では、ビットを切り詰めて変換する。

### C.4 定数フィールド・循環の検出・enum

- **定数フィールドの評価**（`Symbols/ConstantValueUtils.cs:29` `EvaluateFieldConstant`）：初期化子を `ConstantFieldsInProgressBinder` の下でバインドし、`GetAndValidateConstantValue`（`:78`）で検査する。
  - 型パラメータ型 → `ERR_InvalidConstantDeclarationType`。
  - `const object x = "s"` のように string 以外の参照型への null でない定数 → `ERR_NotNullConstRefField`。
  - 定数でない → `ERR_NotConstantExpression`。
- **循環の検出**（`Symbols/Source/SourceFieldSymbol.cs`）：
  - フィールドの定数値は、最初は `ConstantValue.Unset`。
  - 定数の初期化子の中で別の定数フィールドを参照すると、`GetConstantValue(inProgress)`（`:237`）が**依存関係を記録して `Unset` を返す**。
  - 依存グラフを位相順に並べ（`ConstantEvaluationHelpers.cs:52`）、その順に評価する。循環を始めるフィールドに `ERR_CircConstValue`（`:333`）を報告し、値を Bad にする。
  - ローカル定数は自己参照を直接検出する（`SourceLocalSymbol.cs:695`）。
- **enum のメンバー**（`Symbols/Source/SourceEnumConstantSymbol.cs`）：値 0（`:147`）、明示的な値（`:164`）、**暗黙の値**（直前の明示的なメンバー＋オフセット, `:184`）の 3 種類。桁あふれは `ERR_EnumeratorOverflow`。

### C.5 定数が作られるその他の場所

| 場所 | 内容 |
|---|---|
| 既定の引数値 | `SourceComplexParameterSymbol.MakeDefaultExpression`（`:367`）。`decimal` の既定値は `DecimalConstantAttribute` で表す |
| 属性の引数 | `Binder_Attributes.AttributeExpressionVisitor`（`:760`）が `ConstantValueOpt` を `TypedConstant` にする。定数でなければ `ERR_BadAttributeArgument` |
| switch・パターン | 定数パターンは定数を要求する（`ERR_ConstantExpected`）。重複するラベルは値で検出する（`ERR_DuplicateCaseLabel`） |
| 定数の補間文字列（C# 10） | 全ての穴が書式指定なしの文字列定数である間だけ、`FoldStringConcatenation` で定数として畳み込む（`Binder_InterpolatedString.cs:42-151`） |

### C.6 後段での消費

| 段階 | 処理 |
|---|---|
| Lowering | `LocalRewriter.VisitExpressionImpl`（`Lowering/LocalRewriter/LocalRewriter.cs:249`）が、定数を持つ式（null 許容型以外）を **`BoundLiteral` に置き換え、演算子の部分木を捨てる**。`decimal` の定数は `new decimal(...)` の呼び出しにする |
| IL 生成 | `EmitConstantExpression`（`CodeGen/EmitExpression.cs:3644`）→ `ILBuilder.EmitConstantValue`（`ldc.i4.*`、`ldstr` など） |
| メタデータ | `MetadataWriter.PopulateConstantTableRows`（`Core/Portable/PEWriter/MetadataWriter.cs:2401`）がフィールド・パラメータ・プロパティの既定値を Constant テーブルに書く。**`decimal` の const フィールドはメタデータの定数にならず**（`FieldSymbol.IsMetadataConstant`, `Symbols/FieldSymbol.cs:140`）、`DecimalConstantAttribute` で表す |

---

## まとめ

- **診断**：重大度は `ErrorCode` の名前の接頭辞だけで決まり、生成器が表にする。新しい警告は警告ウェーブで既存のコードを壊さないようにする。`#pragma` で抑止した診断も `IsSuppressed` として残す。
- **ref 安全性**：「寿命」を単なる深さの数（`SafeContext`）で表し、宣言・初期化子・代入・return・呼び出しごとに逃げる範囲を計算する、バインド後の独立したパス。規則の世代は呼び出される側のメソッドごとに選ぶ。
- **定数**：`ConstantValue` は小さな値のキャッシュと Rope で軽量化している。畳み込みはバインダで行い、checked が定数式の既定。依存グラフの位相順で循環を検出する。Lowering で `BoundLiteral` に置き換わり、IL とメタデータに流れる。
