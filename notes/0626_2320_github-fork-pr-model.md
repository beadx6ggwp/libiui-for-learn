# GitHub Fork and PR Mental Model

## Question

我現在不只需要知道「這個 repo 怎麼整理」，還需要理解 GitHub fork 的整體概念：

- fork 到底是什麼？
- `origin`、`upstream`、local branch、remote branch 是什麼關係？
- 為什麼不要從 fork 的 `main` 開 PR？
- 為什麼第一次 commit 隨便處理，不一定會毀掉後續 PR？
- 要怎麼把 learning fork、notes、upstream PR 分開？

## Big Picture

GitHub fork 是一種權限與協作模型。

你不能直接 push 到 `sysprog21/libiui`，因為那是別人的 repository。GitHub 讓你 fork 一份到自己的帳號底下：

```text
sysprog21/libiui
  upstream project, maintainer controls it

beadx6ggwp/libiui-for-learn
  your fork, you can push freely
```

你在自己的 fork 裡開 branch、push commits。當你想把某些 commits 送回 upstream，就開 Pull Request。

Pull Request 的本質不是「把整個 fork 送回去」，而是：

```text
請 upstream maintainer review 這個 branch 上的一組 commits，
如果接受，就 merge 到 upstream 的某個 branch。
```

## Four Places To Distinguish

理解 fork 時，先分清楚四個地方。

### 1. Upstream Repository on GitHub

```text
https://github.com/sysprog21/libiui
```

這是原專案。你的目標 PR 通常要 merge 到這裡的 `main`。

在你的 local repo 裡，它叫：

```text
upstream
```

目前設定：

```text
upstream  https://github.com/sysprog21/libiui.git
```

### 2. Your Fork on GitHub

```text
https://github.com/beadx6ggwp/libiui-for-learn
```

這是你的副本。你可以自由 push。你也可以把它當 learning fork，放 notes、experiments、personal README。

在你的 local repo 裡，它叫：

```text
origin
```

目前設定：

```text
origin  https://github.com/beadx6ggwp/libiui-for-learn.git
```

### 3. Local Repository

這是你電腦上的資料夾：

```text
D:\Github\libiui-for-learn
```

你在這裡寫 code、commit、切 branch、看 diff。

### 4. Working Tree / Index / Commit History

local repository 裡還要再分：

```text
working tree
  你目前檔案系統中的實際檔案

index / staging area
  git add 之後，準備進下一個 commit 的內容

commit history
  已經被 git 記錄下來的歷史
```

你現在的 `notes/` 和 `AGENTS.md` 是 working tree 裡的 untracked files。它們還沒進 commit history。

## Your Current Concrete State

目前 graph 是：

```text
e722b54  main, origin/main
|
eb8d9c8  upstream/main
```

意思是：

- `upstream/main` 停在 `eb8d9c8`。
- 你的 local `main` 和 GitHub fork 的 `origin/main` 多了一個 commit：`e722b54`。
- 這個 commit 改了 `README.md` 和 `tests/example.c`。

目前 untracked：

```text
AGENTS.md
notes/
```

這些還不屬於任何 branch 的 commit。branch 只記錄 commits，不記錄「還沒 git add/git commit 的 untracked files」。

## What `origin` and `upstream` Really Are

`origin` 和 `upstream` 最容易混淆，因為它們看起來像 Git 內建概念，但其實不是。它們只是你本機 `.git/config` 裡替遠端 repository 取的名字。

```bash
git remote -v
```

看到：

```text
origin    https://github.com/beadx6ggwp/libiui-for-learn.git
upstream  https://github.com/sysprog21/libiui.git
```

意思是：

```text
origin
  我的 GitHub fork
  我有權限 push
  用來存放我的 branches

upstream
  原作者 / maintainer 的 GitHub repo
  我通常沒有權限 push
  用來取得最新 upstream code
```

關鍵：`origin` / `upstream` 是「本機幫遠端 URL 取的暱稱」，不是 branch。

