# Work063: Neural Stream (SSE) の統合と動的表情・字幕UIの実装

Tag: [[React]], [[Zustand]], [[SSE]], [[Java]], [[UI/UX]]

## 1. 目的

* ローカルLLM実行ノードで稼働するAI（脳）からの推論結果（感情タグ・テキスト）を、クライアントPC上の3Dアバター（肉体）へリアルタイムに同期させる。
* 既存のJava資産を活用し、堅牢で低遅延な非同期プッシュ通信基盤を確立する。

## 2. システム環境

* **Brain (Backend):** ローカルLLM実行ノード - `GemchanStreamServlet` ([port masked])
* **Body (Frontend):** クライアントPC - React, Zustand, `@react-three/fiber`

## 3. 作業ログ・解決プロセス

### 発生した課題

* AIの非同期な応答を、いかにして無駄なオーバーヘッドなくReactの3D描画ループ（状態管理）に割り込ませるか。

### 解決策

* **Server-Sent Events (SSE) の採用:** 双方向のWebSocketではなく、単方向プッシュに特化したSSEを採用。既存の自作監視ツール (`LogStreamServlet`) の仕組みを流用・拡張し、JSONペイロードを配信するエンドポイントを構築。
* **カスタムフックの作成:** React側に `useGemchanStream.js` を実装。SSEを受信し、データタイプが `control` の場合、即座にZustandのStore (`emotion`, `currentText`) を上書きするロジックを組んだ。
* **UIコンポーネントの拡張:** Zustandの `currentText` を監視し、テキストが存在する場合のみ画面下部にオーバーレイ表示される字幕UIを実装した。

## 4. 成果物

* 外部からのHTTP POSTリクエストによって、3D空間のVRMモデルの表情（BlendShape）が動的に変化し、字幕が同期して表示されるパイプラインを構築した。
