# Roslyn C# コンパイラ 内部解説シリーズ（索引）

Roslyn の C# コンパイラが、ソーステキストから IL・PE・PDB を作るまでの内部実装を、ソースコードに基づいて解説するドキュメント群です。全て日本語で、ファイルと行番号の参照付きです。

- 基準コミット：`88579e780fe7`（2026-11-13）。行番号はこの時点のものです。
- 対象は C#（言語共通の `Core` を含む）。VB は対比として触れる程度です。
- 各章はソースの読解に基づいています。実行例の IL など、実際のビルドで確かめていない記述は本文でその旨を明記しています。

---

## 1. 全体の流れと各章の位置

```text
                       ┌────────────────────────── ⑩ Compilation・SemanticModel・ドライバ（全体をまとめる層）──────────────────────────┐
                       │  csc / コンパイラサーバー → ソースジェネレータ → アナライザー接続 → 診断の段階 Parse / Declare / Compile → Emit   │
                       └──────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
ソーステキスト
   │ ① 構文解析（レクサー・パーサー・Green/Red ツリー・Syntax.xml の自動生成）
   ▼
SyntaxTree ──② 宣言とシンボル（宣言テーブル・ソース/メタデータシンボル・遅延完了）──▶ シンボル表
   │                                                                                       │
   │ ③ バインダ（スコープ・名前探索・変換・オーバーロード解決・型推論・パターン）◀─────────┘
   ▼
BoundTree（④ BoundTree と BoundNodes.xml の自動生成）
   ├─▶ ⑤ フロー解析（到達可能性・確定代入・Nullable・領域解析）
   ├─▶ ⑨ 補足：ref 安全性の解析（RefSafetyAnalysis）
   ├─▶ ⑧ IOperation と CFG（アナライザー向けの公開モデル）
   ▼
⑥ Lowering（LocalRewriter・await の Spill・クロージャ変換・イテレータ/async のステートマシン）
   ▼
⑦ IL 生成と Emit（CodeGenerator・ILBuilder・スタックスケジューリング・メタデータ・PE・PDB・EnC）
   ▼
アセンブリ（.dll / .exe）＋ PDB

⑨ 補足トピック（診断の基盤・ref 安全性・定数畳み込み）は全段にまたがります。
```

---

## 2. 章の一覧

