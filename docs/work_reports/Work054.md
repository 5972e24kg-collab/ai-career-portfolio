# Work055: StyleBertVITS2音声合成基盤構築と永続化
Tag: [[Node_C]], [[TTS]], [[Architecture]], [[StyleBertVITS2]]

## 1. 目的
- ジェムちゃんに「声」と「物理的実体」を与えるための新ノード "Node C" のハードウェア構成を評価する。
- 音声合成 (TTS) の技術的仕組み（テキスト解析から音声波形生成まで）を理解し、ブラックボックスを解消する。
- 脳（Node B）と身体（Node C）を連携させるための具体的な実装ロードマップを策定する。

## 2. システム環境 (新規構築)
- **Node Name:** Node C (Body)
- **OS:** Windows 11 Pro + WSL2 (Ubuntu 24.04)
- **CPU:** AMD Ryzen 7 5700G (APU for Display Output)
- **GPU:** NVIDIA RTX 3060 12GB (AI Dedicated)
- **Audio:** YAMAHA HS-5 (via USB-DAC & Balanced Amp)
- **Target Software:** Docker Desktop, NVIDIA Container Toolkit, StyleBertVITS2

## 3. 検討内容・決定事項
### ハードウェア評価
- **GPU専有化:** 画面出力をRyzen 5700G (APU) に任せ、RTX 3060 のVRAM 12GBをすべてAI推論（TTS/VRoid等）に割り当てる構成は、リソース効率の観点から**最適（評価S）**と判断。
- **オーディオ環境:** モニタリングスピーカー (HS-5) の採用により、生成音声の微細なノイズや息遣いの調整が可能。

### TTS技術の理解
- **パイプライン:** 以下の3段階で処理されることを確認。
    1.  **Text Frontend:** テキスト解析、読み仮名・アクセント付与。
    2.  **Acoustic Model:** 音素からメルスペクトログラム（音の設計図）を生成。感情パラメータはここで作用する。
    3.  **Vocoder:** スペクトログラムから音声波形（WAV）をレンダリング。
- **実装方針:** 独自学習（Fine-Tuning）はハードルが高いため、まずは「StyleBertVITS2」の既存モデル（[voice model name masked]等）を利用し、API経由で発声させる構成を採用。

### アーキテクチャ設計
- **疎結合モデル:**
    - Node Cは常時稼働する **API Server** として振る舞う。
    - Node B (Brain) が思考し、発話内容と感情パラメータをJSONでPOSTする。
    - Node Cがそれを受信し、GPUで推論してスピーカーを鳴らす。

## 4. 成果物
- **Phase 11 Roadmap:** 環境構築からLINE連携までの4ステップを定義完了。
- **Design Concept:** 「思考テキスト」と「読み上げテキスト」を分離し、URL等を読ませないロジック（Dual Output Stream）の採用を決定。