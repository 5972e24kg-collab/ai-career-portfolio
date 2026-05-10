# SwitchBotApis.java 現状分析レポート
## 1. 現在の機能整理
対象は SwitchBotApis.java です。今回、コード変更・ファイル作成は行っていません。
- getDevices()
  - https://api.switch-bot.com/v1.1/devices に GET。
  - 成功時は HttpResponse.body() のみ返す。
  - 例外時は空文字 "" を返す。
- getScenes()
  - https://api.switch-bot.com/v1.1/scenes に GET。
  - 戻り値・例外処理は getDevices() と同じ。
- execScene(String sceneId)
  - https://api.switch-bot.com/v1.1/scenes/{sceneId}/execute に POST。
  - Body はなし。
  - 成功時はレスポンス本文のみ返す。
  - 例外時は空文字 "" を返す。
- doGet(String uri)
  - SwitchBot API v1.1 用の署名ヘッダーを生成し、GET リクエストを送信する。
  - HttpResponse を返すが、上位メソッドでは body しか利用していない。
- doPost(String uri)
  - doGet とほぼ同じ署名処理を行い、POST no body で送信する。
  - System.out.println で status/body を出力している。
- Apis 内部クラス
  - API パス文字列を返すだけの public static class。
  - scenesExec(String sceneId) は sceneId をそのままパスへ文字列連結している。

## 2. 認証処理の確認
認証処理は doGet と doPost の両方に重複して実装されています。
- token
  - クラス内の TOKEN フィールドに直接保持。
  - Authorization ヘッダーに設定。
- secret
  - クラス内の SECRET フィールドに直接保持。
  - HmacSHA256 の鍵として使用。
- nonce
  - UUID.randomUUID().toString() で毎回生成。
- timestamp
  - Instant.now().toEpochMilli() を文字列化。
  - t ヘッダーに設定。
- 署名対象データ
  - TOKEN + timestamp + nonce
- 署名方式
  - HmacSHA256
  - SECRET を UTF-8 bytes にして SecretKeySpec を生成。
  - HMAC 結果を Base64 エンコードして sign ヘッダーに設定。
- HTTP ヘッダー
  - Authorization
  - sign
  - nonce
  - t
署名処理の方向性自体は v1.1 認証の形に沿っていますが、実装上は重複・固定値・例外処理の弱さが目立ちます。

## 3. 現状コードの問題点
- token / secret がソースコードにべた書きされている
  - 実値はここでは再掲しませんが、19-20行 に固定値があります。
  - リポジトリ共有や漏えい時のリスクが高く、V2では廃止必須です。
- doGet と doPost の重複
  - nonce、timestamp、HMAC、Base64、ヘッダー設定がほぼ同一。
  - V2では署名生成・ヘッダー生成・送信処理を共通化すべきです。
- HttpClient を毎回生成している
  - 74行 と 95行 でリクエストごとに生成。
  - タイムアウト設定もありません。
- 例外時に空文字を返している
  - 通信失敗、URI不正、署名生成失敗、割り込みがすべて "" になる。
  - 呼び出し元は「空の正常レスポンス」と「失敗」を区別できません。
- InterruptedException の扱いが不適切
  - catch して空文字を返すだけで、割り込み状態を復元していません。
  - V2では Thread.currentThread().interrupt() を行ったうえで専用例外に包むか、明示的に呼び出し元へ伝えるべきです。
- System.out.println が残っている
  - doPost が HTTP status/body を標準出力へ出しています。
  - ライブラリ用途では副作用が強く、レスポンス本文に機密情報が含まれる可能性もあります。
- HTTPステータスコードの扱いが弱い
  - getDevices/getScenes/execScene は body だけ返すため、200/401/403/404/500 を呼び出し元が判別できません。
  - 低レイヤーAPIクライアントとしては status/body を保持して返すべきです。
- URI組み立てが文字列連結
  - host + path + sceneId の単純結合。
  - sceneId の URL エンコードや path segment としての安全性確認がありません。
- sceneId のバリデーションがない
  - null、空文字、空白、/ を含む値などがそのまま URI に入ります。
  - V2では最低限 null/blank を拒否し、必要なら path segment としてエンコードすべきです。
- クラス責務
  - 現状は「APIパス」「認証情報保持」「署名」「HTTP送信」「ログ出力」「例外握りつぶし」を1クラスに持っています。
  - 低レイヤーAPIクライアントとしての範囲内ではありますが、V2ではログ出力と設定読み込みは入れず、責務を絞るべきです。

## 4. SwitchBotApisV2.java の設計方針
- 認証情報の扱い
  - ソースコードべた書きは廃止。
  - 基本はコンストラクタ注入。
  - 環境変数や properties の読み込みは、この低レイヤークラスではなく外側の設定レイヤーで行い、V2には token / secret を渡す形がよいです。
