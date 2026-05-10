# 依頼：SwitchBot制御用 McpServlet の製造

あなたは Java Servlet / MCP Server の実装支援者です。

前工程で、SwitchBot制御用 `McpServlet` の設計前分析は完了しています。  
今回は、その確定方針に基づき、実際に `McpServlet` を作成してください。

この依頼では **コード変更・ファイル作成を行ってよい** です。

---

## 作業対象プロジェクト

```text
[user home masked]\IdeaProjects\switchbot-ctrl-mcp
```

---

## 作成するファイル

以下に `McpServlet.java` を新規作成してください。

```text
src/main/java/vr46/switchbotctrlmcp/web/McpServlet.java
```

package は以下です。

```java
package vr46.switchbotctrlmcp.web;
```

---

## 参照用サンプル

1号機MCPサーバーの `McpServlet` を、参照用として以下に置いてあります。

```text
reference/SampleMcpServlet.java.txt
```

重要：

* このファイルは参照用です。
* 編集しないでください。
* コンパイル対象に移動しないでください。
* 今回の `McpServlet` は、この1号機McpServletと同じ構造・同じ作風で実装してください。
* ソースコードの美しさより、運用者が1号機と同じ感覚で保守できることを優先してください。
* 余計な共通化や抽象化はしないでください。

---

## BaseMcpServlet

共通MCP基底クラスは以下にあります。

```text
[user home masked]\IdeaProjects\MyCommonLibV2\src\baseClass\BaseMcpServlet.java
```

重要：

* `BaseMcpServlet` は完成済み・稼働実績ありの共通基盤です。
* 今回は変更禁止です。
* `BaseMcpServlet` を修正しないでください。
* `BaseMcpServlet` の拡張ポイントだけを利用してください。

主なoverride対象は以下です。

```java
protected InitializeResponse resInitialize(InitializeRequest initializeRequest)

protected ToolsListResponse resToolsList(ToolsListRequest toolsListRequest)

protected ToolsCallResponse resToolsCall(ToolsCallRequest toolsCallRequest)
```

---

## 既存クラス

SwitchBot制御ロジックはすでに以下に実装済みです。

```text
vr46.switchbotctrlmcp.switchbot.AppConfig
vr46.switchbotctrlmcp.switchbot.SceneService
vr46.switchbotctrlmcp.switchbot.SceneSummary
vr46.switchbotctrlmcp.switchbot.SceneExecutionResult
vr46.switchbotctrlmcp.switchbot.SceneServiceException
```

共通ライブラリ側には以下があります。

```text
switchBot.SwitchBotApisV2
```

---

## SceneService の役割

`SceneService` は今回のMCPサーブレットから呼び出す中心サービスです。

public API は以下です。

```java
public List<SceneSummary> listScenes()

public List<SceneSummary> refreshScenes()

public SceneExecutionResult executeBySceneKey(String sceneKey)
```

ただし、今回MCP Toolとして公開するのは以下だけです。

```text
listScenes()
executeBySceneKey(sceneKey)
```

`refreshScenes()` は保守用メソッドであり、MCP Toolとして公開しないでください。

---

## 今回公開するTool

公開するToolは2つだけです。

```text
scenes
executeScene
```

`refreshScenes` はTool公開しないでください。

---

# Tool 1: scenes

## 目的

公開可能なSwitchBotシーン一覧を返します。

## 呼び出す処理

```java
sceneService.listScenes()
```

## 引数

なし。

## 戻り値

`List<SceneSummary>` をJSON文字列化して返してください。

## Tool説明方針

LLMに以下が伝わるように説明してください。

* 利用可能なSwitchBotシーン一覧を取得するTool
* 返却される `sceneKey` を、`executeScene` Toolの引数として使う
* SwitchBotアプリに存在する全シーンではなく、MCP公開許可済みのシーンだけが返る
* ユーザーが「何ができる？」と聞いた場合や、実行前に利用可能シーンを確認したい場合に使う

---

# Tool 2: executeScene

