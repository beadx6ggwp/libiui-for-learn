# Ch06: Rebase Conflict And Recovery

## 本章問題

你已經知道 PR branch 可以從 `upstream/main` 開, 也知道 `git diff upstream/main...HEAD` 可以檢查 PR diff. 下一個會卡住的場景是:

```text
upstream/main 前進了.
maintainer 要你 rebase.
rebase 出現 conflict.
working tree 裡又有 notes 和 code fix 混在一起.
```

naive 想法是:

```text
看到 conflict 就手動改到能編.
看到東西亂了就 reset --hard.
真的不行就 force push.
```

這很危險. 正確思路是先辨認 Git 現在停在哪個狀態, 再用最小操作繼續或退出.

> [Scope]
> 本章整理 `libiui` PR branch 上常見的 rebase, conflict resolution, partial staging, tracking branch, and recovery. 它不鼓勵亂用危險指令, 而是建立 "出事時先看狀態" 的操作順序.

## Rebase 在做什麼

假設 PR branch 長這樣:

```text
O --- U <- upstream/main
 \
  F <- fix/msys2-ucrt-clock, HEAD
```

`git rebase upstream/main` 的目標是把 `F` 重新接到 `U` 後面:

```text
Before:

O --- U <- upstream/main
 \
  F <- HEAD

After:

O --- U <- upstream/main
       \
        F2 <- HEAD
```

注意:

```text
F2 不是原本的 F.
F2 是把 F 的 patch 重新套到 U 後產生的新 commit.
```

所以 rebase 後如果 PR branch 已經 push 過, 更新 remote branch 通常需要:

```sh
git push --force-with-lease origin fix/msys2-ucrt-clock
```

## Rebase 前的安全檢查

先確定你在 PR branch:

```sh
git status --short --branch
git branch -vv
```

理想狀態:

```text
## fix/msys2-ucrt-clock
working tree clean
```

如果還有未提交修改:

```text
 M tests/example.c
?? notes/
```

先不要 rebase. 你要先決定這些修改:

```text
要進 PR commit
  stage and commit it

只是暫時工作
  stash it

是 personal notes
  保持 untracked, 不要 add
```

## Tracking Branch 是什麼

第一次 push PR branch 時可以用:

```sh
git push -u origin fix/msys2-ucrt-clock
```

`-u` 會設定 upstream tracking relationship:

```text
local fix/msys2-ucrt-clock
  tracks origin/fix/msys2-ucrt-clock
```

之後你可以用:

```sh
git push
git pull --rebase
```

但在 upstream PR workflow 裡, 仍建議你對重要操作寫完整:

```sh
git push origin fix/msys2-ucrt-clock
git push --force-with-lease origin fix/msys2-ucrt-clock
```

因為完整命令會提醒你:

```text
remote 是 origin, 不是 upstream.
branch 是 fix/*, 不是 main.
```

用 `git branch -vv` 可以看:

```text
* fix/msys2-ucrt-clock abc1234 [origin/fix/msys2-ucrt-clock] Fix MSYS2 UCRT64 demo clock build
```

## Conflict 是什麼

Conflict 不是 Git 壞掉. 它只是說:

```text
upstream/main 和你的 PR patch 改到同一段內容.
Git 無法自動判斷要保留哪邊.
```

圖:

```text
O --- U <- upstream/main
 \     modifies tests/example.c line 40
  \
   F <- HEAD
     modifies tests/example.c line 40
```

跑:

```sh
git rebase upstream/main
```

可能得到:

```text
CONFLICT (content): Merge conflict in tests/example.c
```

此時 Git 暫停在 rebase 中間:

```text
O --- U <- upstream/main
       \
        applying F as F2...
        stopped because conflict

WT:
  tests/example.c contains conflict markers

Index:
  unresolved entries
```

## Conflict 時的固定流程

先看狀態:

```sh
git status
```

你會看到 Git 告訴你:

```text
You are currently rebasing.
Unmerged paths:
  both modified: tests/example.c
```

看 conflict:

```sh
git diff
```

檔案裡會有 marker:

```text
<<<<<<< HEAD
upstream version
=======
your patch version
>>>>>>> F
```

解 conflict 的意思不是隨便選一邊. 你要重新建立正確結果:

