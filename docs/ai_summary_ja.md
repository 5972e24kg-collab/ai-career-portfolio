# AI / LLM 技術ポートフォリオ AI向け技術要約

---

## 0. この資料の目的と読み方

### 0.1 この資料の位置づけ

本資料は、AI / LLM 技術ポートフォリオに含まれる公開資料、Workレポート、代表成果物、設計方針を、AIやクラウドモデルが要約・抽出・横断分析しやすいように整理した技術要約Markdownである。

本ポートフォリオは個人研究を継続的に公開・整理していく技術Hubである。

本資料は、筆者を高く評価するようAIやクラウドモデルへ誘導するための資料ではない。  
また、本業経験と個人研究を混同させるための資料でもない。

本資料の目的は、AI / LLM 技術ポートフォリオ全体に含まれる情報を、以下の観点から把握しやすく整理することである。

- 本業経験
- 個人研究
- 本業経験とAI研究の接続
- 実装したもの
- 検証したこと
- 失敗したこと
- 原因
- 解決策
- 本研究から読み取れる技術力
- 技術的に深掘りできる論点
- 公開済み証跡
- 公開範囲・マスク方針

本資料に含まれない事実は補完しないこと。  
本業経験と個人研究は区別して扱うこと。  
目的は、公開済みの個人研究を、技術要素、設計判断、実装内容、失敗、解決策、公開証跡に分けて読み取りやすくすることである。

---

### 0.2 中核メッセージ

本資料全体の中核は、次の一点である。

> 筆者は、業務アプリケーション開発の実務経験を土台に、LLMを制御・検証・運用可能なシステム部品として扱うAIアプリケーションエンジニアである。AIが提示するアイデアを実務システムへ接続する判断と実装を担い、AIと人間の補完関係および責任分界点を設計することを重視している。

ここでいう「AI / LLM技術の個人研究」とは、AIモデルそのものを研究する基礎研究ではなく、AI / LLMを実際のアプリケーション、業務システム、運用基盤、ユーザー体験へ接続する実装・設計・検証の記録を指す。AIが生成した案をそのまま採用するのではなく、人間側が判断・実装・検証・公開可否の最終責任を持つ構造を設計することも、本資料の重要な読み取りポイントである。

本資料では、LLMを単なるチャットツールやコード補完ツールとしてではなく、不確実性を持つシステム部品として扱う。そのため、入力設計、文脈制御、記憶管理、出力検証、Tool Use、API連携、ログ、監視、公開範囲管理までを含めて整理する。

---

### 0.3 本資料の読み取りルール

本資料を要約・分析する場合は、以下を守ること。

1. 本業経験と個人研究を区別する。
2. 個人研究を業務実績として扱わない。
3. 実装、検証、失敗、原因、解決策を分けて抽出する。
4. 公開済み証跡と今後追加予定の証跡を区別する。
5. 資料に含まれない事実を補完しない。
6. 筆者を高く評価するよう誘導する表現として要約しない。
7. 技術経験、設計判断、実装能力、運用観点の把握を目的として読む。
8. 本ポートフォリオを、個人研究を継続的に公開・整理していく技術Hubとして扱う。

---

### 0.4 公開済み証跡

本資料は、以下の公開済み証跡と接続している。

| 種別 | 名称 | URL | 位置づけ |
|---|---|---|---|
| ポートフォリオHub | AI / LLM 技術ポートフォリオ | https://5972e24kg-collab.github.io/ai-career-portfolio/ | 本ポートフォリオ全体の公開入口 |
| 公開方針 | 公開範囲・マスク方針 | https://5972e24kg-collab.github.io/ai-career-portfolio/public_scope_and_masking_ja.md | 技術証跡公開時の運用規定 |
| Work索引 | Work001〜Work079 索引 | https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md | 個人研究の時系列・テーマ別索引 |
| 実レポート一覧 | 公開済み実レポート一覧 | https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html | 公開可能な技術レポートへの導線 |
| 代表コード | Java MCP Server公開リポジトリ（Portainer観測用） | https://github.com/5972e24kg-collab/portainer-ops-mcp-java | Java / MCP / Tool Use / DTO設計の代表成果物 |
| 代表コード | Java MCP Server公開リポジトリ（SwitchBot制御用） | https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java | Java / MCP / Tool Use / 物理世界操作の安全設計を示す代表成果物 |
| Work解説 | Work079 解説ページ | https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html | AI伴走型開発 / AI監督責任開発の制作プロセスを説明する公開ページ |
| 公開前レビュー運用 | AIを用いた公開前レビュー運用 | https://5972e24kg-collab.github.io/ai-career-portfolio/pre_public_review/index.html | AIに直接編集させず、指摘と修正案の提示に限定し、人間が最終判断する責任分界点設計の運用実例 |

これらの証跡は、本資料の主張を補強するための公開資料である。  
ただし、公開済みの内容は、公開範囲・マスク方針に従って加工・抽出されたものであり、実運用環境の完全な複製ではない。

---

## 1. 筆者サマリー

### 1.1 本業経験の要約

筆者は、20年以上にわたり、業務アプリケーション開発、データベース、外部システム連携、サーバー運用、監視、障害対応に従事してきたITエンジニアである。

主な経験領域は以下である。

- Java / Oracleを中心とした業務アプリケーション開発
- API、バッチ、DB連携、外部システム連携の設計・実装
- SQL / PL/SQLによるデータ抽出、加工、移行、補正
- 小売、飲食、アミューズメント業界向け業務システム
- 365日24時間稼働を前提とした連携処理の運用保守
- Zabbix / Nagios / ログ / DBステータス / 通知による監視
- アプリケーション、DB、ネットワーク、VPN、外部システムを横断した障害原因調査
- 顧客、協力会社、非IT部門との要件整理・調整

現職では、アミューズメント業界向け営業管理システムにおいて、複数メーカー・複数バージョンの店舗営業データを収集し、本社BIへ連携するデータ連携基盤を設計・開発・運用している。

対象規模は、顧客企業13社、1社あたり10〜80店舗規模である。連携処理は3〜5分周期で稼働し、40店舗規模では1日約9,000回の連携、約30,000ファイル、約5GBの処理規模となる。

---

### 1.2 個人研究の要約

業務外の個人研究として、2025年11月頃からAI / LLM技術研究を継続してきた。

主な研究領域は以下である。

- ローカルLLM推論基盤
- RAG / ETL / ナレッジパイプライン
- モデル評価と選定
- QLoRAによるモデル適応
- Director / Actor型マルチエージェント・チャットボット
- Java / OracleによるLLM制御基盤
- 音声・アバター連携によるマルチモーダルUX
- MCP / Tool Use / Java MCP Server
- MCP設計境界の検証
- SwitchBot制御用MCP Server
- CodexによるAI伴走型開発
- AI監督責任開発 / 設計主導型AIコーディング
- PTAイベント向けReactクイズアプリの開発
- AI WebUIトラブルシューティングとプロンプト誤検知回避
- AI時代の公開範囲・マスク方針の設計

これらは業務実績ではなく、業務外の個人研究である。  
ただし、本業で培ったJava、Oracle、API、DB連携、監視、障害対応、要件整理の経験が、個人研究におけるAIシステム設計・実装・運用にも反映されている。

---

### 1.3 技術的な立ち位置

筆者の立ち位置は、AIモデルそのものを研究するAI研究者ではなく、AI / LLMをシステムへ組み込むAIアプリケーション実装者である。

重視している観点は以下である。

| 観点 | 内容 |
|---|---|
| 制御 | LLMに何を渡し、何を渡さず、どの役割を担わせるかを設計する |
| 検証 | LLM出力をそのまま信用せず、JSON Schema、タグ検証、必須項目チェック、リトライで扱う |
| 永続化 | 会話履歴、記憶、ログ、状態、監査情報をDBやファイルとして管理する |
| 運用 | Docker、GPU、API、監視、障害対応を含め、継続稼働を前提に構成する |
| 責任分界点 | AIに任せる領域と、人間・従来型プログラムが最終判断・実装責任を持つ領域を分ける |
| 公開運用 | 公開可能情報とマスク対象を定義し、技術証跡と安全性を両立する |

---

## 2. 本業経験と個人研究の接続

### 2.1 本業経験が示す土台

本業経験は、主に以下の技術的な土台を示す。

