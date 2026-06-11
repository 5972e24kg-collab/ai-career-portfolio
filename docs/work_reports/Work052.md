# Work052: 運用監視業務のゲーミフィケーション化

Tag: [[React]], [[CSS]], [[UX_Design]], [[Gemini_Log]]

## 1. 目的

* **監視体験のモチベーション向上:** gpt-ossの推論ログ（日記）閲覧時のUXを改善し、運用者の能動的な監視行動を促す。
* **可読性向上:** LLM特有の長文出力を、人間が自然に読める形式（段落分け）へ整形する。
* **没入感の演出:** 「ジェムちゃん」のキャラクター性をUIに反映させ、システムへの愛着形成につなげる。

## 2. システム環境

* **Node:** Localhost (React App)
* **Target:** `src/view/` (DiaryNote, SideBar)
* **Library:** Google Fonts (Zen Kurenaido)

## 3. 作業ログ・解決プロセス

### 発生した課題

* **無機質なUI:** 初期状態ではフレックスボックスによる単純配置のみで、日記としての質感が弱かった。
* **可読性の低さ:** gpt-ossが出力するテキストには改行が含まれておらず、長文の解読が困難だった。

### 解決策

1. **アナログ質感の再現 (CSS):**

   * `linear-gradient` を用いて、CSSのみでノートの罫線を描画。
   * `line-height` と `background-size` を同期させ、文字が罫線上に乗るように調整。
   * Google Fonts「Zen Kurenaido」を採用し、手書き風の筆跡を再現。
2. **サイドバーの改修:**

   * 日付リストを「マスキングテープで貼られたメモ」風に装飾し、選択しやすいインタラクションを追加。
3. **テキスト整形ロジック (React):**

   * 句点（`。`）をデリミタとしてテキストを分割し、`<p>`タグでラップしてレンダリングする関数を実装。

## 4. 成果物

### DiaryNote.css (抜粋: ノートの質感)

```css
.DiaryNote_Background {
    background-color: #fffcf2;
    /* 罫線の描画: 2.5rem間隔 */
    background-image: linear-gradient(transparent 95%, #e0c0c0 96%);
    background-size: 100% 2.5rem;
    
    font-family: 'Zen Kurenaido', sans-serif;
    font-size: 1.5rem;
    line-height: 2.5rem; /* 罫線と同期 */
    color: #554444;
}

```

### Text Formatter (React Logic)

```javascript
const formatDiaryText = (text) => {
  if (!text) return null;
  // 句点で分割し、段落として整形
  return text.split('。').filter(Boolean).map((sentence, index) => (
    <p key={index} style={{ margin: '0 0 1.5rem 0', textIndent: '1em' }}>
      {sentence}。
    </p>
  ));
};

```
