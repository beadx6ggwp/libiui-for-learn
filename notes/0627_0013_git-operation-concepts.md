# Git Operation Concepts

## Purpose

這篇筆記記錄在整理 `libiui-for-learn` 過程中遇到的 Git 操作觀念。它不是單次操作紀錄，而是之後遇到 branch、reset、force push、PR 準備時可以回頭查的教學筆記。

## Local Branch, Remote, And Remote-Tracking Branch

Git 裡常看到三種名字，容易混在一起：

- `main`：本機 branch，可以直接 checkout、commit、reset。
- `origin`：你的 fork remote，這裡是 `beadx6ggwp/libiui-for-learn`。
- `upstream`：原作者 remote，這裡是 `sysprog21/libiui`。
- `origin/main`：本機記住的「上次 fetch/push 時，origin 的 main 長怎樣」。
- `upstream/main`：本機記住的「上次 fetch 時，upstream 的 main 長怎樣」。

重點是：`origin/main` 和 `upstream/main` 不是你正在編輯的 branch，而是 remote-tracking branch。要更新它們，通常要先跑：

```sh
git fetch origin
git fetch upstream
```

## Reset Moves A Branch Pointer

`git reset --hard upstream/main` 的意思是：

1. 把目前 checkout 的本機 branch 移到 `upstream/main` 指向的 commit。
2. 讓 working tree 和 index 都變成那個 commit 的內容。
3. 刪掉 tracked files 裡尚未 commit 的修改。

它不會刪 untracked files，所以目前 `notes/*.md` 如果還是 `??`，不會因為 `reset --hard` 消失。

危險的是：

```sh
git clean -fd
```

這會刪 untracked files，因此會刪掉尚未被 Git 追蹤的 notes。

## Force Push Changes The Remote Branch Pointer

`git push --force-with-lease origin main` 的本質是：把 GitHub 上 `origin/main` 的 branch pointer 改成你本機 `main` 指向的 commit。

它不會直接刪 commit object；它只是讓某個 branch 不再指向原本的 commit。如果原本的 commit 還有其他 branch、tag、PR、或 reflog 指到它，就還能找回。

這次 `e722b54` 已經由以下 branch 保住：

```text
learn/windows-first-attempt
origin/learn/windows-first-attempt
```

所以即使 `origin/main` 被重設回 `upstream/main`，`e722b54` 仍不是 orphan commit。

## Commit Reachability

判斷某個 commit 是否仍被 branch 保住，可以用：

```sh
git branch -a --contains e722b54
```

如果輸出裡有 branch，例如：

```text
learn/windows-first-attempt
remotes/origin/learn/windows-first-attempt
```

代表這個 commit 仍然可達，不是孤立節點。

## Clean PR Workflow

如果目標是投稿給 upstream，推薦流程是：

```sh
git fetch upstream
git switch main
git reset --hard upstream/main
git switch -c fix/windows-build-docs
```

接著只放「要給原作者 review 的最小改動」。學習筆記、實驗紀錄、個人規劃不要放進 PR branch。

## Personal Learning Branch

個人學習內容可以放在獨立 branch，例如：

```text
learn/windows-first-attempt
learn/notes
```

這類 branch 的目的不是投稿，而是保留脈絡、筆記、實驗歷史。它不需要 merge 到每個開發 branch；真正要開 PR 時，從乾淨的 `upstream/main` 另外切 branch。

## Rule Of Thumb

- 要同步原作者狀態：`git fetch upstream`。
- 要讓本機 `main` 回到原作者：`git reset --hard upstream/main`。
- 要保留舊 commit：先建立 branch 或 tag。
- 要確認 commit 是否安全：`git branch -a --contains <commit>`。
- 要刪 untracked files 前：先非常小心檢查 `git status --short --untracked-files=all`。
- 要投稿 PR：從乾淨的 upstream base 開新 branch。
