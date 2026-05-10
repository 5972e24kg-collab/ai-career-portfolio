# Work078: MCP Tool Use検証におけるROCm GPU HangとMCP設計境界の発見

Tag: [[MCP]], [[ToolUse]], [[ROCm]], [[Ollama]], [[OpenWebUI]], [[gpt-oss]], [[Portainer]], [[Troubleshooting]], [[Architecture]]

## 1. 目的

- 自作 Portainer Ops MCP Server を Open WebUI から呼び出し、LLM が Docker / Portainer 系の外部情報を tool-use で取得・解釈できるか検証する
- `gpt-oss:120b` に対して、MCP tool 使用マニュアルを付与することで tool 選択と回答品質が安定するか確認する
- `environments` tool と `containers` tool の挙動差を調査し、MCPサーバー側の実装問題か、Open WebUI / Ollama / ROCm 側の推論基盤問題かを切り分ける
- ローカルLLM環境における MCP tool-use の実用上の限界を確認する
- MCP を Docker監視・構造化データ参照用途に使う設計が妥当か、実験結果から再評価する

## 2. システム環境

- **Node:** Node B (EVO-X2 / Ubuntu 24.04 Native)
- **Main GPU:** AMD Radeon 8060S / Strix Halo / ROCm / UMA VRAM 約96GB以上割当
- **Sub GPU:** RTX 3090 / CUDA / `ollama-cuda`
- **Target LLM:** `gpt-oss:120b`
- **Comparison Model:** `gemma4:e4b`
- **Runtime:** `ollama/ollama:0.20.7-rocm`
- **UI:** Open WebUI `v0.6.36`
- **MCP Server:** 自作 Portainer Ops MCP Java Server
- **MCP Transport:** Streamable HTTP MCP
- **Relevant Container:** `ollama-rocm`
- **Primary Tool Under Test:** `containers(environmentName)`
- **Comparison Tool:** `environments`
- **Test Environment Name:** `[docker-env-name masked]`

## 3. 作業ログ・解決プロセス

### 発生した課題

Open WebUI 上で `gpt-oss:120b` に対し、以下のように依頼した。

```text
[docker-env-name masked]にあるコンテナの一覧を教えて
```

このとき、MCP server の `containers` tool は呼び出されるが、最終的な回答生成時に以下のエラーが発生した。

```text
500: model runner has unexpectedly stopped,
this may be due to resource limitations or an internal error,
check ollama server logs for details
```

一方で、以下のような `environments` tool を使う問い合わせは成功した。

```text
ホストの一覧を教えて
```

そのため、初期仮説としては以下を疑った。

* `containers` tool の戻り値が大きすぎる
* `containers` tool の JSON 構造が複雑すぎる
* MCPサーバー側で例外が発生している
* Open WebUI が tool result を LLM に再投入する過程で破綻している
* `gpt-oss:120b` + ROCm + Ollama runner の組み合わせで GPU Hang が発生している

### 調査1: MCPサーバー側の正常性確認

MCPサーバーのログを確認したところ、`containers` tool は正常に呼び出されていた。

```text
method=tools/call, id=2
tools/call toolName = containers
environmentName = [docker-env-name masked]
```

