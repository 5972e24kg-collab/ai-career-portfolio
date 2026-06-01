# Work087: eGPU障害からのRTX3090専用Node D緊急復旧

Tag: [[Troubleshooting]], [[eGPU]], [[RTX3090]], [[CUDA]], [[Docker]], [[Ollama]], [[Ubuntu]], [[Node-D]], [[Disaster-Recovery]], [[Network]], [[Gem-chan]]

## 1. 目的

* Node B（EVO-X2）で発生したRTX3090 / eGPU認識障害の原因を切り分ける。
* RTX3090本体、EVO-X2側USB4ポート、USBケーブル、NVIDIAドライバ、eGPUドックのどこに問題があるかを特定する。
* ジェムちゃんのデモ環境を継続稼働させるため、RTX3090 / CUDA側の推論環境を復旧する。
* eGPU依存の構成から、RTX3090を直挿しする専用CUDAノード（Node D）へ暫定的に移行する。
* Node BのROCm / Director機能と、Node DのCUDA / Actor機能を分離し、複数ノード構成として再構成する。
* 復旧作業中に発生した物理インフラ、電源、ストレージ、Docker、監視API、NAT設定の課題を記録する。
* 今後の恒久対応として、Node Dの筐体・電源・冷却・サーバールーム収容方針を整理する。

## 2. システム環境

* **Node:** Node B / EVO-X2

  * **Hostname:** `[hostname masked]`
  * **IP:** `[IP address masked]`
  * **OS:** Ubuntu 24.04.3 LTS
  * **Kernel:** `6.8.0-117-generic`
  * **CPU/APU:** AMD Ryzen AI Max+ 395 / Strix Halo
  * **Role:** ROCm Main / Director / Open WebUI / RAG / 中心管理ノード
  * **Target Container:**

    * `ollama-rocm`
    * `open-webui`
  * **Removed Container:**

    * `ollama-cuda`

* **Node:** Node D

  * **Hostname:** `[hostname masked]`
  * **IP:** `[IP address masked]`
  * **OS:** Ubuntu
  * **CPU:** Intel Core i5-6600 / 4 cores / 4 threads
  * **Memory:** DDR4 32GB（在庫品 16GB x 2）
  * **GPU:** NVIDIA GeForce RTX 3090 / 24GB
  * **Role:** CUDA Sub / RTX3090専用推論・QLoRA実験ノード
  * **Target Container:**

    * `ollama-cuda`
    * NVIDIA GPU監視用WebAPI

* **旧構成**

  * Node BにRTX3090をeGPU経由で接続。
  * Node B上にROCm用 `ollama-rocm` とCUDA用 `ollama-cuda` を同居。
  * Open WebUIはNode B上で、ROCm / CUDA両方のOllamaを参照。

* **新構成**

  * Node BはROCm / Open WebUI / RAG / 中心管理に専念。
  * Node DはRTX3090直挿しのCUDA専用ノードとして独立。
  * Open WebUIは引き続きNode B側に集約。
  * Node BからNode DのOllama CUDAへLAN経由で接続。

## 3. 作業ログ・解決プロセス

### 発生した課題1: Node BがRTX3090を見失う

Node Bに接続していたeGPU RTX3090が、最近たびたび認識されなくなっていた。

当初はOS再起動で復帰していたため、一時的なUSB4 / PCIeリンク不調と考えていた。

しかし、今回の障害ではOS再起動後も以下の状態となった。

```text
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
NVRM: No NVIDIA GPU found.
```

`lspci` を確認したところ、AMD Strix Haloの内蔵GPUやUSB4 Host Routerは見えていたが、RTX3090 / NVIDIA / eGPU側PCIe Bridgeは表示されていなかった。

この時点で、NVIDIAドライバやDockerより下位の層、つまりPCIe列挙・USB4・eGPUドック・物理接続側の問題と判断した。

### 判断理由

`nvidia-smi` が失敗していても、`lspci` にNVIDIA GPUが存在していれば、NVIDIAドライバ、DKMS、Secure Boot、Container Toolkitを疑う。

しかし今回は、`lspci` 上にNVIDIAデバイス自体が存在しなかった。

そのため、ドライバ再インストールではなく、先に以下を疑う判断をした。