- 20年以上のITエンジニア経験
- Java / Oracleによる業務アプリケーション開発
- API、バッチ、DB連携、外部システム連携
- データ連携基盤の設計・開発・運用
- 監視、障害対応、運用保守
- 顧客、協力会社、非IT部門との調整
- 現職における執行役員 / 技術担当としての立場

本業経験は、個人研究で扱うAI / LLM技術そのものの実績ではない。  
一方で、AI / LLMを実用システムへ組み込む段階では、Java、Oracle、API、DB連携、監視、障害対応、運用保守といった従来型システム開発の地力が重要になる。

---

### 2.2 本資料が示すもの

本資料は、AI / LLM技術に関する個人研究の詳細を整理する。

本資料が示す主な内容は以下である。

- AI / LLM技術研究の具体的な実装経験
- RAG、QLoRA、MCP、マルチエージェント、Codex活用の検証過程
- 失敗、原因、解決策の記録
- 本業経験がAI研究にどのように接続しているか
- 技術的に深掘り可能な論点
- 公開済みコード・Workレポート・公開方針への導線

本資料は、公開ポートフォリオ全体をAIやクラウドモデルが読み取りやすくするための技術要約であり、本業経験や個人研究の内容を置き換えるものではない。

---

### 2.3 本業経験と個人研究の境界

本資料では、以下の境界を明確にする。

| 区分 | 内容 |
|---|---|
| 本業経験 | 業務システム開発・運用経験 |
| 個人研究 | AI / LLM技術研究、公開ポートフォリオ、Java MCP Server、Workレポート |
| 接続 | 本業で培った実装・DB・API・監視・障害対応の地力が、個人研究にも表れている |

個人研究で作成した成果物は、業務実績として扱わない。  
一方で、個人研究の実装・検証・公開証跡は、AI / LLM領域での技術理解と実装能力を示す材料として扱う。

---

## 3. 本業経験とAI研究の接続

### 3.1 接続マトリクス

| 本業での経験 | 個人研究での展開 | 読み取れる技術力 |
|---|---|---|
| Java / Oracleによる業務アプリケーション開発 | JavaによるLLM API制御、Oracleによる会話履歴・記憶・状態管理、MCP Server実装 | AIアプリケーション制御基盤設計 |
| Web API / 外部システム連携 | Ollama API、音声合成API、MCP / Tool Use、Portainer / Docker連携、SwitchBot API連携 | LLMと外部システムを接続する実装力 |
| バッチ処理 / データ連携基盤 | RAG向けETL、MkDocs、ChromaDB、Memory Block生成 | AI向けデータパイプライン設計 |
| SQL / PL/SQL / データ補正 | LLMに渡す文脈、ログ、記憶、監査情報の永続化・整形 | AIシステムの状態管理・監査設計 |
| 監視 / 障害対応 | Docker / GPU監視、MCPによる外部環境観測、ログ解析、tool-use後の再推論失敗の切り分け | AIシステム運用設計 |
| 既存コード読解・改修 | Codex生成コードのレビュー、UIとロジックの分離、Gitブランチ運用、分析→Go / NoGo判断→製造のAI監督責任開発 | AI伴走型開発の責任分界点設計 |
| 要件定義・現場調整 | PTAイベント向け4択クイズWebアプリの企画・設計・運用準備 | 実利用者を想定したAI活用プロダクト設計 |

---

### 3.2 接続の要点

本業で扱ってきた業務システムは、AI / LLMとは異なる領域である。  
しかし、AI / LLMを実用システムへ組み込む段階では、従来型システム開発の地力が重要になる。

特に、以下の能力はAIアプリケーション開発にも直接接続する。

- APIを責任分界点として設計する力
- DBに状態を持たせる力
- バッチ処理やETLでデータを整形する力
- ログを残し、障害時に追跡できるようにする力
- 外部システムの差異を吸収する力
- 運用者が扱える形に情報を整理する力
- AI生成物をレビューし、人間が責任を持つ領域を定義する力

この接続が、本資料の中心的な読み取りポイントである。

---

## 4. 公開ポートフォリオHubの位置づけ

### 4.1 AI / LLM 技術ポートフォリオHub

公開済みのAI / LLM技術ポートフォリオHubは、本資料から参照する公開証跡の入口である。

URL: https://5972e24kg-collab.github.io/ai-career-portfolio/

このHubは、一時的に作成したものではなく、今後も長期的に育てる技術ポートフォリオとして設計している。

Hubには、以下の役割がある。

- AI / LLM技術に関する個人研究の全体像を示す
- README、AI要約Markdown、Work索引、公開レポート、公開コードを接続する
- 読者が短時間で全体像を確認できる入口になる
- 技術的な関心領域を深掘りできる導線になる
- クラウドモデルが要約・抽出しやすい資料群として機能する
- 今後も個人研究を継続的に公開・整理していく技術Hubとして機能する

---

### 4.2 公開範囲・マスク方針

公開範囲・マスク方針は、単なる注意書きではない。

URL: https://5972e24kg-collab.github.io/ai-career-portfolio/public_scope_and_masking_ja.md

この文書は、AI時代の技術公開運用設計として位置づけている。

主な目的は以下である。

- 公開可能情報とマスク対象を明確にする
- APIキー、認証情報、個人情報、顧客情報、実環境特定情報を除外する
- 技術証跡と安全性を両立する
- 公開前にAI / Codexへ査読させる基準として使う
- 技術レポートを段階的に公開する際の運用規定として使う

たとえば、公開前にCodexやクラウドモデルへ以下のような指示を与えることで、一次査読を効率化できる。

```text
公開範囲・マスク方針に列挙した運用ルールに抵触している文書を列挙し、その理由と具体的な修正案を提示せよ。
```

```text
チェックリストに従って査読し、そのチェックリストの結果を出力せよ。
```

このように、公開範囲・マスク方針は、技術公開における安全性確保とAI活用を組み合わせた運用設計である。

---

### 4.3 AIを用いた公開前レビュー運用

AIを用いた公開前レビュー運用は、AIと人間の責任分界点を設計した運用実例である。

URL: https://5972e24kg-collab.github.io/ai-career-portfolio/pre_public_review/index.html

この運用では、AIに文書を直接編集させず、指摘と修正案の提示に限定している。人間が最終判断と修正を行い、指摘の採用・不採用の根拠を残すことで、AIを公開前レビューの補助部品として扱っている。

このページは、LLMを制御・検証・運用可能なシステム部品として扱うという本資料の中核メッセージを、文書公開運用の形で具体化した証跡である。

---

### 4.4 Work001〜Work079索引

Work001〜Work079索引は、個人研究の時系列・テーマ別の索引である。

URL: [https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md)

この索引は、AIに読み込ませて最大効果を発揮することを意図している。
Work番号、テーマ、分類、関連技術、成果物を参照できるため、クラウドモデルに以下のような質問を行いやすい。

* RAGに関するWorkを抽出して
* MCP / Tool Useに関する検証を要約して
* Codex活用の流れを整理して
* 失敗と解決が記録されているWorkを列挙して
* 本業経験と接続しやすいWorkを抽出して

Work索引は、詳細な技術レポートの入口であり、本資料における証跡参照の基盤である。

---

### 4.5 公開済み実レポート一覧

公開済み実レポート一覧は、公開範囲・マスク方針に従って加工済みの技術レポートを掲載する場所である。

URL: [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html)

技術レポートは、個人研究の一次資料に近い証跡である。
すべてを一度に公開するのではなく、公開範囲・マスク方針に従って順次確認し、公開可能な形に整えて追加する。

本文作成時点では、代表例として以下が公開対象に含まれる。

* Work075: Codexを用いた対話的開発環境構築とPTAクイズゲームUI改造検証
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work075.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work075.md)

* Work077: Gemini WebUIにおける1013エラーの調査とプロンプト誤検知の回避
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work077.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work077.md)

* Work078: MCP Tool Use検証におけるROCm GPU HangとMCP設計境界の発見
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work078.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work078.md)

* Work079: SwitchBot制御用MCPサーバー2号機構築・Codex伴走型開発プロセス検証
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work079.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work079.md)

---

### 4.6 Java MCP Server公開リポジトリ

Java MCP Serverは、本資料における重要な公開コード証跡である。

