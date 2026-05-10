# 依頼：SceneCatalog / SceneSummary / SwitchBotScenesConverter の製造

あなたは Java の設計・実装支援者です。

前工程で、SceneCatalog / SceneSummary / SwitchBotScenesConverter の要求理解と設計方針確認は完了しています。

今回は、その確定方針に基づき、実際にコードを作成してください。

この依頼では **コード変更・ファイル作成を行ってよい** です。

---

## 作業対象ディレクトリ

Javaソースの作業対象は以下です。

```text
[local project path masked]\src\main\java\vr46\switchbotctrlmcp\switchbot
```

このディレクトリ配下は空なので、必要なクラスを新規作成して構いません。

---

## resources

以下のファイルは作成済みです。

```text
[local project path masked]\src\main\resources\sceneCatalog.json
```

このファイルは、classpath resource として読み込んでください。

絶対パス依存にしないでください。

UTF-8 / Jackson / classpath InputStream で読み込む方針にしてください。

---

## package

作成するJavaクラスのpackageは以下にしてください。

```java
package vr46.switchbotctrlmcp.switchbot;
```

---

## 確定済み方針

以下の方針で確定しています。

```text
SceneCatalog:
  resources JSONから公開用シーン定義を読み込む。
  管理用noteなどを含めてよい。

SceneSummary:
  LLM / MCPに正式公開するDTO。
  noteなど管理用情報は含めない。

SwitchBotScenesConverter:
  SwitchBotApisV2.getScenes() のraw JSONとSceneCatalogをsceneIdでinner joinする。
  両方に存在するシーンだけSceneSummary化する。
  SwitchBot側に追加されただけのシーンはLLMへ露出しない。
```

---

## 重要な安全設計

SwitchBotアプリ側でシーンを追加しても、MCPサーバーに自動公開しないことが重要です。

そのため、変換条件は以下です。

```text
SwitchBot APIのgetScenesに存在する
かつ
sceneCatalog.jsonにも存在する
かつ
sceneId が一致する
かつ
sceneCatalog.json 側の executable が true
```

この条件を満たすものだけを `SceneSummary` として出力してください。

---

## 作成するクラス

原則として、以下のクラスを作成してください。

```text
SceneCatalog.java
SceneCatalogEntry.java
SceneSummary.java
SwitchBotScenesResponse.java
SwitchBotScenesConverter.java
```

必要であれば、例外クラスを追加して構いません。

例：

```text
SceneCatalogException.java
SwitchBotScenesConvertException.java
```

ただし、無駄にクラス数を増やさないでください。

---

## 1. SceneCatalogEntry

`sceneCatalog.json` の1件分を表す内部管理用DTOです。

含める項目は以下です。

```text
sceneKey
sceneId
sceneName
publicName
category
action
riskLevel
executable
aliases
note
```

方針：

* Jacksonで読み込める形にしてください。
* `note` は含めてよいです。
* `aliases` は null にならないようにしてください。
* DTOとして扱いやすい実装にしてください。
* Java record でも通常classでも構いません。
* 既存のプロジェクトの作風に合わせてください。

---

## 2. SceneSummary

LLM / MCP / 上位サービスに正式公開するDTOです。

含める項目は以下です。

```text
sceneKey
sceneId
sceneName
publicName
category
action
riskLevel
executable
aliases
```

重要：

* `note` は含めないでください。
* `sceneName` は、SwitchBot API側の実名を優先してください。
* `aliases` は null にならないようにしてください。
* JacksonでJSON出力しやすい形にしてください。

---

## 3. SwitchBotScenesResponse

SwitchBot API `getScenes()` の raw JSON body を受けるDTOです。

raw JSON の想定構造は以下です。

```json
{
  "statusCode": 100,
  "body": [
    {
      "sceneId": "[sceneId masked]",
      "sceneName": "照明系シーン"
    }
  ],
  "message": "success"
}
```

作成方針：

* クラス名は `SwitchBotScenesResponse` にしてください。
* `statusCode`
* `message`
* `body`
* body内要素として `sceneId`, `sceneName`
* Jacksonでパースできる形にしてください。
* `body` が null の場合に扱いやすいようにしてください。

---

## 4. SceneCatalog

`sceneCatalog.json` を読み込み、検証済みの `SceneCatalogEntry` 一覧を返す役割です。

責務：

* classpath上の `sceneCatalog.json` を読み込む
* JSON配列を `SceneCatalogEntry` のリストへ変換する
* 必須項目を検証する
* sceneKey の重複を検出する
* sceneId の重複を検出する
* aliases が null の場合は空リストへ補正するか、検証エラーにする
* 検証済み一覧を返す

読み込み方式：

```text
src/main/resources/sceneCatalog.json
↓
classpath resource として読み込み
```

絶対パスで読まないでください。

SceneCatalogがやらないこと：

* SwitchBot APIを呼ばない
* SwitchBotApisV2を生成しない
* SceneSummaryを作らない
* executeSceneを呼ばない
* aliasでユーザー発話を解釈しない
* MCP Toolを作らない

---

## 5. SwitchBotScenesConverter

`SwitchBot API getScenes raw JSON` と `SceneCatalogEntry一覧` を突合し、`SceneSummary` 一覧を作る役割です。

入力候補：

```text
String rawJson
List<SceneCatalogEntry> catalogEntries
```

