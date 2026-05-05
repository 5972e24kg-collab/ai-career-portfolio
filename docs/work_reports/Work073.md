# Work073: Portainer MCP構築・Open WebUI連携・Java自作MCP基盤

Tag: [[MCP]], [[Portainer]], [[OpenWebUI]], [[Ollama]], [[gemma4:e4b]], [[gpt-oss:120b]], [[Java]], [[Servlet]], [[JSON-RPC]], [[Docker]], [[LLM Tools]]

## 1. 目的

* Portainer を Docker コンテナ監視・観測用ハブとして再構築する
* 公式 Portainer MCP Server を構築し、MCP連携の実挙動を体験する
* Open WebUI 経由で `gemma4:e4b` と MCP を接続し、ローカルLLMがMCPツールをどの程度扱えるか確認する
* Portainer MCPが提供する低レベルツールを、ローカルLLMが実用的に扱えるか検証する
* MCPの仕様、JSON-RPC、Streamable HTTP、`tools/list`、`tools/call`、`Mcp-Session-Id` の基本を体得する
* SDKやSpring AIに依存せず、Java Servletで最小MCPサーバーをフルスクラッチ実装する
* 最終的に、Portainer観測用の高レベルMCPツールを自作できる基盤を構築する

## 2. システム環境

* **Portainer Server Node:** `[Host name masked]`

* **IP:** `[IP address masked]/24`

* **OS:** Ubuntu Server 24.04.3

* **Docker:** Docker Engine 29.4.1

* **Docker Compose:** v5.1.3

* **Work Dir:** `[Path masked]`

* **Portainer Server:** `portainer/portainer-ce:2.31.2`

* **Portainer Agent:** `portainer/agent:2.31.2`

* **Portainer MCP:** `portainer-mcp v0.7.0`

* **MCP Bridge:** `mcpo`

* **MCPO Endpoint:** `[internal MCP endpoint masked]`

* **Self-made MCP Endpoint:** `[internal MCP endpoint masked]/portainer-ops-mcp-1.0/mcp`

* **LLM Node:** Node C

* **IP:** `[IP address masked]`

* **OS:** Windows 11 Pro + WSL

* **LLM Runtime:** Ollama

* **検証モデル:**

  * `gemma4:e4b`
  * `gpt-oss:120b`

* **UI / MCP Client:** Open WebUI

* **Agent Target Host:** `[Host name masked]`

* **用途:** 自宅APIコンテナサーバー

* **Portainer Agent Port:** `9001`

* **Java MCP Project:** `portainer-ops-mcp`

* **Runtime:** Tomcat / Java Servlet

* **Transport:** MCP Streamable HTTP 最小実装

* **Protocol:** JSON-RPC 2.0 / MCP `[Date masked]`

## 3. 作業ログ・解決プロセス

### 課題A: Portainerの再構築方針決定

#### 発生した課題

* 既存Portainerは古く、運用上ほぼ使われていなかった
* 今回の目的はPortainer運用そのものではなく、MCP連携の体得であった
* Portainerの最新版を入れると、公式Portainer MCPとの対応バージョンがずれる可能性があった

#### 解決策

* Portainerは最新版ではなく、公式Portainer MCP対応の `2.31.2` に固定
* Portainer Agentも同じく `2.31.2` に固定
* Portainer MCPは `v0.7.0` を採用
* 最初は write 操作を許可せず、read-only 構成で検証する方針とした

---

### 課題B: Portainer Server構築

#### 発生した課題

* 新規VM `[Host name masked]` 上に、Portainer ServerをDockerコンテナで構築する必要があった
* 初期管理者作成前に時間切れとなり、`adminmonitor timed out` が発生した
* Local環境追加時にPodman系の選択が混ざった可能性があった

#### 解決策

* `[Path masked]` を作業ディレクトリとして使用
* `portainer/portainer-ce:2.31.2` をDocker Composeで起動
* Portainer UI/API の疎通を確認
* `curl -k` でHTTPS疎通確認
* Portainer UIで初期管理者を作成
* 認証情報を安全な非公開設定として管理
* `/api/status` と `/api/endpoints` でAPI疎通確認

---

### 課題C: Portainer Agent接続

#### 発生した課題

* Portainer Serverから、実際にコンテナを抱えたホストの情報を参照する必要があった
* Agent通信には、Portainer Server → Docker Host `[port masked]` の疎通が必要だった

#### 解決策

* 自宅APIコンテナサーバー `[Host name masked]` に `portainer/agent:2.31.2` を配置
* Agent接続に必要な疎通確認を実施
* Portainer UIから `[Host name masked]` をAgent環境として追加
* API `/api/endpoints` で `local` と `[Host name masked]` が正常に見えることを確認

---

### 課題D: 公式 Portainer MCP + mcpo 構築

