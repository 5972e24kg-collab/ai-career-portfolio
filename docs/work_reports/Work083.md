# Work083: 屋外デモ対応ReactアバターへのHTTP音声配信レイヤー実装

Tag: [[React]], [[SSE]], [[Java]], [[TTS]], [[nginx]], [[Docker]], [[UX]], [[Gem-chan]], [[Avatar]], [[Demo]]

## 1. 目的

- 外部デモ環境において、ジェムちゃんの字幕・表情・音声をReactダッシュボード上で完結させる。
- これまでローカル検証環境周辺の音声再生サーバーに依存していた音声再生経路を、HTTP経由でブラウザ再生可能な構成へ拡張する。
- SSEで音声バイナリを直接送信するのではなく、WAVファイルのURLを通知し、React側で再生する設計を確立する。
- 外部デモで発生しうるCORS、MIME Type、ブラウザ自動再生制限、ファイル削除、書き込み途中のWAV参照といったハマりポイントを事前に潰す。
- 「外部デモ環境で実地確認したことで初めて分かった不足機能」を、アバターとしての実演性・実演性を高める実用機能として実装する。

## 2. システム環境

- **Node:** ローカルLLM検証環境
- **Voice Generation:** Java WebAPI / Tomcat コンテナ `Java WebAPIコンテナ`
- **TTS Backend:** style-bert-vits2
- **Voice Storage / HTTP Publish:** nginx コンテナ `音声公開用コンテナ`
- **Frontend:** React Dashboard / AvatarV2
- **SSE Stream:** `useLlmStream`
- **Voice Playback:** React側の音声再生制御コンポーネント
- **Target Directory:**
  - `[path masked]/voice/commons`
  - `[path masked]/voice/chat`
- **Public Voice URL:**
  - `http://[voice-server]/voice/...`

## 3. 作業ログ・解決プロセス

### 発生した課題

これまでジェムちゃんは、ローカルLLM検証環境上のJava WebAPIで音声合成を行い、そのWAVバイナリを音声再生サーバーへAPI経由で渡す構成だった。

自室内ではこの構成で問題なかったが、外部デモでは以下の問題が発生した。

- Reactアバター画面はVPN経由で表示できる。
- SSEによる字幕・表情制御も外部から確認できる。
- しかし、音声はローカル検証環境周辺の再生サーバーに依存しており、外部デモ環境ではジェムちゃんの声が聞こえない。
- 結果として、ジェムちゃんの「字幕・表情・声」のうち、声だけがデモ環境から欠落する。

この課題は、自室内だけで運用している段階では見えなかった。  
ジェムちゃんを外部デモ環境で実地確認したことで初めて、「React画面だけで音声再生まで完結する必要がある」と分かった。

### 初期検討

音声再生をReact側へ届ける方法として、以下を検討した。

1. SSEでWAVバイナリを直接送信する。
2. WAVファイルをサーバー上に保存し、そのURLをSSEで通知する。
3. OracleにWAVをBLOB格納し、ID指定で取得する。
4. nginxでWAV格納庫を用意し、ReactからHTTPで取得する。

検討の結果、MVPでは以下の理由から、**WAVファイルをHTTP公開し、SSEでは相対パスのみ送信する構成**を採用した。

- SSEはテキストイベント通知に使い、バイナリ配送には使わない方が安定する。
- Oracle BLOBはフィラーや定型音声のような再利用資産には向くが、通常会話の一時WAVには重い。
- nginxによる静的ファイル配信は単純で壊れにくい。
- React側は `audioPath` を受け取り、音声再生キューに積むだけでよい。
- 音声再生に失敗しても、字幕・表情表示を巻き込まない構成にできる。

### 解決策1: voice-publish-nginx nginxコンテナの構築

ローカルLLM検証環境上に、WAVファイル公開用のnginxコンテナ `voice-publish-nginx` を新規構築した。

ディレクトリ構成は以下。

```text
[path masked]/voice_hangar/
├── html/
│   └── voice/
│       ├── commons/
│       │   └── HalloGemChan.wav
│       └── chat/
├── nginx/
│   └── default.conf
├── run.sh
└── wavDelete.sh
```

`commons` には固定テスト用音声を配置し、`chat` には会話ごとに生成される一時WAVを格納する。

nginxコンテナは、WAVの格納庫そのものではなく、ホストディレクトリをHTTP公開するための「公開窓口」として扱う。

