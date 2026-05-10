# 依頼：SceneService / cooldown / dry-run の製造

あなたは Java の設計・実装支援者です。

前工程で、SceneService 追加仕様込みの設計分析は完了しています。
今回は、その確定仕様に基づき、実際にコードを作成・変更してください。

この依頼では **コード変更・ファイル作成を行ってよい** です。

---

## 作業対象ディレクトリ

Javaソースの作業対象は以下です。

```text
<local-project-root>\src\main\java\vr46\switchbotctrlmcp\switchbot
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

## 今回の製造対象

今回作成・変更する中心は以下です。

```text
SceneService.java
SceneExecutionResult.java
SceneServiceException.java
```

また、cooldown追加に伴い、必要に応じて以下の既存クラスを変更してください。

```text
SceneCatalogEntry.java
SceneSummary.java
SwitchBotScenesConverter.java
SceneCatalog.java
```

必要なら `sceneCatalog.json` の `cooldown` 項目に対応できるようにしてください。
ただし、`sceneCatalog.json` の構造を今回の仕様以上に勝手に変更しないでください。

---

## package

作成・変更するクラスのpackageは以下です。

```java
package vr46.switchbotctrlmcp.switchbot;
```

---

## 今回の目的

SceneService は、これまで作成した部品を束ね、MCP Tool から使いやすい運用単位を提供する層です。

主な責務は以下です。

```text
1. 公開可能なシーン一覧を返す
2. シーン一覧を24時間キャッシュする
3. 手動更新用 refreshScenes() を提供する
4. sceneKey でシーンを実行する
5. dry-run モードを持つ
6. sceneKey単位の cooldown を持つ
7. 実行結果を SceneExecutionResult として返す
```

---

## 既存クラスの前提

### SwitchBotApisV2

SwitchBot Cloud API v1.1 を呼び出す低レイヤーAPIクライアントです。

主なメソッド：

```java
getScenes()
executeScene(String sceneId)
```

戻り値は `SwitchBotApiResult` です。

HTTP 4xx / 5xx は例外ではなく `SwitchBotApiResult` として返ります。
通信不能・署名失敗・URI生成失敗・割り込みなどは `SwitchBotApiException` になります。

### SceneCatalog

`src/main/resources/sceneCatalog.json` を classpath resource として読み込み、`List<SceneCatalogEntry>` を返します。

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

---

# 追加仕様1：cooldown

## 1. sceneCatalog.json に cooldown を追加する

`sceneCatalog.json` に `cooldown` 項目を追加します。

単位は秒です。

例：

```json
{
  "sceneKey": "room_light_full",
  "sceneId": "[sceneId masked]",
  "sceneName": "照明系シーンA",
  "publicName": "照明系を明るくする",
  "category": "light",
  "action": "set_mode",
  "riskLevel": "safe",
  "executable": true,
  "aliases": ["照明系を明るくして"],
  "cooldown": 5,
  "note": ""
}
```

## 2. フィールド名

Java側のフィールド名は以下にしてください。

```text
cooldown
```

秒単位であることは、設定マニュアル側で説明します。
Javaフィールド名を `cooldownSeconds` などに変更しないでください。

## 3. cooldown未設定時

`cooldown` が未設定、または `null` の場合は、デフォルト値として **10秒** を使ってください。

## 4. cooldown値の扱い

```text
未設定: 10秒
null: 10秒
0以上: 許可
負数: エラー
非数値: エラー
```

`cooldown=0` は、明示的に連続実行を許可する設定として扱います。

## 5. cooldown を持たせるクラス

cooldown は管理メモではなく、実行制御に必要な正式な運用メタ情報です。

そのため、以下に追加してください。

```text
SceneCatalogEntry
SceneSummary
```

また、`SwitchBotScenesConverter` で `SceneCatalogEntry.cooldown` を `SceneSummary.cooldown` に渡してください。

`cooldown` 未設定 / null の補完は、SceneSummary生成時までに行い、SceneServiceが常に確定済み cooldown を見られるようにしてください。

推奨：

```text
SwitchBotScenesConverter で補完する
または SceneSummary 生成時に補完する
```

---

# 追加仕様2：dry-run

dry-run は **SceneService全体のモード** として持たせます。

SceneService生成時に dry-run を固定してください。

例：

```java
new SceneService(switchBotApisV2, true);
```

または、明示DI用コンストラクタで、

```java
new SceneService(switchBotApisV2, sceneCatalog, converter, true);
```

のように指定できる設計にしてください。

dryRun が true の場合、`executeBySceneKey(sceneKey)` は対象シーンを解決しますが、`SwitchBotApisV2.executeScene(sceneId)` は呼びません。

---

# SceneExecutionResult

`SceneExecutionResult.java` を作成してください。

持たせる項目は以下です。

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

意味：

```text
success:
  処理として成功したか

executed:
  実際に SwitchBot API executeScene を呼んだか

dryRun:
  dry-run により実行しなかったか
