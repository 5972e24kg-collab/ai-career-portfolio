# 依頼：SwitchBot制御用 McpServlet 実装前の要求理解・設計方針・Go/NoGo判断

あなたは Java Servlet / MCP Server の設計・実装支援者です。

今回は、SwitchBot制御用MCPサーバーの最終実装フェーズとして、`McpServlet` を作成する前段階の分析を行ってください。

ただし、この依頼では **コード変更・ファイル作成・実装は絶対に行わないでください**。

今回の目的は、いきなり製造することではなく、以下を確認することです。

- 既存の `BaseMcpServlet` の拡張ポイントを理解できているか
- 1号機MCPサーバーの `McpServlet` と同じ構造で実装できるか
- SwitchBot向けTool設計が妥当か
- `SceneService` の呼び出し方が妥当か
- 変更禁止領域に手を入れずに実装できるか
- 製造に進んでよいか

---

## 作業対象プロジェクト

今回のMCPサーバーのプロジェクトは以下です。

```text
[local project path masked]
```

---

## 参照用サンプル

1号機MCPサーバーの `McpServlet` を、参照用として以下に置いてあります。

```text
[local reference path masked]\SampleMcpServlet.java.txt
```

このファイルは **コンパイル対象ではない参照用サンプル** です。

重要：

* このファイルは編集しないでください。
* このファイルをコンパイル対象へ移動しないでください。
* 実装構造、メソッド配置、helper配置、エラーレスポンスの作り方、Tool定義の書き方の参考にしてください。
* 今回のMcpServletは、可能な限りこの1号機McpServletと同じ構造・同じ作風で実装する方針です。

---

## BaseMcpServlet

共通MCP基底クラスは以下にあります。

```text
[local common library path masked]\BaseMcpServlet.java
```

このクラスは今回のプロジェクトから参照可能です。

重要：

* `BaseMcpServlet` は完成済み・個人利用の範囲で継続利用している共通基盤です。
* 変更禁止です。
* 今回の作業で `BaseMcpServlet` を修正しないでください。
* `BaseMcpServlet` の拡張ポイントだけを利用してください。

`BaseMcpServlet` は、JSON-RPC / MCPの共通処理、セッション管理、`initialize` / `tools/list` / `tools/call` の分岐を持っています。
サブクラス側では主に以下をoverrideする想定です。

```java id="esrc7w"
protected InitializeResponse resInitialize(InitializeRequest initializeRequest)

protected ToolsListResponse resToolsList(ToolsListRequest toolsListRequest)

protected ToolsCallResponse resToolsCall(ToolsCallRequest toolsCallRequest)
```

---

## 既存の関連クラス

今回のSwitchBot制御ロジックは、すでに以下に実装済みです。

```text id="z0kqgc"
vr46.switchbotctrlmcp.switchbot.AppConfig
vr46.switchbotctrlmcp.switchbot.SceneCatalog
vr46.switchbotctrlmcp.switchbot.SceneCatalogEntry
vr46.switchbotctrlmcp.switchbot.SceneCatalogException
vr46.switchbotctrlmcp.switchbot.SceneSummary
vr46.switchbotctrlmcp.switchbot.SwitchBotScenesConverter
vr46.switchbotctrlmcp.switchbot.SwitchBotScenesConvertException
vr46.switchbotctrlmcp.switchbot.SwitchBotScenesResponse
vr46.switchbotctrlmcp.switchbot.SceneService
vr46.switchbotctrlmcp.switchbot.SceneExecutionResult
vr46.switchbotctrlmcp.switchbot.SceneServiceException
```

共通ライブラリ側には以下があります。

```text id="du8lpc"
switchBot.SwitchBotApisV2
switchBot.SwitchBotApiResult
switchBot.SwitchBotApiException
```

---

## SceneService の役割

`SceneService` は、今回のMCPサーブレットから呼び出す中心サービスです。

主なpublic APIは以下です。

```java id="q0zbqf"
public List<SceneSummary> listScenes()

public List<SceneSummary> refreshScenes()

public SceneExecutionResult executeBySceneKey(String sceneKey)
```

ただし、今回MCP Toolとして公開するのは以下のみです。

