# Work036: RTX3090導入とGemma-3-27B学習成功

Tag: [[RTX3090]], [[Gemma-3]], [[Unsloth]], [[QLoRA]]

## 1. 目的

* 検証用サブノードのeGPUを RTX 3060 (12GB) から **RTX 3090 (24GB)** へ換装し、VRAM枯渇問題を物理的に解決する。
* **Gemma-3-27B** (4bit量子化) のFine-Tuningを、VRAM 24GB環境下で安定して完走させる。

## 2. システム環境

* **Node:** 検証用サブノード (eGPU / MSI GeForce RTX 3090 VENTUS 3X 24G OC)
* **Power Supply:** 750W SFX (Updated from 500W)
* **Model:** `unsloth/gemma-3-27b-it-bnb-4bit`
* **Method:** QLoRA (Rank=16, Alpha=16)
* **Training Config:**

  * `per_device_train_batch_size = 1`
  * `gradient_accumulation_steps = 8`
  * `max_seq_length = 2048` (Initial Test)

## 3. 作業ログ・解決プロセス

### 発生した課題と対策

1. **物理セットアップの壁:**

   * 500W電源ではPCIe 8pinコネクタ数が不足（eGPUボード用+GPU用で計3本必要）かつ容量不足のため、750W電源へ換装を実施。
2. **市場在庫の確保:**

   * メモリ価格高騰とハイエンドGPU在庫枯渇のトレンドを踏まえ、比較的条件のよい中古個体を確保。
3. **27Bモデル学習のメモリ管理:**

   * 12GB環境ではロードすら困難だった27Bモデルだが、24GB環境では学習実行の余地があると予測。

### 検証結果 (nvidia-smi Analysis)

* **VRAM使用量:** Peak **18,251 MiB** (約74%)
* **GPU負荷:** 100% (Power 307W / Temp 66°C)
* **結果:** OOM (Out of Memory) を発生させることなく、学習プロセスを完走。
* **考察:** 残り約6GBの余剰があるため、次回は `max_seq_length` を **4096** または **8192** へ拡張できる可能性があると判断。

## 4. 成果物

* **Hardware:** 大容量ROCm環境と24GB CUDA環境を併用する検証基盤を構築。
* **Model:** `Gem-v0.2` (Based on Gemma-3-27B)。8B/12Bモデル運用時と比較して、表現力・推論能力の向上を確認。