#### 発生した課題

* Portainer公式MCPはそのままOpen WebUIから扱うにはstdio型に近く、HTTP/OpenAPI化が必要だった
* Open WebUIでは、OpenAPI Toolとして登録する場合とMCP Streamable HTTPとして登録する場合が異なっていた
* 初期登録時に `/openapi.json` まで含めて指定したため、URL解釈がずれた

#### 解決策

* `mcpo` を利用し、Portainer MCPをOpenAPI互換HTTPサーバーとして公開
* MCPO endpointを `http://[IP address masked]:8010` とした
* Open WebUIには `http://[IP address masked]:8010` をOpenAPI Toolとして登録
* `/openapi.json` ではなくベースURLを登録するのが正解と確認
* Open WebUIでは「ユーザー設定」ではなく「管理者パネル」側から登録する必要があることを確認
* read-onlyモードでPortainer MCPを起動し、OpenAPI定義取得・Node Cからの疎通を確認

---

### 課題E: gemma4:e4b から公式Portainer MCPを利用

#### 発生した課題

* `listEnvironments` は呼び出せたが、コンテナ一覧までは自然に取得できなかった
* `dockerProxy` を使う必要があったが、LLMがDocker APIパスとPortainer APIパスを混同した
* `gemma4:e4b` はツール仕様の理解が不安定で、実行せずにテンプレート作文する挙動が見られた
* 英語のOpenAPI schemaやDocker API説明に引っ張られ、応答が英語寄りになることがあった

#### 解決策

* `dockerProxy` に渡すべきパスはPortainer APIではなくDocker Engine APIであると整理

  * 正: `/containers/json`
  * 誤: `/api/endpoints/9/docker/containers`
* `dockerProxy` をcurlから直接叩き、公式MCP自体はコンテナ一覧を取得可能であることを確認
* 問題はMCP接続ではなく、LLMに低レベル汎用ツールを直接使わせる設計にあると判断
* 公式Portainer MCPは学習・検証には有効だが、実運用には高レベルラッパーが必要という結論に到達

---

### 課題F: gpt-oss:120bでの再検証

#### 発生した課題

* ローカル上位モデルである `gpt-oss:120b` でも、Portainer公式MCPの低レベルツールを安定的には扱えなかった
* コンテナ監視のためのAPI候補を大量に提示するなど、回答量が多くなり、実データ取得との区別が分かりにくくなった
* `dockerProxy` にPortainer APIパスを渡し、`page not found` を受けるケースがあった

#### 解決策

* モデルサイズを上げても、低レベルAPIをLLMに直接見せる設計自体に限界があると確認
* `dockerProxy(environmentId, method, dockerAPIPath, queryParams)` のような低レベル汎用ツールではなく、LLM向けに意味単位で分割したTool設計が必要と判断
* 高レベルTool案として以下を定義

  * `listEnvironments`
  * `listContainers`
  * `getContainerLogs`
  * `getContainerStats`
  * `summarizeEnvironmentHealth`

---

### 課題G: JavaでMCPサーバーを自作する方針決定

#### 発生した課題

* MCP Java SDKやSpring AI MCP Server Boot Starterを使うべきか検討が必要だった
* 目的は「早く作ること」ではなく、「MCP仕様を理解して体得すること」だった
* SDKやSpring AIを使うと、JSON-RPCやMCPのプロトコル処理が隠蔽される懸念があった

#### 解決策

* Java Servlet + JacksonでMCPサーバーをフルスクラッチ実装する方針に決定
* MCP仕様全体を実装するのではなく、Portainer観測用途に必要なMCP Server最小サブセットを実装対象とした
* 当初は `SSE方式` と捉えていたが、現行仕様に合わせて `Streamable HTTP方式` を目標とした
* `POST /mcp` によるJSON-RPC request処理を中心に実装
* `GET /mcp` によるSSEストリームは初期段階では未対応とし、`405 Method Not Allowed` を返す方針にした

---

### 課題H: 最小MCPサーバーの実装

#### 発生した課題

* MCPサーバーとして、最低限どのJSON-RPC methodに応答すべきか整理が必要だった
* Open WebUIから実際にMCP Serverとして呼ばせるには、`initialize`、`tools/list`、`tools/call` の流れを成立させる必要があった

#### 解決策

* `McpServlet` を作成し、`/mcp` にマッピング
* 以下を実装

  * `initialize`
  * `notifications/initialized`
  * `tools/list`
  * `tools/call`
  * `ping`
* PowerShellから `Invoke-WebRequest` で動作確認
* Open WebUIの `MCP Streamable HTTP` として登録
* Open WebUIログ上で以下の呼び出しを確認

  * `initialize`
  * `tools/list`
  * `tools/call`
  * `ping`

---

