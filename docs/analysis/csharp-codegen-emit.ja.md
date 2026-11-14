# Roslyn C# IL 生成と Emit 詳細解説
## ― Lowering 済み BoundTree から IL・メタデータ・PE・PDB へ ―

対象:
- `src/Compilers/CSharp/Portable/CodeGen/`（C# 固有の IL 生成：`CodeGenerator`、`Optimizer` ほか）
- `src/Compilers/Core/Portable/CodeGen/`（言語共通：`ILBuilder`、`LocalSlotManager`、switch の生成器、`<PrivateImplementationDetails>`）
- `src/Compilers/CSharp/Portable/Emitter/`、`src/Compilers/Core/Portable/Emit/`（モジュールビルダー、Edit and Continue）
- `src/Compilers/Core/Portable/PEWriter/`、`NativePdbWriter/`（メタデータ・PE・PDB の書き出し）

基準コミット: `88579e780fe7`（2026-11-13）。行番号はこの時点のもの。

関連ドキュメント（同ディレクトリ）: `csharp-lowering.ja.md`（入力を作る Lowering）、`csharp-boundtree-boundnodes.ja.md`（BoundTree）

---

## 0. 要約

- `Compilation.Emit` は次の 3 段で進む。
  1. **全メソッドのコンパイル**：バインド → フロー解析 → Lowering → **IL 生成**。
  2. **モジュールの完成**：合成型、埋め込み属性、`<PrivateImplementationDetails>`。
  3. **書き出し**：`MetadataWriter` がメタデータと IL を作り、`PeWriter` が PE と PDB を書く。
- IL 生成は 2 層に分かれる。
  - **C# の `CodeGenerator`** が Lowering 済みの BoundTree を歩き、命令を出す。
  - **言語共通の `ILBuilder`** が命令を基本ブロックに溜め、最後に分岐の最適化・短縮形の選択・到達不能コードの削除・オフセットの確定をまとめて行う（`Realize`）。
- IL を出す前に **`Optimizer`（スタックスケジューリング）** が木を書き換える。ローカル変数を評価スタック上の値で置き換え（`dup` を使う）、`stloc`/`ldloc` の組を減らす。Release で効く最大の最適化。
- **Debug と Release の違い**：
  - シーケンスポイント用の `nop`
  - 戻り値用の一時変数と `}` へのステップ
  - ローカル変数のスロットの再利用
  - ラベルの付け替えと分岐の書き換え
  - スタックスケジューリングの範囲

  一方、**PDB を出すかどうかは IL を変えない**。
- switch の生成：
  - 整数は密度 50% 以上のバケットに分けて IL の `switch` 命令にし、バケット間は二分探索。
  - 文字列は 7 ケース以上なら FNV-1a ハッシュで分岐する。長さと特定の文字で分岐する方式もある。
- 書き出し：System.Reflection.Metadata（SRM）の `MetadataBuilder` と `ManagedPEBuilder` を使う。IL 中のトークンは**仮トークン**として出し、書き出し時に実際のハンドルに差し替える。
  - **決定的ビルド**：MVID と PDB の ID を内容のハッシュから作る。
  - Portable PDB：シーケンスポイント、スコープ、EnC 用のスロット表などを持つ。
- **Edit and Continue（EnC）**：前の世代の変数スロット、ラムダ、状態番号を対応付け、差分メタデータ（EncLog/EncMap）を書く。

---

## 1. Emit 全体の流れ

```text
Compilation.Emit                                        Core/Portable/Compilation/Compilation.cs:2943
 ├ CheckOptionsAndCreateModuleBuilder → CSharpCompilation.CreateModuleBuilder → PEAssemblyBuilder   CSharpCompilation.cs:3587/3625
 ├ CompileMethods → CSharpCompilation.CompileMethods → MethodCompiler.CompileMethodBodies           CSharpCompilation.cs:3654/3736
 │    名前空間を（ConcurrentBuild なら並列に）歩き、メソッドごとに:
 │      Bind → FlowAnalysis → Lower → GenerateMethodBody(IL 生成) → moduleBuilder.SetMethodBody   MethodCompiler.cs:837
 │    合成型・匿名型 → FreezePrivateImplementationDetails() → その合成メソッドのコンパイル         MethodCompiler.cs:153-195
 ├ リソース・XML ドキュメント → moduleBeingBuilt.CompilationFinished()
 └ SerializeToPeStream → SerializePeToStream → Cci.PeWriter.WritePeToStream                       Compilation.cs:3247/3405, PeWriter.cs:80
      （参照アセンブリも出力するなら metadataOnly: true, isDeterministic: true でもう一度 :3446）
```

