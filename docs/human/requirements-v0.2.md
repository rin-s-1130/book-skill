# Structured Reading Skill 要件定義 v0.2

## 文書状態

- 状態: 合意済みベースライン
- 対象: MVP v0.1の設計・実装
- 本文書の範囲: プロダクト要件、情報モデル、表示要件、品質条件
- 本文書の範囲外: 技術選定、ライブラリ選定、詳細スキーマ、実装計画

本文書では、`MUST` を必須、`SHOULD` を合理的な理由がない限り採用、`MAY` を任意とする。

## 1. 目的

本・論文・新書・専門書などの線形な文字情報を、次の二つへ分離する。

- 省略のない本文テキスト
- 本文から抽出した意味構造

読者が文章から次の情報を頭の中で再構築・保持する負荷を下げる。

- 現在地
- 情報の役割
- 階層
- 概念間関係
- 論証構造
- 長距離の参照関係

目的は情報削減ではなく、情報の分離と外在化である。

> Information Reduction ではなく Information Separation
>
> Compression ではなく Externalization

AIが結論だけを渡すのではなく、文字列に埋め込まれた構造を外在化し、ユーザー自身が全文を読みながら理解できる状態を作る。

## 2. 非目標

本Skillは次を主目的としない。

- 本文を短い要約へ置換する
- 難しい文章を簡単な文章へ全面的に書き換える
- ページ画像を読むための画像ビューアを作る
- AIの解釈を著者の発言として提示する
- 写真に存在しない文章を推測で復元する
- 外部情報によって本文を暗黙に補完する
- 翻訳、批評、事実確認を行う

要約、翻訳、外部照合は、ユーザーが明示的に要求した場合のみ、本文およびSource由来構造とは分離された追加レイヤーとして扱う。

## 3. 入力

MVPの入力は、ユーザーが指定した印刷物のページ写真フォルダとする。

- JPEG、PNG、WebP、TIFFなどの一般的なラスター画像を対象とする。
- サブフォルダを含めた画像列挙を可能とする。
- ファイル名、撮影日時、ファイル列挙順がページ順と一致するとは仮定しない。
- 一枚の写真に見開き二ページが含まれる可能性を考慮する。

PDF、手書き原稿、音声、既存電子書籍ファイルはMVP対象外とする。

## 4. 情報レイヤー

次のレイヤーをデータモデル上で分離しなければならない。

```text
EVIDENCE
ページ画像
    ↓
RAW OCR
機械が最初に認識した文字列
    ↓
READING TEXT
補正履歴付きの逐語転記本文
    ↓
SEMANTIC SPAN
意味単位とSource参照
    ↓
KNOWLEDGE GRAPH
概念・主張・論証・関係
    ↓
READING ATLAS
人間向けの複数ビュー
```

上流レイヤーを下流の解釈で破壊的に上書きしてはならない。

### 4.1 Evidence

ページ画像を最上位の証拠とする。

- 全入力画像へstable IDと暗号学的hashを付与すること。
- 入力画像を変更、リネーム、削除しないこと。
- 出力から各画像へ追跡可能であること。
- 可搬性のため、既定では出力Evidence領域へ画像を複製すること。
- 通常の読書画面へ画像やサムネイルを表示しないこと。
- 画像は明示的に有効化したAudit Modeからのみ参照可能とすること。

ユーザーが画像を読まなければ処理結果を利用できない設計は禁止する。

### 4.2 Raw OCR

- ページ単位で元画像と紐付けること。
- 最初のOCR結果を保持し、破壊的に補正しないこと。
- OCR engine、実行情報、confidenceを保持可能であること。
- 行、段落、見出し、脚注、図表ラベルなどの候補を保持可能であること。

### 4.3 Reading Text

Reading Textを、ユーザーが実際に読むSource-faithfulな本文テキストとする。

- 判読できる本文を省略しないこと。
- 原文の語句、表記、句読点、強調を保持すること。
- 縦書きを横書きへ流し直すこと。
- 撮影や組版に由来する行末改行を除去すること。
- 論理的な段落、見出し、箇条書き、引用、脚注、ページ境界を保持すること。
- 詩、コード、数式、表など意味を持つ改行は保持すること。
- 図表は、判読可能な文字とSourceに根拠を持つ内容記述をテキスト化すること。
- OCR補正を行う場合は、補正前、補正後、理由、confidence、対象範囲を履歴として保持すること。
- 判読不能箇所を文脈から断定して補完しないこと。
- 不確実箇所は候補、判読不能、写真外欠損をテキスト上で区別すること。

