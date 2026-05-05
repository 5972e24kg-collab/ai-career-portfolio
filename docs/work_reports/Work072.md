# Work072: Gemma 4 26B A4B のQLoRA成立とGGUF/Ollama本番化

Tag: [[Gemma4]], [[QLoRA]], [[GGUF]], [[Ollama]], [[Troubleshooting]], [[RTX3090]], [[Docker]]

## 1. 目的

- `unsloth/gemma-4-26B-A4B-it` をベースに、3090 24GB 環境で text-only QLoRA 学習を成立させる
- 学習済みアダプタを GGUF 化し、Ollama 上で全VRAM搭載に近い形で本番運用可能か検証する
- これまで何度も失敗していた 26B クラスのチューニングを、再現可能な手順として確立する
- 学習だけで終わらせず、実際に WebAPI の actor モデルとして差し替え可能なところまで到達する



## 2. システム環境

- **Node:** [Node name masked] ([Host name masked] / Ubuntu 24.04 Native / CUDA Sub: RTX 3090 24GB)
- **Target:** `[Date masked]_QLoRA_injection_Gemma4_26b`
- **Main Container:** `trainer`
- **GGUF Build Container:** `gguf-builder`
- **Runtime Container:** `ollama-cuda`
- **Base Model:** `unsloth/gemma-4-26B-A4B-it`
- **Method:** 4bit量子化 + PEFT LoRA + CPU offload
- **Training Runtime Versions:** `torch 2.9.1+cu128`, `transformers 5.5.0`, `accelerate 1.14.0.dev0`, `bitsandbytes 0.49.1`, `peft 0.18.1`, `trl 0.23.1`
- **Deployment Model Name:** `GemChan7.1:26b`

※ レポート形式は既存の Work070.md と同系統の構成に合わせた。



## 3. 作業ログ・解決プロセス

### 発生した課題

- `vision_tower` を CPU に逃がす方針で開始したが、初期段階で `Params4bit.__new__() got an unexpected keyword argument '_is_hf_initialized'` が発生し、`accelerate` / `bitsandbytes` / `transformers` の offload 経路で不整合が発生した
- `docker compose run --rm` のたびにコンテナ内の `accelerate` 更新が消え、修正版が永続化されていなかった
- `device_map="auto"` を使った自動 offload では、4bit + meta tensor 経路で別の不安定要素を踏みやすく、学習向けとして安定しなかった
- `device_map="auto"` を外すと、26B 本体のロード時点で OOM が発生し、24GB では本体を丸ごと GPU に載せられなかった
- `vision_tower` / `embed_vision` を CPU に逃がしても依然として OOM が継続し、ボトルネックが `language_model` 本体であることが判明した
- `language_model` の直下構造を調査した結果、`embed_tokens / layers / norm / rotary_emb`、かつ `layers=30` であることが分かり、層単位オフロードが必要になった
- 先頭 8 層、12 層では不足し、先頭 16 層 + `embed_tokens` + `lm_head` を CPU に逃がしてようやくモデルロードが成功した
- その後、`FastLanguageModel.get_peft_model(...)` で `NotImplementedError: Unsloth: gemma4 is not yet implemented!` が発生し、Unsloth の Gemma 4 対応不足が判明した
- Unsloth 経路を捨てて PEFT 直結へ切り替えたが、`prepare_model_for_kbit_training()` が追加の FP32 化により約 1.9GiB を要求し、再び OOM になった
- 軽量前処理へ置き換えた後は、`trl 0.23.1` 側の API 差分で `SFTTrainer.__init__() got an unexpected keyword argument 'tokenizer'` が発生した
- さらに Gemma 4 学習時には `mm_token_type_ids` が必須であり、通常の text-only SFT では forward が通らなかった
- これを解消した後も、TRL の `entropy_from_logits(outputs.logits)` による追加計算で、最後の 100MB 前後が足りず OOM した
- 推論比較スクリプトでは、CPU offload モデルに対して入力を CUDA へ明示転送していたため、`model is on meta` 警告と極端な遅延が出やすい構成になっていた
- 学習完了後、GGUF 化前提の merge / quantize / Ollama 登録 / 本番応答まで含めると、中間ファイルでストレージを大量消費した
- GGUF 登録後の実運用では、Gemma 4 の thinking 停止が時々見られ、WebAPI 側での `think:false` 制御が必要になった


### 解決策

