# SceneService 設計前レポート

## 1. 要求仕様の理解
SceneService は、SwitchBot Cloud API と sceneCatalog.json と変換ロジックを束ね、MCP Tool から扱いやすい「公開済みシーン操作用サービス」を提供する層です。理解している中心要件は以下です。

- シーン一覧は getScenes() → SceneCatalog.loadEntries() → SwitchBotScenesConverter.convert(...) で作る
- 上位へ公開するのは raw JSON ではなく List
- キャッシュ対象も変換済み List
- シーン実行は sceneId ではなく sceneKey で受ける
- sceneKey からの実行対象解決は listScenes() の結果だけを使う
- 実行結果は文字列ではなく SceneExecutionResult として構造化する
- 業務的な失敗は SceneServiceException に寄せる

## 2. SceneService の責務
SceneService の責務は妥当です。

- 公開可能なシーン一覧を返す
- その一覧を 24 時間 TTL でキャッシュする
- 手動更新用に refreshScenes() を提供する
- sceneKey を安全に sceneId へ解決する
- SwitchBotApisV2.executeScene(sceneId) を呼び出す
- 実行結果を SceneExecutionResult に変換する

逆に、認証情報読み込み、MCP Tool 定義、自然言語解釈、alias 解決、cooldown、監査ログは、この分析時点では SceneService の責務外でよいです。

## 3. 既存クラスとの接続方針
次の接続は自然です。

- SceneService uses SwitchBotApisV2
- SceneService uses SceneCatalog
- SceneService uses SwitchBotScenesConverter

SwitchBotApisV2 は低レイヤー API クライアント、SceneCatalog は公開許可リスト、SwitchBotScenesConverter は API レスポンスと catalog の突き合わせ担当なので、SceneService が orchestration 層になる構成は責務分離として妥当です。

SceneService 内で AppConfig.load() を呼ぶ設計は避けるべきです。token / secret の読み込みと SwitchBotApisV2 の生成は外側レイヤーの責務にするのが適切です。

## 4. public API案
推奨 API は以下です。

- listScenes()
- refreshScenes()
- executeBySceneKey(String sceneKey)

コンストラクタは、Ver.1 では以下の二段構えを推奨します。

- 簡易利用向け: SceneService(SwitchBotApisV2 switchBotApisV2)
- テスト・明示 DI 向け: SceneService(SwitchBotApisV2 switchBotApisV2, SceneCatalog sceneCatalog, SwitchBotScenesConverter converter)

この形なら、通常利用は簡単にしつつ、テスト時や将来の差し替えにも対応できます。新しい interface を増やす必要はまだありません。

## 5. SceneExecutionResult の仕様案
SceneExecutionResult は record が適しています。持つ項目は要求通りでよいです。

- sceneKey
- sceneId
- sceneName
- success
- httpStatusCode
- switchBotBody
- message

success は SwitchBotApiResult.success()、httpStatusCode は statusCode()、switchBotBody は body() をそのまま反映する方針でよいです。

注意点として、現状の SwitchBotApiResult.success() は HTTP 2xx 判定です。SwitchBot レスポンス body 内の業務ステータスまで成功判定に含める設計ではありません。これは今回の要求通りで問題ありません。

# 6. SceneServiceException の仕様案
SceneServiceException は既存の SceneCatalogException / SwitchBotScenesConvertException と揃えて RuntimeException 系でよいです。用途は以下です。

- sceneKey が null
- sceneKey が blank
- sceneKey が公開シーン一覧に存在しない
- 対象 SceneSummary が executable=false
- listScenes() / refreshScenes() 中の catalog 読み込み失敗
- listScenes() / refreshScenes() 中の変換失敗
- 通信不能などの SwitchBotApiException を SceneService 境界で包む場合

方針としては、SwitchBotApiException も SceneServiceException に包むことを推奨します。MCP Tool 側が SceneService の失敗を一種類で扱いやすくなり、原因は cause で保持できます。

## 7. キャッシュ設計
キャッシュ対象は List で妥当です。キャッシュ項目は以下で十分です。

- cachedScenes
- cachedAt
- CACHE_TTL = Duration.ofHours(24)

有効判定は要求通り、次の AND 条件にすべきです。

