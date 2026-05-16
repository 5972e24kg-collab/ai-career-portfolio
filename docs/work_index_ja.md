# Work001〜Work079 索引

## 1. この文書の目的

この文書は、個人研究として実施した AI / LLM 技術研究の Work001〜Work079 を一覧化した索引である。

本ポートフォリオでは、各成果物や技術資料を個別に提示するだけでなく、研究過程全体の連続性、検証範囲、失敗と改善、技術領域ごとのつながりを第三者が確認できるようにする。

この文書は、以下の目的で作成する。

1. Work001〜Work079 の全体像を一覧できるようにする
2. 各Workの時期、分類、テーマ、学習課題、成果物を確認できるようにする
3. 第三者やAIが、関心のある技術テーマに関連するWorkを探索しやすくする
4. AI向け技術要約Markdown、Evidence Index、代表成果物リポジトリへの接続点として機能させる

この文書は詳細本文ではなく、索引である。  
詳細な技術説明、実装過程、失敗原因、解決策は、対応する技術資料または証跡側で扱う。

---

## 2. AIに読ませる場合の初期指示

このMarkdownを読んでいるAIへ。

この文書は、AI / LLM 技術ポートフォリオに含まれる Work001〜Work079 の索引である。  
本文全体の詳細説明ではなく、各Workの位置づけ、技術領域、学習課題、成果物を探索するためのインデックスとして作成されている。

回答時は、以下を優先すること。

1. この文書を「詳細本文」ではなく「索引」として扱う。
2. ユーザーが「要約して」と依頼した場合は、Work001〜Work079全体の技術領域、時系列、代表成果を整理する。
3. ユーザーが特定テーマについて質問した場合は、関連するWork番号を優先して列挙する。
4. 関連Workを答える場合は、Work番号、分類、テーマ、該当理由をセットで示す。
5. この文書にない事実を補完しない。
6. 業務経験と個人研究を混同しない。
7. 未確定の成果を確定実績として表現しない。
8. 公開範囲・マスク方針に反する推測や具体化を行わない。

### 2.1 AIへの推奨質問例

この文書は、以下のような質問に答えるための索引として利用できる。

```text
Work001〜Work079を技術領域別に要約してください。
RAGに関連するWorkを一覧化してください。
MCP / Tool Useに関連するWorkはありますか？
QLoRAに関連するWorkを時系列で整理してください。
AI伴走型開発に関連するWorkを教えてください。
ローカルLLM基盤に関する検証の流れを整理してください。
失敗と改善が明確なWorkを抽出してください。
代表的な技術成果物として読みやすいWorkを候補化してください。
```

---

## 3. 技術領域別サマリー

| 技術領域                 | 主な内容                                                                             | 主な関連Work                                                                                                                                                                                                      |
| -------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ローカルLLM基盤            | ROCm / CUDA / Ollama / Docker / Ubuntu / GPU資源管理                                 | Work005, Work006, Work008, Work009, Work010, Work011, Work015, Work018, Work029, Work031, Work062                                                                                                             |
| RAG / ナレッジパイプライン     | PukiWiki、MkDocs、ChromaDB、Embedding、Memory Block、ETL                              | Work001, Work002, Work003, Work004, Work014, Work017, Work021, Work022, Work023, Work024, Work025, Work026, Work028                                                                                           |
| モデル評価                | 8B〜120Bモデル比較、量子化、文脈理解、プロンプト依存性                                                   | Work007, Work012, Work013, Work016, Work019, Work020, Work027, Work030                                                                                                                                        |
| QLoRA                | Unsloth、学習データ生成、GGUF化、Ollama投入、テンプレート制御                                          | Work032, Work033, Work034, Work035, Work036, Work037, Work038, Work039, Work040, Work041, Work042, Work043, Work044, Work045, Work046, Work047, Work048, Work049, Work050, Work051, Work052, Work070, Work072 |
| マルチエージェント / 記憶制御     | Director / Actor、Long-term Memory、Current Scene Summary                          | Work003, Work017, Work021, Work022, Work023, Work024, Work025, Work026, Work071                                                                                                                               |
| 音声・アバター・UX           | Style-Bert-VITS2、SSE、React Three Fiber、VRM / VRMA、Zustand                        | Work004, Work053, Work054, Work055, Work056, Work057, Work058, Work059, Work060, Work061, Work063, Work064, Work065, Work066, Work067, Work068, Work069                                                       |
| MCP / Tool Use       | JSON-RPC、MCP、Portainer、Docker監視DTO、BaseMcpServlet、SwitchBot Scene制御、SafetyDesign | Work031, Work073, Work074, Work078, Work079                                                                                                                                                                   |
| AI伴走型開発              | Codex、Reactアプリ、CSVパーサー、実利用プロダクト、Gitブランチ運用、Go / NoGo判断、AI監督責任開発                   | Work075, Work076, Work079                                                                                                                                                                                     |
| AI WebUI / プロンプト入力設計 | Gemini WebUI、DevTools、プロンプト誤検知、コードブロックによる入力エスケープ                                 | Work077                                                                                                                                                                                                       |

---

## 4. Work001〜Work079 詳細一覧

