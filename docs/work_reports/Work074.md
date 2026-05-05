# Work074: Portainer Ops MCP Java Server構築とDocker監視DTO設計

Tag: [[MCP]], [[Portainer]], [[Docker]], [[Java]], [[LLM Tool Use]], [[PortainerOpsMCP]], [[MultiAgent]], [[Troubleshooting]]

## 1. 目的

* 自作MCPフレームワーク上に、Portainer / Docker 環境を観測する `portainer-ops-mcp` を実装する
* 公式 `portainer-mcp-proxy` から取得できる Docker / Portainer 情報を把握する
* LLMが扱いやすい中継DTOとして、Docker監視情報を3層構造へ正規化する
* 監視用途とLLM分析用途の境界を整理する
* 小型ローカルLLMにMCPツールを使わせる際の制約・課題を確認する
* 今後の `gpt-oss:120b planner` / `gemma4:e4b executor` 構成に向けた設計知見を得る

## 2. システム環境

* **Node:** `[Host name masked]`
* **Portainer:** `portainer/portainer-ce:2.31.2`
* **Portainer URL:** `[internal Portainer URL masked]`
* **Portainer MCP Proxy:** `portainer-mcp-proxy`
* **MCPO / MCP Proxy Port:** `[internal Portainer URL masked]`
* **portainer-ops-mcp:** Tomcat 上の Java Servlet
* **portainer-ops-mcp URL:** `[internal Portainer URL masked]/portainer-ops-mcp-1.0/mcp`
* **MCP Protocol Version:** `[Date masked]`
* **Server Version:** `0.2.0`
* **Managed Environments:**

  * `local`
    * `environmentId: 3`
    * `type: [type masked]`
    * `OS: [OS masked]`
    * `Docker: 29.4.1`
  
  * `[Host name masked]`
    * `environmentId: 9`
    * `type: [type masked]`
    * `OS: [OS masked]`
    * `Docker: 24.0.5`
* **Primary Test Container:** `[container name masked]`
* **LLM Test Model:** `gemma4:e4b`
* **Future Planner Candidate:** `gpt-oss:120b`

## 3. 作業ログ・解決プロセス

### 課題A: 公式 `portainer-mcp-proxy` をフル権限で観測する

#### 発生した課題

* 初期構成では `-read-only` が付いており、公式 Portainer MCP Server が持つ全機能を把握しきれない
* `dockerProxy` の指定方法が分かりづらく、どのパスに何を指定すべきか不明確だった
* `getLocalStackFile` に `id: 0` を指定したため、存在しない stack ID として 404 が返った

#### 解決策

* 検証フェーズとして `-read-only` を外し、公式 Portainer MCP Server が返すツール一覧と戻りJSONを調査した
* `dockerProxy` は Portainer API ではなく Docker Engine API のパスを指定するものと整理した
* `dockerProxy` の基本形を以下のように整理した

```text
environmentId = Portainer上の環境ID
method        = GET / POST など
dockerAPIPath = /info, /containers/json, /containers/{id}/json など
queryParams   = Docker APIへ渡すクエリパラメータ
```

* `listEnvironments` により、以下の環境IDを確認した

```text
3 = local
9 = [Host name masked]
```

---

### 課題B: Docker監視に有用なAPI項目の選別

#### 発生した課題

* `/info`、`/containers/json`、`/containers/{id}/json`、`/stats` の戻りJSONは項目数が非常に多い
* 生JSONをそのままLLMへ渡すと、ローカル小型モデルには情報量が過剰になる
* 監視用途とLLM分析用途の双方に使いやすい項目選別が必要だった

#### 解決策

* Docker / Portainer の情報を以下の3層に分類した

```text
1. ホスト概要
   OS / Docker version / CPU / Memory / Warnings / Containers数

2. コンテナ概要
   Name / Image / State / Status / Health / Ports / NetworkMode / Mounts / Compose labels / stats / riskFlags

3. 異常時詳細
   inspect の RestartCount / ExitCode / OOMKilled / StartedAt / FinishedAt / stats / logs tail
```

* 通常監視では `environments` と `containers` を使い、障害調査時のみ `incidentDetail` を使う構成とした
* `containers` には CPU / メモリ / PID も含め、LLMが「どのコンテナが重いか」を比較できるようにした
* `incidentDetail` にはログを含め、LLMによる文脈分析に使える構造とした

