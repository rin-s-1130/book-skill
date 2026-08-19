# Perspective Lens Chat 将来構想

## 文書状態

- 状態: 将来構想
- 対象候補: MVP v0.1以降
- 現行要件との関係: [Structured Reading Skill 要件定義 v0.2](requirements-v0.2.md)から分離された非MVP機能
- 注意: 本文書は実装を確約する要件正本ではない

## 1. 概要

Reading Atlas内にチャット欄を設け、ユーザーが自然言語で指定した視点から、既存の全文テキストと意味構造を再整理できるようにする。

例:

- 「この章を初心者がつまずく順序で整理して」
- 「著者が暗黙に置いている前提を中心に見せて」
- 「実務で意思決定するときに使える観点から整理して」
- 「概念Aと概念Bの違いだけを軸に全体を組み直して」
- 「反論可能な箇所を中心にロジックツリーを作り直して」

チャットは本文を要約へ置換するものではない。ユーザーの視点を追加の整理軸として与え、同じSourceを異なる入口と配置で閲覧できるようにする。

## 2. 基本モデル

Perspective Lens Chatは、Evidence、Raw OCR、Reading Text、Knowledge Graphを変更しない。

```text
CANONICAL LAYERS
Evidence
Raw OCR
Reading Text
Knowledge Graph
        │
        ├── Default Reading Atlas
        │
        └── User Lens Request
                ↓
            Perspective Lens
                ↓
            Lens Graph / Lens View
                ↓
            Reorganized Reading Atlas
```

「出力したドキュメントを再整理する」とは、既存成果物を破壊的に書き換えることではなく、同じ正本から新しい派生ビューを生成することを意味する。

## 3. 主要原則

### 3.1 正本を変更しない

- ページ画像、Raw OCR、Reading Textを変更しない。
- 基本Knowledge Graphを上書きしない。
- Lens固有のノードや関係を基本Graphへ暗黙に混入させない。

### 3.2 複数の視点を並存させる

- 一つの文書に複数の名前付きLensを保存できる。
- Lensの追加によって既存Lensを上書きしない。
- Default Viewへいつでも戻れる。
- Lens同士を比較、複製、改訂、削除できる。

### 3.3 視点と著者の主張を区別する

表示上、最低限次を区別する。

- 著者が明示した構造
- Sourceから強く推定される構造
- ユーザーが指定した視点
- Lens生成時にAIが行った解釈
- Sourceから十分に支持されないLens上の仮説

ユーザーの視点に合うように著者の主張を改変してはならない。

### 3.4 Traceabilityを維持する

- Lens内の全ノードから基本Knowledge Graphへ戻れる。
- 基本GraphからSemantic Span、Reading Text、Raw OCR、ページ画像へ戻れる。
- Lens独自の解釈にもSource参照、epistemic status、confidenceを付与する。

## 4. 想定ユーザー体験

### 4.1 Lensの作成

ユーザーが対象範囲と視点をチャットで指定する。

```text
対象: 第2章
依頼: 初学者が理解するために必要な前提の順番で整理して
```

システムは依頼を、次のLens設定へ変換する。

- Lens名
- 対象範囲
- 読書目的
- 優先する概念、Role、Relation
- 抽象度
- 希望する視覚表現
- 除外条件

曖昧さが結果を大きく変える場合だけ、チャットで追加確認する。

### 4.2 再構成

Lensに応じて次を変更できる。

- 最初に提示する問い
- ノードの表示順
- 強調する概念と関係
- Logic Treeのroot
- 抽象度
- 推奨する読書経路
- 使用する視覚表現
- 作業記憶シェルフへ固定する項目

Reading Textの内容とSource上の順序は保持する。Lensが提示する読書経路と原文順を混同しない。

### 4.3 対話的な改訂

作成したLensに対して、さらに次のような指示を行える。

- 「もっと具体例を中心にして」
- 「概念の定義を一段深く表示して」
- 「この二つの主張を比較して」
- 「この見方で拾えていない箇所を教えて」
- 「一つ前の状態に戻して」

