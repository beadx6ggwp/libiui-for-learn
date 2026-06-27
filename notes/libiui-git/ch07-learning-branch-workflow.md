# Ch07: Learning Branch Workflow

## 本章問題

現在你已經有 `learn/repo-review`, 也知道正式 PR 要從 `upstream/main` 開 `fix/*`. 下一個問題是:

```text
learn branch 之後要怎麼維護?
如果我用 branch A 做功能開發, 但同時有跟這個功能相關的開發 notes, notes 要放哪裡?
learn branch 要不要一直同步 main?
```

naive 想法是:

```text
所有跟功能有關的東西都放在 feature branch.
之後 PR 前再把 notes 刪掉.
```

這會讓 feature branch 同時承擔兩種任務:

```text
給 upstream review 的 code patch.
給自己保留的理解過程.
```

這兩個任務的 audience 不同, 生命週期也不同. 所以要拆開管理.

> [Scope]
> 本章只講 `learn/*` branch 和 feature branch 如何並行. 它不討論 PR body 怎麼寫, 也不討論 rebase conflict 的完整處理. 那些分別在 `notes/libiui-contribution/` 和 Ch06.

## Branch 角色重新整理

目前最穩的分工是:

```text
main
  clean baseline
  follows upstream/main
  not for personal notes

learn/repo-review
  long-lived learning branch
  tracks AGENTS.md and notes/
  saves repo review, build investigation, feature learning notes

fix/*
  short-lived upstream PR branch
  starts from upstream/main
  contains only upstream-facing patch

experiment/*
  optional local experiment branch
  can be messy
  not for PR unless later extracted into fix/*
```

圖:

```text
upstream/main:
O --- U --- V

main:
O --- U --- V

learn/repo-review:
O --- U --- V
             \
              N1 --- N2 --- N3

fix/some-feature:
O --- U --- V
             \
              F
```

重點:

```text
learn/repo-review 和 fix/* 可以從同一個 upstream baseline 分出去.
但 fix/* 不應該包含 notes/.
```

## Learn Branch 什麼時候同步

`learn/repo-review` 不需要每天跟 `upstream/main` 同步. 它需要同步的時機是:

```text
你要開始新一輪 repo review.
upstream/main 有重要變化, 影響你正在看的 build flow or module.
你想讓 notes 裡的路徑, module, command 對齊最新 upstream.
你準備從 review 結論切出新的 fix/* branch.
```

不需要同步的時機:

```text
只是繼續寫既有 notes.
只是整理已經觀察過的 Git/PR 概念.
只是保存個人 planning.
```

原因是:

```text
learn branch 的主要價值是保存理解過程.
不是永遠保持最線性的 upstream history.
```

## Learn Branch 同步方式

如果 `learn/repo-review` 只有 notes, 最簡單是 merge upstream:

```sh
git switch learn/repo-review
git fetch upstream
git merge upstream/main
git push origin learn/repo-review
```

圖:

```text
Before:

O --- U <- upstream/main
 \
  N1 --- N2 <- learn/repo-review

After merge:

O --- U -------- M <- learn/repo-review
 \             /
  N1 --- N2 --
```

這樣會保留 learning history, 適合 long-lived branch.

如果你想讓 learn branch 看起來像線性接在最新 upstream 後面, 可以 rebase:

```sh
git switch learn/repo-review
git fetch upstream
git rebase upstream/main
git push --force-with-lease origin learn/repo-review
```

圖:

```text
Before:

O --- U <- upstream/main
 \
  N1 --- N2 <- learn/repo-review

After rebase:

O --- U <- upstream/main
       \
        N1' --- N2' <- learn/repo-review
```

但我對 `learn/repo-review` 的建議是:

```text
平常用 merge.
只有你明確想整理 personal branch history 時才 rebase.
```

因為這條 branch 是你的筆記歷史, 不是 upstream review branch. 保留 merge commit 沒有問題.

## Feature Branch 旁邊的 Notes 怎麼處理

假設你要做一個功能或修 bug:

```text
fix/input-boundary
```

同時你會產生 notes:

```text
notes/libiui-review/input-boundary-reading.md
notes/0627_1430_input-boundary-question.md
```

不要把這些 notes commit 到 `fix/input-boundary`.

比較穩的做法是兩條線並行:

```text
fix/input-boundary
  only code, tests, upstream docs needed by PR

learn/repo-review
  notes about why the issue exists, failed attempts, reading trail
```

圖:

```text
upstream/main:
O --- U

fix/input-boundary:
O --- U
       \
        F1 --- F2

learn/repo-review:
O --- U
       \
        N1 --- N2
```

## 實際操作 Flow: 先寫 Notes 再切 PR

如果你還在理解問題:

```sh
git switch learn/repo-review
```

在 notes 裡記錄:

```text
問題現象
涉及檔案
naive 假設
為什麼失敗
可能 root cause
需要驗證的 command
```

commit:

```sh
git add notes/
git commit -m "Record input boundary review notes"
git push origin learn/repo-review
```

等到問題範圍清楚, 再開正式 PR branch:

```sh
git fetch upstream
git switch -c fix/input-boundary upstream/main
```

然後只移植必要 code change:

```sh
git restore --source=learn/repo-review -- tests/test-input.c
git diff -- tests/test-input.c
```

或手動重新寫 patch. 最後檢查:

```sh
git diff upstream/main...HEAD -- AGENTS.md notes
```

理想結果是沒有輸出.

## 實際操作 Flow: 已經在 Feature Branch 才想到要寫 Notes

這很常見. 你在 `fix/input-boundary` 改 code 時, 突然想記錄理解.

不要直接:

```sh
git add notes/
git commit -m "Add notes"
```

在 `fix/*` 上這樣做會污染 PR branch.

比較好的做法:

```sh
git status --short
```

如果 code change 還沒 commit, 先只 commit code:

```sh
git add src/input.c tests/test-input.c
git commit -m "Fix input boundary handling"
```

如果 notes 是 untracked, 先不要 add. 切回 learn branch:

```sh
git switch learn/repo-review
```

Git 會把 untracked notes 留在 working tree, 然後你在 learn branch commit:

```sh
git add notes/
git commit -m "Record input boundary investigation"
git push origin learn/repo-review
```

如果 notes 是 tracked 且已修改, 切 branch 可能被 Git 擋下. 這時用 stash:

```sh
git stash push -m "input boundary notes"
git switch learn/repo-review
git stash pop
git add notes/
git commit -m "Record input boundary investigation"
```

注意:

```text
stash 預設不包含 untracked files.
如果 notes 是新檔案, 要用 git stash push -u.
```

## 如果 Feature Branch 已經混進 Notes

假設你不小心在 `fix/input-boundary` commit 了 notes:

```text
O --- F1 --- N1 <- fix/input-boundary
```

先不要慌. 目標是:

```text
fix/input-boundary
  only F1

learn/repo-review
  contains N1
```

做法之一:

```sh
git switch learn/repo-review
git cherry-pick N1
git push origin learn/repo-review
```

然後回到 PR branch, 用 interactive rebase 把 notes commit 移掉:

```sh
git switch fix/input-boundary
git rebase -i upstream/main
```

在 todo list 裡 drop notes commit.

最後檢查:

```sh
git diff upstream/main...HEAD -- AGENTS.md notes
```

圖:

```text
Before:

O --- F1 --- N1 <- fix/input-boundary

After moving N1:

O --- F1 <- fix/input-boundary
 \
  L --- N1 <- learn/repo-review
```

如果這個 PR branch 已經 push 到 origin, rebase 後更新它:

```sh
git push --force-with-lease origin fix/input-boundary
```

## Learn Branch 要不要 merge 功能 Branch

通常不要把 `fix/*` merge 回 `learn/repo-review`.

原因:

```text
fix/* 的 code patch 之後會透過 upstream PR 回到 upstream/main.
learn/repo-review 只需要記錄你的理解, 不需要保存另一份 code history.
```

比較好的做法是:

```text
feature branch 的結論
  寫成 notes.

feature branch 的 code
  送 upstream PR.

PR merged 後
  learn/repo-review 再 merge or rebase 最新 upstream/main.
```

圖:

```text
1. During development:

upstream/main:
O --- U

fix/input-boundary:
O --- U --- F

learn/repo-review:
O --- U --- N

2. After PR merged upstream:

upstream/main:
O --- U --- M
           /
          F

3. Sync learn branch:

learn/repo-review:
O --- U --- N --- S
           \     /
            M ---
```

`S` 可以是 merge commit. 這樣 learn branch 透過 upstream 吸收正式 code 結果, 而不是直接吸收你的 PR branch.

## 一個推薦節奏

日常節奏可以是:

```text
1. 在 learn/repo-review 寫理解 notes.
2. 問題清楚後, 從 upstream/main 開 fix/*.
3. 在 fix/* 只做 code/tests/upstream docs.
4. 如果開發中出現新理解, 切回 learn/repo-review commit notes.
5. PR branch 檢查 AGENTS.md and notes 沒有進 diff.
6. PR merged 後, 同步 learn/repo-review 到新的 upstream/main.
```

對應命令:

```sh
git switch learn/repo-review
git add notes/
git commit -m "Record <topic> investigation"
git push origin learn/repo-review

git fetch upstream
git switch -c fix/<topic> upstream/main

git diff upstream/main...HEAD -- AGENTS.md notes
```

## 本章學到的三件事

> [Summary]
> 1. `learn/repo-review` 是 long-lived learning branch, 不需要每天同步 upstream, 只在 review baseline 需要更新時同步.
> 2. Feature branch 上的 notes 應回到 `learn/repo-review` 保存, `fix/*` 只保留 upstream-facing code, tests, or docs.
> 3. PR merged 後, learn branch 應透過新的 `upstream/main` 吸收正式結果, 不需要直接 merge `fix/*`.

## Source Notes

```text
ch01-fork-origin-upstream.md
ch03-pr-branch-force-push.md
ch04-libiui-repo-policy.md
ch06-rebase-conflict-and-recovery.md
../libiui-review/README.md
```
