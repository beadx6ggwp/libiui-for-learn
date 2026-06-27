# Ch05: PR Operation Playbook

## 本章問題

前面幾章已經談過 PR 思考模型, commit 風格, PR body, validation. 但真正要送 `sysprog21/libiui` PR 時, 還有一個很實際的問題:

```text
我現在到底要下哪些 Git 指令?
每個指令在 PR 貢獻流程裡的目的又是什麼?
```

naive 想法是把 Git 教學和 PR 寫作分開:

```text
先學 Git.
再學 open-source PR.
```

但在 `libiui` 這個 repo 裡, 兩者其實要接在一起. PR 不是一篇獨立作文, 它是某條 branch 相對 upstream 的 diff. 所以 Git 操作的任務就是把你的想法變成 reviewer 可以檢查的最小 diff.

> [Scope]
> 本章不重新解釋 `origin`, `upstream`, `reset`, `force push` 的完整概念. 詳細 mental model 放在 `../libiui-git/`. 這章只整理 "準備一個 libiui upstream PR 時, 實際操作順序應該怎麼接到 PR 寫作".

## 核心模型

一個乾淨 PR 的資料流是:

```text
upstream/main
  -> local fix/* branch
  -> focused commits
  -> origin/fix/* branch
  -> GitHub PR
  -> review update
```

用圖表示:

```text
sysprog21/libiui:main
        |
        | git fetch upstream
        v
local upstream/main
        |
        | git switch -c fix/... upstream/main
        v
local fix/...
        |
        | git commit
        v
local fix/... with focused patch
        |
        | git push origin fix/...
        v
beadx6ggwp/libiui-for-learn:fix/...
        |
        | open PR
        v
sysprog21/libiui PR
```

每一步都在回答 reviewer 的一個問題:

```text
branch from upstream/main
  這個 PR 是從最新上游基準開始嗎?

focused diff
  這個 PR 只改它宣稱要改的問題嗎?

commit subject
  commit history 以後還能讀嗎?

validation
  這個修正有被驗證嗎?

review update
  reviewer 要求的修改有被整理進同一個 PR 嗎?
```

## 圖解符號

後面每個 Git 操作都會用同一組符號:

```text
O = upstream/main 的基準 commit
L = learning commit, 例如 e722b54
F = focused PR commit
R = review update commit

WT    = working tree
Index = staging area
HEAD  = 目前 checkout 的 local branch
```

圖裡的 branch name 是指標, 不是資料夾:

```text
O <- upstream/main
^
|
fix/msys2-ucrt-clock, HEAD
```

這張圖的意思是:

```text
upstream/main 和 fix/msys2-ucrt-clock 都指向同一個 commit O.
目前 checkout 的 branch 是 fix/msys2-ucrt-clock.
```

## Step 0: 先確認上游基準

開 PR 前先更新 `upstream/main`:

```sh
git fetch upstream
git status --short --branch
```

圖解:

```text
Before fetch:

remote sysprog21/libiui
  O --- U <- main

local repo
  O <- upstream/main
  O <- main

After git fetch upstream:

remote sysprog21/libiui
  O --- U <- main

local repo
  O --- U <- upstream/main
  O     <- main

WT:
  unchanged
```

你要知道兩件事:

```text
upstream/main 是最新 snapshot.
目前 working tree 有沒有未處理修改.
```

如果 `status` 出現:

```text
?? notes/
?? AGENTS.md
```

這不代表它們會進 PR. 它只代表這些檔案尚未被 Git 追蹤. 真正關鍵是不要把它們 `git add` 到 upstream PR branch.

## Step 1: 從 upstream/main 開 PR Branch

不要從 `main` 或 `learn/windows-first-attempt` 隨手開. 用明確的 PR branch:

```sh
git switch -c fix/msys2-ucrt-clock upstream/main
```

圖解:

```text
Before:

O <- upstream/main
O <- main, HEAD

Run:

git switch -c fix/msys2-ucrt-clock upstream/main

After:

O <- upstream/main
O <- main
O <- fix/msys2-ucrt-clock, HEAD

WT:
  files match O
```

命名可以照用途:

```text
fix/msys2-ucrt-clock
doc/windows-build
test/slider-regression
ci/clang-analyzer
```

