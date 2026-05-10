# Work079: SwitchBot制御用MCPサーバー2号機構築・Codex伴走型開発プロセス検証

Tag: [[MCP]], [[SwitchBot]], [[Java]], [[Codex]], [[AI伴走開発]], [[LLM Tool Use]], [[SceneService]], [[VibeCoding]], [[責務分離]], [[SafetyDesign]], [[AI監督責任開発]]

## 1. 目的

- 自作MCPフレームワーク上に、SwitchBotシーン制御用MCPサーバー2号機を構築する
- 既存資産である `SwitchBotApis.java` と、自作MCP基盤を接続し、SwitchBotのシーン一覧取得・シーン実行をMCP Toolとして公開する
- SwitchBotの「デバイス直接操作」ではなく「Scene実行」に限定し、LLMによる物理世界操作の責任境界を安全側に寄せる
- IntelliJ IDEA上のCodexを使い、AIに実装させる開発プロセスを実地検証する
- 「分析 → Go / NoGo判断 → 製造」という2段階のAI伴走型開発プロセスを運用し、品質・認知負荷・責任分界を確認する
- 単なる低コストなコード生成ではなく、設計判断・仕様策定・監督責任を人間が握るAI開発スタイルを検証する
- 今回の成果を、ソースコードそのものではなく「AI伴走型開発で得た知見」として記録する

## 2. システム環境

- **Development Machine:** 自宅メイン開発機
- **IDE:** IntelliJ IDEA
- **AI Coding Assistant:** Codex
- **Design / Review Assistant:** ChatGPT
- **Language:** Java 21
- **Servlet Runtime:** Tomcat
- **MCP Base Class:** `BaseMcpServlet`
- **SwitchBot API Client:** `SwitchBotApisV2`
- **Target Project:** `switchbot-ctrl-mcp`
- **Common Library:** `MyCommonLibV2`
- **SwitchBot Control Target:** SwitchBot Cloud API v1.1
- **MCP Endpoint:** `/mcp`
- **MCP Protocol Version:** `2025-06-18`
- **MCP Server Version:** `0.1.0`
- **MCP Tools:**
  - `scenes`
  - `executeScene`
- **Test Method:** 自作PowerShell MCPテストスクリプト、IntelliJデバッグ実行、実機SwitchBotシーン実行
- **Physical Target:** 個人管理下の照明シーンなど、SwitchBotアプリに登録済みのシーン
- **Related Previous Work:** Portainer Ops MCP Java Server / 自作MCP基盤

## 3. 作業ログ・解決プロセス

### 課題A: 「バイブコーディング」の目的を誤解していた

#### 発生した課題

当初、Codexでソースコードを書かせる行為を「バイブコーディング」と捉えていた。

乱暴に言えば、バイブコーディングとは「ノリと雰囲気でAIと一緒にコードを書く開発スタイル」であり、実質的には「低コストでソースコードが完成する」ものだと認識していた。

しかし、今回の題材は自分で実装すれば半日程度で終わる見込みだった。

理由は以下である。

- MCPサーバーのベースクラスはすでに自作済み
- SwitchBot API呼び出しクラスも既存資産として存在
- どちらも別プロダクトとして個人環境で継続利用していた
- 実装すべき構造も頭の中にあった
- つまり、単純に「完成させる」だけなら自力実装の方が圧倒的に低コストだった

#### 解決策

今回の目的を「低コストなコード生成」ではなく、以下に再定義した。

```text
Codexを使って、SwitchBot制御用MCPサーバーを安全に作り上げる
```

そのため、意図的に以下の運用を徹底した。

```text
1. ChatGPTと仕様・方針を整理する
2. Codexに分析させる
3. 人間がGo / NoGo判断を行う
4. Goの場合のみCodexに製造させる
5. 人間がコード・動作・責務境界を確認する
6. 必要なら前工程へ戻す
```

この結果、初期開発コストは大きく増えたが、設計品質・実装品質・判断ログ・中長期保守性を得られた。

---

### 課題B: 最初の `SwitchBotApis.java` リファクタリング分析でスコープが広がりすぎた

#### 発生した課題