メソッド本体（`MethodBody`）はモジュールビルダーに保存される。PE を書くときに、アダプタ（`MethodSymbolAdapter.GetBody`, `:345`）を通じて読み出される。

---

## 2. `MethodCompiler.GenerateMethodBody`（`CSharp/Portable/Compiler/MethodCompiler.cs:1636-1803`）

```csharp
var localSlotManager = new LocalSlotManager(variableSlotAllocatorOpt);      // EnC 時のみアロケータあり
var optimizations = compilation.Options.OptimizationLevel;
ILBuilder builder = new ILBuilder(moduleBuilder, localSlotManager, diagnostics, optimizations, method.AreLocalsZeroed);
var codeGen = new CodeGen.CodeGenerator(method, block, builder, moduleBuilder, diagnostics, optimizations, emittingPdb);
codeGen.Generate(out hasStackalloc);       // async の MoveNext なら yield/resume 位置も受け取る版（:1701）
...
return new MethodBody(builder.RealizedIL, builder.MaxStack, ..., localVariables, builder.RealizedSequencePoints, ...,
                      builder.RealizedExceptionHandlers, builder.AreLocalsZeroed, hasStackalloc, builder.GetAllScopes(), ...,
                      lambdaDebugInfo, closureDebugInfo, stateMachineTypeOpt?.Name, stateMachineHoistedLocalScopes, ...);
```

- ローカル変数が 0xFFFE 個を超えたら `ERR_TooManyLocals`（`:1742`）。
- **エラーのあるメソッドでは IL を生成しない**（`:1655`）。
- async の `MoveNext` では、await の yield/resume 位置の IL オフセットと、例外ハンドラのオフセットを `AsyncMoveNextBodyDebugInfo` として PDB に渡す。デバッガのステップ実行に使う。

---

## 3. `CodeGenerator`（`CSharp/Portable/CodeGen/`）

### 3.1 構造と出力スタイル（`CodeGenerator.cs`）

```csharp
public CodeGenerator(MethodSymbol method, BoundStatement boundBody, ILBuilder builder, PEModuleBuilder moduleBuilder,
                     BindingDiagnosticBag diagnostics, OptimizationLevel optimizations, bool emittingPdb)   // :86
```

**`ILEmitStyle`**（`:108-129`）：

| 条件 | スタイル |
|---|---|
| ユーザーコードを持たない合成メソッド（`!GenerateDebugInfo`） | `Release` |
| `OptimizationLevel.Debug` | `Debug` |
| Release かつ `DebugPlusMode` | `DebugFriendlyRelease` |
| それ以外 | `Release` |

`_emitPdbSequencePoints = emittingPdb && method.GenerateDebugInfo`（`:137`）。コメントにあるとおり、このフラグは**生成する IL に一切影響しない**。

コンストラクタの中で **`Optimizer.Optimize`** を実行する（`:141-144`、§5）。

### 3.2 生成と return

- `GenerateImpl`（`:283-321`）：
  1. 最初のドキュメントを設定する。
  2. `EmitStatement(_boundBody)`。
  3. 必要なら `HandleReturn()`。
  4. エラーが無ければ `_builder.Realize()`。
- **間接 return**（`ShouldUseIndirectReturn`, `EmitStatement.cs:844`）：次のどちらかなら、戻り値を一時変数（Debug では名前付きの `FunctionReturnValue`）に入れて共通の `ret` ラベルへ `br` する。
  - Debug スタイルでブロック本体を持つ場合。デバッガで戻り値を観察し、閉じ括弧 `}` にステップできるようにするため。
  - 例外ハンドラの中の場合。IL では `ret` できないので、`leave` で抜ける必要がある。

  `HandleReturn`（`:323-358`）は `}` にシーケンスポイントを置き、`ldloc; ret` を出す。それ以外は直接 `ret`。

### 3.3 シーケンスポイント

- `EmitSequencePointStatement`（`:387`）：中の文が命令を 1 つも出さなかった場合、**Debug スタイルでは `nop` を出す**（`:413`）。シーケンスポイントが無関係な次の命令に付かないようにするため。
- 一時変数（式の評価用）は、文の終わりごとに `ReleaseExpressionTemps()` で解放する（`:545`）。

### 3.4 文（`EmitStatement.cs`）