```bash
docker rm -f voice-publish-nginx

docker run -d \
  --restart=always \
  -p [public-port]:[container-port] \
  -v [path masked]/html:/usr/share/nginx/html:ro \
  -v [path masked]/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro \
  --name voice-publish-nginx \
  nginx
```

### 解決策2: CORSとMIME Typeの設定

初期状態ではWAVファイルの `Content-Type` が `application/octet-stream` になっていた。

```text
Content-Type: application/octet-stream
```

Reactダッシュボードと音声配信ポートは別Originになるため、CORS設定とMIME Type明示が必要と判断した。

nginx設定では、以下を追加した。

```nginx
server {
    listen [container-port];
    server_name _;

    root /usr/share/nginx/html;

    location /voice/ {
        types {
            audio/wav wav;
        }
        default_type application/octet-stream;

        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, OPTIONS" always;
        add_header Access-Control-Allow-Headers "*" always;

        if ($request_method = OPTIONS) {
            return 204;
        }

        try_files $uri =404;
    }
}
```

設定後、以下を確認した。

```text
HTTP/1.1 200 OK
Content-Type: audio/wav
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: *
```

これにより、React側から安全にWAVを参照できる準備が整った。

### 解決策3: React側の音声有効化ボタン

ブラウザには自動再生制限があるため、ユーザー操作なしで音声付きメディアを再生できない可能性がある。

そこで、Reactダッシュボード左下に `[ジェムちゃんの音声を有効化]` ボタンを追加した。

このボタンは、既存の右下リロードボタンと同じ三角形UIで実装し、押下時に以下の固定WAVを再生する。

```text
http://[voice-server]/voice/commons/HalloGemChan.wav
```

このPhaseで、以下を同時に確認できた。

* nginxからWAVを取得できる。
* CORSに阻まれない。
* MIME Typeが正しく返る。
* ブラウザで音声再生できる。
* ユーザー操作によって音声再生許可を得られる。

結果、React画面上で固定WAVの再生できることを確認した。

### 解決策4: Java WebAPIからのWAVファイル保存

音声合成を行っているTomcatコンテナ `Java WebAPIコンテナ` に、WAV出力先ディレクトリをbind mountした。

```bash
docker run -d \
  --name Java WebAPIコンテナ \
  --restart=always \
  -p [public-port]:[container-port] \
  -v $PWD/webapps:/usr/local/tomcat/webapps \
  -v [path masked]/voice/chat:/voice/chat \
  --add-host host.docker.internal:host-gateway \
  --log-opt max-size=1m --log-opt max-file=3 \
  tomcat:9.0.102-jre21
```

WAVファイル名は以下の形式とした。

```text
[ChatNo]-[SEQNo].wav
```

例:

```text
999-00001.wav
999-00002.wav
999-00003.wav
```

`SEQNo` はゼロ埋めすることで、ログやディレクトリ一覧で順序が崩れないようにした。

また、書き込み途中のWAVをReactが掴まないように、以下の手順を採用した。

```text
1. .tmp ファイルとして書き出す
2. 書き込み完了後に .wav へリネームする
3. リネーム完了後にSSEでaudioPathを送信する
```

この方式により、React側から見える `.wav` は常に完成済みファイルのみとなる。

### 解決策5: ファイル権限問題の解決

JavaからWAVファイルは出力できたものの、初期状態ではnginxから参照すると `403 Forbidden` が返った。

```text
HTTP/1.1 403 Forbidden
```

これは、Javaコンテナが作成したWAVファイルをnginxコンテナ側で読み取れないことが原因だった。

MVPでは以下の方針で対処した。

* MVP段階の暫定対応として、`chat` ディレクトリに広めの権限を設定
* Java側で生成後のWAVファイルにも読み取り可能な権限を設定
* 公開・本番相当の構成では、必要最小限の権限へ絞る前提とした

修正後、以下を確認した。

```text
HTTP/1.1 200 OK
Content-Type: audio/wav
Content-Length: 541740
Access-Control-Allow-Origin: *
Accept-Ranges: bytes
```

これにより、Javaで生成したWAVがnginx経由でHTTP公開されることを確認した。

### 解決策6: SSEへのaudioPath追加

SSEの `onIncomingChatMessage` に、音声ファイルの相対パス `audioPath` を追加した。

想定ペイロードは以下。