最初にCodexへ `SwitchBotApis.java` の分析を依頼した際、MCPサーバー全体の背景を広く伝えた。

その結果、Codexは以下まで深く考慮した。

* MCPから呼ばれること
* LLMに物理世界を操作させる安全性
* allowlist
* dry-run
* cooldown
* 監査ログ
* MCP用DTO
* シーン公開ポリシー

Codexの分析内容自体は妥当だった。

しかし、`SwitchBotApisV2` という低レイヤーAPIクライアントの改修に対しては、背景情報が過剰だった。

このまま製造に進むと、APIクライアントにMCP固有の責務が混入する可能性があった。

#### 解決策

ここでNoGoを選択した。

理由は、今回の改修対象である `SwitchBotApisV2` に求める責務は以下に限定すべきだったためである。

```text
SwitchBot Cloud API v1.1へ署名付きHTTPリクエストを送る
デバイス一覧を取得する
シーン一覧を取得する
指定されたsceneIdのシーンを実行する
HTTPステータスとレスポンス本文を返す
通信不能・署名失敗などを専用例外で通知する
```

再度、低レイヤーAPIクライアントに限定した分析プロンプトを作成し、Codexに再分析させた。

この再分析では以下が明確になった。

* `SwitchBotApisV2` はMCPの存在を背景程度に留める
* token / secret は外側から注入する
* GET / POST の共通化を行う
* HTTP 4xx / 5xx は例外ではなく結果型で返す
* 通信不能など、HTTPレスポンスが返らない失敗は専用例外とする
* MCP用DTOや安全制御は別レイヤーに置く

この判断により、低レイヤーAPIクライアントの責務を明確に保てた。

---

### 課題C: 既存互換性を維持すべきか判断が必要だった

#### 発生した課題

Codexの分析では、既存 `SwitchBotApis.java` の外部仕様維持も選択肢として提示された。

しかし、今回は個人開発であり、既存サービスはすべて自分の管理下にある。

業務案件であれば後方互換性維持は重要だが、今回のV2化では、互換性維持によるコストの方が大きいと判断した。

#### 解決策

後方互換は維持しない方針とした。

* 既存処理は当面 `SwitchBotApis.java` を使う
* 新規処理は `SwitchBotApisV2` を使う
* 既存処理を移行する場合は、呼び出し元をV2仕様へ寄せる
* 個人開発であるため、破壊的変更の影響範囲を自分で吸収できる

この判断により、`SwitchBotApisV2` は以下のように整理できた。

* token / secret のべた書き廃止
* `SwitchBotApiResult` によるHTTPステータス・body・success返却
* `SwitchBotApiException` による通信不能系の通知
* `executeScene` という明確なメソッド名
* raw JSONを保持し、DTO変換は別レイヤーで実施

---

### 課題D: SwitchBot公式のMCP内蔵CLIを後から発見した

#### 発生した課題

仕様確認時点で、SwitchBot公式がMCP内蔵CLIを提供していることを発見した。

今回の目的が「最短でSwitchBotをMCP化すること」だけであれば、公式CLIを使う選択肢もあった。

そのため、自作MCPサーバーを作る意義を再確認する必要があった。

#### 解決策

今回の目的は「SwitchBotをMCP化する最短経路」ではなく、以下であると整理した。

```text
SwitchBot制御用MCPサーバーを、Codexを使って安全に作り上げる
```

また、公式CLIと今回の自作MCPサーバーでは目的が異なると判断した。

```text
公式CLI:
  多機能なSwitchBot操作入口

今回の自作MCP:
  自作MCPフレームワーク上で、Scene実行に限定した安全な制御層
  Codex伴走型開発プロセスの検証対象
  Portainer MCP 1号機と同じ構造で保守可能な2号機
```

そのため、公式CLIの存在は今回の学習価値を下げるものではないと判断した。

---

### 課題E: SwitchBot APIレスポンスをそのままMCPへ渡すと危険だった

#### 発生した課題

SwitchBot API の `getScenes()` は、SwitchBotアプリ上の全シーンを返す。

しかし、MCPサーバーでこれをそのままLLMへ渡すと、以下の問題が発生する。