PowerShell から MCP API を直接叩いた場合も、HTTP 200 で正常にレスポンスが返った。

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"containers\":[...]}"
      }
    ],
    "isError": false
  }
}
```

この時点で、MCPサーバー自体の例外や JSON-RPC 層の失敗は主因から外れた。

### 調査2: `containers` tool の戻り値サイズ確認

`containers-response.json` を保存してサイズを確認した。

```text
File Size: 10613 bytes
```

別途分析した結果は以下。

| 対象                       |      サイズ |
| ------------------------ | -------: |
| JSON-RPC外側ラッパー込み         | 10.35 KB |
| `result.content[0].text` |  8.87 KB |
| 10コンテナの実データ合計            |  9.54 KB |

コンテナ件数は10件であり、各コンテナのトップレベルフィールド数も16程度だった。

主なフィールドは以下。

```text
environmentId
environmentName
hostName
containerId
containerName
image
state
status
networkMode
ports
mounts
compose
riskFlags
createdAt
stateDetail
stats
```

この結果から、以下の仮説は弱まった。

```text
tool result が巨大すぎるため落ちている
```

10KB程度の JSON で `gpt-oss:120b` が通常の意味で処理不能になるとは考えにくい。

### 調査3: Ollama / ROCm ログ確認

`ollama-rocm` のログを確認したところ、失敗時には以下のような GPU Hang が発生していた。

```text
HW Exception by GPU node-1 reason :GPU Hang
post predict error="Post \"http://127.0.0.1:<port>/completion\": EOF"
[GIN] ... 500 ... POST "/api/chat"
```

別ケースでは、以下も確認した。

```text
Memory access fault by GPU node-1
Reason: Page not present or supervisor privilege.
```

さらに、runner が abort していた。

```text
llama runner process no longer running
signal: aborted (core dumped)
```

このため、問題は MCP server や Open WebUI の表層ではなく、Ollama runner / ROCm backend / GPU実行層にある可能性が高まった。

### 調査4: `num_ctx` 変更による切り分け

`num_ctx` を変更しながら、同一の `containers` tool-use を検証した。

#### 131072

```text
gpt-oss:120b
SIZE: 70 GB
PROCESSOR: 100% GPU
CONTEXT: 131072
```

結果: 失敗

```text
HW Exception by GPU node-1 reason :GPU Hang
post predict EOF
/api/chat 500
```

#### 32768

```text
gpt-oss:120b
SIZE: 66 GB
PROCESSOR: 100% GPU
CONTEXT: 32768
```

結果: 失敗

```text
HW Exception by GPU node-1 reason :GPU Hang
post predict EOF
/api/chat 500
```

#### 16384

初回は成功したため、一時的に「16384以下なら安定するのでは」と考えた。

しかし追加で以下の再現確認を行った。

| 条件                                   | 結果          |
| ------------------------------------ | ----------- |
| 毎回 `docker restart ollama-rocm` して5回 | 失敗5回        |
| 毎回 `ollama stop gpt-oss:120b` して5回   | 成功1回 / 失敗4回 |
| モデルロード済みで Open WebUI 新規チャット5回        | 成功1回 / 失敗4回 |

この結果により、`16384` は安全圏ではなく、初回成功は偶然である可能性が高いと判断した。

#### 8192

さらに `8192` でも検証した。

| 条件                                   | 結果          |
| ------------------------------------ | ----------- |
| 毎回 `docker restart ollama-rocm` して5回 | 成功2回 / 失敗3回 |
| 毎回 `ollama stop gpt-oss:120b` して5回   | 成功2回 / 失敗3回 |
| モデルロード済みで Open WebUI 新規チャット5回        | 成功3回 / 失敗2回 |

`8192` では入力が切り詰められるログも出た。

```text
truncating input prompt limit=8192 prompt=9537 keep=4 new=8192
```

それでも失敗が混在したため、問題は単純な `num_ctx` 閾値ではないと判断した。

### 調査5: モデル差の確認

比較対象として `gemma4:e4b` を使用し、同じ MCP tool-use を検証した。

| 条件                               | 結果          |
| -------------------------------- | ----------- |
| 毎回 `docker restart` して5回         | 成功5回        |
| 毎回 `ollama stop gemma4:e4b` して5回 | 成功4回 / 失敗1回 |
| モデルロード済みで Open WebUI 新規チャット5回    | 成功5回        |

また、本検証範囲では、CUDA + `gemma4:e4b` で同種の runner crash / GPU Hang は確認されなかった。

ただし、`gemma4:e4b` には別の問題があった。

* tool 選択が安定しない
* tool は呼べても、回答内容がハルシネーションする
* `gpt-oss:120b` に比べて、tool result の解釈品質が低い

この結果により、問題は以下の2軸に分離された。

```text
A. 推論基盤として安定して呼べるか
B. LLMエージェントとして正しくtoolを選び、正しく回答できるか
```

本検証範囲では、`gpt-oss:120b + ROCm` は、Bは強い一方でAに不安定さが見られた。
本検証範囲では、`gemma4:e4b + ROCm/CUDA` は、Aは比較的安定する一方でBに課題が見られた。

### 調査6: FlashAttention の扱い

ログ上、Ollama server config では以下のように見えた。

```text
OLLAMA_FLASH_ATTENTION:false
```

しかし runner の load request では毎回以下になっていた。

```text
FlashAttention:Enabled
```

このため、環境変数で FlashAttention を無効化したつもりでも、実際の runner では有効になっている可能性がある。

ただし、FlashAttention 無効化は他の検証や通常運用への副作用が大きいため、今回は深追いしない判断とした。

## 4. 解決策・判断

### 4.1 技術的な暫定結論

今回の問題は、以下の組み合わせで発生している可能性が高い。

```text
gpt-oss:120b
+ Ollama 0.20.7 ROCm runner
+ Radeon 8060S / gfx1151
+ FlashAttention enabled
+ Open WebUI MCP tool-use後の再推論
```

特に重要なのは、落ちている箇所が MCPサーバー呼び出し時ではなく、tool result を Open WebUI が LLM へ戻した後の再推論時である点。

推定される処理フローは以下。

```text
1. ユーザーが質問する
   「[docker-env-name masked]にあるコンテナの一覧を教えて」

2. Open WebUI が gpt-oss:120b に1回目の推論を投げる

3. gpt-oss:120b が containers tool を選択する

