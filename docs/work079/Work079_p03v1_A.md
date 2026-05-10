# 依頼：SceneService の要求理解・設計方針・Go/NoGo判断

あなたは Java の設計・実装支援者です。

今回は、SwitchBot制御用MCPサーバーで使用する `SceneService` を製造する前段階として、要求仕様を理解し、設計方針を整理してください。

ただし、この依頼では **コード変更・ファイル作成・実装は絶対に行わないでください**。

今回の目的は、いきなり製造することではなく、以下を確認することです。

- SceneService の責務を正しく理解できているか
- 既存クラスとの接続方針が妥当か
- キャッシュ設計が妥当か
- sceneKey による安全なシーン実行方針が妥当か
- 実装対象ファイル・クラス構成案が妥当か
- 製造に進んでよいか

---

## 作業対象ディレクトリ

Javaソースの作業対象は以下です。

```text
C:\Users\[user masked]\IdeaProjects\switchbot-ctrl-mcp\src\main\java\vr46\switchbotctrlmcp\switchbot
```

このディレクトリには、すでに以下のクラスがあります。

```text
AppConfig.java
SceneCatalog.java
SceneCatalogEntry.java
SceneCatalogException.java
SceneSummary.java
SwitchBotScenesConverter.java
SwitchBotScenesConvertException.java
SwitchBotScenesResponse.java
```

また、共通ライブラリ側には以下があります。

```text
switchBot.SwitchBotApisV2
switchBot.SwitchBotApiResult
switchBot.SwitchBotApiException
```

---

## 既存クラスの役割

### SwitchBotApisV2

SwitchBot Cloud API v1.1 を呼び出す低レイヤーAPIクライアントです。

主な機能：

```java
getScenes()
executeScene(String sceneId)
```

`getScenes()` は `SwitchBotApiResult` を返します。
raw JSON は `SwitchBotApiResult.body()` から取得します。

`executeScene(String sceneId)` も `SwitchBotApiResult` を返します。

HTTP 4xx / 5xx は例外ではなく `SwitchBotApiResult` として返す方針です。
通信不能・署名失敗・URI生成失敗・割り込みなどは `SwitchBotApiException` になります。

### SceneCatalog

`src/main/resources/sceneCatalog.json` を classpath resource として読み込み、`List<SceneCatalogEntry>` を返します。

SceneCatalog は以下を行います。

* sceneCatalog.json の読み込み
* 必須項目検証
* sceneKey重複検出
* sceneId重複検出

SceneCatalog は SwitchBot API を呼びません。

### SwitchBotScenesConverter

`SwitchBotApisV2.getScenes()` の raw JSON と `SceneCatalogEntry` 一覧を受け取り、`List<SceneSummary>` を作ります。

変換条件：

```text
SwitchBot APIのgetScenesに存在する
かつ
sceneCatalog.jsonにも存在する
かつ
sceneId が一致する
かつ
sceneCatalog.json側の executable が true
```

この条件を満たすものだけを `SceneSummary` として出力します。

つまり、SwitchBotアプリ側でシーンを追加しても、sceneCatalog.jsonに登録されない限り、LLM / MCPには公開されません。

---

## 今回作るもの

今回作る予定の中心は `SceneService` です。

SceneService は、これまで作成した部品を束ね、MCP Tool から使いやすい運用単位を提供する層です。

主な責務は以下です。

```text
1. シーン一覧を取得する
2. シーン一覧をキャッシュする
3. 手動更新用のrefreshを提供する
4. sceneKeyでシーンを実行する
5. 実行結果を専用結果型で返す
```

---

## SceneService の要求仕様

### 1. シーン一覧取得

SceneService は、以下の流れで `List<SceneSummary>` を返します。

```text
SwitchBotApisV2.getScenes()
  ↓
SceneCatalog.loadEntries()
  ↓
SwitchBotScenesConverter.convert(rawJson, catalogEntries)
  ↓
List<SceneSummary>
```

public API案：

