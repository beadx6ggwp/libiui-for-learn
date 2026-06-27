# Ch01: Fork, Origin, Upstream In libiui Contribution

## 本章問題

你要為 `sysprog21/libiui` 做貢獻, 但本機 repo 叫 `libiui-for-learn`, GitHub 上又有 fork, upstream, `origin`, `upstream`, `main`, `origin/main`, `upstream/main`. 這些到底是不是同一種東西?

如果這個模型不清楚, 後面所有問題都會變得危險:

```text
我 fetch upstream 會不會改檔案?
我 push origin 是不是推給原作者?
我的 fork main 亂過, PR 會不會也亂?
我 reset main, notes 會不會消失?
```

> [Scope]
> 本章只建立為 `libiui` 做 upstream contribution 時需要的 fork 和 branch reference 模型. 還不處理 `reset`, `clean`, `force push` 的細節. 那些放到 Ch02 和 Ch03.

## 先看 naive 想法

naive 想法通常是:

```text
origin 聽起來像 original project.
upstream/main 好像就是 GitHub 上可以直接改的 main.
fork 好像只是 upstream repo 裡的一條 branch.
PR 好像會把整個 fork 丟給 upstream.
```

這些想法很自然, 但都會導致錯誤操作.

例如如果你以為 `origin` 是原作者 repo, 看到:

```sh
git push origin main
```

你會以為自己正在 push 到 `sysprog21/libiui`. 但在 fork workflow 裡, `origin` 通常是你自己的 fork.

## GitHub 上有兩個 repository

第一層先不要看 branch, 只看 GitHub repo:

```text
sysprog21/libiui
  upstream repository
  原作者和 maintainer 控制
  你通常沒有 push 權限

beadx6ggwp/libiui-for-learn
  your fork
  你有 push 權限
  可以保存學習筆記和實驗 branch
```

fork 不是 branch. fork 是另一個 repository.

可以把它想成:

```text
upstream repo = 對方的正史
your fork     = 你有寫入權限的工作副本
```

你要貢獻 `libiui` 時, 不是把整個 fork 合回去, 而是請 `sysprog21/libiui` 的 maintainer review 你 fork 裡某一條 branch 的差異.

## 本機 remote 名稱只是 URL 暱稱

在本機 repo 裡, 你替兩個 GitHub repository 取了名稱:

```text
origin   -> https://github.com/beadx6ggwp/libiui-for-learn.git
upstream -> https://github.com/sysprog21/libiui.git
```

這個設定可以用:

```sh
git remote -v
```

重要的是:

```text
origin
  你的 fork
  你會 push 到這裡

upstream
  原作者 repo
  你會 fetch 這裡的最新狀態
```

`origin` 和 `upstream` 不是 branch. 它們只是 remote name.

## Local Branch 和 Remote-Tracking Branch

接著才看 branch.

local branch 是你可以 checkout 和 commit 的 branch:

```text
main
learn/windows-first-attempt
fix/windows-build-docs
```

remote-tracking branch 是本機記住的 remote snapshot:

```text
origin/main
upstream/main
origin/learn/windows-first-attempt
```

這裡最重要的句子是:

```text
origin/main 不是 GitHub 上的 main 本體.
origin/main 是本機最後一次知道 origin 的 main 長什麼樣.
```

更新它要跑:

```sh
git fetch origin
```

同理:

```sh
git fetch upstream
```

會更新 `upstream/main` 這個本機 snapshot.

## Fetch 的方向

`fetch` 的方向是 remote -> local snapshot:

```text
sysprog21/libiui:main
        |
        | git fetch upstream
        v
local upstream/main
```

它不會把 `upstream/main` merge 進你目前的 `main`. 它只更新你本機對 upstream 的記憶.

這就是為什麼 `fetch upstream` 是安全的常用動作. 它讓你知道上游最新狀態, 但不會直接改 working tree.

## Branch 是指標, 不是資料夾

Git branch 不是一份複製的資料夾. 它只是指向某個 commit 的名字.

目前這個 repo 的乾淨狀態是:

```text
main          -> eb8d9c8
origin/main   -> eb8d9c8
upstream/main -> eb8d9c8
```

第一次嘗試 commit 是:

```text
e722b54 Add Windows build instructions and documentation
```

它被保存在:

```text
learn/windows-first-attempt
origin/learn/windows-first-attempt
```

圖像化:

```text
eb8d9c8  <- main, origin/main, upstream/main
   \
    e722b54 <- learn/windows-first-attempt, origin/learn/windows-first-attempt
```

這代表:

```text
main 可以乾淨地跟 upstream 對齊.
e722b54 仍然被 learn branch 保留.
兩件事可以同時成立.
```

## PR 不比較整個 fork

另一個關鍵誤解是:

```text
如果我的 fork 裡有學習筆記, PR 是否一定會包含它們?
```

答案是: 不一定.

GitHub PR 比較的是 `base branch` 和 `head branch`, 不是整個 fork.

例如:

```text
base: sysprog21/libiui:main
head: beadx6ggwp/libiui-for-learn:fix/windows-build-docs
```

GitHub 只看 `fix/windows-build-docs` 相對 upstream `main` 多了什麼. 如果你的 notes 沒在這條 branch 裡, reviewer 就不會看到 notes.

## 對這個 repo 的資料流

可以記這張圖:

```text
              Pull Request
origin/fix-* ----------------> upstream/main
      ^                             |
      |                             |
      | git push origin             | git fetch upstream
      |                             v
 local fix-*  <-------------  upstream/main snapshot
```

讀法:

```text
1. 從 upstream fetch 最新狀態.
2. 在 local 從 upstream/main 開 branch.
3. push 這條 branch 到 origin.
4. 用 GitHub PR 請 upstream 合併.
```

所以三個動作的方向完全不同:

```text
fetch upstream = 原作者 repo -> 本機 snapshot
push origin    = 本機 branch -> 你的 fork
open PR        = 你的 fork branch -> 原作者 main
```

## 本章學到的三件事

> [Summary]
> 1. `origin` 是你的 fork, `upstream` 是原作者 repo. 它們是 remote name, 不是 branch.
> 2. `origin/main` 和 `upstream/main` 是本機 remote-tracking snapshot, 不是你直接 commit 的 branch.
> 3. PR 比較的是 base/head branch, 不是整個 fork, 所以 learning branch 和 clean PR branch 可以分開存在.

## 下一章

下一章處理更危險的問題: 如果 branch 只是指標, 那 `reset --hard`, `git clean -fd`, `restore`, `cherry-pick`, `force push` 到底會動哪一層東西?

讀: [ch02-working-tree-branch-safety.md](ch02-working-tree-branch-safety.md)

## Source Notes

```text
../0626_2320_github-fork-pr-model.md
../0627_0013_git-operation-concepts.md
../0626_1731_repo-state-cleanup-strategy.md
```