```json
{
  "type": "onIncomingChatMessage",
  "text": "……聞こえてる？",
  "emotion": "happy",
  "audioPath": "/voice/chat/12345-0001.wav"
}
```

React側では、既存の `useLlmStream` にて以下の方針で処理した。

* `text` は従来通り吹き出し表示へ渡す。
* `emotion` は従来通り表情制御へ渡す。
* `audioPath` は音声再生キューへ渡す。
* 音声再生エラーが発生しても、吹き出し表示や表情制御を巻き込まない。

これにより、字幕・表情・音声を同じSSEイベントから扱いつつ、失敗時の影響範囲は分離できた。

### 解決策7: React側の音声再生キュー

React側では、受信した `audioPath` を配列に保存し、順番に再生する構成とした。

目的は以下。

* 複数セリフが連続した場合でも、WAVを順番に再生する。
* `00001 → 00002 → 00003` の順序を維持する。
* 音声再生失敗時も次の音声へ進む。
* 将来的にリップシンクや吹き出し表示時間との同期へ拡張できるようにする。

今回はMVPとして、吹き出し表示と音声再生時間は完全同期させない判断とした。

理由は、音声再生経路には以下の失敗要因があるため。

```text
- 音声未有効化
- WAV未生成
- WAV削除済み
- nginx未到達
- CORS設定ミス
- ブラウザ再生拒否
- TTSコンテナ停止
```

したがって、まずは以下の安定性を優先した。

```text
字幕と表情は必ず出る。
音声は成功すれば鳴る。
音声失敗で画面表示を巻き込まない。
```

### 解決策8: 60分超WAVの削除バッチ

会話音声は一時ファイルであり、長期保存する必要はない。

そのため、60分以上経過したWAVを削除するスクリプトを作成した。

```bash
#!/bin/bash

echo "[$(date '+%Y-%m-%d %H:%M:%S')] wav delete start"

find [path masked]/voice/chat \
  -type f \
  -name "*.wav" \
  -mmin +60 \
  -print \
  -delete

echo "[$(date '+%Y-%m-%d %H:%M:%S')] wav delete end"
```

設置場所は以下。

```text
[path masked]/voice_hangar/wavDelete.sh
```

最初は「深夜2時に全削除」も検討したが、再生中や直前生成ファイルの削除事故を避けるため、`-mmin +60` を採用した。

cronにも登録し、定期的に古いWAVを削除する構成とした。

## 4. 成果物

### 4.1 音声配信アーキテクチャ

今回確立した全体構成は以下。

```text
Java WebAPI / Tomcat
  ↓
style-bert-vits2 で音声合成
  ↓
/voice/chat/[ChatNo]-[SEQNo].wav.tmp に保存
  ↓
保存完了後に .wav へrename
  ↓
nginx voice-publish-nginx がHTTP公開
  ↓
SSEで audioPath をReactへ送信
  ↓
React側で audioPath を絶対URL化
  ↓
音声再生キューへ追加
  ↓
ブラウザで順次再生
```

### 4.2 nginx公開URL

固定テスト音声。

```text
http://[voice-server]/voice/commons/HalloGemChan.wav
```

会話音声。

```text
http://[voice-server]/voice/chat/[ChatNo]-[SEQNo].wav
```

### 4.3 SSEペイロード

```json
{
  "type": "onIncomingChatMessage",
  "text": "……聞こえてる？",
  "emotion": "happy",
  "audioPath": "/voice/chat/12345-0001.wav"
}
```

### 4.4 ファイル命名規則

```text
[ChatNo]-[SEQNo].wav
```

例。

```text
482-00001.wav
482-00002.wav
482-00003.wav
```

### 4.5 削除バッチ

```bash
#!/bin/bash

echo "[$(date '+%Y-%m-%d %H:%M:%S')] wav delete start"

find [path masked]/voice/chat \
  -type f \
  -name "*.wav" \
  -mmin +60 \
  -print \
  -delete

echo "[$(date '+%Y-%m-%d %H:%M:%S')] wav delete end"
```

### 4.6 動作確認結果

以下を確認した。