```

Java record で構いません。

---

## SceneExecutionResult の値

### 1. 実実行成功時

```text
success=true
executed=true
dryRun=false
httpStatusCode=SwitchBotApiResult.statusCode()
switchBotBody=SwitchBotApiResult.body()
```

### 2. SwitchBot API HTTP 4xx/5xx時

```text
success=false
executed=true
dryRun=false
httpStatusCode=SwitchBotApiResult.statusCode()
switchBotBody=SwitchBotApiResult.body()
```

HTTPレスポンスが返っているため、これは例外ではなく `SceneExecutionResult` としてください。

### 3. dry-run時

```text
success=true
executed=false
dryRun=true
httpStatusCode=0
switchBotBody=""
message="Dry-run: scene execution was skipped."
```

message の文言は多少調整して構いません。

### 4. cooldown中

```text
success=false
executed=false
dryRun=false
httpStatusCode=0
switchBotBody=""
message="Cooldown active."
```

cooldown は異常ではなく安全制御が働いた結果なので、例外ではなく `SceneExecutionResult` として返してください。

---

# SceneService

`SceneService.java` を作成してください。

## public API

以下の public API を用意してください。

```java
public List<SceneSummary> listScenes()

public List<SceneSummary> refreshScenes()

public SceneExecutionResult executeBySceneKey(String sceneKey)
```

## コンストラクタ

以下のような二段構えを推奨します。

```java
public SceneService(SwitchBotApisV2 switchBotApisV2)

public SceneService(SwitchBotApisV2 switchBotApisV2, boolean dryRun)

public SceneService(
    SwitchBotApisV2 switchBotApisV2,
    SceneCatalog sceneCatalog,
    SwitchBotScenesConverter converter,
    boolean dryRun
)
```

Ver.1としてシンプルさを優先しつつ、テストしやすい構成にしてください。

重要：

* SceneService内部で `AppConfig.load()` を呼ばないでください。
* token / secret の読み込みは外側レイヤーの責務です。
* `SwitchBotApisV2` は外から注入してください。

---

# listScenes()

`listScenes()` は、キャッシュが有効な場合、キャッシュ済みの `List<SceneSummary>` を返します。

キャッシュが存在しない、または期限切れの場合は、内部的に `refreshScenes()` 相当の再取得を行います。

処理の流れ：

```text
キャッシュ有効判定
  ↓ 有効なら cachedScenes を返す
  ↓ 無効なら再取得
SwitchBotApisV2.getScenes()
SceneCatalog.loadEntries()
SwitchBotScenesConverter.convert(rawJson, catalogEntries)
List<SceneSummary> をキャッシュ
List<SceneSummary> を返す
```

---

# refreshScenes()

`refreshScenes()` はキャッシュを無視して再取得してください。

処理の流れ：

```text
SwitchBotApisV2.getScenes()
SceneCatalog.loadEntries()
SwitchBotScenesConverter.convert(rawJson, catalogEntries)
成功したらキャッシュ更新
List<SceneSummary> を返す
```

重要：

* 失敗時は既存キャッシュを破壊しないでください。
* 新しい一覧の取得・変換が成功してから cache を差し替えてください。

---

# キャッシュ設計

## キャッシュ対象

```text
List<SceneSummary>
```

raw JSON や `SwitchBotApiResult` はキャッシュ対象にしないでください。

## TTL

24時間です。

Ver.1ではクラス定数で構いません。

```java
private static final Duration CACHE_TTL = Duration.ofHours(24);
```

外部設定化は不要です。

## 有効判定

キャッシュ有効判定は以下の AND 条件です。

```text
cachedScenes != null
かつ
cachedAt != null
かつ
現在時刻が cachedAt + CACHE_TTL より前
```

`or` ではなく `and` です。

---

# executeBySceneKey(sceneKey)

`executeBySceneKey(sceneKey)` の処理順序は以下です。

```text
sceneKey null / blank チェック
  ↓
sceneKey を trim
  ↓
listScenes()
  ↓
sceneKey一致の SceneSummary を探す
  ↓
見つからなければ SceneServiceException
  ↓
executable=false なら SceneServiceException
  ↓
cooldown中なら SceneExecutionResult(success=false, executed=false)
  ↓
dryRun=trueなら SceneExecutionResult(success=true, executed=false, dryRun=true)
  ↓
SwitchBotApisV2.executeScene(sceneId)
  ↓
lastExecutedAt 更新
  ↓
