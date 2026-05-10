# SwitchBotApis.java 現状分析レポート
## 1. 現在の機能整理

対象ファイルは src\switchBot\SwitchBotApis.java です。Java内の直接参照を検索した範囲では、現時点で SwitchBotApis の利用箇所は同ファイル内のみでした。SwitchBotApis は、SwitchBot Cloud API v1.1 を呼び出す最小構成のAPIクライアントです。主な公開メソッドは以下です。
- getDevices()
  - https://api.switch-bot.com/v1.1/devices にGETします。
  - 成功時はレスポンス本文の String を返します。
  - 例外時は空文字 "" を返します。
- getScenes()
  - https://api.switch-bot.com/v1.1/scenes にGETします。
  - 成功時はレスポンス本文の String を返します。
  - 例外時は空文字 "" を返します。
- execScene(String sceneId)
  - https://api.switch-bot.com/v1.1/scenes/{sceneId}/execute にPOSTします。
  - BodyなしのPOSTです。
  - 成功時はレスポンス本文の String を返します。
  - 例外時は空文字 "" を返します。
- doGet(String uri)
  - SwitchBot API v1.1用の署名ヘッダーを作成し、GETリクエストを送信します。
  - 戻り値は HttpResponse です。
- doPost(String uri)
  - doGet とほぼ同じ認証処理を行い、POSTリクエストを送信します。
  - Bodyは noBody() です。
  - ステータスコードとレスポンス本文を System.out.println で出力しています。
  - 戻り値は HttpResponse です。
- Apis 内部クラス
  - APIパスを返すだけの静的内部クラスです。
  - devices() → /v1.1/devices
  - scenes() → /v1.1/scenes
  - scenesExec(String sceneId) → /v1.1/scenes/{sceneId}/execute
現状は「デバイス一覧取得」「シーン一覧取得」「シーン実行」に絞られた薄いAPIラッパーです。

## 2. 認証処理の確認
SwitchBot API v1.1 の署名認証は、doGet と doPost の両方に重複実装されています。確認できる処理は以下です。
- token
  - クラスフィールドとしてソースコードに固定値が直接書かれています。
  - HTTPヘッダー Authorization にそのまま設定されています。
- secret
  - クラスフィールドとしてソースコードに固定値が直接書かれています。
  - HmacSHA256 の秘密鍵として使用されています。
- nonce
  - UUID.randomUUID().toString() で毎回生成されています。
- timestamp
  - Instant.now().toEpochMilli() を文字列化しています。
  - HTTPヘッダー t に設定されています。
- 署名対象文字列
  - token + timestamp + nonce です。
- HMAC
  - アルゴリズムは HmacSHA256 です。
  - SecretKeySpec に secret のUTF-8バイト列を渡しています。
- Base64署名
  - HMAC結果をBase64エンコードし、sign ヘッダーに設定しています。
- HTTPヘッダー
  - Authorization
  - sign
  - nonce
  - t
コード上確認できる認証方式の骨格は、SwitchBot API v1.1 の一般的な署名方式と整合しているように見えます。ただし、同じ処理が doGet と doPost に重複しているため、後続の保守では共通化対象です。

## 3. 現状コードの問題点
最重要の問題は、token と secret がソースコードにべた書きされている点です。これはリポジトリ共有、ログ、バックアップ、Git履歴、LLM支援時の読み取りなどで漏えいリスクになります。製造前に現在の認証情報は再発行・ローテーションを検討すべきです。主な問題点は以下です。
- token / secret がソースコードに固定値で埋め込まれている。
- doGet と doPost に、nonce生成、timestamp生成、署名生成、ヘッダー設定が重複している。
- HttpClient.newBuilder().build() をリクエストごとに生成している。
- getDevices() / getScenes() / execScene() が例外時に空文字を返すため、通信失敗、認証失敗、URI不正、割り込みなどを呼び出し側が判別できない。
- InterruptedException を握りつぶしており、スレッドの interrupt 状態を復元していない。
- doPost() に System.out.println が残っており、MCP連携時に実行結果やエラー内容が標準出力へ流れる。
- HTTPステータスコードを公開API側で見ておらず、4xx/5xxでもレスポンス本文だけが返る。
- レスポンスは raw JSON string 中心で、MCP側で安全に扱うための構造化がまだない。
- Apis 内部クラスを毎回 new SwitchBotApis.Apis() しており、定数的なAPIパス管理としてはやや冗長。
- URI組み立て、HTTPメソッド指定、認証ヘッダー生成、送信、レスポンス処理が同じクラスに混在している。
- sceneId が未検証のままURIに連結されている。
コメントが文字化けして見える箇所があり、後続保守時に意図が読み取りづらい可能性がある。

