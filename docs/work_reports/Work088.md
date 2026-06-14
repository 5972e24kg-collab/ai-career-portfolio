# Work088: GIGABYTE AI TOP ATOM初期セットアップとクリーンバックアップ取得

Tag: [[GIGABYTE-AI-TOP-ATOM]], [[DGX-Spark]], [[GB10]], [[Ubuntu]], [[ARM64]], [[CUDA]], [[Docker]], [[Backup]], [[Disaster-Recovery]], [[QLoRA]], [[Gemma4]], [[HomeLab]], [[Gem-chan]]

## 1. 目的

* 新規導入した GIGABYTE AI TOP ATOM の初期状態を確認する。
* ARM64 / GB10 / CUDA 13.0 / Docker GPU 実行環境が正常に動作するか確認する。
* 今後の gemma4:31b QLoRA 検証、およびジェムちゃん本番基盤移行に入る前に、クリーンな初期状態を保全する。
* Rescuezilla / Clonezilla 系が ARM64 環境では使いにくい可能性を踏まえ、代替バックアップ手順を確立する。
* 起動中OSからのファイル単位バックアップと、Live Ubuntu からのディスク丸ごとイメージ取得の二段構えで、復旧可能性を高める。
* 取得したバックアップを USB NVMe、自作PC共有フォルダ、QNAP NAS（HDD RAID1）の3か所に保存し、初期状態の保全性を高める。

## 2. システム環境

* **Node:** AI TOP ATOM
* **Hostname:** `[hostname masked]`
* **Hardware Vendor:** GIGABYTE
* **Hardware Model:** AI TOP ATOM
* **Architecture:** `arm64 / aarch64`
* **CPU/GPU:** NVIDIA GB10
* **Memory:** 121GiB recognized
* **OS:** Ubuntu 24.04.4 LTS
* **Kernel:** `Linux 6.17.0-1021-nvidia`
* **NVIDIA Driver:** `580.159.03`
* **CUDA:** `13.0`
* **Docker:** Docker Engine Community `29.2.1`
* **Docker Architecture:** `linux/arm64`
* **Network:** 有線LAN接続（IPアドレス非公開）
* **Network Policy:** 有線LANを使用（詳細非公開）
* **Power:** 1200VA / 750W UPS 配下、ワットチェッカー経由
* **Backup Target:** USB接続 2TB NVMe
* **Backup Device:** `/dev/sda2`
* **Backup Filesystem:** NTFS
* **Backup Mount Point:** `/mnt/atom-backup` / Live環境では `/mnt/backup`
* **Internal Disk:** `/dev/nvme0n1`
* **Internal Disk Size:** 931.5G
* **Internal Partitions:**

  * `/dev/nvme0n1p1` : 512M / vfat / EFI
  * `/dev/nvme0n1p2` : 931G / ext4 / root

## 3. 作業ログ・解決プロセス

### 発生した課題

#### 課題1: 高額・高密度な新規本番候補機材の初期状態を汚す前に保全する必要があった

GIGABYTE AI TOP ATOM は非常に小型であり、EVO-X2の半分よりもさらに小さい印象だった。

一方で、価格は約70万円級であり、今後の用途も以下のように重要である。

```text
- gemma4:31b QLoRA 検証
- ジェムちゃん本番基盤の集約先
- director / actor の単一筐体同居
- EVO-X2 / RTX3090 の本番依存解除
- ARM CUDA / GB10 環境の研究
```

このため、OSやライブラリを追加投入する前に、初期状態を記録・保全する必要があった。

#### 判断理由

今後、ATOMには以下のような依存関係が大量に入る可能性がある。

```text
- CUDA / PyTorch / Unsloth / bitsandbytes
- Ollama / llama.cpp
- Oracle 26ai Free ARM
- Tomcat / nginx / React
- Style-Bert-VITS2
- Whisper / STT
- RAG / WebAPI / 監視系
```

これらを導入した後に問題が出ると、原因がOS初期状態なのか、後から入れた依存関係なのか切り分けが難しくなる。

そのため、まず「ATOM clean baseline」を保存する判断をした。

---

### 課題2: 初期状態のシステム情報を網羅的に記録する必要があった

初期状態として、以下を記録した。

