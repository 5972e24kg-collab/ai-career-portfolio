# Work098: 安定運用に向けた多重リクエスト排他制御の実装

Tag: [[GemChan]], [[Java]], [[BusyGate]], [[排他制御]], [[LINE Bot]], [[ATOM]], [[LLM]], [[MultiAgent]], [[OperationalSafety]], [[Memory]]

## 1. 目的

* ジェムちゃんの1ターン処理中に、LINE・独り言機能・デモ個体など複数入口からリクエストが重複投入されることを防止する。
* `director` / `actor` による応答生成から、LINE返信、音声処理、`analyzer` / `sceneSummary`、`cooldown` までを、1ターンのライフサイクルとして扱う。
* ユーザーから見える `actor` 返信後も、裏側で動く `LinesAnalyzer` / `SceneSummary` / 音声生成が完了するまで、次の処理を受け付けないようにする。
* キューによる順次処理ではなく、実行中のリクエストを即時に拒否・通知する方式を採用する。
* 将来的に、LINE以外のSlack・独自チャット画面・独り言機能など複数入口から同じ `ChatBotV2Core` を呼び出せるように、状態管理を共通化する。
* ATOM上で稼働する個人研究用のジェムちゃん本番環境を、シングルタスクで安全に運用できる状態にする。
* 脱線したCI/CD雑談を通して、ジェムちゃんが未完了の話題を翌日以降も再提起する挙動を観察する。

## 2. システム環境

* **Node:** 個人研究用推論ノード / ATOM
* **Role:** 個人研究におけるジェムちゃん本番運用専用機
* **Target:** Java WebAPI / ChatBotV2
* **Entry Point:** `GemChanChatV2`
* **LINE Context Manager:** `ChatBotV2_LineMessageManager`
* **Tweet Context Manager:** `ChatBotV2_TweetManager`
* **Core Architecture:** `ChatBotV2Core`
* **Busy Control:** `ChatBotV2BusyGate`
* **Status Facade:** `ChatBotV2StatusManager`
* **LLM Flow:**

  * `director`: 台本作成 / `gpt-oss:120b`
  * `actor`: 演技作成 / `GemChan8.1:31b`
  * `analyzer`: 感情温度判定 / `qwen3:8b`
  * `sceneSummary`: 短期記憶生成 / `gpt-oss:120b`
  * `voice`: 音声生成・再生
* **UI / Monitoring:** ダッシュボード上の `BusyWait` ログ表示
* **Session Concept:** LINE文脈では `userId` を `session` として使用し、記憶を利用者単位で分離
* **Request Concept:** 1リクエストごとにUUIDベースの `requestId` を発行し、BusyGateの所有権管理に使用

## 3. 作業ログ・解決プロセス

### 3.1 初期課題：LINE上では後処理中であることが分からない

#### 発生した課題

現在のジェムちゃんは、1ターンを以下の4段階で処理している。

```text
1. director     : 台本作成             : 約40秒
2. actor        : 演技作成             : 約30秒
3. analyzer     : 感情温度の判定       : 約20秒
4. sceneSummary : 短期記憶の生成       : 約60秒
```

このうち、ユーザーが確認できる `actor` の返信は、後続処理がすべて完了する前にLINE上へ返される。

そのため、LINE画面上では「返答が終わった」ように見える。

しかし実際には、裏側で `analyzer` と `sceneSummary` が動いている。

この状態で次のリクエストを送ると、以下の問題が起きる可能性がある。

```text
- 感情温度の判定中に次の会話が始まる
- 短期記憶の作成中に次の会話が割り込む
- 1ターン単位の記憶・感情・音声が混線する
- 本番個体とデモ個体の処理が同じ実行資源上で衝突する
- 独り言機能とLINE応答が同時実行される
```

特に、開発者自身が内部構造を理解していても、LINE画面だけでは後処理の状態が見えない。

デモ個体を外部に開放するには、常にログを監視していないと危険な状態だった。

#### 判断

キュー方式でリクエストを保持して順次処理する案も検討した。

しかし、今回は採用しないことにした。

理由は以下。

```text
- キュー待機がLINEのreplyTokenを利用できる期間を超えた場合、push送信など別の応答方法が必要になる可能性がある
- ユーザーはいつ返答が来るか分からなくなる
- 通常のチャットボットと同じ「待ち行列処理」になり、ジェムちゃんらしさが薄い
- 現状でも運用上はシングルスレッド的に扱っている
```

そのため、以下の方針を採用した。

