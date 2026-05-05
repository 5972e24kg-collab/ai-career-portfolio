# Work010: EVO-X2ベンチマーク検証MEM編

Tag: [[EVO-X2]], [[Benchmark]], [[AIDA64]], [[BIOS]], [[Docker]]

## 1. 今回の作業の目的 [#purpose]
#contents
- 新規導入した「GMKtec EVO-X2」のメモリアーキテクチャ（Strix Halo / 256-bit Bus）の実効速度を計測する。
- 本機の最大の特長である「VRAM 96GB」設定をBIOSにて適用し、OSおよびCPU性能への副作用（オーバーヘッド）がないか検証する。
- 今後のコンテナ運用におけるソフトウェアスタック（Docker Desktop vs Linux Native）を決定する。

## 2. システム環境 (設定変更後) [#environment]
- **Model:** GMKtec EVO-X2 (Ryzen AI Max+ 395)
- **Memory Allocation:**
  - **Total Physical:** 128 GB (LPDDR5-8000)
  - **VRAM (Dedicated):** **96 GB** (BIOS設定により拡張)
  - **System RAM:** 32 GB (OS/Docker用)
- **OS:** Windows 11 Pro ([Date masked]) + WSL2
- **BIOS Version:** 1.09

## 3. メモリ帯域幅測定 (AIDA64 Extreme) [#bandwidth]
VRAM 64GB設定時（Default）におけるメモリRead速度の測定結果。

| Category | Score | Status | Analysis |
| :--- | :--- | :--- | :--- |
| **Memory Read** | **121.22 GB/s** | Base | 理論値(256GB/s)の約50%。Hypervisor干渉あり。 |
| **Latency** | 135.7 ns | High | 仮想化レイヤーによる遅延を確認。 |

- **考察:**
  - スコアが伸び悩んだ主因はWindowsの「Hypervisor（仮想化基盤）」が有効であるため（AIDA64画面左下に警告表示あり）。
  - WSL2運用を行う以上、この「120GB/s超」が実運用上のベースラインとなる。一般的なDDR5 Dual-Channel（80-90GB/s）と比較しても約1.3倍〜1.5倍の帯域を確保できており、LLM推論には十分と判断。

## 4. BIOSチューニング (VRAM 96GB化) [#bios_tuning]
タスクマネージャー上で「専用GPUメモリ 96.0GB」を実現するための設定手順。

### 設定手順 [#bios_steps]
1. 起動時に `Del` または `F7` キーでBIOSへ入る。
1. タブメニュー **[Advanced]** を選択。
1. 最下部の **[GFX Configuration]** を選択。
1. **[UMA Frame Buffer Size]** を `Auto` から **[96G]** に変更。
1. 保存して再起動。

### 運用上の注意点 (重要) [#warning]
- **システムメモリ残量:** Windows側は残り32GBとなる。LLMローダー（Ollama/LM Studio等）使用時は、モデルをシステムメモリに展開しないよう `mmap` 無効化や `Keep in memory` 設定に注意が必要。
- **Ollamaロード確認:** ログにて `llm_load_tensors: offloaded ... to GPU` となり、VRAM内で完結していることを必ず確認する運用ルールを策定。

## 5. 変更後の健全性確認 (Regression Test) [#regression]
VRAMを96GB確保した後、CPU性能が低下していないか確認するため Cinebench R23 を再実行。

| Metric | VRAM 64GB (Before) | VRAM 96GB (After) | Delta |
| :--- | :--- | :--- | :--- |
| **Multi Core** | **36,110 pts** | **36,026 pts** | -0.2% (誤差) |
| **Single Core** | 2,045 pts | 2,043 pts | -0.1% (誤差) |

- **結論:** VRAMへの極端な割り当てを行っても、CPU側の演算性能への悪影響は皆無である。このハードウェア構成（[Server name masked]）は「計算資源」と「メモリ資源」を完全に分離できている。

## 6. ソフトウェア構成の決定 [#software]
今後のDocker運用について、以下の理由から「Docker Desktop for Windows」を採用する。

- **採用構成:** Windows 11 + Docker Desktop (WSL2 Backend)
- **選定理由:**
  - **ドライバ連携:** Ryzen AI (Strix Halo) は最新チップであり、Windows側のAMDドライバ（Adrenalin）とWSL2側の連携をDocker Desktopに任せる方が安定性が高い。
  - **リソース管理:** 特殊なメモリ配分（96GB/32GB）であるため、GUIダッシュボードでメモリ消費を常時監視できるメリットが大きい。
  - **一元管理:** Ubuntu以外のディストリビューションを追加した場合も、コンテキストを統一できる。

## 7. 次のステップ [#next]
- **環境構築:** WindowsへのDocker Desktopインストールおよび、WSL2 (Ubuntu [Date masked]) との `WSL Integration` 設定。
- **GPUパススルー検証:** Dockerコンテナ内から `/dev/kfd` デバイスが見えるか、および `rocm-smi` コマンド等で VRAM 96GB が認識されるかの開通テストを行う。