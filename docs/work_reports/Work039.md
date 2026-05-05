# Work039: Hugging Faceトレンド自動収集・要約システム構築
Tag: [[Java]], [[HuggingFace]], [[LLM_Agent]], [[ETL]], [[RAG]]

## 1. 目的
- **情報収集の自動化:** 毎日Hugging Faceのトレンド（Model, Dataset, Space）を巡回する時間を削減する。
- **知識の圧縮:** 難解なREADMEや技術仕様を、`gpt-oss:120b` の推論能力を用いて「初心者でも分かるニュース記事」に変換する。
- **RAG資産化:** 生成された記事を構造化データとして保存し、将来的に「過去のトレンド変遷」をAIが検索・分析できるようにする。

## 2. システム環境
- **Node:** [Host masked] (EVO-X2 / Ryzen AI Max+ 395 / VRAM 96GB)
- **Model:** `gpt-oss:120b` (Ollama)
- **Framework:** Java Custom ETL Framework (`File2Knowledge_Job`)
- **Target Source:** Hugging Face Trending API

## 3. 作業ログ・解決プロセス
### 発生した課題
1.  **API仕様の壁:** 当初想定していた `/api/models?sort=trending` は機能せず、専用エンドポイント `/api/trending` の解析が必要となった。
2.  **リソース種別の混在:** Trendには `model` だけでなく `dataset` や `space` が混在しており、単純なURL生成ではリンク切れが発生した。
3.  **メタデータの整合性:** LLMにYAMLヘッダーを書かせると、日付やUUIDなどの「確定情報」が不正確になるリスクがあった。

### 解決策
1.  **RepoType Strategy:** Java側で `repoType` フィールドを判別し、正しいURLとRaw Content取得先を切り替えるロジックを実装。
2.  **Context解放:** 120Bモデルの長文読解能力を活かすため、Java側の入力制限を15,000文字から75,000文字へ緩和。
3.  **Hybrid Injection:**
    - **LLMの役割:** 記事本文の執筆、タグ付け、重要度スコアリング（感性・知性領域）。
    - **Javaの役割:** `UUID`, `Date`, `SourceUrl` などのシステム情報の注入（正確性領域）。
    - 両者をマージして最終的なMarkdownを生成する方式を採用。

## 4. 成果物 (Core Logic Implementation)

### URL Generation Strategy (Java)
```java
// HuggingFaceTrendingProvider.java (Excerpt)

for (TrendingElement element : root.recentlyTrending) {
    String repoId = element.repoData.id;
    // デフォルトは model と仮定
    String type = (element.repoType != null) ? element.repoType : "model";
    
    String sourceUrl;
    String rawContentUrl;
    
    // リソースタイプに応じたURL生成戦略
    switch (type) {
        case "dataset":
            sourceUrl = "[https://huggingface.co/datasets/](https://huggingface.co/datasets/)" + repoId;
            rawContentUrl = String.format("[https://huggingface.co/datasets/%s/raw/main/README.md](https://huggingface.co/datasets/%s/raw/main/README.md)", repoId);
            break;
        case "space":
            sourceUrl = "[https://huggingface.co/spaces/](https://huggingface.co/spaces/)" + repoId;
            rawContentUrl = String.format("[https://huggingface.co/spaces/%s/raw/main/README.md](https://huggingface.co/spaces/%s/raw/main/README.md)", repoId);
            break;
        default: // model
            sourceUrl = "[https://huggingface.co/](https://huggingface.co/)" + repoId;
            rawContentUrl = String.format("[https://huggingface.co/%s/raw/main/README.md](https://huggingface.co/%s/raw/main/README.md)", repoId);
            break;
    }
    // ... (Fetch logic)
}
```