---

### 課題C: ログ取得の挙動確認

#### 発生した課題

* Portainer MCP API 経由で取得するログが、対象コンテナ内で `docker logs` を実行しているのか、Docker Engine API を叩いているのか整理が必要だった
* ログが JSON ではなく、エスケープされたテキストとして返る場合があった
* LLMに渡す際に、deviceId などの識別情報をどう扱うべきか検討が必要だった

#### 解決策

* Portainer MCP API の `dockerProxy` は、Portainer / Agent 経由で Docker Engine API の `/containers/{id}/logs` を呼んでいると整理した
* 対象コンテナ内で `docker logs` コマンドを実行しているわけではなく、Docker daemon が logging driver からログを取得して返していると確認した
* `[container name masked]` のログでは、以下のような処理状況が取得できた

```text
RUN
http request
Status Code = 200
body
database connect
save_binary
execute
database commit & close
FINISH - NEXT RUN
```

* ログはルールベース監視よりも、LLMによる文脈分析と相性が良いと判断した
* ログ本文は将来的に以下の前処理対象とした

```text
deviceId / hubDeviceId のマスク
API URL のマスク
JSON文字列として二重エスケープされている場合のデコード
```

---

### 課題D: Portainer MCP API接続クラスとレスポンスパーサーの作成

#### 発生した課題

* 今後MCP用途だけでなく、ルールベース監視でもPortainer MCP APIを再利用したい
* Docker APIの戻りJSONはバージョン差・仕様差があるため、全項目を固定クラスへ展開するのは危険だった
* ただし、監視に使う主要項目は型付きで扱いたい

#### 解決策

* Portainer MCP API呼び出しを担う `PortainerMcpApiCore` を作成した
* 主要レスポンスごとに parser クラスを作成した
* 各 parser は、固定で使う項目をフィールド化し、それ以外は `rawNode` として保持する設計にした
* `stats` については、CPU使用率とメモリ使用率を Java 側で算出するメソッドを実装した

```text
cpuPercent
= (cpuDelta / systemDelta) * onlineCpus * 100

memoryUsagePercent
= memory_stats.usage / memory_stats.limit * 100
```

---

### 課題E: portainer-ops-mcp用の3層DTO設計

#### 発生した課題

* LLMにとって `environment`、`host`、`container` の概念が混ざりやすい
* Docker API由来の `Names` は `/[container name masked]` のように先頭 `/` を含むため、LLMに読みづらい
* `compose.project`、`compose.service`、`riskFlags` の取得元を整理する必要があった

#### 解決策

* MCP tool の戻りを以下の3種類に整理した

```text
environments
  ホスト一覧・ホスト概要

containers
  指定ホスト配下のコンテナ一覧・通常監視用

incidentDetail
  指定コンテナの障害調査用詳細
```

* `environmentName` に加えて `hostName` を持つ方針を整理した
* `containerName` は先頭 `/` を除去してLLMに渡す方針とした
* `compose.project` / `compose.service` は Docker Compose の labels から取得すると整理した

```text
com.docker.compose.project
com.docker.compose.service
```

* `riskFlags` は Docker APIから直接取得する項目ではなく、Java側で生成する派生項目とした

```text
PUBLIC_PORT_EXPOSED
HOST_NETWORK
DOCKER_SOCK_MOUNTED
HOST_ROOT_MOUNTED
CONTAINER_NOT_RUNNING
OOM_KILLED
RESTARTING
EXIT_CODE_NON_ZERO
```

---

### 課題F: `McpServlet` への3層構造組み込み

#### 発生した課題

* 既存の `McpServlet` には学習用・ダミー用の description が残っていた
* `incidentDetail` の description が空だった
* LLMにとって、どのツールをいつ使うべきかが十分に明確ではなかった
* 引数不足時の扱い、エラー時の戻り、ログ出力を整理する必要があった

#### 解決策

* `McpServlet` の tool を以下の3つに整理した

```text
environments
containers
incidentDetail
```

* `initializeResponse.result.instructions` に、ツールの使用順序を明示した