```bash
hostnamectl
uname -a
lsb_release -a || cat /etc/os-release
arch
df -h
lsblk
free -h
nvidia-smi
sudo docker version
sudo docker info
```

主な結果。

```text
Hostname: [hostname masked]
OS: Ubuntu 24.04.4 LTS
Kernel: 6.17.0-1021-nvidia
Architecture: aarch64
Hardware Vendor: GIGABYTE
Hardware Model: AI TOP ATOM
Firmware Version: 5.36_0ACUM06
NVIDIA Driver: 580.159.03
CUDA Version: 13.0
GPU: NVIDIA GB10
Docker: 29.2.1 / linux-arm64
Memory: 121GiB total / 118GiB available
Root Disk: 916G usable / 38G used / 832G available
```

ワットチェッカーでの消費電力も記録した。

```text
電源投入直後: 45W
OSログイン後アイドル: 24W
nvidia-smi実行後: 24W
```

#### 判断理由

今後、QLoRAや常用推論を行うと消費電力・温度・メモリ使用量が大きく変化する。

初期アイドル状態の消費電力やOS状態を残しておくことで、後から以下を比較できる。

```text
- QLoRA中の消費電力
- Ollama常駐時の消費電力
- director / actor 同居時の消費電力
- TTS / STT 追加後の常駐負荷
- GUI停止やヘッドレス化による消費電力差
```

---

### 課題3: DockerからGB10 GPUを利用できるか確認する必要があった

初期状態で、ホスト側の `nvidia-smi` は正常に動作した。

次に、DockerコンテナからGPUが見えるか確認した。

```bash
sudo docker run --rm --gpus all nvidia/cuda:13.0.0-base-ubuntu24.04 nvidia-smi
```

結果、コンテナ内からも NVIDIA GB10 が認識された。

```text
NVIDIA-SMI 580.159.03
Driver Version: 580.159.03
CUDA Version: 13.0
GPU: NVIDIA GB10
```

#### 判断理由

今後の検証では、ホストOSをできるだけ汚さず、Dockerコンテナ内で以下を試す方針である。

```text
- Unsloth
- PyTorch
- CUDA
- QLoRA
- Ollama
- llama.cpp
```

そのため、最初に「DockerからGPUが使えること」を確認する必要があった。

この確認により、少なくとも NVIDIA Container Toolkit / Docker GPU runtime は初期状態で機能していると判断した。

---

### 課題4: Rescuezilla / Clonezilla 系が ARM64 環境で使いにくい可能性があった

事前に 2TB NVMe USB接続と Rescuezilla を準備していた。

しかし、AI TOP ATOM は ARM64 / aarch64 環境であり、一般的な x86_64 向けLiveバックアップツールがそのまま起動・利用できない可能性が高かった。

そのため、Rescuezilla前提ではなく、Linux標準コマンドによるバックアップへ方針を変更した。

#### 判断理由

この時点で必要だったのは、GUIバックアップツールではなく、以下の2点である。

```text
- 初期状態のroot filesystemを安全に残すこと
- 内蔵NVMe全体を後から復元可能な形で残すこと
```

ARM64でLive環境が用意できるなら、`tar` / `dd` / `zstd` の方が環境依存が少ない。

---

### 解決策1: 起動中OSからファイル単位バックアップを取得

まず、通常起動中のATOMから、外付けNVMeをマウントした。

```bash
sudo mkdir -p /mnt/atom-backup
sudo mount -t ntfs3 /dev/sda2 /mnt/atom-backup
df -h /mnt/atom-backup
```

結果。

```text
/dev/sda2  1.9T  61G  1.8T  4%  /mnt/atom-backup
```

バックアップディレクトリを作成した。

```bash
BACKUP_DIR="/mnt/atom-backup/atom-clean-20260605"
sudo mkdir -p "$BACKUP_DIR"
sudo chown "$USER:$USER" "$BACKUP_DIR"
```

システム情報を保存した。

