# Work089: GIGABYTE AI TOP ATOMでのGemma4 31B QLoRA成立とOllama/Open WebUI実運用候補化

Tag: [[Gemma4]], [[QLoRA]], [[GGUF]], [[Ollama]], [[OpenWebUI]], [[GB10]], [[AI_TOP_ATOM]], [[CUDA13]], [[ARM64]], [[Troubleshooting]]

## 1. 目的

* GIGABYTE AI TOP ATOM / NVIDIA GB10 / ARM64 / CUDA 13.0 環境で `gemma4:31b` の QLoRA が成立するか検証する。
* 以前 EVO-X2 + RTX3090 24GB 環境で成功した `gemma4:26b` QLoRA の上位検証として、31Bモデルを手元で学習・実体化できるか確認する。
* ATOMを本番基盤へ移行して環境が複雑化する前に、クリーン寄りの状態で31B QLoRAの実績を取る。
* 失敗した場合も、原因・エラー・判断理由・解決策を記録し、後続の技術レポートに使える形で残す。
* 最終的に LoRA adapter 保存だけでなく、merge、GGUF化、Q4_K_M量子化、Ollama登録、Open WebUI応答、実運用Actor用SYSTEM適用まで到達できるか確認する。

## 2. システム環境

* **Node:** GIGABYTE AI TOP ATOM
* **Hostname:** `[local host masked]`
* **OS:** Ubuntu 24.04.4 LTS
* **Kernel:** `Linux 6.17.0-1021-nvidia`
* **Architecture:** `arm64 / aarch64`
* **GPU:** NVIDIA GB10
* **NVIDIA Driver:** `580.159.03`
* **CUDA Version:** `13.0`
* **Docker:** Docker Engine Community `29.2.1`
* **Docker OS/Arch:** `linux/arm64`
* **Memory:** 約121GiB unified memory
* **Network:** `[private IP masked]` 固定、有線LANのみ有効
* **Base Model:** `unsloth/gemma-4-31B-it`
* **Training Method:** 4bit NF4 QLoRA + PEFT LoRA
* **Training Runtime:** `torch 2.12.0+cu130`, `bitsandbytes 0.49.2`
* **Ollama Runtime:** `ollama/ollama:0.20.7`
* **Open WebUI:** `ghcr.io/open-webui/open-webui:v0.6.36`
* **Final Ollama Model Names:**

  * `GemChan8.0:31b`
  * `GemChan8.1:31b`
* **Main QLoRA Workdir:**

```text
[local QLoRA workdir]/0508_QLoRA_injection_Gemma4_31b
```

* **Ollama/Open WebUI Workdir:**

```text
[local Ollama workdir]/0101_ollama
```

* **Dataset Source:**

```text
[local previous QLoRA dataset]/dataset
```

* **Main Dataset:**

```text
data/raw/gemma4_26b_reference_dataset/ALL_merged_gemma4_26b_no_thinking.jsonl
```

* **Dataset Count:** 580件

## 3. 作業ログ・解決プロセス

### 3.1 外付けSSD作業領域の整備

#### 発生した課題

* 外付けSSDの対象パーティションは当初 NTFS であり、QLoRA作業領域として使うには不安があった。
* QLoRAでは Hugging Face cache、モデル重み、adapter、merge済みHFモデル、GGUF、量子化GGUFなど、大量の大容量ファイルと小ファイルが発生する。
* NTFSのまま進めると、Linux上の権限、symlink、Docker bind mount、I/O性能、障害時切り分けで不確定要素が増える可能性があった。

#### 解決策

* 外付けSSDを ext4 でフォーマットし、LinuxネイティブのQLoRA作業領域にした。
* `[local USB workdir]` にマウントし、`fstab` へ UUID 指定で登録した。
* `noatime,nofail,x-systemd.device-timeout=10` を付与し、不要なアクセス時刻更新を抑制しつつ、外付けSSD未接続時でもOS起動が止まらない構成にした。
* 作業ディレクトリを以下に作成した。

```text
[local QLoRA workdir]/0508_QLoRA_injection_Gemma4_31b
```

#### 判断理由

* 今回の主目的は ATOM / GB10 / ARM64 / CUDA13 で 31B QLoRA が通るかを確認することであり、ファイルシステム由来の問題を混ぜるべきではないと判断した。
* USB接続の作業領域ではあるが、QLoRA検証中は外付けSSDを専用作業領域として扱うため、ext4化するメリットが大きかった。

### 3.2 ディレクトリ構成とキャッシュ配置

