# SceneCatalog / SceneSummary / SwitchBotScenesConverter 設計前レポート

## 1. 要求仕様の理解
要求は次の通り理解しました。
- sceneCatalog.json は人間が管理する公開許可リスト。
- SwitchBot API の getScenes() 結果は、そのまま LLM / MCP に公開しない。
- 公開対象は以下をすべて満たすシーンのみ。
  - SwitchBot API の getScenes に存在する
  - sceneCatalog.json にも存在する
  - sceneId が一致する
  - Catalog 側の executable が true
- SceneCatalog は管理用情報を含む内部定義を扱う。
- SceneSummary は LLM / MCP に見せる正式DTOで、note は含めない。
- SwitchBotScenesConverter は raw JSON と Catalog を突合して SceneSummary 一覧を作る。
- SwitchBot アプリ側に追加しただけのシーンは、自動公開しない。この理解で妥当です。

## 2. プロジェクト構成・依存関係の確認
確認結果です。
- build.gradle は見当たりません。
- pom.xml があります。
- Maven project で、packaging は war です。
- Maven Wrapper は存在します。
  - mvnw
  - mvnw.cmd
  - .mvn/wrapper/maven-wrapper.jar
- pom.xml 上の Java 指定は 21 です。
  - maven.compiler.source=21
  - maven.compiler.target=21
- Jackson は利用可能です。
  - jackson-core 2.19.1
  - jackson-databind 2.19.1
  - jackson-annotations 2.19.1
- package 命名は以下の形です。
  - MCP側: vr46.switchbotctrlmcp.*
  - 今回の配置先: vr46.switchbotctrlmcp.switchbot
  - 共通ライブラリ側 SwitchBot API: switchBot.*
- AppConfig.java は存在し、TOKEN / SECRET を環境変数から読みます。
- SwitchBotApisV2 は MCP プロジェクト内ではなく、依存先の MyCommonLibV2 側に存在します。
- SwitchBotApisV2.getScenes() は SwitchBotApiResult を返し、raw JSON は result.body() から取得する構造です。
- src/main/resources/sceneCatalog.json は Maven / WAR では classpath resource として扱うのが適切です。注意点として、このシェル環境では java / mvn が PATH に見えておらず、JAVA_HOME も空でした。IntelliJ 側で動く可能性はありますが、製造後のコマンドライン検証には JDK 21 の PATH または IntelliJ 実行環境が必要です。resources の読み込みは、絶対パスではなく classpath から読む方針が妥当です。JSON は UTF-8 で正しく読めました。PowerShell の既定読み取りでは一度文字化けして見えたため、Java 側でも明示的に UTF-8 / Jackson の classpath InputStream 読み込みを使うべきです。

## 3. sceneCatalog.json の確認観点
実ファイルを確認しました。
- JSON配列: 問題なし
- 件数: 4件
- 必須項目: 4件すべて揃っています
- aliases: すべて配列です
- executable: すべて boolean です
- sceneId 重複: なし
- sceneKey 重複: なし
- note: 存在しますが、管理用情報として扱い、SceneSummary には含めない方針で問題ありません実データは提示された想定形式と一致しています。確認できた sceneKey は以下です。
- room_light_full
- room_light_standard
- room_light_dark
- room_light_off現時点の sceneCatalog.json は製造に進める状態と判断できます。

## 4. getScenes raw JSON のパース方針
SwitchBotApisV2.getScenes() 自体は HTTP 結果として SwitchBotApiResult を返します。設計上は、まず以下を分けて考えるべきです。
- HTTP レベルの結果
  - SwitchBotApiResult.statusCode()
  - SwitchBotApiResult.success()
  - SwitchBotApiResult.body()
- SwitchBot API JSON 本体の結果
  - JSON内の statusCode
  - JSON内の body
  - JSON内の message今回の SwitchBotScenesConverter は、基本的に SwitchBotApiResult.body() に入っている raw JSON を入力として扱えばよいです。raw JSON の期待構造は以下です。