SceneExecutionResult を返す
```

重要：

* `sceneId` を直接受け取らないでください。
* `sceneCatalog.json` 単体から sceneKey → sceneId 変換しないでください。
* `listScenes()` の結果から sceneKey を解決してください。
* alias / 自然言語マッチングは今回やらないでください。
* sceneKey は trim 後の完全一致にしてください。
* 大文字小文字のゆらぎ吸収はしないでください。
* dry-run より先に cooldown を判定してください。

---

# cooldown の記録タイミング

lastExecutedAt は、以下の方針にしてください。

```text
SwitchBot API executeScene を実際に呼び出した場合のみ、lastExecutedAt を更新する
```

つまり：

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

---

# 並列実行時の扱い

Ver.1では、並列実行時の同一sceneKey二重実行を厳密には防ぎません。

理由：

* Ver.1では小規模運用を前提としている
* 複雑な排他制御は今回の学習テーマから外れる
* Ver.1ではシンプルさを優先する

ただし、通常のキャッシュ読み書きや lastExecutedAt の読み書きが壊れない程度のスレッド安全性は持たせてください。

方針：

```text
private lock object などで cachedScenes / cachedAt / lastExecutedAt の読み書きは保護する
ただし、SwitchBot API executeScene 呼び出し中に長時間lockを保持し続ける必要はない
同時実行時に極まれに二重実行される可能性は許容する
```

AtomicReference や非同期キャッシュ更新などの複雑な仕組みは不要です。

---

# SceneServiceException

`SceneServiceException.java` を作成してください。

SceneServiceの業務的な失敗を集約します。

SceneServiceExceptionにするもの：

```text
sceneKey null / blank
sceneKey未登録
executable=false
SceneCatalogException
SwitchBotScenesConvertException
SwitchBotApiException
getScenes() の HTTP 4xx/5xx
```

ただし、以下は例外にしないでください。

```text
executeScene() の HTTP 4xx/5xx
cooldown中
dry-run
```

これらは `SceneExecutionResult` として返します。

---

# getScenes() の HTTP 4xx/5xx

`listScenes()` / `refreshScenes()` 内で `SwitchBotApisV2.getScenes()` が HTTP 4xx / 5xx を返した場合、一覧を作れないため `SceneServiceException` にしてください。

理由：

* 公開可能シーン一覧が作れない
* 空リストで握りつぶすと「シーンがない」と誤認する
* HTTPレベルで失敗しているため、MCP上位へ異常として伝えるべき

---

# executeScene() の HTTP 4xx/5xx

`executeBySceneKey()` 内で `SwitchBotApisV2.executeScene(sceneId)` が HTTP 4xx / 5xx を返した場合は、例外にしないでください。

`SceneExecutionResult` として返します。

```text
success=false
executed=true
dryRun=false
httpStatusCode=SwitchBotApiResult.statusCode()
switchBotBody=SwitchBotApiResult.body()
```

---

# 今回やらないこと

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
AppConfigの修正
複雑な並列実行制御
```

今回は、以下に集中してください。

```text
既存クラスへのcooldown追加
SceneService
SceneExecutionResult
SceneServiceException
dry-run
cooldown
```

---

# 実装後に確認してほしいこと

作業後、以下を確認してください。

```text
SceneCatalogEntry に cooldown が追加されている
SceneSummary に cooldown が追加されている
SwitchBotScenesConverter が cooldown を引き継いでいる
cooldown 未設定/null は 10秒になる
cooldown=0 は許可される
cooldown<0 はエラーになる
SceneService が List<SceneSummary> を24時間キャッシュする
refreshScenes() がキャッシュを無視して再取得する
refreshScenes() 失敗時に既存キャッシュを壊さない
executeBySceneKey() が sceneKey で実行する
executeBySceneKey() が sceneId を直接受け取らない
dry-run時に SwitchBotApisV2.executeScene() を呼ばない
cooldown中に SwitchBotApisV2.executeScene() を呼ばない
dry-runより先に cooldown 判定している
SwitchBot APIを実際に呼んだ場合だけ lastExecutedAt が更新される
executeScene の HTTP 4xx/5xx は SceneExecutionResult.success=false になる
getScenes の HTTP 4xx/5xx は SceneServiceException になる
AppConfig / SwitchBotApisV2 を変更していない
MCP Tool を実装していない
```

---

# テストについて

IntelliJ IDEA上で単体確認する予定です。

そのため、以下は必須ではありません。

```text
mvnwのCLI動作確認
OS環境変数の設定確認
実SwitchBot API呼び出し
```

ただし、可能であれば、テストしやすい構造にしてください。

特に以下の観点を確認しやすい設計にしてください。

```text
listScenes() キャッシュ利用
refreshScenes() 強制更新
sceneKey未登録時の例外
dry-run時にexecuteSceneを呼ばない
cooldown中にexecuteSceneを呼ばない
cooldown=0 の扱い
executeScene HTTP失敗時の SceneExecutionResult
```

---

# 出力してほしい報告形式

作業後、以下の形式で報告してください。

```markdown
# SceneService / cooldown / dry-run 製造結果

## 1. 作成・変更したファイル

## 2. 変更した既存クラス

## 3. SceneExecutionResult の仕様

## 4. SceneService の仕様

## 5. cooldown の仕様

## 6. dry-run の仕様

## 7. キャッシュ仕様

## 8. エラー・例外方針

## 9. テスト・確認結果

## 10. 注意点・未確認事項
```

---

## 重要な制約

* 今回は製造フェーズなのでコード作成・変更してよい
* ただし MCP Tool は実装しないこと
* SceneServiceをMCPに接続しないこと
* SwitchBotApisV2を修正しないこと
* AppConfigを修正しないこと
* 複雑な並列実行制御を入れないこと
* dry-run より先に cooldown 判定すること