各改訂を履歴として保持し、以前のLens状態へ戻せるようにする。

### 4.4 動的な再構成

チャット指示を確定すると、現在表示しているノードを起点として、ノードと関係線を動的に再配置できるようにする。

例として、Claim Aを見ている状態で「反論から整理して」と指示した場合、次の変化を連続的に表示する。

- Claim Aを画面上のアンカーとして固定する。
- ObjectionノードをClaim Aの周辺へ移動する。
- Responseノードを対応するObjectionの隣へ配置する。
- supports、refutes、responds_toなどの関係線をノードと一緒に移動する。
- Evidenceノードは削除せず、必要に応じて折り畳むか視覚的な優先度を下げる。
- 対応するReading TextとSource参照を維持する。

文字入力の一文字ごとにGraph全体を再配置してはならない。指示の送信時、または意味的に安定した入力として認識できた時点で再構成を開始し、入力途中の候補はプレビューとして区別する。

### 4.5 Mental Mapの保持

動的な再配置によって、読者が覚えた位置関係を破壊してはならない。

- 選択中またはピン留めされたノードを原則として動かさない。
- 最初は選択ノードから近い局所Graphだけを再配置する。
- 維持可能な相対位置は保持する。
- 表示対象外になるノードを即座に消さず、縮小またはフェードさせる。
- 移動前と移動後の差分を確認できるようにする。
- 再配置の一時停止、確定、取り消し、やり直しを提供する。
- Default Viewまたは任意のLens versionへ戻れるようにする。

## 5. Lensで利用できる整理軸

- 初学者向けの前提順序
- 専門家向けの論証詳細
- 実務上の意思決定
- 概念間の対立
- 因果関係
- 歴史的変化
- 方法論
- 反論可能性
- 著者の暗黙の前提
- 特定概念の変遷
- ユーザー自身の問いまたは仮説

固定されたプリセットだけでなく、自然言語から新しい整理軸を作成できることを目標とする。

## 6. Lensの出力

Lensごとに、少なくとも次を保持する構想とする。

- lens ID
- 名前
- 元のユーザー指示
- 正規化されたLens設定
- 対象範囲
- 基本Knowledge Graphのversion
- Lens固有のノードと関係
- Source参照
- epistemic statusとconfidence
- 使用したモデルと生成日時
- 改訂履歴
- Lens用のReading Atlas View

Lens Graphは基本Knowledge Graphの代替正本ではない。

## 7. 表示機能候補

- Default ViewとLens Viewの切り替え
- 複数Lensの並列比較
- Lensによって追加された関係だけを強調
- Sourceに強く支持されない箇所の警告
- Lensが重視した箇所と見送った箇所の表示
- 同じ本文に対する異なるLogic Treeの比較
- チャット履歴から任意のLens versionへ復元
- ノードと関係線の連続的な移動
- 即時プレビューと確定結果の視覚的な区別
- 再構成前後の差分表示
- 動的レイアウトの一時停止と再開

## 8. リアルタイム実行モデル

動的再構成を、即時処理と意味解析の二段階に分ける。

```text
Chat Instruction
        ↓
Perspective Lens Spec
        ├── Fast Path
        │   既存Graphの重み、root、filter、折り畳み、配置を変更
        │   → 即時プレビュー
        │
        └── Semantic Path
            AIが新しいLensノードと関係を解析
            → Integrity Audit
            → 検証済み差分をアニメーションで反映
```

### 8.1 Fast Path

既存Knowledge Graphだけで決定できる処理を行う。

- 表示rootの変更
- ノード重要度の再計算
- RoleまたはRelationによるfilter
- ノードの折り畳みと展開
- 局所Graphの再配置
- 既存のTimeline、Matrix、Logic Treeなどへの切り替え

Fast PathはAIの意味解析完了を待たず、操作に対する視覚的な応答を返す。

### 8.2 Semantic Path

新しい解釈、整理軸、関係または読書経路が必要な場合に実行する。