```bash
hostnamectl > "$BACKUP_DIR/hostnamectl.txt"
uname -a > "$BACKUP_DIR/uname-a.txt"
lsb_release -a > "$BACKUP_DIR/lsb_release.txt" 2>&1
arch > "$BACKUP_DIR/arch.txt"
df -h > "$BACKUP_DIR/df-h.txt"
lsblk -o NAME,SIZE,FSTYPE,MODEL,SERIAL,TRAN,MOUNTPOINTS > "$BACKUP_DIR/lsblk-detail.txt"
free -h > "$BACKUP_DIR/free-h.txt"

sudo nvidia-smi > "$BACKUP_DIR/nvidia-smi.txt"
sudo nvidia-smi -q > "$BACKUP_DIR/nvidia-smi-q.txt"

sudo docker version > "$BACKUP_DIR/docker-version.txt"
sudo docker info > "$BACKUP_DIR/docker-info.txt"

sudo sfdisk -d /dev/nvme0n1 > "$BACKUP_DIR/nvme0n1.sfdisk"
sudo blkid > "$BACKUP_DIR/blkid.txt"
sudo efibootmgr -v > "$BACKUP_DIR/efibootmgr-v.txt" 2>&1

dpkg --get-selections > "$BACKUP_DIR/dpkg-selections.txt"
apt-mark showmanual > "$BACKUP_DIR/apt-manual.txt"
```

root filesystem を `tar.zst` として保存した。

```bash
sudo tar \
  --xattrs \
  --acls \
  --selinux \
  --numeric-owner \
  --one-file-system \
  --exclude=/mnt \
  --exclude=/media \
  --exclude=/tmp \
  --exclude=/var/tmp \
  --exclude="$HOME/.cache" \
  -I 'zstd -T0 -10' \
  -cpf "$BACKUP_DIR/rootfs.tar.zst" \
  /
```

EFI領域も別途保存した。

```bash
sudo tar \
  --xattrs \
  --acls \
  --numeric-owner \
  -I 'zstd -T0 -10' \
  -cpf "$BACKUP_DIR/boot-efi.tar.zst" \
  /boot/efi
```

---

### 発生した課題5: tar実行時に警告が出た

`tar` 実行時に以下の警告が出た。

```text
tar: メンバ名から先頭の `/' を取り除きます
tar: /sys: 読み込んだファイルが変更されています
tar: ハードリンク先から先頭の `/' を取り除きます
tar: /var/lib/gdm3/.cache/ibus/dbus-...: ソケットは無視します
tar: ...: 警告: flistxattr 不能: サポートされていない操作です
```

#### 判断理由

これらは、起動中OSをバックアップする場合に発生しうる警告である。

主な意味は以下。

```text
先頭の / を取り除く:
  tar復元時に絶対パスへ強制展開しないための通常動作。

/sys が変更された:
  起動中の仮想ファイルシステムが変化しているため。
  --one-file-system 指定により、実体のない疑似FSは重要対象ではない。

ソケットは無視:
  実行中プロセスの一時ソケットであり、復元対象として不要。

flistxattr不能:
  snap等の読み取り専用squashfsや特殊FSに対するxattr取得警告。
```

したがって、警告内容を確認した範囲では、今回のrootfsバックアップ取得が直ちに失敗したことを示すものではないと判断した。

---

### 発生した課題6: チェックサム作成時に boot-efi.tar.zst がまだ無い状態で実行した

最初に以下を実行した際、`boot-efi.tar.zst` が存在せずエラーとなった。

```bash
sha256sum rootfs.tar.zst boot-efi.tar.zst *.txt *.sfdisk > SHA256SUMS.txt
```

エラー。

```text
sha256sum: boot-efi.tar.zst: そのようなファイルやディレクトリはありません
```

その後、`boot-efi.tar.zst` を取得したうえで、再度チェックサムを作成した。

---

### 発生した課題7: SHA256SUMS.txt 自身をハッシュ対象に含めてしまった

再作成後、以下のように `SHA256SUMS.txt` 自身が失敗した。

```text
SHA256SUMS.txt: 失敗
```

#### 判断理由

`*.txt` に `SHA256SUMS.txt` 自身が含まれていたため、自己参照になり、チェックサム作成後にファイル内容が変化した。

そのため、`SHA256SUMS.txt` 自身の失敗は、バックアップ本体の破損を意味しないと判断した。

実際に以下はOKだった。

```text
rootfs.tar.zst: OK
boot-efi.tar.zst: OK
各種txt: OK
nvme0n1.sfdisk: OK
```

---

### 解決策2: ARM64 Ubuntu Server Live USBを起動し、ディスク丸ごとバックアップを取得

起動中OSからのファイル単位バックアップに加え、ディスク全体のバックアップを取得するため、Ubuntu Server ARM64 Live USBを作成した。