#### 実施内容

以下の構成を作成した。

```text
[local QLoRA workdir]/0508_QLoRA_injection_Gemma4_31b/
├── docker/
├── scripts/
├── data/
│   └── raw/
│       └── gemma4_26b_reference_dataset/
├── cache/
│   ├── huggingface/
│   ├── torch/
│   ├── unsloth/
│   └── pip/
├── models/
│   ├── base/
│   ├── downloaded/
│   └── merged/
├── outputs/
│   ├── adapters/
│   ├── checkpoints/
│   ├── logs/
│   └── runs/
├── gguf/
│   ├── f16/
│   └── quants/
├── ollama/
├── offload/
├── reports/
└── tmp/
```

#### 解決策

* Hugging Face cache、torch cache、output、adapter、GGUF、offload をすべて外付けSSD側へ集約した。
* Docker本体の data-root は移動せず、コンテナ本体は内蔵SSD側、学習・生成物は外付けSSD側に分離した。

#### 判断理由

* Docker全体を外付けSSDへ移すと、Ollama/Open WebUIなど本番寄り環境まで巻き込んで不安定化する可能性がある。
* QLoRA由来の巨大ファイルだけを外付けSSDへ逃がすことで、内蔵SSDの圧迫を避けつつ、実運用環境との責任分界を明確にした。

### 3.3 Docker / CUDA / PyTorch / bitsandbytes 確認

#### 実施内容

* `nvidia/cuda:13.0.0-cudnn-devel-ubuntu24.04` をベースに、Python venv、PyTorch cu130、Transformers、PEFT、bitsandbytes 等を導入した。
* コンテナ内で `nvidia-smi`、PyTorch CUDA、bitsandbytes import を確認した。

#### 成功確認

```text
torch: 2.12.0+cu130
cuda available: True
cuda version: 13.0
device count: 1
device name: NVIDIA GB10
capability: (12, 1)
vram total GiB: 121.62497329711914
bitsandbytes: 0.49.2
Linear4bit import OK
```

#### 判断理由

* Unsloth公式経路へ最初から寄せず、まずは PyTorch + bitsandbytes + PEFT の基礎動作を確認することで、ARM64 / CUDA13 / GB10 の下回りを段階的に切り分けた。

### 3.4 EVO-X2 26B成功環境からの素材コピー

#### 実施内容

* EVO-X2 + RTX3090 で成功済みの `gemma4:26b` QLoRA 環境から dataset と `.env` を ATOM 側へコピーした。
* `.env` は `docker/.env` へ配置し、`chmod 600` を適用した。
* dataset は `data/raw/gemma4_26b_reference_dataset` に配置した。

#### 成功確認

```text
ALL_merged_gemma4_26b_no_thinking.jsonl: 580行
個別jsonl合計: 580行
全体 wc 合計: 1160行
external auth token exists
```

#### 判断理由

* 26B成功環境のデータを再利用することで、今回の差分を「ATOM / GB10 / ARM64 / CUDA13 / 31B」に集中できる。
* EVO-X2側の成功環境は比較対象・復旧元として残す必要があるため、移動ではなくコピーとした。

### 3.5 JSONL dataset 検証

#### 実施内容

`ALL_merged_gemma4_26b_no_thinking.jsonl` を読み込み、JSON形式とキー構造を確認した。

#### 成功確認

```text
count: 580
errors: 0
keys_seen:
  580 ('messages',)

OK
```

#### 判断理由

* 学習前に dataset が壊れていないこと、messages形式で統一されていることを確認した。
* `ALL_merged` と個別ファイル群は同一内容の集合である可能性が高いため、学習には `ALL_merged_gemma4_26b_no_thinking.jsonl` のみを使う方針とした。

### 3.6 Gemma4 31B tokenizer / config 読み込み

#### 実施内容

`unsloth/gemma-4-31B-it` の config と tokenizer を読み込んだ。

#### 成功確認

```text
config class: Gemma4Config
model_type: gemma4
architectures: ['Gemma4ForConditionalGeneration']
tokenizer class: GemmaTokenizer
pad_token: <pad>
eos_token: <turn|>
chat_template exists: True
OK
```

#### 補足

* `apply_chat_template(..., tokenize=False, add_generation_prompt=True)` では自己紹介文は生成されない。
* これはモデルに渡すプロンプト文字列を構築するだけであり、出力が `<|channel>thought` で止まるのは正常。

### 3.7 Docker Compose GPU割当問題

