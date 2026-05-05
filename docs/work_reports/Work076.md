# Work076: Codexバイブコーディング量産工場に向けたGitブランチ運用の習得

Tag: [[Git]], [[Codex]], [[VibeCoding]], [[Branch]], [[React]], [[Workflow]], [[Learning]]

## 1. 目的

* Codexによるバイブコーディングで生成される複数のUIバリエーションを、安全に管理できるGit運用を習得する。
* 従来の「mainにcommitし続けるだけ」の履歴管理から、`branch` / `commit` / `merge` を用いた分岐型の開発運用へ移行する。
* 関係者向けイベント用React 4択クイズアプリを題材に、CodexでUIだけを繰り返し生成し直せる「バリエーション量産工場」を構築する。
* `main` を安定版、`vibe/*` ブランチをAI生成実験場として扱う運用を体得する。
* AIに実装を任せる一方で、人間がGitによる開始地点・保存地点・採用判断を管理する運用思想を確立する。
* 今後、自分自身で再現可能なGit手順として、最低限必要なコマンド群と判断基準を整理する。

---

## 2. システム環境

* **Node:** バイブコーディング専用 Windows 11 Pro PC
* **Target:** 関係者向けイベント用 React 4択クイズアプリ
* **Working Directory:** `[Path masked]`
* **AI Coding Agent:** Codex
* **Version Control:** Git for Windows
* **Repository:** ローカルGitリポジトリ
* **Branch Strategy:**

  * `main`: 安定版・正史・Codex実行前の基準点
  * `vibe/quiz-ui-*`: CodexによるUI生成バリエーション
* **Initial Commit:**

  * `f616bb1 initial import`

---

## 3. 作業ログ・解決プロセス

### 発生した課題

#### 課題A: Codexを回す前にGit運用設計をしていなかった

今回の最も大きな課題は、単に「main上でCodexを実行してしまった」ことではなく、そもそもCodexによる生成作業を始める前に、Git上の管理体制を設計していなかったことである。

Codexは非常に高速に既存Reactアプリを解析し、UI改造を実施できる。一方で、AIによる変更は短時間で多数のファイルに及ぶため、人間側が以下を事前に決めていないと、生成結果の比較・採用・破棄が難しくなる。

* どの状態を安定版とするか
* Codexに作業させるブランチをどこにするか
* 生成結果をどの単位で保存するか
* 複数案をどう比較するか
* 採用案をどのように本流へ戻すか
* 不採用案をどのように破棄するか

今回、初回のCodex実行時にはこれらの運用設計が未整備だった。

これは単なる操作ミスではなく、AI開発を安全に進めるための管理工程が抜けていたという意味で、より本質的な課題である。

---

#### 課題B: main上でCodexを実行していた

実際のGit状態は以下だった。

```powershell
PS [Path masked]> git status
On branch main
Changes not staged for commit:
        modified:   src/App.css
        modified:   src/App.js
        modified:   src/components/ClearScreen.js
        modified:   src/components/ClearScreen.module.css
        modified:   src/components/CorrectScreen.js
        modified:   src/components/CorrectScreen.module.css
        modified:   src/components/IncorrectScreen.js
        modified:   src/components/IncorrectScreen.module.css
        modified:   src/components/QuizScreen.js
        modified:   src/components/QuizScreen.module.css

Untracked files:
        src/components/StartScreen.js
        src/components/StartScreen.module.css
        src/eventConfig.js
```

この時点では `main` ブランチ上にCodex生成後の変更が未commit状態で乗っていた。

ただし、まだcommitしていなかったため、Git履歴としての `main` は汚れていなかった。

```powershell
PS [Path masked]> git log --oneline --decorate -5
f616bb1 (HEAD -> main) initial import
```

このため、未commit変更を新規ブランチへ逃がすことで、安全に回収できる状態だった。

---

#### 課題C: GitをVMwareスナップショットのように理解していた

当初の理解では、Gitのcheckpointやtagは、VMwareなどの仮想ホストにおけるスナップショットに近いものとして捉えていた。

しかし、Gitは仮想マシン全体を復元する仕組みではない。