- root
  - statusCode: number。正常は 100
  - message: string。正常時は success
  - body: array
- body item
  - sceneId: string
sceneName: string方針としては、JSON内の statusCode が 100 以外の場合、空リストにせずエラー扱いがよいです。空リストにすると、API障害・認証エラー・仕様変更を「公開可能シーンがない」と誤認します。

## 5. inner join方針
inner join 方針は要求通りで妥当です。
- join key は sceneId
- sceneId は完全一致で扱う
- Catalog に存在しない SwitchBot シーンは落とす
- SwitchBot API に存在しない Catalog 定義は落とす
- Catalog の executable=false は出力しない
- note は SceneSummary に渡さない出力順は、Catalog の順序を維持する方針を推奨します。Catalog は人間が管理する公開リストなので、LLM / MCP に見せる順序も人間側の定義を優先した方が安定します。sceneName 不一致時の推奨方針は以下です。
- join の可否は sceneId のみで判断する
- sceneName が不一致でも、デフォルトでは除外しない
- 出力する sceneName は SwitchBot API 側の実名を優先する
- 不一致はログまたは診断情報として残す
- テストや起動前検証では strict mode 的にエラー扱いできる余地を残す理由は、SwitchBot アプリ側で単に名前を変更しただけで公開一覧が壊れるのは運用上つらいためです。一方で、Catalog とのズレは管理上重要なので、黙殺はしない方がよいです。

## 6. クラス構成案
候補のクラス構成は概ね妥当です。推奨構成は以下です。
- SceneCatalog.java
  - sceneCatalog.json の読み込み、検証、Catalog entries の提供
- SceneCatalogEntry.java
  - Catalog 1件分の内部DTO
  - note を含む
- SceneSummary.java
  - LLM / MCP 公開用DTO
  - note を含めない
- Scenes.java
  - SwitchBot getScenes raw JSON の受け皿
  - ただし名前は SwitchBotScenesResponse の方が意味は明確
- SwitchBotScenesConverter.java
  - raw scenes と Catalog を突合し、SceneSummary 一覧に変換するクラス数を無理に減らす必要はありません。減らすとすれば、Scenes.java を作らず Jackson の JsonNode で直接読む案があります。ただし、今回のように statusCode / body / message の構造が明確な場合は、raw response DTO を持つ方がテストしやすく、責務も読みやすいです。SceneCatalogEntry を SceneCatalog の内部クラスにする案もありますが、Jackson mapping と単体テストを考えると独立クラスの方が扱いやすいです。

## 7. SceneCatalog の責務
SceneCatalog の責務は以下に限定するのがよいです。
- classpath 上の sceneCatalog.json を読む
- JSON を SceneCatalogEntry 一覧に変換する
- Catalog としての妥当性を検証する
- 検証済みの定義一覧を返す
- 将来 DB / 管理画面に移行しても、呼び出し側に影響を出さないSceneCatalog がやらない方がよいことは以下です。
- SwitchBot API を呼ばない
- SwitchBotApisV2 を直接生成しない
- LLM公開用の SceneSummary 生成ロジックを持たない
- alias マッチングを行わない
 -executeScene の制御を行わないAppConfig や SwitchBotApisV2 は、将来の API 呼び出し層で使うべきであり、SceneCatalog の責務には入れない方が分離がきれいです。

## 8. SceneSummary の責務
SceneSummary は LLM / MCP / 上位サービスに公開する DTO です。含める項目は以下で妥当です。
- sceneKey
- sceneId
- sceneName
- publicName
- category
- action
- riskLevel
- executable
- aliases
  含めない項目は以下です。
- note
  executable=true のものだけ出力するなら SceneSummary.executable は常に true になりますが、公開契約として残す方針は問題ありません。