- HttpClient の扱い
  - フィールドとして保持。
  - 標準コンストラクタでは内部生成してもよいが、テスト容易性のため HttpClient 注入用コンストラクタも用意する方針がよいです。
  - HttpClient の connect timeout と、HttpRequest の request timeout を設定可能にする。
- GET / POST の共通化
  - send(method, path, body) 相当の内部処理に集約。
  - 署名生成、認証ヘッダー生成、URI生成、HTTP送信を分離。
  - GET と POST no body はメソッド指定だけで分岐できる形にする。
- 例外設計
  - 空文字返却は廃止。
  - 通信失敗、URI生成失敗、署名生成失敗、割り込みは専用例外で呼び出し元へ伝える。
  - HTTP 401/403/404/500 などは「HTTPレスポンスが返っている」ため、原則として例外ではなく結果オブジェクトに status/body として返すのが低レイヤークライアントとして扱いやすいです。
- 戻り値設計
  - raw JSON string だけを返すのは避ける。
  - statusCode、body、必要なら headers、isSuccess を持つシンプルな結果型を返す。
  - Java 21 指定なので、結果型は record を使う選択肢があります。
  - デバイスやシーンの DTO 変換はこのクラスの責務外にする。
- APIパス管理
  - 3エンドポイント程度なら private static final 定数と private helper で十分。
  - enum 化は現時点では過剰です。
  - 現在の Apis 内部クラスは public にする必要が薄く、V2では private な定数/メソッドへ寄せる方針がよいです。

## 5. 後方互換を捨てる前提でのV2外部仕様案
推奨する public API は次の方向です。実装コードではなく仕様案です。
- getDevices
  - デバイス一覧 API を呼ぶ。
  - 戻り値は結果オブジェクト。
  - HTTP status/body を呼び出し元が確認できる。
- getScenes
  - シーン一覧 API を呼ぶ。
  - 戻り値は結果オブジェクト。
- executeScene
  - execScene より意図が明確なので、V2では executeScene(String sceneId) を推奨。
  - sceneId が null/blank の場合は IllegalArgumentException。
  - 通信失敗などは専用例外。
- doGet / doPost
  - V2の public API には出さない方がよいです。
  - 任意URI送信を公開すると、SwitchBot APIクライアントとしての責務境界がぼやけます。
- 例外方針
  - 引数不正: IllegalArgumentException
  - HTTPレスポンスあり: 結果オブジェクトで返す
  - HTTPレスポンスなしの失敗: 専用例外
  - 割り込み: interrupt flag を復元して専用例外、または明示的に InterruptedException を投げる設計にする

## 6. クラス責務の境界
入れてよいもの:
- SwitchBot API の host
- token / secret を使った署名生成
- nonce / timestamp 生成
- HTTPリクエスト送信
- HTTPステータスとレスポンス本文の保持
- 最低限の引数バリデーション
- 専用例外
- API呼び出し結果を表すシンプルな結果型

入れないもの:
- 呼び出し元システム固有の安全設計
- シーン allowlist
- シーン名 alias
- ユーザー発話解析
- dry-run
- cooldown
- 監査ログ
- LLM向け説明文生成
- MCP Tool 定義

## 7. リスクと注意点
- 既存コードに含まれていた token / secret は公開対象外とし、必要に応じて SwitchBot 側でローテーションすべきです。
- V2製造前に、SwitchBot Cloud API v1.1 の現行仕様で署名対象、必須ヘッダー、scene execute のHTTPメソッドを再確認するのが安全です。
- 低レイヤーAPIクライアントでは JSON の意味解釈をしすぎない方がよいです。
- HTTP non-2xx を例外にするか結果として返すかは最初に固定してください。今回の目的なら「結果として返す」が適しています。
- 標準出力ログはV2に持ち込まない方針がよいです。

## 8. Go / NoGo 判断
判断: 条件付きGo
理由:
- 現行コードには、SwitchBot API v1.1 の認証付き GET/POST を行うための最小要素は揃っている。
- 問題点は token/secret 固定、重複、例外握りつぶし、HTTPステータス欠落に集中しており、V2新規作成で整理しやすい。
- 既存クラスとの完全互換が不要なため、戻り値と例外方針を改善できる。
製造前に確認すべきこと:
- token/secret の注入元を外側レイヤーに置く方針でよいか。
- V2の戻り値を status/body を持つ結果型にすることでよいか。
- HTTP 4xx/5xx を例外ではなく結果として返す方針でよいか。
- timeout の標準値。
- SwitchBot Cloud API v1.1 の現行仕様。
推奨する次の作業:
- V2の外部仕様を短く確定する。
- 結果型と専用例外の名前・フィールドを決める。
- その後、SwitchBotApisV2.java を新規作成する。