## 4. リファクタリング方針
後続工程で SwitchBotApisV2.java を作るなら、まずは「APIクライアントとしての整理」に留めるのがよいです。MCP向けの意図解釈、許可シーン管理、DTO変換まで同じクラスに詰め込むと責務が膨らみます。認証情報の外出しは、基本方針としては コンストラクタ注入を中心にするのが適切です。

- 環境変数
  - 実運用では扱いやすい。
  - ただしAPIクライアントが直接 System.getenv() に依存するとテストしづらくなる。
  - MCPサーバー起動側、設定ローダー側で読み取り、クライアントへ渡す形がよい。
- コンストラクタ注入
  - 最も推奨。
  - テストしやすく、MCPサーバー側の設定管理とも分離しやすい。
  - token / secret / host / HttpClient などを外から渡せる。
- propertiesファイル
  - ローカル運用では便利。
  - ただし秘密情報を平文ファイルに置く場合、Git管理除外と権限管理が必要。
  - APIクライアント本体が直接読むより、設定クラスが読み込んで注入する方がよい。GET / POST の共通化は、内部に「HTTPメソッド、APIパス、Body有無」を受け取る共通送信処理を置き、署名ヘッダー生成をさらに小さな責務へ分ける方針がよいです。HttpClient はフィールドとして1回生成、またはコンストラクタ注入にします。Java 21の標準 java.net.http.HttpClient で十分です。タイムアウト設定もこの段階で入れるべきです。例外設計は、空文字返却をやめる方向が望ましいです。ただし互換性を壊さないため、V2では次のどちらかが現実的です。
- 低リスク案
  - 公開メソッドは当面 String を返す。
  - 失敗時は独自例外を投げる。
  - 呼び出し側が失敗理由を判断できる。
- 運用向け案
  - HTTPステータス、成功可否、レスポンス本文、例外メッセージを持つ結果オブジェクトを返す。
  - Portainer系クラスにある ResponseParam 的な設計に近い。
  - MCP側でエラー応答へ変換しやすい。戻り値設計は、V2初期では raw response を残すべきです。理由は、現在の利用形態を壊さず、SwitchBot APIレスポンスの実データを確認しながらDTO設計できるためです。そのうえで、別クラスにDTO変換層を置くのがよいです。責務分離の案は以下です。
- SwitchBotApisV2
  - SwitchBot APIとの通信だけを担当。
  - 認証、HTTP送信、レスポンス取得、HTTPエラー判断まで。
- SwitchBotSceneCatalog など
  - getScenes() の raw JSON を読み、シーン一覧を構造化する。
  - sceneId、sceneName、用途、許可状態などを扱う。
- SwitchBotSceneExecutor など
  - 許可済みsceneIdか確認してから execScene を呼ぶ。
  - dry-run、cooldown、監査ログなどを扱う。
- MCP Tool クラス
  - LLMからの入力を受け、意図をシーン候補に変換する。
  - SwitchBot APIクライアントを直接LLMに露出しない。

## 5. 互換性維持方針
現時点では SwitchBotApis の外部利用箇所は検索上見つかりませんでした。ただし現在は個人利用の範囲で継続利用している最小構成のAPIクライアントとして扱うため、公開メソッドの意味は維持すべきです。互換性維持の推奨は以下です。
- getDevices() は当面残す。
  - MCPでは直接デバイス操作しない方針でも、診断・確認用途として一覧取得は有用。
- getScenes() は当面 String を返すままにする。
  - 現行挙動と互換性を保てる。
  - DTO化は別メソッドまたは別クラスに分ける。
- execScene(String sceneId) も当面 String を返すままにする。
  - SwitchBot APIの実レスポンスをそのまま確認できる。
  - ただしV2では失敗時に空文字ではなく、例外または結果オブジェクトで失敗理由を返すべき。
- いきなりDTO化を主目的にしない。
  - V2の第一段階は、認証情報外出し、HTTP共通化、例外設計、ログ整理、sceneId検証までに留めるのが安全。
- MCP用DTO変換は別クラスに分ける。
  - APIクライアントはSwitchBot APIの都合だけを見る。
  - MCP層はLLMに見せる安全な道具名、説明、引数、許可シーンだけを見る。

## 6. MCP連携を見据えた安全設計
LLMから間接的に、SwitchBotアプリに登録済みの個人管理下のシーンを実行する想定では、APIクライアント単体の整理だけでは不十分です。最低限、MCP側と実行サービス側で安全策を入れる必要があります。考慮すべき安全策は以下です。
- sceneId のバリデーション
  - null、空文字、想定外文字、長すぎる値を拒否する。
  - URI連結前に検証する。