低confidence箇所があっても、ユーザーへ画像確認を強制してはならない。

## 5. Page Reconstruction

本格的なOCRと意味解析の前に、次を可能な限り検査する。

- 印刷ページ番号
- ページの向き
- 見開き
- 欠落候補
- 完全重複
- 内容重複
- 極端なぼけ
- クロップによる欠損
- ページ順の不確実性

画像hash、ページ番号、OCR内容、前後ページの文脈を組み合わせる。ファイル名や撮影日時だけでページ順を確定してはならない。

ページ番号の不連続について、入力が抜粋である可能性と実際の欠落を区別する。不明な順序は確定扱いせず、confidenceと判断根拠を保持する。

## 6. Semantic Span

本文を固定文字数ではなく意味単位へ分割する。

Semantic Spanは、一つの主張、定義、根拠、例、説明、推論などとして機能する範囲である。

- 一段落を複数Spanへ分割できること。
- 複数段落または複数ページを一つのSpanにできること。
- 全SpanからReading Textの正確な文字範囲へ戻れること。
- Reading TextからRaw OCR、ページ画像まで追跡できること。

## 7. 抽出する意味構造

### 7.1 Position

最低限、次を表現できること。

- 文書
- 章
- 節
- 小節
- 現在の話題
- 現在処理している問い
- 文書全体に対する現在位置

### 7.2 Role

MVPでは次を扱う。

- QUESTION
- CLAIM
- DEFINITION
- PREMISE
- EVIDENCE
- EXAMPLE
- CONCLUSION
- EXPLANATION
- TRANSITION
- OBJECTION
- RESPONSE
- QUALIFICATION
- COUNTEREXAMPLE
- ASSUMPTION

Role分類だけで完了とせず、他ノードとの関係を抽出すること。

### 7.3 Hierarchy

次を別データとして保持する。

- Document Hierarchy: 章、節、段落などの文書上の階層
- Semantic Hierarchy: 概念、論点、主張などの意味上の階層

### 7.4 Relation

最低限、次を第一級データとして扱う。

- supports
- exemplifies
- defines
- contrasts_with
- depends_on
- derives_from
- elaborates
- qualifies
- refutes
- responds_to
- generalizes
- specializes
- refers_to

### 7.5 Concept

概念ごとに次を保持可能とする。

- 原語と訳語
- 初出
- 局所的な意味
- 文書全体での意味
- 定義の変化
- 対立概念
- 上位・下位概念
- 関連する主張
- Source参照

入力が文書の一部に限られる場合、文書全体での意味を断定してはならない。

### 7.6 Argument

Roleとは別に論証単位を保持する。

```text
Question
  ↓
Premise
  ↓
Evidence / Example
  ↓
Inference
  ↓
Conclusion
```

必要に応じてObjectionとResponseを接続する。

## 8. Knowledge Graph

Knowledge Graphを意味構造の内部正本とする。Logic Tree、Concept Map、TimelineなどはGraphから生成する派生ビューである。

すべてのAI生成ノードと関係に、次を保持する。

- stable ID
- node typeまたはrelation type
- 一つ以上のSource Span参照
- epistemic status
- confidence
- 著者、引用者、AIの話者区別

Epistemic statusは次の4段階とする。

- explicit
- strongly_inferred
- inferred
- uncertain

Source参照のないAI解釈を著者の主張として表示してはならない。

## 9. Reading Atlas

最終成果物を、複数の見方を持つReading Atlasと定義する。

HTMLは有力な実装候補だが、要件として特定技術へ固定しない。実装方式にかかわらず、後述する操作能力を満たさなければならない。

### 9.1 必須ビュー

1. **Atlas View**

   文書全体、章、主要概念、主要論証を俯瞰する。

2. **Position View**

   章、節、現在の論点を路線図、ツリー、パンくずなどで示す。

3. **Logic View**

   問い、前提、根拠、推論、結論、反論、応答を図解する。

4. **Concept View**

   概念間関係、定義、対立、包含、意味の変化を示す。

5. **Text Reader**

   省略のないReading Textと対応構造を読む。ページ画像は表示しない。

### 9.2 適応的ビュー

素材に該当する意味関係が存在する場合、次の表現から適切なものを選択する。

| 内容 | 適した表現 |
|---|---|
| 時系列 | Timeline |
| 手順 | Flowchart |
| 主体別の進行 | Swimlane |
| 原因と結果 | Causal Diagram |
| 分類 | Taxonomy、Treemap |
| 比較 | Matrix |
| 長距離参照 | Arc Diagram、Backlink |
| 概念の変化 | Concept Evolution Timeline |
| 章ごとの概念分布 | Heatmap |
| 反論構造 | Claim、Objection、Responseのレーン表示 |

