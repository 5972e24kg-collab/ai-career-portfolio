# Work025: RAG Embedding OffloadとNode A GPU統合

Tag: [[System_Architecture]], [[RAG]], [[Ollama]], [[Troubleshooting]]

## 1. 目的

* **ROCmボトルネックの解消:** Node B (EVO-X2) において、Embeddingモデル `BAAI/bge-m3` がCPU実行となり、処理速度とコンテキスト長に制約が発生していた問題を解決する。
* **チャンクサイズの拡張:** 当時のCPU実行環境では約1500〜2000を運用上の目安としていたチャンクサイズを拡張し、**4000〜8192トークン** の長大コンテキストでのベクトル化を検証する。
* **遊休資産の活用:** 待機系であった Node A (RTX 4060) を「演算専用ワーカー」として定義し、システム全体のリソースを有効活用する。

## 2. システム環境

* **Requestor (Client):** Node B (EVO-X2 / Ubuntu 24.04)

  * Service: Java ETL (`CreateMemoryBlockJob`)
  * Role: テキストデータの送信、ベクトルデータのDB格納
* **Worker (Server):** Node A (DeskMeet / Win11 + WSL2)

  * Hardware: Core i5-12600K / RTX 4060 8GB
  * Service: Ollama (v0.5.x)
  * Role: GPUによるベクトル演算 (CUDA Native)

## 3. 作業ログ・解決プロセス

### 発生した課題

* **WSL2のネットワーク隔離:** WSL2上で稼働するOllamaはデフォルトで `localhost` にバインドされており、かつWindows側のIPとは異なる仮想ネットワーク内にいるため、LAN内の他ノード (Node B) から直接アクセスできなかった。

### 解決策

1. **Ollamaの公開設定:** Node Aにて環境変数 `OLLAMA_HOST` を設定し、必要な範囲からアクセス可能な待受設定に変更。
2. **Windows PortProxy:** `netsh interface portproxy` コマンドを使用し、Windows側の指定ポートへの着信を、WSL2の仮想IPへ転送するブリッジを作成。
3. **コンテキスト長の明示:** APIコール時に `num_ctx: 8192` を指定し、当時確認した既定値（2048）への依存を避けた。

## 4. 成果物

### A. Node A 接続設定 (PowerShell / 公開用サンプル)

```powershell
# 公開用サンプル
# 待受アドレスとポートは、実行環境の環境変数から取得する
$listen_address = $env:OLLAMA_LISTEN_ADDRESS
$ollama_port = $env:OLLAMA_PORT

# WSL2のIP動的取得とポートフォワード設定
$wsl_ip = (wsl -e ip addr show eth0) -match 'inet ' -replace '.*inet ([\d\.]+)/.*','$1'
netsh interface portproxy add v4tov4 listenport=$ollama_port listenaddress=$listen_address connectport=$ollama_port connectaddress=$wsl_ip

# 確認
netsh interface portproxy show all
```