Gitの実態は以下に近い。

```text
commit
  = ファイル状態の保存点

tag
  = 特定のcommitに貼る固定ラベル

branch
  = commitを指す可動式のしおり

HEAD
  = 現在作業している場所

switch
  = 作業ディレクトリを別のしおりの状態へ切り替える操作
```

つまり、「過去に戻って作業を続ける」場合は、tagや過去commitへ直接作業を続けるのではなく、そこから新しいbranchを作るのが安全な運用となる。

---

#### 課題D: `git log` の見え方で混乱した

`vibe/quiz-ui-01` で作業しているときは、以下のように履歴が見えた。

```powershell
PS [Path masked]> git log --oneline --decorate -5
0d17281 (HEAD -> vibe/quiz-ui-01) Generate quiz UI variant 01
f616bb1 (main) initial import
```

しかし、`main` に戻った後は以下のように `vibe/quiz-ui-01` が表示されなかった。

```powershell
PS [Path masked]> git log --oneline --decorate -5
f616bb1 (HEAD -> main) initial import
```

原因は、通常の `git log` が「現在の `HEAD` から親方向にたどれる履歴」を表示するためである。

`main` から見ると、`vibe/quiz-ui-01` は別世界線の未来側にあるため、通常の `git log` には表示されない。

全ブランチを含む世界線マップを見るには以下を使う。

```powershell
git log --oneline --decorate --graph --all
```

実際の表示は以下。

```powershell
PS [Path masked]> git log --oneline --decorate --graph --all
* 9596c3d (vibe/quiz-ui-02) Generate quiz UI variant 02
| * 0d17281 (vibe/quiz-ui-01) Generate quiz UI variant 01
|/
* f616bb1 (HEAD -> main) initial import
```

この表示により、Gitのbranch構造を「世界線マップ」として直感的に把握できた。

---

#### 課題E: バイブコーディング前にcommitすべきか迷った

`main` から `vibe/quiz-ui-02` を作成した直後、バイブコーディング前にcommitすべきか迷った。

結論として、バイブコーディング前にcommitする必要はない。

理由は、`git switch -c vibe/quiz-ui-02` の時点で、`main` と同じcommitを指す新しいbranchが作成されているためである。

```powershell
git switch -c vibe/quiz-ui-02
```

この時点で作業ツリーがcleanであれば、分岐点は確保されている。

一方、バイブコーディング後は必ずcommitする必要がある。

未commitのまま `main` に戻ると、変更が作業ツリーに残ったまま移動し、main側へ変更を持ち越したように見えて混乱する可能性がある。

---

### 解決策

#### 解決策A: mainを安定版、vibeブランチを実験場とする運用に整理した

今回の運用を以下のように定義した。

```text
main
  = 正史 / 安定版 / Codex実行前に戻れる基準点

vibe/*
  = Codexに自由に作業させる世界線

commit
  = その世界線の保存ポイント

tag
  = 特定commitに貼る固定ラベル

git switch
  = 世界線移動

git log --graph --all
  = 世界線マップ

git merge --ff-only
  = 勝ち筋を正史に昇格する操作
```

これにより、Codexの創造性を活かしながら、人間がGitで安全柵を張る構造を整理できた。

---

#### 解決策B: main上の未commit変更をvibeブランチへ退避した

初回Codex生成結果は、以下の手順で `vibe/quiz-ui-01` へ保存した。

```powershell
git switch -c vibe/quiz-ui-01
git add -A
git commit -m "Generate quiz UI variant 01"
```

実行結果。

```powershell
[vibe/quiz-ui-01 0d17281] Generate quiz UI variant 01
 13 files changed, 923 insertions(+), 207 deletions(-)
 create mode 100644 src/components/StartScreen.js
 create mode 100644 src/components/StartScreen.module.css
 create mode 100644 src/eventConfig.js
```

確認。

```powershell
PS [Path masked]> git status
On branch vibe/quiz-ui-01
nothing to commit, working tree clean
```

この時点で、初回Codex生成結果は `vibe/quiz-ui-01` として安全に保存された。