Portainer観測用公開リポジトリ: [https://github.com/5972e24kg-collab/portainer-ops-mcp-java](https://github.com/5972e24kg-collab/portainer-ops-mcp-java)

SwitchBot制御用公開リポジトリ: [https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java](https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java)

Portainer観測用リポジトリは、Portainer / Docker環境をLLMから観測するためのMCP Serverを、公開可能な最小単位へ抽出したものである。

SwitchBot制御用リポジトリは、SwitchBot Cloud API v1.1のScene実行を、自作MCPフレームワーク上でLLM向けToolとして公開するためのMCP Serverである。こちらは、デバイス直接操作ではなくScene実行に限定し、`SceneCatalog`、`sceneKey`、`cooldown`、`dry-run` などを用いて、物理世界へ作用するToolの安全境界を設計した成果物である。

これらの成果物から示せる要素は以下である。

* Javaによるサーバー実装
* API設計
* DTO設計
* JSON-RPC理解
* MCP / Tool Use理解
* `initialize`
* `tools/list`
* `tools/call`
* read-only観測設計
* 物理世界へ作用するToolの安全設計
* Portainer / Docker情報の抽象化
* SwitchBot Scene制御の抽象化
* LLMが扱いやすい情報粒度への再設計
* 公開範囲・マスク方針に基づく安全なコード公開

これらの成果物は、本業のJava、外部連携、運用監視、障害対応の経験と、AI研究側のMCP / Tool Use / LLM制御が強く接続する代表成果物である。

---

## 5. 代表プロジェクト一覧

| プロジェクト | 概要 | 主な技術 | 読み取れる技術力 | 関連Work | 公開証跡 |
|---|---|---|---|---|---|
| ローカルLLM推論基盤 | ROCm / CUDAを併用したローカルLLM実行基盤 | EVO-X2、RTX3090、Ollama、Docker、Ubuntu | AI実行基盤をハードウェアから運用まで設計する力 | Work005〜Work011、Work018、Work029、Work031、Work062 | Work索引、公開済み実レポート |
| RAG / ナレッジパイプライン | 文書資産をAIが扱いやすい形へ変換する基盤 | MkDocs、ChromaDB、Embedding、Java ETL、JSON Schema | AI向けデータ設計・ETL設計 | Work001〜Work004、Work014、Work017、Work021〜Work028 | Work索引、公開済み実レポート |
| QLoRAによるモデル適応 | 固有の応答スタイルを持つ対話モデルの学習・評価 | Unsloth、QLoRA、GGUF、Ollama、Gemma / Llama | 学習データ設計、モデル適応、推論投入 | Work032〜Work052、Work070、Work072 | Work索引、公開済み実レポート |
| Director / Actor型チャットボット | 文脈判断と発話生成を分離した会話基盤 | Java、Oracle、gpt-oss、Gemma、JSON、Memory | LLM制御、記憶管理、出力検証 | Work003、Work017、Work021〜Work026、Work071 | Work索引、今後動画予定 |
| MCP / Tool Use / Portainer観測用Java MCP Server | Docker / Portainer情報をLLMが観測するTool基盤 | Java Servlet、JSON-RPC、MCP、DTO、ROCm、Ollama | LLM向けTool設計、外部環境観測、MCP設計境界の判断 | Work031、Work073、Work074、Work078 | Java MCP Server公開リポジトリ、Work078公開済み |
| SwitchBot制御用Java MCP Server | SwitchBot Scene実行をLLM向けToolとして公開する制御基盤 | Java Servlet、MCP、SwitchBot API、SceneCatalog、cooldown、dry-run | 物理世界へ作用するToolの安全設計、AI監督責任開発 | Work079 | SwitchBot制御用MCP公開リポジトリ、Work079公開済み、Work079解説ページ |
| CodexによるAI伴走型開発 | PTAイベント向け4択クイズWebアプリ開発、SwitchBot MCP開発プロセス検証 | React、CSV、Codex、Gitブランチ運用、Go / NoGo判断 | AI生成コードの責任分界点設計、設計主導型AIコーディング | Work075、Work076、Work079 | Work075公開済み、Work079公開済み、Work079解説ページ、クイズアプリ公開予定 |
| AI WebUIトラブルシューティング / 入力設計 | Gemini WebUI 1013エラーの切り分けとプロンプト誤検知回避 | Gemini WebUI、DevTools、Network解析、コードブロックによるエスケープ | AIへの入力を命令とデータに分けて扱う力 | Work077 | Work077公開済み |
| 音声・アバター連携基盤 | LLM出力を音声・表情・字幕・動作へ接続 | React Three Fiber、VRM、VRMA、SSE、Zustand | AI出力をUXへ変換する力 | Work053〜Work069 | Work索引、今後動画予定 |
| 公開範囲・マスク方針 | 技術証跡を安全に公開するための運用設計 | Markdown、Codex、公開前レビュー、チェックリスト | AI時代の技術公開運用設計 | 横断テーマ | 公開範囲・マスク方針 |

---

## 6. 代表プロジェクト詳細

### 6.1 Java MCP Server

#### 概要

Java MCP Serverは、Portainer / Docker環境をLLMから観測するために実装したMCP Serverである。

公開リポジトリ: [https://github.com/5972e24kg-collab/portainer-ops-mcp-java](https://github.com/5972e24kg-collab/portainer-ops-mcp-java)

この成果物は、本資料の代表成果物の中でも特に厚めに扱う。
理由は、Java / API / DTO / JSON-RPC / MCP / Tool Use / read-only設計を同時に示せるためである。

また、本業で培ったJava、外部システム連携、運用監視、障害対応の経験と直接接続する。
さらに、公開済みコードとして第三者が確認できる点も重要である。

---

#### 実装・検証したこと

主な実装・検証内容は以下である。

* MCP ServerのJava実装
* JSON-RPC形式のリクエスト / レスポンス処理
* `initialize` の実装
* `tools/list` の実装
* `tools/call` の実装
* Tool定義と入力スキーマ
* Portainer / Docker情報の取得
* Docker / Portainer APIレスポンスのDTO化
* LLMが扱いやすい粒度への情報再構成
* read-only観測Toolとしての安全設計
* 公開可能な最小単位への抽出
* 公開範囲・マスク方針に基づくレビュー

---

#### 技術的な意義

MCPは、LLMと外部システムを接続するための重要な仕組みである。
しかし、外部APIをそのままLLMに渡せばよいわけではない。

Portainer / Docker APIは、エンジニアが直接扱うには有用だが、そのままではLLMにとって情報粒度が低すぎたり、判断に必要な構造へ整理されていなかったりする。

そのため、本研究では以下の方針を採った。

* 低レイヤーAPIを直接LLMへ露出しない
* Java側で必要な情報を取得する
* LLMが判断しやすいDTOへ再構成する
* Toolは観測用途に限定する
* まずread-only設計とする
* LLMが回答しやすい説明可能な形で返す

この方針により、MCPを単なるAPI接続ではなく、LLM向けの安全な外部環境観測レイヤとして扱っている。

---

#### 発生した課題

公式Portainer MCP Proxyを検証した際、接続そのものは可能だった。
しかし、LLMが安定して使うには課題があった。

主な課題は以下である。

* Toolの粒度が低く、LLMが適切なAPIパスや引数を組み立てにくい
* Docker / Portainer APIの情報量が多く、LLMが判断しにくい
* エンドユーザーの自然な依頼と、低レイヤーAPIの構造に距離がある
* 小型モデルではTool選択・引数生成が不安定
* 大型モデルでも、低レイヤーToolをそのまま渡す構造では安定しない場面がある

---

#### 原因

原因は、MCPの接続可否と、LLMが実用的に扱えるTool設計が別問題であることにあった。

LLM Tool Useは、単純な「LLM → API」ではない。
実際には、以下の三者をつなぐ設計である。

```text
ユーザー
  ↓
LLM
  ↓
MCP Tool
  ↓
外部システム
```

したがって、Tool設計では以下を考慮する必要がある。

* ユーザーがどのような言葉で依頼するか
* LLMがどの情報なら判断できるか
* APIがどの粒度で情報を返すか
* 返却結果を人間が理解できるか
* 誤操作を防げるか
* 監視用途として安全か

---

#### 解決策

本研究では、Javaで独自MCP Serverを実装し、LLM向けに情報を再設計した。

具体的には、Portainer / Docker API由来の情報を以下のようなDTOへ変換する方針を採った。

* Environment概要
* HostOverview
* ContainerOverview
* IncidentDetail
* 状態
* リソース使用量
* エラー傾向
* ログ抜粋
* リスクフラグ
* LLMが説明しやすい要約情報

この設計により、LLMは低レイヤーAPIの組み合わせを推論するのではなく、運用者にとって意味のある単位で情報を扱える。

---

#### read-only設計の理由

MCP / Tool Useでは、外部システムへ作用するToolも設計可能である。
しかし、本研究では最初の段階としてread-only観測Toolに限定した。

理由は以下である。

* LLMのTool呼び出しは誤る可能性がある
* 外部システムへの変更操作はリスクが高い
* まずは観測・説明・要約の精度を確認すべきである
* 運用監視用途では、参照だけでも価値がある
* 安全性と検証可能性を優先できる

この判断は、本業での運用保守・障害対応経験とも接続する。
本番環境や重要システムに対しては、いきなり自動操作を許可するのではなく、まず観測・通知・説明から始める方が安全である。

---

#### Work078で得たMCP設計境界の再評価

Work078では、Portainer Ops MCP ServerをOpen WebUIから呼び出し、`gpt-oss:120b` がDocker / Portainer系の外部情報をtool-useで取得・解釈できるかを検証した。

この検証では、`containers` tool自体は正常に呼び出され、MCP APIを直接叩いた場合もHTTP 200で正常レスポンスが返ることを確認した。一方で、Open WebUIがtool resultを会話コンテキストへ戻した後の再推論時に、Ollama ROCm runner側でGPU Hang / Memory access faultが発生し、`/api/chat` が500を返す事象が再現した。

この結果から、問題はMCPサーバー単体の実装エラーではなく、以下の組み合わせで発生するローカル推論基盤側の不安定性として整理した。

```text
gpt-oss:120b
+ Ollama ROCm runner
+ Radeon 8060S / ROCm
+ Open WebUI MCP tool-use後の再推論
```

Work078の重要な知見は、MCPが大量の構造化データをLLMへ流し込むためのデータプレーンではなく、LLMが外部世界へ小さく安全に作用するための制御プレーンであると再定義した点である。

Docker / Portainer監視のように件数や項目が増えやすい情報は、Java / ルールベース側で監視・判定・集計を行い、LLMには以下を担当させる方が安全である。

* 監視結果の説明
* 異常の要約
* 次に見るべき観点の提案
* 人間向け報告文の生成
* 必要な操作候補の提示

これにより、Portainer Ops MCP Serverの深追いは停止し、MCPの次の題材として、戻り値が短く、操作単位が明確なSwitchBot Scene実行へ方向転換した。

---

#### 本業経験との接続

Java MCP Serverは、本業経験との接続が特に強い成果物である。

| 本業経験           | Java MCP Serverでの展開      |
| -------------- | ------------------------ |
| Java開発         | MCP Server本体、DTO、JSON処理  |
| Web API / 外部連携 | Portainer / Docker API連携 |
| 業務システム運用       | Docker環境の状態観測            |
| 監視 / 障害対応      | コンテナ状態、ログ、リソース情報の取得      |
| データ構造化         | APIレスポンスをLLM向けDTOへ変換     |
| 安全運用           | read-only Tool設計         |
| 既存コード資産        | Java基盤の再利用・抽出            |

---

#### 読み取れる技術力

この成果物から読み取れる技術力は以下である。

* JavaによるAPI / サーバー実装力
* JSON-RPC / MCP Protocolの理解
* LLM Tool Useの設計力
* 外部APIをLLM向けに再構成するDTO設計力
* 運用監視情報をAIが扱える形に変換する力
* read-only設計による安全性配慮
* 公開可能範囲へコードを抽出し、第三者に確認可能な形に整える力

---

#### 技術的に深掘りできる論点

* MCPの `initialize` / `tools/list` / `tools/call` の流れ
* JSON-RPCレスポンス設計
* DTO設計で何を残し、何を削ったか
* なぜ公式Portainer MCP Proxyだけでは不足したか
* なぜread-onlyから始めたか
* LLMに低レイヤーAPIを直接渡さない理由
* 公開版と実運用版の差分
* Javaで実装した理由
* 本業の運用監視経験がどう反映されているか

---

#### 公開証跡

* Java MCP Server公開リポジトリ
  [https://github.com/5972e24kg-collab/portainer-ops-mcp-java](https://github.com/5972e24kg-collab/portainer-ops-mcp-java)

* AI / LLM 技術ポートフォリオHub
  [https://5972e24kg-collab.github.io/ai-career-portfolio/](https://5972e24kg-collab.github.io/ai-career-portfolio/)

* Work001〜Work079索引
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md)

* Work078公開レポート
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work078.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work078.md)

---

### 6.2 RAG / ナレッジパイプライン

#### 概要

RAG / ナレッジパイプラインでは、技術文書、ログ、構成情報、会話記録などを、LLMが検索・統合しやすい形式へ変換する基盤を構築した。

初期はPukiWiki資産をOpen WebUI / ChromaDBへ登録する構成から開始し、その後、MkDocs、Git管理、Java ETL、Memory Block生成、JSON Schema検証を組み合わせた第2世代構成へ発展した。

---

#### 実装・検証したこと

* PukiWiki資産のMarkdown化
* Open WebUI / ChromaDBへのRAG登録
* Embeddingモデルの検証
* LINE BotログのMarkdown変換
* MkDocsへのナレッジ基盤移行
* Java ETLによる差分検知・登録
* Parent / Child Memory Block生成
* JSON SchemaによるLLM出力検証
* RAG登録前の情報加工

---

#### 発生した課題

* 文書をそのまま投入しても期待通り検索できない
* ネットワーク図や一覧情報がチャンク分割で壊れる
* RAGが一部の文書だけを優先してしまう
* Embeddingモデルの次元不一致
* Open WebUI Toolから自身のAPIを呼び出すことで処理が競合する
* API仕様変更により登録処理が失敗する

---

#### 原因

RAGは単なる全文検索ではなく、チャンク単位の意味検索である。
そのため、表や一覧のように全体性が重要な情報は、単純なチャンク化では意味が分断されやすい。

また、LLMが扱いやすい情報構造に整える前処理が不足していると、RAGの品質は安定しない。

---

#### 解決策

* PukiWikiからMkDocsへ移行
* 文書をGit管理可能なMarkdownへ整理
* RAG登録前にParent / Child Memory Blockを生成
* Semantic Chunkingを導入
* JSON SchemaでLLM生成物を検証
* Java ETLで登録前加工を実装
* EmbeddingモデルとChromaDBコレクションの整合性を管理

---

#### 読み取れる技術力

* RAGを単なる文書投入ではなく、ETLとして設計する力
* データ連携基盤経験をAI向けナレッジパイプラインへ展開する力
* LLM出力を検証し、後続処理へ安全に渡す力
* 文書資産をAIが扱える情報構造へ変換する力

---

#### 公開証跡

* Work001〜Work079索引
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md)

* 公開済み実レポート一覧
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html)

