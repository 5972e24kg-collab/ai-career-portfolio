# Work001: LLMベンチマークと自宅PukiWiki資産のRAG取り込み

Tag: [[LLM]], [[RAG]], [[Docker]], [[Python]], [[OpenWebUI]]

## 1. 今回の作業の目的 [#purpose]
#contents
- 自宅サーバーで長年運用しているPukiWikiのナレッジ（Markdown化されたテキスト）を、ローカルLLM（Open WebUI）のRAG（検索拡張生成）に自動的に取り込ませる。
- GPUリソース（VRAM 8GB）の制約の中で、システムをクラッシュさせずに安定して同期・更新を行うパイプラインを確立する。
- 最終的にチャットで「Wikiを取り込んで」と指示するだけで、最新のWiki情報に基づいた回答が得られる環境を構築する。

## 2. システム環境 [#environment]
- **Host OS:** Windows 11 (WSL2 / Ubuntu 22.04)
- **Middleware:** Docker Desktop (Compose V2), Ollama (v0.x), Open WebUI (v0.x)
- **Target Component:** Open WebUI Tools (Python Script), ChromaDB
- **Hardware:** NVIDIA GeForce RTX 4060 (8GB)
- **External Resource:** PukiWiki Server (SSH Access)

## 3. 作業時間 [#time]
- 開始日時: [Date masked] 13:00頃 (推定)
- 完了日時: [Date masked] 22:45 (最終動作確認)
- 所要時間: 約 34 時間 (断続的なトラブルシューティング含む)

## 4. 作業の軌跡と解決プロセス [#process]

### 4.1 発生した課題・エラー [#issues]
今回のプロジェクトでは、以下の技術的障壁によりRAGの登録が阻害されました。

1. **デッドロック（Deadlock）**
  - Tool実行中にAPIを呼び出すと `Read timed out` が発生。Open WebUIが自身の処理完了待ち状態で、自身へのAPIリクエストを受け取れずフリーズした。
1. **VRAM不足によるクラッシュ**
  - 14Bモデル（Qwen2.5:14b）とEmbeddingモデルを同時稼働させた際、VRAM 8GBが枯渇しOpen WebUIコンテナが応答不能になった。
1. **API仕様の不一致（422 Error）**
  - `{"file_ids": [...]}` で一括登録しようとした際、`Field required` エラーが発生。また、ナレッジベースのメタデータ（name/description）の再送信が求められた。
1. **Embeddingモデルの次元不一致**
  - 初期設定（384次元）のコレクションに対し、高性能モデル `nomic-embed-text`（768次元）のデータを投入しようとして `InvalidArgumentError` が発生。

### 4.2 試行錯誤のプロセス [#trials]
1. **API経由での同期実行** -> タイムアウトで失敗（デッドロック）。
1. **待機時間（sleep）の導入** -> 改善せず。根本的なプロセスの競合が原因と判明。
1. **非同期処理（Threading）の導入** -> **成功**。メインプロセスを即座に解放し、裏でAPIを叩くことでデッドロックを回避。
1. **軽量モデル（0.5B）への切り替え** -> **成功**。RAG登録作業時は `qwen2.5:0.5b` を使用することでVRAMをEmbedding処理に全振りする運用ルールを確立。
1. **直接DB書き込み（ChromaDB）** -> 失敗。稼働中のDBファイルへのアクセス権限エラー。API経由が正攻法と判断。
1. **コレクションの再作成** -> **成功**。古いコレクションを削除し、`nomic-embed-text` 設定下で再作成することで768次元対応のDBを構築。

### 4.3 解決策・教訓 [#solution]
- **非同期処理の徹底:** Open WebUIのToolから自身を操作する場合、必ず `threading` でバックグラウンドに逃がす。
- **1ファイルずつのループ処理:** APIのタイムアウトや仕様変更（`file_id` 単数形要求）に柔軟に対応するため、一括ではなくループ処理を採用。
- **GPUリソース管理:** RTX 4060環境では「推論」と「学習/登録」を同時に行わない。作業用モデル（0.5B）と実用モデル（14B）を明確に使い分ける。

## 5. 成果物 [#artifact]
RAGへの登録を安定して行う最終版スクリプトです。
Wikiのファイル数を動的にカウントし、1ファイルずつ確実に登録を行います。