その後、`main` に戻った。

```powershell
git switch main
```

確認。

```powershell
PS [Path masked]> git status
On branch main
nothing to commit, working tree clean

PS [Path masked]> git log --oneline --decorate -5
f616bb1 (HEAD -> main) initial import
```

これにより、`main` は初期状態のまま汚れていないことを確認できた。

---

#### 解決策C: 2つ目のCodex生成世界線を作成した

次に、`main` を起点として新しい世界線 `vibe/quiz-ui-02` を作成した。

```powershell
git switch -c vibe/quiz-ui-02
```

確認。

```powershell
PS [Path masked]> git status
On branch vibe/quiz-ui-02
nothing to commit, working tree clean

PS [Path masked]> git log --oneline --decorate -5
f616bb1 (HEAD -> vibe/quiz-ui-02, main) initial import
```

この状態でCodexを実行し、2つ目のUIバリエーションを生成した。

今回の生成結果は、可愛い系のモンスターが登場する魔王城デザインとなった。

Codex実行後、以下の状態となった。

```powershell
PS [Path masked]> git status
On branch vibe/quiz-ui-02
Changes not staged for commit:
        modified:   src/App.css
        modified:   src/App.js
        modified:   src/components/ClearScreen.js
        modified:   src/components/ClearScreen.module.css
        modified:   src/components/CorrectScreen.js
        modified:   src/components/CorrectScreen.module.css
        modified:   src/components/IncorrectScreen.js
        modified:   src/components/IncorrectScreen.module.css
        modified:   src/components/QuizScreen.js
        modified:   src/components/QuizScreen.module.css

Untracked files:
        src/components/StartScreen.js
        src/components/StartScreen.module.css
        src/config/
```

これを以下で保存した。

```powershell
git add -A
git commit -m "Generate quiz UI variant 02"
```

実行結果。

```powershell
[vibe/quiz-ui-02 9596c3d] Generate quiz UI variant 02
 13 files changed, 1129 insertions(+), 237 deletions(-)
 create mode 100644 src/components/StartScreen.js
 create mode 100644 src/components/StartScreen.module.css
 create mode 100644 src/config/eventConfig.js
```

最終的な世界線マップ。

```powershell
PS [Path masked]> git log --oneline --decorate --graph --all
* 9596c3d (HEAD -> vibe/quiz-ui-02) Generate quiz UI variant 02
| * 0d17281 (vibe/quiz-ui-01) Generate quiz UI variant 01
|/
* f616bb1 (main) initial import
```

---

#### 解決策D: 量産工場の基本サイクルを確立した

今回の作業を通じて、CodexによるUIバリエーション量産サイクルを以下に固定した。

```text
1. mainに戻る
2. mainがcleanであることを確認する
3. 新しいvibeブランチを作る
4. Codexを実行する
5. 生成結果を確認する
6. git add -A で新規ファイルを含めて登録する
7. commitでその世界線を保存する
8. mainに戻る
9. 必要な回数だけ繰り返す
10. 勝ち筋が決まるまではmainにmergeしない
```

今回の時点では、`vibe/quiz-ui-01` と `vibe/quiz-ui-02` の2つの世界線を作成し、どちらもmainには合流していない。

---

#### 解決策E: Gitを「AI生成時代の管理者ツール」として位置付けた

今回の作業により、CodexのようなAIコーディングエージェントを安全に使うには、人間がGit管理を握る必要があると整理した。

```text
AI / Codex
  = 世界線の中で大量に変更を生み出す作業者

人間 / Git管理者
  = どこから始めるか、どこを保存するか、どれを正史にするかを決める責任者
```

AIに実装を任せる領域が増えるほど、人間側は以下を管理する必要がある。

* 生成開始地点
* 作業ブランチ
* 保存単位
* 採用判断
* 不採用案の破棄
* mainの保護
* 失敗時の復旧経路

今回の学習により、単なるGitコマンドではなく、AI生成時代に必要な「管理運用の型」を得た。

---

## 4. 成果物

### 4.1 Codexバイブコーディング量産工場 Git手順