- Lens固有のノードと関係を生成する。
- 基本Graphとの差分として保持する。
- Source参照、epistemic status、confidenceを検査する。
- Integrity Auditが完了した差分だけを確定表示へ昇格する。
- 解析途中の結果を著者の確定した構造として表示しない。

Semantic Pathが失敗または中断しても、Fast Pathの表示と基本Reading Atlasを破損してはならない。

### 8.3 大規模Graph

文書全体の全ノードを毎回再配置しない。

- 現在の表示範囲に必要な部分Graphだけを描画する。
- Semantic Zoomに応じて詳細度を変える。
- 画面外のノードは配置情報を保持しつつ描画を省略できる。
- 全体Graphの更新はバックグラウンドで行い、局所表示を優先する。

### 8.4 実行環境の境界

既存Graphのfilterと再配置は、静的なオフライン成果物でも実行できる。自由入力された視点を理解して新しい意味構造を生成するには、実行時に利用可能なAIモデルが必要となる。

将来の構成候補は次とする。

```text
Local Reading Atlas
        ↕
Local Companion / Agent Service
        ↕
Configured AI Model
```

AIモデルへ接続できない場合も、保存済みLens、既存Graphの再配置、Default Viewへの復帰は利用可能とする。認証情報をReading Atlas成果物へ埋め込んではならない。

## 9. Multi-agent構想

Lens生成も将来的には複数エージェントで行える。

- Lens Interpreter: ユーザーの視点を構造化する
- Recomposition Agent: Lens Graphと読書経路を生成する
- Counter-perspective Agent: 視点によって見落とされた箇所を検出する
- Integrity Auditor: Source参照と著者／ユーザー／AIの区別を検査する
- Root Orchestrator: 結果を統合し、Lens Viewを確定する

視点に迎合するだけでなく、その視点では説明できないSource箇所も保持する。

## 10. 将来のAcceptance Criteria候補

- 自然言語の指示から名前付きPerspective Lensを作成できる。
- 基本Knowledge GraphとReading Textを変更せずに再整理できる。
- Default ViewとLens Viewを切り替えられる。
- 複数Lensを同時に保持し比較できる。
- Lensの全解釈からSourceまで追跡できる。
- 著者の主張、ユーザーの視点、AIの解釈を区別できる。
- Lensの生成と改訂履歴を保持し、以前の状態へ戻せる。
- Lensに合わないSourceを削除または不可視化しない。
- 根拠が弱いLens解釈を明示できる。
- Lens生成に失敗しても基本Reading Atlasを破損しない。
- 選択中またはピン留めされたノードをアンカーとして再構成できる。
- ノードと支持、反論、応答などの関係線が一体として連続的に移動する。
- 既存Graphだけで可能な変更を即時プレビューできる。
- AIによる意味再解析を非同期に行い、検証済み差分だけを確定表示できる。
- 再構成の一時停止、確定、取り消し、やり直しができる。
- 再構成前後の差分を確認できる。
- 大規模Graphでも局所表示を優先し、文書全体の再配置完了を待たずに操作できる。
- AIモデルに接続できない状態でも、保存済みLensとDefault Viewを利用できる。

## 11. 前提条件

本機能の設計開始前に、MVP v0.1で次が安定している必要がある。

- Reading Textの完全性
- Semantic Spanの安定したID
- Knowledge Graphのversion管理
- Source Traceability
- 派生ビューの再生成
- epistemic statusとconfidence
- 安定したGraph layout IDと表示状態のversion管理
- Fast PathとSemantic Pathを分離できるLens差分モデル

## 12. 現時点の対象外

- 基本Sourceの自動書き換え
- ユーザーの思想に合わせた著者発言の改変
- Sourceにない結論の事実扱い
- Lens結果による基本Knowledge Graphの自動更新
- 複数ユーザー間のLens共有
- 長期的な個人プロファイルの自動学習

個人プロファイルや複数文書を横断するLensは、プライバシーと保存範囲を別途定義してから検討する。
