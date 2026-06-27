# Ch03: PR Branch And Force Push For libiui

## 本章問題

你之前的直覺是 "盡量不用 force push". 這個直覺是對的, 但如果要為 `sysprog21/libiui` 發 PR, 需要更精確.

真正要問的是:

```text
force push 的對象是哪條 branch?
那條 branch 是 shared main, 還是 short-lived PR branch?
有沒有人依賴它?
有沒有備份重要 commit?
```

這章用 GitHub PR 的 `base/head` 模型, 解釋為什麼 PR #30 可以看到 force push, 但那不代表可以隨便 force push `main`.

> [Scope]
> 本章只講送 `libiui` PR 時的 PR branch 協作模型. 這個 repo 的具體 branch policy 放到 Ch04.

## Naive 想法: PR 是一包固定 patch

常見想法是:

```text
我開 PR 時, GitHub 把當下 diff 存起來.
之後如果要修改, 應該重開 PR.
```

這是錯的.

GitHub PR 更像一個持續更新的比較器:

```text
PR = compare(base branch, head branch)
```

只要 head branch 移動, PR 頁面就會更新.

## Base And Head

每個 PR 有兩邊:

```text
base branch <- head branch
目標 branch    來源 branch
```

對你的 fork 來說:

```text
base: sysprog21/libiui:main
head: beadx6ggwp/libiui-for-learn:fix/windows-build-docs
```

意思是:

```text
請 review head branch 相對於 base branch 的改動.
如果接受, 請 merge 到 base branch.
```

PR branch 通常就是 `head branch`.

## PR 背後其實有三個物件

```text
local branch
  你電腦上的 fix/windows-build-docs

remote branch
  GitHub fork 上的 origin/fix/windows-build-docs

GitHub PR object
  記錄 base repo/branch 和 head repo/branch 的比較關係
```

你在 local branch 修改. 你 push 到 remote branch. GitHub PR object 看到 head branch 更新, 就重新計算 diff.

## Case 1: 開 PR

從 upstream 開 branch:

```sh
git fetch upstream
git switch -c fix/windows-build-docs upstream/main
```

做一個 commit:

```text
O  eb8d9c8 <- upstream/main
|
W  Add Windows build docs <- fix/windows-build-docs
```

push 到 fork:

```sh
git push origin fix/windows-build-docs
```

GitHub PR 記錄:

```text
base = sysprog21/libiui:main -> O
head = beadx6ggwp/libiui-for-learn:fix/windows-build-docs -> W
```

PR diff 是:

```text
O..W
```

## Case 2: Reviewer 要你追加 commit

reviewer 說少了 MSYS2 package list. 你在同一條 branch 上再 commit:

```text
O  upstream/main
|
W  Add Windows build docs
|
P  Add MSYS2 package list <- fix/windows-build-docs
```

普通 push:

```sh
git push origin fix/windows-build-docs
```

GitHub PR 仍是同一個 PR, 只是:

```text
head 從 W 移到 P
diff 從 O..W 變成 O..P
```

這就是為什麼不用重開 PR.

## Case 3: Reviewer 要你整理 commit

如果 reviewer 不想看到:

```text
W  Add Windows build docs
P  Fix typo
Q  Fix typo again
```

你可能用 `rebase -i` 或 `commit --amend` 整理成:

```text
W2 Add Windows build docs
```

graph 會變成:

```text
remote before:

O
|
W
|
P <- origin/fix/windows-build-docs

local after rewrite:

O
|
W2 <- fix/windows-build-docs
```

`W2` 不是 `P` 的 descendant, 所以普通 push 會失敗. 這時更新 remote PR branch 需要:

```sh
git push --force-with-lease origin fix/windows-build-docs
```

這叫 force push PR branch.

## 為什麼用 --force-with-lease

`--force` 是直接改 remote branch pointer.

`--force-with-lease` 會先檢查:

```text
remote branch 是否還是我上次 fetch 時看到的樣子?
```

如果別人已經推了新 commit, 它會拒絕, 避免你覆蓋別人的工作.

所以日常規則是:

```text
不要裸用 --force.
需要 rewrite PR branch 時, 用 --force-with-lease.
```

## 為什麼不要 force push main

PR branch 通常短命:

```text
fix/windows-build-docs
  為一個 PR 存在
  review 後可以刪
```

`main` 通常是共享基準:

```text
main
  fork default branch
  可能被拿來開新 branch
  可能有人 fetch 過
```

如果你 force push `main`:

```text
old:
O -> A -> B <- origin/main

new:
O -> C      <- origin/main
```

其他人如果基於 `B` 工作, 歷史就分裂.

所以比較精確的規則是:

```text
可以在必要時整理自己的 short-lived PR branch.
不要隨便 rewrite shared main branch.
```

## PR #30 的例子

你看到 `sysprog21/libiui#30` 有 force push, 但它 force push 的不是 `main`.

它的形狀是:

```text
base: sysprog21/libiui:main
head: sysprog21/libiui:rendering
```

圖:

```text
main      O ------------------------ M
           \                        /
rendering   R1 -> R2 -> force -> R3
```

也就是 maintainer 在 `rendering` 這條 PR branch 上整理 commit, 最後 merge 整理後的結果到 `main`.

這不是:

```text
force push main
```

而是:

```text
整理 PR branch
```

## 如何檢查 PR 是否乾淨

GitHub PR diff 比較接近:

```text
merge-base(base, head)..head
```

本機可以用:

```sh
git diff upstream/main...HEAD --stat
git diff upstream/main...HEAD --name-only
```

檢查 personal files 是否進 PR:

```sh
git diff upstream/main...HEAD -- README.md notes AGENTS.md
```

理想結果:

```text
no output
```

## 本章學到的三件事

> [Summary]
> 1. PR 不是固定 patch, 而是 GitHub 持續比較 base branch 和 head branch.
> 2. 普通追加 commit 用 ordinary push, rewrite commit history 才需要 `--force-with-lease`.
> 3. force push PR branch 和 force push `main` 風險不同, 真正要看 branch 的用途和依賴者.

## 下一章

下一章把前面模型套回 `libiui-for-learn`: `main`, `learn/*`, `fix/*`, `notes/` 應該怎麼分工, 第一個 Windows PR 應該怎麼開.

讀: [ch04-libiui-repo-policy.md](ch04-libiui-repo-policy.md)

## Source Notes

```text
../0627_0026_pr-branch-concept.md
../0626_2320_github-fork-pr-model.md
../0626_1731_repo-state-cleanup-strategy.md
```
