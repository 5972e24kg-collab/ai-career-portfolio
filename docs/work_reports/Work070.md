# Work070: Gemma 4 E4B QLoRA学習・GGUF化・Ollama実装

Tag: [[Gemma4]], [[E4B]], [[Unsloth]], [[QLoRA]], [[GGUF]], [[Ollama]], [[Troubleshooting]], [[GemChan6.0]]

## 1. 目的

- `Gemma 4 E4B` をベースに、既存のジェムちゃん用データセットで QLoRA 学習できる環境を構築する
- 学習済み LoRA の効果を、base / tuned 比較で確認する
- `<inner_voice>` / `<response>` を含む疑似CoT構造が、Gemma 4 でも実用レベルで成立するか検証する
- 学習済み tuned モデルを GGUF 化し、`Ollama` に `GemChan6.0:E4B` として登録する
- 既存プロダクトへモデル差し替え可能な状態まで到達し、今後の本番調整フェーズへ移行する

## 2. システム環境

- **Project Dir:** `0507_QLoRA_injection_Gemma4_E4B`
- **Node:** ローカル検証環境（RTX 3090 eGPU）
- **GPU:** NVIDIA GeForce RTX 3090 24GB
- **Base Model:** `google/gemma-4-E4B-it`
- **Trainer Container:** `docker-compose.yml` 上の `trainer`
- **GGUF Builder Container:** `docker-compose.gguf.yml` 上の `gguf-builder`
- **Target Adapter:** `adapters/gemma4_e4b_lora_v2_textonly`
- **Merged HF Export:** `export/merged_hf/GemChan6.0-E4B-merged`
- **Ollama Import Target:** `GemChan6.0:E4B`
- **Prompt Assets:** `prompts/system_production_lite.txt`, `system_production_close_step1.txt`, `system_production_close_step2.txt`
- **Main Scripts:**
  - `scripts/train_gemma4_e4b_unsloth.py`
  - `scripts/infer_base_gemma4_e4b.py`
  - `scripts/infer_tuned_gemma4_e4b.py`
  - `scripts/merge_gemma4_e4b_lora_to_full.py`
  - `scripts/build_gguf_gemchan6_in_docker.sh`
  - `scripts/register_ollama_gemchan6_gguf.sh`

## 3. 作業ログ・解決プロセス

### 課題A: Gemma 4 E4B の LoRA 学習環境構築

- 既存の Gemma 3 系資産を流用しつつ、Gemma 4 E4B 用に学習データと学習スクリプトを再構成した
- 学習データは `dataset/` 配下にある JSONL 群を利用し、Gemma 4 用に整形して学習へ接続した
- Unsloth ベースの Docker 完結環境を構築し、ホスト Python へ依存を入れずに学習できるようにした

### 解決策

- `scripts/prepare_unsloth_dataset_gemma4_e4b.py` で学習用データを整形
- `scripts/train_gemma4_e4b_unsloth.py` を作成し、E4B 向け LoRA 学習を実装
- 学習対象モジュールを精査し、`audio_tower` / `vision_tower` / `multi_modal_projector` を除外した **text-only LoRA** へ修正
- 結果として `adapters/gemma4_e4b_lora_v2_textonly` を本命成果物として確立

---

### 課題B: Gemma 4 特有の `Gemma4ClippableLinear` と PEFT / Unsloth の相性問題

- 学習後の比較推論・LoRA 再読込で、`Gemma4ClippableLinear` 非対応や compiled cache 周りの問題が発生
- `compare_base_vs_tuned_gemma4_e4b.py` および tuned 推論系で、Gemma 4 固有実装へのパッチが必要になった

### 解決策

- `Gemma4ClippableLinear` を `nn.Linear` 継承クラスへ差し替えるパッチを実装
- `transformers.models.gemma4.modeling_gemma4` だけでなく、`unsloth_compiled_cache` / `builtins` にも注入
- `infer_tuned_gemma4_e4b.py` 側で再読込時にも再パッチする構成に修正
- これにより tuned モデルの単体推論が成立

---

### 課題C: base / tuned 比較時の OOM と stop 制御崩れ

- 1プロセスで base → tuned を連続実行すると VRAM 解放が不十分で OOM になった
- さらに初期比較では `<turn|>` 以降まで伸びたり、`model` が混ざるなど stop 制御が崩れた

### 解決策

- base 専用 / tuned 専用の推論スクリプトに分離
- `infer_base_gemma4_e4b.py` と `infer_tuned_gemma4_e4b.py` を個別実行型へ変更
- Gemma 4 の text-only 正攻法に合わせ、
  - `apply_chat_template(tokenize=False)`
  - `processor(text=...)`
  - `stop_strings`
  - `parse_response`
  を組み込んだ
- これにより、短文比較・production-lite・production-close step1 / step2 の評価が安定して実施可能になった

---

### 課題D: tuned モデルの有効性確認

- 本当に LoRA の効果が出ているか
- system prompt を強めた時でも base と tuned の差が残るか
- `<inner_voice>` / `<response>` の二層構造が疑似CoTとして実用になるか
- `emotion` / `temperature` のような追加制約を守れるか

### 解決策

- 比較用スクリプトと `prompts/` 配下の system prompt を段階的に作成
  - `system_light.txt`
  - `system_production_lite.txt`
  - `system_production_close_step1.txt`
  - `system_production_close_step2.txt`
- production-close step1 で `<inner_voice>` / `<response>` の安定性を確認
- production-close step2 で `[emotion=...][temperature=..]` を追加し、本番代替候補レベルで評価
- 結果として、短文試験では **tuned の方が base より人格の軸が安定し、LoRA 効果が明確** という結論に到達

---

