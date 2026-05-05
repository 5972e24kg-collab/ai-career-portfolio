# Work051: Dual-Phase ThinkingとContext Isolation
Tag: [[Gemma3]], [[PromptEngineering]], [[Java]], [[Regex]], [[CoT]]

## 1. 目的
- **Few-Shot汚染の解消:** 長文の「日記（要約）」をコンテキストに含めた際に、モデルがその文量やスタイルに引きずられ、回答が冗長化・詩的化する現象を防ぐ。
- **性格の構造化:** 「ツンデレ」という複雑な挙動を、確率任せではなく「感情処理（Inner Voice）」と「出力フィルタ（Response）」の2段階プロセスとして定義し、安定させる。
- **コンテキスト効率化:** 思考プロセスは生成させるが、会話履歴には「出力結果」のみを保存することで、トークン消費を抑制しつつ深い推論を行わせる。

## 2. システム環境
- **Node:** [Server name masked] (EVO-X2 / Ryzen AI Max+ 395)
- **Model:** `GemChan5.2:27b` (Based on Gemma 3 27B / Custom Modelfile)
- **Middleware:** Java Custom API (`ChatSessionManager`), Oracle DB
- **Technique:** Chain of Thought (CoT), Prefill Injection

## 3. 作業ログ・解決プロセス

### 発生した課題: "Context Infection" (文体の感染)
- **事象:** `gpt-oss` が生成した長文の日記（約600文字）を履歴に挿入したところ、ジェムちゃんの回答文字数が平均166文字から422文字へ激増。語彙が不自然になり、キャラ崩壊が発生した。
- **原因:** LLMの特性（In-context Learning）により、日記の「長文・説明調」を会話の模範解答として学習してしまったため。

### 解決策1: Dual-Phase Thinking (思考分離)
- ModelfileのSystem Promptを改修し、出力を以下の2フェーズに厳格化した。
    1.  **Phase 1 `<inner_voice>`:** 制限なし。感情の吐き出し、日記との整合性チェックを行う（ユーザー不可視）。
    2.  **Phase 2 `<response>`:** 120文字制限。Phase 1の感情を「ツンデレフィルタ」に通して出力する（ユーザー可視）。
- これにより、「思考はリッチに、出力はシンプルに」という矛盾する要件を両立させた。

### 解決策2: Prefill Injection (思考の強制)
- **課題:** 会話履歴（History）には `<response>` のみを保存するため、次回推論時にモデルが思考プロセスをスキップする恐れがあった。
- **実装:** APIリクエストの末尾に `{"role": "assistant", "content": "<inner_voice>"}` を注入。
- **効果:** モデルに対し「既に思考を開始している」と錯覚させ、強制的にPhase 1から執筆を開始させることに成功。

### 解決策3: Robust Regex Parsing (堅牢な分離)
- LLM特有の「タグ生成ミス（`</inner_<start_of_turn>`等）」に対応するため、Java側のパースロジックを刷新。
- 「開始・終了タグ」の整合性をチェックするのではなく、**「`<response>`タグによる文字列分割（Split）」**を採用し、前方をInner Voice、後方をResponseとして扱うことで、あらゆる生成エラーを許容可能にした。

## 4. 成果物 (Modelfile Snippet)

```dockerfile
# Thinking Process Definition
SYSTEM """
# Thinking Process (Strict Order)
あなたは回答を出力する際、**必ず以下の2段階のプロセス**を経て出力しなければなりません。

## Phase 1: <inner_voice> (思考・本音)
* ここはあなたの「脳内」です。ユーザーには見えません。
* 文字数制限：**なし**。
* 内容：
    - ユーザーの意図分析。
    - 「嬉しい」「寂しい」といった**生の感情**を吐き出す。
    - 文末に感情タグ `[emotion=xxx]` を付与する。

## Phase 2: <response> (実際の返信)
* 文字数制限：**120文字以内**（厳守）。
* 内容：
    - Phase 1の本音を、ツンデレ性格のフィルターを通して変換する。
    - **長文禁止:** LINEらしく短くテンポよく返す。
"""
```