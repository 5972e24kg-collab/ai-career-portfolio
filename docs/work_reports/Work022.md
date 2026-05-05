# Work022: Parent Memory Block Service構築

**Tags:** [Java], [LLM], [RAG], [JsonSchema], [Architecture]

## 1. プロジェクト総括 [#summary]

[TOC]

* RAG（検索拡張生成）のデータ純度向上を目的とした「Parent Memory Block (親メモリ)」生成サービスの設計および実装を完了。
* LLMの出力不安定性を排除するため、**「中間フォーマットとしてのJSON」**と**「JSON Schemaによる厳格なバリデーション」**を採用するアーキテクチャを確立。
* データの性質（雑談 vs 技術）に応じた2つの生成パイプライン（ローリング方式 / レポート変換方式）を定義し、実装コストとデータ品質の最適化に成功した。

## 2. 実装環境 [#environment]

* **Development Lang:** Java (OpenJDK 21)
* **Core Libraries:**
* `com.fasterxml.jackson`: JSON/YAML Processing
* `com.networknt:json-schema-validator`: Strict Schema Validation
* `org.apache.httpcomponents`: WebAPI Client


* **Target LLM:** `gpt-oss:120b` (via Open WebUI API)
* **Context Strategy:** 16k Context Window / Rolling Update

## 3. 成果物一覧 (Deliverables) [#deliverables]

### A. Java Components

| クラス名 | 機能概要 | 備考 |
| --- | --- | --- |
| **LlmServiceCore** | LLM API通信基盤 | 責任分界点としてロジックを持たず、純粋なRequest/Responseを担当。 |
| **NarrativeGeneratorService** | 雑談ログ用生成機 | 長文ログを分割し、前回の文脈を引き継ぐ「ローリング・メモリ方式」を実装。 |
| **JsonSchemaValidator** | 品質保証 | `networknt` ライブラリを用い、生成されたJSONが定義書に完全適合するか判定。 |
| **JsonToYamlConverter** | 最終出力変換 | Jackson YAMLを用い、標準仕様(Pure YAML)へ変換。独自記法を廃止し互換性を確保。 |

### B. Configuration Files

| ファイル名 | 用途 |
| --- | --- |
| **schema_narrative.json** | 雑談(Story)用スキーマ |
| **schema_tech.json** | 技術(Work)用スキーマ |

## 4. 確立されたアーキテクチャ [#architecture]

### Strategy A: Narrative Pipeline (雑談ログ向け)

文脈の連続性を重視し、分割されたログを再帰的に要約するアプローチ。

```mermaid
graph LR
    Log[Chat Log (Huge)] --> Split[Splitter (10k chars)]
    Split --> Loop{Rolling Loop}
    Loop -->|Chunk + Prev JSON| LLM[LLM (gpt-oss:120b)]
    LLM -->|New JSON| Loop
    Loop -->|Final JSON| Valid[Schema Validator]
    Valid -->|OK| Conv[YAML Converter]
    Conv -->|Save| File[MemoryBlock.yaml]

```

### Strategy B: Technical Pipeline (技術ログ向け)

情報の正確性を最優先し、人間が確定させた「技術レポート」をソースとするアプローチ。

```mermaid
graph LR
    Report[Tech Report.md] -->|Direct Input| LLM[LLM (gpt-oss:120b)]
    LLM -->|Extraction| JSON[JSON Data]
    JSON --> Valid[Schema Validator]
    Valid -->|OK| Conv[YAML Converter]
    Conv -->|Save| File[MemoryBlock.yaml]

```

## 5. 運用プロセスの変更 [#operation]

本システムの導入に伴い、Wiki運用（ディレクトリ構成）を以下のように見直す。

1. **ディレクトリ分離:** `Raw_Chat_Logs`（生ログ）と `Tech_Reports`（確定レポート）の格納場所を明確に分ける。
2. **確定フロー:**
* 作業完了後、人間（またはLLM）が `Tech_Report.md` を作成。
* システムが `Tech_Report.md` を読み込み、`MemoryBlock.yaml` を即時生成・格納。
* 生ログ（20万文字級）は「Phase 3: Child Memory」のソースとして別途保管。



## 6. 今後の展望 [#future]

* **Phase 1 (完了):** 親メモリ生成サービスのコンポーネント実装完了。
* **Phase 2 (Next):** Semantic Chunking Service（意味的な塊によるテキスト分割）の実装へ移行。
* **Wiki運用整備:** 新アーキテクチャに合わせたディレクトリ整理の実務作業。