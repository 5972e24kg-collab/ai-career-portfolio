# Work028: ハイブリッドAI育成基盤の設計

Tag: [[SystemDesign]], [[eGPU]], [[FineTuning]], [[QLoRA]], [[Unsloth]], [[EVO-X2]]

## 1. 今回の作業の目的

#contents

* 自作小説の登場キャラクターを、違和感なく対話可能なAIエージェントとして具現化するプロジェクト "Soul Injection" の始動。
* 既存のNode A (RTX 4060) の不安定さを解消し、Node B (EVO-X2) に **eGPU (RTX 3060)** を統合することで、推論と学習を同時に行う「反復的な学習・改善サイクル」を確立する。
* 「知識の先生 (120B)」と「演技の生徒 (8B)」による役割分担アーキテクチャを定義する。

## 2. システム環境 (Hybrid Compute)

Node B 1台の中に、ROCm (AMD) と CUDA (NVIDIA) が共存する異種混合環境を構築する。

* **Host Node:** Node B (GMKtec EVO-X2 / Ubuntu 24.04 Native)
* **Compute Resource A (Teacher / Brain):**

  * **Hardware:** AMD Ryzen AI Max+ 395 (Strix Halo)
  * **VRAM:** **96GB** (UMA Frame Buffer)
  * **Driver:** ROCm (via `/dev/kfd`)
  * **Role:** 教材作成（データセット生成）、RAG検索、論理推論。
* **Compute Resource B (Student / Soul):**

  * **Hardware:** NVIDIA GeForce RTX 3060 (via USB4 eGPU)
  * **VRAM:** **12GB**
  * **Driver:** CUDA (via `--gpus all`)
  * **Role:** QLoRA学習 (Unsloth)、軽量モデルによる高速ロールプレイ対話。

## 3. アーキテクチャ: The Self-Improvement Loop

「賢いが重いモデル」と「軽くて速いモデル」の特性を活かした育成フロー。

### Phase A: 教科書作成 (Dataset Generation)

* **Actor:** Teacher (gpt-oss:120b / Llama 4 Scout)
* **Input:** 自作小説のテキストデータ (Markdown)。
* **Process:** 大規模モデルの文脈処理能力を活用して小説内のシーンを解析し、「キャラクターのセリフ」と「想定される問いかけ」のペア（Q&A）を大量に合成する。
* **Output:** JSON形式の学習用データセット。

### Phase B: 魂の吹き込み (QLoRA Training)

* **Actor:** Student (Llama-3.1-8B-Instruct)
* **Tool:** Unsloth (高速学習ライブラリ)
* **Process:** RTX 3060上で、生成されたデータセットを用いてFine-tuningを行う。知識の追加を主目的とせず、「口調」「思考の癖」「性格表現」を重点的に学習させる。
* **Output:** LoRAアダプターファイル（キャラクターの人格表現に関する学習差分）。

### Phase C: 実戦配備 (Hybrid Inference)

* **Flow:**

  1. **User:** 問いかけ
  2. **Teacher (Node B):** RAGで小説内の関連知識を検索・抽出。
  3. **Student (RTX 3060):** 抽出された知識をコンテキストとして受け取り、学習済みの「人格表現」（口調や応答傾向）で演技して回答。

## 4. 今後の展望・ロードマップ

* **Step 1:** USB4 eGPUの物理セットアップと、DockerコンテナによるAMD/NVIDIA環境の分離構築。
* **Step 2:** 小説データのテキスト整形と、Teacherモデルによるデータセット生成テスト。
* **Step 3:** RTX 3060上でのUnsloth導入と、テスト学習の実施。
* **Goal:** 電源ユニット到着後、速やかに環境構築を開始し、最初の「魂」を生成する。