- `EmitStatement`（`:27`）は、Lowering 後に残る少数のノードだけを扱う：ブロック、シーケンスポイント類、式文、return、goto、ラベル、**条件付き goto**、throw、try、switch dispatch、ステートマシンのスコープ、`NoOp`。DEBUG ビルドでは、**全ての文の後で評価スタックが空**であることを assert する（`:108-113`）。
- **条件分岐 `EmitCondBranch`**（`:358`、`EmitCondBranchCore` `:393`）は条件の形で分岐する。
  - 定数 → 無条件 `br`、または何も出さない。
  - `&&`/`||` → 明示スタックで、値を作らずに分岐の連鎖へ展開する（`:424-489`）。
  - `x == null`、`x != 0` など → `brtrue`/`brfalse` に簡約する（`TryReduce`）。
  - 比較 → `blt`、`bge` などを 1 命令で出す。**逆の分岐命令も記録しておき**、後で ILBuilder が分岐を反転できるようにする。
  - `!x` → 分岐の向きを反転するだけ。`x is T` → `isinst` ＋ `brtrue`。
  - `float`/`double` は NaN があるので `brfalse` に簡約しない（`CanPassToBrfalse`, `:200`）。
- **try/catch/finally/filter**（`EmitTryStatement`, `:932`）：
  - 入口でスタックが空であることを要求する。
  - catch と finally を両方持つ try は、`try { try {…} catch {…} } finally {…}` の 2 重構造で出す。
  - 例外フィルタは `isinst` → 条件の評価 → `endfilter` の形にする（コメント `:1018-1055`）。
  - `PreferFaultHandler` が設定されていれば、finally を `fault` ハンドラ＋finally の複製として出す。
- **ローカル変数**（`DefineLocal`, `:1790`）：
  - 定数は `LocalConstantDefinition`（PDB にだけ載る）。
  - スタックローカル（§5）はスロットを作らない。
  - `fixed` の変数は `ByRef | Pinned`。
  - Debug スタイルでは `LocalDebugId(syntaxOffset, ordinal)` を付け、EnC で前の世代のスロットと対応付ける（`:1877`）。

### 3.5 switch（`EmitSwitchDispatch`, `EmitStatement.cs:1256`）

- 整数のキー → `_builder.EmitIntegerSwitchJumpTable`（§6.5）。
- `string`/`Span<char>`/`ReadOnlySpan<char>` のキー：
  - Lowering が `LengthBasedStringSwitchData` を用意していれば、**長さ → 特定位置の文字 → 文字列比較**の順に分岐する（`:1367`）。
  - そうでなければ、**7 ケース以上でハッシュによる分岐**にする（`SwitchStringJumpTableEmitter.ShouldGenerateHashTableSwitch`, `Core/.../SwitchStringJumpTableEmitter.cs:219`。Dev10 時代からのヒューリスティック）。`<PrivateImplementationDetails>.ComputeStringHash` を呼んで `uint` の値で整数 switch し、各バケットの中で `string.op_Equality`（Span なら `SequenceEqual`）で比較する。
  - ハッシュは **FNV-1a**（初期値 2166136261、乗数 16777619。`CSharp/Portable/Compiler/MethodBodySynthesizer.Lowered.cs:24-35`）。コンパイル時と同じ関数を `<PrivateImplementationDetails>` に合成して実行時にも使う。JIT が範囲検査を外へ出せるよう、goto のループで書かれている。

### 3.6 式（`EmitExpression.cs`）

- `EmitExpression(expr, used)`（`:36`）：
  - **定数が最優先**。使われない定数は何も出さない。`decimal` と null 許容型以外の定数は `EmitConstantExpression`（`:3644`）。
  - 定数畳み込みはバインダと Lowering で済んでいるので、IL 生成は `ConstantValueOpt` を使うだけ。
  - それ以外はスタックガード付きの巨大な `switch`（`EmitExpressionCore`, `:94`）。
- **呼び出しの命令の選択**（`EmitInstanceCallExpression`, `:1783`、規則は `:1940-2017`）：

  | 受け手 | 命令 |
  |---|---|
  | 参照型で、非仮想メソッドかつ null でないことが分かっている（`this`、`new`、ボックス化、デリゲート生成、クロージャの環境など。`CanUseCallOnRefTypeReceiver`, `:1597`） | `call` |
  | 参照型で、それ以外 | **`callvirt`**（非仮想メソッドでも null 検査のために使う。C# の暗黙の null 検査の正体） |
  | 値型で、その struct で宣言されたメソッド | アドレスを取って `call` |
  | 値型で、継承した仮想メソッド（`ToString` など） | `constrained. T callvirt`（ボックス化を避ける） |
  | ジェネリックの型パラメータ | `constrained. T callvirt` |

  さらに、受け手が `this` で同じモジュールの sealed 型であるなど、確実な場合は `callvirt` を `call` に変える（**非仮想化**, `:2051-2134`）。