- `accelerate` 修正版を手動 `pip install` ではなく Dockerfile に固定し、コンテナ再生成後も維持されるようにした
- `device_map="auto"` 依存をやめ、`AutoConfig + init_empty_weights()` でモジュール構造を確認したうえで、手動 `device_map` に切り替えた
- `model.vision_tower` と `model.embed_vision` を CPU 側へ固定し、multimodal 部分を明示的に offload した
- `language_model.layers.0-15` を CPU、`layers.16-29` を GPU とする層単位オフロードへ移行し、`embed_tokens` と `lm_head` も CPU 側へ寄せた
- LoRA 対象モジュールは GPU 側に残した text backbone のみを動的に抽出し、CPU 側に逃がした層・multimodal 層は明示除外した
- Unsloth の Gemma 4 未対応を確認した時点で、`FastLanguageModel.get_peft_model` を捨てて `PEFT + get_peft_model()` へ切り替えた
- `prepare_model_for_kbit_training()` を使わず、`requires_grad=False`、`gradient_checkpointing_enable()`、`enable_input_require_grads()` だけを行う軽量前処理へ差し替えた
- `trl 0.23.1` に合わせて `TrainingArguments` ではなく `SFTConfig` を使い、`tokenizer=` ではなく `processing_class=` に変更した
- Gemma 4 学習用に custom data collator を実装し、`token_type_ids` と `mm_token_type_ids` を text-only バッチへゼロ埋めで注入した
- `SFTTrainer` を軽量継承し、TRL が loss 以外に計算していた entropy 部分を省略して、学習 1 step 目の OOM を回避した
- スモークテストでは `max_train_samples=10`、`max_seq_length=512`、`epochs=1` で 2 step を完走し、その後 551 件の本番データで 1 epoch / 69 step を完走した
- 学習後は LoRA adapter を base model に merge し、merged HF を `llama.cpp` で GGUF 化した
- GGUF は FP16 中間ファイルから `Q4_K_M` を作成し、最終的に約 16GB 級へ圧縮した
- Ollama 用 Modelfile を整備し、`GemChan7.1:26b` として `ollama-cuda` へ登録した
- `ollama ps` で `SIZE 18 GB / 100% GPU / CONTEXT 2048` を確認し、VRAM 常駐と実応答を確認した
- 実応答では `response_token/s: 118.04`、`prompt_token/s: 1581.08`、概算 2 秒級の応答を確認し、速度面でも本番候補と判断した
- WebAPI は Ollama の chat API を直接叩いているため、thinking 停止については WebAPI 側で `think:false` を送る方針に切り替えた
- 作業完了後は、Ollama 登録用複製 GGUF、F16 中間 GGUF、merged HF、`.venv-gguf`、`llama.cpp`、`.cache` などの削除候補を洗い出し、不要ファイル整理を実施した


## 4. 成果物

### 主要成果物

```text
[Date masked]_QLoRA_injection_Gemma4_26b/
├── scripts/train_gemma4_26b_unsloth.py
├── scripts/infer_base_gemma4_26b.py
├── scripts/infer_tuned_gemma4_26b.py
├── scripts/merge_gemma4_26b_lora_to_full.py
├── scripts/build_gguf_gemchan26b_in_docker.sh
├── scripts/register_ollama_gemchan7.1_gguf.sh
├── adapters/gemma4_26b_run01
├── output/GGUF_GemChan6.0-26B-A4B/GemChan6.0-26B-A4B-Q4_K_M.gguf
└── output/Modelfile_GemChan7.1_GGUF/Modelfile
```

### 学習実行コマンド

```bash
docker compose run --rm \
  trainer \
  python scripts/train_gemma4_26b_unsloth.py \
    --max-seq-length 512 \
    --epochs 1 \
    --save-steps 9999 \
    --logging-steps 10 \
    --offload-first-n-layers 16 \
    --offload-embed-tokens-to-cpu \
    --offload-lm-head-to-cpu \
    --output-dir /workspace/adapters/gemma4_26b_run01
```

### LoRA merge コマンド

```bash
docker compose run --rm trainer \
  python scripts/merge_gemma4_26b_lora_to_full.py \
    --output-dir /workspace/export/merged_hf/GemChan6.0-26B-A4B-merged
```

### GGUF 変換コマンド

```bash
docker compose -f docker-compose.gguf.yml run --rm gguf-builder \
  bash ./scripts/build_gguf_gemchan26b_in_docker.sh
```

### Ollama 登録スクリプト

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR="[Path masked]/[Date masked]_QLoRA_injection_Gemma4_26b"
MODEL_NAME="GemChan7.1:26b"
CONTAINER_NAME="ollama-cuda"

GGUF_SRC_DIR="${PROJECT_DIR}/output/GGUF_GemChan6.0-26B-A4B"
BUILD_CTX_DIR="${PROJECT_DIR}/output/Modelfile_GemChan7.1_GGUF"

GGUF_FILE="GemChan6.0-26B-A4B-Q4_K_M.gguf"
TMP_DIR_IN_CONTAINER="/tmp/Modelfile_GemChan7.1_GGUF"

cp -f "${GGUF_SRC_DIR}/${GGUF_FILE}" "${BUILD_CTX_DIR}/${GGUF_FILE}"
docker exec "${CONTAINER_NAME}" rm -rf "${TMP_DIR_IN_CONTAINER}" || true
docker cp "${BUILD_CTX_DIR}" "${CONTAINER_NAME}:/tmp/"
docker exec "${CONTAINER_NAME}" ollama stop "${MODEL_NAME}" || true
docker exec "${CONTAINER_NAME}" ollama rm "${MODEL_NAME}" || true
docker exec "${CONTAINER_NAME}" bash -lc "
cd '${TMP_DIR_IN_CONTAINER}' && \
ollama create '${MODEL_NAME}' -f Modelfile
"
```

### 運用確認結果

```text
NAME              ID              SIZE     PROCESSOR    CONTEXT    UNTIL
GemChan7.1:26b    636263867a57    18 GB    100% GPU     2048       4 minutes from now
```

```text
response_token/s: 118.04
prompt_token/s: 1581.08
approximate_total: "0h0m2s"
```

### 最終結論

* Gemma 4 26B A4B の text-only QLoRA は、3090 24GB 単騎でも成立した
* ただし、手動 device_map・CPU offload・PEFT直結・TRL 回避実装など、複数の工夫を組み合わせた限定条件付きである
* さらに、GGUF Q4_K_M 化と Ollama 登録により、本番運用可能な応答速度と VRAM 使用量まで到達した
* これは「26B はチューニングできるか」の検証を超え、「26B を実運用 actor として差し替え可能か」の検証まで通した、大きな到達点である