| # | 章 | ファイル | 主な内容 |
|---|---|---|---|
| ① | 構文解析パイプライン | [`csharp-syntax-pipeline.ja.md`](csharp-syntax-pipeline.ja.md) | レクサー（高速パス、キャッシュ、trivia、前処理ディレクティブ）、パーサー（再帰下降、優先順位上昇法、誤り回復、増分解析）、GreenNode の実装、Red ツリー、`Syntax.xml`/`.xsd` とソースジェネレータによる自動生成 |
| ② | 宣言とシンボル | [`csharp-declarations-symbols.ja.md`](csharp-declarations-symbols.ja.md) | `DeclarationTreeBuilder`、編集に強い `DeclarationTable`、partial 型の併合、`SourceNamedTypeSymbol` のメンバー構築、`CompletionPart` による段階的な完成、基底型の循環の検出、属性の 2 段階デコード、PE シンボル、参照マネージャー、リターゲット、構築型、公開 API のラッパ |
| ③ | バインダ | [`csharp-binder.ja.md`](csharp-binder.ja.md) | バインダチェーン、`BinderFactory`/`LocalBinderFactory`、名前探索と `LookupResult`、アクセス可能性、式と文のバインド、未変換の式と `CreateConversion`、変換の分類、オーバーロード解決、型推論、演算子、ラムダの再バインド、`foreach`、パターンと決定 DAG |
| ④ | BoundTree と BoundNodes.xml | [`csharp-boundtree-boundnodes.ja.md`](csharp-boundtree-boundnodes.ja.md) | BoundTree の位置づけ、`BoundNodes.xml`/`.xsd` の属性、`BoundTreeGenerator` の生成規則、Visitor/Walker/Rewriter/NullabilityRewriter、`BoundNode` のフラグ、`DoesNotSurvive` と `PipelinePhaseValidator` |
| ⑤ | フロー解析 | [`csharp-compiler-dataflow-analysis.ja.md`](csharp-compiler-dataflow-analysis.ja.md) | `AbstractFlowPass` の格子と条件状態、保留中の分岐、不動点反復、try/finally、ローカル関数、`ControlFlowPass`、確定代入、IDE 向け領域解析（DataFlowsIn/Out）、`NullableWalker` |
| ⑥ | Lowering | [`csharp-lowering.ja.md`](csharp-lowering.ja.md) | 段階の順序、`LocalRewriter`（制御構文の分解、糖衣構文の展開、定数、計装）、`SpillSequenceSpiller`、クロージャ変換、ステートマシンの共通基盤、イテレータ、async、async イテレータ、runtime-async |
| ⑦ | IL 生成と Emit | [`csharp-codegen-emit.ja.md`](csharp-codegen-emit.ja.md) | `CodeGenerator`、条件分岐と呼び出し命令の選択、防御的コピー、switch の生成、`ILBuilder` の基本ブロックと分岐の最適化、`Optimizer`（スタックスケジューリング）、メタデータ・PE・Portable PDB、決定的ビルド、EnC、Debug と Release の違い |
| ⑧ | IOperation と CFG | [`csharp-ioperation-cfg.ja.md`](csharp-ioperation-cfg.ja.md) | `OperationInterfaces.xml` と `IOperationGenerator`、`Operation` 基底、`CSharpOperationFactory`、`GetOperation` のキャッシュ、`ControlFlowGraph`/`BasicBlock`/領域、`ControlFlowGraphBuilder` による構文の分解、アナライザーとの統合 |
| ⑨ | 補足トピック | [`csharp-supplementary-diagnostics-refsafety-constants.ja.md`](csharp-supplementary-diagnostics-refsafety-constants.ja.md) | 診断の基盤（`ErrorCode` の接頭辞と重大度、`ErrorFacts` の自動生成、警告ウェーブ、`MessageID`、`#pragma`、`CSharpDiagnosticFilter`）、ref 安全性の解析（`SafeContext`）、定数と定数畳み込み（`ConstantValue`、循環の検出、後段での消費） |
| ⑩ | Compilation・SemanticModel・ドライバ | [`csharp-compilation-semanticmodel-driver.ja.md`](csharp-compilation-semanticmodel-driver.ja.md) | 不変の `CSharpCompilation` と状態の再利用、診断の 3 段階、コンパイルのイベント、`SyntaxTreeSemanticModel`/`MemberSemanticModel` の増分バインド、csc の `CompileAndEmit`、コンパイラサーバー、インクリメンタルジェネレータのノードグラフ、アナライザードライバ |

---

## 3. 読む順番の目安

| 目的 | おすすめの順番 |
|---|---|
| コンパイラ全体を通しで理解したい | ① → ② → ③ → ④ → ⑤ → ⑥ → ⑦（必要に応じて ⑨、最後に ⑩） |
| アナライザーやソースジェネレータを書く | ① → ⑩ → ⑧ →（必要に応じて ② ③） |
| IDE の機能（補完、リファクタリング）の裏側を知りたい | ① → ② → ⑩（SemanticModel）→ ③ → ⑤（領域解析） |
| 新しい言語機能をコンパイラに追加したい | ①（`Syntax.xml`、パーサー）→ ③ → ④（`BoundNodes.xml`）→ ⑤ → ⑥ → ⑦ → ⑨（診断と `MessageID`） |
| 生成される IL や性能を理解したい | ⑥ → ⑦ →（③ の変換と演算子） |
| Nullable 参照型の仕組みを知りたい | ⑤ §8 → ④（`NullabilityRewriter`）→ ⑩（SemanticModel）→ ⑨（警告のフィルタ） |

---

## 4. 横断的なテーマと参照先

