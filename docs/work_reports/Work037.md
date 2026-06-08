# Work037: Gemma 3 27B QLoRA学習環境の完全構築と量産

Tag: [[Gemma3]], [[Unsloth]], [[QLoRA]], [[GGUF]], [[Troubleshooting]]

## 1. 目的

* Googleの `Gemma 3 27B` をベースに、独自データセット（Normal/Tsun/Dere）を学習させる環境を構築する。
* 前回発生した「推論時の無限ループ（酔っ払い現象）」を、正しいトークナイザー設定と学習テンプレートで解決する。
* RTX 3090 (24GB) 単騎で、学習からGGUF量子化(Q4_K_M)までを完結させるパイプラインを確立する。

## 2. システム環境

* **Node:** ローカル検証機（RTX 3090 24GB eGPU）
* **Target Model:** `unsloth/gemma-3-27b-it-bnb-4bit`
* **Container:** Unsloth (Latest) + Custom Build llama.cpp
* **Dataset:** Markdown形式の学習用対話データ（Normal, Tsun, Dere, Mix）

## 3. 作業ログ・解決プロセス

### 課題A: 推論時の無限ループ（酔っ払い現象）

* **原因:** Gemma 3 特有のチャットテンプレート（`<start_of_turn>user` 等）が学習時に正しく適用されておらず、EOSトークンを見失っていた。
* **解決策:**

  * Unslothの `get_chat_template(..., chat_template="gemma")` を明示的に使用。
  * Modelfile側でもテンプレートを厳格に定義し、Ollamaにトークン構造を強制認識させた。
  * 結果、`eval count` が適切な値で止まり、正常な応答を得られる状態になった。

### 課題B: GGUF変換エラー (Unsloth vs llama.cpp)

* **現象:** Unsloth内蔵の変換スクリプトが、最新の `llama.cpp` (CMakeベース) のビルドに対応しておらず、自動変換が失敗する。
* **解決策:**

  * 「Unslothの自動機能」を放棄し、コンテナ内で `llama.cpp` を手動ビルド（CMake + CUDA有効化）。
  * Pythonスクリプトではなく、直接 `convert_hf_to_gguf.py` と `llama-quantize` を叩くシェルスクリプトで処理を実行。

### 課題C: VRAM容量の壁 (27Bモデル)

* **設定:** `batch_size=1`, `gradient_accumulation_steps=4`, `max_seq_length=2048`
* **結果:** VRAM使用量を約18GB〜20GBに抑え込み、RTX 3090 (24GB) での安定学習を実現。

## 4. 成果物

以下の4つの量子化済みモデル（Q4_K_M）が完成。

1. **gemma3-27b-mix-q4_k_m.gguf:** 全データ学習済み（ベースライン）。
2. **gemma3-27b-normal-q4_k_m.gguf:** 通常対話特化。
3. **gemma3-27b-tsun-q4_k_m.gguf:** ツンロジック特化。
4. **gemma3-27b-dere-q4_k_m.gguf:** デレモード特化。

これにより、次回以降「性格のマージ実験」や「動的な人格切り替え」を行うための準備が整った。
