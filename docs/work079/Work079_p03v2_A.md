# 依頼：SceneService 仕様追加後の再分析・設計方針・Go/NoGo判断

あなたは Java の設計・実装支援者です。

前回、SceneService の設計前分析を行いましたが、追加仕様が発生したため、今回はその内容を踏まえて再分析してください。

この依頼では **コード変更・ファイル作成・実装は絶対に行わないでください**。

今回の目的は、追加仕様を含めたうえで、以下を確認することです。

- cooldown / dry-run を SceneService に入れる方針が妥当か
- cooldown 追加に伴い、既存クラスへどのような変更が必要か
- SceneExecutionResult の追加項目が妥当か
- 既存責務分離が崩れていないか
- 製造に進んでよいか

---

## 作業対象ディレクトリ

Javaソースの作業対象は以下です。

```text
[local project path masked]\src\main\java\vr46\switchbotctrlmcp\switchbot
```

既存クラスは以下です。

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

共通ライブラリ側には以下があります。

```text
switchBot.SwitchBotApisV2
switchBot.SwitchBotApiResult
switchBot.SwitchBotApiException
```

---

## 既存設計の前提

### SceneCatalog

`src/main/resources/sceneCatalog.json` を classpath resource として読み込み、`List<SceneCatalogEntry>` を返します。

### SwitchBotScenesConverter

`SwitchBotApisV2.getScenes()` の raw JSON と `SceneCatalogEntry` 一覧を受け取り、`List<SceneSummary>` を作ります。

変換条件は以下です。

```text
SwitchBot APIのgetScenesに存在する
かつ
sceneCatalog.jsonにも存在する
かつ
sceneId が一致する
かつ
sceneCatalog.json側の executable が true
```

SwitchBot側に追加しただけのシーンは、LLM / MCPには公開されません。

### SceneService

前回までの要求では、SceneService は以下を担当する予定でした。

```text
- List<SceneSummary> の取得
- List<SceneSummary> の24時間キャッシュ
- refreshScenes()
- sceneKey指定によるシーン実行
- SceneExecutionResult の返却
```

---

# 今回の追加仕様

## 1. cooldown の追加

同じ `sceneKey` については cooldown を設定します。

cooldown は `sceneCatalog.json` に項目を追加して管理します。
単位は秒です。

```json
{
  "sceneKey": "lighting_scene_full",
  "sceneId": "[sceneId masked]",
  "sceneName": "照明系シーンA",
  "publicName": "照明系の状態を変更する",
  "category": "light",
  "action": "set_mode",
  "riskLevel": "safe",
  "executable": true,
  "aliases": ["照明系を変更して"],
  "cooldown": 5,
  "note": ""
}
```

cooldown が未設定の場合は、デフォルトで **10秒** としてください。

### 重要

cooldown は管理メモではなく、実行制御に必要な正式な運用メタ情報です。

そのため、以下の変更が必要になる想定です。

```text
SceneCatalogEntry:
  cooldown を追加する

SceneSummary:
  cooldown を追加する

SwitchBotScenesConverter:
  SceneCatalogEntry.cooldown を SceneSummary.cooldown に渡す
  cooldown未設定なら10秒を補完する

SceneService:
  SceneSummary.cooldown を使って sceneKey 単位の cooldown 制御を行う
```

この方針が妥当か確認してください。

---

## 2. cooldown の目的

cooldown の目的は、LLMやMCP Toolからの誤連打・連続実行を防ぐことです。

例：

```text
lighting_scene_off を実行
↓
1秒以内に lighting_scene_off を再実行
↓
cooldown中なので SwitchBot API executeScene は呼ばない
```

cooldown は `sceneKey` 単位で管理してください。

`sceneId` 単位ではなく、上位公開キーである `sceneKey` 単位を基本とします。

---

## 3. cooldown 中の戻り値

cooldown中は例外ではなく、`SceneExecutionResult` を返してください。

理由：

* cooldownは異常ではなく、安全制御が働いた結果である
* MCP Tool側でユーザーに説明しやすい
* SwitchBot APIのHTTP 4xx/5xxと同様、結果型で扱える

cooldown中の想定：

```text
success=false
executed=false
dryRun=false
httpStatusCode=0
switchBotBody=""
message="Cooldown active."
```

message の文言は実装側で適切にして構いません。