4. MCPサーバーが containers tool を正常実行する

5. Open WebUI が tool result を会話コンテキストへ追加する

6. Open WebUI が gpt-oss:120b に2回目の推論を投げる

7. Ollama ROCm runner が GPU Hang / Memory access fault を起こす

8. /completion が EOF となり、/api/chat が 500 を返す
```

つまり、MCP server の実装エラーではなく、tool-use後の再推論がローカル推論基盤にとって高リスクであることが分かった。

### 4.2 真因特定について

運用上の真因は以下まで特定できた。

```text
gpt-oss:120b + ROCm + Ollama runner において、
Open WebUI tool-use 後の再推論で GPU Hang / Memory access fault が発生する。
```

ただし、ソースコードレベルの真因、すなわち、

* Ollama runner の不具合
* ggml HIP backend の不具合
* ROCm driver の不具合
* gfx1151 / Radeon 8060S 対応の成熟度問題
* FlashAttention kernel の問題
* gpt-oss:120b / MXFP4 との相性問題

のどれが直接原因かまでは、個人研究の範囲で追い切るにはコストが大きすぎると判断した。

### 4.3 Portainer Ops MCP Server 深追い停止

当初は、Docker / Portainer 情報を MCP tool として LLM に渡し、監視や異常判断に使うことを想定していた。

しかし、今回の検証を通じて、以下の結論に至った。

```text
少なくとも本検証で扱ったローカルLLM環境では、MCPを構造化データ大量投入用のデータプレーンとして扱うのはリスクが高い。
MCPは、LLMが外部世界へ小さく安全に作用するための制御プレーンとして設計する方が適している。
```

Docker監視やコンテナ一覧のような情報は、横方向の項目数は削減できても、縦方向の件数は意図せず増える。
そのため、MCP tool result の総量制御には限界がある。

ページングも可能ではあるが、LLM側の制御が複雑になり、ローカルLLMでは安定運用が難しい。

よって、Portainer Ops MCP Server の深追いは停止する。

### 4.4 今後の方向転換

Portainer / Docker 監視は、MCPではなく、従来通り Java / ルールベースで構築する方が妥当と判断した。

LLMを使う場合でも、役割は以下に限定する。

```text
- 監視結果の説明
- 異常の要約
- 次に見るべき観点の提案
- 人間向け報告文の生成
- 必要な操作候補の提示
```

一方で、MCPの次の題材として SwitchBot API を採用する方針に切り替える。

理由は、SwitchBot操作はMCPの本質に合っているため。

```text
ユーザー: ジェムちゃん、照明付けて

LLM: turnOnDevice(deviceAlias="照明") を呼ぶ

MCP: SwitchBot API を実行する

Tool result:
{"status":"success"}

Actor:
「まったく、手くらい伸ばしなさいよ……まあ、付けてあげたけど」
```

このような操作系MCPは、戻り値が短く、LLMに大量の構造化データを読ませる必要がない。

## 5. 成果物

今回の成果物は、コードやモデルではなく、以下の設計知見である。

### 5.1 得られた技術知見

```text
「データ量が超巨大だから駄目」ではなく、
ローカル推論基盤では、tool-use後の再推論そのものがかなり繊細である。
```

今回の `containers-response.json` は約10KBであり、一般的な意味では巨大ではない。
それでも `gpt-oss:120b + ROCm + Ollama + Open WebUI tool-use` では、8192 context でも失敗が混在した。

したがって、問題は単純なデータサイズではなく、以下の複合要因である。

```text
- tool-useにより推論が2段階になる
- tool result が会話コンテキストへ再投入される
- ローカルLLMではコンテキスト処理・JSON解釈・runner安定性がモデル依存になる
- ROCm backend では GPU Hang / Memory access fault が発生しうる
- 120B級モデルでは、軽微な入力差や実行状態差が失敗率に影響する
```

### 5.2 MCP設計原則

今回の実験から、ローカルLLM向けMCP設計原則を以下のように整理した。

```text
1. MCPはデータプレーンではなく制御プレーンとして扱う

2. LLMに外部システムの生データを丸ごと読ませない

3. tool result は短く、目的別に、意味のある単位へ圧縮する

4. 監視・判定・集計はルールベース側で行う

5. LLMには説明・判断補助・自然言語応答・操作選択を担当させる

6. ローカルLLMでは、tool-use後の再推論負荷を常に疑う