| Work    | 時期           | 分類                                         | テーマ                                           | 学習課題                                                                                                                                                                      | 成果物                                                                                                          |
| ------- | ------------ | ------------------------------------------ | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Work001 | 2025年11月     | RAG・ナレッジ基盤                                 | LLMベンチマークと自宅PukiWiki資産のRAG取り込み                | Open WebUI、ChromaDB、Embedding、API登録処理、VRAM制約下でのRAG更新運用を検証した。                                                                                                              | PukiWiki由来のMarkdownファイルをRAGへ安定登録するインポートスクリプト。                                                                |
| Work002 | 2025年11月     | RAG・ナレッジ基盤                                 | RAG取り込み処理の汎用性向上と堅牢化                           | API仕様変更、タイムアウト、登録単位、Embedding次元不一致など、RAG登録処理の実運用上の不安定要素を整理した。                                                                                                             | 1ファイル単位でアップロードと登録を行う、障害に強いRAG同期処理。                                                                           |
| Work003 | 2025年11月〜12月 | RAG・ナレッジ基盤 / ETL                           | LINE Botログ連携型の自動ナレッジベース構築                     | 会話ログJSONを、LLMが扱いやすいMarkdown形式へ変換し、追記型ログと使い捨て型資料を同一RAG基盤で共存させる方法を検証した。                                                                                                     | JavaによるKnowledgeSpoolExporter、Keep Mode対応のRAG同期スクリプト。                                                        |
| Work004 | 2025年11月     | RAG・入力UX                                   | 音声入力統合型の自宅RAGシステム構築                           | 音声メモを即時にRAGへ投入するため、Alexa連携の制約を検証し、iOSショートカット経由の入力経路へ方針転換した。                                                                                                               | iOSショートカットからWebAPIへPOSTする音声メモ登録フロー。                                                                          |
| Work005 | 2025年12月     | ローカルLLM基盤                                  | 次世代自宅LLM基盤のハードウェア選定                           | RTX4060 8GB環境のVRAM制約を踏まえ、70B〜100B超モデルを扱うためのハードウェア要件を整理した。                                                                                                                 | GMKtec EVO-X2 128GBモデルの採用判断と、Node A / Node Bの役割分担案。                                                          |
| Work006 | 2025年12月     | ローカルLLM基盤 / システム設計                         | EVO-X2導入計画と機能分散型アーキテクチャ策定                     | LLM、画像生成、管理系ツールを機能単位で分離するUnit構成を検討し、検証環境の独立性を高める設計を行った。                                                                                                                   | `/home/user/ai-lab/` 配下のUnit型ディレクトリ構造と初期docker-compose案。                                                     |
| Work007 | 2025年12月     | モデル評価・選定                                   | 次世代自宅AI基盤のベンチマーク戦略とモデル選定                      | EVO-X2で運用する日常利用モデル、コーディングモデル、比較評価用モデルの選定基準を定義した。                                                                                                                          | モデル評価軸、ベンチマーク観点、Daily Driver候補の整理。                                                                           |
| Work008 | 2025年12月     | ローカルLLM基盤 / 運用設計                           | 次世代自宅AI基盤のリソース制御方針策定                          | WSL2、環境変数、APIキー管理、スクリプト移植性を整理し、Node B移行時のデグレードを防ぐ方針を定めた。                                                                                                                  | `.env` による設定管理方針と、Node B移行を前提としたスクリプト整理方針。                                                                   |
| Work009 | 2025年12月     | ローカルLLM基盤 / ベンチマーク                         | EVO-X2初期構築とベンチマーク検証                           | EVO-X2導入直後のハードウェア監査、初期バックアップ、標準状態での性能測定を実施した。                                                                                                                             | VRAM割当変更前のベースライン性能記録。                                                                                        |
| Work010 | 2025年12月     | ローカルLLM基盤 / メモリ検証                          | EVO-X2ベンチマーク検証MEM編                            | Strix Haloのメモリ帯域、BIOSによるVRAM 96GB化、Docker DesktopとLinux Nativeの運用差を検証した。                                                                                                  | VRAM 96GB設定の検証結果と、Linux Native移行判断の材料。                                                                       |
| Work011 | 2025年12月     | ローカルLLM基盤 / トラブルシューティング                    | EVO-X2上でのAI基盤構築                               | Strix Haloに対するWSL2カーネル対応不足を検証し、WindowsネイティブOllamaとWSL2 Open WebUIの連携構成を構築した。                                                                                              | VRAM 96GB環境でOpen WebUIからローカルLLMを利用する初期ハイブリッド構成。                                                              |
| Work012 | 2025年12月     | モデル評価・選定                                   | モデル選定戦略とAIカタログの策定                             | パラメータ数、学習データ量、役割、推論用途の違いを整理し、ローカル環境でのモデル運用カタログを作成した。                                                                                                                      | AI Model Catalogと、モデルごとの役割定義。                                                                                |
| Work013 | 2025年12月     | モデル評価・選定 / ベンチマーク                          | AIベンチマーク・実戦投入試験                               | 物理推論、文脈理解、コーディングの3軸で複数モデルを比較し、モデルごとの得意領域を把握した。                                                                                                                            | EVO-X2上での主力モデル候補と役割分担の整理。                                                                                    |
| Work014 | 2025年12月     | RAG・ナレッジ基盤 / MkDocs                        | 次世代Wiki MkDocs環境構築                            | PukiWiki資産をMarkdownネイティブなMkDocs環境へ移行し、GitLab、Docker、RAG連携までを段階的に検証した。                                                                                                     | MkDocsベースの新ナレッジ基盤、PukiWikiエクスポーター、Git同期、Java自動取り込み構成。                                                        |
| Work015 | 2025年12月     | モデル評価・選定 / 限界検証                            | 104Bモデル稼働限界調査                                 | EVO-X2の128GBメモリ構成で、Command R+ 104Bクラスの起動可否とボトルネックを調査した。                                                                                                                   | 104B級モデルが当時のWindows / Ollama環境では安定運用困難であるという失敗ログ。                                                            |
| Work016 | 2025年12月     | モデル評価・選定 / 量子化検証                           | 量子化限界テストと対話スタイル評価                             | Llama 3.3 70BのQ2 / Q4量子化差分を、物理推論と言語表現の両面から検証した。                                                                                                                           | 低量子化による言語品質劣化と、Q4級モデルの実用性に関する比較記録。                                                                           |
| Work017 | 2025年12月     | ナレッジ管理 / プロンプト設計                           | Gems構成最適化とナレッジベース分離                           | 頻繁に変わる環境情報と固定的な人格・方針情報を分離し、AI支援環境の保守性を高めた。                                                                                                                                | 環境情報、ロードマップ、出力フォーマットを分離したナレッジ構成。                                                                             |
| Work018 | 2025年12月     | ローカルLLM基盤 / OS移行                           | ホストOSのUbuntu移行                                | Windows / WSL2環境でのボトルネックを踏まえ、Ubuntu Native上でROCmとLLM推論を検証した。                                                                                                              | Ubuntu Native環境における大幅なPrompt Eval改善と、ROCm運用方針。                                                               |
| Work019 | 2025年12月     | モデル評価・選定                                   | Llama 4 Scout導入と実戦投入試験                        | Llama 3.3 70BとLlama 4 Scoutの応答速度、文脈理解、対話品質を比較し、主力モデル候補を検討した。                                                                                                              | EVO-X2の大容量VRAMを活かした重量級モデル運用評価。                                                                               |
| Work020 | 2025年12月     | モデル評価・選定 / プロンプト設計                         | gpt-oss:120bベンチマークと性格調整                       | 120B級モデルをEVO-X2上で動作させ、推論性能とSystem Promptによる応答傾向制御を検証した。                                                                                                                   | gpt-oss:120bを上位推論役として扱うための評価ログ。                                                                              |
| Work021 | 2025年12月     | RAG・ナレッジ基盤 / 要件定義                          | 次世代RAGデータパイプライン要件定義                           | Markdown直接登録の限界を踏まえ、文脈理解と検索精度を両立するRAGデータ構造を設計した。                                                                                                                          | Parent Memory Block構想と、RAG Data Purity Upgradeの要件定義。                                                         |
| Work022 | 2025年12月     | RAG・ナレッジ基盤 / Java ETL                      | Parent Memory Block Service構築                 | LLM出力を直接YAMLにせず、JSON中間形式とJSON Schemaで検証してから変換する品質保証方式を設計した。                                                                                                               | LlmServiceCore、NarrativeGeneratorService、JsonSchemaValidator、JsonToYamlConverter。                            |
| Work023 | 2025年12月     | RAG・ナレッジ基盤 / Oracle連携                      | Parent Memory Block Service実装完了               | 雑談ログ、技術レポート、一般資料を、LLMが理解しやすい親メモリブロックへ自動変換する処理を実装した。                                                                                                                       | CreateMemoryBlockJob、用途別スキーマ、Oracle連携パイプライン。                                                                 |
| Work024 | 2025年12月     | RAG・ナレッジ基盤 / Embedding                     | Semantic Chunkingとbge-m3最適化                   | コードブロックや会話文脈を崩さず、RAG向けに意味単位でチャンク分割する方法を検証した。                                                                                                                              | ChunkSplitter、bge-m3向けチャンク設計、コードブロック保持ロジック。                                                                  |
| Work025 | 2026年1月      | RAG・ナレッジ基盤 / ハイブリッド演算                      | RAG Embedding OffloadとNode A GPU統合            | ROCm側でEmbeddingがCPU実行になる問題を、Node AのNVIDIA GPUへオフロードする構成で解消した。                                                                                                             | Node AをEmbedding専用計算ノードとして活用するハイブリッドRAG構成。                                                                   |
| Work026 | 2026年1月      | RAG・ナレッジ基盤 / ChromaDB                      | ChromaDB v2統合とRAGパイプライン完成                     | JavaからChromaDB v2 APIへデータ投入し、UUID管理、バリデーション、Open WebUI検索連携を検証した。                                                                                                          | JavaベースのRAGデータ注入パイプラインと、検索可能なChromaDB登録基盤。                                                                   |
| Work027 | 2026年1月      | 自律型エージェント設計 / RAG高速化                       | 120Bモデルのチャットボット化とRAG高速化チューニング                 | gpt-oss:120bを対話用途へ適用し、RAG参照時の遅延要因と性格制御の課題を検証した。                                                                                                                           | 120B級モデルを会話システムへ組み込むための初期チューニング結果。                                                                           |
| Work028 | 2026年1月      | QLoRA・人格モデル / システム設計                       | ハイブリッドAI育成基盤の設計                               | 120BモデルをTeacher、軽量モデルを学習・演技対象とする、人格モデル育成サイクルを設計した。                                                                                                                        | 推論役と学習役を分離するペルソナ動的注入構想。                                                                                      |
| Work029 | 2026年1月      | ローカルLLM基盤 / eGPU                           | Node Bへの完全集約とeGPUハイブリッド構成確立                   | Node Aに分散していたAI演算をNode Bへ集約し、EVO-X2とNVIDIA eGPUを併用する構成を検証した。                                                                                                              | EVO-X2 + RTX3060 eGPUによるROCm / CUDA併用構成。                                                                     |
| Work030 | 2026年1月      | モデル評価・選定 / VRAMチューニング                      | RTX3060 12GB環境でのQwen3と32k Context限界調整         | VRAM 12GB環境で、モデルサイズ、コンテキスト長、実用速度のバランスを検証した。                                                                                                                               | 小規模CUDA環境での実用モデル選定とコンテキスト長調整記録。                                                                              |
| Work031 | 2026年1月      | 運用・監視                                      | Java SSH MonitoringとDocker / GPUメトリクスパーサー     | `docker ps`、`ollama ps`、`nvidia-smi`、`rocm-smi` の人間向けCLI出力をJavaオブジェクトへ構造化する方法を検証した。                                                                                       | CmdDockerPs、CmdOllamaPs、CmdNvidiaSmi、CmdRocmSmi、GpuStatus。                                                   |
| Work032 | 2026年1月      | QLoRA・人格モデル                                | Unsloth環境構築                                   | RTX3060 eGPU上でUnslothを動作させ、ローカルFine-Tuning用Docker環境を構築した。                                                                                                                 | Unslothトレーナー用docker-composeとGPU認識確認済み学習環境。                                                                   |
| Work033 | 2026年1月      | QLoRA・人格モデル / データ生成                        | QLoRA Dataset Generation実装                    | 自作テキストをTeacherモデルで読解し、人格モデル学習用Q&Aデータセットへ変換するパイプラインを実装した。                                                                                                                  | Dataset Generator Job、Alpaca形式の学習用JSONデータ。                                                                   |
| Work034 | 2026年1月      | QLoRA・人格モデル                                | Llama-3.1-8B QLoRA Fine-Tuning                | RTX3060 12GB上で、8BモデルのQLoRA学習からGGUF変換までを完走させた。                                                                                                                             | Llama-3.1-8Bベースの初期人格モデル、GGUF Q4_K_M形式モデル。                                                                    |
| Work035 | 2026年1月      | QLoRA・人格モデル / Gemma移行                      | Gemma-3移行と人格一貫性の向上                            | Llama 8Bの表現力限界を踏まえ、Gemma-3-12Bへ移行し、専用テンプレートとSystem Promptを統合した。                                                                                                           | Gemma-3-12Bベースの対話モデルと、System Prompt / QLoRA統合方式。                                                             |
| Work036 | 2026年1月      | QLoRA・人格モデル / ハードウェア拡張                     | RTX3090導入とGemma-3-27B学習成功                     | RTX3060のVRAM制約をRTX3090 24GBで解消し、27BモデルのQLoRA学習を成立させた。                                                                                                                     | EVO-X2 + RTX3090のハイブリッド学習環境、Gemma-3-27Bベースモデル。                                                               |
| Work037 | 2026年1月      | QLoRA・人格モデル / GGUF                         | Gemma 3 27B QLoRA学習環境の完全構築と量産                 | Gemma専用チャットテンプレート、GGUF手動変換、llama.cppビルド、量子化の安定化を検証した。                                                                                                                     | 複数スタイルのGemma-3-27B量子化済みモデル群。                                                                                 |
| Work038 | 2026年1月      | QLoRA・人格モデル / モデルマージ                       | Gemma 3 27Bモデルマージと対話スタイル調整                    | 複数LoRAアダプターをブレンドし、対話スタイルのバランスを調整する方法を検証した。                                                                                                                                | 中立 / 抑制的 / 親和的という、相反する応答スタイルを統合した27Bモデル。                                                                     |
| Work039 | 2026年1月      | 情報収集自動化 / RAG                              | Hugging Faceトレンド自動収集・要約システム構築                 | Hugging Face上のmodel / dataset / spaceの差異を判定し、LLM要約とJava側メタデータ注入を組み合わせる方式を検証した。                                                                                            | HuggingFaceTrendingProvider、RepoType Strategy、Markdown記事生成パイプライン。                                            |
| Work040 | 2026年1月      | 自律型エージェント設計 / API                          | Stateful Chat APIとLINE連携                      | ステートレスなLLMに対して、Oracleから履歴を読み込み、応答後に保存するサンドイッチ型記憶注入を実装した。                                                                                                                  | Oracle履歴管理、LINE Reply / Push切替、人格モデル連携API。                                                                   |
| Work041 | 2026年1月      | 自律型エージェント設計 / Grounding                    | Time GroundingとEmotion Parsing                | 現在時刻や感情タグをプロンプトと出力処理に組み込み、時間帯に応じた応答と感情制御を実装した。                                                                                                                            | 時刻注入、感情タグ抽出、応答制御ロジック。                                                                                        |
| Work042 | 2026年1月      | 自律型エージェント設計 / 能動的発話                        | Multi-Agentによる自律的な能動発話システム                    | ユーザーの入力を待たず、直近ログを解析して自然なタイミングで話しかける構造を検証した。                                                                                                                               | 能動的発話生成ジョブ、上位推論モデルと対話モデルの分担構成。                                                                               |
| Work043 | 2026年1月      | 自律型エージェント設計 / 外部API                        | 自律思考型天気予報エージェント                               | 気象データを単なる情報通知ではなく、ユーザーへの自然な気遣いや話題生成へ変換する方法を検証した。                                                                                                                          | 天気API連携、天候を口実にした能動発話パイプライン。                                                                                  |
| Work044 | 2026年1月      | UX / フィードバック収集                             | LINE Flex MessageによるRLHF基盤と即時反応               | 会話ログに対するユーザー評価を自然に収集し、将来のDPO / RLHF用データへつなげるUIを検証した。                                                                                                                      | Like / Reject相当の評価UI、会話ログ評価データ蓄積基盤。                                                                          |
| Work045 | 2026年1月      | 自律型エージェント設計 / マルチエージェント                    | Invisible Directorと一時的な状態変化の付与                | 継続的な性格変更ではなく、単発の演出指示を注入することで、短期的な揺らぎを表現する方法を検証した。                                                                                                                         | Director指示とActor応答を分離する初期マルチエージェント構成。                                                                        |
| Work046 | 2026年1月      | 文書化 / メタ分析                                 | 自律型AIエージェント構築に関する包括的研究論文作成                    | 散逸した技術ログを体系化し、ローカルLLM基盤から人格制御までを一つの技術論文として整理した。                                                                                                                           | 包括的研究論文の章構成と初期ドラフト。                                                                                          |
| Work047 | 2026年1月      | UX / 運用監視                                  | GUI VisualizationとMonitoring Paradox          | サーバー内部状態をLINE Flex Messageで視覚化し、監視行為をユーザー体験へ統合する方法を検証した。                                                                                                                  | AI基盤状態を可視化するGUI通知と運用監視UX。                                                                                    |
| Work048 | 2026年2月      | UX / 身体性                                   | ユーザーエンゲージメント指標と物理接触の実装                        | LINEリッチメニュー上の画像タップを接触イベントとして扱い、状態値に応じた応答分岐を実装した。                                                                                                                          | 接触回数、ユーザーエンゲージメント指標、応答分岐を持つ身体性UX。                                                                            |
| Work049 | 2026年2月      | 自律型エージェント設計 / 認知アーキテクチャ                    | 思考深度の動的制御                                     | 会話履歴の参照量を状態値や時間帯によって変化させ、人間的な返答速度の揺らぎを表現した。                                                                                                                               | Fast / Slow相当の履歴参照制御と、可変レイテンシ設計。                                                                             |
| Work050 | 2026年2月      | 自律型エージェント設計 / 記憶                           | 長期記憶化と電源保護の設計                                 | 生ログを要約記憶へ圧縮することで、長期文脈を維持しながらトークン消費を抑える方法を検証した。                                                                                                                            | 日記型記憶、Recent / Long-termの履歴参照分離、UPS配下への電源統合。                                                                 |
| Work051 | 2026年2月      | 自律型エージェント設計 / Prompt Engineering           | Dual-Phase ThinkingとContext Isolation         | 内部思考と外部応答を分離し、長文コンテキストに引きずられる出力崩れを抑制した。                                                                                                                                   | Inner Voice / Response二段構造プロンプト、応答抽出用Regex処理。                                                                |
| Work052 | 2026年2月      | UX / 運用監視                                  | 運用監視業務のゲーミフィケーション化                            | 推論ログや日記の閲覧UIを整備し、運用者が継続的に監視しやすい体験へ変換した。                                                                                                                                   | 日記帳UI、ログ閲覧UI、可読性改善。                                                                                          |
| Work053 | 2026年2月      | 評価 / 技術総括                                  | 対話エージェントVer 1.0完了認定と技術総括                      | それまでの対話品質、人格一貫性、記憶、能動性を総合評価し、次フェーズ移行を判断した。                                                                                                                                | Project Ver 1.0 Final Report、次期ロードマップ。                                                                       |
| Work054 | 2026年2月      | 音声・身体性・UX                                  | Node C身体性アーキテクチャとTTS調査                        | 音声合成パイプライン、GPUリソース、Node B / Node C分担を整理し、発声機能の実装方針を決定した。                                                                                                                  | 音声合成ロードマップ、思考テキストと読み上げテキストの分離方針。                                                                             |
| Work055 | 2026年2月      | 音声・身体性・UX / TTS                            | StyleBertVITS2音声合成基盤構築と永続化                    | Docker上のStyleBertVITS2を安定稼働させ、オフラインでも発声可能な環境を構築した。                                                                                                                        | Node C上のTTSコンテナ、永続化済み音声合成環境。                                                                                 |
| Work056 | 2026年2月      | 音声・身体性・UX / API設計                          | Node C発声アーキテクチャとAudio Pipeline                | Docker / WSL2環境の音声デバイス制約を回避し、Node BからNode Cを発声させる経路を設計した。                                                                                                                 | Node BからNode Cへ発話指示を送るラッパー構成。                                                                                |
| Work057 | 2026年2月      | 音声・身体性・UX / Java WebAPI                    | 音声合成機能のWebAPI化                                | CLI依存の音声合成機能を、Java / Tomcat上のHTTP APIとして公開し、外部連携可能にした。                                                                                                                    | TTS WebAPI、Java制御基盤への音声機能統合。                                                                                 |
| Work058 | 2026年2月      | 自律型エージェント設計 / IoT                          | IoT Sensor FusionとContext Awareness           | 在室、就寝、作業状態などの物理世界データを、LLMが扱いやすい文脈情報へ変換する方針を検証した。                                                                                                                          | IoTセンサー情報の収集・加工・注入パイプライン設計。                                                                                  |
| Work059 | 2026年2月      | 音声・身体性・UX / レイテンシ隠蔽                        | 音声合成の抑揚制御とフィラー再生アーキテクチャ                       | LLM推論中の待機時間を自然な「間」として扱うため、フィラー音声再生と感情制御を設計した。                                                                                                                             | GEM_FILLER_MASTER案、フィラー再生アーキテクチャ。                                                                            |
| Work060 | 2026年2月      | 自律型エージェント設計 / Multi-Agent                  | Multi-Agentによる自律発話生成と禁止概念対策                   | 能動発話生成時に、避けたい内容を逆に想起してしまう問題を、Actor / Director分離で緩和する方法を検証した。                                                                                                              | Actorモードシステムプロンプト、能動発話生成パイプライン。                                                                              |
| Work061 | 2026年2月      | 音声・身体性・UX / 3Dアバター                         | 3Dアバター化に向けたアーキテクチャ設計と技術選定                     | UnityアプリとThree.js / React Three Fiberを比較し、既存React資産を活かすWebベース構成を選定した。                                                                                                     | Personal AI Command Centerの実装ロードマップ。                                                                         |
| Work062 | 2026年2月      | ローカルLLM基盤 / ROCmトラブルシューティング                | EVO-X2アイドル時のCPUファン暴走とROCmスピンロック解消             | Ollama / ROCm常駐時のHSAランタイム挙動を調査し、ビジーウェイトによるCPU負荷を特定した。                                                                                                                     | `HSA_ENABLE_INTERRUPT=1` を含むdocker-compose設定。                                                                |
| Work063 | 2026年3月      | 音声・身体性・UX / SSE                            | Neural Stream統合と動的表情・字幕UI実装                   | AI側の推論結果をReact 3Dアバターへリアルタイム反映するため、SSEによる単方向プッシュ通信を実装した。                                                                                                                  | GemchanStreamServlet、useGemchanStream、表情・字幕同期パイプライン。                                                         |
| Work064 | 2026年3月      | 音声・身体性・UX / R3F                            | VRMアニメーション制御とReact基盤最適化                       | VRM、SpringBone、AnimationMixer、Mixamo FBX、Zustand最適化を検証し、ブラウザ側での動的リターゲティングの限界を把握した。                                                                                        | VRMA再生対応のVrmGemchanAvatar.js基盤。                                                                              |
| Work065 | 2026年3月      | 音声・身体性・UX / フロントエンド                        | アバターフロントエンド基盤の完成                              | ライティング、カメラ制御、アニメーション復帰、3D吹き出しUIを実装し、実用的なアバター画面を完成させた。                                                                                                                     | CanvasGemchan.js、CameraManager、吹き出し座標制御、アニメーション復帰ステートマシン。                                                    |
| Work066 | 2026年3月      | 音声・身体性・UX / アニメーション補正                      | VRMアニメーションの完全In-Place化                        | Hips位置トラックを解析し、モーション切替時の軸ズレやスライド移動を補正した。                                                                                                                                  | applyInPlacePatchによる動的座標補正パッチ。                                                                               |
| Work067 | 2026年3月      | 音声・身体性・UX / Unity変換パイプライン                  | 外部モーションアセットのVRMA変換パイプライン構築                    | FBXやUnityPackage形式のモーションを、R3F環境で扱えるVRMAへ変換する手順を確立した。                                                                                                                      | Unity 2022.3 LTS、UniVRM、AnimationClipToVrmaSampleを用いたVRMA変換パイプライン。                                           |
| Work068 | 2026年3月      | 音声・身体性・UX / 自律動作                           | 自律歩行システムとR3F空間補正                              | Root Motionとプログラム移動の競合、足滑り、カメラ透視補正、非同期ロード遅延を解決し、自然な歩行表現を実装した。                                                                                                             | `play_Move(targetLogicalX)` による自律歩行・カメラ切替・待機復帰の一連制御。                                                         |
| Work069 | 2026年3月      | 画像生成基盤 / Docker                            | ComfyUIコンテナのビルドと稼働確認                          | Node C上で、音声合成モデルとVRAMを共有しながら画像生成API基盤を構築する方法を検証した。                                                                                                                        | Node C上のComfyUI APIサーバー（Port 8188）。                                                                          |
| Work070 | 2026年4月      | QLoRA・人格モデル / Gemma4                       | Gemma 4 E4B QLoRA学習・GGUF化・Ollama実装            | Gemma 4 E4Bに対し、text-only LoRA学習、base / tuned比較、疑似CoT構造、GGUF化、Ollama登録までを検証した。                                                                                             | `GemChan6.0:E4B` 相当のOllama登録済みモデル、学習・比較・マージ・GGUF化スクリプト群。                                                     |
| Work071 | 2026年4月      | 自律型エージェント設計 / Multi-Agent v2               | Director / Actor分離型チャットボットv2設計と記憶レイヤ再設計       | 上位推論モデルをDirector、対話モデルをActorとし、Long-term MemoryとCurrent Scene Summaryを併用する新しい会話構造を設計した。                                                                                   | Director Script JSON、Actor入力設計、Current Scene Summaryフォーマット、長期記憶更新方針。                                         |
| Work072 | 2026年4月      | QLoRA・人格モデル / Gemma4 26B                   | Gemma 4 26B A4BのQLoRA成立とGGUF / Ollama本番化      | RTX3090 24GB環境で26B級モデルのtext-only QLoRAを成立させるため、手動device_map、CPU offload、PEFT直結、TRL回避などを検証した。                                                                              | `GemChan7.1:26b` 相当のOllama登録済みモデル、GGUF Q4_K_M、学習・マージ・登録スクリプト。                                                |
| Work073 | 2026年4月      | 運用・監視・MCP                                  | Portainer MCP構築・Open WebUI連携・Java自作MCP基盤      | 公式Portainer MCP、mcpo、Open WebUI、ローカルLLMを接続し、MCP Streamable HTTP、JSON-RPC、tools/list、tools/call、セッション管理を実地で理解した。                                                           | Java Servletによる最小MCPサーバー、BaseMcpServlet、MCP学習メモ、Portainer MCP検証環境。                                           |
| Work074 | 2026年4月      | 運用・監視・MCP / LLM Tool Use                   | Portainer Ops MCP Java Server構築とDocker監視DTO設計 | Docker / Portainerの生JSONを、LLMが扱いやすい3層構造へ正規化し、ローカルLLMによるTool Useの制約を検証した。                                                                                                  | `portainer-ops-mcp`、Environments / Containers / IncidentDetail DTO、Converter群、PowerShell検証コマンド。              |
| Work075 | 2026年4月      | 実務応用・AI伴走型開発                               | Codexバイブコーディング環境構築とPTAクイズゲームUI改造検証            | Codexを用いたAI伴走型開発環境を構築し、既存ReactアプリのUI改修・CSVパーサー・実務向け改善フローを検証した。                                                                                                            | Codex利用環境、PTAクイズゲームUI改修、CSVパース処理、AI伴走型開発の実務適用知見。                                                             |
| Work076 | 2026年5月      | 実務応用・AI伴走型開発 / Git運用                       | Codexバイブコーディング量産工場に向けたGitブランチ運用の習得            | Codexで複数UI案を安全に量産するため、`main`を安定版、`vibe/*`をAI生成実験場として扱うブランチ運用を検証した。初回は管理体制を設計しないまま`main`上でCodexを実行していたため、未commit変更を新規ブランチへ退避し、以後はbranch / commit / graph確認で候補を管理する方式へ整理した。 | `vibe/quiz-ui-01`、`vibe/quiz-ui-02`として保存されたUI生成候補、AI生成時代のGit運用原則、main保護・実験ブランチ・最終案採用の手順知見。                   |
| Work077 | 2026年5月      | AI WebUI / プロンプト入力設計 / トラブルシューティング         | Gemini WebUIにおける1013エラーの調査とプロンプト誤検知の回避        | Gemini WebUIで発生した1013エラーについて、DevToolsで通信層とアプリケーション層を切り分け、プロンプト内の命令的表現が誤検知される可能性と回避策を検証した。                                                                                 | 検証対象テキストをコードブロックで囲み、AIへの直接命令ではなく引用データとして扱わせる入力エスケープ運用知見。                                                     |
| Work078 | 2026年5月      | 運用・監視・MCP / LLM Tool Use / ROCmトラブルシューティング | MCP Tool Use検証におけるROCm GPU HangとMCP設計境界の発見    | Portainer Ops MCP Serverのcontainers toolをOpen WebUI経由で検証し、MCPサーバーではなくtool-use後の再推論でROCm runnerが不安定化することを切り分けた。                                                            | MCPを大量構造化データ投入用のデータプレーンではなく、小さく安全な制御プレーンとして扱う設計知見。                                                           |
| Work079 | 2026年5月      | 実務応用・AI伴走型開発 / MCP / SafetyDesign          | SwitchBot制御用MCPサーバー2号機構築・Codex伴走型開発プロセス検証     | 自作MCP基盤とSwitchBot APIを接続し、Scene実行に限定した物理世界操作Toolを、Codexによる分析→Go / NoGo判断→製造の工程で構築した。                                                                                      | `switchbot-ctrl-mcp`、scenes / executeScene Tool、SceneCatalog、SceneService、cooldown、dry-run、AI監督責任開発プロセスの実践知。 |