* SwitchBotアプリに追加しただけのシーンが自動的にLLMへ露出する
* LLMに触らせたくないシーンまで候補に入る
* sceneIdをLLMに直接扱わせることになる
* 人間側の公開許可プロセスが存在しない

これは、LLMによる物理世界操作として危険だった。

#### 解決策

`SceneCatalog` を導入し、公開許可されたシーンだけをMCPに出す設計にした。

```text
SwitchBot API getScenes()
+
sceneCatalog.json
↓
sceneIdでinner join
↓
SceneSummary
```

方針は以下とした。

* `sceneCatalog.json` に登録されたシーンだけを公開対象とする
* join key は `sceneId`
* SwitchBot側に存在しないCatalog定義は落とす
* Catalogに存在しないSwitchBotシーンは落とす
* `executable=false` は公開一覧に出さない
* SwitchBot側でsceneNameが変わった場合は、API側の実名を優先する
* `note` など管理用情報はLLMに渡さない

これにより、SwitchBotアプリ側でシーンを追加しても、MCPには自動公開されない安全設計として確認できた。

---

### 課題F: Converterに責務を混ぜ込みすぎる危険があった

#### 発生した課題

当初、以下の機能を `SwitchBotScenesConverter` に混ぜ込む案が出た。

* 公開用シーン定義
* 危険シーンの扱い
* シーン一覧キャッシュ

しかし、Converterにこれらを混ぜると、変換層が以下の責務まで持ってしまう。

* ポリシー管理
* キャッシュ管理
* 実行許可判断
* 実行制御

これは、Portainer MCPで構築したDTO / Converter設計と比べても責務が崩れる可能性があった。

#### 解決策

以下の3層に分離した。

```text
SceneCatalog:
  公開用シーン定義、alias、危険度、cooldown、管理用noteを持つ

SwitchBotScenesConverter:
  SwitchBot API raw JSON + SceneCatalog を突合し、SceneSummaryへ変換する

SceneService:
  キャッシュ、API呼び出し、実行制御を持つ
```

この分離により、各クラスの責務が明確になった。

* `SceneCatalog` は設定を読む
* `Converter` は変換する
* `SceneService` は運用単位にまとめる
* `McpServlet` はToolとして公開する

---

### 課題G: `SceneSummary` に何を含めるべきか整理が必要だった

#### 発生した課題

`SceneCatalog` には人間が管理するための情報を含めたい。

例:

* note
* 将来の dateModified
* owner
* 管理用コメント

一方、LLM / MCPへ渡すDTOには、管理用情報を含めるべきではない。

#### 解決策

`SceneCatalogEntry` と `SceneSummary` を分離した。

```text
SceneCatalogEntry:
  人間管理用定義
  noteを含む
  cooldownを含む

SceneSummary:
  LLM / MCP公開用DTO
  noteを含まない
  sceneKey / sceneId / sceneName / publicName / category / action / riskLevel / executable / aliases / cooldown を含む
```

`note` は管理用情報として `SceneSummary` から除外した。

一方、`cooldown` は管理メモではなく実行制御に必要な正式な運用メタ情報と判断し、`SceneSummary` にも含めた。

---

### 課題H: キャッシュ対象を raw JSON にするか、`SceneSummary` にするか判断が必要だった

#### 発生した課題

当初は `SwitchBotApisV2.getScenes()` の戻り、つまり `SwitchBotApiResult` または raw JSON を24時間キャッシュする案があった。

しかし、MCP Toolや上位サービスが最終的に必要とするのはraw JSONではなく `SceneSummary` である。

raw JSONをキャッシュすると、以下の問題がある。

* 上位が再度変換を意識する
* 管理用情報やAPI由来の生データが残る
* SceneServiceの利用者にとって抽象度が低い

#### 解決策

キャッシュ対象を `List<SceneSummary>` に変更した。

SceneServiceの処理は以下とした。

```text
listScenes()
  ↓
キャッシュ有効なら List<SceneSummary> を返す
  ↓
期限切れなら SwitchBotApisV2.getScenes()
  ↓
SceneCatalog.loadEntries()
  ↓
SwitchBotScenesConverter.convert()
  ↓
List<SceneSummary> をキャッシュして返す
```