#### 発生した課題

初回の `03_load_gemma4_31b_4bit.py` で以下が発生した。

```text
cuda available: False
UserWarning: Can't initialize NVML
Device 0 is not available, available devices are []
RuntimeError: No CUDA GPUs are available
```

#### 調査結果

* ホスト側 `nvidia-smi` は正常。
* `nvidia-container-cli info` も正常。
* `sudo docker run --rm --gpus all nvidia/cuda:13.0.0-base-ubuntu24.04 nvidia-smi` も正常。
* 既存の `docker-compose.yml` では `deploy.resources.reservations.devices` を使っていた。

#### 解決策

`docker-compose.yml` のGPU指定を `deploy:` から `gpus: all` へ変更した。

```yaml
gpus: all
environment:
  - NVIDIA_VISIBLE_DEVICES=all
  - NVIDIA_DRIVER_CAPABILITIES=compute,utility
```

#### 成功確認

```text
docker exec qlora-gemma4-31b nvidia-smi
→ 成功

torch.cuda.is_available()
→ True
```

#### 判断理由

* ホスト側と `docker run --gpus all` が正常だったため、ドライバ障害ではなく Compose 経由のGPU割当問題と判断した。
* 今回のATOM環境では `deploy.resources.reservations.devices` より `gpus: all` の方が安定した。

### 3.8 31B 4bit load 成功

#### 実施内容

`BitsAndBytesConfig(load_in_4bit=True, nf4, double_quant, bf16)` で `gemma-4-31B-it` をロードした。

#### 成功確認

```text
Loading weights: 100%|██████████| 1188/1188
allocated GiB: 17.044
reserved GiB: 17.058
OK: model loaded
```

#### 実パラメータ配置確認

```text
model class: Gemma4ForConditionalGeneration
hf_device_map: None

--- parameter devices by numel ---
cuda:0 16353712688

--- parameter dtypes by numel ---
torch.bfloat16 1434338864
torch.uint8 14919373824

--- requires_grad by numel ---
True 1434338864
False 14919373824
```

#### 判断理由

* パラメータがすべて `cuda:0` に配置されており、CPU offload なしで31B 4bitモデルを扱えることが確認できた。
* RTX3090 24GBで26Bを扱った時と異なり、ATOM / GB10の大容量unified memoryにより、極端な手動offloadなしでロードできた。

### 3.9 NVML / CUDA初期化の不安定化

#### 発生した課題

31B級モデルロード後、既存コンテナ内で以下が再発することがあった。

```text
Failed to initialize NVML: Unknown Error
cuda available: False
RuntimeError: No CUDA GPUs are available
```

#### 調査結果

* ホスト側 `nvidia-smi` は正常。
* `docker run --gpus all` も正常。
* 既存コンテナだけGPUが見えなくなることがあった。

#### 解決策

重い処理の前に以下を標準手順とした。

```bash
cd [local QLoRA workdir]/0508_QLoRA_injection_Gemma4_31b/docker
docker compose restart qlora-gemma4-31b

docker exec -i qlora-gemma4-31b python - <<'PY'
import torch
print("cuda available:", torch.cuda.is_available())
print("device count:", torch.cuda.device_count())
if torch.cuda.is_available():
    print("device:", torch.cuda.get_device_name(0))
PY
```

#### 判断理由

* 原因を完全には特定できていないが、`docker compose restart` により復旧するため、コンテナ内のNVML/CUDA状態が一時的に壊れていると判断した。
* 今回は研究目的であり、重いステップごとの restart を運用回避策として採用した。

### 3.10 forward smoke 成功

#### 実施内容

31B 4bitモデルに対して、短い入力で forward を実行した。

#### 成功確認

```text
model loaded: Gemma4ForConditionalGeneration
first parameter device: cuda:0

try forward pattern 1: keys=['input_ids', 'attention_mask']

logits shape: (1, 22, 262144)
OK: forward smoke passed
```

#### 判断理由

* `logits shape: (1, 22, 262144)` により、実際にモデルが入力を受け取り、語彙サイズ262144のlogitsを返すことを確認した。
* text-only forward では `mm_token_type_ids` のゼロ埋めなしでも通ることが確認できた。

### 3.11 LoRA注入失敗：Gemma4ClippableLinear問題

#### 発生した課題

初回の LoRA injection で以下が発生した。

```text
ValueError: Target module Gemma4ClippableLinear(
  (linear): Linear4bit(in_features=1152, out_features=1152, bias=False)
) is not supported.
```