## 目的

指定された `sceneKey` のSwitchBotシーンを実行します。

## 呼び出す処理

```java
sceneService.executeBySceneKey(sceneKey)
```

## 引数

必須引数：

```text
sceneKey
```

型：

```text
string
```

説明：

```text
Scene key returned by the scenes tool. Example: sample_lighting_off.
```

## 戻り値

`SceneExecutionResult` をJSON文字列化して返してください。

## Tool説明方針

LLMに以下が伝わるように説明してください。

* `scenes` Toolで返された `sceneKey` を指定してシーンを実行する
* `sceneId` ではなく `sceneKey` を使う
* 自然言語やaliasではなく、正確な `sceneKey` を使う
* 実行結果には `success`, `executed`, `dryRun`, `message` が含まれる
* cooldown中やdry-run時は、SwitchBot APIを呼ばない場合がある

---

## initialize の設定

`resInitialize` では、1号機McpServletと同じ構造で以下を設定してください。

```java
SERVER_NAME = "switchbot-ctrl-mcp-java";
SERVER_TITLE = "SwitchBot Control MCP Java Server";
SERVER_VERSION = "0.1.0";
```

`instructions` では、以下を明示してください。

* SwitchBotシーンを安全に一覧取得・実行するMCPサーバーである
* デバイス直接操作ではなく、公開許可済みSceneのみを扱う
* `scenes` で利用可能な `sceneKey` を確認できる
* `executeScene` で `sceneKey` を指定して実行できる
* `sceneId` を直接指定しない
* cooldown / dry-run の実行制御はサーバー側で行われる
* `refreshScenes` はTool公開していない

---

## SceneService の生成

`SceneService` は `McpServlet` のフィールドとして保持してください。

重要：

* `SceneService` を `tools/call` のたびにnewしないでください。
* キャッシュとcooldown履歴を維持するため、Servletのフィールドとして保持してください。
* `init()` で生成してください。

想定：

```java
private SceneService sceneService;

@Override
public void init() {
    AppConfig config = AppConfig.load();
    SwitchBotApisV2 api = new SwitchBotApisV2(config.TOKEN, config.SECRET);
    this.sceneService = new SceneService(api, false);
}
```

dry-runは Ver.1 では **false固定** としてください。

環境変数による切り替え案は今回実装しないでください。
AppConfigも変更しないでください。

---

## 実装構造

`reference/SampleMcpServlet.java.txt` と同じ構造をできるだけ踏襲してください。

以下のような構成にしてください。

```text
- static final ObjectMapper mapper
- SERVER_NAME / SERVER_TITLE / SERVER_VERSION
- TOOL_* 定数
- SceneService フィールド
- constructorで super(SERVER_VERSION)
- init()
- resInitialize override
- resToolsList override
- addScenesTool private method
- addExecuteSceneTool private method
- resToolsCall override
- scenesToolResponse private method
- executeSceneToolResponse private method
- successResponse helper
- errorResponse helper
- baseToolCallResponse helper
- toErrorJson helper
- getToolName helper
- getRequiredArgument helper
- trimToNull helper
- logException helper
- safeMessage helper
- safeForJson helper
```

1号機と同じ運用感を優先してください。
helperを外部クラス化したり、過度に共通化したりしないでください。

---

## 既存Request / Response DTO

以下の既存DTOを使ってください。

```text
llm.mcp.JsonRpcRequest.InitializeRequest
llm.mcp.JsonRpcRequest.ToolsListRequest
llm.mcp.JsonRpcRequest.ToolsCallRequest

llm.mcp.JsonRpcResponse.InitializeResponse
llm.mcp.JsonRpcResponse.ToolsListResponse
llm.mcp.JsonRpcResponse.ToolsCallResponse
```

`ToolsCallRequest` は `getParams().getArguments().getValue("sceneKey")` のように引数取得できます。

`ToolsListResponse` は `Tool`, `InputSchema`, `Property` を使ってTool定義を作れます。

---

## tools/list 実装