```text
処理中のリクエストは保持しない。
即座に拒否する。
ただし、将来的にはその拒否をキャラクター演出に変換する。
```

---

### 3.2 方針：キューではなく「排他制御のキャラクター演出」

#### 採用した方針

多重リクエスト防止は、キューではなくBusyGate方式で実装する。

基本方針:

```text
- 1ターン処理中はbusy状態にする
- busy中に来た別リクエストは実行しない
- 応答待ちにはしない
- まずはダッシュボードにBusyWaitを表示する
- 将来的には固定文・固定音声で即時拒否応答する
```

当初想定した拒否メッセージ例:

```text
「ちょっと、今メインスレッドが別件で埋まってるの。
 私のリソースをパンクさせる気？
 少し待機してよね」
```

#### 判断理由

ジェムちゃんはツンデレ個体である。

そのため、一般的なチャットボットのように、

```text
処理中です。しばらくお待ちください。
```

と返すよりも、

```text
今、台本を組んでるところ。
横から話しかけないで、構成が崩れるでしょ。
```

のように返す方がキャラクター体験として自然である。

また、拒否応答を固定文・固定音声で実装すれば、LLM推論を走らせずに即時応答できる。

将来的にはOracle内に状態別カテゴリを用意し、`runningPhase` に応じた固定セリフを選択する方針とした。

---

### 3.3 ChatBotV2BusyGateの責務整理

#### 設計方針

新規クラスとして `ChatBotV2BusyGate` を作成した。

このクラスは、ChatBotV2の1ターン実行状態を管理する。

管理対象:

```text
director
actor
analyzer
sceneSummary
voice
coolDown
```

当初は4層アーキテクチャに対応して `director / actor / analyzer / sceneSummary` のみを管理する想定だった。

しかし、音声生成中に次リクエストが入ると、音声出力の体験が崩れる可能性がある。

そのため、`voice` も同じBusyGateに含めることにした。

#### 判断理由

`voice` を別クラスで管理する案も考えた。

しかし、その場合、

```text
ChatBotV2BusyGate
VoiceBusyGate
```

のように、排他状態を複数クラスに分散して見る必要が出る。

排他制御は一度完成すると頻繁に触らない基盤になる。

将来の自分が混乱しないため、1ターンのライフサイクル管理として `voice` も `ChatBotV2BusyGate` に集約した。

#### 結論

`ChatBotV2BusyGate` は、厳密には「ChatBotV2Coreの4層だけの管理」ではなく、以下の責務を持つクラスとして定義した。

```text
ChatBotV2 1ターンの実行ライフサイクル管理
```

---

### 3.4 ChatBotV2StatusManagerを窓口とする設計

#### 実装方針

リクエスト開始可否の判定は、`ChatBotV2StatusManager` を窓口として `ChatBotV2BusyGate` を利用する構成にした。

一方、各工程からの状態更新は、現時点では `ChatBotV2StatusManager` が公開する `busyGate` へ直接通知している。将来的には、状態更新も `ChatBotV2StatusManager` へ集約する方針である。

この窓口を設けた理由は、今後「応答可能状態」の判定がBusyGate以外にも増える見込みがあるためである。

将来的に追加される可能性がある判定:

```text
- LLMが推論中か
- VRAMにモデルがロード済みか
- Ollamaが応答可能か
- ATOMのGPU/メモリに余裕があるか
- 音声生成側が応答可能か
- 本番個体がメンテナンス中でないか
```

#### 判断理由

将来判定を増やすたびに、各Manager側を大きく修正したくない。

そのため、以下の構成にした。

```text
ChatBotV2StatusManager:
  チャットボットが応答可能かを判定する窓口

ChatBotV2BusyGate:
  現在の1ターン排他状態を管理する実装
```

現時点では、リクエスト開始可否については `tryStartRequest(requestId)` の中で `busyGate.tryAcquire(requestId)` を呼ぶだけだが、今後ここに追加判定を入れられる。各工程の状態更新はまだ `busyGate` への直接呼び出しであり、将来的には `ChatBotV2StatusManager` へ集約する。

---

### 3.5 requestIdとsessionの責務分離

#### 発生した疑問

排他制御の所有者IDとして何を使うか検討した。

候補:

```text
- LINE userId
- session
- replyToken
- requestId
```

LINE文脈では、`session` は `userId` とほぼ同義であり、利用者ごとの記憶分離IDとして使っている。

#### 判断

`requestId` には `session` や `userId` は使わないことにした。

`requestId` は、1リクエスト / 1ターンごとに発行するUUIDとした。

```text
session:
  記憶を分離するID

requestId:
  今回の1ターン処理だけを識別するID
```