```text
origin     = remote name
upstream   = remote name
main       = branch name
origin/main = remote-tracking branch name
upstream/main = remote-tracking branch name
```

## Three Different Layers

要真正搞懂，要分三層。

### Layer 1: GitHub 上真的存在的 repositories

```text
GitHub:

sysprog21/libiui
  real repository on GitHub
  maintainer owns it

beadx6ggwp/libiui-for-learn
  real repository on GitHub
  you own it
```

這兩個 repo 都在 GitHub 上。fork 不是 branch，而是一整份 repository copy。

### Layer 2: 你本機對這兩個 repositories 的命名

```text
Local .git/config:

upstream -> https://github.com/sysprog21/libiui.git
origin   -> https://github.com/beadx6ggwp/libiui-for-learn.git
```

這層只是「本機命名」。你也可以把它們取名為 `teacher` 和 `mine`，Git 也能工作；只是開源慣例叫 `upstream` 和 `origin`。

### Layer 3: 你本機記住的 remote branch snapshots

```text
Local remote-tracking branches:

upstream/main
  last fetched snapshot of sysprog21/libiui main

origin/main
  last fetched snapshot of beadx6ggwp/libiui-for-learn main
```

這些不是 GitHub 上的 branch 本體，而是你本機 `.git` 裡的快照。`git fetch` 會更新它們。

## Direction Of Data Flow

對 fork-based contribution，資料流通常長這樣：

```text
Fetch direction:

sysprog21/libiui
        |
        | git fetch upstream
        v
your local repo

Push direction:

your local repo
        |
        | git push origin my-branch
        v
beadx6ggwp/libiui-for-learn

PR direction:

beadx6ggwp/libiui-for-learn:my-branch
        |
        | GitHub Pull Request
        v
sysprog21/libiui:main
```

也就是：

```text
從 upstream 拉最新 code。
在 local 做修改。
推到 origin。
用 PR 請 upstream 合併。
```

不要把這幾件事混在一起：

```text
fetch upstream
  從原專案拿最新歷史到本機

push origin
  把我的 branch 推到我的 fork

open PR
  請原專案 merge 我的 fork branch
```

## Your Current Remotes And Branches

你現在的 remote 設定是：

```text
origin
  fetch/push -> beadx6ggwp/libiui-for-learn

upstream
  fetch/push -> sysprog21/libiui
```

你現在的 branch 狀態是：

```text
local main
  e722b54
  tracks origin/main

origin/main
  e722b54

upstream/main
  eb8d9c8
```

圖像化：

```text
GitHub upstream repo:
  sysprog21/libiui:main
        |
        v
  upstream/main in your local repo
        eb8d9c8

Your local repo:
  main
        e722b54

GitHub fork:
  beadx6ggwp/libiui-for-learn:main
        |
        v
  origin/main in your local repo
        e722b54
```

所以：

```text
main == origin/main
main is ahead of upstream/main by one commit
```

那個多出來的 commit 就是：

```text
e722b54 Add Windows build instructions and documentation
```

## Common Misunderstandings

### Misunderstanding 1: `origin` means original project

不是。

在 fork workflow 裡，`origin` 通常是「你自己的 fork」，不是原作者專案。

```text
origin = my fork
upstream = original project
```

名字 `origin` 只是因為你通常是從自己的 fork clone 下來，所以它成為 clone 的 origin。

### Misunderstanding 2: `upstream/main` is a branch I can commit to

不是。

`upstream/main` 是 remote-tracking branch，是你本機記住的 upstream 狀態。你不應該直接在它上面 commit。

你要做的是：

```bash
git switch -c fix-something upstream/main
```

意思是：從 `upstream/main` 的位置開一條新的 local branch。

### Misunderstanding 3: push to upstream is how PR works

不是。

一般 contributor 沒有權限 push 到 upstream。PR flow 是：

```text
push to origin
open PR to upstream
```

### Misunderstanding 4: fork main messy means PR must be messy

不是。

PR 比較的是你選的 compare branch，不是整個 fork。只要 compare branch 乾淨，fork `main` 可以保留 learning history。