### ファイル名: RAG Importer (Tools) [#code]
```python
"""
title: Import Wiki to RAG (Single Loop)
author: Gemini Partner
content: Imports markdown files to RAG one by one using 'file_id' to satisfy API requirements.
"""

import os
import requests
import json
import time
import threading

# --- 【設定】 ---
OPENWEBUI_API_KEY = "[token masked]"
KNOWLEDGE_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# ---------------

TARGET_DIR = "[Path masked]"
API_BASE_URL = "[internal Open WebUI API endpoint masked]"


def run_import_task():
    print("--- [Background] RAG Import Started (Single Loop) ---")

    try:
        headers = {"Authorization": f"Bearer {OPENWEBUI_API_KEY}"}

        # 1. ファイル確認
        if not os.path.exists(TARGET_DIR):
            print(f"[Background] Error: Directory not found {TARGET_DIR}")
            return

        files = [f for f in os.listdir(TARGET_DIR) if f.endswith(".md")]
        if not files:
            print("[Background] Error: No markdown files found.")
            return

        print(f"[Background] Found {len(files)} files.")

        uploaded_count = 0
        registered_count = 0

        # 2. ループ処理：アップロード -> 即登録
        for i, filename in enumerate(files):
            file_path = os.path.join(TARGET_DIR, filename)

            # A. アップロード
            try:
                with open(file_path, "rb") as f:
                    files_data = {"file": (filename, f, "text/markdown")}
                    resp = requests.post(
                        f"{API_BASE_URL}/files/",
                        headers=headers,
                        files=files_data,
                        timeout=60,
                    )

                if resp.status_code == 200:
                    file_id = resp.json()["id"]
                    uploaded_count += 1

                    # B. 登録 (ここを修正: file_id 単数形で送る)
                    payload = {"file_id": file_id}
                    url_add = f"{API_BASE_URL}/knowledge/{KNOWLEDGE_ID}/file/add"

                    resp_add = requests.post(
                        url_add, headers=headers, json=payload, timeout=60
                    )

                    if resp_add.status_code == 200:
                        registered_count += 1
                        # ログが多いと大変なので10件ごとに表示
                        if registered_count % 10 == 0:
                            print(
                                f"[Background] Progress: {registered_count}/{len(files)} files registered."
                            )
                    else:
                        print(
                            f"[Background] Add failed for {filename}: {resp_add.text}"
                        )
                else:
                    print(
                        f"[Background] Upload failed: {filename} ({resp.status_code})"
                    )
            except Exception as e:
                print(f"[Background] Error processing {filename}: {e}")

            # サーバー負荷軽減のため少し待つ
            time.sleep(0.5)

        print(f"[Background] SUCCESS: Processed all files.")
        print(
            f"[Background] Uploaded: {uploaded_count}, Registered: {registered_count}"
        )

    except Exception as e:
        print(f"[Background] CRITICAL ERROR: {str(e)}")

    print("--- [Background] Task Finished ---")


class Tools:
    def __init__(self):
        pass

    def import_wiki_to_rag(self) -> str:
        """
        Wikiの取り込み（1ファイルずつループ登録）をバックグラウンドで開始します。
        """
        # ファイル数をカウント
        file_count = 0
        if os.path.exists(TARGET_DIR):
            file_count = len([f for f in os.listdir(TARGET_DIR) if f.endswith(".md")])

        thread = threading.Thread(target=run_import_task)
        thread.start()

        return f"✅ 全ファイルの登録処理を開始しました。\n{file_count}ファイルあるため、完了まで数分かかります。Dockerのログで進捗を確認できます。"
```

## 6. 今後の展望・拡張性 [#outlook]
- **環境の固定化 (IaC):** `docker-compose.yml` と `start.sh` を整備し、コンテナ再構築時も自動でライブラリやモデルが復元される仕組みを導入済み。
- **Gemの活用:** 今回の構築情報を学習させたカスタムGem「Home Server Architect」を作成し、今後のトラブルシューティングや構成変更の相談役として活用する。
- **さらなる自動化:** 今後は、Wikiの更新をトリガーとした完全無人同期や、Home Assistant等のIoT連携への応用を検討する。