すべての文書へ同じ図を強制してはならない。Sourceに根拠のない空間配置や接続を生成しないこと。

## 10. インタラクション

### 10.1 Semantic Zoom

次の解像度を相互に移動できること。

```text
文書全体
→ 章の目的
→ 中心的な問いと主張
→ 詳細な論証
→ 構造と対応本文
→ 省略のない全文
```

上位レベルは下位レベルの代替ではない。

### 10.2 Focus Mode

ノードを選択したとき、最低限次を表示する。

- 選択ノード
- 直接接続されたノード
- 上位階層
- 対応本文
- 長距離参照
- confidenceとepistemic status

巨大なKnowledge Graphを最初から全面表示してはならない。

### 10.3 双方向移動

- 構造ノードから対応本文へ移動できること。
- 本文範囲から対応構造を表示できること。
- 概念から文書内の出現、定義、利用、変更箇所へ移動できること。

### 10.4 Semantic Lens

次の強調表示を切り替えられること。

- 主張
- 定義
- 根拠
- 具体例
- 反論と応答
- 特定概念
- 不確実な解釈

### 10.5 Argument Playback

問いから結論までの論証を一段ずつたどれること。

### 10.6 作業記憶シェルフ

読書中に次を固定表示できること。

- 現在の問い
- 中心主張
- 重要概念
- 前提
- 未解決の反論
- ユーザーがピン留めした項目

## 11. 視覚文法

全ビューで意味表現を統一する。

- 入れ子: 階層
- 上下または左右の配置: 進行または包含
- 矢印: 関係の方向
- 実線: 明示的関係
- 破線: 推定関係
- 薄い線: 低confidence
- 曲線: 長距離参照
- 形とアイコン: Role
- 色: 概念群または関係種別

色だけで意味を伝えてはならない。形、線種、テキストラベル、凡例を併用する。段階的開示とFocus Modeを優先し、視覚的な密集を避ける。

## 12. Multi-agent実行

本SkillはMulti-agent実行を標準とする。

```text
Root Orchestrator
├─ Page Reconstruction Agent
├─ OCR Workers
├─ Structural Analysis Workers
├─ Global Structure Agent
└─ Integrity Auditor
```

### 12.1 責務

- Root Orchestrator: 工程管理、競合解決、Knowledge Graph統合
- Page Reconstruction Agent: 順序、欠落、重複、向きの判定
- OCR Worker: 担当ページのRaw OCRとReading Text候補
- Structural Analysis Worker: Semantic Span、Role、局所Relation
- Global Structure Agent: 概念変化、長距離参照、文書全体の論証
- Integrity Auditor: 未根拠解釈、参照切れ、OCR補正履歴の検査

### 12.2 競合防止

- Workerは担当範囲別のshardへ出力すること。
- 複数エージェントが同じ正本を同時編集しないこと。
- Knowledge Graph正本はRoot Orchestratorだけが統合すること。
- Integrity Auditorは読み取り専用とすること。
- 章や担当範囲の境界では前後のページを参照し、連続性を検査すること。
- 低confidence箇所は、先行判断を与えられていない別エージェントが独立判定すること。

Multi-agentが利用できない場合、単一エージェントへ黙って切り替えず、処理開始前に停止して報告する。異なるモデルを利用できない場合も明示し、ユーザーの許可なく同一モデル構成へ変更しない。

## 13. 機械処理とAIの責務分離

### 13.1 Machine / Script

- 画像一覧取得
- hash計算
- stable ID生成
- Evidence登録
- 完全重複検出
- ページ番号管理
- 参照整合性検査
- schema validation
- Graph integrity検査
- report生成
- 派生ビュー構築
- Source復元確認

### 13.2 AI

- OCR文脈補正
- Semantic Segmentation
- Role分類
- Concept抽出
- Claim抽出
- Relation抽出
- Argument解析
- Semantic Hierarchy生成
- 長距離参照解析
- 適切な視覚表現の選択
- 不確実性評価

決定的に処理できる作業をAIの自由記述へ委ねない。

## 14. 概念出力構造

実装は最低限、次の責務境界を持つ。具体的なファイル名とschemaは設計段階で決定する。