以下は、今回確立した最低限の運用手順である。

---

### 現在地確認

作業前に必ず確認する。

```powershell
cd [Path masked]

git status
git branch
git log --oneline --decorate -5
```

確認観点。

```text
git status
  On branch がどこかを見る
  working tree clean かを見る
  Untracked files がないかを見る

git branch
  * が現在地

git log --oneline --decorate -5
  HEAD がどこを指しているかを見る
```

全世界線を確認する場合。

```powershell
git log --oneline --decorate --graph --all
```

---

### 既存の未commit変更を新しい世界線へ保存する

main上でCodexを実行してしまった場合、まだcommitしていなければ以下で救出する。

```powershell
git switch -c vibe/quiz-ui-01
git add -A
git commit -m "Generate quiz UI variant 01"
```

意味。

```text
git switch -c vibe/quiz-ui-01
  今の状態から新しい世界線を作って移動する

git add -A
  変更ファイルと新規ファイルをすべてcommit対象に入れる

git commit -m "Generate quiz UI variant 01"
  Codex生成結果を、その世界線の保存ポイントとして確定する
```

---

### mainへ戻る

```powershell
git switch main
```

確認。

```powershell
git status
git log --oneline --decorate -5
```

期待する状態。

```text
On branch main
nothing to commit, working tree clean
```

---

### 新しいCodex生成世界線を作る

```powershell
git switch -c vibe/quiz-ui-02
```

確認。

```powershell
git status
git branch
git log --oneline --decorate -5
```

期待する状態。

```text
On branch vibe/quiz-ui-02
nothing to commit, working tree clean
```

この状態でCodexを実行する。

---

### Codex生成後に保存する

Codex実行後、変更を確認する。

```powershell
git status
```

問題なければ保存する。

```powershell
git add -A
git commit -m "Generate quiz UI variant 02"
```

確認。

```powershell
git status
git log --oneline --decorate -5
git log --oneline --decorate --graph --all
```

---

### 量産サイクル

3案目以降は、mainに戻ってから同じことを繰り返す。

```powershell
git switch main
git status

git switch -c vibe/quiz-ui-03

# ここでCodexを実行

git status
git add -A
git commit -m "Generate quiz UI variant 03"

git log --oneline --decorate --graph --all
```

以後、必要に応じて以下のように増やす。

```text
vibe/quiz-ui-01
vibe/quiz-ui-02
vibe/quiz-ui-03
vibe/quiz-ui-04
```

---

### 勝ち筋をmainへ採用する

今回はまだ実施していないが、採用案が決まった場合は以下を使う。

例: `vibe/quiz-ui-02` を採用する場合。

```powershell
git switch main
git merge --ff-only vibe/quiz-ui-02
```

意味。

```text
mainを、vibe/quiz-ui-02の位置まで進める
```

`--ff-only` は安全装置である。

単純にmainを前へ進められる場合のみ成功し、複雑なmergeが必要な場合は失敗する。

---

### 不採用ブランチを削除する

安全削除。

```powershell
git branch -d vibe/quiz-ui-01
```

強制削除。

```powershell
git branch -D vibe/quiz-ui-01
```

最初のうちは `-D` は慎重に使う。

---

### checkpointを作る

特定時点に名前を付ける場合はtagを使う。

```powershell
git tag checkpoint/quiz-rules-stable-[Date masked]
```

注意点。

```text
tagは空commitではない
tagは今いるcommitに名前付きの付箋を貼るだけ
未commit変更はtagに含まれない
```

checkpointから新しい世界線を作る場合。

```powershell
git switch -c vibe/retry-from-checkpoint checkpoint/quiz-rules-stable-[Date masked]
```

---

### 今回使ったコマンド一覧

```powershell
git status
git branch
git log --oneline --decorate -5
git log --oneline --decorate --graph --all

git switch main
git switch -c vibe/quiz-ui-01
git switch -c vibe/quiz-ui-02

git add -A
git commit -m "Generate quiz UI variant 01"
git commit -m "Generate quiz UI variant 02"

git merge --ff-only vibe/quiz-ui-02
git branch -d vibe/quiz-ui-01
git branch -D vibe/quiz-ui-01

git tag checkpoint/quiz-rules-stable-[Date masked]
git switch -c vibe/retry-from-checkpoint checkpoint/quiz-rules-stable-[Date masked]
```