## 9. SwitchBotScenesConverter の責務
SwitchBotScenesConverter の責務は以下です。
- getScenes raw JSON をパースする
- SwitchBot API JSON の statusCode を確認する
- body 配列から sceneId / sceneName を取り出す
- Catalog entries と sceneId で inner join する
- executable=false を除外する
- SceneSummary を生成する
- note を出力しない
- Catalog 未登録の SwitchBot シーンを出力しないSwitchBotScenesConverter がやらない方がよいことは以下です。
- SwitchBot API を実際に呼ぶ
- resources を直接読む
- .env や AppConfig を扱う
- alias でユーザー発話を解釈する
- executeScene を呼ぶ
- cooldown / dry-run / 監査ログを扱うつまり、入力を受けて変換する純粋な変換層として設計するのがよいです。

# 10.  例外・エラー方針
推奨方針です。
- sceneCatalog.json が存在しない場合
  - 起動時または Catalog load 時に失敗扱い
  - 空リストにはしない
- sceneCatalog.json のJSON形式が壊れている場合
  - 失敗扱い
  - Jackson の例外を包んで、Catalog 読み込み失敗として扱う
- 必須項目が欠落している場合
  - 失敗扱い
  - 何番目の entry のどの項目が欠落しているか分かる形が望ましい
- sceneKey / sceneId が重複している場合
  - 失敗扱い
  - 公開許可リストとして曖昧になるため許容しない
- getScenes raw JSON が壊れている場合
  - 失敗扱い
  - 空リストにはしない
- SwitchBot API JSON内の statusCode が 100 以外の場合
  - 失敗扱い
  - message があれば診断情報に含める
- body が配列でない、または存在しない場合
  - 失敗扱い
- body 内の sceneId / sceneName が欠落している場合
  - 失敗扱い
- sceneName が Catalog と API で一致しない場合
  - デフォルトは warning / 診断扱い
  - 出力 sceneName は API 側を優先
  - strict 検証では失敗扱いにできる設計が望ましい

## 11.  今回やらないこと
今回の分析範囲では、以下は対象外でよいです。
- SwitchBot API の実呼び出し
- SwitchBotApisV2 の修正
- AppConfig の修正
- MCP Tool 定義
- SceneService のキャッシュ
- executeScene の実行制御
- cooldown
- dry-run
- 監査ログ
- alias マッチング
- LLM による意図解釈
- シーン実行 API の実装今回の対象はあくまで以下に限定するのが適切です。
- sceneCatalog.json
- getScenes raw JSON
- SceneSummary 一覧への変換

## 12.  Go / NoGo 判断
判断: 条件付きGo理由:
- 要求仕様は明確です。
- クラス責務の分離方針は妥当です。
- sceneCatalog.json は確認対象データとして JSON 配列で、必須項目・型・重複なしを確認できました。
- Jackson は pom.xml 上で利用可能です。
- Java 21 指定も確認できました。
- SwitchBotApisV2.getScenes() の raw JSON は SwitchBotApiResult.body() から受け取る設計で問題ありません。
- inner join により、SwitchBot 側に追加しただけのシーンを LLM に公開しない目的を満たせます。製造前に確認すべきこと:
- sceneName 不一致時の最終方針を「warning + API名優先」でよいか。
- raw response DTO の名前を Scenes にするか、より明確な SwitchBotScenesResponse にするか。
- 製造後にテストを実行できるよう、JDK 21 / Maven Wrapper の実行環境を確認すること。

## 13.  推奨する次の作業
次の製造フェーズでは、まず以下の順で進めるのがよいです。
- SceneCatalogEntry と SceneSummary の DTO を作る
- SceneCatalog で sceneCatalog.json を classpath から読み込む
- Catalog validation を入れる
- SwitchBot getScenes raw JSON 用 DTO を作る
- SwitchBotScenesConverter で sceneId inner join を実装する
- 単体テストで以下を確認する
  - Catalog 正常読み込み
  - Catalog 重複検出
  - API側にだけあるシーンを除外
  - Catalog側にだけあるシーンを除外
  - executable=false を除外
  - statusCode != 100 をエラー
  - sceneName 不一致時の挙動この段階では製造に進める設計材料は揃っています。条件は、上記の小さな方針確認と実行環境確認です。