### 課題I: ダミーJSONによるTool追加

#### 発生した課題

* Portainer実データ変換ロジックを先に実装すると、MCP学習用ソースが読みにくくなる
* まずはMCP toolの定義と呼び出しを理解する必要があった

#### 解決策

* `resources` 配下にダミーJSONを配置
* `listEnvironments`
* `listContainers(environmentName)`
* 上記2ツールをMCPに追加
* `tools/list` で3つのToolを返すように変更

  * `ping`
  * `listEnvironments`
  * `listContainers`
* `tools/call` でダミーJSONを `content[].text` に格納して返却
* Open WebUIから `gemma4:e4b` 経由でDocker環境一覧が表形式で返ることを確認

---

### 課題J: MCP-Protocol-VersionとMcp-Session-Idの扱い

#### 発生した課題

* `MCP-Protocol-Version` ヘッダを取得・検証していなかった
* `Mcp-Session-Id` によるセッション管理を後から入れると、実装の差し替え範囲が広くなる懸念があった
* Open WebUIが `notifications/initialized` を必ず送るとは限らないため、厳密実装との互換性に注意が必要だった

#### 解決策

* `MCP-Protocol-Version` ヘッダを取得し、未対応バージョンを拒否する処理を追加
* `initialize` 時に `Mcp-Session-Id` を発行
* `Mcp-Session-Id` を `Access-Control-Expose-Headers` に含めるよう修正
* `ConcurrentHashMap` でセッションを保持
* TTLを1時間に設定
* `DELETE /mcp` でセッション破棄できるように実装
* Open WebUI互換性を考慮し、`notifications/initialized` が来ない場合でも互換モードでTool呼び出しを許容
* 将来的には厳密モードへ切り替え可能なフラグを設けた

---

### 課題K: BaseMcpServlet化と部品化

#### 発生した課題

* 最小MCPサーバーを作れたが、今後MCPサーバーを量産するには共通処理を毎回書くのは非効率だった
* JSON-RPCの受信、method dispatch、セッション管理、CORS、エラー応答などをベース化する必要があった
* Portainer観測用MCPだけでなく、今後別用途のMCPにも横展開したい

#### 解決策

* `BaseMcpServlet` を作成
* JSON-RPC共通処理を基底クラスに集約
* 以下を基底化

  * `doPost`
  * `doGet`
  * `doOptions`
  * `doDelete`
  * CORS処理
  * `initialize`
  * `notifications/initialized`
  * `tools/list`
  * `tools/call`
  * `Mcp-Session-Id` 管理
  * JSON-RPC error response
* サブクラス側では以下をoverrideするだけでMCPを定義できる構成にした

  * `resInitialize`
  * `resToolsList`
  * `resToolsCall`

---

### 課題L: Portainer観測用MCPサーバーの完成

#### 発生した課題

* 公式Portainer MCPの `dockerProxy` は低レベルすぎ、LLMが迷う
* Docker監視用途では、LLMに見せるTool粒度と返却JSON設計が重要だった
* 監視・一覧・障害調査を同じToolに詰め込むと、小型モデルが判断しにくい

#### 解決策

* 具象 `McpServlet` で、Portainer観測用に以下3Toolを定義

  * `environments`
  * `containers`
  * `incidentDetail`
* `instructions` にTool利用順序を明記

  * まず `environments`
  * 次に `containers`
  * 必要な時だけ `incidentDetail`
* Tool descriptionにも用途と呼び出しタイミングを明記
* Portainer公式MCP / Portainer APIから得た情報を、LLM向けの3層構造に正規化する方針とした
* read-only観測専用とし、create / update / restart / stop / delete など破壊的操作は提供しない方針を明示

---

## 4. 成果物

### Portainer / MCP検証環境

* `[Host name masked]`

  * Portainer Server用VM
  * IP: `[IP address masked]`
* `portainer/portainer-ce:2.31.2`

  * Portainer Server本体
* `portainer/agent:2.31.2`

  * Dockerホスト観測用Agent
* `portainer-mcp v0.7.0`

  * 公式Portainer MCP Server
* `mcpo`

  * Portainer MCPをOpenAPI互換HTTPサーバーとして公開
* Open WebUI登録

  * OpenAPI Toolとして公式Portainer MCPを登録
  * MCP Streamable HTTPとして自作MCPサーバーを登録

### 自作MCP基盤

* `BaseMcpServlet`

  * Java Servletベースの自作MCP基底クラス
  * JSON-RPC 2.0の受信、method dispatch、CORS、セッション管理、`initialize`、`tools/list`、`tools/call`、JSON-RPC error応答を担当
  * サブクラスで `resInitialize`、`resToolsList`、`resToolsCall` をoverrideすることでMCPサーバーを量産できる構成