キャッシュTTLはVer.1では24時間固定とした。

理由は以下である。

* SwitchBot側でシーンを追加しても、sceneCatalog.jsonを更新しない限りMCP公開されない
* 即時反映の必要性が低い
* TTL外部設定化は今回の本質ではない
* Ver.1ではシンプルさを優先する

ただし、手動更新用の escape hatch として `refreshScenes()` を実装した。

---

### 課題I: `refreshScenes()` をTool公開すべきか判断が必要だった

#### 発生した課題

`SceneService` には保守用として `refreshScenes()` を実装した。

しかし、これをMCP Toolとして公開すると、LLMにとって選択肢が増える。

今回のMCPは、小型モデルでも扱える可能性を重視していたため、Tool数を増やすことは望ましくなかった。

#### 解決策

`refreshScenes()` はTool公開しない方針とした。

公開Toolは以下の2つに限定した。

```text
scenes
executeScene
```

これにより、LLMに求めるTool使用は非常に単純になった。

```text
1. scenes で使える sceneKey を確認する
2. executeScene に sceneKey を渡して実行する
```

この構成は、Gemma4:e4bのような小型モデルでも扱いやすい可能性がある。

---

### 課題J: シーン実行時に sceneId を直接受けるべきではなかった

#### 発生した課題

SwitchBot APIの実行には `sceneId` が必要である。

しかし、MCP Toolとして `sceneId` を直接受け取る設計にすると、以下の問題がある。

* LLMが生のsceneIdを扱う
* Catalogを経由しない実行余地が生まれる
* sceneIdが漏れた場合、LLMが想定外のシーンを指定できる
* 人間が管理する公開名・alias・riskLevelなどの意味づけが使われない

#### 解決策

MCP Toolの実行引数は `sceneKey` に限定した。

```text
executeScene(sceneKey)
```

実行時は以下の流れとした。

```text
sceneKey null / blank チェック
↓
sceneKey trim
↓
listScenes()
↓
SceneSummary一覧からsceneKey完全一致で検索
↓
SceneSummary.sceneIdで SwitchBotApisV2.executeScene()
```

重要なのは、`sceneCatalog.json` 単体から `sceneKey → sceneId` 変換しないことである。

必ず `listScenes()` の結果から解決することで、以下を保証した。

```text
SwitchBot APIに存在する
かつ
sceneCatalog.jsonに存在する
かつ
executable=true
```

---

### 課題K: `cooldown` と `dry-run` を失念していた

#### 発生した課題

SceneServiceの初期設計では、以下が抜けていた。

* cooldown
* dry-run

どちらも、LLMから物理世界を操作する上で重要な安全制御である。

この漏れに気づいたのは、Codexの分析結果を読んでいる段階だった。

もし分析フェーズを挟まずに製造へ進んでいれば、後から手戻りが発生していた。

#### 解決策

ここでもNoGoを選択し、SceneServiceの分析をやり直した。

追加仕様として以下を確定した。

```text
cooldown:
  同じsceneKeyの連続実行を抑制する
  sceneCatalog.jsonにcooldown秒数を設定
  未設定/nullの場合は10秒
  0秒は明示的な連続実行許可
  負数・非数値はエラー

dry-run:
  SceneService全体のモードとして持つ
  dryRun=trueの場合はexecuteSceneを呼ばない
```

`SceneExecutionResult` には以下を追加した。

```text
executed:
  実際にSwitchBot API executeSceneを呼んだか

dryRun:
  dry-runにより実行しなかったか
```

実行順序は以下にした。

```text
sceneKey検証
↓
listScenes()
↓
対象SceneSummary解決
↓
cooldown判定
↓
dryRun判定
↓
SwitchBot API実実行
```

dry-run時もcooldown中ならcooldown結果を返す方針とした。

---

### 課題L: cooldownの記録タイミングを整理する必要があった

#### 発生した課題

cooldownを実装するにあたり、いつ `lastExecutedAt` を更新するか整理が必要だった。

考慮すべきケースは以下だった。

* dry-run時
* cooldown拒否時
* SwitchBot API実行成功時
* SwitchBot API HTTP 4xx / 5xx時