| テーマ | 該当箇所 |
|---|---|
| **XML からの自動生成**（3 系統＋診断） | ①§1・§5〜§7（`Syntax.xml`、ビルド時のソースジェネレータ）、④§2〜§4（`BoundNodes.xml`、スクリプトで生成しコミット）、⑧§1（`OperationInterfaces.xml`）、⑨A.2（`ErrorCode.cs` → `ErrorFacts.Generated.cs`）。再生成は全て `eng/generate-compiler-code.cmd` |
| **不変性と構造の共有** | ①§4・§8（Green/Red）、②§1.3（宣言テーブル）、④§4.6（`Update`/`VisitList`）、⑩§1.1・§4.1 |
| **遅延計算とスレッド安全性** | ②§3.3・§7（`CompletionPart`、CAS）、①§8（Red ノードの遅延生成）、③§2.5〜2.6（バインダのキャッシュ）、⑩§2.3（増分バインド） |
| **増分処理** | ①§3.12（パーサーの Blender）、②§1.3、⑩§1.1（参照マネージャーの再利用）・§4.5（ジェネレータの NodeStateTable） |
| **評価順序と一時変数の退避（spill）** | ⑥§2.10・§3（複合代入、名前付き引数、`SpillSequenceSpiller`）、⑧§3.3（CFG のフローキャプチャ） |
| **深い木への対策（スタックガード、反復処理）** | ①§3.10、③§9.1、④§4.7、⑤§2.5、⑦§3.4、⑧§2.3 |
| **エラー回復と連鎖エラーの抑止** | ①§3.2〜3.3・§3.9（欠落トークン、スキップ trivia）、③§4.4（`BadExpression`、`HasAnyErrors`）、④§5.1（`HasErrors`） |
| **Edit and Continue（EnC）** | ⑥§4.5・§5.2（ラムダ、クロージャ、状態番号の ID）、⑦§9（スロットの対応付け、差分メタデータ） |
| **Debug と Release の違い** | ⑦§10（一覧表）、⑥§4.2・§5.1・§7.1（環境の併合、変数の持ち上げ、struct ステートマシン） |
| **同じ解析の再利用** | ⑤§7（確定代入を細工した DataFlowsIn/Out）、⑥§5.1（中断をまたぐ変数の判定に確定代入を流用）、③§12 → ⑤§8.6・⑥§2.8（決定 DAG の共有） |

---

## 5. 主要な用語

| 用語 | 意味 | 主な章 |
|---|---|---|
| Green / Red ツリー | 不変・位置なし・共有可能な内部の木と、親と位置を持つ公開の木 | ① |
| trivia | 空白、コメント、ディレクティブなど、トークンに付随するテキスト | ① |
| 宣言（Declaration） | 構文木から抜き出した型と名前空間の形の要約 | ② |
| CompletionPart | シンボルの各部分（基底型、メンバー、属性…）が計算済みかを表すビット | ② |
| バインダチェーン | スコープを表すバインダの連結リスト | ③ |
| 未変換（unconverted）ノード | 型が文脈で決まる式の、確定前の表現 | ③④ |
| BoundTree | バインドの結果の内部表現 | ④ |
| 条件状態 | ブール式の評価後の「真の場合／偽の場合」の 2 つの状態 | ⑤ |
| Lowering | 高水準の BoundTree を IL に近い形へ書き換える段階 | ⑥ |
| Spill | 評価途中の値を一時変数へ退避すること | ⑥⑧ |
| display class | ラムダが捕捉する変数を持つ合成クラス（`<>c__DisplayClass`） | ⑥ |
| ステートマシン | イテレータ/async のメソッド本体を移した合成型（`<M>d__N`） | ⑥ |
| スタックローカル | 評価スタック上の値で置き換えられたローカル変数 | ⑦ |
| IOperation / CFG | アナライザー向けの公開の意味モデルと制御フローグラフ | ⑧ |
| 警告ウェーブ | 新しい警告を警告レベルの引き上げ時だけ出す仕組み | ⑨ |
| SafeContext | ref 安全性の解析で寿命を表す数 | ⑨ |
| SemanticModel | 構文ノードに対する意味（シンボル、型、操作）の問い合わせ API | ⑩ |
| インクリメンタルジェネレータ | 前回と比較して変化した部分だけを再計算するソースジェネレータ | ⑩ |
