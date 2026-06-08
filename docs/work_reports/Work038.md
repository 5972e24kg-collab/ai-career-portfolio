# Work038: Gemma 3 27Bモデルマージと対話スタイル調整

Tag: [[Gemma3]], [[Unsloth]], [[ModelMerge]], [[Docker]]

## 1. 目的

* 個別に学習させた3つのLoRAアダプター（Normal, Tsun, Dere）をブレンドし、目的とする性格バランスに近い27Bモデルを作成する。
* RTX 3090 (24GB) 単騎で、MergeからGGUF変換までを完結させる手法を確立する。

## 2. システム環境

* **Node:** 検証用ローカルノード（RTX 3090 eGPU）
* **Container:** `gemma3-merger:latest` (Custom Build)
* **Base Model:** `unsloth/gemma-3-27b-it-bnb-4bit`
* **Adapters:**

  * `gemma3-27b-tsun` (ツン)
  * `gemma3-27b-dere` (デレ)
  * `gemma3-27b-normal` (通常)

## 3. 作業ログ・解決プロセス

### 発生した課題

1. **Dependency Hell:** 最新の `Unsloth` と `vllm`, `mamba_ssm` 等のライブラリが競合し、ABIエラーやインポートエラーが多発。
2. **Identity Crisis:** Gemma 3 アーキテクチャが新しいため、Unslothがロードしたモデルが `PeftModel` として正しく認識されず、マージ保存処理がスキップされる。
3. **Matryoshka Directory:** PEFTの保存仕様により、出力ディレクトリがネストされ、GGUF変換器がファイルを見失う。

### 解決策

1. **Surgical Dockerfile:** 不要なライブラリ（vllm, mamba等）を削除し、PyTorchとUnslothを正しい順序で強制再インストールする `Dockerfile.merge_base` を構築。
2. **Monkey Patch Injection:** Pythonスクリプト内で、モデルを強制的に `PeftModel` でラップし、Unslothの隠蔽されたGGUF保存メソッド (`save_pretrained_gguf`) を外部から注入して実行する回避策を実装。
3. **Clean State Strategy:** 1レシピ処理するごとにVRAMとメモリを完全開放し、常にクリーンな状態でベースモデルをリロードする方式を採用。

## 4. 成果物: The Golden Ratio

検証時点で最もバランスが良いと判断した配合比率：

* **Name:** `Golden_Tsundere`
* **Ratio:** `Tsun: 0.7` + `Dere: 0.3`
* **Result:**

  * ユーザーに対し「貴方」と呼びかけ、厳しい指摘を行いつつも、協調性を見せる応答傾向を確認。
  * ※現時点ではEOSトークンの認識に甘さがあり、長文生成時にループする傾向があるため、推論パラメータでの制御が必要。

## 5. Next Action

* マージモデルのファインチューニング（DPO等）による、長文生成時のループ傾向の改善。
* `docker-compose up` 一発で新規マージが完了する自動化ワークフローの運用。