```text
book/
├─ evidence/
│  ├─ images/
│  └─ page-index
├─ source/
│  ├─ raw-ocr/
│  ├─ reading-text/
│  └─ correction-history/
├─ structure/
│  ├─ spans
│  ├─ graph
│  ├─ concepts
│  └─ arguments
├─ views/
│  ├─ interactive-atlas
│  └─ static-exports/
└─ report
```

内部正本はEvidence、Raw OCR、Reading Text、Knowledge Graphである。表示ファイルは正本から再生成可能な派生物とする。

インタラクティブビューはローカルかつオフラインで利用できること。静的なSVGまたはPDF出力をSHOULDとする。

## 15. Integrity Report

最低限、次を機械集計して報告する。

- 入力画像数
- 論理ページ数
- 欠落候補
- 重複候補
- ページ順不確実数
- OCR不確実箇所数
- Semantic Span数
- Role別ノード数
- Concept数
- Argument数
- Relation数
- Source参照のないAIノード数
- 壊れた参照数
- 未処理ページ数
- 処理に参加したエージェントとモデル
- エージェント間の競合数と再判定数

部分処理を完全処理として報告してはならない。

## 16. Acceptance Criteria

### AC-01 Evidence Preservation

全入力画像がEvidenceとして保持され、stable IDとhashで識別できる。

### AC-02 Text-first UX

通常の読書画面にページ画像またはサムネイルが表示されず、画像を読まなくても全文と構造を利用できる。

### AC-03 Page Integrity

ページの欠落候補、重複候補、順序および不確実性を検証できる。

### AC-04 Layer Separation

Evidence、Raw OCR、Reading Text、AI Interpretationがデータモデル上で混在しない。

### AC-05 Full-text Coverage

判読可能な全文がReading Textに含まれ、欠損と判読不能が明示される。

### AC-06 Correction Trace

OCR補正の前後、理由、confidence、対象箇所を追跡できる。

### AC-07 Traceability

全AI生成ノードからSemantic Span、Reading Text、Raw OCR、ページ画像まで遡れる。

### AC-08 Structural Coverage

Claim、Definition、Evidence、Example、Conclusion、Concept、Argument、Relation、Hierarchyを識別できる。

### AC-09 Relation Visibility

分類だけでなく、何が何をどのように支え、定義し、対比し、参照するかを表示できる。

### AC-10 Graph Canonicality

Knowledge Graphが意味構造の内部正本であり、各ビューを再生成できる。

### AC-11 Multi-resolution Visualization

Atlas、Position、Logic、Concept、Textの必須ビューとSemantic Zoomを提供する。

### AC-12 Bidirectional Navigation

構造から本文、本文から構造へ移動できる。

### AC-13 Adaptive Visualization

Sourceに存在する関係に応じて適切な視覚表現を選び、不適切な図を強制しない。

### AC-14 Uncertainty

AI推論にepistemic statusとconfidenceを保持し、視覚表現にも反映する。

### AC-15 Multi-agent Integrity

Multi-agentで実行され、正本への並列書き込みが発生しない。

### AC-16 Mechanical Validation

schema、ID、Source参照、Graph参照、Evidence hashを機械的に検査できる。

### AC-17 Scope Honesty

入力が文書の一部の場合、本全体の中心命題や概念体系を推測で完成させない。

### AC-18 Offline Availability

主要な読書ビューをローカルかつオフラインで利用できる。

## 17. MVP v0.1

### MUST

- 画像フォルダ入力
- Page Reconstruction
- Raw OCRと補正履歴
- 省略のないReading Text
- Semantic Span
- Role、Hierarchy、Relation、Concept、Argument
- Knowledge Graph
- Multi-agent処理
- Atlas、Position、Logic、Concept、Textの5ビュー
- Semantic Zoom
- Focus Mode
- 構造と本文の双方向リンク
- Evidenceの非表示保持
- Integrity Report

### MVP対象外

- 音声読み上げ
- 翻訳
- 外部版との本文照合
- 手書き認識
- ユーザー間の共同注釈
- クラウド同期
- モバイル専用アプリ
- 高度な自動レイアウト最適化

## 18. v0.1からの確定変更

- 最終成果物を単一のLogic TreeではなくReading Atlasとした。
- HTMLを必須技術とせず、必要なインタラクション能力を要件とした。
- ページ画像を読書階層から除外し、非表示のEvidenceとした。
- Level 5をページ画像ではなく、省略のない全文テキストとした。
- 内容に応じてTimeline、Matrix、Flowchartなどを選ぶ適応的視覚化を追加した。
- Multi-agentと複数モデル構成を標準とし、暗黙の単一エージェントfallbackを禁止した。
