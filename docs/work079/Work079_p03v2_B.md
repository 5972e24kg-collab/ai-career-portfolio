# SceneService 追加仕様込み設計前レポート
## 1. 追加仕様の理解
cooldown と dry-run は、SceneService の実行制御責務に含めてよいです。
- cooldown は sceneKey 単位の誤連打防止
- dry-run は SceneService インスタンス全体の実行抑止モード
- どちらも MCP Tool ではなく、SceneService が判断する運用ルール
- 実行結果は例外ではなく SceneExecutionResult で返す場面がある

## 2. 既存設計への影響
cooldown は既存DTOの公開・変換経路に乗せる必要があります。
- SceneCatalogEntry に cooldown を追加
- SceneSummary に cooldown を追加
- SwitchBotScenesConverter で cooldown を SceneSummary に渡す
- cooldown 未設定時の 10秒補完を変換時点までに済ませる
- SceneService は補完済みの SceneSummary.cooldown を使う
現状確認では SceneCatalogEntry / SceneSummary は record で、aliases は List.copyOf(...) 済みです。この方針は維持できます。

## 3. cooldown の設計方針
cooldown は sceneKey 単位で管理する方針が妥当です。
- sceneId ではなく、上位公開キーである sceneKey を基準にする
- lastExecutedAtBySceneKey のような内部状態で直近実行時刻を持つ
- cooldown 中は executeScene を呼ばない
- cooldown 中は例外ではなく SceneExecutionResult(success=false, executed=false) を返す
記録タイミングは、提案どおり「実際に executeScene を呼んだ場合のみ更新」が妥当です。
- dry-run では更新しない
- cooldown 拒否では更新しない
- HTTP 4xx/5xx でも、API呼び出し自体は行ったため更新する

## 4. dry-run の設計方針
dry-run は SceneService 生成時に固定する案Aで妥当です。
- リクエストごとの切替より単純
- テストしやすい
- MCP Tool 側の引数設計に依存しない
- 実行抑止というサービス運用モードとして自然
  要求フローどおり、cooldown 判定後に dry-run 判定するなら、「dry-run でも現在の cooldown ルールを検証する」動きになります。これは妥当ですが、dry-run を cooldown 無視の純粋な解決テストにしたい場合は順序を変える必要があります。現仕様では cooldown 優先でよいです。

## 5. SceneExecutionResult の仕様案
項目は以下で妥当です。
- sceneKey
- sceneId
- sceneName
- success
- httpStatusCode
- switchBotBody
- message
- executed
- dryRun
意味も整理できています。
- 実実行成功: success=true, executed=true, dryRun=false
- executeScene HTTP 4xx/5xx: success=false, executed=true, dryRun=false
- dry-run: success=true, executed=false, dryRun=true
- cooldown 中: success=false, executed=false, dryRun=false
httpStatusCode=0 と switchBotBody="" は「HTTPレスポンスが存在しない」ことを表す値として Ver.1 では妥当です。

## 6. SceneService の責務
SceneService の責務は以下に限定するのがよいです。
- シーン一覧取得
- List<SceneSummary> のキャッシュ
- 手動更新 refreshScenes
- sceneKey による安全なシーン解決
- cooldown 判定
- dry-run 判定
- 実行結果の構造化
- SceneService レイヤーとしての例外変換
AppConfig の読み込み、MCP Tool 接続、自然言語解釈は含めない方針でよいです。

## 7. public API案
public API は次で十分です。
- listScenes()
- refreshScenes()
- executeBySceneKey(String sceneKey)
executeBySceneId は作らない方がよいです。安全境界が sceneKey と listScenes() にあるためです。

## 8. コンストラクタ案
推奨は、簡易コンストラクタと明示DIコンストラクタの併用です。
- SceneService(SwitchBotApisV2 switchBotApisV2)
- SceneService(SwitchBotApisV2 switchBotApisV2, boolean dryRun)
- SceneService(SwitchBotApisV2 switchBotApisV2, SceneCatalog sceneCatalog, SwitchBotScenesConverter converter, boolean dryRun)
Ver.1利用では前2つが簡単で、テストでは3つ目を使えます。SceneService 内で AppConfig.load() する設計は避けるべきです。