```java
public List<SceneSummary> listScenes()
```

`listScenes()` はキャッシュが有効な場合、キャッシュ済み `List<SceneSummary>` を返します。

キャッシュが存在しない、または期限切れの場合は、SwitchBot APIとSceneCatalogを読み直して再変換します。

---

### 2. キャッシュ対象

キャッシュ対象は `SwitchBotApiResult` や raw JSON ではなく、変換済みの `List<SceneSummary>` としてください。

理由：

* MCP Tool / 上位サービスが欲しいのは raw JSON ではなく SceneSummary
* SceneSummary はLLM公開用DTOとして整理済み
* raw JSONや管理用noteを上位に漏らさないため

---

### 3. キャッシュTTL

Ver.1ではキャッシュTTLはクラス定数で構いません。

```text
24時間
```

理由：

* 今回のMCPサーバーの本質的要件ではない
* sceneCatalog.jsonを更新しない限り新シーンは公開されない
* Ver.1ではシンプルさを優先する

Durationを外部設定化する必要はありません。

ただし、設計上あまり不自然でなければ、private static final などで一箇所にまとめてください。

---

### 4. キャッシュ有効判定

キャッシュ有効判定は以下の考え方にしてください。

```text
cachedScenes != null
かつ
cachedAt != null
かつ
現在時刻が cachedAt + cacheTtl より前
```

重要：

* `or` ではなく `and` です。
* 期限切れキャッシュを返さないようにしてください。

---

### 5. 手動更新用 escape hatch

手動更新用の refresh メソッドを用意してください。

public API案：

```java
public List<SceneSummary> refreshScenes()
```

`refreshScenes()` はキャッシュを無視して、以下を再実行します。

```text
SwitchBotApisV2.getScenes()
SceneCatalog.loadEntries()
SwitchBotScenesConverter.convert(...)
```

その結果をキャッシュへ保存して返します。

MCP Toolとして公開するかどうかは後工程で決めます。
SceneServiceには実装しておいてください。

---

### 6. シーン実行

シーン実行は `sceneKey` で受け取ります。

public API案：

```java
public SceneExecutionResult executeBySceneKey(String sceneKey)
```

重要：

* `sceneId` を直接受け取らないでください。
* `sceneCatalog.json` 単体から sceneKey → sceneId 変換しないでください。
* `listScenes()` の結果から sceneKey を探してください。

理由：

`listScenes()` の結果は、以下を満たす安全な公開済みシーンだけです。

```text
SwitchBot APIに存在する
かつ
sceneCatalog.jsonにも存在する
かつ
executable=true
```

Catalog単体から変換すると、SwitchBot側に存在しないシーンを実行しようとする可能性があります。

---

### 7. sceneKey 解決方針

`executeBySceneKey(sceneKey)` は以下の流れにしてください。

```text
sceneKey null / blank チェック
  ↓
listScenes()
  ↓
sceneKey一致のSceneSummaryを探す
  ↓
見つからなければ SceneServiceException
  ↓
executable=falseなら SceneServiceException
  ↓
SceneSummary.sceneIdで SwitchBotApisV2.executeScene(sceneId)
  ↓
SceneExecutionResult を返す
```

なお、現在の `listScenes()` は executable=true のものだけ返す想定ですが、将来変更に備えて `executeBySceneKey` 側でも `executable` を確認してください。

---

## SceneExecutionResult

シーン実行の戻り値は文字列ではなく、専用結果型にしてください。

作成候補：

```text
SceneExecutionResult.java
```

持たせる項目：

```text
sceneKey
sceneId
sceneName
success
httpStatusCode
switchBotBody
message
```

方針：

* `success` は `SwitchBotApiResult.success()` を反映する
* `httpStatusCode` は `SwitchBotApiResult.statusCode()` を反映する
* `switchBotBody` は `SwitchBotApiResult.body()` を保持する
* `message` はSceneService側の人間向け/上位向けの短い説明でよい
* MCP Toolで最終的に文字列化またはJSON化するのは後工程