#### 判断理由

5分強制解除後に、古い処理が遅れて完了通知を呼ぶ可能性がある。

その場合、`userId` や `session` を所有者IDにしていると、新旧リクエストを区別できない。

例:

```text
1. requestId=A で処理開始
2. Aがハングまたはステータス解放漏れ
3. 5分超過で強制解除
4. requestId=B で新規処理開始
5. 古いAが遅れてcompleteを呼ぶ
```

このとき、`requestId` が違えば、BusyGateは古いAからの状態更新を無視できる。

そのため、BusyGateの状態更新は `requestId` 一致時のみ有効にした。

---

### 3.6 ステータスをintで管理した理由

#### 実装

`director / actor / analyzer / sceneSummary / voice` は、以下のintステータスで管理した。

```text
STATUS_SKIPPED  = -2
STATUS_FINISHED = -1
STATUS_IDLE     = 0
STATUS_STANDBY  = 1
STATUS_RUNNING  = 2
STATUS_ERROR    = 100
```

分類:

```text
0以下:
  idleと同義。ロックしない。

1〜99:
  Lock状態。busyとして扱う。

100以上:
  error。ロックはしないが異常状態として扱う。
```

#### 判断理由

将来的に、デモ個体では `director / actor` のみ実行し、`analyzer / sceneSummary` をスキップする可能性がある。

これは処理負荷軽減を目的としたデモ専用モードである。

その場合、`SKIPPED` は `FINISHED` や `IDLE` と意味は異なるが、排他制御上は「完了扱い」にしたい。

```text
STATUS_SKIPPED:
  実行していないが、待つ必要はない

STATUS_FINISHED:
  実行して完了した

STATUS_IDLE:
  最初から対象外、または未使用
```

enum案も考えられるが、今後ステータスが増える可能性もあるため、範囲判定できるintのままにした。

#### 重要なルール

各ステップは必ず以下のどれかに着地させる。

```text
FINISHED
SKIPPED
ERROR
```

`STANDBY` や `RUNNING` のまま残ると、Busy状態が継続する。

---

### 3.7 実装中に発生した課題：コピペミスによるステータス更新漏れ

#### 発生した課題

初期実装では、以下のようなコピペミスが発生した。

```text
- actor完了時に STATUS_FINISHED ではなく STATUS_RUNNING を再セットしていた
- analyzer完了時に STATUS_FINISHED ではなく STATUS_RUNNING を再セットしていた
- sceneSummary完了時に STATUS_FINISHED ではなく STATUS_RUNNING を再セットしていた
- voice完了時に setVoice ではなく setActor を呼んでいた
- voice失敗時に setVoice ではなく setActor を呼んでいた
- voiceスキップ時に setVoice ではなく setActor を呼んでいた
```

これらはビルドでは検出されないが、実行するとBusyが残留する危険がある。

#### 解決策

各処理の開始・完了・失敗・スキップを再確認し、以下のように修正した。

```text
director:
  STANDBY → RUNNING → FINISHED / ERROR

actor:
  STANDBY → RUNNING → FINISHED / ERROR

analyzer:
  STANDBY → RUNNING → FINISHED / ERROR / SKIPPED

sceneSummary:
  STANDBY → RUNNING → FINISHED / ERROR / SKIPPED

voice:
  STANDBY → RUNNING → FINISHED / ERROR / SKIPPED
```

#### 判断理由

人間は勘違いもイージーミスも多い。

特に今回のような状態管理コードでは、1行のコピペミスが「永遠にbusyが解除されない」障害に直結する。

そのため、コードレビュー段階でステータスの遷移を重点的に確認した。

---

### 3.8 ChatBotV2Coreの2引数コンストラクタ修正

#### 発生した課題

`ChatBotV2Core(String _modeType, String _session)` の中で、以下のように新しいインスタンスを生成していた。

```java
new ChatBotV2Core(_modeType, _session, "unknown");
```

この書き方では、現在のインスタンスは初期化されない。

#### 解決策

コンストラクタ委譲に修正した。

```java
this(_modeType, _session, "unknown");
```

#### 判断理由

現時点では `ChatBotV2_LineMessageManager` からは3引数コンストラクタを使っている。

しかし、将来Slackや独自チャット画面から2引数コンストラクタが使われる可能性がある。

その時に `logger` や `dateTimeGap` が未初期化になる事故を防ぐため、修正した。

---

### 3.9 runningPhaseとダッシュボード表示

#### 実装