```text
- eGPUドック
- USB4 / Thunderboltリンク
- USBケーブル
- RTX3090補助電源
- eGPU側のPCIeブリッジ
- eGPU内部電源
```

### 解決策1: 放電再起動とUSB経路の切り分け

過去にデータセンターのストレージ障害を放電再起動で復旧させた経験があったため、今回も同様に試した。

実施内容。

```text
1. EVO-X2のシャットダウン
2. eGPU側の電源断
3. ACケーブル取り外し
4. USBケーブル抜線
5. 放電待機
6. 再接続
7. 起動確認
```

さらに、EVO-X2の背面・前面USB4ポートとUSBケーブルを組み合わせ、複数パターンで検証した。

USBケーブルについては、長さ違い、メーカー違い、チップ違いの可能性を考慮し、自宅内にあるケーブルを5種類程度かき集めて検証した。

結果、すべての組み合わせでRTX3090は認識しなかった。

### 発生した課題2: eGPUドックがWindows側でも認識されない

Node B側のUbuntu問題である可能性を排除するため、eGPUをWindows PCへ接続した。

確認箇所。

```text
- タスクマネージャー
- デバイスマネージャー
  - ディスプレイアダプター
  - ユニバーサル シリアル バス コントローラー
  - USB コネクトマネージャー
  - 不明なデバイス
```

結果、Windows側でもeGPUドックおよびRTX3090らしきデバイスは表示されなかった。

また、RTX3090を抜いた空のeGPUドックでも、Ubuntu / WindowsのどちらにもUSB4 / Thunderboltデバイスとして認識されなかった。

一方で、PD給電には反応していた。

### 判断理由

USB-C機器では、以下の機能は同一基板上にあっても独立した系統として動作する可能性がある。

```text
- USB-C PD給電交渉
- USB4 / Thunderboltデータ通信
- PCIe tunneling
- eGPU内部PCIeブリッジ
- GPU本体
```

PD給電が生きていることは、eGPU全体が正常であることを意味しない。

今回の状態は、PD系は生きているが、USB4 / Thunderbolt / PCIe tunneling系が死んでいるように見えた。

### 解決策2: RTX3090本体の生存確認

eGPUドックではなくRTX3090本体が故障している可能性を排除するため、自宅内にあったパーツをかき集めて検証用PCを一台構築した。

当初見つかったパーツはCeleron搭載機で、CPU性能としては頼りなかったが、RTX3090の生存確認には十分と判断した。

PCケースは小型向けで、RTX3090と電源がケース内に共存できなかったため、外装を分解し、ほぼオープンフレーム状態で仮組みした。

Ubuntu上で以下を確認した。

```bash
lspci | grep -E "VGA|3D|Display"
```

結果。

```text
01:00.0 VGA compatible controller: NVIDIA Corporation GA102 [GeForce RTX 3090] (rev a1)
```

さらに、RTX3090からのモニタ出力も確認できた。

これにより、RTX3090本体は生存していると判断した。

### 判断理由

RTX3090をPCIe直挿しした状態でOSからGA102として認識し、モニタ出力も可能であった。

そのため、以下の可能性は大きく低下した。

```text
- RTX3090本体故障
- RTX3090のVBIOS破損
- RTX3090の致命的な電源不良
```

一方で、以下の可能性が高まった。

```text
- 旧eGPUドック故障
- eGPUドックのUSB4 / Thunderboltデータ系故障
- eGPUドックのPCIeトンネル系故障
- eGPUドック内部電源の部分故障
```

### 発生した課題3: 新品eGPUドックも認識されない

Amazonで新品のeGPUドックを購入し、旧eGPUドックと交換して検証した。

しかし、新品の空eGPUドックをNode Bに接続しても、`boltctl` / `lsusb` / `lspci` に新規デバイスとして表示されなかった。

`journalctl -kf` でUSB4 / Thunderbolt / PCIe / UCSI / Type-C関連のイベントを監視しながら抜き差ししたが、ログは流れなかった。

確認コマンド。

```bash
sudo journalctl -kf | egrep -i 'thunderbolt|usb4|bolt|typec|ucsi|pcie|tunnel|usb'
```

同じEVO-X2のUSBポートにUSB外付けSSDを接続すると反応したため、少なくとも通常のUSBデータ通信は生きていると判断した。

