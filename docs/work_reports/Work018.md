# Work018: ホストOSのUbuntu移行

Tag: [[Ubuntu]], [[Strix Halo]], [[ROCm]], [[Docker]]

## 1. 目的
- 最新チップ "Strix Halo" (Ryzen AI Max+ 395) をUbuntu [Date masked]で動作させ、ROCm経由でLLM推論を行う。

## 2. システム環境
- **Hardware:** GMKtec EVO-X2 (VRAM 96GB設定)
- **OS:** Ubuntu [Date masked] LTS
- **Driver:** GMKtec提供 `amdgpu-install_6.4.60402-1_all.deb`
- **Target:** Ollama (Docker Container)

## 3. 解決プロセス
### 発生した課題
1. **ドライバ認識:** 標準カーネルではiGPU (Radeon 8060S) の性能が出ない。
2. **Dockerエラー:** `could not select device driver "" with capabilities: [[gpu]]` が発生しコンテナが起動しない。

### 解決策
#### A. ドライバインストール
メーカー提供スクリプトを使用。`amdkcl` (Kernel Compatibility Layer) がロードされることで正常動作する。
```bash
chmod +x SU_install_by_user.sh
./SU_install_by_user.sh
# 完了後、 rocm-smi でデバイスID 0x1586 (Strix Halo) が見えることを確認
```

#### B. Docker起動オプションの最適化
NVIDIA用の `--gpus=all` は削除し、Linux Nativeなデバイスマッピングを使用する。
```yaml
# docker-compose.yml 抜粋
services:
  ollama:
    image: ollama/ollama:rocm
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    # environment:
    #   - HSA_OVERRIDE_GFX_VERSION=11.0.0  # ※今回は不要だったが、認識しない場合の保険
```

## 4. 成果
- **Prompt Eval Rate:** 132.72 t/s (爆速)
- **Eval Rate:** 4.82 t/s (Llama 3.3 70B Q4_K_M)
- Hypervisorのオーバーヘッドがなくなり、ハードウェア理論値に近い帯域効率を達成。