7. 大きいDTOより、小さい意図単位toolを複数用意する
```

### 5.3 悪いMCP tool設計例

```text
containers(environmentName)
```

全コンテナについて以下を返す。

```text
containerId
image
state
status
networkMode
ports
mounts
compose
riskFlags
createdAt
stateDetail
stats
```

これは人間やプログラムにとっては便利だが、LLM tool result としては重い。
また、コンテナ数が増えると縦方向に無制限に増える。

### 5.4 良いMCP tool設計例

```text
containerNames(environmentName)
unhealthyContainers(environmentName)
containerRiskSummary(environmentName)
exposedPorts(environmentName)
dockerHostHealth(environmentName)
restartContainer(environmentName, containerName)
```

戻り値例。

```json
{
  "environmentName": "[docker-env-name masked]",
  "containerCount": 10,
  "containers": [
    "[container 01 masked]",
    "[container 02 masked]",
    "[container 03 masked]",
    "[container 04 masked]",
    "[container 05 masked]",
    "[container 06 masked]",
    "[container 07 masked]",
    "[container 08 masked]",
    "[container 09 masked]",
    "[container 10 masked]"
  ]
}
```

リスク要約の場合。

```json
{
  "environmentName": "[docker-env-name masked]",
  "riskSummary": [
    {
      "containerName": "[container 01 masked]",
      "riskFlags": ["PUBLIC_PORT_EXPOSED", "DOCKER_SOCK_MOUNTED"]
    },
    {
      "containerName": "[container 03 masked]",
      "riskFlags": ["PUBLIC_PORT_EXPOSED", "HOST_ROOT_MOUNTED"]
    }
  ]
}
```

操作系の場合。

```json
{
  "action": "restartContainer",
  "target": "[container 07 masked]",
  "status": "success"
}
```

### 5.5 SwitchBot MCP 方針

次の検証対象は SwitchBot API MCP とする。

想定tool。

```text
listSwitchBotDevices()
getDeviceStatus(deviceName)
turnOnDevice(deviceName)
turnOffDevice(deviceName)
setAirConditionerTemperature(deviceName, temperature)
```

戻り値例。

```json
{
  "action": "turnOnDevice",
  "target": "リビング照明",
  "status": "success"
}
```

ジェムちゃん統合時の期待動作。

```text
ユーザー:
ジェムちゃん、照明付けて

Director:
照明操作が必要。turnOnDevice を呼ぶ。

MCP:
SwitchBot API を実行し、status=success を返す。

Actor:
まったく、手くらい伸ばしなさいよ……まあ、付けてあげたけど。
```

この構成では、LLMは外部世界の大量データを読むのではなく、短い操作結果だけを受け取る。
そのため、MCP本来の用途に近い。

## 6. 最終結論

今回の検証により、以下を確認した。

* 自作 Portainer Ops MCP Server の `containers` tool 自体は正常に動作していた
* `containers` のレスポンスは約10KBであり、通常の意味で巨大ではなかった
* Open WebUI 上で `gpt-oss:120b` が tool-use 後に再推論する段階で、Ollama ROCm runner が GPU Hang / Memory access fault を起こしていた
* `num_ctx=131072 / 32768 / 16384 / 8192` のいずれでも、程度差はあるが `gpt-oss:120b + ROCm` では失敗が混在した
* `gemma4:e4b` では同じ MCP 呼び出しの実行安定性は高かったが、tool選択と回答品質に課題が残った
* 本検証範囲では、CUDA + `gemma4:e4b` で同種の runner crash は確認されなかった
* したがって、問題は MCP server ではなく、ローカルLLM推論基盤、特に `gpt-oss:120b + ROCm + Ollama runner + tool-use後再推論` の組み合わせにあると判断した
* Portainer / Docker のような構造化データ取得型MCPは、ローカルLLMでは設計上のリスクが大きい
* 少なくとも本検証で扱ったローカルLLM環境では、MCPを外部システムの生データを大量にLLMへ渡す仕組みとして扱うより、LLMが外部世界へ小さく安全に作用するための制御プレーンとして設計する方が適している

今回の成果物は、実装物ではなく、以下の実戦的な設計知見である。

```text
ローカルLLMにおけるMCP tool-useでは、
toolが正しく動いても、
tool-use後の再推論が安定するとは限らない。

そのため、MCP tool result は短く、目的別に、LLMが扱いやすい形へ圧縮すべきである。

特にローカルLLMでは、
MCPを「構造化データ取得口」として大きく使うより、
「外部操作の安全な窓口」として小さく使う方が適している。
```

## 7. 今後の展望

* Portainer Ops MCP Server の深追いは停止する
* Docker監視はルールベースの Java / 既存監視基盤で実装する
* MCPとして残す場合は、`containerNames` や `containerRiskSummary` のような軽量・目的別toolへ再設計する
* 次のMCP検証対象は SwitchBot API とする
* ジェムちゃんチャットボットに SwitchBot MCP を統合し、自然言語による家電制御の実現を目指す
* Director / Actor 構成では、Director が操作判断、MCP が実行、Actor が人格応答を担当する設計にする
* 今後のMCP設計では、「戻り値の短さ」「操作単位の明確さ」「LLM再推論負荷の低さ」を最優先にする