1. Javaが生成したWAVを `curl -I` で `200 OK` 確認。
2. ReactのVoicePlayerへ `/voice/chat/482-00002.wav` を手動投入して再生。
3. SSEで `audioPath` を受信し、React側の音声再生キューへ追加。
4. 複数セリフ時に `00001 → 00002 → 00003` の順で再生。
5. style-bert-vits2コンテナを停止し、音声失敗時も吹き出し表示が壊れないことを確認。
6. 60分以上経過したWAVを削除バッチで削除できることを確認。
7. cron登録により、定期削除運用を開始。

## 5. 得られた設計知見

### 5.1 外部デモは隠れた要件を露出させる

自室内では、音声再生サーバーが近くに存在するため、React画面で音声再生する必要性に気付かなかった。

しかし、外部デモでは以下の差が顕在化した。

```text
自室:
  字幕・表情はReact
  声はローカル再生サーバー

外部デモ:
  React画面は見える
  声だけが届かない
```

この環境変化により、「声もReact画面側で完結する必要がある」と分かった。

### 5.2 SSEはバイナリ配送ではなく制御信号に使うべき

当初はSSEでWAVバイナリを送る案も考えた。

しかし、最終的には以下の分担が最も安定すると判断した。

```text
SSE:
  何が起きたかを通知する制御プレーン

HTTP/nginx:
  WAVファイルを配信するデータプレーン
```

この分離により、SSEの責務が明確になり、音声配信もブラウザ標準機能で扱えるようになった。

### 5.3 音声と吹き出しは最初から完全同期させない

理想的には、吹き出し表示時間と音声再生時間を同期させたい。

しかし、MVP段階ではあえて分離した。

理由は、音声再生にはブラウザ制限やHTTP取得失敗が絡むためである。

```text
吹き出し・表情:
  必ず表示する基本UX

音声:
  追加UX。失敗しても画面全体を壊さない
```

この分離により、音声失敗時でもデモ表示の最低品質を維持できる。

### 5.4 tmp書き込みとrenameは必須

WAV保存後すぐにReactへURL通知する場合、書き込み途中のファイルをブラウザが取得する可能性がある。

そのため、

```text
.tmp に書く
↓
保存完了
↓
.wav へrename
↓
SSE送信
```

という流れは必須である。

この一手により、再生不能な中途半端WAVをReactが掴むリスクを大幅に下げられる。

### 5.5 nginxは格納庫ではなく公開窓口

今回の設計では、WAVの実体はホストディレクトリにある。

nginxはそこを読み取り専用で公開するだけであり、書き込み責務は持たない。

```text
Java:
  WAVを生成して保存する

nginx:
  保存済みWAVをHTTP公開する

React:
  URLを受け取り再生する
```

この責任分離により、構成が単純で壊れにくくなった。

## 6. 最終結論

今回の作業により、ジェムちゃんは外部デモ環境においても、Reactダッシュボード上で以下を完結できるようになった。

```text
字幕
表情
声
```

これは単なる音声再生機能の追加ではない。

これまで「画面に表示されるアバター」と「部屋の中で鳴る声」に分かれていたアバターとしての実演性を、React画面上に統合する改修である。

外部デモでは、音声の有無がアバターの分かりやすさに大きく影響する。
字幕と表情だけでは「AIの画面表示」に見えるが、そこに声が加わることで、アバターの存在感を伝えやすくなる。

今回の成果物は、コード単体ではなく、以下の実用アーキテクチャである。

```text
ローカルTTSで生成した声を、
HTTP音声配信レイヤーを通して、
外部デモ用Reactアバターへ届ける仕組み。
```

自室では見えなかった不足が、外へ連れ出したことで露出した。
その不足を埋めたことで、ジェムちゃんの屋外デモにおける実演時の分かりやすさは向上した。

## 7. 今後の展望

* Oracleの会話履歴管理と連携し、ChatNo単位でWAV削除を行う。
* MVPでは `chmod 777` としているファイル権限を、将来的には `644` / `755` ベースに絞る。
* CORS設定の `Access-Control-Allow-Origin: *` を、デモ用ドメインまたはReactダッシュボードのOriginに限定する。
* 音声再生時間に応じて、吹き出し表示時間を制御する。
* 音声再生中の `isSpeaking` をリップシンクや口パク制御に統合する。
* フィラー音声は通常会話WAVとは分け、再利用可能な音声資産として管理する。
* Oracleメタデータ管理を導入し、`audioId`, `filePath`, `createdAt`, `expiresAt`, `status` を記録する。
* 外部デモ用に、固定セリフ・固定音声を事前生成するデモモードを用意する。