その後、RTX3090を新品eGPUドックへ挿して検証したが、Node B / Windows PCのどちらでも反応しなかった。

### 判断理由

空ドックでは動作しない製品もあり得るため、RTX3090搭載状態でも検証した。

それでもLinux / Windowsの両方で反応しなかったため、以下を疑った。

```text
- 新品eGPUドックの初期不良
- 製品相性
- USB4 / Thunderboltデータリンクの未成立
```

ただし、複数ホストで無反応だったため、単なるEVO-X2との相性よりも初期不良疑いが強いと判断した。

結果として、新品eGPUドックは初期不良として返品する方針とした。

### 発生した課題4: eGPU復旧に固執するとデモ環境の復旧が遅れる

eGPUドックの交換・返品・再購入を繰り返すと、ジェムちゃんのデモ環境復旧が遅れる。

一方で、RTX3090本体は生きていることが確認済みである。

そのため、eGPU経由の復旧ではなく、RTX3090をPCIe直挿しする専用ノードを新設する方針へ切り替えた。

### 判断理由

eGPU構成は省スペースで便利だが、以下の不安定要素がある。

```text
- USB4 / Thunderboltコントローラ
- PCIe tunneling
- ケーブル相性
- 接続順
- ホットプラグ
- eGPUドック品質
- 電源・熱・物理接続
```

今回の障害により、ローカルLLM常用・イベント持ち出し・RTX3090級負荷にはeGPU構成のリスクがあると判断した。

RTX3090を直挿しするNode D構成の方が、物理的には大きくなるものの、PCIe接続としては安定すると考えた。

### 解決策3: Node Dの構築

Node Dは、過去に子供用PCとして使っていたPCの残骸を流用した。

このPCはWindows 11非対応のため、以前「使うな」と伝えて子供には新しいPCを与えていたものである。

構成。

```text
CPU: Intel Core i5-6600 / 4 cores / 4 threads
Memory: DDR4 32GB
GPU: RTX3090
OS: Ubuntu
IP: [IP address masked]
User: [user masked]
```

当初メモリは16GBだったが、DDR4メモリは自宅に多数在庫があったため、16GB x 2を探して32GBへ増設した。

OSはUbuntuへ変更した。

### 発生した課題5: PCケース内にRTX3090と電源が収まらない

RTX3090が大きすぎて、既存PCケース内で電源と干渉した。

追加で別のPCケースも探したが、こちらにも収まらなかった。

そのため、現在はケースから電源が飛び出した状態で稼働している。

### 解決策4: 暫定オープンフレーム運用

恒久対応ではないが、まずはデモ環境復旧を優先し、ケース外へ電源を逃がす形でNode Dを稼働させた。

現在はほぼオープンフレーム状態であり、RTX3090と電源は暫定配置で運用している。

冷却については、サーキュレーターで送風している。

### 未解決課題

現在のNode Dは、物理的には本番運用に向いた状態ではない。

残課題。

```text
- RTX3090を安全に固定できるケースが必要
- 電源ユニットをケース内または安全な位置に収容する必要がある
- PCIeスロットへの荷重を減らす必要がある
- 補助電源ケーブルの引っ張り・接触・短絡リスクを減らす必要がある
- サーキュレーターによる埃の巻き込みに注意が必要
```

### 発生した課題6: サーバールームにNode Dを格納するスペースがない

自宅サーバールームはウォークインクローゼットをリフォームしたものだが、収納棚の空きスペースには限界がある。

特に縦方向の余裕がなく、現在のNode D構成ではそのまま格納できない。

そのため、現在はサーバールーム内ではなく、仕事机とは別の作業机上で稼働している。

### 未解決課題

今後、Node Dをサーバールームに収めるためには、以下が必要である。

```text
- RTX3090対応かつ高さを抑えられるPCケースの選定
- サーバールーム内の棚配置見直し
- 1200VA/700W UPS配下への移設
- 排熱経路の確保
- 電源ケーブル・LANケーブルの整理
```

### 発生した課題7: 作業机のUPSではRTX3090を維持できない

作業机側にもUPSはあるが、仕様は500VA / 330Wであった。