```text
保留 upstream 新 invariant.
重新套上你的最小 fix.
移除 conflict markers.
```

解完後:

```sh
git add tests/example.c
git rebase --continue
```

如果判斷這次 rebase 不該繼續:

```sh
git rebase --abort
```

`--abort` 的意思是回到 rebase 前的狀態.

## Partial Staging: 把混在一起的修改拆開

`libiui-for-learn` 很容易同時出現:

```text
tests/example.c
  upstream PR fix

README.md
  personal learning rewrite

notes/
  thinking notes
```

甚至同一個檔案也可能混了兩種修改:

```text
tests/example.c
  real build fix
  temporary debug printf
```

這時不要用:

```sh
git add .
```

用 partial staging:

```sh
git add -p tests/example.c
git diff --cached
git diff
```

圖:

```text
Before git add -p:

WT:
  tests/example.c
    hunk A = real fix
    hunk B = debug print

Index:
  empty

After staging only hunk A:

WT:
  tests/example.c
    hunk A = staged
    hunk B = unstaged

Index:
  tests/example.c hunk A
```

檢查 staged patch:

```sh
git diff --cached
```

如果 staged patch 是乾淨的:

```sh
git commit -m "Fix MSYS2 UCRT64 demo clock build"
```

如果 stage 錯了, 可以反向 partial restore:

```sh
git restore -p --staged tests/example.c
```

## Stash 什麼時候有用

如果你正在寫 notes, 但突然要 rebase PR branch, 可以 stash tracked modifications:

```sh
git stash push -m "wip before rebase"
```

但要注意:

```text
default stash 不會包含 untracked files.
```

如果要包含 untracked:

```sh
git stash push -u -m "wip notes before rebase"
```

對這個 repo 來說, `notes/` 很重要. 用 `-u` 前要先看清楚:

```sh
git status --short
```

stash 是臨時抽屜, 不是長期保存. 重要 notes 應該 commit 到 learning branch 或至少確認有備份.

## Recovery: Reflog 是最後防線

如果你 reset, rebase, force push 後覺得 commit 不見了, 先不要繼續亂操作. 先查:

```sh
git reflog
```

你可能看到:

```text
abc1234 HEAD@{0}: rebase finished
e722b54 HEAD@{1}: checkout: moving from learn/windows-first-attempt
```

如果找到要救的 commit:

```sh
git branch rescue/windows-first-attempt e722b54
```

圖:

```text
Before rescue:

e722b54
  no branch name points here
  only reflog remembers it

After rescue:

e722b54 <- rescue/windows-first-attempt
```

也可以檢查某個 commit 是否仍被 branch 包住:

```sh
git branch -a --contains e722b54
```

如果有輸出:

```text
learn/windows-first-attempt
remotes/origin/learn/windows-first-attempt
```

代表它還不是孤立 commit.

## 出事時的順序

不要先猜. 先收集狀態:

```sh
git status --short --branch
git branch -vv
git log --oneline --decorate --graph --all -n 30
git reflog -n 20
```

然後分類:

```text
只是 conflict
  resolve -> git add -> git rebase --continue

rebase 方向錯了
  git rebase --abort

stage 錯了
  git restore --staged <file>
  or git restore -p --staged <file>

working tree 改壞了
  restore specific files only

commit 找不到
  git reflog -> git branch rescue/<name> <commit>
```

盡量避免第一反應就是:

```sh
git reset --hard
git clean -fd
git push --force
```

這些不是不能用, 而是必須在你知道它們會動哪一層時才用.

## 本章學到的三件事

> [Summary]
> 1. Rebase 是把你的 PR patch 重新套到新的 `upstream/main` 上, 所以 push 過的 PR branch 通常要用 `--force-with-lease` 更新.
> 2. Conflict 時先看 `git status` 和 `git diff`, 解完後 `git add` 再 `git rebase --continue`, 不對就 `git rebase --abort`.
> 3. Partial staging 和 reflog 是兩個安全工具: 前者讓 commit scope 乾淨, 後者讓錯誤 history 操作還有救回機會.

## Source Notes

```text
../0627_0013_git-operation-concepts.md
../0626_1731_repo-state-cleanup-strategy.md
ch02-working-tree-branch-safety.md
ch03-pr-branch-force-push.md
ch05-diff-range-and-merge-base.md
```
