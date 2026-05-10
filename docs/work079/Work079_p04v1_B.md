# SwitchBot McpServlet 設計前レポート
## 1. 要求仕様の理解
今回の目的は、SceneService を MCP Tool として公開する薄い McpServlet の設計確認です。
コード変更・新規ファイル作成・実装は行わず、参照ファイルと既存構成の確認だけを行いました。
公開する Tool は scenes と executeScene の2つのみで、refreshScenes は公開しない方針で問題ないと判断します。

## 2. BaseMcpServlet の拡張ポイント確認
BaseMcpServlet.java は以下を共通処理として持っています。
- JSON-RPC 2.0 の検証
- MCP protocol version の検証
- CORS設定
- セッション作成・検証
- initialize
- notifications/initialized
- tools/list
- tools/call
- 未知method / 未知toolのJSON-RPC error応答
サブクラスで使う拡張ポイントは、依頼内容どおり以下で足りると判断します。
- resInitialize
- resToolsList
- resToolsCall
未知Toolは resToolsCall で null を返せば、BaseMcpServlet 側が Unknown tool の JSON-RPC error にします。

## 3. SampleMcpServlet の構造確認
SampleMcpServlet.java.txt は、今回そのまま踏襲できる構造です。
流用すべき作風は以下です。
- static final ObjectMapper mapper
- SERVER_NAME / SERVER_TITLE / SERVER_VERSION
- Tool名定数
- constructorで super(SERVER_VERSION)
- resInitialize
- resToolsList
- addXxxTool
- resToolsCall
- xxxToolResponse
- successResponse
- errorResponse
- baseToolCallResponse
- toErrorJson
- request helper
- logging helper
SwitchBot版でも、余計な共通化はせず、この並びと粒度を維持するのが妥当です。

## 4. 今回のMcpServletの責務
McpServlet の責務は、MCP request / response と SceneService の薄い接続に限定する方針が妥当です。
具体的には以下のみです。
- initialize応答の設定
- tools/listのTool定義
- tools/callのTool名分岐
- sceneKey 引数取得
- SceneService.listScenes() 呼び出し
- SceneService.executeBySceneKey(sceneKey) 呼び出し
- 戻り値のJSON文字列化
- Tool内部エラーの isError=true 応答
SwitchBot API直接呼び出し、scene catalog読込、cooldown、dry-run、alias解釈は McpServlet に入れない方針で妥当です。

## 5. Tool設計
Tool名は以下で問題ないと判断します。
- scenes
- executeScene
scenes は引数なし、公開許可済みSceneだけを返す一覧Toolです。返却された sceneKey を executeScene に渡す、という導線を説明に明記するべきです。
executeScene は必須引数 sceneKey のみでよいと判断します。sceneId、自然言語、aliasではなく、scenes が返した正確な sceneKey を使うことをTool説明に明記するべきです。

## 6. initialize応答方針
以下の値で問題ないと判断します。
- SERVER_NAME: switchbot-ctrl-mcp-java
- SERVER_TITLE: SwitchBot Control MCP Java Server
- SERVER_VERSION: 0.1.0
instructions には以下を入れる方針が妥当です。
- SwitchBotシーンを安全に一覧取得・実行するMCPサーバー
- デバイス直接操作ではなく公開許可済みSceneのみ扱う
- scenes で利用可能な sceneKey を確認する
- executeScene で sceneKey を指定して実行する
- sceneId は直接指定しない
- cooldown / dry-run はサーバー側で制御される
- refreshScenes はTool公開していない

## 7. SceneService生成方針
SceneService は McpServlet のフィールドとして保持する方針でよいと判断します。
理由は明確です。
- SceneService が24時間キャッシュを持つ
- cooldown履歴を lastExecutedAtBySceneKey で保持する
- tools/callごとにnewすると cooldown / cache の意味が弱くなる
init() で AppConfig.load()、SwitchBotApisV2、SceneService を生成する方針で問題ないと判断します。
Servlet lifecycle上、init() 失敗時は起動失敗として扱うのが自然な整理です。
dry-run は Ver.1 では false 固定で問題ないと判断します。将来案としては、AppConfigを広げずに McpServlet 側で SWITCHBOT_MCP_DRY_RUN のような環境変数を読む方式が自然な整理です。