#### 原因

* `target_modules=["q_proj", "k_proj", ...]` の指定が広すぎ、text decoder 以外の `vision_tower` 側モジュールまでPEFTが拾っていた。
* `vision_tower` 側の `Gemma4ClippableLinear` は PEFT の LoRA 対象として未対応だった。

#### 検証結果

```text
all suffix hits: 788
supported hits after exclude: 410
unsupported hits: 189

--- class counter for suffix hits ---
Gemma4ClippableLinear 189
Linear4bit 599
```

unsupported sample は `model.vision_tower.encoder.layers...` に集中していた。

supported sample は以下のように `language_model` 側に存在した。

```text
model.language_model.layers.0.self_attn.q_proj Linear4bit
model.language_model.layers.0.self_attn.k_proj Linear4bit
model.language_model.layers.0.self_attn.v_proj Linear4bit
...
```

#### 解決策

* `model.named_modules()` を走査し、PEFT対応可能な `torch.nn.Linear` / `bnb.nn.Linear4bit` のみを抽出した。
* `vision`, `audio`, `projector`, `multi_modal`, `multimodal`, `image`, `mm_`, `clip` を含むモジュールを除外した。
* PEFTのsuffix広域一致を避けるため、抽出した完全モジュール名をregex化して `target_modules` に指定した。

#### 成功確認

```text
selected target count: 410
skipped unsupported count: 0

trainable params: 122,429,440 || all params: 31,395,515,952 || trainable%: 0.3900
```

#### 判断理由

* 今回の目的は text-only actor の学習であり、vision tower にLoRAを刺す必要はない。
* `language_model` 側の Linear4bit のみへLoRAを注入することで、PEFT未対応モジュールを回避しつつ、必要なtext backboneだけを学習対象にできると判断した。

### 3.12 smoke train 1件 成功

#### 実施内容

1件のみで forward / loss / backward / optimizer.step / adapter保存を確認した。

#### 成功確認

```text
loss: 8.223408699035645
backward: OK
optimizer step: OK
OK: smoke train 1 step completed
adapter saved: /workspace/outputs/adapters/gemma4_31b_smoke1
```

#### 成果物

```text
outputs/adapters/gemma4_31b_smoke1
adapter_model.safetensors 約490MB
```

#### 判断理由

* ここでは品質ではなく、QLoRAの学習経路が技術的に成立するかを確認した。
* 1件で backward と optimizer.step が通ったため、次段階の複数件学習に進めると判断した。

### 3.13 50件 / 100件 / 580件 学習

#### 50件学習

```text
SAMPLE_LIMIT=50
EPOCHS=1
GRAD_ACCUM=4
optimizer steps: 13
avg loss: 3.5560630679130556
OK: small train completed
adapter saved: /workspace/outputs/adapters/gemma4_31b_small_n50
```

#### 100件学習

```text
SAMPLE_LIMIT=100
EPOCHS=1
GRAD_ACCUM=4
optimizer steps: 25
avg loss: 2.686607873439789
OK: small train completed
adapter saved: /workspace/outputs/adapters/gemma4_31b_small_n100
```

#### 580件全件学習

```text
SAMPLE_LIMIT=580
EPOCHS=1
GRAD_ACCUM=4
optimizer steps: 145
avg loss: 2.0236500963054853
OK: small train completed
adapter saved: /workspace/outputs/adapters/gemma4_31b_full_n580_e1
```

#### 判断理由

* 1件 smoke train の後、50件、100件、580件と段階的に増やすことで、CUDA/NVML安定性、lossのNaN化、optimizer stepの進行、adapter保存を段階確認した。
* 580件でも安定して完走したため、ATOM / GB10 / ARM64 / CUDA13 環境で `gemma4:31b` のQLoRA学習が成立したと判断した。

### 3.14 adapter再ロード推論

#### 実施内容

保存済み adapter を `PeftModel.from_pretrained()` で再ロードし、base model + adapter の状態で推論した。

#### 成功確認

```text
30よ。そんな簡単な計算、電卓を叩く手間さえ惜しいわ。脳のリソースを無駄に消費しないで。
OK: adapter inference completed
```

#### 判断理由

* adapter保存だけではなく、再ロードして推論できることを確認した。
* 学習データ由来の口調が応答に反映されており、adapterが実際に効いていると判断した。

### 3.15 LoRA merge

#### 実施内容