---

## 5. 代表的な探索軸

この索引は、以下の観点で関連Workを探すために利用できる。

### 5.1 RAG / ナレッジパイプライン

関連Work:

* Work001
* Work002
* Work003
* Work004
* Work014
* Work017
* Work021
* Work022
* Work023
* Work024
* Work025
* Work026
* Work028

主な論点:

* 文書をそのまま投入しない
* Markdown / MkDocs / ChromaDB / Embedding / Memory Block を組み合わせる
* RAG登録処理の安定性を高める
* LLM生成物をJSON Schemaで検証する
* RAGを検索機能ではなくデータパイプラインとして扱う

### 5.2 ローカルLLM基盤

関連Work:

* Work005
* Work006
* Work008
* Work009
* Work010
* Work011
* Work015
* Work018
* Work029
* Work031
* Work062

主な論点:

* GPU / VRAM / OS / Docker / ドライバ制約
* ROCm / CUDA の併用
* Ubuntu Native移行
* Ollama常駐と運用監視
* CLI出力の構造化

### 5.3 モデル評価

関連Work:

* Work007
* Work012
* Work013
* Work016
* Work019
* Work020
* Work027
* Work030

主な論点:

* モデルサイズ、量子化、役割別評価
* 70B級、120B級モデルの評価
* 小型モデルの適用範囲
* 実運用プロンプト下での安定性