当初、通常のDELキー連打ではUEFI/BIOSへ入れなかった。

そのため、OS上から以下を実行した。

```bash
sudo systemctl reboot --firmware-setup
```

これにより、UEFI設定へ入り、USB起動を行うことができた。

#### 判断理由

GIGABYTE AI TOP ATOM / DGX Spark系では、通常のPCと同じ感覚でDELキー連打が効かない場合がある。

本環境では、OSから `reboot --firmware-setup` を実行することでUEFI設定へ遷移できた。

---

### 解決策3: Ubuntu Server Live Installer からTTYへ入り、root相当のシェルを使用

Ubuntu Server Live起動後、以下の手順でシェルへ入った。

```text
1. 「Try or Install Ubuntu Server」を選択
2. インストーラー起動後、Alt + F2 / F3 等で別TTYへ切り替え
3. Enterを押してコンソールを有効化
4. sudoを使って必要コマンドを実行
```

Live環境でデバイスを確認した。

```bash
lsblk -o NAME,SIZE,FSTYPE,MODEL,SERIAL,TRAN,MOUNTPOINTS
```

結果。

```text
sda           1.8T  USB外付けNVMe
sda2          1.8T  ntfs

sdb          58.6G  USB Flash Drive
sdb1         58.6G  vfat /cdrom

nvme0n1     931.5G 内蔵SSD
nvme0n1p1   512M   vfat
nvme0n1p2   931G   ext4
```

この時点で、以下の対応関係を確認した。

```text
読み取り元:
  /dev/nvme0n1
  AI TOP ATOM 内蔵SSD全体

書き込み先:
  /mnt/backup/atom-clean-20260605/nvme0n1-full.img.zst
  USB接続2TB NVMe上の圧縮イメージファイル
```

---

### 発生した課題8: sudoを付け忘れ、ddがPermission deniedになった

最初に以下を実行した。

```bash
dd if=/dev/nvme0n1 bs=64M status=progress conv=noerror,sync \
  | zstd -T0 -10 \
  -o /mnt/backup/atom-clean-20260605/nvme0n1-full.img.zst
```

結果、`dd` が内蔵NVMeを開けず失敗した。

```text
dd: failed to open '/dev/nvme0n1': Permission denied
```

しかし、後段の `zstd` は空に近い入力を圧縮し、13バイトのゴミファイルを作成した。

```text
nvme0n1-full.img.zst 13 bytes
```

#### 判断理由

ブロックデバイス `/dev/nvme0n1` を読むにはroot権限が必要である。

パイプラインにおいて、前段の `dd` が失敗しても、後段の `zstd` がファイルを作成する場合がある。

そのため、出力ファイルサイズを確認して、異常な13バイトファイルがゴミであると判断した。

---

### 発生した課題9: ゴミファイルが存在したため、sudo付き再実行でもzstdが上書きを拒否した

次に `sudo dd` で再実行した。

```bash
sudo dd if=/dev/nvme0n1 bs=64M status=progress conv=noerror,sync \
  | zstd -T0 -10 \
  -o /mnt/backup/atom-clean-20260605/nvme0n1-full.img.zst
```

しかし、既に13バイトの `nvme0n1-full.img.zst` が存在していたため、`zstd` が上書きを拒否した。

```text
zstd: /mnt/backup/atom-clean-20260605/nvme0n1-full.img.zst already exists; stdin is an input - not proceeding.
```

#### 解決策

一度ゴミファイルを削除した。

```bash
rm nvme0n1-full.img.zst
```

その後、再度 `sudo dd` から実行した。

```bash
sudo dd if=/dev/nvme0n1 bs=64M status=progress conv=noerror,sync \
  | zstd -T0 -10 \
  -o /mnt/backup/atom-clean-20260605/nvme0n1-full.img.zst
```

結果、内蔵NVMe全体の読み取りと圧縮が完了した。

```text
1000257617920 bytes copied
932 GiB read
524 MB/s
```

圧縮後のファイルサイズ。

```text
nvme0n1-full.img.zst
14,658,204,972 bytes
```

チェックサムも作成・検証した。

```bash
cd /mnt/backup/atom-clean-20260605
sha256sum nvme0n1-full.img.zst > nvme0n1-full.img.zst.sha256
sync
sha256sum -c nvme0n1-full.img.zst.sha256
```