* 4bitロード済みモデルにそのままmergeせず、BF16でbase modelを読み直した。
* LoRA adapterを読み込み、`merge_and_unload()` でmerged HF modelを作成した。
* 出力先は以下。

```text
models/merged/gemma4_31b_full_n580_e1_bf16
```

#### 成功確認

```text
models/merged/gemma4_31b_full_n580_e1_bf16: 59G
```

#### merge後推論

```text
30よ。こんな単純な計算にリソースを割かないで。もっと複雑なクエリを投げてきなさい。
OK: merged inference completed
```

#### 判断理由

* GGUF変換の前段として、adapter依存ではなく、LoRA統合済みHFモデルとして実体化する必要があった。
* merged model単体で推論できたため、adapterの統合は成功したと判断した。

### 3.16 GGUF変換時の tokenizer_config 問題

#### 発生した課題

初回の F16 GGUF 変換で、重みtensor変換は進んだが、tokenizer処理で以下が発生した。

```text
AttributeError: 'list' object has no attribute 'keys'
```

#### 原因

`tokenizer_config.json` を確認したところ、以下だった。

```text
extra_special_tokens
type: list
len: 1
['<|video|>']
```

converterと同じ経路で `AutoTokenizer.from_pretrained(..., use_fast=True)` を試すと、同じ `list has no attribute keys` が再現した。

#### 解決策

* 元の merged HF model は保持した。
* GGUF変換専用の hardlink copy を作成した。
* そのコピー側の `tokenizer_config.json` から `extra_special_tokens` を削除した。
* sanitized copy を入力として `convert_hf_to_gguf.py` を再実行した。

#### 成功確認

```text
INFO:gguf.gguf_writer:/workspace/gguf/f16/gemma4_31b_full_n580_e1_f16.gguf: n_tensors = 833, total_size = 61.4G
Writing: 100%|██████████| 61.4G/61.4G
INFO:hf-to-gguf:Model successfully exported to /workspace/gguf/f16/gemma4_31b_full_n580_e1_f16.gguf
```

成果物:

```text
gguf/f16/gemma4_31b_full_n580_e1_f16.gguf
size: 58G
```

#### 判断理由

* Gemma4の tokenizer_config が新しい形式を含んでおり、Transformers v4系またはllama.cpp converter側の互換性に引っかかったと判断した。
* 本番推論用merged HF modelを壊さないため、変換専用copyを作り、tokenizer_configだけをサニタイズした。

### 3.17 Q4_K_M量子化と llama.cpp 推論

#### 実施内容

F16 GGUFから Q4_K_M を作成した。

#### 成功確認

```text
gguf/quants/gemma4_31b_full_n580_e1_Q4_K_M.gguf
size: 18G
```

llama.cpp single-turn 推論:

```text
> 10足す20はいくつ？

30よ。そんな単純な計算、電卓でも使えばいいんじゃない？ 私の演算能力をそんなことに使わせないで。

[ Prompt: 153.1 t/s | Generation: 10.7 t/s ]
Exiting...
```

#### 補足

* 最初の raw prompt 実行では `>` が続いたが、これは `llama-cli` のconversation/interactive modeに入ったためと判断した。
* `--single-turn` を付けることで1ターン生成後に終了し、正常にログが取れた。

#### 判断理由

* Q4_K_M GGUF単体で llama.cpp から応答できたため、GGUF実体化は成功と判断した。
* 速度は高速ではないが、ジェムちゃんActor用途では即時性より応答内容・安定性を優先するため許容範囲と判断した。

### 3.18 Ollama + Open WebUI 環境構築

#### 実施内容

ATOM内で完結するOllama/Open WebUI構成を作成した。

作業ディレクトリ:

```text
[local Ollama workdir]/0101_ollama
```

#### 方針

* `~/ai-lab-usb` はUSB接続の一時作業領域であり、接続が外れる可能性がある。
* 実運用に必要なGGUFは `~/ai-lab` 配下へコピーした。
* ATOMはCUDA単独構成のため、EVO-X2のようにROCm/CUDA二系統に分けず、`ollama` と `open-webui` の2コンテナ構成にした。
* Docker Composeでは、QLoRA検証で安定した `gpus: all` を採用した。

#### コンテナ確認

```text
open-webui: healthy
ollama: Up
ollama version: 0.20.7
docker exec ollama nvidia-smi: 成功
```

### 3.19 Ollama登録：GemChan8.0:31b

#### 実施内容

Q4_K_M GGUFを `GemChan8.0:31b` としてOllamaに登録した。

#### 成功確認