- **アドレスと防御的コピー**（`EmitAddress.cs`、`CodeGenerator_HasHome.cs`）：
  - `AddressKind { Writeable, Constrained, ReadOnly, ReadOnlyStrict }`。
  - `HasHome` は「アドレスを取ってよい場所があるか」を判定する。
    - `in` パラメータ、`ref readonly` ローカル、readonly メンバーの中の `this`、コンストラクタ外の readonly フィールドには、書き込み可能なアドレスが無い。そうした場所で非 readonly メソッドを呼ぶと、**一時変数へのコピー**（`EmitAddressOfTempClone`, `:363`）を作ってそのアドレスを渡す。
    - readonly struct のメソッドは `ReadOnly` のアドレスで呼べるので、コピーが不要になる。
  - 配列要素のアドレスに `readonly.` 接頭辞を付けると、配列の共変性検査を省ける（`:451-469`）。
- **変換と checked**：
  - 数値変換は `EmitNumericConversion`（checked なら `conv.ovf.*`）。
  - checked の `+ - *` は `add.ovf`/`sub.ovf`/`mul.ovf`（符号なしは `.un`, `EmitOperators.cs:696-731`）。
- **配列初期化子**（`EmitArrayInitializer.cs:45`）：定数が多ければ（3 個以上で全て定数、または混在で `max(3, n/3)` 個以上）、`<PrivateImplementationDetails>` に**データ blob のフィールド**を作り、`ldtoken; call RuntimeHelpers.InitializeArray` で一括コピーする。
  - 定数の `ReadOnlySpan<T>` は、`RuntimeHelpers.CreateSpan(ldtoken)` で**割り当てなし**に作る。無ければキャッシュした配列を使う（`:397-626`）。
- **stackalloc**：`localloc`。初期化子は、全バイトが同じなら `initblk`、それ以外は blob から `cpblk` でコピーする。メソッド本体に `hasDynamicStackAllocation` フラグを立てる。

---

## 4. 言語共通の `ILBuilder`（`Core/Portable/CodeGen/`）

### 4.1 構造

- 命令は**基本ブロック**（`BasicBlock`, `BasicBlock.cs:54`）の単方向リストに溜める。
  - 各ブロックは「通常の命令列（blob）」と「末尾の分岐 1 つ」（分岐命令、分岐先ラベル、**逆の分岐命令**）を持つ。
  - 例外ハンドラの先頭と `switch` 用の派生ブロックがある。
- `EmitOpCode`（`ILBuilderEmit.cs:25-30`）は命令を書くと同時に**評価スタックの深さを追跡する**（`EmitState.AdjustStack`：`CurStack += n; MaxStack = max(MaxStack, CurStack)`）。`MaxStack` はメソッドヘッダの max stack になる。分岐命令は `EmitBranch` などの専用 API を通す必要がある。
- `EmitBranch`（`:165`）は、同じラベルへの全ての分岐が同じスタックの深さであることを DEBUG で検査し、ブロックを閉じる。`MarkLabel`（`:108`）は新しいブロックを始める。
- **トークン**：`EmitToken` は `module.GetFakeSymbolTokenForIL`（`CommonPEModuleBuilder.cs:481`）で得た**仮トークン**を書く。メタデータの行番号は書き出しまで決まらないため。
- **短縮形**：
  - 整数定数：`ldc.i4.m1`〜`ldc.i4.8`、`ldc.i4.s`、`ldc.i4`。`long` も値が収まれば `ldc.i4` ＋ `conv.i8`。
  - ローカルと引数：`ldloc.0`〜`ldloc.3`、`ldloc.s`、`ldloc`（`:438-683`）。

### 4.2 `Realize` → `RealizeBlocks`（`ILBuilder.cs:203`, `:862-982`）

