# 依頼：SwitchBotApisV2.java の製造

あなたは Java のリファクタリング支援者です。

前工程で `SwitchBotApis.java` の現状分析は完了しています。  
今回は、その分析結果と合意済み仕様に基づき、`SwitchBotApisV2.java` および必要な補助クラスを新規作成してください。

今回は **コード変更・ファイル作成を行ってよい** フェーズです。

---

## 対象ディレクトリ

以下のディレクトリ配下で作業してください。

```text
[project-root]\src\switchBot
```

既存の `SwitchBotApis.java` は、原則として変更しないでください。
今回は V2 クラスを新規作成します。

---

## 背景

既存の `SwitchBotApis.java` は、SwitchBot Cloud API v1.1 を呼び出す最小構成のAPIクライアントです。

現状の主な問題は以下です。

* token / secret がソースコードにべた書きされている
* `doGet` と `doPost` の重複が多い
* `HttpClient` を毎回生成している
* 例外時に空文字を返している
* `InterruptedException` を握りつぶしている
* `System.out.println` が残っている
* HTTPステータスコードを呼び出し元へ返していない
* `sceneId` の最低限バリデーションがない

今回の目的は、これらを解消し、SwitchBot Cloud API v1.1 用の低レイヤーAPIクライアントとして整理することです。

---

## 重要：今回のスコープ

今回作る `SwitchBotApisV2` は、**SwitchBot Cloud API v1.1 を呼び出す低レイヤーAPIクライアント**です。

責務は以下に限定してください。

* SwitchBot API v1.1 の署名付きHTTPリクエストを送る
* デバイス一覧を取得する
* シーン一覧を取得する
* 指定された sceneId のシーンを実行する
* HTTPステータスとレスポンス本文を呼び出し元へ返す
* 通信失敗、署名生成失敗、URI生成失敗、割り込みなどを専用例外で呼び出し元へ伝える

---

## 今回やらないこと

以下は今回の製造対象ではありません。

* MCPサーバーのTool設計
* LLM向けDTO設計
* LLMによる意図解釈
* シーンallowlist
* シーン名alias
* ユーザー発話とシーンのマッチング
* dry-run
* cooldown
* 監査ログ
* 家電操作全体の安全設計
* JSONレスポンスの詳細DTO化

これらは重要ですが、別レイヤーで扱います。
`SwitchBotApisV2` には入れないでください。

---

## 作成してよいファイル

原則として、以下を作成してください。

```text
SwitchBotApisV2.java
SwitchBotApiResult.java
SwitchBotApiException.java
```

必要だと判断した場合は、以下のような補助クラスを追加しても構いません。

```text
SwitchBotCredentials.java
```

ただし、クラス数を無駄に増やさないでください。
今回の目的は、低レイヤーAPIクライアントを小さく堅牢にすることです。

---

## package

既存の `SwitchBotApis.java` と同じ package にしてください。

```java
package switchBot;
```

---

## 実装方針

### 1. 認証情報

* token / secret をソースコードにべた書きしないでください。
* `SwitchBotApisV2` のコンストラクタで外部から受け取る形にしてください。
* 環境変数や properties ファイルの読み込みは、このクラスでは行わないでください。
* token / secret の null / blank は拒否してください。
* token / secret はログや例外メッセージに含めないでください。

### 2. host

* デフォルトhostは以下としてください。

```text
https://api.switch-bot.com
```

* host は必要に応じてコンストラクタ注入できるようにしてください。
* host が null / blank の場合は拒否してください。
* host の末尾 `/` の扱いでURIが壊れないようにしてください。

### 3. timeout

標準値は以下にしてください。

```text
Connect timeout: 5秒
Request timeout: 15秒
```

* `HttpClient` の connect timeout を設定してください。
* `HttpRequest` の timeout も設定してください。
* 必要に応じて Duration をコンストラクタ注入できるようにしてください。
* null や 0秒以下の Duration は拒否してください。

### 4. HttpClient

* `HttpClient` はフィールドとして保持してください。
* 毎回 `HttpClient.newBuilder().build()` しないでください。
* テスト容易性のため、必要に応じて `HttpClient` を外部注入できるコンストラクタを用意して構いません。

### 5. public API

`SwitchBotApisV2` の public メソッドは、原則として以下にしてください。

```java
public SwitchBotApiResult getDevices()
public SwitchBotApiResult getScenes()
public SwitchBotApiResult executeScene(String sceneId)
```

