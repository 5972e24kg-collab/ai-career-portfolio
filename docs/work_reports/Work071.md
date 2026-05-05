# Work071: Director / Actor分離型チャットボットv2設計と記憶レイヤ再設

Tag: [[LLM]], [[gpt-oss:120b]], [[gemma4:e4b]], [[QLoRA]], [[Oracle]], [[Java]], [[Chatbot]], [[Memory]], [[MultiAgent]]

## 1. 目的

* 既存の初期プロダクトを、`director / actor` 分離型のフレームワーク v2 として再設計する。
* `gpt-oss:120b` を director、`gemma4:e4b` 系を actor として役割分割し、性能特性に合った構成へ移行する。
* actor を「演技専用機」として固定し、将来的なモデル差し替え耐性を高める。
* 長期記憶と短期記憶を分離し、director に渡す素材をローカル環境だけで安定生成できるようにする。
* 実運用前に、actor・director・記憶レイヤの各部品が最低限実用になるかを検証する。

## 2. システム環境

* **Node:** [Host masked] ([Hardware masked] / ROCm環境)
* **Web API:** Java 製自作 WebAPI
* **LLM Director:** `gpt-oss:120b`
* **LLM Actor:** `gemma4:e4b`, `GemChan6.x:e4b (gemma4:e4b QLoRA tuned)`
* **Memory Store:** Oracle
* **Input / Output:** 公式LINE, 音声合成API, Reactアバター
* **Target:** チャットボット「ジェムちゃん」v2 フレームワーク設計

## 3. 作業ログ・解決プロセス

### 発生した課題

* 既存フレームワークは、初期学習用プロダクトとしては有効だったが、責務分離が曖昧で、actor 側に文脈理解・関係性管理・演技・出力形式順守を同時に背負わせていた。
* `gemma4:e4b` は短文演技に強い可能性がある一方、長文文脈保持と多役割同時処理には不安があった。
* actor 出力は `<voice_line>` / `<chat_line>` を軸にした新形式へ移行したかったが、タグ崩れや旧 Modelfile の影響が懸念だった。
* director に渡す `[Long-term Memory]` を ChatGPT 的な手作業要約に依存すると、ローカル完結できない。
* 記憶要約を1層で済ませようとすると、「数日単位の土台」と「今この瞬間のシーン進行」が混ざってしまい、gpt-oss が要点を見失いやすかった。
* Current Scene Summary は会話が進まないと評価しづらく、設計だけでは妥当性判定が難しかった。

### 解決策

* フレームワーク方針を `director = 何を言うか決める / actor = どう言うか演じる` に明確化した。
* actor 用の最小台本スキーマを策定し、`user_message / goal / topic_anchor / topic_mode / surface_tone / subtext / must_include / avoid` で制御する方針に固定した。
* actor 用 system prompt を「演技専用」に全面改修し、`<performance><voice_line><chat_line>...` 形式を最優先出力にした。
* `gemma4:e4b` と `GemChan6.x:e4b` の両方で、同一台本に対する演技テストを実施した。
* 10問規模の台本バリエーション試験を行い、以下を確認した。

  * `subtext` は強めに書いた方が効く
  * `must_include` は概念より表現レベルに落とした方が効く
  * `avoid` は抽象語より具体語が効く
  * `topic_mode` は会話制御に有効
* GemChan 系 tuned モデルはキャラ性が強い反面、旧 Modelfile の影響で出力が崩れやすかったため、新 system prompt ベースへ更新する方針に整理した。
* director は script_json だけを書かせる設計に固定し、system prompt を「自然文フィールドは日本語固定」「must_include は完成セリフ禁止」「avoid は具体語で書く」方向へ修正した。
* 長期・短期記憶を次の2層に分離した。

  * **Long-term Memory:** 3日分の EXTRACTION_REPORT を Java で軽量化し、gpt-oss で圧縮して1日1回生成
  * **Current Scene Summary:** 直近の user input / director script / actor output / 前回 summary を元に、会話進行の短期要約を更新