```csharp
MarkReachableBlocks(); RewriteSpecialBlocks(); DropUnreachableBlocks();       // 常に（Debug でも）
if (_optimizations == OptimizationLevel.Release && OptimizeLabels()) { 再マーク; DropUnreachableBlocks(); }
RewriteBranchesAcrossExceptionHandlers();                                     // 保護領域をまたぐ br → leave
while (ComputeOffsetsAndAdjustBranches()) { ...; if (!DropUnreachableBlocks()) break; }   // 収束するまで
// 線形化：ブロックを書き出し、IL マーカー・switch 表・分岐オフセットを確定
RealizeSequencePoints(); RealizedExceptionHandlers = _scopeManager.GetExceptionHandlerRegions();
```

- **到達不能コードの削除は Debug でも必ず行う**。到達不能なコードが検証で問題を起こすのを避けるため（コメント `:864-872`）。
- **ラベルの付け替え**（`OptimizeLabels`, `:507`、Release のみ）：「分岐先がさらに分岐しているだけ」の連鎖を短絡する。ただし catch で保護された try の外へラベルを移すことはしない（Thread.Abort との互換性のため, `:629`）。
- **分岐の調整**（`ComputeOffsetsAndAdjustBranches`, `:835`）：コードサイズが縮まなくなるまで繰り返す。
  - `ShortenBranches`（常に）：オフセットが sbyte に収まれば `br.s` などの短縮形にする（3 バイト減る）。
  - `OptimizeBranches`（Release のみ, `BasicBlock.cs:356`）：
    - 次のブロックへの `br` を消す。
    - `ret` だけのブロックへの `br` を `ret` にする。
    - 「条件分岐が無条件分岐を飛び越える」形を、逆の分岐命令で 1 つにまとめる。
- `RealizeSequencePoints`（`:984`）：IL マーカーをオフセットに変換し、到達不能になったものを捨てる。同じオフセットに複数ある場合は最後のものを使う。

### 4.3 スコープと例外領域

- `ScopeType { Variable, TryCatchFinally, Try, Catch, Filter, Finally, Fault, StateMachineVariable }`。
- `OpenLocalScope`（`ILBuilder.cs:1116`）は必ずブロックを閉じる。**基本ブロックはスコープの境界をまたがない**。ハンドラの入口には先頭ブロックと隠しシーケンスポイントを置く。
- `LocalScopeManager` が、ローカル変数のスコープの木、例外ハンドラの領域、ステートマシンの持ち上げ変数のスコープを管理する。

### 4.4 `LocalSlotManager`（`LocalSlotManager.cs:31`）

- `DeclareLocal`（`:126`）：`isSlotReusable` なら、**同じ型と制約の空きスロットを再利用する**。そうでなければ新しいスロットを作る。EnC では `VariableSlotAllocator.GetPreviousLocal` で前の世代の番号を優先する。
- 再利用の可否は `SynthesizedLocalKind.IsSlotReusable(isDebug)`（`SynthesizedLocalKind.cs:273`）で決まる。Debug では長寿命の変数（ユーザー変数など）を再利用しない。
- `AllocateSlot`：一時変数（`EmitterTemp`）用。

### 4.5 整数 switch（`SwitchIntegralJumpTableEmitter.cs`）

1. ラベルを値でソートする。
2. **密なバケットに分ける**（`GenerateSwitchBuckets`, `:185`）。隣のバケットと併合しても密度が 50% 以上なら併合する（`IsSparse`：バケットの幅 ≥ ラベル数 × 2 なら疎）。
3. バケット間は**二分探索**、3 バケット未満なら線形比較（`LinearSearchThreshold = 3`, `:52`）。
4. 各バケットは範囲検査 → `key - start` に正規化 → IL の `switch` 命令。ラベルが 1 つなら `beq`。

### 4.6 `<PrivateImplementationDetails>`（`PrivateImplementationDetails.cs:27`）

モジュールごとに 1 つの合成クラス。

- **データ blob のフィールド**：内容とアラインメントで重複を除き、内容のハッシュで名前を付ける。型は `__StaticArrayInitTypeSize=N` という明示サイズの struct。
- `ComputeStringHash` などのヘルパーメソッド、例外を投げるヘルパー、インライン配列のヘルパー。
- 全メソッドのコンパイル後に `Freeze()` し、その後で自身のメソッドをコンパイルする。EnC の世代 N では名前に `#N` を付ける。

---

## 5. `Optimizer`：スタックスケジューリング（`CSharp/Portable/CodeGen/Optimizer.cs`）

IL はスタックマシンなので、「値をローカルに保存して直後に読む」代わりに「スタックに置いたまま使う」ことができる。

