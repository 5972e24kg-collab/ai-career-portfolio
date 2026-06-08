# Work033: QLoRA Dataset Generation実装

Tag: [[Java]], [[LLM]], [[QLoRA]], [[ETL]], [[DataEngineering]]

## 1. 目的

* 自作小説のキャラクター「ジェム」の人格を再現するため、QLoRA学習用の高品質なQ&Aデータセットを自動生成する。
* 120Bパラメータを持つ `gpt-oss:120b` (Teacher) の読解力を利用し、小説の本文から「ジェムらしい思考と言動」を抽出・構造化する。

## 2. システム環境

* **Node:** ローカルLLM検証環境（EVO-X2 / Ryzen AI Max+ 395）
* **Teacher Model:** `gpt-oss:120b` (Ollama Native / VRAM 96GB)
* **Framework:** Java Custom ETL (`LlmServiceCore`, `JsonSchemaValidator`)
* **Source:** 自作小説テキスト (Markdown / 全約60話)

## 3. 作業ログ・解決プロセス

### 確立されたパイプライン

* **Hybrid ETL:** Javaジョブが小説ファイルを読み込み、LLMへプロンプトを送信。返却されたJSONを検証し、学習用ファイル(`dataset.json`)へ追記するフローを構築。

### 発生した課題と解決策

1. **JSONスキーマ不適合 (Missing Keys):**

   * **課題:** LLMが最適化を行い、空のフィールド（`"input": ""`）を省略して出力するケースが発生し、バリデーションエラーとなった。
   * **解決策:** `JsonSchemaValidator` が正常に機能している証拠と捉え、再生成（リトライ）ロジックで対応。必須キーの欠落を防いだ。
2. **固有名詞の表記揺れ (Strix vs Strik):**

   * **課題:** 小説内では商標回避のため「Strik-Halo」等の伏せ字を使用していたが、チャットボットとしては現実のRAG検索に支障が出る恐れがあった。
   * **解決策:** 学習ソースを変換前の「実名原稿（Strix Halo等）」に全量差し替え。これにより、ハードウェア認識と言語モデルの知識を接続しやすい学習データに整理できた。

## 4. 成果物

* **Dataset Generator Job:** 小説ディレクトリを巡回し、シーンごとにQ&Aを生成するJavaバッチ。
* **gem_soul_dataset.json:** Unsloth で利用する Alpaca format を想定した学習データ（目標600ペア）。

  * **Instruction:** 状況や問いかけ
  * **Input:** (Empty)
  * **Output:** ツンデレ黄金比と技術用語変換が適用されたジェムのセリフ

## 5. 今後の展望

* **Phase 3 (Next):** 生成されたデータセットを別GPU検証環境へ引き渡し、Unslothを用いた `Llama-3.1-8B-Instruct` のFine-Tuningを開始する。