SceneServiceの段階では、実行結果を構造化して返してください。

---

## SceneServiceException

SceneServiceの業務的な失敗は `SceneServiceException` に集約してください。

作成候補：

```text
SceneServiceException.java
```

SceneServiceExceptionにするもの：

* sceneKey が null
* sceneKey が blank
* sceneKey が公開シーン一覧に存在しない
* 対象SceneSummaryが executable=false
* listScenes / refreshScenes 中に発生したエラーをSceneServiceとして包みたい場合

ただし、SwitchBot APIのHTTP 4xx / 5xx は、`SceneExecutionResult.success=false` として返す方針です。
HTTPレスポンスが返っている場合は、原則としてSceneServiceExceptionにしないでください。

通信不能などで `SwitchBotApisV2` が例外を投げた場合は、SceneServiceExceptionに包むか、そのまま上げるか、方針を提案してください。

---

## 今回やらないこと

以下は今回の対象外です。

* MCP Tool定義
* SceneServiceをMCPに接続すること
* ユーザー発話とaliasのマッチング
* LLMによる意図解釈
* cooldown
* dry-run
* 監査ログ
* SceneCatalogのDB化
* キャッシュTTLの外部設定化
* SwitchBotApisV2の修正
* sceneCatalog.jsonの構造変更

今回は、以下に集中してください。

```text
SceneService
+
SceneExecutionResult
+
SceneServiceException
```

---

## 確認してほしいこと

### 1. 既存クラスとの接続方針

以下の接続が自然か確認してください。

```text
SceneService
  uses SwitchBotApisV2
  uses SceneCatalog
  uses SwitchBotScenesConverter
```

### 2. コンストラクタ設計

SceneServiceの生成方法を提案してください。

候補：

```java
public SceneService(SwitchBotApisV2 switchBotApisV2)
```

または、

```java
public SceneService(
    SwitchBotApisV2 switchBotApisV2,
    SceneCatalog sceneCatalog,
    SwitchBotScenesConverter converter
)
```

テスト容易性とVer.1のシンプルさのバランスを考えて提案してください。

ただし、SceneService内部で AppConfig.load() して token/secret を読む設計は避けたいです。
認証情報の読み込みは外側レイヤーの責務です。

### 3. キャッシュのスレッド安全性

MCPサーバーやServletから複数リクエストで呼ばれる可能性があります。

Ver.1として、どの程度のスレッド安全性を持たせるべきか提案してください。

候補：

* synchronizedで十分
* private lock object を使う
* AtomicReferenceなどは過剰
* 完全に非同期キャッシュは不要

### 4. キャッシュの戻り値

キャッシュした `List<SceneSummary>` は外部から変更されないようにしてください。

候補：

* `List.copyOf(...)`
* SceneSummary が record なら基本的に安全
* aliases も `List.copyOf(...)` されているか確認

### 5. エラー方針

以下を整理してください。

* SceneCatalogException が出た場合
* SwitchBotScenesConvertException が出た場合
* SwitchBotApiException が出た場合
* SwitchBot API HTTP 4xx/5xx が返った場合
* sceneKey未登録の場合
* executable=false の場合

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
* MCP ToolやSceneService以外の設計に話を広げすぎないこと
* AppConfigやSwitchBotApisV2を変更しないこと
* sceneCatalog.jsonを変更しないこと

---

## 出力形式

以下のMarkdown形式で回答してください。

```markdown
# SceneService 設計前レポート

## 1. 要求仕様の理解

## 2. SceneService の責務

## 3. 既存クラスとの接続方針

## 4. public API案

## 5. SceneExecutionResult の仕様案

## 6. SceneServiceException の仕様案

## 7. キャッシュ設計

## 8. スレッド安全性の方針

## 9. sceneKey実行方針

## 10. エラー方針

## 11. 今回やらないこと

## 12. Go / NoGo 判断

## 13. 推奨する次の作業
```

この形式に従って、まず分析だけを行ってください。