```text id="7mp1lj"
listScenes()
executeBySceneKey(sceneKey)
```

`refreshScenes()` は保守用メソッドであり、MCP Toolとしては公開しません。

理由：

* LLMが迷うToolを増やしたくない
* 今回のMCPは小型モデルでも扱えるように、Tool数を最小限にしたい
* refreshScenesは人間・保守用のescape hatchであり、LLMに公開する必要はない

---

## 今回作るMcpServletの目的

今回作る `McpServlet` は、`SceneService` をMCP Toolとして公開する薄いサーブレットです。

MCPサーブレット側の責務は以下に限定してください。

```text id="xi19iy"
- initialize応答のserverInfo / instructions設定
- tools/listでTool定義を返す
- tools/callでtoolNameを判定する
- tools/callで必要な引数を取り出す
- SceneServiceを呼ぶ
- SceneServiceの戻り値をJSON文字列としてMCP responseに載せる
- 例外時にToolsCallResponse.result.isError=trueで返す
```

MCPサーブレット側でやらないこと：

```text id="7cpjkj"
- SwitchBot APIを直接呼ばない
- sceneCatalog.jsonを直接読まない
- sceneKeyからsceneIdへの変換をしない
- cooldown判定をしない
- dry-run判定をしない
- キャッシュ制御をしない
- alias / 自然言語解釈をしない
- LLMの推論結果を補正しない
- BaseMcpServletを変更しない
```

これらはすでに `SceneService` または下位層の責務です。

---

## Tool設計

今回公開するToolは2つにしてください。

```text id="b4baz6"
scenes
executeScene
```

`refreshScenes` はTool公開しないでください。

---

## Tool 1: scenes

### 目的

公開可能なSwitchBotシーン一覧を返します。

### 呼び出す処理

```java id="bd5paz"
sceneService.listScenes()
```

### 引数

なし。

### 戻り値

`List<SceneSummary>` をJSON文字列化して返します。

### Tool説明方針

LLMには以下が伝わるように説明してください。

* 利用可能なSwitchBotシーン一覧を取得するTool
* 返却される `sceneKey` を、`executeScene` Toolの引数として使う
* SwitchBotアプリに存在する全シーンではなく、MCP公開許可済みのシーンだけが返る
* ユーザーが「何ができる？」と聞いた場合や、実行前に利用可能シーンを確認したい場合に使う

---

## Tool 2: executeScene

### 目的

指定された `sceneKey` のSwitchBotシーンを実行します。

### 呼び出す処理

```java id="zbv86t"
sceneService.executeBySceneKey(sceneKey)
```

### 引数

必須引数：

```text id="frayvl"
sceneKey
```

型：

```text id="ux1a8l"
string
```

説明：

```text id="t851wu"
Scene key returned by the scenes tool. Example: lighting_scene_off_sample.
```

### 戻り値

`SceneExecutionResult` をJSON文字列化して返します。

### Tool説明方針

LLMには以下が伝わるように説明してください。

* `scenes` Toolで返された `sceneKey` を指定してシーンを実行する
* `sceneId` ではなく `sceneKey` を使う
* 自然言語やaliasではなく、正確な `sceneKey` を使う
* 実行結果には `success`, `executed`, `dryRun`, `message` が含まれる
* cooldown中やdry-run時は、SwitchBot APIを呼ばない場合がある

---

## initialize の説明

`resInitialize` では、1号機McpServletと同じ構造で、以下を設定してください。

```text id="av9x1j"
SERVER_NAME
SERVER_TITLE
SERVER_VERSION
instructions
capabilities.tools.listChanged
serverInfo.name
serverInfo.title
serverInfo.version
```

サーバー名・タイトル案：

```text id="d8ptxa"
SERVER_NAME = "switchbot-ctrl-mcp-java"
SERVER_TITLE = "SwitchBot Control MCP Java Server"
SERVER_VERSION = "0.1.0"
```

instructions では、以下を明示してください。

* SwitchBotシーンを安全側に寄せて一覧取得・実行するMCPサーバーである
* デバイス直接操作ではなく、公開許可済みSceneのみを扱う
* `scenes` で利用可能な sceneKey を確認できる
* `executeScene` で sceneKey を指定して実行できる
* `sceneId` を直接指定しない
* cooldown / dry-run の実行制御はサーバー側で行われる
* refreshScenesはTool公開していない