#### 解決策

以下の方針とした。

```text
dry-run時:
  物理実行していないため lastExecutedAt を更新しない

cooldown拒否時:
  拒否しただけなので lastExecutedAt を更新しない
  更新するとcooldownが延長され続ける可能性がある

SwitchBot APIを実際に呼んだ時:
  HTTP成功/失敗に関わらず lastExecutedAt を更新する
```

理由は、HTTP 4xx / 5xxであっても実行リクエスト自体は送っているため、短時間連打は防ぐべきだからである。

---

### 課題M: 並列実行時の厳密な二重実行防止をどこまで行うか判断が必要だった

#### 発生した課題

SceneServiceはServletから呼ばれるため、複数リクエストが同時に来る可能性がある。

cooldownを厳密に実装するなら、同一sceneKeyの同時実行競合を完全に防ぐ排他制御が必要になる。

しかし、それは実装を複雑化させる。

#### 解決策

Ver.1では、同一sceneKeyの並列二重実行を厳密には防がない方針とした。

理由は以下である。

* 個人利用の自宅MCPサーバーである
* 複雑な排他制御は今回の学習テーマから外れる
* 「Codexを使った安全な段階的開発」が本質である
* 過剰な堅牢化より、理解しやすさ・保守しやすさを優先する

ただし、以下は保護した。

* `cachedScenes`
* `cachedAt`
* `lastExecutedAtBySceneKey`

この読み書きはlockで保護した。

---

### 課題N: `McpServlet` を美しく作り直すのではなく、1号機と同じ構造にする必要があった

#### 発生した課題

MCPサーバー1号機である Portainer Ops MCP の `McpServlet` には、すでに自分が運用・保守しやすい形が存在していた。

一方で、Codexにそのまま実装させると、より美しい構造や抽象化を提案する可能性があった。

しかし今回求めたのは、ソースコード上の美しさではなく、以下である。

```text
未来の自分が迷わず保守できること
1号機と同じ感覚で読めること
```

#### 解決策

1号機McpServletを参照用として以下に配置した。

```text
reference/SampleMcpServlet.java.txt
```

Codexには以下を明示した。

* 参照用ファイルであり編集禁止
* コンパイル対象に移動しない
* 構造・メソッド配置・helper配置を参考にする
* 過度な共通化をしない
* 1号機と同じ運用感を優先する

その結果、SwitchBot版 `McpServlet` も以下の構造になった。

```text
SERVER_NAME / SERVER_TITLE / SERVER_VERSION
TOOL_* 定数
SceneServiceフィールド
init()
resInitialize()
resToolsList()
addScenesTool()
addExecuteSceneTool()
resToolsCall()
scenesToolResponse()
executeSceneToolResponse()
successResponse()
errorResponse()
request helper
logging helper
```

これにより、1号機と2号機の保守感が揃った。

---

### 課題O: `SceneService` をtools/callごとにnewするとキャッシュとcooldownが壊れる

#### 発生した課題

動作確認用の `TestServlet` では、リクエストごとに `SceneService` を生成する形でも動作確認はできた。

しかし、本番のMCP Servletで同じことをすると、以下が成立しない。

* 24時間キャッシュ
* sceneKeyごとのcooldown履歴
* dry-runモードの一貫性

#### 解決策

MCP Servletでは `SceneService` をフィールドとして保持し、`init()` で生成する方針とした。

```text
Servlet起動
↓
AppConfig.load()
↓
SwitchBotApisV2生成
↓
SceneService生成
↓
以後tools/callから同じSceneServiceを利用
```

これにより、SceneServiceの状態がリクエスト間で維持される。

---

### 課題P: `TestServlet.java` をGitに含めるか判断が必要だった

#### 発生した課題

`TestServlet.java` は、ソースコードとしては壊滅的に雑であり、公開用プロジェクトには含めるべきではない。

一方で、自宅開発環境では、未来の自分が動作確認・再検証を行うために非常に有用である。

#### 解決策

自宅開発用プロジェクトには `TestServlet.java` をGit管理対象として残す方針とした。

理由は以下である。

