# Work029: Node Bへの完全集約とeGPUハイブリッド構成確立
Tag: [[EVO-X2]], [[eGPU]], [[RTX3060]], [[Docker]], [[Architecture]]

** 1. 今回の作業の目的
#contents
- Node A (RTX 4060) に依存していたEmbedding/画像生成タスクを、Node B (EVO-X2) に物理的に統合する。
- Node AをSteam Linkサーバーおよびゲーム専用機として完全に開放する。
- Strix Halo (ROCm) と RTX 3060 (CUDA) を単一OS上で共存させ、役割分担による最適化を図る。

** 2. システム環境
- **Node B:** GMKtec EVO-X2 (Ubuntu 24.04 Native)
- **iGPU:** Ryzen AI Max+ 395 (VRAM 96GB) -> ROCm (Ollama Main)
- **dGPU (eGPU):** GeForce RTX 3060 12GB -> CUDA (Ollama Sub)
- **Software:** Docker Compose (Hybrid Containerization)

** 3. 作業時間
- 開始: [Date masked] / 完了: [Date masked]

** 4. 作業の軌跡と解決プロセス
*** 4.1 発生した課題
- Node A (WSL2) の不安定さにRAGパイプラインが巻き込まれ、ベクトル化に失敗するケースがあった。
- Strix Halo単体では、CUDA専用資産（特定モデルやツール）の動作に制約があった。

*** 4.2 解決策・決定打
- **物理拡張:** eGPUドック経由でRTX 3060をEVO-X2に接続。
- **ハイブリッドドライバ環境:** Ubuntu上でNVIDIA DriverとROCmを共存させ、`nvidia-container-toolkit` を導入。
- **コンテナ分割:** `ollama-rocm` (Port [Port masked]) と `ollama-cuda` (Port [Port masked]) を立ち上げ、Open WebUIから統合制御する構成を確立。

** 5. 成果物
- **完全自立型AIサーバー:** 外部ノードに依存せず、推論・学習・RAGの全てがNode B内で完結。
- **省電力性:** VRAM合計108GBの超大容量環境ながら、ピーク時200W未満 / アイドル60Wという驚異的なワットパフォーマンスを達成。
- **ETL安定化:** LANを経由しない内部通信により、RAG取り込みの安定性が劇的に向上。

** 6. 今後の展望
- **RTX 3090への換装:** 現在の構成のまま、eGPUをVRAM 24GBモデルへ換装することで、合計VRAM 120GB環境を目指す。
- **マルチモーダル活用:** CUDA側リソースを用いた画像生成パイプライン (ComfyUI等) の構築。