## 8. tools/list実装方針
ToolsListResponse.Tool、InputSchema、Property はサンプルと同じ作法で使えます。
scenes は以下の定義で十分と判断します。
- inputSchema.type = object
- additionalProperties = false
- requiredなし
executeScene は以下の定義にします。
- inputSchema.type = object
- additionalProperties = false
- sceneKey property
- sceneKey required

## 9. tools/call実装方針
resToolsCall では、サンプル同様に Tool名を取り出して switch 分岐する方針で問題ないと判断します。
- scenes → sceneService.listScenes()
- executeScene → required argument sceneKey → sceneService.executeBySceneKey(sceneKey)
- 未知Tool → null を返して BaseMcpServlet に任せる
- Tool名欠落 → Tool errorとして errorResponse

## 10.  エラー応答方針
1号機と同じく、Tool内部エラーは JSON-RPC error ではなく ToolsCallResponse.result.isError=true で返す方針が妥当です。
SceneServiceException もその他例外も catch して、errorResponse に寄せる形でよいと判断します。
未知Toolだけは null を返して BaseMcpServlet の Unknown tool に任せる整理が一貫しています。

## 11.  作成ファイル・配置案
既存プロジェクトではServletが vr46.switchbotctrlmcp.web 配下にあります。
そのため、自然な配置は以下です。
src/main/java/vr46/switchbotctrlmcp/web/McpServlet.java
ただし、1号機と同じ package 感を最優先するなら、依頼文の案どおり以下でも成立します。
src/main/java/vr46/switchbotctrlmcp/McpServlet.java
推奨は前者です。理由は、既存の HelloServlet / TestServlet が web package にあり、業務ロジックの switchbot package とWeb入口を分離できるためです。

## 12.  今回やらないこと
以下は今回も実装時も触らない方針でよいと判断します。
- BaseMcpServlet 修正
- DTO修正
- SceneService 修正
- SwitchBotApisV2 修正
- SceneCatalog / sceneCatalog.json 修正
- refreshScenes Tool公開
- alias / 自然言語マッチング
- cooldown / dry-run の再実装
- 監査ログ追加
- 複雑な共通化

###  Go / NoGo 判断
判断: Go
理由:
- BaseMcpServlet のoverrideポイントだけで実装可能です。
- 1号機 McpServlet の構造をほぼそのまま踏襲できます。
- SceneService のpublic APIは今回の2 Tool設計に適合しています。
- SceneService はcache / cooldownを内部保持しており、Servletフィールド保持方針が妥当です。
- 変更禁止領域に手を入れずに実装できます。
製造前に確認すべきこと:
- Servlet実行環境に TOKEN / SECRET 環境変数が設定されていること
- dry-runを Ver.1 で本当に false 固定にすること
- 配置packageを vr46.switchbotctrlmcp.web にするか、1号機寄せでroot packageにするか

###  推奨する次の作業
次フェーズで McpServlet を1ファイル追加し、サンプルと同じ構造で実装するのがよいと判断します。
推奨配置は vr46.switchbotctrlmcp.web.McpServlet、URLは @WebServlet で /mcp です。実装後は compile / test と、MCPの initialize、tools/list、tools/call scenes、tools/call executeScene の疎通確認を行う流れが妥当です。 

---
# 製造前に確認すべきこと:
## Servlet実行環境に TOKEN / SECRET 環境変数が設定されていること
確認済みです。

# dry-runを Ver.1 で本当に false 固定にすること
false 固定です

# 配置packageを vr46.switchbotctrlmcp.web にするか、1号機寄せでroot packageにするか
vr46.switchbotctrlmcp.webです。

これはサンプル側を合わせてあげるべきでした。判断に迷う元を与えてしまいました。私の配慮不足ですね。

---
この認識で問題なければGo判断でよいと判断します。
ChatGPTの方で指摘や問題なければ、製造用プロンプトを出力してください。