Node D / RTX3090稼働中、RTX3090の消費電力スパイクに耐えられず、UPSが警告音を出して周辺機器を巻き込んで落ちた。

作業机周辺には低電力常駐サーバーも複数存在していたため、二次災害となった。

現在、Node DはUPSを通さずコンセント直結で電源を取っている。

### 判断理由

作業机側UPSは、RTX3090を含むGPUノードを支える容量ではない。

一方、サーバールーム側にはLLM専用の1200VA / 700W UPSがある。

したがって、最終的にはNode Dをサーバールーム内へ移設し、LLM専用UPS配下に収める必要がある。

### 暫定対策

現時点では、復旧優先のためNode Dをコンセント直結で稼働させている。

ただし、RTX3090は消費電力スパイクが大きいため、安定運用には電力制限を導入する価値がある。

候補。

```bash
sudo nvidia-smi -pm 1
sudo nvidia-smi -pl 250
nvidia-smi
```

これは未実施の恒久設定候補であり、性能より安定性・発熱・電源負荷抑制を優先する場合に有効である。

### 発生した課題8: Node DにNVIDIA Container Toolkitが入っていなかった

Node Dは新規に用意したUbuntu環境である。

そのため、DockerやNVIDIAドライバだけでなく、NVIDIA Container Toolkitも必要であった。

初期状態ではCUDAコンテナからGPUを利用できなかった。

### 解決策5: NVIDIA Container Toolkitのセットアップ

過去の手順書を参照し、Node DへNVIDIA Container Toolkitを導入した。

確認観点。

```bash
nvidia-smi
docker --version
docker compose version
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

これにより、DockerコンテナからRTX3090を利用できる状態にした。

### 発生した課題9: Node BからCUDAコンテナを除去する必要がある

旧構成では、Node B上に以下の2系統のOllamaが同居していた。

```text
ollama-rocm: Strix Halo / ROCm
ollama-cuda: RTX3090 / CUDA
```

RTX3090がNode Dへ移動したため、Node B上の`ollama-cuda`は不要となった。

### 解決策6: Node Bのdocker-compose.ymlをROCm専用構成へ変更

Node Bの`docker-compose.yml`から`ollama-cuda`セクションを削除した。

また、Open WebUIの接続先からNode B内の`ollama-cuda`を除外した。

修正前。

```yaml
- OLLAMA_BASE_URLS=http://ollama-rocm:[ollama-port];http://ollama-cuda:[ollama-port]
```

Node B単独復旧時。

```yaml
- OLLAMA_BASE_URLS=http://ollama-rocm:[ollama-port]
```

Node D接続後。

```yaml
- OLLAMA_BASE_URLS=http://ollama-rocm:[ollama-port];http://[node-d-host]:[cuda-host-port]
```

`depends_on`からも`ollama-cuda`を削除した。

修正前。

```yaml
depends_on:
  - ollama-rocm
  - ollama-cuda
```

修正後。

```yaml
depends_on:
  - ollama-rocm
```

反映コマンド。

```bash
docker compose up -d --remove-orphans
```

不要になったNode B側の`ollama-cuda`コンテナは停止・削除対象とした。

ただし、モデルデータ保護のため、Node B側の`ollama_cuda_data` volumeは即時削除しない方針とした。

### 判断理由

Open WebUIはNode Bに残す方針とした。

理由。

```text
- UIとRAGと中心管理をNode Bに集約できる
- Node DはCUDA専用ワーカーとして単純化できる
- Open WebUIの永続データを移行しなくてよい
- Node D障害時もNode BのROCm側は利用可能
```

### 発生した課題10: Node DにCUDA用Ollamaコンテナを立てる必要がある

Node Dでは、Open WebUIは立てず、CUDA用Ollamaだけを立てる方針とした。

Node B時代の`ollama-cuda`セクションを移植し、Node D専用の`docker-compose.yml`を作成した。

### 解決策7: Node D用docker-compose.ymlの作成

Node Dのディレクトリ。

```text
[ollama-compose-dir]
```

Node D用`docker-compose.yml`。

```yaml
services:
  # --- [Sub Brain] RTX 3090 (CUDA) ---
  ollama-cuda:
    image: ollama/ollama:0.20.2
    container_name: ollama-cuda
    ports:
      - "[host-cuda-port]:[container-ollama-port]" # Node D Host -> Container
    volumes:
      - ollama_cuda_data:/root/.ollama
    environment:
      - OLLAMA_HOST=[bind-address masked]
      - NVIDIA_VISIBLE_DEVICES=all
      - TZ=Asia/Tokyo
      - OLLAMA_KV_CACHE_TYPE=f16
      # - OLLAMA_KV_CACHE_TYPE=q8_0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: always