```csharp
_boundBody = Optimizer.Optimize(boundBody, debugFriendly: _ilEmitStyle != ILEmitStyle.Release, stackLocals: out _stackLocals);
```

1. **Pass1**（`StackOptimizerPass1`, `:394`）：木を歩きながら評価スタックの深さと文脈（値、アドレス、代入先、副作用…）を追跡し、各ローカルの**定義と使用の区間**を記録する（`LocalDefUseInfo`）。ラベルや分岐の位置には「スタックの状態の柵」（`DummyLocal`）を置く。
2. **区間の選別**（`FilterValidStackLocals` `:92`、`RemoveIntersectingLocals` `:125`）：互いに交差しない区間を、短いものから貪欲に採用する。区間が 5,000 を超えたら打ち切る。
3. **Pass2**（`StackOptimizerPass2`, `:2097`）：採用したローカル（**スタックローカル**）について、
   - 最後でない読み取りは `BoundDup` にする。
   - 最後の読み取りはそのまま（値は既にスタックにある）。
   - 保存は右辺だけにする。

**debug-friendly モード**（Debug と DebugFriendlyRelease）では、ユーザー変数を消さず、文をまたいで値をスタックに残さない（`:25-56`）。スタックローカルにできるのは短寿命の合成一時変数だけになる（`CanScheduleToStack`, `:2004`）。**Release の IL で `stloc`/`ldloc` が少ないのは主にこのパスのため**。

---

## 6. モジュールビルダーとアダプタ

- `CommonPEModuleBuilder`（`Core/Portable/Emit/CommonPEModuleBuilder.cs:25`）：
  - 仮トークンの対応表（シンボル、ユーザー文字列、ドキュメント）とメソッド本体の対応表を持つ。
  - 型定義は `<Module>` → 追加型・埋め込み型 → ソースの型 → `<PrivateImplementationDetails>` → NoPia の埋め込み型の順に列挙する。
- C# の `PEModuleBuilder`（`CSharp/Portable/Emitter/Model/PEModuleBuilder.cs:28`）：
  - シンボルを Cci（コンパイラ内部のメタデータ抽象）の参照へ変換する（`Translate`）。
  - NoPia の型埋め込み（`EmbeddedTypesManager`）。
- **埋め込み属性**：バインド中に必要と分かった属性を記録しておき（`EnsureEmbeddableAttributeExists`, `:1948`）、参照先に存在しなければ `PEAssemblyBuilder.CreateEmbeddedAttributesIfNeeded`（`PEAssemblyBuilder.cs:421`）がアセンブリ内に合成する。
  - 例：`Microsoft.CodeAnalysis.EmbeddedAttribute`、`NullableAttribute`、`NullableContextAttribute`、`IsReadOnlyAttribute`、`IsUnmanagedAttribute`、`NativeIntegerAttribute`、`ScopedRefAttribute`、`RefSafetyRulesAttribute` など。
- **アダプタ**：DEBUG ビルドではシンボルごとに別の `*SymbolAdapter` オブジェクトを作る。Release ではシンボル自身が Cci のインターフェイスを実装し、オブジェクトの数を減らす（`MethodSymbolAdapter.cs:20-31`）。

---

## 7. メタデータの書き出し（`Core/Portable/PEWriter/`）

- `FullMetadataWriter.Create`（`FullMetadataWriter.cs:39`）は、型システム用の SRM `MetadataBuilder` を作る。Portable PDB（別ファイルまたは埋め込み）を出すなら、デバッグ用にもう 1 つ作る。
- `MetadataWriter.BuildMetadataAndIL`（`MetadataWriter.cs:1786`）：
  1. `CreateIndices`（`:468`）：型定義に行番号を振る。**幅優先**なので、外側の型が入れ子の型より先になる（ECMA の要件）。続いて `ReferenceIndexer`（`MetadataVisitor` の派生クラス）で全ての参照を集め、行番号を振る。
  2. Portable PDB のドキュメント（コマンドラインの順）、インポートスコープ、Source Link、コンパイルオプション。
  3. **`SerializeMethodBodies`**（`:2985`）：メソッドごとにローカル変数のシグネチャを書き、`MethodBodyStreamEncoder.AddMethodBody(codeSize, maxStack, 例外領域数, …, InitLocals, hasDynamicStackAllocation)` で本体を書く。
     - `WriteInstructions`（`:3286`）は IL をコピーしながら、**仮トークンを実際のハンドルに差し替える**（`ResolveEntityHandleFromPseudoToken`, `:3170`）。
     - **小さな本体（64 バイト未満、max stack 8 以下、ローカル無し、例外ハンドラ無し）は、同じ内容なら 1 つを共有する**。ただし**決定的ビルドでは行わない**（issue 7595, `:3053-3075`）。
  4. `PopulateTypeSystemTables`（`:1928`）で全ての ECMA テーブルを埋める（GenericParam は所有者 → 番号の順にソート、CustomAttribute は最後）。

