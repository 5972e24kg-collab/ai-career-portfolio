# Work032: Unsloth環境構築

Tag: [[Unsloth]], [[Docker]], [[RTX3060]], [[QLoRA]], [[EVO-X2]]

## 1. 目的

* 自作小説キャラクターの対話スタイル・キャラクター性を学習させるプロジェクト「Soul Injection」の始動。
* Node B（検証用ローカルLLMノード / EVO-X2）に接続された eGPU (RTX 3060) を使用し、高速学習ライブラリ `Unsloth` が動作するDocker環境を構築する。

## 2. システム環境

* **Node:** Node B（検証用ローカルLLMノード / EVO-X2 / Ubuntu 24.04 Native）
* **GPU:** NVIDIA GeForce RTX 3060 12GB (via USB4 eGPU)
* **Container:** Docker (NVIDIA Container Toolkit 導入済み)
* **Target Library:** Unsloth（構築時点の latest）

## 3. 作業ログ・解決プロセス

### 発生した課題

1. **Dockerイメージの不整合:**

   * 当初指定した `unslothai/unsloth:latest-cu121-torch240` がリポジトリに存在せず、Pullエラーが発生した。
   * 作業時点の確認では、Unsloth公式のDockerリポジトリ名が変更され、タグの命名規則も変わっていたことが判明。

### 解決策

* **公式イメージへの変更:**

  * イメージ名を `unsloth/unsloth:latest` に修正。
  * これにより、環境構築時点での最新のCUDA/PyTorch環境が自動的に適用される構成とした。

### 検証結果

* **GPU認識:** コンテナ内から `nvidia-smi` を実行し、RTX 3060 が認識されていることを確認。
* **ライブラリ動作:** Pythonシェルにて `import unsloth` および `torch.cuda.is_available()` が `True` を返すことを確認。

## 4. 成果物 (docker-compose.yml)

```yaml
services:
  unsloth-trainer:
    image: "unsloth/unsloth:latest"
    container_name: soul-injection-trainer
    command: sleep infinity
    volumes:
      - ./:/workspace
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    shm_size: 16gb
    restart: unless-stopped
    environment:
      - HF_TOKEN=${HF_TOKEN}  # 実トークンは公開しない
```

## 5. 次のアクション

* Phase 2: 小説テキストデータの整形と、120Bモデルを用いた学習用Q&Aデータセットの生成。
