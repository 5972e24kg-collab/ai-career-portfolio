# Work006: EVO-X2導入計画と機能分散型アーキテクチャ策定

Tag: [[Architecture]], [[GMKtec]], [[Docker]], [[SystemDesign]]

## 1. 今回の作業の目的 [#purpose]

#contents

* 新規導入するハードウェア「GMKtec EVO-X2 (RAM 128GB)」の到着に備え、既存環境(Node A)との役割分担を明確化する。
* 従来の「1つのdocker-composeですべてを賄う」方式から、機能単位（Unit）で環境を切り分ける「モジュール型開発」へシフトし、検証の独立性を高める。
* ゲーム用途(Steam)とAI開発用途を物理的に分離し、主要な計算リソースの競合を抑える。

## 2. 新・システム環境（3ノード構成） [#environment]

今回の再設計により、以下の3台体制への移行を計画する。

##* Node B: AI Core Server (新規) [#node_b]

* ##Hardware:## GMKtec EVO-X2 (Ryzen AI Max+ 395 / RAM 128GB)
* ##IP Address:## [IP address masked] (Fixed)
* ##OS:## Windows 11 + WSL2 (Ubuntu) + Docker Desktop
* ##Role:## AI計算資源の集約。LLM推論、RAGデータベース、画像生成エンジンのホスト。
* ##Policy:## ヘッドレス運用。機能ごとにディレクトリを分けた独立コンテナ運用。

##* Node A: Legacy / Gaming Server (既存) [#node_a]

* ##Hardware:## NVIDIA GeForce RTX 4060 (VRAM 8GB)
* ##IP Address:## [IP address masked] (Fixed)
* ##Role:## Steam Linkホスト、CUDAベンチマーク、旧世代環境との比較検証。

##* Client: Control Center (操作端末) [#client]

* ##Hardware:## ASUS Zenbook 14 OLED (Core Ultra 9 / RAM 32GB)
* ##IP Address:## [IP address masked]
* ##Role:## RDP/SSHによる各サーバーの管理、Webブラウザ経由でのAIサービス利用。

## 3. 作業時間 [#time]

* 開始日時: 2025-12-02 08:00
* 完了日時: 2025-12-02 10:00
* 所要時間: 約 2 時間 (設計・要件定義)

## 4. 作業の軌跡と解決プロセス [#process]

##* 4.1 発生した課題・検討事項 [#issues]

* ##リソース競合の懸念:## Node Aで「重いLLM」と「Steamのゲーム」がVRAMを取り合い、双方が不安定になるリスクがあった。
* ##構成の肥大化:## `docker-compose.yml` にLLM、RAG、Toolなどが混在し、一部を変更・再起動すると全体が停止する「密結合」な状態になっていた。
* ##検証の複雑さ:## 新しいAI技術（例：画像生成）を試す際、既存の安定稼働しているチャット環境を壊す恐れがあった。

##* 4.2 解決策：機能分散型（ユニットベース）アーキテクチャ [#solution]

* ##物理分離:## Node Aを「ゲーム/予備」、Node Bを「AI専用」とする役割分担を策定した。
* ##論理分離 (Unit戦略):## Node B内でのDocker運用を「全部入り」から「機能別ディレクトリ」へ変更する。
  -- 最小単位の機能（例：「チャットだけ」「画像生成だけ」）で `docker-compose.yml` を完結させる。
  -- これにより、ある機能の変更や障害による影響を、他機能へ波及させにくい構成を目指す。

## 5. 成果物 (設計図) [#artifact]

EVO-X2到着後に構築するディレクトリ構造と、初期導入を予定している「Unit 1」の構成案です。

##* ディレクトリ構造 (Node B) [#dir_structure]
/home/user/ai-lab/
├── 01_llm_chat/             # 【Unit 1: 言語モデル機能】
│   ├── docker-compose.yml   # Ollama + Open WebUI (チャット機能の最小単位)
│   └── data/                # ベクトルDBや設定ファイル
│
├── 02_image_gen/            # 【Unit 2: 画像生成機能】(将来作成)
│   ├── docker-compose.yml   # ComfyUI または Forge
│   └── outputs/
│
└── 99_maintenance/          # 【Unit 99: 管理ツール】
├── docker-compose.yml   # Portainer (コンテナ管理GUI) など
└── scripts/

##* Unit 1: LLM Chat 構成案 (docker-compose.yml) [#unit1_code]
services:
ollama:
image: ollama/ollama:latest
container_name: ollama
ports:
- "11434:11434"
volumes:
- ollama_data:/root/.ollama
devices:
- /dev/kfd:/dev/kfd  # AMD GPUデバイス割り当て候補（WSL2上で要確認）
- /dev/dri:/dev/dri
environment:
- OLLAMA_HOST=0.0.0.0
restart: always

open-webui:
image: ghcr.io/open-webui/open-webui:main
container_name: open-webui
ports:
- "8080:8080"
volumes:
- open-webui_data:/app/backend/data
environment:
- OLLAMA_BASE_URL=http://ollama:11434
depends_on:
- ollama
restart: always

volumes:
ollama_data:
open-webui_data:

## 6. 今後の展望・ロードマップ [#roadmap]

* ##EVO-X2到着時:## BIOSでのVRAM割り当て（UMA Frame Buffer）設定を最優先で行う。
* ##Step 1:## WSL2上でのAMD GPU認識確認と、上記「Unit 1」の起動。
* ##Step 2:## 70Bクラス（Llama 3.3 70B等）のモデルロードと応答速度の検証。
* ##Step 3:## Node Aで使用していたRAGデータの移行と再インデックス。
