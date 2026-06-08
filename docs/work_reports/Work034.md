# Work034: Llama-3.1-8B QLoRA Fine-Tuning

Tag: [[Fine-Tuning]], [[Unsloth]], [[QLoRA]], [[Ollama]], [[Gem]]

## 1. 目的

* 自作小説キャラクター「ジェム」の応答・人格表現データセット（約450件）を、Llama-3.1-8B モデルに学習（Fine-Tuning）させる。
* RTX 3060 (VRAM 12GB) 単体で学習からGGUF変換まで完走する、持続可能なローカル学習パイプラインを確立する。

## 2. システム環境

* **Node:** Local GPU検証環境 (eGPU / RTX 3060 12GB)
* **Container:** `unsloth/unsloth:latest` (CUDA 12.x)
* **Model:** `unsloth/Meta-Llama-3.1-8B-Instruct`
* **Method:** QLoRA (4-bit Quantization), LoRA Rank=16
* **Target:** Ollama (GGUF format)

## 3. 作業ログ・解決プロセス

### 発生した課題

1. **WandB/HuggingFace 認証エラー:**

   * 学習スクリプトがデフォルトで外部ログサービスへの接続を試み、Token不在でエラー終了した。
   * **Fix:** `TrainingArguments` に `report_to = "none"` を追加し、完全オフラインモードで実行。

2. **GGUF変換時の権限エラー:**

   * Unslothコンテナ内での `llama.cpp` 自動ビルド時、一般ユーザー (`unsloth`) では `sudo` が通らず失敗。
   * **Fix:** `docker exec -u 0` (rootユーザー) でコンテナに入り、依存パッケージのインストールと変換スクリプトを実行することで解決。

### 実装した工夫

* **ディレクトリ一括読み込み:**

  * 増え続ける学習データファイル (`*.md`) を `glob` で収集し、自動結合するロジックを `train.py` に実装。
* **VRAM最適化:**

  * `max_seq_length = 2048`
  * `per_device_train_batch_size = 2`
  * `gradient_accumulation_steps = 4`
  * 上記設定により、検証時点では12GB VRAM内で学習を完走できる構成を確認。

## 4. 成果物

* **Model:** `Gem-v0.1` (Based on Llama-3.1-8B-Instruct)
* **Format:** GGUF (Q4_K_M) / Size: ~4.9GB
* **Note:** モデル重み・学習済み成果物は公開対象に含めず、検証記録のみを公開する。
* **Persona:**

  * ユーザーを「マスター」と呼び、技術用語（VRAM, CPUクロック等）を比喩に用いる「Tech-Tsundere」傾向の応答を観察。