* Long-term Memory 用には、EXTRACTION_REPORT から残すタグと削るタグを整理し、Java の前処理ルールを確定した。
* Current Scene Summary 用には、専用 JSON フォーマット、system prompt、実行時 prompt を定義し、複数ケースでサンプル生成を確認した。
* 全体評価としては、100点ではないが「監視付きで動かせる初期プロダクト」として Go 判断できる水準まで到達した。

## 4. 成果物

### 4-1. actor 用最小台本スキーマ

```json
{
  "schema_version": "1.0-min",
  "cue_type": "chat_reply",
  "user_message": "",
  "goal": "",
  "topic_anchor": "",
  "topic_mode": "follow_with_soft_detour",
  "surface_tone": "light_tsun",
  "subtext": "",
  "must_include": [],
  "avoid": ["説教", "長文"],
  "voice_style": "soft_teasing",
  "chat_length_max": 90,
  "voice_length_max": 60,
  "end_style": "soft_hook"
}
```

### 4-2. actor 出力形式

```xml
<performance>
  <voice_line>...</voice_line>
  <chat_line>...</chat_line>
  <emotion_tags>
    <emotion primary="..." secondary="..." />
  </emotion_tags>
  <gesture_tags>
    <face>...</face>
    <motion>...</motion>
    <subtitle_mode>...</subtitle_mode>
  </gesture_tags>
</performance>
```

### 4-3. Long-term Memory 出力フォーマット

```json
{
  "schema_version": "1.0-long-term-memory",
  "generated_at": "[Date masked] [Time masked]",
  "source_dates": ["[Date masked]", "[Date masked]", "[Date masked]"],
  "relationship_baseline": {
    "summary": "...",
    "closeness": 0,
    "trust": 0,
    "tension": 0,
    "care": 0,
    "dependency": 0
  },
  "stable_topics": ["...", "..."],
  "persistent_open_threads": ["..."],
  "gem_tendencies": ["...", "..."],
  "user_patterns": ["...", "..."],
  "recall_candidates": ["...", "..."],
  "avoid_old_topics": ["...", "..."],
  "long_term_memory_text": "...",
  "confidence": 0.0
}
```

### 4-4. Current Scene Summary 出力フォーマット

```json
{
  "schema_version": "1.0-current-scene",
  "generated_at": "[Date masked] [Time masked]",
  "source_range": {
    "chat_turn_window": "short description",
    "last_user_message_included": true,
    "director_script_included": true,
    "actor_output_included": true,
    "previous_summary_included": true
  },
  "current_scene": {
    "summary": "...",
    "phase": "...",
    "priority": "..."
  },
  "current_topic": "...",
  "scene_goal": "...",
  "scene_emotional_tone": {
    "user": "...",
    "gem": "...",
    "interaction": "..."
  },
  "next_natural_moves": ["...", "...", "..."],
  "must_preserve": ["...", "...", "..."],
  "should_avoid_now": ["...", "...", "..."],
  "director_scene_text": "...",
  "confidence": 0.0
}
```

### 4-5. 実装方針の最終整理

```text
1. ユーザー入力を受け付ける
2. director が
   - Last User Message
   - Long-term Memory
   - Current Scene Summary
   を読んで script_json を生成する
3. actor が script_json を演技し、<performance> を返す
4. Java が
   - LINE / 音声合成 / アバターへ配信する
   - actor 出力の妥当性を検証する
5. 裏側で Current Scene Summary を更新し、次回 director の材料にする
6. 1日1回、3日分の軽量化 EXTRACTION_REPORT から Long-term Memory を更新する
```

### 4-6. 現段階の評価

```text
- actor: 実用下限を超えた。タグ崩れはあるが許容範囲
- director: 台本 JSON を安定生成できる見通しが立った
- Long-term Memory: director の土台として十分
- Current Scene Summary: 短期進行要約として成立
- 全体: 「研究用試作」から「監視付きで運用可能な初期プロダクト」へ移行可能
```