### Misunderstanding 5: `git fetch upstream` will change my files

不會。

`fetch` 只更新 remote-tracking branch，例如 `upstream/main`。它不會修改 working tree。

會改你目前 branch/檔案的是：

```bash
git merge upstream/main
git rebase upstream/main
git pull
git reset --hard
```

## Why `origin/main` And `main` Can Differ

`main` 是你本機 branch。

`origin/main` 是你本機記住的「GitHub fork main 上次 fetch 時的狀態」。

它們可以一樣，也可以不一樣。

例如你本機 commit 後還沒 push：

```text
main        = new local commit
origin/main = old GitHub fork state
```

這時 `git status` 可能會說你 ahead of origin/main。

如果你 push：

```bash
git push origin main
```

GitHub fork 更新後，再 fetch 或 push 完成後 tracking 狀態會接近：

```text
main == origin/main
```

你現在就是這種：

```text
main == origin/main == e722b54
upstream/main == eb8d9c8
```

## Why `upstream/main` And `origin/main` Differ

因為它們是兩個不同 GitHub repo 的 `main`。

```text
upstream/main
  sysprog21/libiui 的 main

origin/main
  beadx6ggwp/libiui-for-learn 的 main
```

你的 fork 可以比 upstream 多 commit，也可以落後 upstream，也可以兩者都有。

常見狀態：

```text
fork ahead
  origin/main 有你的 commit，upstream/main 沒有

fork behind
  upstream/main 有新 commit，origin/main 還沒同步

fork diverged
  兩邊都有彼此沒有的 commit
```

你目前是：

```text
fork ahead by 1 commit
```

因為 `origin/main` 有 `e722b54`，但 `upstream/main` 沒有。

## The Most Important Command Pair

對 fork PR 來說，最核心的兩個方向是：

```bash
git fetch upstream
git push origin my-branch
```

第一個是拿 upstream 的資訊：

```text
upstream -> local
```

第二個是把你的 branch 放到 fork：

```text
local -> origin
```

之後 PR 是 GitHub server 上的動作：

```text
origin branch -> upstream branch
```

如果你只記一張圖，記這張：

```text
              Pull Request
origin/my-branch ---------> upstream/main
      ^                         |
      |                         |
      | git push origin         | git fetch upstream
      |                         v
   local branch <---------- upstream/main snapshot
```

## Local Branch vs Remote-Tracking Branch

### Local branch

```text
main
fix-msys2-ucrt-clock
learn/notes
```

這些是你本機可以 checkout、commit 的 branch。

### Remote-tracking branch

```text
origin/main
upstream/main
```

這些是你本機記住的「上次 fetch 時，remote branch 長什麼樣子」。

重要：`origin/main` 不是 GitHub 上的 branch 本體，而是你本機的快照。要更新它，要跑：

```bash
git fetch origin
```

要更新 `upstream/main` 的本機快照，要跑：

```bash
git fetch upstream
```

## Fetch, Pull, Push

### `git fetch`

只更新 remote-tracking branches，不改你目前檔案。

```bash
git fetch upstream
```

意思是：

```text
去 GitHub 看 upstream 有沒有新 commit，
更新本機的 upstream/main 指標。
```

安全，適合常做。

### `git pull`

大致等於：

```text
fetch + merge/rebase into current branch
```

所以它會改你目前 branch。初學時容易不小心把 upstream changes merge 進 learning branch。

### `git push`

把你的 local branch commits 推到 remote。

```bash
git push origin fix-msys2-ucrt-clock
```

意思是：

```text
把本機 fix-msys2-ucrt-clock branch 推到我的 fork。
```

不要 push 到 `upstream`，除非你是 maintainer。你現在 remote 裡有 `upstream` push URL，但通常沒有權限；實務上 PR flow 只需要 push 到 `origin`。

## What A Pull Request Compares

GitHub PR 會有兩邊：

```text
base repository: sysprog21/libiui
base branch:     main

head repository: beadx6ggwp/libiui-for-learn
compare branch:  fix-msys2-ucrt-clock
```