結果。

```text
nvme0n1-full.img.zst: OK
```

#### 判断理由

内蔵SSD全体は約1TBだが、初期状態で使用量が少ないため、zstd圧縮後のサイズは約13.7GiBとなった。

`sha256sum -c` がOKであるため、チェックサム作成後に圧縮イメージファイルの内容が変化していないことを確認した。ただし、実際に復元して起動できるかは未検証である。

---

### 解決策4: バックアップファイルを3か所へ複製

取得したバックアップは、USB接続NVMeだけでなく、家庭内の共有フォルダとNASにも保存した。

保存先。

```text
1. USB接続 2TB NVMe
2. 自作PCの共有フォルダ
3. QNAP NAS（HDD RAID1）
```

#### 判断理由

AI TOP ATOMは高額かつ、今後の本番基盤として重要である。

そのため、初期状態バックアップは単一ストレージでは不十分と判断した。

以下の3段構成にすることで、媒体故障・誤削除・作業ミスに対する耐性を高めた。

```text
USB NVMe:
  現場復旧用。すぐ接続して使える。

自作PC共有フォルダ:
  LAN内の一次保管。扱いやすい。

QNAP NAS RAID1:
  HDD冗長化された長期保管。
```

## 4. 成果物

### 4.1 初期状態記録ファイル

保存先。

```text
/mnt/atom-backup/atom-clean-20260605
```

Live環境では以下。

```text
/mnt/backup/atom-clean-20260605
```

主な保存ファイル。

```text
hostnamectl.txt
uname-a.txt
lsb_release.txt
arch.txt
df-h.txt
lsblk-detail.txt
free-h.txt
nvidia-smi.txt
nvidia-smi-q.txt
docker-version.txt
docker-info.txt
nvme0n1.sfdisk
blkid.txt
efibootmgr-v.txt
dpkg-selections.txt
apt-manual.txt
```

これらは復旧用バックアップに含まれる内部記録であり、実ファイル自体は公開対象に含めない。本レポートでは、ファイル名と取得手順、公開可能な範囲の確認結果のみを記載する。

### 4.2 root filesystem バックアップ

```text
rootfs.tar.zst
```

サイズ。

```text
12,354,009,469 bytes
```

### 4.3 EFI領域バックアップ

```text
boot-efi.tar.zst
```

サイズ。

```text
1,978,775 bytes
```

### 4.4 内蔵NVMe丸ごとイメージ

```text
nvme0n1-full.img.zst
```

サイズ。

```text
14,658,204,972 bytes
```

チェックサム。

```text
nvme0n1-full.img.zst.sha256
```

検証結果。

```text
nvme0n1-full.img.zst: OK
```

### 4.5 rootfs バックアップ取得コマンド

```bash
sudo tar \
  --xattrs \
  --acls \
  --selinux \
  --numeric-owner \
  --one-file-system \
  --exclude=/mnt \
  --exclude=/media \
  --exclude=/tmp \
  --exclude=/var/tmp \
  --exclude="$HOME/.cache" \
  -I 'zstd -T0 -10' \
  -cpf "$BACKUP_DIR/rootfs.tar.zst" \
  /
```

### 4.6 EFI バックアップ取得コマンド

```bash
sudo tar \
  --xattrs \
  --acls \
  --numeric-owner \
  -I 'zstd -T0 -10' \
  -cpf "$BACKUP_DIR/boot-efi.tar.zst" \
  /boot/efi
```

### 4.7 ディスク丸ごとバックアップ取得コマンド

※ デバイス名は実行環境によって異なるため、実行前に `lsblk` 等で読み取り元と書き込み先を確認する。

```bash
sudo dd if=/dev/nvme0n1 bs=64M status=progress conv=noerror,sync \
  | zstd -T0 -10 \
  -o /mnt/backup/atom-clean-20260605/nvme0n1-full.img.zst
```

### 4.8 チェックサム作成・検証コマンド

```bash
cd /mnt/backup/atom-clean-20260605
sha256sum nvme0n1-full.img.zst > nvme0n1-full.img.zst.sha256
sync
sha256sum -c nvme0n1-full.img.zst.sha256
```

### 4.9 UEFI起動コマンド