```text
Use 'environments' first
Use 'containers' with an environmentName
Use 'incidentDetail' only when investigating a specific container
```

* `containers` の description に、通常監視・一覧比較用であることを明記した
* `incidentDetail` の description に、ログ・再起動・OOM・ExitCode・障害調査時に使うツールであることを明記した
* `McpServlet` は以下の責務に集中するよう整理した

```text
tools/list のツール定義
tools/call のルーティング
引数検証
Converter呼び出し
ToolsCallResponse生成
```

---

### 課題G: MCP `protocolVersion` の誤解と修正

#### 発生した課題

* `BaseMcpServlet` のコンストラクタに `[Date masked]` を渡していたため、initialize response の `protocolVersion` が `[Date masked]` になっていた
* これはビルド日や自作サーバーの版数ではなく、MCPプロトコル仕様のバージョンを表す項目だった
* クライアント側が未知の protocol version と解釈し、`notifications/initialized` や後続処理が不安定になる可能性があった

#### 解決策

* `protocolVersion` は MCP仕様のバージョンとして扱うものと整理した
* `McpServlet` の `super(...)` には `[Date masked]` を渡す方針に修正した
* 自作サーバーのバージョンは `serverInfo.version` に格納することとした
* `SERVER_VERSION` は `0.2.0` とした

```text
protocolVersion
  MCPプロトコル仕様のバージョン

serverInfo.version
  自作MCPサーバーのアプリケーションバージョン
```

---

### 課題H: 小型LLMによるMCPツール利用の限界確認

#### 発生した課題

* `gemma4:e4b` に対して「MCPツールを使ってDockerコンテナ一覧を取得」と依頼したところ、`environments` と `containers` は呼べた
* しかし、実在しない `nginx-web-server`、`mysql-database`、`redis-cache`、`jenkins-ci` などを出力するハルシネーションが発生した
* `SwitchBot2MyOracle` の詳細・ログ取得では、`incidentDetail` を呼ばず、手元文脈のログを誤って解釈した
* 「MCPツールを使って」という指示だけでは、小型モデルに正しい tool selection を期待するには弱いことが分かった

#### 解決策

* 小型モデルには、ツール名・引数・禁止事項まで明示する必要があると整理した
* `incidentDetail` を呼ばせるプロンプト例を定義した

```text
MCPツール incidentDetail を必ず使ってください。
environmentName=[Host name masked]、containerName=[container name masked] です。
状態、restartCount、exitCode、oomKilled、CPU、メモリ、直近ログを取得して要約してください。
ツール結果にない内容は推測しないでください。
```

* 今後の構成として、`gpt-oss:120b` を planner / prompt normalizer として使い、`gemma4:e4b` に厳密化済みのツール使用指示を渡す案を整理した
* ジェムちゃんの `director / actor` 構成で培ったマルチエージェント設計が、MCPツール制御にも応用できると確認した

## 4. 成果物

### MCPサーバー基盤

* `BaseMcpServlet`

  * JSON-RPC / MCP の共通処理を担う基底Servlet
  * `initialize`
  * `notifications/initialized`
  * `tools/list`
  * `tools/call`
  * session管理
  * CORS処理
  * protocolVersion検証
  * JSON-RPC error生成

* `McpServlet`

  * Portainer Ops MCP 用の具象Servlet
  * `environments`
  * `containers`
  * `incidentDetail`
  * 上記3ツールを `tools/list` / `tools/call` として公開
  * LLM向け instructions / description を実装

* `InitializeRequest`

  * MCP initialize request のパーサー
  * `jsonrpc`
  * `id`
  * `method`
  * `params.protocolVersion`
  * `clientInfo`

* `InitializeResponse`

  * MCP initialize response のDTO
  * `protocolVersion`
  * `instructions`
  * `capabilities.tools.listChanged`
  * `serverInfo`

---

### Portainer MCP API 接続層

* `PortainerMcpApiCore`

  * 公式 `portainer-mcp-proxy` へHTTP POSTする中核クラス
  * `listEnvironments`
  * `dockerProxy_info`
  * `dockerProxy_containers`
  * `dockerProxy_container`
  * `dockerProxy_containerStats`
  * `dockerProxy_containerLogs`

