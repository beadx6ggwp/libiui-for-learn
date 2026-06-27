# Repo State Cleanup Strategy

## Question

目前第一次 commit 處理得比較隨意：把 Windows build fix、README 教學、個人學習方向混在一起。現在需要理解：

- 目前 repo 到底是什麼狀態？
- 為什麼會覺得混亂？
- 有哪些整理路線？
- 哪些適合 upstream PR，哪些適合 learning fork？
- 什麼情況下要保留、拆分、重設或另外開 branch？

如果 GitHub fork / `origin` / `upstream` / Pull Request 的整體概念還不熟，先讀：

```text
notes/0626_2320_github-fork-pr-model.md
```

## Current State

目前狀態已經整理過一次。

第一層：`main` 已經回到 upstream baseline：

```text
main          -> eb8d9c8
origin/main   -> eb8d9c8
upstream/main -> eb8d9c8
```

也就是本地 `main`、GitHub fork 的 `origin/main`、原專案的 `upstream/main` 目前都指向：

```text
eb8d9c83f828ac50412d7d51e7c52b2678c09450
Merge pull request #30 from sysprog21/rendering
```

第二層：第一次嘗試 commit 已經從 `main` 移出，但有被 learn branch 保存：

```text
learn/windows-first-attempt
origin/learn/windows-first-attempt
  -> e722b54d289b3a833ba5eafde2ce60668bbfcbcf
```

這個 commit 是：

```text
e722b54 Add Windows build instructions and documentation
```

它相對 `upstream/main` 改了：

```text
README.md       | 119 insertions
tests/example.c | 13 insertions, 1 deletion
```

也就是：

- `README.md`：個人 Windows/MSYS2 UCRT64 教學、環境說明、Pixel-Renderer 後續想法。
- `tests/example.c`：真正可能適合 upstream 的 Windows build compatibility fix。

因為 `e722b54` 仍然被 local 和 remote 的 `learn/windows-first-attempt` 指著，所以它不是孤兒 commit。jserv 在該 commit 下的 comment 也仍有公開 branch context 可以追。

第三層：目前還沒進 git 的 learning files：

```text
AGENTS.md
notes/AGENTS.md
notes/0626_1705_project-thoughts-plan-questions.md
notes/0626_1723_upstream-pr-submission-plan.md
notes/0626_1726_jserv-pr-guidelines.md
```

這些是你自己的協作規範與思考紀錄，目前只是 untracked，不會出現在 upstream PR，除非你手動 `git add`。它們也不會被 `git reset --hard` 刪掉；要避免的是 `git clean -fd`。

## Why It Feels Confusing

一開始混亂不是因為 git 壞掉，而是因為不同用途的東西被放在同一條 `main` 線上：

```text
upstream-facing code fix
personal README teaching
learning notes
Codex collaboration rules
future PR plan
```

但 open-source PR 需要的形狀是：

```text
one problem -> one root cause -> one small patch -> clear validation
```

所以問題本質是「用途沒有分層」。目前已經先做了第一步修正：`main` 回到 upstream baseline，第一次嘗試移到 `learn/windows-first-attempt` 保存。

## Vocabulary

### `upstream/main`

`sysprog21/libiui` 的正史。所有要送 upstream PR 的 branch，都應該從這裡開始。

### `origin/main`

你的 fork：`beadx6ggwp/libiui-for-learn`。現在 `origin/main` 已經回到 `upstream/main`，而第一次嘗試內容由 `origin/learn/windows-first-attempt` 保存。

### local `main`

你現在工作區所在的本地 branch。目前它追蹤 `origin/main`，且已經和 `upstream/main` 一樣乾淨。

### PR branch

為某一個 upstream PR 專門開的乾淨 branch，例如：

```text
fix-msys2-ucrt-clock
```

這種 branch 應該從 `upstream/main` 開，不應該從已混入學習內容的 `main` 開。

## Core Rule

把 repo 分成三個用途：