### scenes

* inputSchema.type = `"object"`
* additionalProperties = `false`
* requiredなし
* propertiesなし

### executeScene

* inputSchema.type = `"object"`
* additionalProperties = `false`
* property `sceneKey`
* required `sceneKey`

---

## tools/call 実装

`resToolsCall` では、1号機と同じようにTool名を取り出してswitch分岐してください。

```text
scenes
  -> scenesToolResponse(...)

executeScene
  -> executeSceneToolResponse(...)

未知Tool
  -> nullを返してBaseMcpServletに任せる

Tool名欠落
  -> errorResponse(...)
```

---

## エラー方針

1号機McpServletと同じ方針にしてください。

### 未知Tool

`resToolsCall` で `null` を返し、`BaseMcpServlet` に任せてください。

### Tool内部エラー

JSON-RPC errorではなく、`ToolsCallResponse.result.isError=true` のToolエラーとして返してください。

つまり、以下のようにしてください。

```text
SceneServiceException
その他例外
  -> catch
  -> logException(...)
  -> errorResponse(...)
```

`content.text` はerror JSON文字列にしてください。

---

## 今回やらないこと

以下は今回の対象外です。

```text
BaseMcpServletの修正
JsonRpcRequest / JsonRpcResponse DTOの修正
SceneServiceの修正
SwitchBotApisV2の修正
SceneCatalogの修正
sceneCatalog.jsonの修正
refreshScenesのTool公開
alias / 自然言語マッチング
LLMによる意図解釈
cooldown / dry-run の再実装
監査ログの追加
複雑な共通化
AppConfigの修正
dry-runの環境変数対応
```

今回は以下に集中してください。

```text
McpServlet
+
SceneServiceの2Tool公開
+
1号機McpServletと同じ構造
```

---

## 実装後に確認してほしいこと

作業後、以下を確認してください。

```text
McpServlet.java が src/main/java/vr46/switchbotctrlmcp/web に作成されている
package が vr46.switchbotctrlmcp.web になっている
@WebServlet の urlPatterns が /mcp になっている
BaseMcpServlet を修正していない
SampleMcpServlet を修正していない
SceneService を tools/call ごとに new していない
SceneService をフィールド保持している
SceneService を init() で生成している
dry-run が false 固定になっている
refreshScenes が Tool 公開されていない
tools/list に scenes と executeScene だけが出る
executeScene の必須引数が sceneKey になっている
sceneId をTool引数にしていない
scenes Tool が sceneService.listScenes() を呼んでいる
executeScene Tool が sceneService.executeBySceneKey(sceneKey) を呼んでいる
Tool内部エラーは isError=true で返している
未知Toolは null を返してBaseMcpServletに任せている
1号機McpServletと同じ構造・helper配置になっている
```

---

## テストについて

可能であれば、以下を確認してください。

```text
initialize
notifications/initialized
tools/list
tools/call scenes
tools/call executeScene
未知tool
sceneKey欠落
```

CLIでMaven実行ができない場合は、その理由を報告してください。
IntelliJやServlet上での確認でも構いません。

---

## 出力してほしい報告形式

作業後、以下の形式で報告してください。

```markdown
# SwitchBot McpServlet 製造結果

## 1. 作成・変更したファイル

## 2. 実装したMCP Server情報

## 3. 公開Tool

## 4. SceneService生成方針

## 5. tools/list仕様

## 6. tools/call仕様

## 7. エラー応答方針

## 8. 1号機McpServletとの構造整合

## 9. テスト・確認結果

## 10. 注意点・未確認事項
```

---

## 重要な制約

* 今回は製造フェーズなのでコード作成してよい
* BaseMcpServletは変更しないこと
* SampleMcpServletは変更しないこと
* SceneServiceや下位クラスは変更しないこと
* SwitchBotApisV2は変更しないこと
* AppConfigは変更しないこと
* refreshScenesをTool公開しないこと
* dry-runはfalse固定にすること
* 1号機McpServletと同じ構造を優先すること