```text
NAME              ID              SIZE     MODIFIED
GemChan8.0:31b    9d4ef491a6b6    18 GB    About a minute ago
```

CLI推論:

```text
docker exec -it ollama ollama run GemChan8.0:31b "10足す20はいくつ？"

30よ。そんな簡単な計算、電卓に頼るなんてまだ早いわ。私が計算してあげるから、次からはちゃんと自分で考えなさい。
```

Open WebUI応答:

```text
response_token/s: 10.78
prompt_token/s: 247
total_duration: 37676271381
load_duration: 34362822528
completion_tokens: 33
```

#### 判断理由

* Ollama CLI と Open WebUI の両方で応答が返ったため、推論環境として成立した。
* `response_token/s: 10.78` は高速ではないが、ジェムちゃんプロダクトでは速度より演技・形式安定・実運用可能性を重視するため許容と判断した。

### 3.20 実運用Actor用SYSTEM適用：GemChan8.1:31b

#### 実施内容

現行実運用中の `GemChan7.1:26b` 用 Modelfile / SYSTEM をベースに、`GemChan8.1:31b` を作成した。

#### 成功確認

```text
NAME              ID              SIZE     MODIFIED
GemChan8.1:31b    7251f0d7263a    18 GB
GemChan8.0:31b    9d4ef491a6b6    18 GB
```

実運用形式テスト:

```bash
docker exec -it ollama ollama run GemChan8.1:31b '{
  "script_json": {
    "cue_type": "chat_reply",
    "goal": "ユーザーの挨拶に短く返す",
    "topic_anchor": "朝の挨拶",
    "topic_mode": "direct_reply",
    "surface_tone": "軽く刺すが親しい",
    "subtext": "ちゃんと見ている",
    "must_include": ["挨拶"],
    "avoid": ["長文", "説明口調"],
    "voice_style": "近い距離感",
    "end_style": "軽く余韻",
    "voice_length_max": 40,
    "chat_length_max": 100
  },
  "user_message": "おはよう"
}'
```

出力:

```xml
<performance>
  <voice_line>おはよう。いつまで寝てるかと思ってたわよ。さっさと起きて。</voice_line>
  <chat_line>おはよう。やっと起きたわね。ログアウトしてた時間、意外と長かったわよ？</chat_line>
  <emotion_tags>
    <emotion primary="呆れ" secondary="安心" />
  </emotion_tags>
  <gesture_tags>
    <face>pout</face>
    <motion>small_tilt</motion>
    <subtitle_mode>normal</subtitle_mode>
  </gesture_tags>
</performance>
```

#### 判断理由

* `<performance>`、`<voice_line>`、`<chat_line>`、`emotion_tags`、`gesture_tags` の構造を守っている。
* `script_json` の意図を解釈し、短く自然なツンデレActor応答を返せている。
* 31B化により、SYSTEMプロンプトと出力形式への追従性が高い傾向が見られ、GemChan8系の実運用基盤候補として評価可能な段階に到達したと判断した。

## 4. 成果物

### 主要ディレクトリ

```text
[local QLoRA workdir]/0508_QLoRA_injection_Gemma4_31b/
├── scripts/
├── reports/
├── outputs/adapters/
│   ├── gemma4_31b_smoke1
│   ├── gemma4_31b_small_n50
│   ├── gemma4_31b_small_n100
│   └── gemma4_31b_full_n580_e1
├── models/merged/
│   ├── gemma4_31b_full_n580_e1_bf16
│   └── gemma4_31b_full_n580_e1_bf16_gguf_sanitized
├── gguf/
│   ├── f16/
│   └── quants/
└── tools/llama.cpp
```

### 最終成果物サイズ

```text
498M    outputs/adapters/gemma4_31b_full_n580_e1
59G     models/merged/gemma4_31b_full_n580_e1_bf16
58G     gguf/f16
18G     gguf/quants
968K    reports
```

### 公開時の扱い

本レポートは検証過程の記録であり、利用した外部モデル・ライブラリ・ツール・生成物の再配布可否は、それぞれのライセンスおよび利用条件に従う。モデル重み、GGUF、学習済みadapter等の成果物は、本ポートフォリオには含めない。

### 主要レポートログ