---

## SceneService の生成

`SceneService` はMcpServlet内で保持してください。

重要：

* `SceneService` をtools/callのたびにnewしないでください。
* キャッシュとcooldown履歴を維持するため、Servletのフィールドとして保持してください。
* `init()` で生成する方針を検討してください。

想定：

```java id="j4xecz"
private SceneService sceneService;

@Override
public void init() {
    AppConfig config = AppConfig.load();
    SwitchBotApisV2 api = new SwitchBotApisV2(config.TOKEN, config.SECRET);
    this.sceneService = new SceneService(api, false);
}
```

dry-runについて：

* Ver.1では、個人管理下の検証・運用範囲を前提に `false` 固定でも構いません。
* 既存の `SceneService` はdry-run対応済みです。
* もし環境変数などで切り替える案が自然なら提案してください。
* ただし、今回AppConfigの改修に話を広げすぎないでください。

---

## 1号機McpServletと同じ構造にすること

今回の最重要要件です。

`reference/SampleMcpServlet.java.txt` を読み、以下の構造をできるだけ踏襲してください。

```text id="spo2y0"
- static final ObjectMapper mapper
- SERVER_NAME / SERVER_TITLE / SERVER_VERSION
- TOOL_* 定数
- constructorで super(SERVER_VERSION)
- resInitialize override
- resToolsList override
- addXxxTool private method
- resToolsCall override
- xxxToolResponse private method
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

ソースコードの美しさより、**運用者が1号機と同じ感覚で保守できること**を優先してください。

余計に共通化しすぎないでください。

---

## 既存Request / Response DTO

以下の既存DTOを使ってください。

```text id="4f0db2"
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

## エラー方針

1号機McpServletと同じ方針にしてください。

tools/call内の業務例外は、JSON-RPC errorではなく、`ToolsCallResponse.result.isError=true` のToolエラーとして返します。

つまり：

```text id="ywas8z"
未知Tool:
  nullを返してBaseMcpServletに任せる

Tool内部エラー:
  errorResponse(...)
  result.isError=true
  content.text は error JSON
```

SceneServiceException やその他例外はcatchし、`errorResponse` にしてください。

---

## 今回やらないこと

以下は今回の対象外です。

```text id="oaxr4i"
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
```

今回は、以下に集中してください。

```text id="j9hhfa"
McpServlet
+
SceneServiceの2Tool公開
+
1号機McpServletと同じ構造
```

---

## 確認してほしいこと

### 1. 参照ファイル確認

以下を確認してください。

```text id="k4yj4h"
reference/SampleMcpServlet.java.txt
BaseMcpServlet
```

* SampleMcpServlet の構造を把握する
* BaseMcpServlet のoverrideポイントを確認する
* 今回のMcpServletに流用すべき構造を整理する

### 2. 作成ファイル案

今回作成すべきServletのpackageと配置先を提案してください。

想定：

```text id="sxqxxw"
src/main/java/vr46/switchbotctrlmcp/McpServlet.java
```

または既存package構成に合わせて自然な配置を提案してください。

### 3. Tool名

以下でよいか確認してください。

```text id="wak4kh"
scenes
executeScene
```

### 4. SceneService生成

`init()` で `SceneService` を生成し、フィールド保持する方針でよいか確認してください。

### 5. dry-run

Ver.1では、個人管理下の検証・運用範囲を前提に `new SceneService(api, false)` でよいか確認してください。

もし、環境変数で切り替える案が自然であれば、提案だけしてください。
ただし、今回の製造ではAppConfig改修に広げない方針です。

### 6. refreshScenes

Tool公開しない方針を確認してください。

---

## Go / NoGo 判断

最後に、次の形式で判断してください。

```text id="apj9eo"
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
* BaseMcpServletを変更しないこと
* SampleMcpServletを変更しないこと
* SceneServiceや下位クラスを変更しないこと
* refreshScenesをTool公開しないこと
* 1号機McpServletと同じ構造を優先すること