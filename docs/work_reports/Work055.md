# Work055: StyleBertVITS2音声合成基盤構築と永続化
Tag: [[Node C]], [[StyleBertVITS2]], [[Docker]], [[WSL2]], [[TTS]]

## 1. 目的
- 「分散型身体性アーキテクチャ」の Phase 1 として、Node C (身体担当) に発声能力を持たせる。
- Dockerコンテナ上で StyleBertVITS2 を稼働させ、YAMAHA HS-5 から物理音声を出力する。
- 外部ネットワークに依存せず、オフラインでも数秒で起動・発声可能な「完全永続化環境」を構築する。

## 2. システム環境
- **Node:** Node C (Ryzen 5700G + RTX 3060 12GB)
- **OS:** Windows 11 Pro + WSL2 (Ubuntu 24.04)
- **Middleware:** Docker Desktop (WSL2 Backend), NVIDIA Container Toolkit
- **Software:** `litagin/style-bert-vits2:latest`
- **Device:** YAMAHA HS-5 (Audio Output via Host)

## 3. 作業ログ・解決プロセス

### 課題1: WSL2からのGPU認識 (The Driver Trap)
- **事象:** WSL2上のUbuntuから `nvidia-smi` コマンドが見つからない。
- **原因:** Windows側のドライバーは正常だが、WSL内のライブラリパス (`/usr/lib/wsl/lib`) が環境変数に含まれていなかった。
- **解決:** `ldconfig` 設定ファイルの追加と、`/usr/local/bin` へのシンボリックリンク作成により、ドライバ再インストールなしで認識させた。

### 課題2: モデルパスの迷宮 (The Path Labyrinth)
- **事象:** コンテナ内のモデルディレクトリと、ホスト側のマウント先が一致せず、モデル ([voice model name masked]) が認識されなかった。
- **原因:** プログラムは `/sbv2/model_assets` を参照していたが、`docker-compose.yml` で `/app/model_assets` にマウントしていた。
- **解決:** マウントパスを `/sbv2/model_assets` に修正し、コンテナ内ディレクトリ構造に合わせることで解決。

### 課題3: 起動時のダウンロード地獄 (The Eternal Download)
- **事象:** コンテナ再起動のたびに `initialize.py` が数GB単位のBERTモデル等をダウンロードし、起動に数分を要した。
- **原因:** `model_assets` 以外の中間ファイル（BERT, Pretrained, NLTK）が永続化されておらず、コンテナ破棄と共に消滅していた。
- **解決:**
    1. 起動済みコンテナから `docker cp` で必要なディレクトリ (`bert`, `pretrained`, `pretrained_jp_extra`, `nltk_data`) をホスト側に救出。
    2. これらを `docker-compose.yml` の `volumes` に追加。
    3. ダウンロード処理がキャッシュヒット（ローカル参照）することで、起動時間を**数秒**に短縮。

## 4. 成果物 (docker-compose.yml)
起動コマンドを `initialize.py && app.py` とすることで、整合性チェックと高速起動を両立させた最終構成。

```yaml
services:
  style-bert-vits2:
    image: litagin/style-bert-vits2:latest
    container_name: style-bert-vits2
    
    # 初期化スクリプト(高速) -> WebUI起動
    command: >
      bash -c "python3 initialize.py && python3 app.py"
    
    ports:
      - "[port masked]:[bind address masked]" # API (Phase 2で使用)
      - "[port masked]:[bind address masked]" # WebUI (調整用)
      
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    volumes:
      # Model Assets ([voice model name masked], [voice model name masked])
      - ./data/models:/sbv2/model_assets
      # Application Logs
      - ./data/logs:/sbv2/Data
      # Fonts
      - ./data/fonts:/sbv2/Styles/Fonts
      # HuggingFace Cache
      - ./data/hf_cache:/root/.cache/huggingface
      
      # --- System Persistence (For Offline/Fast Boot) ---
      - ./data/bert:/sbv2/bert
      - ./data/pretrained:/sbv2/pretrained
      - ./data/pretrained_jp_extra:/sbv2/pretrained_jp_extra
      - ./data/nltk_data:/root/nltk_data
      
    environment:
      - TZ=Asia/Tokyo
      - GRADIO_SERVER_NAME=[bind address masked]
    tty: true
    stdin_open: true
    restart: always
```