---

### 4.2 今回作成された世界線

今回の最終状態。

```powershell
PS [Path masked]> git log --oneline --decorate --graph --all
* 9596c3d (vibe/quiz-ui-02) Generate quiz UI variant 02
| * 0d17281 (vibe/quiz-ui-01) Generate quiz UI variant 01
|/
* f616bb1 (HEAD -> main) initial import
```

状態。

```text
main
  = initial import
  = Codex実行前の安定版

vibe/quiz-ui-01
  = Codex生成UI案1
  = 初回生成結果
  = main上で実行してしまった未commit変更を退避して保存

vibe/quiz-ui-02
  = Codex生成UI案2
  = mainから正式に分岐して生成
  = 可愛い系モンスターが出る魔王城デザイン
```

この時点では、まだ勝ち筋をmainへmergeしていない。

---

### 4.3 運用ルール

今回の学習を踏まえ、Codexバイブコーディング時の運用ルールを以下に定める。

```text
1. main上でCodexを回さない
2. Codex実行前に必ずgit statusを見る
3. mainは常に安定版として扱う
4. Codex作業は必ずvibe/*ブランチ上で行う
5. Codex生成後は必ずcommitする
6. 新規ファイル漏れを防ぐため git add -A を使う
7. 複数案を作る場合は、毎回mainから新しいvibeブランチを切る
8. 採用案が決まるまではmainにmergeしない
9. 世界線確認には git log --graph --all を使う
10. 勝ち案だけを git merge --ff-only でmainへ戻す
```

---

### 4.4 今回の学習結果

今回の学習テーマは「Gitの動きを体得する」だった。

達成したこと。

```text
- mainとbranchの関係を体感した
- HEADの意味を理解した
- commitをcheckpointとして扱う感覚を得た
- branchを世界線として捉えると理解しやすいことが分かった
- 通常のgit logと、--graph --allの違いを理解した
- main上で作業してしまった場合の救出方法を経験した
- Codex生成結果を複数branchとして保存できた
- mainへ戻ることで、初期状態から再生成できることを確認した
- CodexによるUIバリエーション量産の基本サイクルを確立した
```

今回の最大の収穫。

```text
AIが優秀なプログラマーになるほど、
人間はGitによる管理者である必要がある。
```

Codexは短時間で大量の変更を行える。
だからこそ、人間は以下を管理しなければならない。

```text
- どこから始めるか
- どこで保存するか
- どれを採用するか
- どれを捨てるか
- どの状態を安定版と呼ぶか
```

今回の結論。

```text
Codexによるバイブコーディング環境は、
Gitブランチ運用と組み合わせることで、
単発の自動生成環境から、
複数案を安全に量産・比較・採用できる開発工場へ進化する。
```

---

### 4.5 今後の展望

今後は、以下を実践しながらGit運用を身体化する。

```text
- vibe/quiz-ui-03 以降の生成を繰り返す
- 生成案ごとの特徴を記録する
- 実際に勝ち案をmainへmergeする
- 不採用branchを削除する
- 必要に応じてtag/checkpointを使う
- 自筆のWiki手順書を作成する
```

将来的には、以下のような運用へ発展させる。

```text
main
  = 安定版

vibe/*
  = AI生成UI案

fix/*
  = 人間による軽微修正

release/*
  = 関係者向けテスト環境・本番反映候補

checkpoint/*
  = イベント前・仕様確定時点の固定ラベル
```

ただし、現段階では複雑なGit運用を増やしすぎない。

まずは以下の基本型を反復する。

```text
mainに戻る
↓
vibeブランチを切る
↓
Codexを回す
↓
commitする
↓
世界線マップを見る
↓
mainに戻る
```

---

※ 構成は、過去レポート「Codexバイブコーディング環境構築とクイズゲームUI改造検証」の章立て・記録粒度を参考にした。