volumes:
  ollama_cuda_data:
```

起動。

```bash
cd ~/[ollama-compose-dir]
docker compose up -d
```

確認。

```bash
curl http://localhost:[cuda-host-port]/api/tags
```

Node Bからの確認。

```bash
curl http://[node-d-host]:[cuda-host-port]/api/tags
```

### 判断理由

Node Dのホスト側ポートは、CUDA用の専用ポートにした。

理由。

```text
- 旧Node B構成でCUDA側は専用ポートとして扱っていた
- 既存の認知負荷を下げられる
- Open WebUIや周辺スクリプトから見ても、「CUDA側の接続先」という意味が残る
```

### 発生した課題11: QLoRA環境をNode Dへ移行する必要がある

Node Bには、Gemma4 26B QLoRA環境が存在していた。

移行対象ディレクトリ。

```text
[qlora-project-dir]
```

この中には、以下が含まれていた。

```text
- docker-compose.yml
- [private config masked]
- scripts
- dataset
- dataset_prepared
- adapters
- offload
- output
- llama.cpp
- logs
- prompts
- unsloth_compiled_cache
```

Node Dでも同じパス構成で利用するため、SSH経由で転送した。

### 解決策8: rsyncによるQLoRA環境転送

Node B側で実行。

```bash
ssh [user]@[node-d-host] 'mkdir -p [qlora-project-dir]'
```

dry-run。

```bash
rsync -avhn --info=progress2 \
  [qlora-project-dir]/ \
  [user]@[node-d-host]:[qlora-project-dir]/
```

本実行。

```bash
rsync -avh --info=progress2 --partial \
  [qlora-project-dir]/ \
  [user]@[node-d-host]:[qlora-project-dir]/