- 許可済みシーンだけを実行
  - getScenes() に存在するだけでは実行許可にしない。
  - 明示的な allowlist を持つ。
  - 例: 照明系ON、照明系OFF、空調系OFF、その他の許可済みシーンのように安全なシーンだけ登録。
- sceneIdではなく論理名で受ける
  - MCPツールの引数に生の sceneId を直接渡させるのは避ける。
  - LLMには lighting_scene_on のような内部論理名を選ばせ、サーバー側でsceneIdへ変換する方が安全。
- dry-run モード
  - 初期導入時は実行せず「どのシーンを実行する予定か」だけ返せるようにする。
  - テスト環境・本番環境で切り替え可能にする。
- cooldown
  - 同一シーンの連続実行を一定秒数抑制する。
  - 特に照明系・空調系などの操作では誤連打を防ぐ。
- 監査ログ
  - 実行日時、MCP session、tool名、入力、選択された論理シーン、[sceneId masked]、HTTPステータス、成功可否を記録する。
  - token、secret、署名、nonceは記録しない。
- ログのマスク
  - Authorization、sign、secret、レスポンス中の機微情報は出さない。
  - System.out.println ではなく既存の MyLogger またはSLF4J系に寄せる。
- MCP側の責任
  - LLM入力の解釈。
  - 許可シーンの選択。
  - dry-run、確認フロー、cooldown、監査ログ。
  - ユーザーに返す安全な説明文の生成。
- SwitchBot APIクライアント側の責任
  - 認証付きHTTPリクエスト。
  - APIパス管理。
  - HTTPステータスとレスポンス本文の取得。
  - 通信失敗やAPI失敗を呼び出し側へ正しく伝える。

## 7. リスクと注意点
一番大きなリスクは、現在の認証情報がソースコードに存在していることです。V2製造前に外出しするだけでなく、既に露出した可能性がある前提で再発行を検討してください。次に大きいリスクは、LLMに近い層へ execScene(sceneId) をそのまま公開することです。sceneIdを知っていれば任意シーンを実行できるため、MCPツールとしては raw sceneId 実行ツールを作らず、許可済み論理シーンだけを実行する設計にすべきです。また、現在は失敗時に空文字が返るため、MCPサーバー側で「失敗したのに成功っぽく見える」状態が起き得ます。照明系・空調系などのシーン実行では、失敗理由を明確に返せる設計が必要です。HTTP 4xx/5xxの扱いも注意点です。現状は例外ではなく通常レスポンスとして本文が返るため、呼び出し側がステータスコードを見ない限り失敗判定できません。

## 8. Go / NoGo 判断
判断: 条件付きGo
理由:
- 現行クラスの機能範囲は小さく、V2として整理する対象は明確。
- SwitchBot API v1.1の署名処理の骨格は既に実装されている。
- MCPではシーン実行に限定する方針が妥当で、デバイス直接操作を避ける設計判断は安全面で正しい。
- ただし、認証情報べた書き、例外握りつぶし、sceneId未検証、raw sceneId実行の危険があるため、このままMCPへ組み込むのは不可。
- V2製造は可能だが、先に認証情報管理と安全境界を決める必要がある。
製造前に確認すべきこと:
- 現在のSwitchBot token / secret を再発行するか。
- V2の認証情報は環境変数から設定ローダーが読み、コンストラクタ注入する方針でよいか。
- MCPで実行許可するシーン一覧をどこで管理するか。
- LLMに sceneId を直接渡させない方針でよいか。
- dry-run モードを初期実装に含めるか。
- 失敗時は例外方式にするか、結果オブジェクト方式にするか。
- V2で現行の String 戻り値メソッドを維持するか。
推奨する次の作業:
- SwitchBotApisV2 の外部仕様を先に確定する。
- 認証情報クラス、APIクライアント、MCP用シーン実行サービスの責務分離を決める。
- 許可済みシーン定義の形式を決める。
- その後、SwitchBotApisV2.java の製造に進む。

## 9. 推奨する次の作業
次工程では、いきなりMCPツールまで作らず、まず SwitchBotApisV2 の仕様を固めるのがよいです。優先順は以下です。
- 認証情報の注入方式を決める。
- getScenes() / execScene(String sceneId) の互換メソッドを残すか決める。
- 失敗時の扱いを、例外方式か結果オブジェクト方式か決める。
- raw JSONを返すAPIクライアント層と、MCP用DTO変換層を分ける。
- 許可済みシーン、dry-run、cooldown、監査ログの責務をMCP側サービスとして設計する。この条件を満たせば、SwitchBotApisV2.java の製造に進んで問題ありません。