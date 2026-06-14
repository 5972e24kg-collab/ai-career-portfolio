# Work030: RTX3060 12GB環境でのQwen3と32k Context限界調整

Tag: [[Tuning]], [[Ollama]], [[RTX3060]], [[Qwen3]], [[RAG]]

## 1. 目的
- Node B (Hybrid) のサブGPUである RTX 3060 (12GB) において、RAG検索結果（ネットワーク図等の大量データ）を処理しきれずに回答精度が低下していた問題を解決する。
- 物理メモリの限界値を見極め、シングルモデル運用における最大パフォーマンスを引き出す設定を確定する。

## 2. システム環境
- **Node:** Node B (Sub GPU: NVIDIA GeForce RTX 3060 / 12GB)
- **Model (Main):** `qwen3:8b` (Context: 32k)
- **Model (Embed):** `bge-m3` (Context: 4k)
- **Driver:** NVIDIA Driver 580.95.05 / CUDA 13.0

## 3. 作業ログ・解決プロセス
### 発生した課題
- `qwen3:8b` のデフォルト設定（Context 4k）では、RAGが検索した約1.5万トークンの情報を読み込めず、`truncating input prompt`（切り捨て）が発生していた。

### 解決策
- **モデル選定:** 日本語と論理推論に強い `qwen3:8b` を採用。
- **コンテキスト拡張:** `num_ctx` を **32768 (32k)** に設定。
- **メモリ管理:** 同時稼働するEmbeddingモデル `bge-m3` との共存を計算し、VRAM 12GB内に収まるギリギリのラインを狙った。

## 4. 成果 (Resource Verification)
`nvidia-smi` および `ollama ps` による実測値。

| Process | VRAM Usage | Status |
| :--- | :--- | :--- |
| **qwen3:8b (32k)** | **9610 MiB** (~9.4GB) | 100% GPU Offload |
| **bge-m3** | **1014 MiB** (~1.0GB) | 100% GPU Offload |
| **Total Usage** | **10770 / 12288 MiB** (87%) | **Safe Range** |

- **結論:** 12GB VRAM環境において、実用的なRAG（32kコンテキスト）と高速なEmbeddingを両立させる「黄金比」設定を確立した。