```

確認。

```bash
ssh [user]@[node-d-host] '
cd [qlora-project-dir] &&
pwd &&
ls -al &&
du -sh .
'
```

### 判断理由

`rsync`を選択した理由。

```text
- 隠しファイルも含めて転送できる
- 実行権限を保持できる
- ディレクトリ構造を維持できる
- 再実行時に差分転送できる
- 途中中断時も再開しやすい
```

### 発生した課題12: ボリュームの空き容量が足りない

Node DのSSDは500GBだった。

しかし、Ubuntu初期インストール時に100GB程度しか割り当てられておらず、QLoRAディレクトリの転送後にOllamaへモデル展開できなかった。

### 解決策9: Ubuntu側のボリューム割り当て変更

Ubuntu側のボリューム割り当てを変更し、利用可能容量を拡張した。

これにより、Ollamaのモデル展開に必要な空き容量を確保した。

### 判断理由

QLoRA環境やOllamaモデルは、単一モデルでも数十GB単位の容量を消費する。

500GB SSDであっても、ルートボリュームが100GB程度では、以下の用途に耐えられない。

```text
- Ollama model store
- QLoRA dataset
- adapter
- merged model
- GGUF export
- logs
- Docker image / volume
```

そのため、ディスク全体をNode D用途へ適切に割り当てる必要があった。

### 発生した課題13: 監視用WebAPIがTomcat 404エラーになる

RTX3090の状態をReactダッシュボードへ表示するため、`nvidia-smi`の結果をパースする監視用WebAPIをNode BからNode DへSSHでコピーした。

設定を維持したままコンテナを立ち上げたが、Tomcatで404エラーとなった。

確認したところ、展開済みディレクトリの権限が以下になっていた。

```text
drwxr-x--- 2 [user] [group] 4096 4月 19 12:01 llmStatus-1.0
```

Other権限がないため、Docker内のTomcatプロセスがマウントされた展開済みディレクトリを読み取れない状態だった。

Tomcatは自動展開時、元のWARより既存の展開済みディレクトリを優先しようとする。

その結果、存在するが読めないディレクトリを参照し、WebAPIのロードに失敗していた。

### 解決策10: 展開済みディレクトリを削除し、Node D上で自動展開させる

権限問題のある展開済みディレクトリを削除した。

その後、Node D側でTomcatにWARを自動展開させた。

これにより、Tomcatコンテナ内から参照可能な状態でアプリケーションが展開され、404エラーは解消した。

### 判断理由

権限を無理に変更する方法もあったが、今回の問題は「展開済みディレクトリをコピーしたこと」が原因だった。

そのため、WARを正として扱い、Node D上でTomcat自身に再展開させる方が安全と判断した。

### 発生した課題14: デモ用アバターからNode D監視APIへ到達できない

デモ用アバターはVPN経由で自宅内の各APIへ通信する。

Node Dを新設したため、中継プロキシ2か所にNAT設定を追加する必要があった。

この時点で、アバターを動かすために必要な通信経路は11個目となった。

### 解決策11: Node D監視API用NAT設定の追加

Node Dの監視API用に、NAT ruleを追加した。

```text
set nat destination rule [rule-id masked] description '[description masked]'
set nat destination rule [rule-id masked] destination port '[external-port masked]'
set nat destination rule [rule-id masked] inbound-interface '[interface masked]'
set nat destination rule [rule-id masked] protocol 'tcp_udp'
set nat destination rule [rule-id masked] translation address '[node-d-ip masked]'
set nat destination rule [rule-id masked] translation port '[internal-api-port masked]'
```

これにより、デモ用アバターからNode D上の監視APIへ到達できるようにした。

### 判断理由

Reactアバターは、単にLLM応答を表示するだけではなく、GPU状態や内部ステータスも表示する。

Node DへRTX3090を移したため、RTX3090監視系もNode Dへ追従させる必要があった。

そのため、NAT経路の追加はデモ環境の復旧に必要な作業だった。

## 4. 成果物

### 4.1 Node B / ROCm専用docker-compose.yml方針

Node BではCUDAコンテナを削除し、ROCmとOpen WebUIを中心に再構成した。

```yaml
services:
  # --- [Main Brain] Strix Halo (ROCm) ---
  ollama-rocm:
    image: ollama/ollama:0.20.7-rocm
    container_name: ollama-rocm
    ports:
      - "[host-rocm-port]:[container-ollama-port]"
    volumes:
      - ollama_rocm_data:/root/.ollama
    environment:
      - OLLAMA_HOST=[bind-address masked]
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_MAX_LOADED_MODELS=2
      - OLLAMA_NUM_PARALLEL=1
      - HSA_ENABLE_SDMA=0
      - HSA_ENABLE_INTERRUPT=1
      - HIP_VISIBLE_DEVICES=0
      - TZ=Asia/Tokyo
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    restart: always

  # --- [UI] Open WebUI ---
  open-webui:
    image: ghcr.io/open-webui/open-webui:v0.6.36
    container_name: open-webui
    ports:
      - "[host-webui-port]:[container-webui-port]"
    environment:
      - OLLAMA_BASE_URLS=http://ollama-rocm:[ollama-port];http://[node-d-host]:[cuda-host-port]
      - CORS_ALLOW_ORIGIN=*
      - WEBUI_URL=http://[node-b-host]:[webui-port]
      - TZ=Asia/Tokyo
    volumes:
      - open-webui_data:/app/backend/data
    depends_on:
      - ollama-rocm
    restart: always

volumes:
  ollama_rocm_data:
  open-webui_data:
```

### 4.2 Node D / CUDA専用docker-compose.yml

Node DではOpen WebUIを立てず、RTX3090用のOllama CUDAのみを稼働させる。

```yaml
services:
  # --- [Sub Brain] RTX 3090 (CUDA) ---
  ollama-cuda:
    image: ollama/ollama:0.20.2
    container_name: ollama-cuda
    ports:
      - "[host-cuda-port]:[container-ollama-port]"
    volumes:
      - ollama_cuda_data:/root/.ollama
    environment:
      - OLLAMA_HOST=[bind-address masked]
      - NVIDIA_VISIBLE_DEVICES=all
      - TZ=Asia/Tokyo
      - OLLAMA_KV_CACHE_TYPE=f16
      # - OLLAMA_KV_CACHE_TYPE=q8_0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: always