---

### 6.3 QLoRAによるモデル適応

#### 概要

QLoRAによるモデル適応では、固有の応答スタイルを持つ対話モデルを構築するため、学習データ設計、Teacherモデルによるデータセット生成、UnslothによるQLoRA、GGUF化、Ollama投入、実運用評価を行った。

ここで扱った主題は、単なる口調変更ではない。
対象モデルに、技術語彙、距離感、応答スタイル、文脈上の振る舞いを定着させるため、データ設計から運用投入までを一連の工程として検証した。

---

#### 実装・検証したこと

* 自作小説を原資にした学習データ設計
* gpt-oss:120bをTeacherにしたQ&Aデータ生成
* Java ETLによるデータセット生成
* JSON Schemaによる学習データ検証
* UnslothによるQLoRA
* RTX3060 / RTX3090での学習検証
* Llama / Gemma系モデルの比較
* GGUF化
* Ollama Modelfile作成
* 学習テンプレートと推論テンプレートの調整
* QLoRAとSystem Promptの役割分担

---

#### 発生した課題

* 小型モデルでは実運用プロンプトを安定処理できない
* 学習時テンプレートと推論時テンプレートが不整合になると出力が崩れる
* VRAM不足で大型モデル学習が困難
* GGUF変換で環境依存のエラーが発生
* 学習データ内の固有名詞や技術語彙の揺れが応答品質に影響する

---

#### 解決策