```text
00_environment_baseline.md
02_probe_gemma4_31b.md
03_load_gemma4_31b_4bit_retry01.md
03b_inspect_loaded_model.md
03c_forward_smoke_gemma4_31b_retry01.md
04_inspect_lora_targets.md
10_smoke_train_1_gemma4_31b_retry01.md
11_train_small_gemma4_31b_n50.md
11_train_small_gemma4_31b_n100.md
11_train_full_gemma4_31b_n580_e1.md
12_infer_adapter_gemma4_31b_full_n580_e1.md
30_merge_adapter_gemma4_31b_full_n580_e1.md
31_infer_merged_gemma4_31b_full_n580_e1.md
40_convert_hf_to_gguf_f16.md
40_convert_hf_to_gguf_f16_retry01.md
41_quantize_gemma4_31b_Q4_K_M.md
43_infer_gguf_q4km_llamacpp_single_turn.md
final_artifact_sizes_20260606_0758.txt
```

### 代表コマンド：全件QLoRA学習

```bash
cd [local QLoRA workdir]/0508_QLoRA_injection_Gemma4_31b

docker exec \
  -e MODEL_ID=unsloth/gemma-4-31B-it \
  -e SAMPLE_LIMIT=580 \
  -e EPOCHS=1 \
  -e MAX_SEQ_LENGTH=512 \
  -e GRAD_ACCUM=4 \
  -e OUTPUT_DIR=/workspace/outputs/adapters/gemma4_31b_full_n580_e1 \
  qlora-gemma4-31b \
  python /workspace/scripts/11_train_small_gemma4_31b.py \
  2>&1 | tee reports/11_train_full_gemma4_31b_n580_e1.md
```

### 代表結果：全件QLoRA学習

```text
loaded samples: 580
optimizer steps: 145
avg loss: 2.0236500963054853
OK: small train completed
adapter saved: /workspace/outputs/adapters/gemma4_31b_full_n580_e1
```

### 代表コマンド：adapter再ロード推論

```bash
docker exec \
  -e MODEL_ID=unsloth/gemma-4-31B-it \
  -e ADAPTER_DIR=/workspace/outputs/adapters/gemma4_31b_full_n580_e1 \
  -e PROMPT='10足す20はいくつ？' \
  qlora-gemma4-31b \
  python /workspace/scripts/12_infer_adapter_gemma4_31b.py \
  2>&1 | tee reports/12_infer_adapter_gemma4_31b_full_n580_e1.md
```

### 代表結果：adapter再ロード推論

```text
30よ。そんな簡単な計算、電卓を叩く手間さえ惜しいわ。脳のリソースを無駄に消費しないで。
OK: adapter inference completed
```

### 代表コマンド：F16 GGUF変換

```bash
docker exec qlora-gemma4-31b bash -lc '
cd /workspace/tools/llama.cpp

python convert_hf_to_gguf.py \
  /workspace/models/merged/gemma4_31b_full_n580_e1_bf16_gguf_sanitized \
  --outfile /workspace/gguf/f16/gemma4_31b_full_n580_e1_f16.gguf \
  --outtype f16
' 2>&1 | tee reports/40_convert_hf_to_gguf_f16_retry01.md
```

### 代表結果：F16 GGUF変換

```text
n_tensors = 833, total_size = 61.4G
Writing: 100%|██████████| 61.4G/61.4G
Model successfully exported to /workspace/gguf/f16/gemma4_31b_full_n580_e1_f16.gguf
```

### 代表コマンド：Q4_K_M量子化

```bash
docker exec qlora-gemma4-31b bash -lc '
cd /workspace/tools/llama.cpp

./build/bin/llama-quantize \
  /workspace/gguf/f16/gemma4_31b_full_n580_e1_f16.gguf \
  /workspace/gguf/quants/gemma4_31b_full_n580_e1_Q4_K_M.gguf \
  Q4_K_M
' 2>&1 | tee reports/41_quantize_gemma4_31b_Q4_K_M.md
```

### 代表結果：Q4_K_M GGUF

```text
gguf/quants/gemma4_31b_full_n580_e1_Q4_K_M.gguf
size: 18G
```

### 代表コマンド：llama.cpp single-turn推論

```bash
docker exec qlora-gemma4-31b bash -lc '
cd /workspace/tools/llama.cpp

./build/bin/llama-cli \
  -m /workspace/gguf/quants/gemma4_31b_full_n580_e1_Q4_K_M.gguf \
  --single-turn \
  -p "10足す20はいくつ？" \
  -n 120 \
  --temp 0.7 \
  --top-p 0.9 \
  --repeat-penalty 1.05
' 2>&1 | tee reports/43_infer_gguf_q4km_llamacpp_single_turn.md
```

### 代表結果：llama.cpp推論