### 課題E: Ollama への adapter 直接取り込み失敗

- `FROM gemma4:e4b` + `ADAPTER ./gemma4_e4b_lora_v2_textonly` 形式で `ollama create` を試したが、
  `no Modelfile or safetensors files found`
  で失敗
- サブディレクトリ配置・同一ディレクトリ + `ADAPTER .` の両方で同じ結果になった

### 解決策

- Safetensors adapter 直読みに固執せず、**merged full model → GGUF → Ollama** 経路へ方針転換
- `scripts/merge_gemma4_e4b_lora_to_full.py` で full model をマージ
- その成果物を `export/merged_hf/GemChan6.0-E4B-merged` に保存

---

### 課題F: GGUF 化の実行環境

- ホスト側で `build_gguf_gemchan6.sh` を実行したところ、`python3-venv` 不足で失敗
- さらに Docker 化後は `llama.cpp` に対する Git の dubious ownership で停止した

### 解決策

- `docker/gguf.Dockerfile` と `docker-compose.gguf.yml` を追加し、GGUF 変換専用コンテナを用意
- `scripts/build_gguf_gemchan6_in_docker.sh` を作成し、変換処理を Docker コンテナ内へ隔離
- `git config --global --add safe.directory /workspace/llama.cpp` を加えて Git ownership 問題を回避
- 結果として GGUF 生成に成功

---

### 課題G: Ollama への GGUF 登録

- GGUF 生成後、`scripts/register_ollama_gemchan6_gguf.sh` で `/tmp` へ転送し、Ollama モデル登録を実施
- 登録自体は成功し、`GemChan6.0:E4B` として起動確認できた

### 解決策

- `output/Modelfile_GemChan6.0_GGUF/Modelfile` を作成
- `GemChan6.0-E4B-Merged-7.5B-BF16.gguf` を `FROM ./...gguf` で取り込む構成に変更
- `ollama create GemChan6.0:E4B` に成功
- `ollama run GemChan6.0:E4B` で実応答確認を実施

---

### 課題H: 既存プロダクトへそのまま差し替えた時の違和感

- gemma4 モデル自体は動作し、レスポンスも返る
- ただし、既存プロダクトは gemma3 用 system prompt / 履歴投入 / parser 前提で作られているため、そのままでは従来動作を維持できない
- 特に `<inner_voice>` を履歴へ入れない前提や、長い日記・会話履歴の渡し方、タグ前提の downstream 処理など、フレームワーク側の思想が gemma3 用に最適化されていた

### 解決策 / 今後方針

- 今回の目的は「技術的に超えるべき壁を越えること」であり、その目的は達成できた
- 今後は **gemma4 を実プロダクトとして活用するための調整フェーズ** へ移行する
- system prompt の gemma4 向け再設計
- 日記・履歴の投入方法の再整理
- `<inner_voice>` / `<response>` パース戦略の再構築
- フレームワーク全体も Ver2 としてリファクタリング検討

## 4. 成果物

### 学習・推論関連
- `adapters/gemma4_e4b_lora_v2_textonly`
  - Gemma 4 E4B text-only QLoRA の本命成果物
- `scripts/train_gemma4_e4b_unsloth.py`
- `scripts/infer_base_gemma4_e4b.py`
- `scripts/infer_tuned_gemma4_e4b.py`
- `scripts/compare_base_vs_tuned_gemma4_e4b.py`

### プロンプト・検証関連
- `prompts/system_light.txt`
- `prompts/system_production_lite.txt`
- `prompts/system_production_close_step1.txt`
- `prompts/system_production_close_step2.txt`
- `prompts/prompts_production_lite.txt`
- `prompts/prompts_production_close_step1.txt`
- `prompts/prompts_production_close_step2.txt`
- `compare_outputs/` 以下の比較結果群

### マージ・GGUF・Ollama 関連
- `scripts/merge_gemma4_e4b_lora_to_full.py`
- `export/merged_hf/GemChan6.0-E4B-merged`
- `docker/gguf.Dockerfile`
- `docker-compose.gguf.yml`
- `scripts/build_gguf_gemchan6_in_docker.sh`
- `scripts/register_ollama_gemchan6_gguf.sh`
- `output/Modelfile_GemChan6.0_GGUF/Modelfile`
- `output/Modelfile_GemChan6.0_GGUF/GemChan6.0-E4B-Merged-7.5B-BF16.gguf`
- Ollama 登録済みモデル: `GemChan6.0:E4B`

※ モデル重み、merged model、GGUF ファイル本体は、ライセンスおよび配布条件を考慮し、公開リポジトリには含めない。

### 再現用メモ
- `001_train_gemma4_e4b.sh`
- `0031_base_text.sh`
- `0032_tuned_text.sh`
- `004_merge.sh`
- `005_gguf.sh`
- `006_register_ollama.sh`

## 5. 結論

- Gemma 4 E4B に対する QLoRA 学習は、今回の検証環境では成功した
- LoRA の効果は base / tuned 比較で明確に確認できた
- `<inner_voice>` / `<response>` を用いた疑似CoT構造も、短文試験では十分に実用レベルと判断できた
- GGUF 化と Ollama 実装まで到達し、`GemChan6.0:E4B` として既存プロダクトへ差し替え検証可能な状態になった
- 一方で、既存プロダクトは gemma3 用の prompt / 履歴 / parser / framework に最適化されているため、gemma4 を本格運用するには周辺設計の調整が必要
- 今回の最大の収穫は、**Gemma 4 E4B をジェムちゃんの実運用候補として成立させるための技術的障壁を突破したこと**
- 次フェーズは、**gemma4 実運用向けの prompt・履歴投入・parser・framework の再設計** である