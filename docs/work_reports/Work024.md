# Work024: Semantic Chunkingとbge-m3最適化

Tag: [[Java]], [[RAG]], [[Ollama]], [[Strix Halo]]

## 1. 目的

* **Embedding:** `bge-m3` を用いたベクトル化環境の構築。
* **Chunking:** コードブロックや会話文脈を維持した、LLMフレンドリーなデータ分割の実装。

## 2. システム環境

* **Node:** Node B (EVO-X2 / Ryzen AI Max+ 395)
* **Target:** Open WebUI (RAG), Ollama, Java Batch (Unit A)
* **Model:** `BAAI/bge-m3` (Embedding)

## 3. 作業ログ・解決プロセス

### 課題A: Embedding処理の長時間化とエラー

* **事象:** `bge-m3` 設定後、ファイルの埋め込み処理が長時間完了しない、またはエラーになる。
* **原因:**

1. **次元数の不一致:** 既存DBが `384` 次元で作成されており、`bge-m3` の `1024` 次元データを受け付けなかった。
2. **処理負荷に関する推定:** 当時の実行環境では、BERTアーキテクチャの計算においてGPUオフロードが十分に効いていない、または一部に限られているように見え、CPU側の処理がボトルネックになっている可能性があった。Chunk Size 4000では処理負荷が高かった。

* **解決策:**
* ナレッジ（コレクション）の再作成による次元数リセット。
* Chunk Size を `2048`、Overlap を `200` に設定し、処理負荷と文脈保持のバランスを取った。

### 課題B: テキスト分割によるコード破壊

* **事象:** 単純な文字数分割では、コードブロックがチャンク境界で分断され、シンタックスハイライトが機能しなくなる。
* **解決策 (Java実装):**
* `ChunkSplitter.java` にて、コードブロックの状態（開閉・言語）を追跡。
* チャンクをまたぐ際、**前半には閉じタグを、後半には言語指定付きの開きタグを自動付与**するロジックを実装。

## 4. 成果物 (Key Logic Snippet)

````java
// コードブロック整合性維持ロジック（ChunkSplitter.javaより抜粋）
boolean overlapHasOpener = overlapText.contains("```" + currentCodeLanguage) 
                            || (currentCodeLanguage.isEmpty() && overlapText.contains("```"));

// オーバーラップ内に開きタグが無いのに、論理的にコードブロック内である場合 -> 補完する
if (insideCodeBlock && !overlapHasOpener) {
    // 次のファイルの先頭に、前のファイルと同じ言語指定(java等)を付与する
    currentBuffer.append("```").append(currentCodeLanguage).append("\n");
}

````