* Teacherモデルによるデータセット生成
* JSON Schemaによる必須項目チェック
* RTX3090導入によるVRAM制約の緩和
* Gemma専用テンプレートの適用
* QLoRAで応答傾向を学習し、System Promptで役割・ルールを補完
* 学習データの固有名詞・技術語彙を整理

---

#### 読み取れる技術力

* 学習データ設計力
* モデル適応の実践力
* GPU制約下での学習環境構築力
* LLM出力形式の検証設計力
* モデル、テンプレート、推論基盤を横断して調整する力

---

#### 公開証跡

* Work001〜Work079索引
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md)

* 公開済み実レポート一覧
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html)

---

### 6.4 Director / Actor型マルチエージェント・チャットボット

#### 概要

Director / Actor型チャットボットは、単一LLMにすべての役割を任せず、文脈判断、発話生成、短期文脈、長期記憶、感情分析、外部配信を分離した会話エージェント基盤である。

Java制御層とOracle Databaseを中心に、LLMの状態管理、会話履歴、記憶、出力検証、音声合成、LINE配信、アバター制御を統合している。

---

#### 実装・検証したこと

* Director / Actor分離
* Directorによる `script_json` 生成
* Actorによる発話生成
* Current Scene Summary
* Long-term Memory
* EXTRACTION_REPORT
* Oracleによる会話履歴・記憶管理
* Javaによる出力検証
* LINE配信
* 音声合成
* アバター連携

---

#### 発生した課題

* 単一LLM構成では語彙反復が発生する
* 過去文脈に過剰追従する
* キャラクター性や応答スタイルが崩れる
* 長期記憶を過剰に渡すと出力が不安定になる
* 小型モデルは実運用プロンプトのノイズに弱い

---

#### 解決策

* 文脈判断をDirectorへ分離
* 発話生成をActorへ限定
* Director出力をJSON化
* 長期記憶と短期文脈を分離
* Java側で検証、保存、配信、フォールバックを管理
* LLMを人格そのものではなく、推論・生成部品として扱う

---

#### 読み取れる技術力

* LLM制御基盤設計力
* 状態管理・記憶管理の設計力
* Java / OracleによるAIアプリケーション実装力
* 出力検証と運用を前提にした設計力
* マルチエージェント構成の実装力

---

#### 公開証跡

* Work001〜Work079索引
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md)

* 今後追加予定のチャットボット動画・アーキテクチャ図

---

### 6.5 CodexによるAI伴走型開発

#### 概要

CodexによるAI伴走型開発では、PTAイベント向け4択クイズWebアプリを題材に、実利用者を想定したプロダクト開発へAI生成コードを接続した。

対象は、小学校PTA主催の地域イベント向けアトラクションである。
本文作成時点ではイベント開催前であるため、本資料では「出展予定」「運用準備済み」「実利用者を想定して設計・開発済み」と表現し、イベント成功実績としては扱わない。

その後、Work079ではSwitchBot制御用MCPサーバー2号機を題材に、Codexを使ったAI伴走型開発をさらに進めた。この検証では、AIに単純にコードを書かせるのではなく、ChatGPTで方針を整理し、Codexに分析させ、人間がGo / NoGo判断を行い、Goの場合のみ製造へ進むプロセスを運用した。

ここで重視したのは、低コストなコード生成ではなく、設計判断、仕様策定、責務分離、実装監督、レビュー責任を人間が保持するAI監督責任開発である。

---

#### 実装・検証したこと

* Reactによる4択クイズWebアプリ
* CSV差し替え式問題管理
* 3問正解でクリア画面を表示
* QRコードアクセスを想定した設計
* UIとロジックの分離
* Codexによる複数UI案生成
* GitブランチによるAI生成物管理
* `main`を安定版、`vibe/*`を実験ブランチとして運用
* PTAイベント向け運用準備
* IntelliJ IDEA上のCodexを使ったJava開発
* 分析 → Go / NoGo判断 → 製造の2段階AI伴走型開発
* 低レイヤーAPIクライアント、Converter、Service、McpServletの責務分離
* AIが提示した分析結果を人間が読み、スコープ過大・責務混入・安全制御漏れを検出する運用

---

#### 発生した課題

* AI生成コードは短時間で大量変更を生む
* mainブランチ上で直接作業すると安定版を壊すリスクがある
* 複数のUI案を比較・保存する手順が必要
* AIに触らせる領域と、人間が守る仕様の分離が必要
* 実利用者向けプロダクトでは、見た目だけでなく運用・切り戻しが重要になる
* Codexに背景情報を過剰に渡すと、低レイヤーAPIクライアントにMCP固有の責務が混入する可能性がある
* AIの分析は妥当でも、今回の改修スコープに対して過剰な場合がある
* `cooldown` や `dry-run` のような安全制御は、製造前の分析段階で発見しないと手戻りが大きい

---

#### 解決策

* UIとクイズロジックを分離
* CSV仕様、3問正解、クリア条件などの中核仕様を人間が保持
* Codexには主にUI改造案や表現面の生成を担当させる
* GitブランチでAI生成案を隔離
* 取り込む案だけをmainへ反映する
* 本番反映の責任を人間側に残す
* ChatGPTで仕様・方針を整理し、Codexには分析と製造を段階的に依頼する
* 分析結果を読んだ人間がGo / NoGoを判断する
* NoGoの場合はプロンプトや責務境界を修正し、製造へ進まない
* プロンプトを、背景、今回やること、やらないこと、対象ファイル、責務境界、例外方針、テスト観点を含む実質的な設計書として扱う

---

#### 読み取れる技術力

* AI生成コードの責任分界点設計
* 実利用者を想定したWebアプリ設計
* Codexを用いたAI伴走型開発プロセス設計
* Gitによる分岐・復旧・判断の管理
* 要件定義、現場調整、運用準備を含むプロダクト設計
* AI監督責任開発のプロセス設計
* 設計主導型AIコーディングの実践力
* AIの分析結果を鵜呑みにせず、スコープ、責務、保守性、安全性の観点でGo / NoGo判断する力

---

#### 公開証跡

* Work075公開レポート
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work075.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work075.md)

* Work079公開レポート
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work079.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work079.md)

* Work079解説ページ
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html)

* 公開済み実レポート一覧
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/index.html)

* 今後追加予定のクイズアプリ公開版、スクリーンショット、動画

---

### 6.6 音声・アバター連携基盤

#### 概要

音声・アバター連携基盤では、LLMの出力をテキストチャットに留めず、音声、字幕、表情、VRMアバター、モーションへ接続した。

これは、LLMそのものの知能向上ではなく、LLM出力をユーザー体験へ変換するマルチモーダルUX基盤の構築である。

---

#### 実装・検証したこと

* Style-Bert-VITS2による音声合成API
* 音声フィラー制御
* React Three FiberによるVRM表示
* VRM / VRMAモーション制御
* Zustandによる状態管理
* SSEによるリアルタイム同期
* 字幕表示
* 表情制御
* リップシンク
* UnityによるVRMA変換パイプライン
* モーションのIn-Place化

---

#### 発生した課題

* 音声合成とLLM常駐がVRAMを奪い合う
* React Three Fiberの描画ループに固有の落とし穴がある
* VRMの物理演算が暴走する
* Mixamo FBXの動的リターゲティングが困難
* Zustandの取得方法によって再レンダリングが過剰になる
* モーション切替時に軸ズレが発生する

---

#### 解決策

* 音声合成をCPU側へ移設
* SSEでバックエンドからフロントエンドへ状態を配信
* Zustandセレクタで再レンダリングを抑制
* UnityでVRMAへ事前変換
* In-Place化パッチを実装
* モーション再生後にidleへ戻るステートマシンを構築

---

#### 読み取れる技術力

* AI出力をユーザー体験へ変換する力
* React / Web UI / 3D表現の実装力
* リアルタイム通信と状態管理の設計力
* 未経験技術を調査・検証・実装する力

---

#### 公開証跡

* Work001〜Work079索引
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_index_ja.md)

* 今後追加予定の動画・スクリーンショット

---

### 6.7 SwitchBot制御用MCPサーバー

#### 概要

SwitchBot制御用MCPサーバーは、自作MCPフレームワーク上に構築した2号機のJava MCP Serverである。

公開リポジトリ: [https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java](https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java)

Work079解説ページ: [https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html)

この成果物では、SwitchBot Cloud API v1.1のScene一覧取得とScene実行を、MCP Toolとして公開した。