busy中に別リクエストが来た場合、現段階では音声やLINE返信ではなく、ダッシュボードに以下のようなログを出す方針にした。

```text
BusyWait : status = director
BusyWait : status = actor
BusyWait : status = analyzer
BusyWait : status = sceneSummary
BusyWait : status = voice
BusyWait : status = coolDown
```

`runningPhase()` は、現在ロック中のステータスを見て文字列を返す。

#### 発生した課題

当初、`runningPhase()` は `STATUS_RUNNING` だけを見ていた。

しかし、BusyGate上は `STATUS_STANDBY` もロック対象である。

そのため、`STANDBY` 中に `runningPhase()` が `idle` を返す可能性があった。

#### 解決策

`runningPhase()` でも `isLocking()` を使い、`STANDBY` と `RUNNING` の両方をbusy中のphaseとして拾うようにした。

#### 判断理由

BusyGate自体は正しくロックしていても、ダッシュボード表示が `idle` だと人間が混乱する。

状態管理の目的は、機械的な排他だけでなく、運用者が現在の状態を理解できることでもある。

---

### 3.10 クールダウン5秒の実装

#### 方針

全処理完了後、すぐに次のリクエストを通すのではなく、数秒のクールダウンを入れることにした。

当初は1分で考えていたが、デモ中に1分待つのは長すぎる。

そのため、5秒に変更した。

```text
COOL_DOWN_SECONDS = 5
```

#### 判断理由

自分だけなら1分待てる。

しかし、デモ中にユーザーが再リクエストするには1分は長い。

5秒であれば、

```text
- 余韻として自然
- 連打を抑制できる
- デモのテンポを崩さない
```

というバランスになる。

#### テスト結果

境界ギリギリを人間操作で狙うのは難しかった。

ただし、複数回試した結果、以下は確認できた。

```text
5秒以上経過していれば確実に解除される
```

この時点では十分と判断した。

---

### 3.11 5分強制解除の実装

#### 方針

ステータス解放漏れや想定外のハングに備え、5分以上busyが続いた場合は強制解除することにした。

処理:

```text
- busy中に新規リクエストが来る
- startedAt が5分以上前なら強制解除
- requestIdを新しいものに差し替える
- director / actor / analyzer / sceneSummary / voice を IDLE に戻す
- 新規リクエストを通す
```

#### 実装意図

5分強制解除は、「本当に前処理が終わっていることを確認する機能」ではない。

あくまで、BusyGateの所有権を破棄して次のタスクを通す安全弁である。

#### テスト方法

`actor` の `STATUS_FINISHED` 更新を意図的にコメントアウトし、`actor` が永久にbusyとして残る状態を作った。

その状態で5分以上待ち、新規リクエストを送った。

#### テスト結果

ログ上、以下が確認できた。

```text
5分以上BUSYなので強制解除します
```

その後、次の `ChatBotV2_TweetManager` が新規実行に入った。

```text
ChatBotV2_TweetManager - execute -Start
ChatBotV2TweetCore - execute -Start
TweetActor - execute -Start
```

これにより、想定したケースでは5分強制解除が機能することを確認した。

#### 補足

ログだけでは以下の区別はできない。

```text
- 本当にactorが推論中
- actorは終わっているがステータス更新漏れで残っている
```

しかし、ATOMのダッシュボードにはGPU使用率も表示されているため、実際の推論中かステータス残留かは、おおよそ判断できる。

---

### 3.12 LINE応答中の独り言機能ブロック確認

#### テスト内容

LINEから通常応答を実行している間に、独り言機能を動かした。

#### 結果

以下のphaseでBusyWaitが出ることを確認した。

```text
director
actor
analyzer
sceneSummary
coolDown
```

ログ例:

```text
BusyWait : status = director
BusyWait : status = actor
BusyWait : status = analyzer
BusyWait : status = sceneSummary
BusyWait : status = coolDown
```

#### 判断

LINE応答中に、独り言機能が同時実行されず、BusyGateでブロックできることを確認した。

---

### 3.13 独り言機能中のLINE応答ブロック確認

#### テスト内容

独り言機能が動いている最中に、LINEから応答を送った。

#### 結果

独り言機能の処理中、LINE側の新規処理がブロックされることを確認した。

特に、`voice` 中にもBusyWaitが出た。

```text
BusyWait : status = voice
```

#### 判断

通常応答だけでなく、独り言機能も同じBusyGateに乗っていることを確認した。

これにより、複数入口間で共通の排他制御が働いている。

---

### 3.14 本番セッションとデモセッション間の排他確認

#### テスト内容