## 9. キャッシュ設計
前回方針のままで妥当です。
- キャッシュ対象は List<SceneSummary>
- TTL は 24時間
- TTL は private static final 相当で一箇所にまとめる
- 有効条件は cachedScenes != null かつ cachedAt != null かつ期限内
- refreshScenes() は成功時のみキャッシュ更新
- 失敗時は既存キャッシュを壊さない
戻り値は List.copyOf(...) 済みの不変リストを保持・返却する方針でよいです。

## 10.  スレッド安全性の方針
Ver.1 では private lock object で十分です。
対象は以下です。
- cachedScenes
- cachedAt
- lastExecutedAtBySceneKey
注意点は、SwitchBot API 呼び出し中に長時間 lock を保持しないことです。
実行時は、scene 解決後に cooldown 判定を lock 内で行い、実行可能なら「実行予約」相当の状態にするか、少なくとも同一 sceneKey の二重通過を防ぐ設計が必要です。単純に lock 外で API 呼び出し後に更新すると、並列リクエストが同時に cooldown をすり抜ける可能性があります。

## 11.  エラー方針
SceneServiceException にするもの:
- sceneKey null / blank
- trim 後の sceneKey が未登録
- executable=false
- SceneCatalogException
- SwitchBotScenesConvertException
- SwitchBotApiException
- getScenes() の HTTP 4xx/5xx
- SceneExecutionResult.success=false で返すもの:
- executeScene() の HTTP 4xx/5xx
- cooldown 中
SceneExecutionResult.success=true / executed=false で返すもの:
- dry-run
getScenes() は一覧構築の前提なので HTTP 失敗を例外化し、executeScene() は実行結果として返す、という分離は自然です。

## 12.  既存クラスへの変更範囲
必要な変更範囲は以下です。
- SceneCatalogEntry: cooldown 追加。未設定を扱うなら nullable な数値型が自然
- SceneSummary: 補完済み cooldown 追加。こちらは非nullの数値が自然
- SwitchBotScenesConverter: cooldown の補完と伝搬
- SceneCatalog: cooldown の任意項目検証を追加
- sceneCatalog.json: cooldown 項目を必要に応じて追加
cooldown の検証方針は以下が妥当です。
- 未設定: 10秒
- 0: 許可
- 正数: 許可
- 負数: エラー
- 非数値: エラー
- 明示的な null: 仕様として許可するか要確認。厳密にはエラー推奨

## 13.  今回やらないこと
今回やらない範囲は明確で、設計上も分離できています。
- MCP Tool 定義
- SceneService の MCP 接続
- alias / 自然言語マッチング
- LLM による意図解釈
- 監査ログ
- DB化
- TTL 外部設定化
- SwitchBotApisV2 修正
- AppConfig 修正

## 14.  Go / NoGo 判断
判断: 条件付きGo
理由:
- SceneService に cooldown / dry-run を入れる方針は妥当
- 既存の SceneCatalog -> Converter -> SceneSummary -> SceneService の責務分離は崩れていない
- sceneKey 解決を listScenes() 経由にする安全方針も妥当
- キャッシュ対象を List<SceneSummary> にする方針も妥当
- ただし、cooldown の型・null扱い・dry-run と cooldown の優先順は製造前に明文化した方がよい
製造前に確認すべきこと:
- cooldown の Java フィールド名を cooldown にするか、秒単位を明示する名前にするか
- cooldown: null を未設定扱いにするか、設定エラーにするか
- dry-run 時も cooldown 中なら cooldown 結果を返す仕様で確定するか
- 並列実行時に同一 sceneKey の二重実行をどこまで厳密に防ぐか

## 15.  推奨する次の作業
次は製造に進めますが、最初に変更範囲を小さく分けるのがよいです。
1. cooldown を既存DTOと converter / catalog validation に追加
2. SceneExecutionResult と SceneServiceException を追加
3. SceneService の list / refresh / cache を実装
4. executeBySceneKey に sceneKey解決、cooldown、dry-run、実実行を追加
5. 単体テストで cache、cooldown、dry-run、HTTP失敗、例外変換を確認