または、設計上自然であれば、

```text
SwitchBotScenesResponse response
List<SceneCatalogEntry> catalogEntries
```

でも構いません。

ただし、Converter自身が `sceneCatalog.json` を直接読まないようにしてください。

責務：

* getScenes raw JSON をパースする
* SwitchBot API JSON内の `statusCode` を確認する
* `statusCode != 100` の場合は失敗扱いにする
* `body` 配列から `sceneId` / `sceneName` を取り出す
* `sceneId` でCatalogとinner joinする
* Catalogに存在しないSwitchBotシーンは出力しない
* SwitchBot APIに存在しないCatalog定義は出力しない
* Catalogの `executable=false` は出力しない
* `SceneSummary` を作成する
* `note` は出力しない
* 出力順はCatalogの順序を維持する

sceneName不一致時の方針：

* join可否は sceneId のみで判断してください。
* sceneName が不一致でも除外しないでください。
* 出力する sceneName は SwitchBot API側の実名を優先してください。
* 不一致は将来の診断対象とします。
* 今回は `SceneSummary` に note や診断用メッセージを入れないでください。
* 必要ならコードコメントで「sceneName mismatch はAPI名優先」と分かるようにしてください。

---

## 6. エラー方針

以下は失敗扱いにしてください。

### SceneCatalog側

* `sceneCatalog.json` が存在しない
* JSON形式が壊れている
* JSON配列ではない
* 必須項目が欠落している
* 必須項目が blank
* sceneKey が重複している
* sceneId が重複している
* aliases が配列ではない

### SwitchBotScenesConverter側

* getScenes raw JSON が壊れている
* SwitchBot API JSON内の `statusCode` が 100 ではない
* `body` が存在しない
* `body` が配列ではない
* body内の sceneId / sceneName が欠落している
* body内の sceneId / sceneName が blank

方針：

* 空リストで握りつぶさないでください。
* 「公開可能シーンがない」と「API異常 / JSON異常」は区別してください。
* RuntimeException系の専用例外を作っても構いません。
* 例外メッセージは英語でも日本語でも構いません。
* ただし、どの原因で失敗したか追えるメッセージにしてください。

---

## 7. 今回やらないこと

以下は今回の対象外です。

* SwitchBot APIを実際に呼び出すこと
* SwitchBotApisV2の修正
* AppConfigの修正
* MCP Tool定義
* SceneServiceのキャッシュ実装
* executeSceneの実行制御
* cooldown
* dry-run
* 監査ログ
* ユーザー発話とaliasのマッチング実装
* LLMによる意図解釈
* シーン実行APIの実装

今回は、あくまで以下に集中してください。

```text
sceneCatalog.json
+
getScenes raw JSON
↓
SceneSummary一覧
```

---

## 8. テストについて

IntelliJ IDEA上で単体確認する予定です。

そのため、以下は必須ではありません。

* mvnwのCLI動作確認
* OS環境変数の設定確認
* 実SwitchBot API呼び出し

ただし、可能であれば、製造後に以下を確認できる簡単なテストまたは確認用コードを作れる構成にしてください。

確認観点：

* SceneCatalog 正常読み込み
* sceneKey重複検出
* sceneId重複検出
* API側にだけあるシーンを除外
* Catalog側にだけあるシーンを除外
* executable=false を除外
* statusCode != 100 をエラー
* sceneName 不一致時にAPI名優先で出力

テストファイルを作るかどうかは、既存プロジェクトの構成を見て判断してください。

---

## 9. 実装後に確認してほしいこと

作業後、以下を確認してください。

* package が `vr46.switchbotctrlmcp.switchbot` になっていること
* `sceneCatalog.json` を絶対パスで読んでいないこと
* `note` が `SceneSummary` に含まれていないこと
* `SwitchBotScenesResponse` という名前になっていること
* `sceneId` inner joinになっていること
* `executable=false` が出力されないこと
* SwitchBot側にしかないシーンが出力されないこと
* Catalog側にしかないシーンが出力されないこと
* sceneName不一致時はAPI名優先になっていること
* SceneCatalog が SwitchBot APIを呼ばないこと
* Converter が resources を直接読まないこと
* MCP ToolやSceneServiceを実装していないこと

---

## 10. 出力してほしい報告形式

作業後、以下の形式で報告してください。

```markdown
# SceneCatalog / SceneSummary / SwitchBotScenesConverter 製造結果

## 1. 作成・変更したファイル

## 2. 実装したクラス構成

## 3. SceneCatalog の仕様

## 4. SceneSummary の仕様

## 5. SwitchBotScenesResponse の仕様

## 6. SwitchBotScenesConverter の仕様

## 7. エラー・例外方針

## 8. テスト・確認結果

## 9. 注意点・未確認事項
```

---

## 重要な制約

* 今回は製造フェーズなのでコード作成してよい
* ただし、MCP ToolやSceneServiceには進まないこと
* SwitchBot APIを実際に呼び出さないこと
* SwitchBotApisV2を修正しないこと
* sceneCatalog.jsonの構造を勝手に変更しないこと
* noteをSceneSummaryに含めないこと
* raw response DTO名は `SwitchBotScenesResponse` にすること
* sceneName不一致時は warning相当 + API名優先の方針にすること
* mvnwのCLI実行確認は必須ではない