DELキー連打でUEFIへ入れなかったため、OS上から以下を使用した。

```bash
sudo systemctl reboot --firmware-setup
```

## 5. 得られた設計知見

### 5.1 ARM64環境では、x86前提のバックアップツールに依存しない方がよい

Rescuezilla / Clonezilla 系は便利だが、ARM64 / GB10 環境では、起動可否やドライバ対応が不確実である。

そのため、最低限以下の標準コマンドで復元に利用できる可能性のあるバックアップを取る方が堅実である。

```text
tar
dd
zstd
sha256sum
sfdisk
blkid
efibootmgr
```

### 5.2 初期状態では「ファイル単位」と「ディスク丸ごと」の両方を取る価値がある

ファイル単位バックアップは、内容確認や部分復元に向いている。

```text
rootfs.tar.zst
boot-efi.tar.zst
```

一方、ディスク丸ごとバックアップは、パーティションやブート構成を含む復元に利用できる可能性がある。

```text
nvme0n1-full.img.zst
```

両方を残すことで、復旧シナリオを複数持てる。

### 5.3 パイプラインでは前段が失敗しても後段がファイルを作ることがある

今回、`dd` がPermission deniedで失敗したにもかかわらず、`zstd` が13バイトの出力ファイルを作成した。

この種のコマンドでは、以下の確認が必要である。

```text
- コマンド終了後のファイルサイズ
- 入力デバイスを最後まで読めたか
- sha256sum が通るか
- ログに Permission denied や already exists がないか
```

特に `dd | zstd -o file` では、失敗時のゴミファイルを削除してから再実行する必要がある。

### 5.4 `sudo dd | zstd` では、sudoが必要なのは読み取り元デバイスを開く前段である

`/dev/nvme0n1` のようなブロックデバイスを読むにはroot権限が必要である。

今回のコマンドでは、以下のように `dd` 側に `sudo` を付ける必要がある。

```bash
sudo dd if=/dev/nvme0n1 ...
```

一方、出力先がユーザー書き込み可能なディレクトリであれば、`zstd` 側は通常ユーザーでも動作する。

### 5.5 UEFIに入るには `systemctl reboot --firmware-setup` が有効だった

AI TOP ATOMでは、DELキー連打ではUEFI設定に入れなかった。

しかし、Ubuntu上から以下を実行することでUEFIに入れた。

```bash
sudo systemctl reboot --firmware-setup
```

この手順は、今後USBブートやBoot Option変更が必要な場合の運用知見として有用である。

### 5.6 Ubuntu Desktopは初期状態では怖く見えるが、検証段階では問題ない

AI TOP ATOMはUbuntu Desktopとして起動した。

ヘッドレス運用予定のためGUIがあることに少し不安はあったが、初期状態の検証では問題にならなかった。

`nvidia-smi` 上では Xorg / gnome-shell がGPUプロセスとして表示されたが、初期検証時点では負荷は軽微だった。

本格運用前には、必要に応じてヘッドレス化を検討する。

```bash
sudo systemctl set-default multi-user.target
```

ただし、QLoRA検証や初期バックアップが済むまでは、急いでGUIを無効化しない方針とした。

### 5.7 初期状態バックアップは、QLoRA検証前に取得する価値が大きい

今後、最初に試す予定の重要検証は `gemma4:31b QLoRA` である。

この検証では、以下が導入される可能性がある。

```text
- PyTorch
- CUDA runtime / CUDA dev libraries
- Unsloth
- bitsandbytes
- transformers
- peft
- trl
- llama.cpp
- Hugging Face cache
```

これらは依存関係が複雑であり、特にARM64 / Blackwell / CUDA 13.0 では失敗時の切り分けが難しくなる。

そのため、検証前にクリーンバックアップを取れたことは大きい。

### 5.8 バックアップは3か所に分散保存することで意味が増す

単にUSB NVMeへ保存しただけでは、USB側の故障や誤削除に弱い。

今回、以下の3か所に保存した。

```text
1. USB NVMe
2. 自作PC共有フォルダ
3. QNAP NAS RAID1
```

これにより、初期状態の保全性が高まった。

## 6. 未解決の課題

### 6.1 実際の復元テストは未実施

バックアップ取得とチェックサム検証は完了した。

しかし、実際に別ディスクへ復元して起動できるかまでは未検証である。