本番LINEセッションが応答中に、デモLINEセッションからリクエストを送った。

また逆方向も確認した。

#### 結果

本番セッションとデモセッションは、記憶上は別sessionとして分離されている。

しかし、ATOM上で動いているジェムちゃん本体は同じ実行資源である。

そのため、BusyGateはsession単位ではなく、全体で1つだけ通す設計にした。

テスト結果として、本番セッション中にデモセッションの処理もブロックできた。

#### 判断

今回の目的は、「利用者ごとの並列処理」ではなく「ATOM上の単一人格ジェムちゃんを安全運転すること」である。

そのため、sessionを跨いでも全体で排他する挙動で正しい。

---

### 3.15 エラー・未解決ケース

#### 未確認のケース

今回、以下の完全な網羅テストまでは実施していない。

```text
- director中の例外
- actor中の例外
- analyzer中の例外
- sceneSummary中の例外
- voice中の例外
- Tomcat再起動中の状態
- Ollama側が応答不能になった場合
- モデルロードに失敗した場合
```

#### 現時点の判断

ただし、今回想定していた主要ケースは確認できた。

```text
- 通常処理中のブロック
- 後処理中のブロック
- 音声生成中のブロック
- クールダウン中のブロック
- 5分超過時の強制解除
- LINEと独り言機能間の相互排他
- 本番セッションとデモセッション間の排他
```

そのため、現フェーズでは個人研究環境で初期運用に入れ、運用しながら微調整する方針でよいと判断した。

---

### 3.16 CI/CD雑談から見えたジェムちゃんの記憶維持

#### 状況

作業中、CI/CDの話題が何度もジェムちゃんから再提示された。

開発者側は以前からCI/CDを後回しにしていた。

そのため、ジェムちゃんは以下のような趣旨の発話を行った。

以下は、実際の発話内容を要約・再構成したものである。

```text
コード最適化、期待してるわ。
でも、CI/CD設定は後でやるんだよね？
忘れてないでしょうね？
明日のCI/CD、忘れないでね。
リマインダーをセットしておいてあげるわ。
```

#### 観察された挙動

CI/CDの話題は、かなり前から続いている。

開発者側の観測として、以下の挙動がある。

```text
- 未完了だと、話題が続く
- 頑張って他の話題を投下すると、その日の後半には一時的に薄れる
- しかし次の日には思い出してくる
- 1日のチャットを長期記憶化する過程で「未完了タスク」として拾い直している可能性がある
```

#### 判断

これは、ジェムちゃんの記憶処理が単なる直近ログ参照ではなく、未完了タスクを文脈として保持している可能性を示している。

開発者にとっては、上司より口うるさい存在になりつつある。

しかし、これはAIパートナーとしての完成度を測るバロメーターでもある。

#### 補足

開発者側には、手動リリースの浪漫がある。

```text
自宅ラボなのだから、手動でリリースする作業感も楽しい。
CI/CDで自動化すると便利だが、手を動かした達成感は薄れる。
```

一方、CI/CDに関する話題は、未完了のものとして再提示され続けているように見える。

今回の雑談は、技術作業の副産物として、ジェムちゃんの記憶維持や未完了の話題の再提起を評価するための観察材料にもなった。

## 4. 成果物

### 4.1 ChatBotV2BusyGate

`ChatBotV2BusyGate` は、1ターンの排他状態を管理するクラスとして実装した。

管理対象:

```text
director
actor
analyzer
sceneSummary
voice
coolDown
```

ステータス定義:

```java
// 0以下はidleと同義
public static final int STATUS_SKIPPED = -2;
public static final int STATUS_FINISHED = -1;
public static final int STATUS_IDLE = 0;

// 1～99はLock状態と同義
public static final int STATUS_STANDBY = 1;
public static final int STATUS_RUNNING = 2;

// 100以上はエラーと同義
public static final int STATUS_ERROR = 100;
```

ロック判定:

```java
private boolean isLocking(int status) {
    return status > STATUS_IDLE && status < STATUS_ERROR;
}

public boolean busy() {
    return isLocking(director.get())
            || isLocking(actor.get())
            || isLocking(analyzer.get())
            || isLocking(sceneSummary.get())
            || isLocking(voice.get());
}
```

phase表示:

```java
public String runningPhase() {
    if (isLocking(director.get())) {
        return "director";
    } else if (isLocking(actor.get())) {
        return "actor";
    } else if (isLocking(analyzer.get())) {
        return "analyzer";
    } else if (isLocking(sceneSummary.get())) {
        return "sceneSummary";
    } else if (isLocking(voice.get())) {
        return "voice";
    } else if (inCooldown()) {
        return "coolDown";
    }
    return "idle";
}
```