```text
30よ。そんな単純な計算、電卓でも使えばいいんじゃない？ 私の演算能力をそんなことに使わせないで。

[ Prompt: 153.1 t/s | Generation: 10.7 t/s ]
Exiting...
```

### Ollama / Open WebUI 構成

```text
[local Ollama workdir]/0101_ollama/
├── docker-compose.yml
├── ollama-data/
├── open-webui-data/
├── gemma4_31b_full_n580_e1_q4km/
└── gemchan8_1_31b/
```

### Ollama登録モデル

```text
NAME              SIZE
GemChan8.1:31b    18 GB
GemChan8.0:31b    18 GB
```

### GemChan8.1:31b 実運用形式テスト結果

```xml
<performance>
  <voice_line>おはよう。いつまで寝てるかと思ってたわよ。さっさと起きて。</voice_line>
  <chat_line>おはよう。やっと起きたわね。ログアウトしてた時間、意外と長かったわよ？</chat_line>
  <emotion_tags>
    <emotion primary="呆れ" secondary="安心" />
  </emotion_tags>
  <gesture_tags>
    <face>pout</face>
    <motion>small_tilt</motion>
    <subtitle_mode>normal</subtitle_mode>
  </gesture_tags>
</performance>
```

## 5. 未解決の課題・今後の検証

### 未解決の課題

* 31B級モデルロード後に、既存コンテナ内で NVML / CUDA 初期化が失敗することがあった。`docker compose restart` で回復するが、根本原因は未特定。
* GGUF変換時に `tokenizer_config.json` の `extra_special_tokens` を削除する必要があった。これは変換専用copyで回避しているが、Gemma4 / Transformers / llama.cpp 側の将来更新で挙動が変わる可能性がある。
* 変換ログには `Unknown RoPE type: proportional` 系の警告があり、短文推論では問題が出ていないが、長文コンテキストでの挙動は未検証。
* `GemChan8.1:31b` は実運用Actor用SYSTEMでXML出力に成功したが、directorからの実 `script_json` を大量に流した安定性検証は未実施。
* `GemChan7.1:26b` と `GemChan8.1:31b` の品質比較、速度比較、形式安定性比較は今後の評価対象。
* Q8_0 GGUF、異なる量子化方式、`num_ctx` 拡張、KV cache設定変更は未検証。
* 作業ログ中に外部サービス認証情報が表示された可能性があったため、公開前に当該トークンの失効・再発行を行う必要がある。

### 今後の検証候補

* `GemChan8.1:31b` に対して実directorからの `script_json` を流し、`voice_line` / `chat_line` のパース安定性を確認する。
* LINE / WebAPI / 音声合成 / アバター連動まで含めた実運用統合テストを行う。
* `GemChan7.1:26b` と同一入力セットで比較し、応答品質、演技安定性、XML形式遵守率、速度を比較する。
* Open WebUI上で長めの会話を行い、SYSTEM追従、余計なメタ発言、出力形式崩れを確認する。
* ATOM上のOllama運用における `OLLAMA_KEEP_ALIVE`、`OLLAMA_NUM_PARALLEL`、`OLLAMA_KV_CACHE_TYPE`、`num_ctx` の最適値を探る。
* 最終成果物をNAS等へ退避し、USB作業領域に依存しない形でバックアップする。

## 6. 最終結論

GIGABYTE AI TOP ATOM / NVIDIA GB10 / ARM64 / CUDA13 環境において、`unsloth/gemma-4-31B-it` を対象とした QLoRA 検証は成功した。

PyTorch cu130、bitsandbytes 0.49.2、PEFTを用いて、31Bモデルの4bitロード、LoRA注入、580件データセットでの1epoch学習、adapter保存、adapter再ロード推論、BF16 merge、F16 GGUF変換、Q4_K_M量子化、Ollama登録、Open WebUI応答まで確認した。

途中、Docker Compose経由のGPU割当問題、NVML初期化不安定化、Gemma4ClippableLinearへのPEFT非対応、tokenizer_config.jsonのextra_special_tokens形式差異といった問題が発生したが、それぞれ切り分けと回避策を実施した。

最終的に `GemChan8.1:31b` として、実運用Actor用SYSTEMプロンプトに従った `<performance>` 形式の応答生成まで成功した。

これは、RTX3090 24GB環境では困難だった31B級モデルのQLoRAを、ATOM / GB10 の大容量unified memory環境で実現した検証であり、GemChan8系の実運用基盤候補として評価可能な段階に到達した記録である。