- cachedScenes != null
- cachedAt != null
- 現在時刻が cachedAt + TTL より前

期限切れキャッシュは返さない方針でよいです。キャッシュ保存時は List.copyOf(...) を使うべきです。確認した既存 SceneSummary は record で、aliases も List.copyOf(...) されているため、外部変更耐性は十分です。

refreshScenes() 失敗時は、既存キャッシュを破壊しない方がよいです。新しい一覧の取得・変換が成功してから cache を差し替える設計が安全です。

## 8. スレッド安全性の方針
Ver.1 では synchronized または private lock object で十分です。推奨は private lock object です。

- cache 読み書きを同じ lock で守る
- listScenes() と refreshScenes() の cache 判定・更新を排他する
- AtomicReference や非同期更新は不要
- TTL が 24 時間なので、高頻度更新を最適化する必要は薄い

executeBySceneKey() は listScenes() で安全な一覧を取得した後、実行 API 呼び出し自体は cache lock の外で行う設計がよいです。

## 9. sceneKey実行方針
要求通り、sceneId 直接指定は受けない方針が妥当です。実行フローは以下でよいです。

- sceneKey の null / blank を検証
- listScenes() を呼ぶ
- sceneKey 一致の SceneSummary を探す
- 見つからなければ SceneServiceException
- executable=false なら SceneServiceException
- SceneSummary.sceneId() で SwitchBotApisV2.executeScene(...)
- SceneExecutionResult を返す

照合は alias や自然言語ではなく、sceneKey の完全一致に限定するのが今回の安全設計に合っています。前後空白だけ trim するかは実装前に決めれば十分ですが、大文字小文字のゆらぎ吸収はしない方がよいです。

## 10.  エラー方針
- SceneCatalogException: SceneServiceException に包む。cache は成功時のみ更新。
- SwitchBotScenesConvertException: SceneServiceException に包む。cache は成功時のみ更新。
- SwitchBotApiException: SceneServiceException に包む方針を推奨。通信不能・署名失敗・URI生成失敗などは結果型ではなく例外でよい。
- getScenes() の HTTP 4xx/5xx: 一覧を作れないため SceneServiceException が妥当。converter に渡す前に HTTP 失敗を検出すると原因が明確になる。
- executeScene() の HTTP 4xx/5xx: SceneExecutionResult.success=false として返す。HTTP レスポンスが返っているため原則例外にしない。
- sceneKey 未登録: SceneServiceException。
- executable=false: SceneServiceException。現状 list 側で除外されるが、将来変更への防御として実行側でも確認する。

## 11.  今回やらないこと
この分析時点では、今回の範囲は SceneService、SceneExecutionResult、SceneServiceException に限定するのが適切です。以下は広げない方がよいです。

- MCP Tool 定義
- alias / 自然言語マッチング
- AppConfig 変更
- SwitchBotApisV2 変更
- sceneCatalog.json 変更
- cooldown
- dry-run
- 監査ログ
- TTL 外部設定化
- DB 化

## 12.  Go / NoGo 判断
判断: 条件付きGo

理由:
- 中心設計は既存クラスの責務と自然に接続できている
- sceneKey を listScenes() 経由で解決する方針は安全性が高い
- 変換済み List をキャッシュする方針は上位公開 DTO と整合している
- 24 時間 TTL と手動 refresh の組み合わせは Ver.1 として妥当
- record と List.copyOf(...) により戻り値の不変性も確保しやすい

製造前に確認すべきこと:
- コンストラクタは「簡易 1 引数 + 明示 DI 3 引数」の二段構えでよいか
- SwitchBotApiException は SceneServiceException に包む方針でよいか
- getScenes() の HTTP 4xx/5xx は SceneServiceException として扱う方針でよいか
- sceneKey は前後空白 trim 後の完全一致にするか、完全な raw 一致にするか

## 13.  推奨する次の作業
次は製造フェーズとして、以下 3 ファイルを追加するのが適切です。

- SceneService.java
- SceneExecutionResult.java
- SceneServiceException.java

実装時は、まず listScenes() / refreshScenes() のキャッシュ更新と例外方針を固め、その後 executeBySceneKey() を接続する順番がよいです。