---

## 8. PE・PDB の書き出しと決定的ビルド（`PeWriter.cs:80-319`）

`WritePeToStream` の処理：

1. メタデータと IL を作る。
2. Windows PDB なら仕上げる。
3. `PEHeaderBuilder` を作る。
4. **決定的ビルド**なら、`peIdProvider = content => BlobContentId.FromHash(ComputeSourceHash(content))`（`:192`）。**MVID とタイムスタンプを内容のハッシュから作り**、後から書き込む。
5. Portable PDB の ID も、内容のハッシュから作る。
6. デバッグディレクトリ：CodeView（PDB のパス。決定的ビルドでは詰め物をしない）、PdbChecksum、**Reproducible**（決定的ビルドの印）、EmbeddedPortablePdb。
7. `ManagedPEBuilder` の派生クラスで、IL、FieldRVA のデータ、リソース、強い名前の署名領域、エントリポイントをまとめて `Serialize`。
8. 秘密鍵があれば強い名前で署名する。

**Portable PDB の内容**（`MetadataWriter.PortablePdb.cs`）：

- MethodDebugInformation：シーケンスポイント。
- LocalScope / LocalVariable / LocalConstant。
- async メソッドのステップ情報、ステートマシンの持ち上げ変数のスコープ、dynamic とタプル要素名の情報。
- **EnC 用**：EncLocalSlotMap、EncLambdaAndClosureMap、EncStateMachineStateMap（`SerializeEncMethodDebugInformation`, `:806`）。Lowering ドキュメント §4.5・§5.2 で付けた ID がここに記録される。
- Source Link、コンパイルオプション、参照の一覧、埋め込みソース。
- **シーケンスポイントの blob**（`:626`）：IL オフセットの差分と、行と列の差分で圧縮する。隠しシーケンスポイントは `0,0`。

Windows PDB は `Cci.PdbWriter`（`NativePdbWriter/PdbWriter.cs:25`）が `SymUnmanagedWriter` 経由で書く。

---

## 9. Edit and Continue（`Emitter/EditAndContinue/`、`Core/Portable/Emit/EditAndContinue/`）

1. `EmitHelpers.EmitDifference`（`EmitHelpers.cs:22`）が `CSharpSymbolMatcher` で前の世代と今の世代のシンボルを対応付け、`CSharpDefinitionMap` を作る。
2. 変更されたメソッドだけをコンパイルする。
3. `DefinitionMap.TryCreateVariableSlotAllocator`（`DefinitionMap.cs:268`）が `EncVariableSlotAllocator` を提供する。`LocalDebugId` と構文上のオフセットでローカル変数を対応付けるので、**変更されなかった変数は同じスロットを保つ**。ラムダ、クロージャ、状態番号、awaiter、持ち上げフィールドも同様（Lowering ドキュメント参照）。
4. `DeltaMetadataWriter`（`DeltaMetadataWriter.cs:24`）が差分テーブルと EncLog/EncMap を書き、次の世代の基準（`EmitBaseline`）を作る。

---

## 10. Debug と Release の違い（まとめ）

| 項目 | Debug（`ILEmitStyle.Debug`） | Release |
|---|---|---|
| 命令の無いシーケンスポイントと `NoOp` に対する `nop` | 出す | 出さない |
| return | 戻り値の一時変数 → 共通の `ret`（`}` にステップ可能） | 直接 `ret`（ハンドラ内を除く） |
| スタックローカル | 短寿命の合成一時変数のみ | ユーザー変数も含め、文をまたいでも可 |
| スロットの再利用 | 長寿命の変数は再利用しない | 型ごとに再利用 |
| ローカルの EnC 用 ID | 付ける | 付けない |
| ラベルの付け替え・分岐の書き換え | しない | する |
| 短縮形の分岐・到達不能コードの削除 | する | する |
| クロージャ環境の併合（Lowering） | しない | する |
| ステートマシンの変数の持ち上げ（Lowering） | 全ユーザー変数 | 中断をまたぐ変数のみ（フィールドも再利用） |
| async ステートマシン（Lowering） | class（EnC 有効時） | struct |

