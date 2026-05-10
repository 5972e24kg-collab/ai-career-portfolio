# 依頼：SwitchBot SceneCatalog / SceneSummary / ScenesConverter の要求理解・設計方針・Go/NoGo判断

あなたは Java の設計・実装支援者です。

今回は、SwitchBot制御用MCPサーバーで使用する以下の3要素を設計・製造する前段階として、要求仕様を理解し、コーディング方針を整理してください。

ただし、この依頼では **コード変更・ファイル作成・実装は絶対に行わないでください**。

今回の目的は、いきなり製造することではなく、以下を確認することです。

- 要求仕様を正しく理解できているか
- クラス責務の分離が妥当か
- 実装対象ファイル・構成案が妥当か
- 依存ライブラリや読み込み方式に問題がないか
- 製造に進んでよいか

---

## 作業ディレクトリ

Javaソースの作業対象は以下です。

```text
C:\Users\sysop\IdeaProjects\switchbot-ctrl-mcp\src\main\java\vr46\switchbotctrlmcp\switchbot
```

このディレクトリは空です。
後続の製造フェーズでは、必要なクラスを自由に作成して構いません。

ただし、今回の分析フェーズでは作成しないでください。

---

## resources

以下のファイルを用意済みです。

```text
C:\Users\sysop\IdeaProjects\switchbot-ctrl-mcp\src\main\resources\sceneCatalog.json
```

このJSONファイルには、人間が管理する公開用シーン定義が入っています。

想定形式は以下です。

```json
[
  {
    "sceneKey": "light_scene_full",
    "sceneId": "[sceneId masked 01]",
    "sceneName": "照明系シーンA",
    "publicName": "照明系を明るくする",
    "category": "light",
    "action": "set_mode",
    "riskLevel": "safe",
    "executable": true,
    "aliases": ["照明を明るくして"],
    "note": ""
  },
  {
    "sceneKey": "light_scene_standard",
    "sceneId": "[sceneId masked 02]",
    "sceneName": "照明系シーンB",
    "publicName": "照明系を標準状態にする",
    "category": "light",
    "action": "turn_on",
    "riskLevel": "safe",
    "executable": true,
    "aliases": ["照明をつけて", "照明を標準状態にして"],
    "note": "標準シーンとして管理"
  },
  {
    "sceneKey": "light_scene_dark",
    "sceneId": "[sceneId masked 03]",
    "sceneName": "照明系シーンC",
    "publicName": "照明系を暗めにする",
    "category": "light",
    "action": "set_mode",
    "riskLevel": "safe",
    "executable": true,
    "aliases": ["照明を暗くして"],
    "note": ""
  },
  {
    "sceneKey": "light_scene_off",
    "sceneId": "[sceneId masked 04]",
    "sceneName": "照明系シーンD",
    "publicName": "照明系をオフにする",
    "category": "light",
    "action": "turn_off",
    "riskLevel": "safe",
    "executable": true,
    "aliases": ["照明を消して"],
    "note": ""
  }
]
```

実際の `sceneCatalog.json` を読み、形式が上記方針と一致しているか確認してください。

---

## SwitchBot getScenes のraw JSON

SwitchBot API の `getScenes()` は、以下のようなraw JSONを返します。

```json
{
  "statusCode": 100,
  "body": [
    {
      "sceneId": "[sceneId masked 01]",
      "sceneName": "照明系シーンA"
    },
    {
      "sceneId": "[sceneId masked 02]",
      "sceneName": "照明系シーンB"
    },
    {
      "sceneId": "[sceneId masked 03]",
      "sceneName": "照明系シーンC"
    },
    {
      "sceneId": "[sceneId masked 04]",
      "sceneName": "照明系シーンD"
    }
  ],
  "message": "success"
}
```

実際には、Catalogに登録していないSwitchBotシーンも多数含まれます。

---

## 確定済み方針

以下の設計方針で確定しています。

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

## 重要な設計意図

SwitchBotアプリ側で新しいシーンを追加しても、MCPサーバーに自動公開しないことが重要です。

そのため、`SwitchBotScenesConverter` は以下のinner join方針にしてください。

```text
SwitchBot APIのgetScenesに存在する
かつ
sceneCatalog.jsonにも存在する
かつ
executable=true
```

この条件を満たすものだけを `SceneSummary` として出力する方針です。

これにより、SwitchBot側に追加されただけのシーンはLLMには見えません。

---

## 今回作りたい概念

### 1. SceneCatalog

`sceneCatalog.json` を読み込み、人間が管理する公開用シーン定義を返す役割です。

このレイヤーでは、以下のような管理用情報を持ってよいです。

* sceneKey
* sceneId
* sceneName
* publicName
* category
* action
* riskLevel
* executable
* aliases
* note

将来的にはDBや管理画面に移行する可能性があります。
そのため、resourcesから読むか、DBから読むかは `SceneCatalog` の内部に閉じ込める方針です。

