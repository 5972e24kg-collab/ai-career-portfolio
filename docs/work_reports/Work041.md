# Work041: Time GroundingとEmotion Parsing

Tag: [[Java]], [[Gemma3]], [[PromptEngineering]], [[Grounding]], [[Regex]]

## 1. 目的

* **時間軸の共有:** LLM (Gemma 3) に「現在」を認識させ、ユーザーとの共時性を確立する。
* **感情の可視化:** LLMが生成するト書き（Stage Direction）を構造化データとして抽出し、UI表現（アイコン変更）に利用する。
* **哲学の転換:** 「AIの回答精度は、入力された情報の解像度に依存する」という Grounding の原則をシステム設計に適用する。

## 2. システム環境

* **Node:** 検証用ローカルLLM環境（VRAM 96GB級）
* **Model:** `GemChan5.1:27b` (Custom LoRA)
* **Middleware:** Java Custom API (`ChatSessionManager`), Oracle DB
* **Frontend:** LINE Bot / WebUI

## 3. 作業ログ・解決プロセス

### 課題: "Lost in Time" (日付認識の失敗)

* **事象:** `Gemma 3` が Open WebUI から暗黙的に渡される日付情報を十分に参照できず、学習データ上の日付（2024年）を回答するケースがあった。
* **原因:** System Prompt 内の人格定義（ツンデレ設定）が多く、末尾のメタ情報への注意が安定しなかった可能性がある。
* **解決策 (Physical Injection):**

  * AIの「推論能力」だけを疑うのではなく、「入力情報」を疑うアプローチへ変更。
  * Java側でユーザーメッセージの直前に `[System Info] Current Date: ...` を強制挿入。
  * 結果、このケースではAIが「目の前にある事実」として時刻を参照しやすくなり、日付に関する応答が安定した。

### 課題: 感情表現の抽出

* **事象:** `（顔を赤らめて）` や `……(呆れ)` といった表記揺れを含むト書きを、プログラムで正確に切り出す必要があった。
* **解決策 (Regex Parsing):**

  * 行頭のゴミ（三点リーダーや空白）を許容し、全角半角カッコに対応する正規表現を実装。

## 4. 成果物 (Core Implementation)

### A. Time & Context Injection (Java)

```java
// "Grounding" Philosophy Implementation
public String injectContext(String userMessage) {
    LocalDateTime now = LocalDateTime.now();
    long hoursSinceLastTalk = conversationDao.getHoursSinceLastTalk();
    
    StringBuilder sb = new StringBuilder();
    sb.append("[System Info] Current Date: ").append(now.toString());
    
    // 時間差分による応答トーン制御（文脈の提供）
    if (hoursSinceLastTalk > 72) {
        sb.append("\n[System Warning] Long interval since last conversation. Adjust response tone accordingly.");
    }
    
    return sb.toString() + "\n\n" + userMessage;
}

```

### B. Robust Emotion Parser (Java)

```java
// 行頭の記号を無視してカッコ内を抽出
private static final Pattern EMOTION_PATTERN = Pattern.compile("^[…\\.\\s・]*[（\\(](.*?)[）\\)]");

public BotResponse parse(String raw) {
    Matcher m = EMOTION_PATTERN.matcher(raw);
    if (m.find()) {
        String emotion = m.group(1); // "顔を赤らめて" 等
        String text = m.replaceFirst("").trim(); // 本文のみ抽出
        return new BotResponse(text, mapToIconCode(emotion));
    }
    return new BotResponse(raw, "normal");
}

```

## 5. 結論

* 今回のような日付誤認は、モデルの能力不足だけでなく、ユーザー（システム）側の「情報提供不足」や「提示位置の悪さ」によって発生する場合がある。
* 「窓のない部屋の賢者」に対し、正確な時計と状況報告書（Prompt）を差し入れることで、モデルは根拠ある応答を返しやすくなる。
