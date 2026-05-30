# Work061: 3Dアバター化に向けたアーキテクチャ設計と技術選定

Tag: [[Architecture]], [[Three.js]], [[React Three Fiber]], [[VRM]], [[UX]]

## 1. 目的

* 専属マスコット「ジェムちゃん（Gemma 3:27b）」に視覚的な肉体（VRM）を与え、日常の作業環境に常駐させるための最適なアプローチを選定する。
* ユーザーの既存スキル（React）と既存のローカル実行環境（GPUサーバー / 音声合成ノード）を最大限に活かした、拡張性の高いシステム構成を定義する。
* 開発の順序と、各フェーズに潜む技術的な罠（CORS、ブラウザの音声再生制限など）を事前に洗い出し、ロードマップを策定する。

## 2. システム環境

* **Client (Controller):** 開発用クライアントPC / IntelliJ IDEA
* **Frontend Hosting:** ローカルGPUサーバー（[server name masked]） / Nginx (Docker)
* **AI Brain (WebSocket):** ローカルGPUサーバー / `gpt-oss:120b` (Director) & `Gemma 3:27b` (Actor)
* **Vocal Cord & Mouth:** ローカルWindows音声合成ノード / `style-bert-vits2` & `mouth.py`（口パク制御）

## 3. 作業ログ・解決プロセス

### 発生した課題

* ジェムちゃんの3Dモデル（VRM）を稼働させるにあたり、「Unityによるネイティブアプリ」か「Three.jsによるWebアプリ」のどちらを採用すべきか、学習コストと将来の拡張性の観点から判断基準が不足していた。
* 特にThree.jsを用いた場合の、LLMとの非同期通信やモーション制御の具体的な実現可能性が不透明であった。

### 解決策

* 両者の技術的特性を比較分析。ユーザーが「Reactの実装経験を持つ」という既存スキルを活かし、**Three.js（React Three Fiber）アプローチの採用**を決定した。
* Webベースとすることで、将来的な「自分新聞」等のUI統合が容易になり、Nginxのリバースプロキシを活用することで、ローカル/グローバルアクセスの切り分けも可能になることを確認。
* 開発中のハマりポイント（非同期ロード、骨格の互換性、ブラウザのAutoplay制約、Mixed Content制約）を予見し、これらを回避・解決しながら段階的に進める「4フェーズの実装ロードマップ」を策定した。

## 4. 成果物

「Project: Personal AI Command Center」の実装ロードマップ（Phase 1 〜 Phase 4）
※ 詳細は `Development_Roadmap.md` の Phase 6 として定義済み。