* 未来の自分が最も迷わず再検証できる
* 自宅用開発環境では可読性より再現性を優先する
* GitHub公開用には別プロジェクトを作成し、必要なクラスだけ移植する
* 公開用プロジェクトには `TestServlet.java` を含めない

この判断により、ローカル運用効率と公開品質を分離した。

---

### 課題Q: MCP Toolを最小限に絞る必要があった

#### 発生した課題

SceneServiceには `refreshScenes()` が存在する。

しかし、これをToolとして公開すると、LLMにとって選択肢が増え、特に小型モデルでは迷う可能性がある。

今回のMCPサーバーは、Gemma4:e4bなどの小型モデルでも使える可能性を意識していた。

#### 解決策

公開Toolを以下2つに限定した。

```text
scenes
executeScene
```

`refreshScenes()` は保守用として内部に残し、Tool公開しない方針とした。

Tool使用フローは極めて単純になった。

```text
scenes:
  利用可能なsceneKey一覧を返す

executeScene:
  sceneKeyを受け取り、該当シーンを実行する
```

---

### 課題R: 実世界への介入を伴うため、dry-run検証が重要だった

#### 発生した課題

SwitchBot MCPは、LLM Tool経由で個人管理下の照明シーンなどを操作する。

そのため、疎通確認時にいきなり実世界へ介入するのは危険である。

#### 解決策

SceneServiceにdry-runモードを持たせた。

dry-run時は以下の挙動とした。

```text
SceneSummary解決までは行う
実際の物理操作は行わない
SceneExecutionResult.success = true
SceneExecutionResult.executed = false
SceneExecutionResult.dryRun = true
```

これにより、実行対象の解決・MCP応答・戻り値整形を確認しつつ、実世界への介入を止められる。

最終的にdry-run offでも確認し、個人管理下の照明シーンで動作を確認した。

---

### 課題S: MCPサーバーとしての疎通確認が必要だった

#### 発生した課題

Javaコード単体では成立しても、MCPサーバーとして以下が通る必要があった。

* initialize
* notifications/initialized
* tools/list
* tools/call scenes
* tools/call executeScene
* session delete

#### 解決策

自作PowerShell MCPテストスクリプトで以下を確認した。

```text
initialize
notifications/initialized
tools/call scenes
tools/call executeScene
doDelete
```

`scenes` の戻りでは、MCP公開許可済みの `SceneSummary` 一覧が返った。

`executeScene` の戻りでは、以下が返った。

```json
{
  "sceneKey": "sample_light_scene",
  "sceneId": "[XXXXXXXX masked]",
  "sceneName": "照明シーン（サンプル）",
  "success": true,
  "httpStatusCode": 200,
  "message": "Scene executed.",
  "executed": true,
  "dryRun": false
}
```

個人管理下の照明シーンで動作を確認し、MCPサーバー2号機としての成立を確認した。

---

### 課題T: 認知負荷が非常に高かった

#### 発生した課題

今回の作業は約2日かかった。

単純なコーディングなら半日で終わる見込みだったが、今回は以下をすべて行った。

* ChatGPTと仕様作成
* Codex用プロンプト作成
* Codex分析結果の精読
* Go / NoGo判断
* Codex製造結果のコードレビュー
* 実機動作確認
* 仕様漏れの再分析
* 既存クラス変更の差分確認

特に重かったのは、ChatGPTやCodexの出力をすべて読み、自分が同じ理解度で把握しようとしたことである。

#### 解決策

作業の途中で意図的に休憩を挟んだ。

* プロンプト発行後に休憩
* Codex実行中に離席
* Codex回答確認後に休憩
* 散歩や入浴を挟んで再読

これにより、高い認知負荷を分割して処理した。

また、この負荷そのものが学習であると整理した。

AIに任せるということは、人間が楽をすることだけではない。

人間が責任を持つ場合、AIの出力を理解し、判断し、受け入れる必要がある。

---

### 課題U: 「AIに作らせる責任」は誰が持つのか明確化する必要があった

#### 発生した課題

今回、人間はプロンプトやソースコードを直接書かなかった。