branch 名稱不是給 Git 看而已, 它也是給自己和 reviewer 的 scope reminder. 如果 branch 叫 `fix/msys2-ucrt-clock`, diff 就不該同時包含 learning notes, README 大改, unrelated formatting.

## Step 2: 從 Learning Commit 抽出可投稿 Patch

目前 `e722b54` 是 first attempt. 它有價值, 但它混合了 personal notes 和 upstream-facing change. 所以不要直接:

```sh
git cherry-pick e722b54
```

比較好的做法是只取需要檔案:

```sh
git restore --source=e722b54 -- tests/example.c
git diff -- tests/example.c
```

圖解:

```text
Commit graph:

O <- upstream/main, fix/msys2-ucrt-clock, HEAD
 \
  L <- learn/windows-first-attempt

Run:

git restore --source=L -- tests/example.c

Commit graph after restore:

O <- upstream/main, fix/msys2-ucrt-clock, HEAD
 \
  L <- learn/windows-first-attempt

WT after restore:

tests/example.c  = version from L
README.md        = version from O
notes/           = not touched

Index:

empty
```

如果你不確定需要哪些檔案, 可以先 no-commit cherry-pick:

```sh
git cherry-pick -n e722b54
git status --short
git diff --stat
```

圖解:

```text
Before:

O <- fix/msys2-ucrt-clock, HEAD
 \
  L <- learn/windows-first-attempt

Run:

git cherry-pick -n L

After:

O <- fix/msys2-ucrt-clock, HEAD
 \
  L <- learn/windows-first-attempt

WT:
  README.md        = changed by L
  tests/example.c  = changed by L

Index:
  README.md        = staged or ready from cherry-pick
  tests/example.c  = staged or ready from cherry-pick

No new commit yet.
```

然後把不該進 PR 的檔案還原:

```sh
git restore --source=upstream/main --staged --worktree README.md
git restore --source=upstream/main --staged --worktree AGENTS.md
git restore --source=upstream/main --staged --worktree notes
```

還原後應該變成:

```text
WT:
  tests/example.c  = candidate patch
  README.md        = back to O
  notes/           = absent from PR diff

Index:
  only files you intentionally stage later
```

這一步的原則是:

```text
learning commit 可以當材料庫.
PR commit 不能照搬 learning commit.
```

## Step 3: 用 Diff 檢查 PR Scope

在 commit 前先看 diff:

```sh
git diff upstream/main...HEAD --stat
git diff upstream/main...HEAD --name-only
```

如果還沒有 commit, 也要看 working tree:

```sh
git diff --stat
git diff --name-only
```

圖解:

```text
Case A: 已經有 commit

O <- upstream/main
 \
  F <- fix/msys2-ucrt-clock, HEAD

git diff upstream/main...HEAD:

compare merge-base(O, F) to F
show files introduced by F
```

如果還沒有 commit:

```text
Case B: 還在 working tree

O <- upstream/main, fix/msys2-ucrt-clock, HEAD

WT:
  tests/example.c modified

git diff:

compare O to WT
show unstaged local changes
```

檢查 personal files:

```sh
git diff upstream/main...HEAD -- README.md notes AGENTS.md
git diff -- README.md notes AGENTS.md
```

理想情況是沒有輸出. 如果有輸出, 你要問:

```text
這真的是 upstream PR 需要的改動嗎?
還是只是我的學習脈絡?
```

這裡對應 Ch01 的因果鏈:

```text
problem -> root cause -> minimal change -> validation -> maintainability
```

`diff --name-only` 是用來檢查 `minimal change`. 如果檔案列表和 PR title 對不上, PR 還沒準備好.

## Step 4: Commit 前先整理 Evidence

在 `git add` 前, 先把 validation 記下來:

```text
failing command before patch
exact error message
fixed command after patch
environment
```

Windows/MSYS2 類 PR 至少要有:

```text
MSYS2 environment: UCRT64
compiler: gcc version
packages: mingw-w64-ucrt-x86_64-SDL2, make, python
command: make defconfig && make
failure before: exact error
result after: build succeeds or test passes
```

然後才 stage 精準檔案:

```sh
git add tests/example.c
git status --short
```

圖解:

```text
Before git add:

WT:
  tests/example.c  modified
  notes/           untracked

Index:
  empty

Run:

git add tests/example.c

After git add:

WT:
  tests/example.c  modified and staged
  notes/           untracked

Index:
  tests/example.c
```

不要用:

```sh
git add .
```

除非你已經非常確定 working tree 沒有 personal files. 對這個 repo 來說, `notes/` 和 `AGENTS.md` 很容易被一起加進去.

## Step 5: 寫 Commit Subject

commit subject 要描述 upstream 會在意的 behavior:

```sh
git commit -m "Fix MSYS2 UCRT64 demo clock build"
```

圖解:

```text
Before commit:

O <- upstream/main
O <- fix/msys2-ucrt-clock, HEAD

Index:
  tests/example.c

Run:

git commit -m "Fix MSYS2 UCRT64 demo clock build"

After commit:

O <- upstream/main
 \
  F <- fix/msys2-ucrt-clock, HEAD

Index:
  empty
```

不要寫:

```text
Update files
Windows stuff
Apply notes from first attempt
```

如果這其實是 documentation PR:

```sh
git commit -m "Document Windows build requirements"
```

subject 是 PR review 的第一個索引. 它要讓 maintainer 看到:

```text
動作: Fix / Document / Add / Replace
範圍: MSYS2 UCRT64 / Windows build / demo clock
結果: build / requirements / compatibility
```

## Step 6: Push 到 Origin, 再開 PR

push 的對象是自己的 fork:

```sh
git push origin fix/msys2-ucrt-clock
```

圖解:

```text
Before push:

local repo
  O <- upstream/main
   \
    F <- fix/msys2-ucrt-clock, HEAD

origin fork
  no fix/msys2-ucrt-clock yet

Run:

git push origin fix/msys2-ucrt-clock

After push:

local repo
  O <- upstream/main
   \
    F <- fix/msys2-ucrt-clock, HEAD

origin fork
  F <- origin/fix/msys2-ucrt-clock

GitHub PR:
  base = sysprog21/libiui:main at O
  head = beadx6ggwp/libiui-for-learn:fix/msys2-ucrt-clock at F
```

這不是把 code 直接推到 `sysprog21/libiui`. 它只是把 PR branch 放到你的 fork. 接著在 GitHub 開:

```text
base: sysprog21/libiui:main
head: beadx6ggwp/libiui-for-learn:fix/msys2-ucrt-clock
```

PR body 可以直接使用前面收集的 evidence:

```markdown
## Problem

...

## Root Cause

...

## Solution

...

## Validation

- ...

## Notes

...
```

Git 操作產生 diff, PR body 解釋 diff. 兩者要一致.

## Step 7: Review 後怎麼更新同一個 PR

如果 reviewer 要你補一個小修正, 先直接 commit:

```sh
git add tests/example.c
git commit -m "Fix MSYS2 UCRT64 demo clock build"
git push origin fix/msys2-ucrt-clock
```

圖解:

```text
Before review update push:

origin fork
  O --- F <- origin/fix/msys2-ucrt-clock

local repo
  O --- F --- R <- fix/msys2-ucrt-clock, HEAD

Run:

git push origin fix/msys2-ucrt-clock

After:

origin fork
  O --- F --- R <- origin/fix/msys2-ucrt-clock

GitHub PR diff:
  O..R
```

同一個 PR 會自動更新, 不用重開.

如果 reviewer 或 maintainer 希望整理 commit history, 例如把:

```text
Fix MSYS2 UCRT64 demo clock build
Fix typo
Address review
```

整理成一個清楚 commit, 才使用 interactive rebase:

```sh
git rebase -i upstream/main
git push --force-with-lease origin fix/msys2-ucrt-clock
```

圖解:

```text
Before cleanup:

origin fork:
  O --- F --- T --- A <- origin/fix/msys2-ucrt-clock

local after git rebase -i upstream/main:
  O --- F2 <- fix/msys2-ucrt-clock, HEAD

F2 is not a descendant of A.
ordinary push will be rejected.

Run:

git push --force-with-lease origin fix/msys2-ucrt-clock

After:

origin fork:
  O --- F2 <- origin/fix/msys2-ucrt-clock

GitHub PR diff:
  O..F2
```