volumes:
  ollama_cuda_data:
```

### 4.3 QLoRA環境転送コマンド

```bash
ssh [user]@[node-d-host] 'mkdir -p [qlora-project-dir]'
```

```bash
rsync -avhn --info=progress2 \
  [qlora-project-dir]/ \
  [user]@[node-d-host]:[qlora-project-dir]/
```

```bash
rsync -avh --info=progress2 --partial \
  [qlora-project-dir]/ \
  [user]@[node-d-host]:[qlora-project-dir]/
```

### 4.4 Node D監視API用NAT設定

```text
set nat destination rule [rule-id masked] description '[description masked]'
set nat destination rule [rule-id masked] destination port '[external-port masked]'
set nat destination rule [rule-id masked] inbound-interface '[interface masked]'
set nat destination rule [rule-id masked] protocol 'tcp_udp'
set nat destination rule [rule-id masked] translation address '[node-d-ip masked]'
set nat destination rule [rule-id masked] translation port '[internal-api-port masked]'
```

### 4.5 動作確認結果

以下を確認した。

```text
- RTX3090がNode D上で認識される
- Node D上でCUDA / NVIDIA Container Toolkitが動作する
- Node D上でollama-cudaコンテナが起動する
- Node BのOpen WebUIからNode DのCUDA Ollamaへ接続できる
- Node BからCUDAコンテナを除去してもROCm側は継続稼働する
- QLoRA環境をNode Dへ転送できる
- ストレージ割り当て変更後、Ollamaモデル展開が可能になる
- 監視用WebAPIのTomcat 404を解消できる
- NAT追加により、デモ用アバターからNode D監視APIへ到達できる
- ジェムちゃんからの応答を得られる
```

## 5. 得られた設計知見

### 5.1 `nvidia-smi`の失敗だけでドライバ障害と決めない

`nvidia-smi`が失敗しても、最初に見るべきは`lspci`である。

```text
lspciにNVIDIAがいる:
  ドライバ / DKMS / Secure Boot / Container Toolkitを疑う

lspciにNVIDIAがいない:
  PCIe / USB4 / eGPU / 電源 / 物理接続を疑う
