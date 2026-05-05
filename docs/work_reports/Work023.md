# Work023: Parent Memory Block Service実装完了

Tag: [[Java]], [[LLM]], [[RAG]], [[ETL]], [[Oracle]]

## 1. 目的

* **RAGデータの純度向上:** 雑談ログ、技術レポート、一般資料を、LLMが理解しやすい「親メモリブロック（構造化YAML）」へ自動変換する。
* **堅牢なパイプライン構築:** LLM特有の出力揺らぎ（ハルシネーションやフォーマット崩れ）を、厳格なスキーマバリデーションによって排除する。
* **人機一体の運用:** システムによる自動生成と、人間による手動作成（Override）が競合しない運用フローを確立する。

## 2. システム環境

* 
**Node:** Node B (EVO-X2 / Ubuntu Native) 


* **Container:** Java App Container (OpenJDK 21)
* **Database:** Oracle (File Logic & State Management)
* **Target:** `[Path masked]` (MkDocs Markdown Files)

## 3. 作業ログ・解決プロセス

### 確立されたアーキテクチャ

* **Hybrid ETL Pipeline:**
* **Logic (Oracle):** ディレクトリ巡回、対象ファイルの選別、差分検知、ユーザー作成済みファイルの保護（Skipロジック）を担当。
* **Generation (Java + LLM):** テキストの読解、JSON生成、バリデーション、YAML変換を担当。



### 発生した課題と解決策

1. **LLM出力の不安定性**
* **課題:** LLMに直接YAMLを書かせると、インデントずれや配列の閉じ忘れが発生する。
* 
**解決策:** 中間フォーマットとして「JSON」を採用し、`JsonSchemaValidator` で厳格に型チェックを実施。適合したもののみを `JsonToYamlConverter` で標準YAMLへ変換するフローを確立。




2. **データ特性による手法の乖離**
* **課題:** 「雑談（時系列）」と「技術レポート（事実）」では、最適な要約手法が異なる。
* **解決策:**
* **Narrative (雑談):** ローリング・メモリ方式（再帰的要約）を採用し、文脈を維持。
* **Technical / General:** レポート変換方式（One-Shot）を採用し、事実の正確性を優先。




3. **ネットワーク図の構造化失敗**
* **課題:** ネットワーク図をフラットなリストに分解すると、トポロジー（階層構造）が失われる。
* **解決策:** 「セグメント単位」でエンティティをグルーピングするスキーマ運用に変更し、YAML上でもネットワーク階層を表現可能にした。


4. **自動生成と手動運用の競合**
* **課題:** LLMが生成できない複雑な構成図などを、人間が手動で書きたい場合に上書きされる恐れがある。
* **解決策:** Oracle側のロジックで「ユーザー定義のメモリブロックが存在する場合は自動生成をスキップする」仕様を実装。



## 4. 成果物

### Java Components

* `LlmServiceCore`: API通信とトークン管理。
* `NarrativeGeneratorService`: ローリングメモリ生成ロジック。
* `JsonSchemaValidator`: `networknt` ライブラリを用いたスキーマ検証。
* `JsonToYamlConverter`: `Jackson` を用いた標準YAML変換。

### Job & Configuration

* `CreateMemoryBlockJob`: Oracleと連携してパイプラインを駆動する統合ジョブ。
* **Schemas:**
* `schema_narrative.json`: 雑談・ストーリー用
* `schema_tech.json`: 技術レポート用
* `schema_others.json`: 一般資料・スペック表用



### Result (Log Evidence)

```text
13:12:41.201 ... checked Json schema
13:12:41.211 ... convert YAML @ 3775
13:12:41.238 ... ★ output MemoryBlock File @ ai-routine-tasks.md

```

フォーマット崩れなしの完全なYAMLファイルが、Wikiディレクトリに自動生成されることを確認。