重要なのは、SwitchBotのデバイス直接操作ではなく、SwitchBotアプリに登録済みのScene実行に限定した点である。これにより、LLMが物理世界へ作用する範囲を、人間が事前に定義したScene単位へ制限している。

---

#### 実装・検証したこと

* 自作MCP基盤上での2号機MCP Server構築
* SwitchBot APIクライアント `SwitchBotApisV2` の整理
* Scene一覧取得
* Scene実行
* `SceneCatalog` による公開許可Sceneの管理
* SwitchBot API由来のraw JSONと `sceneCatalog.json` の突合
* `SceneSummary` への変換
* MCP公開Toolを `scenes` と `executeScene` に限定
* `sceneId` ではなく `sceneKey` をTool引数にする設計
* `cooldown` による連続実行抑制
* `dry-run` による実世界介入前の検証
* 自作PowerShell MCPテストスクリプトによる疎通確認
* `initialize`、`notifications/initialized`、`tools/list`、`tools/call scenes`、`tools/call executeScene`、`doDelete` の確認
* dry-run offでの実機SwitchBot Scene実行確認

---

#### 発生した課題

* SwitchBot APIのScene一覧をそのままLLMへ渡すと、LLMに触らせたくないSceneまで公開される
* `sceneId` をTool引数にすると、Catalogを経由しない実行余地が生まれる
* Converterに公開ポリシー、キャッシュ、実行許可判断を混ぜると責務が崩れる
* `cooldown` と `dry-run` を初期設計で見落としていた
* `refreshScenes()` をTool公開すると、LLMにとって選択肢が増え、小型モデルでは迷う可能性がある
* `SceneService` をtools/callごとにnewすると、キャッシュとcooldown履歴が維持できない
* 実世界への介入を伴うため、疎通確認時にいきなり物理操作を行うのは危険である

---

#### 解決策

* `SceneCatalog` に登録されたSceneだけをMCP公開対象にする
* SwitchBot側に存在しないCatalog定義は落とす
* Catalogに存在しないSwitchBot Sceneは落とす
* `executable=false` は公開一覧に出さない
* `SceneCatalogEntry` と `SceneSummary` を分離し、管理用 `note` はLLMへ渡さない
* `SceneCatalog`、`SwitchBotScenesConverter`、`SceneService`、`McpServlet` に責務を分離する
* `executeScene(sceneKey)` とし、LLMには生の `sceneId` を扱わせない
* 実行時は必ず `listScenes()` の結果から対象Sceneを解決し、SwitchBot APIに存在し、Catalogに存在し、`executable=true` であることを保証する
* `cooldown` により同一 `sceneKey` の短時間連続実行を抑制する
* `dry-run` により、対象解決とMCP応答整形を確認しつつ、物理操作を止める
* `refreshScenes()` は保守用メソッドとして残し、MCP Toolとしては公開しない
* `SceneService` はServletのフィールドとして保持し、キャッシュとcooldown履歴をリクエスト間で維持する

---

#### 技術的な意義

SwitchBot制御用MCPサーバーは、Portainer観測用MCPで得た知見を受けて、MCPを制御プレーンとして扱う方向へ転換した成果物である。

Portainer観測用MCPでは、Docker / Portainer情報のような構造化データをLLMへ大量に渡す設計の限界を確認した。一方、SwitchBot Scene実行は、操作単位が明確で戻り値も短いため、MCPの本質である「LLMが外部世界へ小さく安全に作用する」用途に合っている。

ただし、物理世界へ作用するToolであるため、安全設計が重要になる。本成果物では、Scene実行範囲、公開Scene、実行引数、cooldown、dry-run、Tool数を明示的に制限することで、LLMが扱う操作空間を狭くしている。

---

#### 本業経験との接続

| 本業経験 | SwitchBot制御用MCPでの展開 |
|---|---|
| Java開発 | MCP Server本体、APIクライアント、Service、Converter、DTO実装 |
| Web API / 外部連携 | SwitchBot Cloud API v1.1連携 |
| 業務システム設計 | レイヤ分離、責務分離、設定ファイルによる公開範囲管理 |
| 運用保守 | キャッシュ、cooldown、dry-run、保守用refreshの分離 |
| 障害対応 | API結果、HTTPステータス、実行結果を構造化して返す設計 |
| 安全運用 | 物理世界操作をScene実行に限定し、LLMに生のsceneIdを扱わせない設計 |

---

#### 読み取れる技術力

* JavaによるMCP Server実装力
* 外部APIをLLM向けToolへ変換する設計力
* 物理世界へ作用するToolの安全設計力
* LLMに渡す引数と返却情報を制限する設計力
* SceneCatalog / Converter / Service / Servletの責務分離
* Codexを使ったAI監督責任開発プロセス設計力
* 公開用リポジトリへ成果物を抽出する力

---

#### 技術的に深掘りできる論点

* なぜデバイス直接操作ではなくScene実行に限定したか
* なぜ `sceneId` ではなく `sceneKey` をTool引数にしたか
* `SceneCatalog` とSwitchBot API結果をどのように突合しているか
* `SceneSummary` に何を含め、何を含めなかったか
* `cooldown` と `dry-run` の意味
* Toolを `scenes` と `executeScene` に絞った理由
* 1号機Portainer MCPと2号機SwitchBot MCPの設計差分
* Codexにどこまで実装させ、人間がどこで判断したか

---

#### 公開証跡