```text
upstream/main
  upstream baseline

origin/main or learn/*
  learning fork, notes, experiments

fix-* / ci-* / docs-* branches
  clean upstream PR branches
```

這樣你不需要把所有東西都整理成同一條「完美 main」。你只要確保每次 PR branch 是乾淨的。

## Normal Developer Baseline

先不管你目前的 `e722b54`，一般開源貢獻者最常被建議的標準流程是：

```text
fork 只當作推送權限所在
main 保持接近 upstream
每個 PR 用一條短生命週期 feature branch
notes / experiments 不混進 PR branch
```

換成實際分支模型：

```text
upstream/main
  source of truth

origin/main
  mirror of upstream/main, or at least kept clean enough to branch from

fix-something
  one focused PR branch

learn/*
  optional personal notes and experiments
```

標準操作會像這樣：

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main

git switch -c fix-specific-bug
# edit only the files needed for that bug
git add <files>
git commit -m "Fix specific bug"
git push origin fix-specific-bug
```

然後在 GitHub 開 PR：

```text
base:    sysprog21/libiui:main
compare: beadx6ggwp/libiui-for-learn:fix-specific-bug
```

這套做法背後有幾個原因：

- `main` 保持乾淨，代表任何新 PR 都能從穩定起點開始。
- 一個 feature branch 只處理一件事，reviewer 才能快速判斷問題與解法。
- PR 合併後，feature branch 可以刪除，不會長期累積雜訊。
- 個人筆記、失敗嘗試、環境紀錄可以存在，但應該放在 `learn/*`、另一個 repo、或本機 notes，而不是混進 PR branch。

更嚴格的開發者會這樣要求自己：

```text
main never contains personal experiments.
PR branches are disposable.
Every PR branch starts from current upstream/main.
Every PR has one purpose.
Every commit should be explainable without reading your local notes.
```

這是「理想標準流程」。它適合你之後養成習慣，但它不是說你現在一定要立刻把 fork `main` reset 掉。現在的整理策略是補救現況；這一節描述的是未來預防混亂的 baseline。

## Normal Workflow Variants

不同開發者會依照風險和習慣選不同變體。

### Variant 1: Clean Fork Main

這是最常見、最容易教學的做法。

```text
origin/main ~= upstream/main
每個 PR 從 main 開 branch
個人實驗不要進 main
```

優點：

- 心智模型最簡單。
- GitHub fork 頁面看起來乾淨。
- 不容易不小心把 notes 帶進 PR。

缺點：

- 個人學習紀錄要放別處。
- 不適合把 fork 當研究筆記本。

### Variant 2: Fork Main As Learning Branch, PR Branches From Upstream

這比較符合你目前的需求。

```text
origin/main = learning fork
upstream/main = PR source of truth
fix-* = clean PR branch from upstream/main
```

優點：

- 可以保留學習紀錄。
- 不必重寫已推送的 fork main。
- 仍然能產出乾淨 upstream PR。

缺點：

- 你必須非常清楚：不能從 `main` 開 PR。
- 開 PR 前要固定檢查 diff。

### Variant 3: Separate Worktree

比較專業、隔離性最好。

```text
libiui-for-learn/
  notes and experiments

libiui-pr-xxx/
  clean PR workspace
```

優點：

- learning workspace 和 PR workspace 完全隔離。
- 幾乎不會被 untracked notes 干擾。

缺點：

- 多一個資料夾，多一點操作成本。

### Variant 4: Separate Repository For Notes

有些人會完全不把 notes 放在 fork 裡，而是放到：

```text
libiui-notes
obsidian vault
personal blog repo
```

優點：

- code repo 永遠乾淨。
- notes 可以跨專案累積。

缺點：

- notes 和 code diff 的連結需要自己維護。
- Codex 或其他工具讀 context 時，可能要額外打開另一個位置。

## What I Would Recommend If Starting Fresh

如果今天重新開始，沒有 `e722b54`，我會建議：

```text
1. fork sysprog21/libiui
2. clone your fork
3. add upstream remote
4. keep main clean and synced with upstream/main
5. create learn/notes for personal notes
6. create one fix-* branch per upstream PR
```

具體是：

```bash
git clone https://github.com/beadx6ggwp/libiui-for-learn.git
cd libiui-for-learn
git remote add upstream https://github.com/sysprog21/libiui.git
git fetch upstream
git switch main
git merge --ff-only upstream/main
```

做 notes：

```bash
git switch -c learn/notes
git add notes/
git commit -m "Add libiui learning notes"
git push origin learn/notes
```

做 PR：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
# edit tests/example.c only
git add tests/example.c
git commit -m "Fix MSYS2 UCRT64 clock demo build"
git push origin fix-msys2-ucrt-clock
```

這樣 fork 同時有兩種價值：

```text
learn/notes
  保存你的學習與推理

fix-msys2-ucrt-clock
  給 upstream review 的乾淨修正
```

而 `main` 不承擔任何個人歷史，只當乾淨起點。

## What I Recommend For You Now

因為你現在已經有 `e722b54`，我不會建議立刻照 fresh-start baseline 強行 reset。更務實的做法是：

```text
短期：採用 Variant 2
中期：需要時用 worktree 隔離 PR
長期：等你熟悉後，再決定 main 是否要回到 Clean Fork Main 模式
```

也就是：

```text
現在先不要修歷史。
之後所有 PR branch 從 upstream/main 開。
notes 可以 commit 到 learn/notes。
等第一個 PR 做完，再決定是否清理 fork main。
```

這不是因為 clean main 不好，而是因為現在最重要的是先把「PR 應該乾淨」和「learning history 可以保留」這兩件事分開。

## Situation A: I Want To Submit An Upstream PR Soon

這是最推薦、風險最低的路線。

現在 `main` 已經回到 `upstream/main`，但仍建議每次明確從 `upstream/main` 開新 branch：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
```

然後只重做 `tests/example.c` 的最小 patch。完成後檢查：

```bash
git diff upstream/main..HEAD --stat
git diff upstream/main..HEAD -- README.md notes AGENTS.md
```

理想結果：

```text
tests/example.c | small diff
```

第二個指令應該沒有輸出。這代表 README、notes、AGENTS 沒混進 PR。

這條路的好處：

- 不依賴 fork `main` 是否乾淨。
- 不會丟掉 learning notes。
- 不會讓 upstream reviewer 看到 personal docs。
- 第一次 commit 的混亂不影響 PR。

## Situation B: I Want To Preserve My First Attempt

如果你覺得 `e722b54` 有紀念或學習價值，可以保留它。

目前它已經在：

```text
local main
origin/main
```

你可以把這條線視為：

```text
learning history
```

也可以另外做一個明確 branch name：

```bash
git branch learn/windows-first-attempt main
```

這只是建立指標，不會改檔案，也不會改歷史。之後看到它就知道：這是第一次嘗試，不是 upstream PR branch。

## Situation C: I Want To Split The Bad Commit Into A Clean PR

如果你想從 `e722b54` 裡只拿 `tests/example.c`，可以用 `cherry-pick -n`，但要刪掉不該進 PR 的 `README.md` change。

概念流程：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
git cherry-pick -n e722b54
git restore --source=upstream/main --staged --worktree README.md
git status
git diff --stat
```

這樣會把 `e722b54` 的變更先套進 working tree，但把 `README.md` 回復成 upstream 版本，只留下 `tests/example.c`。

但注意：這只是機械拆分。正式 PR 前還要手動整理 `tests/example.c`：

- 把繁中 code comment 改成英文。
- 移除 trailing whitespace。
- 確認 helper 命名與 style 符合 upstream。
- 補 exact build failure 和 validation。

這條路適合你想快速重用已寫 patch，但不適合無腦直接提交。

## Situation D: I Want My Fork `main` To Become Clean Again

這件事目前已經做完：`main` 和 `origin/main` 都已回到 `upstream/main`。

以下保留當作「如果未來再次需要清 fork main」的操作紀錄。

目標是讓你的 fork `main` 回到 `upstream/main`，把 `e722b54` 從 `origin/main` 移除。

安全前提：

- 先備份目前狀態。
- 確認沒有其他人依賴你的 fork `main`。
- 接受需要 force push。

概念流程：

```bash
git branch learn/windows-first-attempt main
git switch main
git reset --hard upstream/main
git push --force-with-lease origin main
```

這會重寫 `origin/main` 歷史。因為它有破壞性，不建議現在做。你的 fork 是個人 fork，技術上可以，但目前沒有必要。

更穩的做法是：保留 `main`，PR branch 一律從 `upstream/main` 開。

## Situation D2: The Commit Already Has Maintainer Comments

`e722b54` 在 GitHub commit page 上有 jserv 的 commit comment：

```text
Please consider to submit pull request(s) for Windows support.
```

這會讓問題變成：已經有人在 commit 底下留言，還能不能 force push？

答案分三層。

### Technically: 可以

`beadx6ggwp/libiui-for-learn` 是你的 fork。你有權限改自己的 `origin/main`，所以技術上可以：

```bash
git push --force-with-lease origin main
```

但「可以」不代表「現在應該做」。

### GitHub UX: comment 會留在舊 commit 上

GitHub commit comment 是綁在 commit SHA 上。jserv 的留言不是抽象地綁在 `main` branch 上，而是綁在：

```text
e722b54d289b3a833ba5eafde2ce60668bbfcbcf
```

如果你 force push，把 `origin/main` 改成不再包含 `e722b54`，那麼：

- 留言不會自動搬到新的 commit。
- 舊 commit page 可能仍可用 URL 開到。
- 但它不再是 `origin/main` 歷史的一部分。
- 這個上下文會變得比較難找。

所以 force push 會讓「jserv 為什麼叫我發 PR」的公開脈絡變弱。

### Collaboration: 目前不需要

這個 comment 不是 PR review，也不是要求你修改某一行。它的意思比較像：

```text
你這個 Windows support 方向可以整理成 pull request(s)。
```

所以你不需要改寫 `e722b54` 才能回應他。更好的做法是：

```text
保留 e722b54 作為 first attempt / public context
從 upstream/main 開乾淨 PR branch
在 PR body 裡說明這是整理後的 Windows support fix
```

也就是：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
```

不要從目前 `main` 直接開 PR。

### If You Later Want To Clean Main

如果未來你真的想讓 fork `main` 回到乾淨狀態，先保留一個 branch 指向 `e722b54`：

```bash
git branch learn/windows-first-attempt e722b54
git push origin learn/windows-first-attempt
```

這樣即使之後 force push `origin/main`，舊 commit 仍然有 branch 指著它：

```text
learn/windows-first-attempt -> e722b54
```

jserv 的 commit comment 也比較不會變成孤兒上下文。

### Practical Rule

```text
有人已經在 commit 上留言
  不代表不能 force push
  但代表你應該先問：這個公開上下文有沒有保存價值？
```

對目前這個 repo，我的判斷是：

```text
不要 force push main。
保留 e722b54。
另開乾淨 PR branch。
等 PR 流程跑過一次，再決定是否整理 fork main。
```

## Situation E: I Want Notes To Be Version Controlled

如果你想把 `notes/` 當作這個 learning fork 的重要資產，可以 commit 它們，但不要放進 upstream PR branch。

可選策略：

```bash
git switch -c learn/notes main
git add AGENTS.md notes/
git commit -m "Add learning notes and repository guidelines"
git push origin learn/notes
```

這會讓 notes 有版本紀錄，但 branch 名稱明確告訴自己：這不是 upstream PR。

優點：

- 思考紀錄不會丟。
- Codex 後續能讀到一致的 note system。
- learning fork 有自己的價值。

缺點：

- 如果你從這個 branch 開 PR，就會把 notes 混進去。
- 所以仍然要記住：upstream PR branch 從 `upstream/main` 開。

## Situation E2: Do Notes Branches Need To Stay Synced?

直覺上會覺得：如果 `notes/` 放在 `learn/notes` branch，那是不是每次 code branch 變動，都要把 notes branch 同步過去？

答案是：**不一定**。要看這個 notes branch 的用途。

### Case 1: Standalone Learning Notes

如果 `notes/` 是概念、PR 策略、GitHub fork 心智模型、jserv PR 規範這類內容，它不需要跟每個開發 branch 同步。

它比較像獨立筆記本：

```text
learn/notes
  records thinking, plans, concepts, PR strategy

fix-msys2-ucrt-clock
  clean code branch for upstream PR
```

這兩條線可以分開長。你平常要讀 notes 時，回到 `learn/notes` 或在目前 workspace 保持 untracked notes 都可以。你不需要把 `learn/notes` merge 進 `fix-*` branch，因為那會污染 PR。

適合放在 standalone notes 的內容：

- Git / fork / PR 概念。
- upstream review 規範。
- 學習計畫。
- 決策紀錄。
- 對專案架構的閱讀筆記。

### Case 2: Code-coupled Notes

如果 note 是某個具體 code change 的設計文件，而且 PR 真的需要它，例如：

```text
docs/windows-msys2.md
docs/rendering-benchmark.md
```

那它就不是純 learning note，而是 PR artifact。這時應該放在該 PR branch 裡，和 code 一起 review。

例子：

```text
fix-msys2-ucrt-clock
  tests/example.c
  docs/windows-msys2.md   only if upstream should review this doc
```

這種文件才需要跟開發 branch 同步，因為它本身就是開發成果的一部分。

### Case 3: Notes Need Latest Code Context

有時 notes 是專案閱讀紀錄，需要偶爾更新到最新 upstream，避免路徑或 code reference 過時。這時可以偶爾同步，不必每次同步。

概念上是：

```bash
git switch learn/notes
git fetch upstream
git merge upstream/main
```

但這只在你真的需要讓 notes branch 包含最新 code tree 時才做。它不是每天必做的機械動作。

### Why Constant Sync Is Usually Wrong

如果你把 `learn/notes` 一直 merge 到所有開發 branch，會造成：

- PR branch 帶入 personal notes。
- diff 變髒，reviewer 看到不相關檔案。
- 你開始把「學習紀錄」和「提交成果」混在一起。

這正是目前要避免的問題。

### Better Mental Model

不要把 `learn/notes` 想成所有 branch 都要繼承的基底。

應該想成：

```text
learn/notes
  independent notebook branch

fix-*
  disposable PR branches

main
  either learning fork branch or clean mirror, depending on policy
```

如果你需要同時看 notes 和做 clean PR，最佳方法不是 merge notes，而是：

```text
Option A: use two worktrees
Option B: keep notes in separate repo / Obsidian
Option C: keep notes untracked locally
```

### Practical Rule

```text
notes 是給自己看的
  不要同步進 PR branch

docs 是給 upstream review 的
  放進 PR branch

notes 需要更新 code reference
  偶爾從 upstream/main merge 到 learn/notes
```

所以你的疑問答案是：

```text
不需要一直跟開發 branch 同步。
只有 code-coupled docs 才需要跟該開發 branch 走。
純 learning notes 應該保持獨立，避免污染 PR。
```

## Situation F: I Want Notes To Stay Local Only

如果你不想讓 `notes/` 進 git，只想當本機筆記，可以把它加到 local exclude。

```bash
notepad .git/info/exclude
```

加入：

```text
AGENTS.md
notes/
```

這只影響本機，不會改 repo，也不會被 commit。

優點：

- `git status` 乾淨。
- 不會誤加 notes 到 PR。

缺點：

- 換電腦或重 clone 會消失。
- learning history 不會被版本控制。

我的建議是：你的 notes 有明顯長期價值，應該 commit 到 fork-only branch，而不是完全 local-only。

## Situation G: I Want To Keep Working On Main Casually

可以，但要接受 `main` 是 learning branch，不是 PR branch。

適合放：

- personal README。
- notes。
- experiments。
- failed attempts。
- Windows setup records。

不適合直接做：

- 從 `main` 開 upstream PR。
- 把 `main` 當成乾淨 upstream mirror。

如果你要在 `main` 上繼續學習，最好在 note 或 branch naming 裡明確標記：

```text
main = learning fork branch
upstream/main = clean upstream baseline
fix-* = PR branches
```

## Situation H: I Want A Completely Separate PR Workspace

如果你不想讓 learning notes 和 PR work 互相干擾，可以用 `git worktree`。

概念：

```bash
git worktree add ..\libiui-pr-fix-msys2 upstream/main
```

然後在新資料夾裡做 PR：

```text
D:\Github\libiui-pr-fix-msys2
```

原本資料夾繼續當 learning workspace：

```text
D:\Github\libiui-for-learn
```

優點：

- 完全隔離。
- 不會被 untracked notes 干擾。
- PR branch 永遠乾淨。

缺點：

- 多一個資料夾。
- 要記得在哪個 workspace 操作。

如果你常常同時有 learning notes 和 PR work，這是最乾淨的方案。

## Recommended Decision For Now

目前已經完成第一階段整理：

```text
main / origin/main / upstream/main
  都指向 eb8d9c8，保持乾淨

learn/windows-first-attempt
  保存 e722b54 first attempt 和 jserv commit comment 的公開脈絡

notes/
  仍是 untracked learning assets，尚未決定要 commit 到哪裡

upstream PR
  一律從 upstream/main 開新 branch
```

也就是：

```text
main 回到乾淨基線。
第一次嘗試由 learn branch 保存。
notes 暫時留在 working tree。
PR branch 之後從 upstream/main 開。
```

接下來不要再從 `learn/windows-first-attempt` 或舊的 first attempt commit 開 upstream PR。它的用途是保存脈絡，不是 PR 起點。

## Practical Next Steps

### Step 1: 記錄 branch policy

把規則寫死：

```text
main: clean fork branch, kept close to upstream/main
upstream/main: upstream baseline
fix-*: upstream PR branch
learn/*: notes, experiments, first attempts
```

### Step 2: 決定 notes 是否 commit

二選一：

```text
A. commit 到 learn/notes branch
B. 加到 .git/info/exclude，只留本機
```

我偏向 A，但要先想清楚 notes branch 是否要獨立於 code branch；純 learning notes 不需要 merge 進 `fix-*` PR branch。

### Step 3: 做第一個乾淨 PR branch

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
```

只處理 `tests/example.c`。

### Step 4: 開 PR 前檢查污染

```bash
git diff upstream/main..HEAD --stat
git diff upstream/main..HEAD -- README.md notes AGENTS.md
```

如果第二個指令有輸出，就代表 PR branch 被 learning content 污染。

### Step 5: 等 PR 合併或被 review 後，再決定是否清 `main`

這一步已經不再是當前優先事項，因為 `main` 和 `origin/main` 目前已經回到 `upstream/main`。之後只需要維持這個習慣：

- 不要直接在 `main` 做個人實驗。
- 個人實驗放 `learn/*`。
- upstream PR 放 `fix-*`。

## Mental Model To Remember

不要把 git branch 想成「一定要有一條完美主線」。

更適合你的模型是：

```text
一條線保存學習歷程。
一條線追 upstream。
每個 PR 開一條乾淨短線。
```

這樣第一次 commit 隨便處理的成本會被局部化，不會污染未來每一次 PR。
