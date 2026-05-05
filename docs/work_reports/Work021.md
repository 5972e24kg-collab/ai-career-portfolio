# Work021: 次世代RAGデータパイプライン要件定義
Tag: [[RAG]], [[Architecture]], [[EVO-X2]], [[HybridSearch]]

## 1. 目的
- 従来の「Markdownファイルをそのまま登録する」手法による検索精度の限界（ノイズ、文脈欠落）を打破する。
- EVO-X2 (VRAM 96GB) のリソースを活かし、人間のような文脈理解（右脳）と、データベースのような正確なキーワード検索（左脳）を両立する「賢い司書」システムを構築する。

## 2. システム環境・採用技術
- **Target Node:** Node B (EVO-X2)
- **Embedding Model:** `BAAI/bge-m3` (Dense / Sparse 両対応)
- **Metadata Generator:** `gpt-oss:120b` or `Llama 3.3 70B`
- **Database:** Vector DB (Dense + Sparse Hybrid Schema対応)

## 3. 策定された仕様 (The 4 Pillars)

### A. Structure (構造化・分割)
- **セマンティック・チャンキング:** 機械的な文字数カットではなく、Markdownの見出し (`#`) や段落を基準に分割。
- **コンテキスト注入:** 分割後の全チャンクに対し、親ファイル名や見出し階層 (`Source: file > H2`) を強制的に埋め込む。
- **オーバーラップ戦略:**
    - `Chat_Log`: 直前の1ターンを必ず含める（指示詞「あれ」「それ」対策）。
    - `Tech_Log` / `Novel`: 10-20% の重複を持たせる。
- **コードブロック保護:** 分割時は ` ``` ` を補完し、構文崩れを防ぐロジックを実装。

### B. Enrichment (メタデータ付与)
- **YAML Frontmatter:** 各ファイル先頭にメタデータを配置。
- **生成フロー:**
    - **親 (Parent):** ファイル全体からカテゴリ (`Novel`, `Tech`, `Chat`) 等を抽出し、全チャンクへ継承。
    - **子 (Child):** チャンクごとの固有キーワードをAIが生成し追記。

### C. Search Strategy (検索戦略)
- **ハイブリッド検索:** `Dense Vector` (意味) と `Sparse Vector` (キーワード) を併用。
- **実装方針:** `BAAI/bge-m3` を使用し、1回の推論で2種類のベクトルを生成・DB登録するパイプラインを構築。

### D. Refinement (リランキング)
- **アーキテクチャ定義:** 検索後の「2次審査」として Cross-Encoder (`BAAI/bge-reranker-v2-m3`) の導入を想定した設計とする。（実装はIngestion完了後の検索時）

## 4. 実装ロードマップ (概要)
以下の5段階のマイクロサービス（スクリプト）群として実装し、中間ファイルを生成しながら進める「各個撃破」方式を採用。

1.  **Parent Memory Block Service:** 文書全体から親メタデータ(JSON)を作成。
2.  **Semantic Chunking Service:** 文脈維持・コード保護を考慮したファイル分割。
3.  **Child Memory Block Service:** チャンクごとの詳細タグ付け。
4.  **RAG File Synthesis Service:** JSONをYAML化し、本文と結合した `.rag` ファイル生成。
5.  **Hybrid Ingestion Service:** Dense/Sparse両対応の新規登録スクリプト。

## 5. 次のアクション
- `gpt-oss:120b` を用いたカテゴリ別メタデータ抽出プロンプトの設計と実験（Phase 9-1 着手）。