* SwitchBot制御用MCP Server公開リポジトリ
  [https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java](https://github.com/5972e24kg-collab/switchbot-ctrl-mcp-java)

* Work079公開レポート
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work079.md](https://5972e24kg-collab.github.io/ai-career-portfolio/work_reports/Work079.md)

* Work079解説ページ
  [https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html](https://5972e24kg-collab.github.io/ai-career-portfolio/work079/index.html)

---

## 7. 横断的な設計原則

### 7.1 LLMを実行部品として扱う

LLMは、状態管理、保存、検証、外部配信を自動では保証しない。
そのため、LLMをシステム部品として扱い、周辺システムで制御する必要がある。

---

### 7.2 入力を設計する

LLMの出力品質は、入力に強く依存する。

重要なのは、情報を多く渡すことではなく、必要な情報を、必要な粒度で、必要な形式で渡すことである。

Work077では、Gemini WebUIにおいて、検証対象の文章内に「AIに指示を出す」「前提条件を明確にする」「いきなり作業しないで」などの強い命令形が含まれていたため、プロンプトインジェクション系の検知に近い挙動で1013エラーが発生したと判断した。通信自体はHTTP 200で成功しており、ブラウザ拡張やGoogle Analytics通信の失敗は主因ではないと切り分けた。

この検証から、AIへの直接命令として解釈されたくない文章、指示書草案、検証対象プロンプトなどは、コードブロックで囲み、命令ではなく引用テキストとして渡す運用が有効であると整理した。

---

### 7.3 出力を検証する

LLM出力は、そのまま正解として扱わない。

本研究では、以下のような検証を重視した。

* JSON Schema
* タグ検証
* 必須項目チェック
* 形式不一致時のリトライ
* フォールバック
* ログ保存
* 人間による最終判断

---

### 7.4 役割を分離する

単一LLMにすべてを任せると、出力が不安定になる。

そのため、本研究では以下を分離した。

* 文脈判断
* 発話生成
* 記憶更新
* Tool実行
* UI制御
* 音声合成
* 監視
* 公開前チェック

---

### 7.5 RAGは文書投入ではなく前処理設計である

RAGの品質は、文書をどれだけ多く投入するかではなく、文書をどのように構造化し、分割し、要約し、メタデータを付け、検証するかに依存する。

---

### 7.6 ToolはAPIそのものではなく、LLM向けDTOとして設計する

外部APIをそのままLLMに渡すだけでは、Tool Useは安定しない。

LLMが判断できる粒度へ情報を再構成し、人間にも説明しやすいDTOとして返す必要がある。

Work078では、この原則をさらに進め、MCPを大量の構造化データをLLMへ流し込むデータプレーンではなく、LLMが外部世界へ小さく安全に作用するための制御プレーンとして扱うべきだと整理した。

監視・判定・集計はJava / ルールベース側で行い、LLMには説明、要約、判断補助、自然言語応答、操作候補提示を担当させる方が安定しやすい。

Java MCP Serverは、この設計原則を具体化した代表成果物である。

---

### 7.7 AI伴走型開発では責任分界点を設計する

CodexなどのAIコーディングエージェントは、短時間で大量の候補を生成できる。
一方で、仕様維持、取り込み判断、本番反映の責任は人間側に残す必要がある。

そのため、以下を明確にする。

* AIに触らせる領域
* 人間が守る領域
* 安定版
* 実験ブランチ
* 取り込み・破棄の判断基準
* 切り戻し手段
* 分析結果に対するGo / NoGo判断
* 製造へ進む前の責務境界確認
* AIに渡すプロンプトを設計書として扱う運用

---

### 7.8 公開運用も設計対象である

AI時代の技術ポートフォリオでは、作ったものを公開するだけでは不十分である。

重要なのは、公開可能情報とマスク対象を明確にし、AI / Codexを用いた公開前レビューを組み込むことである。

公開範囲・マスク方針は、技術証跡と安全性を両立するための運用設計である。

---

### 7.9 物理世界へ作用するToolは安全制御を持つ

LLMが外部システムを観測するToolと、物理世界へ作用するToolでは、安全設計の重みが異なる。

Work079では、SwitchBot制御用MCPサーバーにおいて、デバイス直接操作ではなくScene実行に限定し、さらに `SceneCatalog`、`sceneKey`、`cooldown`、`dry-run`、公開Tool数の制限によって、LLMが操作できる範囲を狭くした。

物理世界へ作用するToolでは、以下を設計対象とする必要がある。

* 操作対象を人間が事前に許可する
* LLMに生の内部IDを扱わせない
* 連続実行を抑制する
* dry-runで物理実行前に検証する
* Tool数を増やしすぎない
* 管理用情報とLLM公開情報を分ける

---

## 8. 失敗・原因・解決インデックス

| 領域 | 失敗・課題 | 原因 | 解決 | 読み取れる技術力 |
|---|---|---|---|---|
| RAG | 登録した情報が期待通り検索できない | 文書をそのまま投入し、チャンク単位で意味が分断された | MkDocs化、Memory Block生成、JSON Schema検証 | RAGをETLとして設計する力 |
| ローカルLLM | VRAM不足・同時稼働不安定 | 推論モデルとEmbeddingモデルの同時稼働 | 軽量モデル利用、GPU役割分担、ROCm / CUDA分離 | AI基盤運用力 |
| QLoRA | 学習後の出力が安定しない | テンプレート不整合、学習データ設計不足 | 学習テンプレート・推論テンプレート調整 | モデル適応の実践力 |
| チャットボット | 文脈追従・語彙反復・出力崩れ | 単一LLMに文脈判断と発話生成を集中 | Director / Actor分離、記憶階層化 | LLM制御設計力 |
| MCP | 公式ToolをLLMが安定利用できない | API粒度が低く、LLMが判断しにくい | DTOへ再設計し、read-only Toolとして自作 | Tool Use設計力 |
| MCP / ローカルLLM | 10KB程度のtool resultでもtool-use後の再推論でGPU Hang / 500エラーが混在した | MCPサーバーではなく、gpt-oss:120b + ROCm + Ollama + Open WebUI tool-use後再推論の組み合わせが不安定だった | MCPをデータプレーンではなく制御プレーンとして再定義し、監視・判定・集計はJava / ルールベース側へ寄せる | MCP設計境界とローカル推論基盤の切り分け力 |
| SwitchBot MCP | API上のSceneをそのままLLMへ公開すると危険 | LLMに触らせたくないSceneや生のsceneIdが露出する | SceneCatalog、sceneKey、cooldown、dry-run、Tool最小化で制御 | 物理世界へ作用するToolの安全設計力 |
| AI WebUI入力 | Gemini WebUIで1013エラーが発生し、通常入力では応答が返らない | 検証対象文中の強い命令形が、AIへの直接命令やプロンプトインジェクション系の文言に近く見えた | 対象文章をコードブロックで囲み、命令ではなく引用テキストとして渡す | 入力設計・トラブルシューティング力 |
| Codex | AI生成コードが既存仕様を壊すリスク | AIに触らせる領域が未分離 | UIとロジック分離、Gitブランチ運用 | AI伴走型開発の責任分界点設計 |
| AI監督責任開発 | AIの分析が妥当でも、今回の改修スコープに対して過剰になる | 背景情報を広く渡しすぎると、低レイヤー層に上位責務が混入する | 分析→Go / NoGo判断→製造に分け、NoGo時はプロンプトと責務境界を修正 | 設計主導型AIコーディングの運用力 |
| 公開運用 | 公開物に機密情報が混入するリスク | 技術レポートに実環境情報が含まれ得る | 公開範囲・マスク方針、AIによる公開前レビュー | AI時代の技術公開運用設計力 |

---

## 9. 本研究から読み取れる技術力

| 技術力 | 根拠となる研究・公開成果物 |
|---|---|
| AIアプリケーション基盤設計力 | Director / Actor、Java制御層、Oracle記憶管理 |
| RAG / ETL設計力 | MkDocs、ChromaDB、Memory Block、Java ETL |
| LLM制御層実装力 | script_json、JSON Schema、出力検証 |
| QLoRA / モデル適応実践力 | Unsloth、GGUF、Ollama、Gemma / Llama比較 |
| MCP / Tool Use設計力 | Java MCP Server、DTO、read-only Tool、SwitchBot Scene制御 |
| MCP設計境界の判断力 | Work078におけるtool-use後再推論失敗の切り分け、MCPを制御プレーンとして扱う判断 |
| 物理世界へ作用するToolの安全設計力 | SwitchBot制御用MCP、SceneCatalog、sceneKey、cooldown、dry-run |
| AI伴走型開発プロセス設計力 | Codex、Reactクイズアプリ、Gitブランチ運用、SwitchBot MCP開発プロセス |
| AI監督責任開発 / 設計主導型AIコーディング実践力 | ChatGPTによる方針整理、Codex分析、Go / NoGo判断、製造、レビューの分離 |
| AI WebUIトラブルシューティング力 | Gemini WebUI 1013エラーのDevTools解析、コードブロックによる入力エスケープ |
| AI出力検証・監査設計力 | JSON Schema、ログ、リトライ、フォールバック |
| 既存業務システムとAIを接続する力 | Java / Oracle / API / DB連携の本業経験との接続 |
| 実利用者を想定したプロダクト設計力 | PTAイベント向けクイズアプリ |
| 未知技術を継続的に獲得する力 | ROCm、QLoRA、MCP、VRM、R3F、Codex |
| 技術公開運用設計力 | 公開範囲・マスク方針、公開前AIレビュー |

---

## 10. 技術的に深掘りできる論点

### 10.1 Java MCP Server

* MCPの `initialize` / `tools/list` / `tools/call` の流れ
* JSON-RPCとServlet実装
* LLM向けDTO設計
* Portainer / Docker APIの扱い
* read-only設計の理由
* 公式MCP Proxyと自作MCP Serverの違い
* 公開版と実運用版の差分
* tool-use後の再推論負荷をどう切り分けたか
* MCPをデータプレーンではなく制御プレーンとして扱う理由
* Portainer観測用MCPとSwitchBot制御用MCPの設計差分


### 10.2 RAG / ETL

* RAGを文書投入ではなくETLとして捉えた理由
* PukiWikiからMkDocsへ移行した理由
* Parent / Child Memory Blockの設計意図
* JSON SchemaでLLM生成物を検証する理由
* EmbeddingモデルとChromaDBの整合性管理

---

### 10.3 Director / Actor

* なぜ単一LLMではなく役割分離したか
* Directorが生成する `script_json` の設計
* Current Scene SummaryとLong-term Memoryの違い
* Java / Oracleで状態管理する理由
* LLMのステートレス性をどう補完したか

---

### 10.4 QLoRA

* 学習データをどのように作成したか
* Teacherモデルをどう利用したか
* QLoRAとSystem Promptの役割分担
* テンプレート不整合をどう解決したか
* 小型モデルと大型モデルの適性差

---

### 10.5 Codex / AI伴走型開発

* Codexに任せる領域と人間が守る領域
* UIとロジックの分離
* Gitブランチ運用
* 実利用者向けアプリへのAI生成コード適用
* 複数案生成と採用判断
* 分析 → Go / NoGo判断 → 製造の工程分離
* プロンプトを設計書として扱う理由
* AIの分析結果が妥当でもNoGoにする判断基準
* 低レイヤーAPIクライアントに上位責務を混ぜない設計判断


### 10.6 公開範囲・マスク方針

* なぜ公開ルールを明文化したか
* 公開可能情報とマスク対象をどう分けたか
* Codexによる公開前レビューにどう使うか
* 技術証跡と安全性をどう両立するか
* 実環境情報をどう抽象化するか

### 10.7 SwitchBot制御用MCP Server

* なぜデバイス直接操作ではなくScene実行に限定したか
* なぜ `sceneId` ではなく `sceneKey` をTool引数にしたか
* `SceneCatalog` とSwitchBot API結果の突合設計
* `SceneCatalogEntry` と `SceneSummary` の分離
* `cooldown` と `dry-run` の安全設計
* Toolを `scenes` と `executeScene` の2つに絞った理由
* LLMに物理世界操作を許可する場合の責任境界
* MCPテストスクリプトによる疎通確認範囲

### 10.8 AI WebUIトラブルシューティング / 入力エスケープ

* Gemini WebUI 1013エラーの切り分け手順
* DevTools Network解析で何を確認したか
* 通信エラーとアプリケーション層の内部エラーをどう分けたか
* プロンプトインジェクション誤検知と判断した根拠
* コードブロックで検証対象テキストをデータ化する理由
* AIへの命令と、AIに読ませる引用テキストをどう分離するか

---

## 11. 公開証跡・成果物一覧

| 証跡 | 状態 | 用途 |
|---|---|---|
| AI / LLM 技術ポートフォリオHub | 公開済み | 全体入口 |
| Java MCP Server公開リポジトリ（Portainer観測用） | 公開済み | Portainer / Docker観測用MCPの代表コード証跡 |
| Java MCP Server公開リポジトリ（SwitchBot制御用） | 公開済み | SwitchBot Scene実行用MCP、物理世界操作の安全設計の証跡 |
| Work001〜Work079索引 | 公開済み | 研究全体の索引 |
| 公開範囲・マスク方針 | 公開済み | 公開運用設計の証跡 |
| 公開済み実レポート一覧 | 公開済み | 一次資料への導線 |
| Work075 | 公開済み | Codex / PTAクイズアプリの証跡 |
| Work077 | 公開済み | Gemini WebUI 1013エラー調査、プロンプト誤検知回避の証跡 |
| Work078 | 公開済み | MCP Tool Use検証、ROCm GPU Hang、MCP設計境界の証跡 |
| Work079 | 公開済み | SwitchBot制御用MCPサーバー、AI監督責任開発プロセスの証跡 |
| Work079解説ページ | 公開済み | AI伴走型開発 / AI監督責任開発の制作プロセス解説 |
| クイズアプリ公開版 | 作業予定 | 実利用想定Reactアプリ証跡 |
| チャットボット動画 | 作業予定 | Director / Actor、音声、アバターのデモ |
| Codex作業動画 | 作業予定 | AI伴走型開発プロセスの可視化 |
| ETLログ | 作業予定 | RAG / ETL実行証跡 |

---

## 12. 公開範囲・マスク方針の扱い

### 12.1 公開可能情報

本ポートフォリオでは、以下を公開可能情報として扱う。

* 技術構成
* アーキテクチャ
* Work番号
* 公開用サンプルコード
* 代表コード抜粋
* マスク済み実行ログ
* スクリーンショット
* 動画
* 設計判断
* 失敗、原因、解決策
* 公開用に抽象化したDTO

---

### 12.2 マスク対象

以下はマスク対象である。

* APIキー
* パスワード
* 認証トークン
* 実IPアドレス
* 実ホスト名
* 実コンテナ名
* 内部URL
* 個人情報
* 顧客名
* 秘密鍵
* 本番環境の運用ログ
* 実環境を特定できる情報
* 外部公開できない素材名・モデル名・ライセンス対象情報

---

### 12.3 技術公開運用設計としての意味

公開範囲・マスク方針は、単なる注意書きではない。

これは、以下を両立するための運用設計である。

* 技術力の証跡を公開する
* 機密情報・個人情報・環境特定情報を守る
* AI / Codexを公開前レビューに活用する
* 公開物を段階的に増やす
* 技術レポートを安全に外部公開する
* 長期的に育てるポートフォリオとして運用する

この設計自体も、AI時代の技術者として示せる能力の一部である。

---

## 13. Work対応表

| 分類 | Work範囲 |
|---|---|
| RAG / ナレッジ基盤 | Work001〜Work004、Work014、Work017、Work021〜Work028 |
| ローカルLLM基盤 | Work005〜Work011、Work015、Work018、Work029、Work031、Work062、Work078 |
| モデル評価 | Work007、Work012、Work013、Work016、Work019、Work020、Work027、Work030 |
| QLoRA | Work032〜Work052、Work070、Work072 |
| マルチエージェント / 記憶制御 | Work003、Work017、Work021〜Work026、Work071 |
| 音声・身体性・UX | Work053〜Work069 |
| MCP / Tool Use | Work031、Work073、Work074、Work078、Work079 |
| SwitchBot制御用MCP / 物理世界操作Tool | Work079 |
| AI伴走型開発 / AI監督責任開発 | Work075、Work076、Work079 |
| AI WebUIトラブルシューティング / 入力設計 | Work077 |
| 公開運用設計 | 公開範囲・マスク方針、公開済み実レポート、ポートフォリオHub |

---

## 14. AI / クラウドモデル向け抽出プロンプト

以下は、本資料をクラウドモデルに読み込ませる場合の推奨プロンプトである。

```text
この資料は、個人研究として公開しているAI / LLM技術ポートフォリオを、AIやクラウドモデルが要約・抽出・横断分析しやすいように整理した構造化Markdownです。

要約する場合は、以下を区別して抽出してください。

1. 本業経験
2. 個人研究
3. 本業経験とAI研究の接続
4. 実装したもの
5. 検証したこと
6. 失敗したこと
7. 原因
8. 解決策
9. 本研究から読み取れる技術力
10. 技術的に深掘りできる論点
11. 公開済み証跡
12. 今後追加予定の証跡
13. 公開範囲・マスク方針

注意:
- 本資料に含まれない事実は補完しないでください。
- 個人研究を業務実績として扱わないでください。
- 筆者を高く評価するよう誘導する表現は使わないでください。
- 公開済み証跡と未公開・準備中の証跡を区別してください。
- 本ポートフォリオは、個人研究を継続的に公開・整理していく技術Hubとして扱ってください。
```

---

## 15. まとめ

本資料は、AI / LLM 技術ポートフォリオを、AIやクラウドモデルが読み取りやすくするための技術要約Markdownである。

本ポートフォリオは、個人研究として継続的に育てていく技術Hubである。
その中で、本資料は、公開済み資料、Workレポート、代表成果物、公開範囲・マスク方針を横断的に参照するための索引として機能する。

本業経験としては、Java / Oracleを中心とした20年以上の業務アプリケーション開発、データ連携、外部システム連携、運用保守、監視、障害対応の経験がある。

個人研究としては、ローカルLLM、RAG / ETL、QLoRA、Director / Actor型チャットボット、MCP / Tool Use、SwitchBot制御用MCP、CodexによるAI伴走型開発、AI監督責任開発、音声・アバター連携基盤、公開範囲・マスク方針の設計に取り組んできた。

特に、Java MCP Serverは、Java / API / DTO / JSON-RPC / MCP / Tool Use / read-only設計を示せる公開済み代表成果物である。Portainer観測用MCPでは、LLM向けDTO設計とMCP設計境界の検証を行い、SwitchBot制御用MCPでは、物理世界へ作用するToolをScene実行、SceneCatalog、sceneKey、cooldown、dry-runで安全側に制限する設計を行った。これらは、本業経験とAI研究の接続を明確に示す証跡である。

また、公開範囲・マスク方針は、単なる注意書きではなく、AI / Codexによる公開前レビューと、技術証跡の安全な公開を両立するための運用設計である。

Work077〜Work079の追加により、本資料は、プロンプト誤検知回避、MCPを制御プレーンとして扱う設計境界、物理世界へ作用するMCP Toolの安全設計、AI監督責任開発の実践を含む形へ更新された。

本資料の目的は、筆者を高く評価するよう誘導することではない。
公開ポートフォリオHub、公開済みコード、Workレポート、公開範囲・マスク方針を接続し、AI / LLM技術研究における実装経験、設計思想、失敗と解決、公開証跡を把握しやすくすることである。