* プロンプトはChatGPTが作成
* ソースコードはCodexが作成
* 人間はGo / NoGo判断と受け入れ確認を実施

この場合、責任の所在を曖昧にすると「AIが作ったものだから」という逃げ道が生まれる。

#### 解決策

責任は人間が持つ、と明確化した。

今回の役割分担は以下である。

```text
ChatGPT:
  設計整理
  Codex指示プロンプト作成
  レビュー観点提示

Codex:
  分析
  実装
  差分生成
  テスト補助

人間:
  目的設定
  最終責任
  Go / NoGo判断
  仕様確定
  コード受け入れ
  実機確認
```

この構造により、コードを書いていないにもかかわらず、プロダクト責任は人間が持つことが明確になった。

また、判断ログがプロンプトとCodex回答として残るため、後から仕様判断を追跡できる。

---

## 4. 成果物

今回の主成果物は、ソースコードではなく、以下の知見である。

### 成果物A: AI伴走型・監督責任開発という開発スタイルの実体験

今回の開発は、一般的な「バイブコーディング」とは異なる体験だった。

```text
バイブコーディング:
  低コストで素早く作る
  試す
  UIやプロトタイプを広げる
  ノリと雰囲気で前進する

今回の開発:
  初期投資コストを上げる
  設計を明文化する
  AIの分析を人間が読む
  Go / NoGoを人間が判断する
  実装品質・保守性・判断ログを得る
```

今回の実感として、AIコーディングは必ずしも初期開発コストを下げるものではなかった。

むしろ、以下のように整理できる。

```text
AIを使って、初期投資コストを上げて、中長期的コストを下げた
```

---

### 成果物B: Go / NoGo判断の重要性

今回、特に重要だったNoGo判断は2つある。

```text
1. SwitchBotApisV2分析後のNoGo
   低レイヤーAPIクライアントにMCP文脈を入れすぎていたため、
   スコープを絞って再分析した。

2. SceneService分析後のNoGo
   cooldown / dry-run の仕様漏れに気づき、
   製造前に仕様を追加して再分析した。
```

どちらも、製造に進んでいても最終的には直せた可能性がある。

しかし、分析フェーズで止めたことで、手戻りを最小化し、責務境界を綺麗に保てた。

---

### 成果物C: プロンプトが設計書になるという知見

Codexへ渡したプロンプトは、単なる命令文ではなく、実質的な設計書だった。

そこには以下が含まれていた。

* 背景
* 今回やること
* 今回やらないこと
* 対象ファイル
* 責務境界
* 例外方針
* 戻り値方針
* Go / NoGo条件
* テスト観点
* 出力形式

これにより、ソースコードを読まなくても、プロンプトを追えばおおよその設計意図が分かる状態になった。

---

### 成果物D: 責務分離の実践知

最終構造は以下となった。

```text
SwitchBotApisV2:
  SwitchBot APIへの低レイヤー通信

SceneCatalog:
  resources JSONから公開用シーン定義を読む

SwitchBotScenesConverter:
  raw JSON + Catalog を突合して SceneSummaryへ変換

SceneService:
  キャッシュ、sceneKey解決、cooldown、dry-run、実行制御

McpServlet:
  MCP Toolとして scenes / executeScene を公開するだけ
```

この分離により、MCPサーブレットは非常に薄くなった。

---

### 成果物E: LLMに物理世界を操作させる安全設計

以下の安全設計を実装できた。

```text
デバイス直接操作を禁止
Scene実行に限定
sceneIdではなくsceneKeyを受ける
sceneCatalog.jsonに登録されたシーンだけ公開
SwitchBot APIとCatalogのinner join
executable=false除外
cooldown
dry-run
refreshScenes非公開
```

これにより、LLMが直接SwitchBotデバイスや未許可シーンを操作する経路を作らずに済んだ。

---

### 成果物F: dry-runという概念の獲得

今回、dry-runという言葉と設計概念を新たに学んだ。

dry-runは、物理実行を伴うTool Useでは特に重要である。

```text
実行対象解決までは行う
実際の物理操作は行わない
結果型として「実行されなかった」ことを返す
```

この概念は、今後のMCP / Tool Use / IoT制御でも再利用できる。