* `McpServlet`

  * Portainer / Docker観測用MCP Servlet
  * `BaseMcpServlet` を継承
  * Portainer観測用Toolを定義
  * `environments`、`containers`、`incidentDetail` の3層構造で、LLM向けに観測情報を提供する

### JSON-RPC Request系クラス

* `InitializeRequest`

  * `initialize` requestを表現するパーサークラス
  * `jsonrpc`、`id`、`method`、`params.protocolVersion`、`params.capabilities`、`params.clientInfo` を安全に取り出す
  * 後に `id` の型固定問題を修正し、JSON-RPCの仕様に合わせて扱えるよう調整

* `ToolsListRequest`

  * `tools/list` requestを表現するクラス
  * request idやmethodを保持し、response生成時のid対応に利用

* `ToolsCallRequest`

  * `tools/call` requestを表現するクラス
  * `params.name` と `params.arguments` を取得
  * Tool名によるdispatchや、Tool固有引数の取り出しに利用

### JSON-RPC Response系クラス

* `InitializeResponse`

  * `initialize` responseを表現するクラス
  * `protocolVersion`、`capabilities.tools.listChanged`、`serverInfo`、`instructions` を返却
  * `McpServlet` 側でPortainer観測用instructionsとserverInfoを上書き

* `ToolsListResponse`

  * `tools/list` responseを表現するクラス
  * `Tool`、`inputSchema`、`Property` を保持
  * Toolの `name`、`description`、`inputSchema.properties`、`required`、`additionalProperties` を定義

* `ToolsCallResponse`

  * `tools/call` responseを表現するクラス
  * `content[].type = text`
  * `content[].text` にLLM向けJSON文字列を格納
  * `isError` によりTool実行エラーを表現

### Portainer観測用DTO / Converter

* `Environments`

  * Portainer配下のDocker環境一覧をLLM向けに正規化したDTO
  * ホスト名、環境名、Dockerバージョン、OS、CPU、メモリ、コンテナ数、警告情報などを格納する想定

* `Containers`

  * 指定Docker環境配下のコンテナ一覧をLLM向けに正規化したDTO
  * コンテナ名、イメージ、状態、ステータス、ポート、mount、networkMode、compose情報、riskFlags、statsなどを格納する想定

* `IncidentDetail`

  * 指定コンテナの障害調査用詳細DTO
  * inspect由来の状態情報、restartCount、exitCode、OOMKilled、stats、直近ログなどを格納する想定

* `EnvironmentsOverviewConverter`

  * Portainer / Docker環境情報を取得し、`Environments` DTOへ変換する処理を担当

* `ContainersOverviewConverter`

  * 指定環境のコンテナ一覧を取得し、`Containers` DTOへ変換する処理を担当

* `IncidentDetailConverter`

  * 指定環境・指定コンテナの詳細情報を取得し、`IncidentDetail` DTOへ変換する処理を担当

### ドキュメント / 学習メモ

* `WikiJs @ JSON-RPC.md`

  * `initialize`
  * `notifications/initialized`
  * `tools/list`
  * `tools/call`
  * `MCP-Protocol-Version`
  * `Mcp-Session-Id`
  * PowerShellによる検証方法
  * Open WebUI / コンテナ内疎通確認方法
  * 上記をまとめた学習メモ

## 5. 結論

* Portainer Server、Portainer Agent、公式Portainer MCP、mcpo、Open WebUI、ローカルLLMを接続し、MCP連携を実体験できた
* `gemma4:e4b` と `gpt-oss:120b` の両方で、公式Portainer MCPの挙動を確認した
* 公式Portainer MCPは機能としては強力だが、`dockerProxy` のような低レベルToolをLLMに直接扱わせるには難易度が高いと分かった
* モデルサイズを上げても、Tool設計が低レベルすぎると、LLMはPortainer APIとDocker APIを混同するなど安定しないことを確認した
* MCPは「繋げば終わり」ではなく、LLMが迷わず使える粒度までToolを設計する必要があると理解した
* Java ServletでMCP Streamable HTTPの最小実装をフルスクラッチし、`initialize`、`tools/list`、`tools/call`、セッション管理まで実装できた
* その後、`BaseMcpServlet` としてMCP基盤を部品化し、今後のMCPサーバー量産に耐える土台を構築できた
* Portainer観測用MCPでは、`environments`、`containers`、`incidentDetail` の3層Tool設計により、LLMが段階的に必要情報へ到達できる構造を実現した
* 今回の最大の収穫は、**MCPを利用する側の体験だけでなく、MCPサーバーを自作し、LLM向けTool設計まで含めて制御できる基礎能力を獲得したこと**
* 次フェーズは、`BaseMcpServlet` の整理、各DTO/Converterの読み込み、Portainer実データ変換ロジックの精査、Tool返却JSONの最適化である