視覺化：

```text
sysprog21/libiui:main
        ^
        | Pull Request asks to merge
        |
beadx6ggwp/libiui-for-learn:fix-msys2-ucrt-clock
```

PR 不是比較整個 fork，也不是比較你的 fork `main`。它比較的是你指定的 compare branch 和 upstream 的 base branch。

所以即使你的 fork `main` 有 learning notes，只要 PR 的 compare branch 是乾淨的，就不會污染 PR。

## Why Not Use Fork `main` For PR

因為 fork `main` 很容易變成混合用途：

```text
personal README
notes
experiments
first attempts
temporary fixes
```

如果你從 fork `main` 開 PR，GitHub 會拿：

```text
beadx6ggwp/libiui-for-learn:main
```

去比較：

```text
sysprog21/libiui:main
```

那麼 `e722b54` 裡的 `README.md` 119 行個人教學就會一起出現在 PR diff。

這就是為什麼 maintainer 會提醒 contributor：不要用 default branch (`main`) 開 PR，用 feature branch。

## Branch Is Just A Pointer

Git branch 不是整份資料夾副本，它只是指向某個 commit 的名字。

例如：

```text
main -> e722b54
upstream/main -> eb8d9c8
```

新開 branch：

```bash
git switch -c fix-msys2-ucrt-clock upstream/main
```

意思是：

```text
建立一個叫 fix-msys2-ucrt-clock 的新指標，
它從 upstream/main 目前指到的 commit 開始。
```

所以你可以保留混亂的 `main`，同時有乾淨的 PR branch。這兩件事不衝突。

## Current Best Model For This Repo

建議你把 repo 想成三層：

```text
upstream/main
  原專案正史，只拿來當乾淨起點

origin/main or learn/*
  你的 learning fork，可以保存 notes、README、嘗試紀錄

fix-* branches
  每個 upstream PR 一條乾淨短 branch
```

具體對應：

```text
main
  目前有 e722b54，可以視為 learning fork branch

notes/
  learning assets，不該進 upstream PR

fix-msys2-ucrt-clock
  未來從 upstream/main 開，專門放 Windows build fix
```

## Scenario 1: I Want To Submit A Clean PR

流程：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
```

做最小修改後：

```bash
git diff upstream/main..HEAD --stat
git diff upstream/main..HEAD -- README.md notes AGENTS.md
```

如果第二個指令沒有輸出，就代表 PR 不含 personal docs。

接著：

```bash
git push origin fix-msys2-ucrt-clock
```

到 GitHub 開 PR：

```text
base: sysprog21/libiui main
compare: beadx6ggwp/libiui-for-learn fix-msys2-ucrt-clock
```

## Scenario 2: I Want To Keep Notes In My Fork

可以。只要不要從 notes branch 開 upstream PR。

例如：

```bash
git switch -c learn/notes main
git add AGENTS.md notes/
git commit -m "Add learning notes and PR planning records"
git push origin learn/notes
```

這表示：

```text
learn/notes 是我的學習資料分支。
它不是要送 upstream 的 PR branch。
```

它也不需要一直 merge 進每個開發 branch。純 learning notes 是獨立筆記本；只有 upstream 需要 review 的 docs 才應該放進 PR branch。

## Scenario 3: I Want My Fork Main To Track Upstream Cleanly

這是另一種策略：讓 `origin/main` 永遠跟 `upstream/main` 一樣乾淨。

好處：

- 心智負擔小。
- `main` 永遠可以拿來開新 branch。

缺點：

- personal notes 要放別的 branch。
- 你已 push 的 `e722b54` 如果要移除，需要 rewrite fork history。

如果真的要做，安全流程是：

```bash
git branch learn/windows-first-attempt main
git switch main
git reset --hard upstream/main
git push --force-with-lease origin main
```

但這是破壞性操作。現在不建議急著做。

## Scenario 4: I Accidentally Committed To Main

這就是現在的狀況。

不要慌，因為 commit 只是多在 `main` 上，不代表未來 PR 一定會包含它。

你有三種選擇：

### Keep it

把 `main` 當 learning branch。

### Split it

從 `upstream/main` 開乾淨 branch，再把需要的檔案 cherry-pick 過去。

### Rewrite it

備份後 reset `main`，再 force push。這適合你已經確定 fork `main` 要保持乾淨時再做。

## Scenario 5: I Want To Reuse Only One File From A Bad Commit

例如你只想拿 `e722b54` 的 `tests/example.c`，不要 `README.md`。

可以在乾淨 branch 上：

```bash
git switch -c fix-msys2-ucrt-clock upstream/main
git checkout e722b54 -- tests/example.c
```

然後手動整理 `tests/example.c`。

這比整個 cherry-pick 更直覺，因為你明確只拿一個檔案。

但注意：如果 `tests/example.c` 裡也有不乾淨的註解或 whitespace，你仍然要修。

## Scenario 6: I Want To Sync My Fork With Upstream

有兩種意思。

### Update local knowledge of upstream

```bash
git fetch upstream
```

這只是更新 `upstream/main` 的本機快照。

### Bring upstream commits into my branch

如果你在 local `main`：

```bash
git merge upstream/main
```

或：

```bash
git rebase upstream/main
```

但如果 `main` 是 learning branch，就不一定要常做。你可以只在開 PR branch 時從最新 `upstream/main` 開。

## Scenario 7: I Want Total Separation

使用 `git worktree`。

```bash
git worktree add ..\libiui-pr-fix-msys2 upstream/main
```

結果：

```text
D:\Github\libiui-for-learn
  learning workspace