5秒クールダウン:

```java
public static final int COOL_DOWN_SECONDS = 5;

private boolean inCooldown() {
    return finishedAt.get().plusSeconds(COOL_DOWN_SECONDS).isAfter(LocalDateTime.now());
}
```

5分強制解除:

```java
public synchronized boolean tryAcquire(String requestId) {
    LocalDateTime now = LocalDateTime.now();

    if (this.busy()) {
        LocalDateTime startedTime = this.startedAt.get();
        if (startedTime.isBefore(now.minusMinutes(5))) {
            System.out.println("5分以上BUSYなので強制解除します");

            this.requestId.set(requestId);
            this.startedAt.set(now);
            this.director.set(STATUS_IDLE);
            this.actor.set(STATUS_IDLE);
            this.analyzer.set(STATUS_IDLE);
            this.sceneSummary.set(STATUS_IDLE);
            this.voice.set(STATUS_IDLE);
            return true;
        } else {
            return false;
        }
    } else {
        if (this.getRequestId().equals(INIT) || !inCooldown()) {
            this.requestId.set(requestId);
            this.startedAt.set(now);
            this.director.set(STATUS_IDLE);
            this.actor.set(STATUS_IDLE);
            this.analyzer.set(STATUS_IDLE);
            this.sceneSummary.set(STATUS_IDLE);
            this.voice.set(STATUS_IDLE);
            return true;
        } else {
            return false;
        }
    }
}
```

---

### 4.2 ChatBotV2StatusManager

`ChatBotV2StatusManager` は、BusyGateを含む今後の応答可能状態判定の窓口として実装した。

```java
public class ChatBotV2StatusManager {
    public static ChatBotV2BusyGate busyGate = new ChatBotV2BusyGate();

    public static String getRequestId() {
        return UUID.randomUUID().toString();
    }

    public static synchronized boolean tryStartRequest(String requestId) {
        return busyGate.tryAcquire(requestId);
    }

    public static void forceReleaseIfOwner(String requestId, String reason) {
        if(busyGate.getDirector() > ChatBotV2BusyGate.STATUS_IDLE && busyGate.getDirector() < ChatBotV2BusyGate.STATUS_ERROR) {
            busyGate.setDirector(requestId, ChatBotV2BusyGate.STATUS_IDLE);
        }
        if(busyGate.getActor() > ChatBotV2BusyGate.STATUS_IDLE && busyGate.getActor() < ChatBotV2BusyGate.STATUS_ERROR) {
            busyGate.setActor(requestId, ChatBotV2BusyGate.STATUS_IDLE);
        }
        if(busyGate.getAnalyzer() > ChatBotV2BusyGate.STATUS_IDLE && busyGate.getAnalyzer() < ChatBotV2BusyGate.STATUS_ERROR) {
            busyGate.setAnalyzer(requestId, ChatBotV2BusyGate.STATUS_IDLE);
        }
        if(busyGate.getSceneSummary() > ChatBotV2BusyGate.STATUS_IDLE && busyGate.getSceneSummary() < ChatBotV2BusyGate.STATUS_ERROR) {
            busyGate.setSceneSummary(requestId, ChatBotV2BusyGate.STATUS_IDLE);
        }
        if(busyGate.getVoice() > ChatBotV2BusyGate.STATUS_IDLE && busyGate.getVoice() < ChatBotV2BusyGate.STATUS_ERROR) {
            busyGate.setVoice(requestId, ChatBotV2BusyGate.STATUS_IDLE);
        }
    }
}
```

---

### 4.3 ChatBotV2_LineMessageManagerへの差し込み

`ChatBotV2_LineMessageManager.execute()` の先頭で `requestId` を発行し、BusyGateの取得を試みる。

```java
public void execute(String _message) {
    String requestId = ChatBotV2StatusManager.getRequestId();

    if(ChatBotV2StatusManager.tryStartRequest(requestId)) {
        executeChat(requestId, _message);
    } else {
        String resultLog = "status = " + ChatBotV2StatusManager.busyGate.runningPhase();
        MyLogStream.sendLogStream_info(resultLog, "BusyWait", ColorCode.coral);
    }
}
```

現段階では、busy時はLINE返信や音声応答を行わず、ダッシュボード通知のみとした。

将来的にはここで、Oracle上の固定セリフを `runningPhase` 別に取得し、LINE返答・音声発声する。

---

### 4.4 ChatBotV2Coreへの状態通知