```

今回、`lspci`にNVIDIAがいなかったため、ドライバ再インストールではなく物理層の切り分けを優先できた。

### 5.2 PD給電はデータ経路の正常性を意味しない

eGPUドックでPD給電が反応していても、USB4 / Thunderbolt / PCIe tunnelingが正常とは限らない。

PD管理系だけが生きていて、eGPUとしてのデータ経路が死んでいる状態はあり得る。

### 5.3 eGPUは便利だが、RTX3090常用には不安定要素が多い

eGPUは省スペースで便利だが、以下の層が追加される。

```text
GPU
↓
PCIeスロット
↓
eGPU基板
↓
PCIe Bridge
↓
Thunderbolt / USB4 Controller
↓
USB-C Cable
↓
Host USB4 Controller
↓
OS
```

直挿しPCIe構成と比べると、障害点が多い。

RTX3090級のGPUをローカルLLM用に常用する場合、専用ノード化は十分に合理的である。

### 5.4 原因追及とサービス復旧は分けて考える

eGPU障害の原因追及を続けることと、ジェムちゃんデモ環境を復旧させることは別問題である。

今回は、RTX3090本体の生存確認後、eGPU復旧に固執せずNode D構築へ切り替えた。

これにより、サービス復旧を優先できた。

### 5.5 家庭内在庫パーツは災害復旧資産になる

今回、子供用PCの残骸、DDR4メモリ在庫、余剰PCケース、各種ケーブルを活用した。

結果として、追加費用0円でNode Dを構築し、RTX3090を復旧できた。

ただし、在庫パーツによる復旧は物理配置や安全性が暫定になりやすい。

### 5.6 WebAPI移行では展開済みディレクトリの権限に注意する

TomcatアプリをNode間コピーする場合、WARだけでなく展開済みディレクトリをコピーしてしまうと、権限差で404やロード失敗が発生する。

今回のようにDocker内Tomcatからホストマウントを読む場合は、以下の方針が安全である。

```text
- WARを正とする
- 展開済みディレクトリはコピーしない
- 新ノード側でTomcatに自動展開させる
```

### 5.7 デモ環境の通信経路は増殖する

Node D追加により、アバターに必要な通信経路は11個目となった。

今後は、NAT、VPN、プロキシ、API、ポート、宛先ノードの一覧化が必要である。

## 6. 未解決の課題

### 6.1 Node Dの物理収容

現在、Node Dはケースから電源が飛び出した暫定構成である。

恒久運用には、RTX3090と電源を安全に収められるケースが必要である。

### 6.2 サーバールームへの移設

現在、Node Dは作業机上に展開されている。

最終的にはサーバールームへ移設し、LLM専用UPS配下へ収容する必要がある。

### 6.3 UPS容量

作業机UPS 500VA / 330WではRTX3090を維持できなかった。

RTX3090ノードは、1200VA / 700W UPS配下に移す必要がある。

### 6.4 電力制限

RTX3090の消費電力スパイクを抑えるため、`nvidia-smi -pl`による電力制限を検討する。

候補。

```bash
sudo nvidia-smi -pm 1
sudo nvidia-smi -pl 250
```

systemdで起動時に自動適用することも検討する。

### 6.5 冷却と埃対策

現在はサーキュレーターで送風している。

オープンフレーム状態は冷却面では有利な面もあるが、埃・接触・短絡リスクがある。

### 6.6 eGPUドックの返品

新品eGPUドックは初期不良疑いとして返品する。

旧eGPUドックについては、PD系は生きているが、USB4 / Thunderbolt / PCIeデータ系が故障している可能性が高い。

### 6.7 Node A〜D全体構成の棚卸し

LLM専用機材がNode A〜Dまで増加し、API群やネットワークを支える仮想ホストも含めると、ホームラボ全体がかなり複雑化している。

今後は、以下を整理する必要がある。

```text
- Node一覧
- IP一覧
- 役割一覧
- Docker Compose配置
- API一覧
- NATルール一覧
- 監視経路
- UPS収容状況
- 障害時の復旧手順
```

## 7. 最終結論

今回の作業により、Node Bで発生したeGPU / RTX3090認識障害から、ジェムちゃんのデモ環境を復旧できた。

最終的な原因は、RTX3090本体ではなく、旧eGPUドックのUSB4 / Thunderbolt / PCIe系故障が濃厚である。

新品eGPUドックもLinux / Windows双方で認識されなかったため、初期不良疑いとして返品方針となった。

一方で、RTX3090本体はPCIe直挿しで正常動作を確認できた。

そのため、eGPU経由での復旧を諦め、旧子供用PCをベースにNode Dを緊急構築した。

Node Dは、i5-6600 / DDR4 32GB / RTX3090 / Ubuntuの構成で、CUDA専用ノードとして稼働している。

Node BはROCm / Open WebUI / RAG / 中心管理ノードとして残し、Node DはRTX3090 / CUDA / QLoRA / 監視APIを担当する構成へ分離した。

これにより、結果的に以下の構成へ進化した。

```text
Node B:
  EVO-X2 / Strix Halo / ROCm
  Director
  Open WebUI
  RAG
  中心管理

Node D:
  i5-6600 / RTX3090 / CUDA
  Actor
  QLoRA
  NVIDIA監視API
```

今回の復旧作業により、デモ環境として必要な応答経路は回復できた。

ジェムちゃんからの応答確認まで実施できており、デモ環境は継続利用可能な状態まで回復した。

ただし、物理インフラ面では未解決課題が残っている。

```text
- ケース収容
- 電源固定
- UPS配下への移設
- 冷却
- 埃対策
- ケーブル整理
- NAT / API経路の棚卸し
```

現在のNode Dは、災害復旧モードのまま本番稼働している状態である。

短期的にはこのまま維持しつつ、中期的には安全な筐体・電源・収納・UPS構成へ移行する必要がある。

今回の作業は、単なるハードウェア交換ではない。

eGPU障害をきっかけに、RTX3090を外付け補助GPUから独立したCUDAノードへ昇格させ、ホームLLM基盤をNode B / Node Dの分散構成へ再編成した復旧作業である。

結果として、我が家のLLM基盤はまた一段階育った。

ただし、その成長に合わせて、物理インフラと運用管理も追いつかせる必要がある。