今後、予備NVMeを使って以下を確認できると理想である。

```text
- nvme0n1-full.img.zst を展開
- パーティションが復元される
- EFIから起動できる
- NVIDIA / Docker / CUDA が元通り動作する
```

### 6.2 取得済みSHA256SUMS.txtの自己参照問題

`SHA256SUMS.txt` 自身を含めてしまったため、同ファイルだけ検証失敗した。

バックアップ本体は問題ないが、後で気になる場合は、`SHA256SUMS.txt` を対象外にして作り直すとよい。

### 6.3 GUI / ヘッドレス運用方針

現時点ではUbuntu Desktopのまま運用開始している。

今後、完全ヘッドレス運用へ寄せる場合は、以下を検討する。

```text
- multi-user.target への変更
- GUI停止時の消費電力差
- nvidia-smi上のXorg / gnome-shellプロセス消失確認
- リモート復旧手段の確保
```

ただし、初期検証段階ではGUIを急いで無効化しない方針とする。

### 6.4 外付けNTFSの長期運用方針

今回のバックアップ保存先はNTFSであった。

tar.zst / img.zst のような単一ファイル保存には問題ないが、Linuxの権限・ACL・xattrを保持したrsync展開先としては不向きである。

今後、Linuxバックアップ専用領域を作る場合は、ext4やxfsの外付け領域を別途用意する価値がある。

### 6.5 バックアップの世代管理

今回は `atom-clean-20260605` として初期状態を保存した。

今後、以下の節目ごとに世代を分けるとよい。

```text
- clean-20260605:
  OS初期状態 / Docker GPU確認済み

- qlora-base:
  QLoRA最小環境構築後

- gemma4-31b-success:
  gemma4:31b QLoRA / merge / GGUF / Ollama登録成功後

- production-before-cutover:
  ジェムちゃん本番切替直前
```

## 7. 最終結論

今回の作業により、GIGABYTE AI TOP ATOMの初期状態確認と、クリーンバックアップ取得が完了した。

確認できたこと。

```text
- Ubuntu 24.04.4 LTS / arm64 環境で起動する
- Kernel 6.17.0-1021-nvidia が動作している
- NVIDIA GB10 がホストから認識される
- NVIDIA Driver 580.159.03 / CUDA 13.0 が有効
- Docker Engine 29.2.1 / linux-arm64 が動作している
- NVIDIA CUDA 13.0 コンテナからGB10を認識できる
- 有線LAN経由で通信できることを確認した
- アイドル消費電力は約24W
```

バックアップとして、以下を取得した。

```text
rootfs.tar.zst
boot-efi.tar.zst
nvme0n1-full.img.zst
nvme0n1-full.img.zst.sha256
各種システム情報txt
nvme0n1.sfdisk
blkid.txt
efibootmgr-v.txt
```

さらに、バックアップファイルは以下の3か所へ保存した。

```text
1. USB接続 2TB NVMe
2. 自作PCの共有フォルダ
3. QNAP NAS（HDD RAID1）
```

作業中には、以下の失敗も発生した。

```text
- tar実行時の警告
- boot-efi.tar.zst 作成前にチェックサムを作ろうとした
- SHA256SUMS.txt 自身をハッシュ対象に含めた
- Live環境でddにsudoを付け忘れた
- 13バイトのゴミ nvme0n1-full.img.zst が作成された
- ゴミファイルの存在により、zstdが再実行時に上書きを拒否した
```

しかし、それぞれ原因を確認し、適切に対処した。

最終的には、Live Ubuntu環境から内蔵NVMe全体を読み取り、圧縮イメージとして保存し、チェックサム検証まで完了した。

これにより、AI TOP ATOMは今後の `gemma4:31b QLoRA clean-room検証` に進める状態になった。

今回の作業は、単なる初期設定ではない。

約70万円級の小型GB10マシンを、今後の本番・研究・QLoRA検証に投入する前に、復元に利用できる初期状態のイメージとして保存した作業である。

このバックアップにより、今後ライブラリ依存やARM CUDA環境で問題が発生した場合に、「購入直後に近い正常状態」へ戻すための復元手段を確保した。ただし、実ディスクへの復元と起動確認は今後の検証課題である。

次のフェーズでは、このクリーン状態を土台として、gemma4:31b QLoRAの最小検証に進む。