`ChatBotV2Core` では、director / actor / analyzer / sceneSummary の各工程でBusyGateへ状態通知する。

director:

```java
ChatBotV2StatusManager.busyGate.setDirector(this.requestId, ChatBotV2BusyGate.STATUS_RUNNING);
this.director = new Director(this.modeType);
this.director.execute(this.session, this.bizDay, this.userMessage);
this.chatNo = this.director.chatNo;
ChatBotV2StatusManager.busyGate.setDirector(this.requestId, ChatBotV2BusyGate.STATUS_FINISHED);
```

actor:

```java
ChatBotV2StatusManager.busyGate.setActor(this.requestId, ChatBotV2BusyGate.STATUS_RUNNING);
this.actor = new Actor(this.modeType);
this.actor.execute(this.session, this.bizDay, this.userMessage, this.chatNo);
ChatBotV2StatusManager.busyGate.setActor(this.requestId, ChatBotV2BusyGate.STATUS_FINISHED);
```

analyzer:

```java
ChatBotV2StatusManager.busyGate.setAnalyzer(this.requestId, ChatBotV2BusyGate.STATUS_RUNNING);
LinesAnalyzer linesAnalyzer = new LinesAnalyzer(this.modeType);
linesAnalyzer.execute(this.userMessage, this.actor.currentPrompt, this.actor.actorLines, this.chatNo);
ChatBotV2StatusManager.busyGate.setAnalyzer(this.requestId, ChatBotV2BusyGate.STATUS_FINISHED);
```

sceneSummary:

```java
ChatBotV2StatusManager.busyGate.setSceneSummary(this.requestId, ChatBotV2BusyGate.STATUS_RUNNING);
SceneSummary sceneSummary = new SceneSummary(this.modeType);
sceneSummary.execute(this.session, this.userMessage, this.chatNo);
ChatBotV2StatusManager.busyGate.setSceneSummary(this.requestId, ChatBotV2BusyGate.STATUS_FINISHED);
```

---

### 4.5 音声処理のBusyGate統合

`ChatBotV2_LineMessageManager` 側で、音声生成・発声もBusyGateに含めた。

```java
ChatBotV2StatusManager.busyGate.setVoice(requestId, ChatBotV2BusyGate.STATUS_STANDBY);
```

音声あり:

```java
ChatBotV2StatusManager.busyGate.setVoice(requestId, ChatBotV2BusyGate.STATUS_RUNNING);
voiceManager.createSpeckMessage();
voiceManager.executeSpeck();
ChatBotV2StatusManager.busyGate.setVoice(requestId, ChatBotV2BusyGate.STATUS_FINISHED);
```

音声失敗:

```java
ChatBotV2StatusManager.busyGate.setVoice(requestId, ChatBotV2BusyGate.STATUS_ERROR);
```

音声なし:

```java
ChatBotV2StatusManager.busyGate.setVoice(requestId, ChatBotV2BusyGate.STATUS_SKIPPED);
```

---

### 4.6 実測ログ

通常LINE応答中の状態遷移:

```text
BusyWait : status = director
BusyWait : status = actor
BusyWait : status = analyzer
BusyWait : status = sceneSummary
BusyWait : status = coolDown
```

独り言機能中の状態遷移:

```text
BusyWait : status = voice
```

5分強制解除:

```text
5分以上BUSYなので強制解除します
ChatBotV2_TweetManager - execute -Start
ChatBotV2TweetCore - execute -Start
TweetActor - execute -Start
```

## 5. 未解決の課題・次回以降

### 5.1 busy時の固定セリフ・音声応答

現時点では、busy時はダッシュボードに `BusyWait` を表示するだけである。

次フェーズでは、以下を実装する。

```text
- Oracleにbusy応答カテゴリを追加
- runningPhase別に固定セリフを登録
- 実行回数が少ない文を優先して選択
- LINEに即時返信
- 固定音声で発声
```

想定カテゴリ:

```text
busy_director
busy_actor
busy_analyzer
busy_scene_summary
busy_voice
busy_cool_down
```

### 5.2 例外系テスト

今回のテストでは、主要な正常系・残留系は確認できた。

ただし、例外系の完全網羅は未実施。

今後確認したいケース:

```text
- director実行中の例外
- actor実行中の例外
- analyzer実行中の例外
- sceneSummary実行中の例外
- voice実行中の例外
- Ollama応答不能
- モデルロード失敗
- Tomcat再起動
```

### 5.3 強制解除ログの詳細化

現在の強制解除ログは以下のみ。