這裡的重點是:

```text
force-with-lease 用在 short-lived PR branch.
不要裸用 --force.
不要把這個規則套到 shared main.
```

## Step 8: PR 合併後收尾

PR 合併後, 本機和 fork 可以清理 PR branch:

```sh
git fetch upstream
git switch main
git branch -d fix/msys2-ucrt-clock
```

圖解:

```text
Before merge:

upstream:
  O <- main

origin fork:
  O --- F <- origin/fix/msys2-ucrt-clock

After upstream merges PR:

upstream:
  O --- M <- main
       /
      F

After deleting local branch name:

local repo:
  O --- M <- upstream/main

branch name fix/msys2-ucrt-clock is gone.
merged change is still reachable through M.
```

如果 remote branch 也要刪:

```sh
git push origin --delete fix/msys2-ucrt-clock
```

這不會刪掉 upstream 已 merge 的 commit. 它只是移除已完成任務的 branch name.

## 操作前後的固定檢查清單

開 PR 前:

```sh
git status --short --branch
git log --oneline upstream/main..HEAD
git diff upstream/main...HEAD --stat
git diff upstream/main...HEAD --name-only
git diff upstream/main...HEAD -- README.md notes AGENTS.md
```

review update 前:

```sh
git fetch upstream
git status --short --branch
git log --oneline upstream/main..HEAD
git diff upstream/main...HEAD --stat
```

force push 前:

```sh
git branch -vv
git log --oneline --decorate --graph upstream/main..HEAD
git diff upstream/main...HEAD --name-only
```

你要確認:

```text
我在 fix/* branch.
我要 push 的 remote 是 origin.
我要更新的是 PR branch.
diff 沒有 personal notes.
commit history 已經整理過.
```

## 操作背後的 Git 深入章節

這章是 PR operation playbook, 只保留送 PR 時的必要順序. 如果某一步開始不直覺, 回到 `notes/libiui-git/` 讀對應章節:

```text
git diff upstream/main...HEAD
  讀 ../libiui-git/ch05-diff-range-and-merge-base.md

upstream/main 前進後要不要 rebase
  讀 ../libiui-git/ch05-diff-range-and-merge-base.md

rebase conflict
  讀 ../libiui-git/ch06-rebase-conflict-and-recovery.md

git add -p / partial staging
  讀 ../libiui-git/ch06-rebase-conflict-and-recovery.md

commit 找不到或 reset 後害怕弄丟
  讀 ../libiui-git/ch06-rebase-conflict-and-recovery.md
```

## Upstream 規範對照

如果你不確定某一步是不是只是我們自己的 workflow, 回到:

```text
CONTRIBUTING.md
  upstream 明文規範

ch06-upstream-contributing-guide.md
  本資料夾對 CONTRIBUTING.md 的整理
```

特別是這些規則不是臆測:

```text
coherent commits
avoid WIP commits
test before committing
format C, shell, Python files
use imperative commit subject
use body to explain what and why
```

而 `learn/*` 和 `fix/*` 的分工, 則是為了讓上述 upstream 規則在這個 fork 裡容易做到.

## 本章學到的三件事

> [Summary]
> 1. PR Git 操作的目標不是炫技, 而是把 problem, root cause, minimal patch, validation 固定成 reviewer 可檢查的 branch diff.
> 2. `e722b54` 這類 learning commit 可以當材料庫, 但正式 PR 應從 `upstream/main` 開 `fix/*` branch, 再抽出最小 patch.
> 3. 普通 review update 用 ordinary push. 只有整理 PR branch history 時才用 `git push --force-with-lease origin fix/...`.

## Source Notes

```text
../libiui-git/ch01-fork-origin-upstream.md
../libiui-git/ch02-working-tree-branch-safety.md
../libiui-git/ch03-pr-branch-force-push.md
../libiui-git/ch04-libiui-repo-policy.md
../libiui-git/ch05-diff-range-and-merge-base.md
../libiui-git/ch06-rebase-conflict-and-recovery.md
ch06-upstream-contributing-guide.md
../0626_1723_upstream-pr-submission-plan.md
../0626_1731_repo-state-cleanup-strategy.md
```