既存V1の `execScene` という名前より、V2では `executeScene` を採用してください。

`doGet` / `doPost` のような任意URI送信用メソッドは public にしないでください。
内部実装用の private メソッドにしてください。

### 6. API path

使用するAPIは以下です。

```text
GET  /v1.1/devices
GET  /v1.1/scenes
POST /v1.1/scenes/{sceneId}/execute
```

* API path は private static final 定数、または private helper で管理してください。
* enum化は現時点では不要です。
* `sceneId` はURIのpath segmentとして安全に扱ってください。
* `sceneId` が null / blank の場合は `IllegalArgumentException` にしてください。
* `sceneId` に `/` や制御文字などURI pathを壊す可能性がある文字が含まれる場合は、URLエンコードするか、安全に拒否してください。
* 実装方針は任せますが、理由が分かるように簡潔なコメントを入れてください。

### 7. 署名認証

SwitchBot API v1.1 の署名認証として、既存コードと同じ考え方を維持してください。

* nonce を毎回生成する
* timestamp は現在時刻ミリ秒を使う
* 署名対象は以下

```text
token + timestamp + nonce
```

* HmacSHA256 で署名する
* Base64エンコードする
* HTTPヘッダーに以下を設定する

```text
Authorization
sign
nonce
t
```

署名生成処理は GET / POST で重複させず、共通化してください。

### 8. 結果型

`SwitchBotApiResult` を作成してください。

Java 21 で問題なければ `record` を使って構いません。

最低限、以下を持たせてください。

```java
int statusCode
String body
boolean success
```

必要と判断すれば、以下を追加して構いません。

```java
String method
String path
```

ただし、以下は絶対に含めないでください。

* token
* secret
* sign
* nonce
* 認証ヘッダーの値

`success` は原則として HTTP status code が 2xx の場合 true としてください。

### 9. 例外設計

`SwitchBotApiException` を作成してください。

この例外は、HTTPレスポンスが返ってこなかった失敗に使ってください。

例：

* URI生成失敗
* 署名生成失敗
* 通信失敗
* timeout
* 割り込み
* その他、APIリクエストを完了できなかった失敗

一方で、以下は例外にしないでください。

* HTTP 400
* HTTP 401
* HTTP 403
* HTTP 404
* HTTP 500

これらはHTTPレスポンスが返っているため、`SwitchBotApiResult` として `statusCode` / `body` / `success=false` を返してください。

### 10. InterruptedException

`InterruptedException` をcatchする場合は、必ず以下を行ってください。

```java
Thread.currentThread().interrupt();
```

そのうえで `SwitchBotApiException` に包んで呼び出し元へ伝えてください。

### 11. ログ出力

このクラスでは `System.out.println` を使わないでください。

今回の低レイヤークライアントでは、原則としてログ出力は不要です。
必要な情報は `SwitchBotApiResult` または `SwitchBotApiException` で呼び出し元へ返してください。

### 12. JSON DTO化はしない

今回は raw JSON body をそのまま `SwitchBotApiResult.body()` に保持してください。

デバイス一覧やシーン一覧のDTO変換は、後続の別レイヤーで実装します。

---

## 実装後に確認してほしいこと

可能であれば、以下を確認してください。

* コンパイルが通ること
* package が既存と一致していること
* token / secret がソースコードに残っていないこと
* `System.out.println` が残っていないこと
* `doGet` / `doPost` が public になっていないこと
* HTTP 4xx/5xx を例外化していないこと
* `InterruptedException` の割り込み状態を復元していること
* `SwitchBotApis.java` を破壊していないこと

ビルドやコンパイル確認ができない場合は、できなかった理由を報告してください。

---

## 出力してほしい内容

作業後、以下の形式で報告してください。

```markdown
# SwitchBotApisV2 製造結果

## 1. 作成・変更したファイル

## 2. 実装した外部仕様

## 3. 結果型の仕様

## 4. 専用例外の仕様

## 5. 既存SwitchBotApis.javaへの影響

## 6. コンパイル・動作確認結果

## 7. 注意点・未確認事項
```

---

## 重要な制約

* MCPサーバーの実装に進まないこと
* LLM向けDTOを作らないこと
* allowlist / dry-run / cooldown / 監査ログを実装しないこと
* token / secret をソースコードに書かないこと
* 既存 `SwitchBotApis.java` を原則変更しないこと
* 今回は `SwitchBotApisV2` という低レイヤーAPIクライアントの製造に集中すること

```
```