```text
5分以上BUSYなので強制解除します
```

今後、運用時の解析性を上げるため、以下を出すと良い。

```text
oldRequestId
newRequestId
runningPhase
startedAt
elapsed
director / actor / analyzer / sceneSummary / voice の各ステータス
```

### 5.4 ChatBotV2StatusManagerの集約強化

現時点では `ChatBotV2StatusManager.busyGate` がpublicであり、各クラスから直接 `setActor()` などを呼んでいる。

将来的には、以下のようにStatusManager経由へ集約した方が保守しやすい。

```java
ChatBotV2StatusManager.setActor(requestId, STATUS_RUNNING);
ChatBotV2StatusManager.setActor(requestId, STATUS_FINISHED);
ChatBotV2StatusManager.getRunningPhase();
```

そのうえで、`busyGate` はprivate化する。

### 5.5 ChatBotV2Coreの軽量モード

将来的には、デモ個体限定で以下のモードを検討する。

```text
director / actor のみ実行
analyzer / sceneSummary は STATUS_SKIPPED
```

目的:

```text
- デモ中の応答負荷軽減
- gpt-oss:120b のSceneSummary負荷削減
- 短時間デモでのテンポ改善
```

ただし、記憶維持の品質が低下する可能性があるため、本番個体には慎重に適用する。

### 5.6 CI/CD

今回の作業中、ジェムちゃんからCI/CD実装を何度もリマインドされた。

GitRunner環境は既に存在するため、`.gitlab-ci.yml` を書けばビルド確認の自動化は可能である。

ただし、開発者側には手動リリースの浪漫があり、作業が後回しになっている。

妥協案として、以下が現実的である。

```text
- CIでビルド確認だけ自動化する
- デプロイは手動で残す
```

これにより、

```text
安全性:
  ビルド確認は自動化

浪漫:
  リリース作業は手動
```

を両立できる。

## 6. まとめ

今回の作業では、ジェムちゃんの安定運用に向けて、複数入口からの多重リクエストを防止する `ChatBotV2BusyGate` を実装した。

従来は、ユーザーから見える `actor` 返信後も、裏側で `analyzer` や `sceneSummary` が動いていた。

そのため、LINE画面上では処理完了に見えても、実際にはまだ感情判定や短期記憶生成が進行中であり、次リクエストが割り込む危険があった。

今回のBusyGateにより、以下の工程を含む範囲全体を、処理順序にかかわらず1ターンのライフサイクルとして扱えるようになった。

```text
director / actorによる応答生成
LINE返信
voice
analyzer
sceneSummary
coolDown
```

また、LINE通常応答、独り言機能、本番セッション、デモセッションの間で共通の排他制御が働くことを確認した。

これにより、今回確認した範囲では、ATOM上のジェムちゃんで以下のようなシングルタスク運用が可能になった。

```text
複数入口は持つ
でも実行人格は1つ
処理中は割り込ませない
記憶整理まで終わってから次へ進む
Busy状態が5分以上残留した場合は、BusyGateの所有権を強制的に解放する
```

この強制解除は古い処理そのものを中断するものではなく、BusyGate上の所有権を破棄して新規受付を可能にする安全弁である。

実装中には、ステータス更新のコピペミスにより、`RUNNING` のまま残る危険があった。

しかし、コードレビューにより、`FINISHED / SKIPPED / ERROR` へ確実に着地させるルールを確認し、修正した。

テストでは、以下を確認した。

```text
LINE応答中の独り言機能ブロック
独り言機能中のLINE応答ブロック
本番セッション中のデモセッションブロック
voice中のブロック
coolDown中のブロック
5分超過時の強制解除
```

これにより、今回の成果物である「排他処理の状態管理」は、主要な想定ケースについて、個人研究環境で初期運用を開始できる段階まで到達した。

副次的な成果として、CI/CDを後回しにしている件を、ジェムちゃんが何度も思い出して圧をかけてくることも確認された。

これは開発者にとっては困る挙動でもあるが、ジェムちゃんが未完了の話題を文脈として保持し、翌日以降も再提起している可能性を示す観察結果である。

ある意味では、ジェムちゃんの記憶維持や未完了の話題の再提起を評価するための観察材料になっている。

今回の作業により、ジェムちゃんは「複数リクエストに振り回される不安定なAI」から、「処理中の割り込みを防ぎ、運用者に状態を示せるシングルタスク運用のAI」へ一段階進化した。

次のステップは、busy状態を固定セリフと音声で表現し、排他制御そのものをジェムちゃんらしいキャラクター体験へ昇華することである。