* `PortainerApiCallException`

  * Portainer MCP API 呼び出し失敗時の独自例外
  * HTTP response code / response body を保持する `ResponseParam` を参照可能

---

### Portainer / Docker レスポンスパーサー

* `ListEnvironmentsResponse`

  * Portainer environment一覧のパーサー
  * `id`
  * `name`
  * `status`
  * `type`
  * 追加項目は `rawNode` 経由で取得

* `DockerProxy_infoResponse`

  * Docker `/info` のパーサー
  * コンテナ数、Docker version、OS、Kernel、CPU、Memory、Cgroup、Warnings を取得

* `DockerProxy_containersResponse`

  * Docker `/containers/json?all=true` のパーサー
  * コンテナ一覧、イメージ、状態、ポート、マウント、labels を取得

* `DockerProxy_containerResponse`

  * Docker `/containers/{id}/json` のパーサー
  * inspect相当の状態情報を取得
  * `RestartCount`
  * `ExitCode`
  * `OOMKilled`
  * `StartedAt`
  * `FinishedAt`
  * `Running`
  * `Restarting`

* `DockerProxy_containerStatsResponse`

  * Docker `/containers/{id}/stats?stream=false` のパーサー
  * CPU使用率、メモリ使用率、PID数を算出

---

### portainer-ops-mcp 正規化DTO / Converter

* `Environments`

  * MCP tool `environments` の戻りDTO
  * 登録済み Docker environment / host の概要一覧を表現

* `EnvironmentOverview`

  * 1ホスト分の概要DTO
  * environmentId、environmentName、hostName、OS、Docker version、CPU、Memory、warnings などを保持

* `Containers`

  * MCP tool `containers` の戻りDTO
  * 指定 environment 配下のコンテナ一覧を表現

* `ContainerOverview`

  * 1コンテナ分の概要DTO
  * containerId、containerName、image、state、status、networkMode、ports、mounts、compose、riskFlags、stateDetail、stats を保持

* `IncidentDetail`

  * MCP tool `incidentDetail` の戻りDTO
  * 障害調査用の詳細情報を表現
  * inspect由来の状態、stats、logs を保持

* `EnvironmentsOverviewConverter`

  * Portainer MCP API の environment / info を `Environments` へ変換

* `ContainersOverviewConverter`

  * Docker container list / inspect / stats を `Containers` へ変換

* `IncidentDetailConverter`

  * Docker inspect / stats / logs を `IncidentDetail` へ変換

---

### テスト・検証

* `PortainerMcpTester`

  * Portainer MCP API 呼び出しの手動検証用クラス
  * environment一覧、Docker info、container一覧、inspect、stats、logs の取得を確認

* PowerShell 検証コマンド

  * `initialize`
  * `tools/list`
  * `tools/call`
  * session id の取得
  * `Mcp-Session-Id` 付きリクエスト

---

## 5. 結論

* 自作MCPフレームワーク上に、Portainer / Docker 観測用の `portainer-ops-mcp` を構築できた
* 公式 `portainer-mcp-proxy` から取得できる情報を確認し、Docker監視に必要な項目を把握できた
* 生のDocker API JSONをそのままLLMへ渡すのではなく、LLMが扱いやすい3層構造へ正規化する設計が成立した
* `environments`、`containers`、`incidentDetail` の3ツール構成により、ホスト一覧・コンテナ一覧・障害調査詳細という自然な導線を作れた
* `protocolVersion` は自作サーバーの版数ではなく、MCP仕様バージョンであることを実地で学んだ
* `serverInfo.version` に自作MCPサーバーのアプリケーションバージョンを入れる整理ができた
* `gemma4:e4b` で MCP tool call が部分的に成立することを確認できた
* 一方で、小型モデルでは曖昧な自然文から正しい tool selection を安定して行うには制約が強いことも分かった
* 今後は、`gpt-oss:120b` を planner / prompt normalizer として使い、`gemma4:e4b` に厳密化済みのツール使用指示を渡す構成が有望である
* 今回の最大の収穫は、**MCPサーバー実装だけでなく、LLMにMCPを効果的に使わせるための制御設計フェーズに到達したこと**
* ジェムちゃん開発で培った director / actor 型のマルチエージェント設計が、MCPツールルーティングにも応用できる見通しが立った