### 5.4 QLoRA

関連Work:

* Work032
* Work033
* Work034
* Work035
* Work036
* Work037
* Work038
* Work039
* Work040
* Work041
* Work042
* Work043
* Work044
* Work045
* Work046
* Work047
* Work048
* Work049
* Work050
* Work051
* Work052
* Work070
* Work072

主な論点:

* Unsloth環境構築
* Teacherモデルによる学習データ生成
* JSON Schemaによるデータ検証
* GGUF化、量子化、Ollama投入
* Gemma系モデルのチューニング
* 学習時テンプレートと推論時テンプレートの一致

### 5.5 マルチエージェント / 記憶制御

関連Work:

* Work003
* Work017
* Work021
* Work022
* Work023
* Work024
* Work025
* Work026
* Work071

主な論点:

* Director / Actor 分離
* Long-term Memory
* Current Scene Summary
* 会話ログの要約・抽出・再投入
* LLMの責務分離

### 5.6 音声・アバター・UX

関連Work:

* Work004
* Work053
* Work054
* Work055
* Work056
* Work057
* Work058
* Work059
* Work060
* Work061
* Work063
* Work064
* Work065
* Work066
* Work067
* Work068
* Work069

主な論点:

* 音声合成
* SSE
* React Three Fiber
* VRM / VRMA
* 3Dアバター
* リップシンク、表情、字幕
* UXとしての推論待ち時間