- `DebugFriendlyRelease`（`DebugPlusMode`）は、IL 生成が debug-friendly、ILBuilder が Release の折衷。
- **PDB を出すかどうかは IL を変えない**（シーケンスポイントは常に生成し、PDB に書くかどうかだけを切り替える）。
- C# コンパイラは **`tail.` 接頭辞を出さない**（オペコード表にあるだけ）。

---

## 11. 不変条件

- 各文の開始時と try の入口では、評価スタックが空（`EmitStatement.cs:108`, `:946`）。
- 同じラベルへの分岐は全て同じスタックの深さ（`ILBuilderEmit.cs:183`）。
- 基本ブロックはスコープの境界をまたがない。ハンドラの中から `ret` しない（`ILBuilderEmit.cs:306`）。
- エラーのあるメソッドでは IL を生成しない（`MethodCompiler.cs:1655`）。
- `<PrivateImplementationDetails>` は、自身のメソッドをコンパイルする前に凍結する。
- **決定的ビルド**の条件：
  - MVID と PDB の ID を内容のハッシュから作る。
  - Reproducible のエントリを付け、PDB のパスに詰め物をしない。
  - 小さな本体の共有をしない。
  - ドキュメントはコマンドラインの順に並べる。
  - ラベルの付け替えは固定の順序で繰り返す（`ILBuilder.cs:509-512`）。

---

## 付録 A: 主要ファイル早見表

| ファイル | 役割 |
|---|---|
| `CSharp/Portable/CodeGen/CodeGenerator.cs` | 出力スタイル・return・シーケンスポイント |
| `CSharp/Portable/CodeGen/EmitStatement.cs` | 文・条件分岐・try・switch・ローカル |
| `CSharp/Portable/CodeGen/EmitExpression.cs` / `EmitAddress.cs` / `CodeGenerator_HasHome.cs` | 式・呼び出し命令の選択・アドレスと防御的コピー |
| `CSharp/Portable/CodeGen/EmitConversion.cs` / `EmitOperators.cs` / `EmitArrayInitializer.cs` / `EmitStackAllocInitializer.cs` | 変換・演算子・配列初期化子・stackalloc |
| `CSharp/Portable/CodeGen/Optimizer.cs` | スタックスケジューリング |
| `Core/Portable/CodeGen/ILBuilder*.cs` / `BasicBlock.cs` / `EmitState.cs` | 基本ブロック・スタック追跡・Realize |
| `Core/Portable/CodeGen/LocalSlotManager.cs` / `LocalScopeManager.cs` | スロット・スコープ・例外領域 |
| `Core/Portable/CodeGen/SwitchIntegralJumpTableEmitter*.cs` / `SwitchStringJumpTableEmitter.cs` | switch の生成 |
| `Core/Portable/CodeGen/PrivateImplementationDetails.cs` | データ blob・ヘルパー |
| `CSharp/Portable/Compiler/MethodCompiler.cs` | `GenerateMethodBody`・`CompileMethodBodies` |
| `CSharp/Portable/Emitter/Model/PEModuleBuilder.cs` / `PEAssemblyBuilder.cs` / `*Adapter.cs` | モジュールビルダー・埋め込み属性・アダプタ |
| `Core/Portable/PEWriter/MetadataWriter*.cs` / `FullMetadataWriter.cs` / `PeWriter.cs` | メタデータ・Portable PDB・PE |
| `Core/Portable/NativePdbWriter/PdbWriter.cs` | Windows PDB |
| `Core/Portable/Emit/EditAndContinue/*`、`CSharp/Portable/Emitter/EditAndContinue/*` | EnC |

## 付録 B: `static int Max(int a, int b) => a > b ? a : b;` の IL

※ 本節の IL は、上記の規則から導いた**説明用の例**。この環境には .NET SDK が無く、実際にコンパイルしての確認はしていない。

Release:

```text
ldarg.0
ldarg.1
bgt.s   L1        // a > b の比較を条件分岐 1 命令に（EmitCondBranch）
ldarg.1
ret               // 条件演算子の各分岐の後の br→ret を ret に置き換え（OptimizeBranches）
L1: ldarg.0
ret
```

Debug（概念図）: 戻り値の一時変数 `V_0` を使い、`stloc.0; br.s RET; … RET: ldloc.0; ret` の形になる。式本体 `=>` の開始位置にシーケンスポイントが付く。ラベルの付け替えなどの最適化はされない。