---

### 成果物G: cooldownをシーン単位の運用メタ情報として扱う設計

cooldownは単なる固定値ではなく、sceneCatalog.jsonに持たせた。

```json
{
  "sceneKey": "sample_light_scene",
  "cooldown": 1
}
```

これにより、シーンごとに連続実行制御を変えられる。

また、cooldownは `note` と違い、LLM公開DTOである `SceneSummary` に含めるべき正式な実行メタ情報だと整理できた。

---

### 成果物H: 小型LLMを意識したTool設計

今回のMCP Toolは2つに絞った。

```text
scenes
executeScene
```

これは、将来的にGemma4:e4bのような小型モデルに使わせることを考慮した設計である。

Tool数を増やさず、`refreshScenes` を非公開にしたことで、LLMの迷いを減らした。

---

### 成果物I: Codexによる既存クラス変更の観察

今回、Codexには新規クラス作成だけでなく、既存クラスの変更も実施させた。

対象は以下である。

* `SceneCatalogEntry`
* `SceneSummary`
* `SwitchBotScenesConverter`
* `SceneCatalog`

これにより、Codexが既存構造を壊さずに拡張できるかを観察できた。

結果として、差分は妥当であり、既存責務を大きく崩さずにcooldown対応が追加された。

---

### 成果物J: MCPサーバー2号機の完成

実装上の成果として、SwitchBot制御用MCPサーバー2号機が完成した。

公開Toolは以下。

```text
scenes:
  公開可能なSwitchBotシーン一覧を返す

executeScene:
  sceneKeyを受け取り、該当SwitchBotシーンを実行する
```

MCPテストスクリプトにより以下を確認した。

```text
initialize
notifications/initialized
tools/call scenes
tools/call executeScene
doDelete
```

実際に `executeScene` で個人管理下の照明シーンが動作することも確認した。

---

### 成果物K: AI開発における監督責任の言語化

今回の役割は、コードを書くことではなかった。

人間の役割は以下だった。

```text
目的を定める
責務境界を決める
AIの分析を読む
Go / NoGoを判断する
成果物を受け入れる
最終責任を持つ
```

AIに作らせたとしても、その成果物に責任を持つのは人間である。

今回、その責任構造を実地で確認できた。

---

## 5. 結論

今回の学習テーマである「CodexによるコーディングでSwitchBot制御用MCPサーバー2号機を作る」は完走した。

単純なコード完成という意味では、自分で実装した方が低コストだった。

しかし、今回得たものはコードそのものではなかった。

主な収穫は以下である。

```text
文書化された設計書
高品質な実装
責務分離の再確認
Go / NoGo判断の実地経験
AI出力を監督する認知負荷の実感
dry-run / cooldown という安全制御知見
物理世界操作MCPの安全設計
AIに作らせたものへ人間が責任を持つ構造
```

今回の最大の学びは、AIコーディングは必ずしも初期コストを下げるものではないということだった。

むしろ、設計・分析・レビュー・判断を丁寧に行う場合、初期コストは上がる。

しかし、その代わりに以下を得られる。

```text
中長期的な保守性
設計判断ログ
品質の高い初版
責任境界の明確化
後から追えるプロンプト資産
```

今回の作業は、一般的な「バイブコーディング」ではなく、以下に近い。

```text
AI伴走型・監督責任開発
```

または、

```text
設計主導型AIコーディング
```

この開発スタイルでは、AIはコードを書くが、責任は人間が持つ。

人間の役割は、コードを書くことではなく、以下になる。

```text
目的を定める
責務境界を決める
AIの分析を読む
Go / NoGoを判断する
成果物を受け入れる
最終責任を持つ
```

今回、この形を実地で経験できたことが最大の成果である。

次回以降は、完成したSwitchBot MCPサーバーをローカルLLMに接続し、以下を検証する。

```text
LLMが scenes → executeScene の順序を守れるか
sceneKeyを正しく選べるか
自然言語から適切なsceneKeyへ誘導できるか
小型モデルでも安定してTool Useできるか
誤実行を防げるか
```

これはMCPサーバー実装とは別テーマであり、次の検証フェーズとして扱う。