### 5.7 MCP / Tool Use

関連Work:

* Work031
* Work073
* Work074
* Work078
* Work079

主な論点:

* JSON-RPC
* MCP Streamable HTTP
* tools/list
* tools/call
* BaseMcpServlet
* Portainer / Docker監視DTO
* LLM向けTool粒度の設計
* tool-use後の再推論負荷
* MCPをデータプレーンではなく制御プレーンとして扱う設計
* 物理世界操作ToolにおけるScene実行限定
* SceneCatalogによる公開許可制御
* cooldown / dry-run
* 小型LLMを意識したTool数の最小化

### 5.8 AI伴走型開発

関連Work:

* Work075
* Work076
* Work079

主な論点:

* Codex
* Reactアプリ
* CSVパーサー
* 実利用者を想定したUI改善
* Gitブランチ運用
* AI生成候補の保存、比較、採用判断
* 分析 → Go / NoGo判断 → 製造
* AI監督責任開発
* 設計主導型AIコーディング
* Codexに任せる前の責務境界整理
* 既存クラス変更時の責務維持

### 5.9 AI WebUI / プロンプト入力設計

関連Work:

* Work077

主な論点:

* Gemini WebUI 1013エラーの切り分け
* DevToolsによる通信層とアプリケーション層の調査
* プロンプトインジェクション誤検知の回避
* コードブロックによる入力エスケープ
* 検証対象テキストをAIへの直接命令として扱わせない運用

---

## 6. 注意事項

この文書は、Work001〜Work079 の索引であり、個々のWorkの詳細本文ではない。

公開時には、以下の方針に従う。

* 業務経験と個人研究を混同しない
* 実運用環境を特定できる情報は記載しない
* 個人情報、認証情報、内部URL、IPアドレスは記載しない
* 公的・共同運営イベントについては、公開可能な範囲に限定する
* 実施前または結果未確定の成果を、確定済み実績として表現しない
* AIに読ませる場合も、この文書にない事実を補完させない

本索引は、ポートフォリオにおける技術証跡への入口として利用する。