D:\Github\libiui-pr-fix-msys2
  clean PR workspace
```

這適合你目前這種狀況：一邊需要 notes，一邊又不想 PR 被 notes 影響。

## The Clean PR Flow

標準流程：

```text
1. fetch upstream
2. create feature branch from upstream/main
3. make one focused change
4. run validation
5. push branch to origin
6. open PR from origin/branch to upstream/main
7. address review
8. rebase/squash if needed
9. PR merged
10. delete PR branch
```

命令版本：

```bash
git fetch upstream
git switch -c fix-something upstream/main
# edit files
git status
git diff
git add <files>
git commit -m "Fix something precise"
git push origin fix-something
```

## The Learning Fork Flow

你的 learning fork 可以長這樣：

```text
main
  first attempts, personal README, notes pointers

learn/notes
  structured notes

learn/windows-build
  experiments and logs
```

這些 branch 的價值是學習與保存脈絡，不是直接送 upstream。

## Important Distinction

不要把這兩句混在一起：

```text
I want my fork to contain my learning process.
I want my PR to be clean.
```

這兩件事可以同時成立，靠的是 branch separation：

```text
fork can be messy
PR branch must be clean
```

更精確地說：

```text
learning branches can be messy
upstream PR branches must be clean
```

## Recommended Policy For You

目前建議：

```text
1. 保留 main 作為 learning fork，不急著 reset。
2. 把 notes commit 到 learn/notes 或保持 untracked，先不要混進 PR。
3. 第一個 upstream PR 從 upstream/main 開 fix-msys2-ucrt-clock。
4. 每次開 PR 前檢查 README.md、notes、AGENTS.md 沒有進 diff。
5. 等你熟悉後，再決定要不要把 fork main reset 成乾淨 mirror。
```

短版心法：

```text
fork 是你的實驗場。
upstream 是對方的正史。
PR branch 是你拿給對方 review 的乾淨證據包。
```

## Follow-ups

- [ ] 決定 `main` 是否正式定義為 learning branch。
- [ ] 決定 `notes/` 要 commit 到 `learn/notes` 還是留 local-only。
- [ ] 建立第一個乾淨 PR branch：`fix-msys2-ucrt-clock`。
- [ ] 練習用 GitHub PR UI 確認 base/head：`sysprog21/libiui:main <- beadx6ggwp/libiui-for-learn:fix-msys2-ucrt-clock`。