---

## 4. dry-run の追加

dry-run は **SceneService全体のモード** として持たせます。

今回は以下の案を採用します。

```text
案A: SceneService生成時に dryRun モードを固定する
```

つまり、SceneService生成時に dry-run を指定できるようにします。

例：

```java
new SceneService(switchBotApisV2, true);
```

または、明示DI用コンストラクタで、

```java
new SceneService(switchBotApisV2, sceneCatalog, converter, true);
```

のように指定できる設計を検討してください。

dry-run=true の場合、`executeBySceneKey(sceneKey)` は対象シーンを解決しますが、`SwitchBotApisV2.executeScene(sceneId)` は呼びません。

dry-run時の想定：

```text
success=true
executed=false
dryRun=true
httpStatusCode=0
switchBotBody=""
message="Dry-run: scene execution was skipped."
```

---

## 5. SceneExecutionResult の追加仕様

SceneExecutionResult は以下の項目を持つ専用結果型にしてください。

```text
sceneKey
sceneId
sceneName
success
httpStatusCode
switchBotBody
message
executed
dryRun
```

意味は以下です。

```text
success:
  処理として成功したか

executed:
  実際に SwitchBot API executeScene を呼んだか

dryRun:
  dry-run により実行しなかったか
```

### 実実行成功時

```text
success=true
executed=true
dryRun=false
httpStatusCode=SwitchBotApiResult.statusCode()
switchBotBody=SwitchBotApiResult.body()
```

### SwitchBot API HTTP 4xx/5xx 時

```text
success=false
executed=true
dryRun=false
httpStatusCode=SwitchBotApiResult.statusCode()
switchBotBody=SwitchBotApiResult.body()
```

### dry-run時

```text
success=true
executed=false
dryRun=true
httpStatusCode=0
switchBotBody=""
```

### cooldown中

```text
success=false
executed=false
dryRun=false
httpStatusCode=0
switchBotBody=""
```

---

## 6. SceneService の実行フロー

`executeBySceneKey(sceneKey)` は以下の流れにしてください。

```text
sceneKey null / blank チェック
  ↓
sceneKey を trim
  ↓
listScenes()
  ↓
sceneKey一致のSceneSummaryを探す
  ↓
見つからなければ SceneServiceException
  ↓
executable=falseなら SceneServiceException
  ↓
cooldown中なら SceneExecutionResult(success=false, executed=false)
  ↓
dryRun=trueなら SceneExecutionResult(success=true, executed=false, dryRun=true)
  ↓
SwitchBotApisV2.executeScene(sceneId)
  ↓
SceneExecutionResult を返す
```

注意：

* `sceneId` を直接受け取らない
* `sceneCatalog.json` 単体から sceneKey → sceneId 変換しない
* `listScenes()` の結果から sceneKey を解決する
* alias / 自然言語マッチングは今回やらない
* sceneKey は trim 後の完全一致
* 大文字小文字のゆらぎ吸収はしない

---

## 7. cooldownの記録タイミング

以下の方針を提案・検討してください。

推奨案：

```text
SwitchBot API executeScene を実際に呼び出した場合のみ、lastExecutedAt を更新する
```

つまり、

```text
dry-run時:
  lastExecutedAt を更新しない

cooldownで拒否された時:
  lastExecutedAt を更新しない

SwitchBot APIを呼び出した時:
  HTTP成功/失敗に関わらず lastExecutedAt を更新する
```

理由：

* dry-runは物理実行していないため cooldown を消費しない
* cooldown拒否で時刻更新すると、永久に延長される可能性がある
* HTTP 4xx/5xxでも実行リクエスト自体は投げているため、短時間連打は防ぎたい

この方針が妥当か確認してください。

---

## 8. SceneServiceのキャッシュ仕様

前回方針を維持します。

### キャッシュ対象

```text
List<SceneSummary>
```

### TTL

```text
24時間
```

Ver.1ではクラス定数で構いません。

### 有効判定

```text
cachedScenes != null
かつ
cachedAt != null
かつ
現在時刻が cachedAt + cacheTtl より前
```

`or` ではなく `and` です。

### refreshScenes()

`refreshScenes()` はキャッシュを無視して再取得し、成功時のみキャッシュを更新します。

失敗時は既存キャッシュを壊さない方針が望ましいです。