### 2. SceneSummary

LLM / MCP / 上位サービスに正式公開するDTOです。

`SceneCatalog` から管理用情報を落としたものです。

基本項目は以下を想定しています。

* sceneKey
* sceneId
* sceneName
* publicName
* category
* action
* riskLevel
* executable
* aliases

`note` は含めないでください。

### 3. SwitchBotScenesConverter

SwitchBot API の `getScenes()` raw JSON と、SceneCatalog の定義を突合し、公開可能な `SceneSummary` 一覧へ変換する役割です。

方針は以下です。

* join key は `sceneId`
* SwitchBot APIに存在しないCatalog定義は出力しない
* Catalogに存在しないSwitchBotシーンは出力しない
* `executable=false` のCatalog定義は、LLM公開用一覧には出力しない
* `sceneName` は基本的にSwitchBot API側の実名を優先するか、Catalog側と一致確認する
* `note` はSceneSummaryへ渡さない

---

## 今回やらないこと

以下は今回の対象外です。

* SwitchBot APIを実際に呼び出すこと
* SwitchBotApisV2の修正
* MCP Tool定義
* SceneServiceのキャッシュ実装
* executeSceneの実行制御
* cooldown
* dry-run
* 監査ログ
* ユーザー発話とaliasのマッチング実装
* LLMによる意図解釈
* シーン実行APIの実装

今回は、あくまで以下の設計に集中してください。

```text
sceneCatalog.json
+
getScenes raw JSON
↓
SceneSummary一覧
```

---

## 確認してほしいこと

### 1. 既存プロジェクト構成

以下を確認してください。

* build.gradle または pom.xml の有無
* JacksonなどJSONパース用ライブラリが利用可能か
* Javaバージョン
* package命名規則
* resources読み込み方法

ただし、確認のみで、変更はしないでください。

### 2. sceneCatalog.json の形式

以下を確認してください。

* JSON配列であること
* 必須項目が揃っていること
* aliases が配列であること
* executable が boolean であること
* sceneId が重複していないこと
* sceneKey が重複していないこと
* `note` は存在してもよいが、SceneSummaryには含めない方針でよいこと

### 3. SwitchBot getScenes raw JSONのパース方針

以下を整理してください。

* `statusCode`
* `body`
* `message`
* body配列内の `sceneId`
* body配列内の `sceneName`

SwitchBot API側の `statusCode` が100以外の場合の扱いも方針だけ提案してください。

### 4. inner join方針

以下を確認してください。

* `sceneId` で突合する
* Catalogに存在しないSwitchBotシーンは落とす
* SwitchBot APIに存在しないCatalog定義は落とす
* `executable=false` は出力しない
* `sceneName` 不一致時の扱いをどうするか提案する

### 5. クラス構成案

後続の製造フェーズで作るクラス案を提案してください。

候補は以下です。

```text
SceneCatalog.java
SceneCatalogEntry.java
SceneSummary.java
Scenes.java
SwitchBotScenesConverter.java
```

クラス数を減らすべきなら、その理由も含めて提案してください。

### 6. 例外・エラー方針

以下を提案してください。

* sceneCatalog.json が存在しない場合
* sceneCatalog.json のJSON形式が壊れている場合
* 必須項目が欠落している場合
* sceneKey / sceneId が重複している場合
* getScenes raw JSON が壊れている場合
* SwitchBot API側のstatusCodeが100以外の場合
* sceneNameがCatalogとAPIで一致しない場合

この段階では実装しないでください。方針だけでよいです。

---

## Go / NoGo 判断

最後に、次の形式で判断してください。

```text
判断: Go / 条件付きGo / NoGo

理由:
- ...

製造前に確認すべきこと:
- ...

推奨する次の作業:
- ...
```

---

## 重要な制約

* この依頼ではコード変更しないこと
* 新規ファイルを作成しないこと
* 実装コードを出力しないこと
* まだ製造に進まないこと
* SceneServiceやMCP Toolの実装に話を広げすぎないこと
* 今回は SceneCatalog / SceneSummary / SwitchBotScenesConverter の設計方針に集中すること

---

## 出力形式

以下のMarkdown形式で回答してください。

```markdown
# SceneCatalog / SceneSummary / SwitchBotScenesConverter 設計前レポート

## 1. 要求仕様の理解

## 2. プロジェクト構成・依存関係の確認

## 3. sceneCatalog.json の確認観点

## 4. getScenes raw JSON のパース方針

## 5. inner join方針

## 6. クラス構成案

## 7. SceneCatalog の責務

## 8. SceneSummary の責務

## 9. SwitchBotScenesConverter の責務

## 10. 例外・エラー方針

## 11. 今回やらないこと

## 12. Go / NoGo 判断

## 13. 推奨する次の作業
```

この形式に従って、まず分析だけを行ってください。