---

## 9. スレッド安全性

MCPサーバーやServletから複数リクエストで呼ばれる可能性があります。

Ver.1では以下の方針を想定しています。

```text
private lock object で十分
AtomicReference や非同期更新は不要
```

対象：

* cachedScenes
* cachedAt
* sceneKeyごとのlastExecutedAt

注意：

* SwitchBot API executeScene 呼び出し中に長時間lockを保持しない方がよい
* cooldown判定とlastExecutedAt更新の競合に注意する

この観点で設計方針を提案してください。

---

## 10. エラー方針

### SceneServiceExceptionにするもの

```text
sceneKey null / blank
sceneKey未登録
executable=false
SceneCatalogException
SwitchBotScenesConvertException
SwitchBotApiException
getScenes() の HTTP 4xx/5xx
```

### SceneExecutionResult.success=falseで返すもの

```text
executeScene() の HTTP 4xx/5xx
cooldown中
```

### SceneExecutionResult.success=true / executed=falseで返すもの

```text
dry-run
```

---

## 11. 今回やらないこと

以下は今回の対象外です。

```text
MCP Tool定義
SceneServiceをMCPに接続すること
ユーザー発話とaliasのマッチング
LLMによる意図解釈
監査ログ
SceneCatalogのDB化
キャッシュTTLの外部設定化
SwitchBotApisV2の修正
sceneCatalog.jsonの構造変更以外の大幅変更
```

今回は、以下に集中します。

```text
既存クラスへのcooldown追加
SceneService
SceneExecutionResult
SceneServiceException
dry-run
cooldown
```

---

## 確認してほしいこと

### 1. 既存クラスへの変更範囲

cooldown追加に伴い、以下の変更が必要か確認してください。

```text
SceneCatalogEntry
SceneSummary
SwitchBotScenesConverter
SceneCatalog
sceneCatalog.json
```

### 2. コンストラクタ設計

dry-run追加後のSceneServiceコンストラクタ案を提案してください。

候補：

```java
public SceneService(SwitchBotApisV2 switchBotApisV2)
public SceneService(SwitchBotApisV2 switchBotApisV2, boolean dryRun)
public SceneService(SwitchBotApisV2 switchBotApisV2, SceneCatalog sceneCatalog, SwitchBotScenesConverter converter, boolean dryRun)
```

Ver.1のシンプルさとテスト容易性のバランスで提案してください。

### 3. cooldown未設定時のデフォルト値

`sceneCatalog.json` に cooldown がない場合は **10秒** にしてください。

以下のどこで補完するのが自然か提案してください。

```text
SceneCatalogEntry
SwitchBotScenesConverter
SceneSummary
SceneService
```

推奨は、`SwitchBotScenesConverter` または `SceneSummary生成時` で補完し、SceneServiceが常に確定済みcooldownを見られる形です。

### 4. cooldown値のバリデーション

以下をどう扱うか提案してください。

```text
cooldown 未設定
cooldown = 0
cooldown < 0
cooldown が数値ではない
```

現時点の希望：

```text
未設定: 10秒
0以上: 許可
負数: エラー
非数値: エラー
```

0秒は明示的に連打許可を意味するため、許可してよいです。

### 5. cooldown中の戻り値

cooldown中は例外ではなく SceneExecutionResult とします。

`message` や `httpStatusCode=0` の扱いが妥当か確認してください。

### 6. dry-run時の戻り値

dry-run時は SceneExecutionResult とします。

`success=true`, `executed=false`, `dryRun=true`, `httpStatusCode=0` の扱いが妥当か確認してください。

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

---

## 出力形式

以下のMarkdown形式で回答してください。

```markdown
# SceneService 追加仕様込み設計前レポート

## 1. 追加仕様の理解

## 2. 既存設計への影響

## 3. cooldown の設計方針

## 4. dry-run の設計方針

## 5. SceneExecutionResult の仕様案

## 6. SceneService の責務

## 7. public API案

## 8. コンストラクタ案

## 9. キャッシュ設計

## 10. スレッド安全性の方針

## 11. エラー方針

## 12. 既存クラスへの変更範囲

## 13. 今回やらないこと

## 14. Go / NoGo 判断

## 15. 推奨する次の作業
```

この形式に従って、まず分析だけを行ってください。