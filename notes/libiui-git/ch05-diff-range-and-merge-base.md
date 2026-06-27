# Ch05: Diff Range And Merge Base

## 本章問題

你在準備 `libiui` PR 時會一直看到這幾個指令:

```sh
git log upstream/main..HEAD
git diff upstream/main...HEAD --stat
git diff upstream/main...HEAD --name-only
```

naive 想法是:

```text
兩個點和三個點應該差不多.
反正都是比較 upstream/main 和 HEAD.
```

但這個直覺會害你看錯 PR diff. GitHub PR 不是單純比較兩個 branch tip 的檔案差異, 它更接近 "從共同祖先開始, 看 head branch 帶來什麼改動".

> [Scope]
> 本章只解釋 `A..B`, `A...B`, `merge-base`, 以及它們在 `libiui` PR 檢查中的用途. `rebase` 和 conflict 放到 Ch06.

## 先建立圖上的物件

假設 `upstream/main` 是 `O`, 你的 PR branch 是 `fix/msys2-ucrt-clock`:

```text
O <- upstream/main
 \
  F <- fix/msys2-ucrt-clock, HEAD
```

這時共同祖先是 `O`:

```sh
git merge-base upstream/main HEAD
```

輸出會是 `O` 的 commit hash.

`merge-base` 的意思是:

```text
兩條 history 分開之前, 最後共同擁有的 commit.
```

## `A..B`: 看 B 有但 A 沒有的 commits

`git log upstream/main..HEAD` 的問題是:

```text
HEAD 有哪些 commits 不在 upstream/main 裡?
```

圖:

```text
O <- upstream/main
 \
  F1 --- F2 <- HEAD
```

命令:

```sh
git log --oneline upstream/main..HEAD
```

結果會列出:

```text
F2
F1
```

所以兩個點常用來看 PR branch 裡有幾個 commit:

```sh
git log --oneline upstream/main..HEAD
```

它回答的是 commit list, 不是完整檔案比較模型.

## `A...B`: 從共同祖先看 B 的改動

`git diff upstream/main...HEAD` 的問題是:

```text
從 merge-base(upstream/main, HEAD) 到 HEAD, 檔案改了什麼?
```

在簡單圖裡:

```text
O <- upstream/main
 \
  F <- HEAD
```

`merge-base(upstream/main, HEAD)` 是 `O`, 所以:

```sh
git diff upstream/main...HEAD
```

等價於:

```sh
git diff O HEAD
```

這正好接近 GitHub PR 的直覺:

```text
base branch 和 head branch 分開後, head branch 帶來了什麼?
```

## 為什麼 upstream 前進時三個點更重要

真正容易混淆的是 upstream 在你開 branch 後前進:

```text
O --- U <- upstream/main
 \
  F <- HEAD
```

現在:

```text
upstream/main tip = U
HEAD tip          = F
merge-base        = O
```

如果你用:

```sh
git diff upstream/main HEAD
```

這是在比較 `U` 和 `F` 的最終檔案狀態. 它可能混入 upstream 從 `O` 到 `U` 的變化, 讓你誤以為自己的 PR 改了更多東西.

但:

```sh
git diff upstream/main...HEAD
```

是比較:

```text
O -> F
```

也就是只看你的 PR branch 從共同祖先以後做了什麼.

圖解:

```text
Full graph:

O --- U <- upstream/main
 \
  F <- HEAD

git diff upstream/main HEAD:
  compare U vs F

git diff upstream/main...HEAD:
  compare O vs F
```

## GitHub PR 用哪個模型

GitHub PR 的 core model 是:

```text
base: sysprog21/libiui:main
head: beadx6ggwp/libiui-for-learn:fix/...
```

它要回答:

```text
如果把 head branch merge 到 base branch, 會帶入哪些 commits and changes?
```

所以本機檢查 PR diff 時, 優先用:

```sh
git diff upstream/main...HEAD --stat
git diff upstream/main...HEAD --name-only
git diff upstream/main...HEAD -- README.md notes AGENTS.md
```

這組指令的意義是:

```text
stat
  看改動規模是否符合 PR title.

name-only
  看檔案清單是否 upstream-facing.

path-limited diff
  確認 personal notes 沒有進 PR diff.
```

## 上游前進後要不要 rebase

如果 upstream 前進:

```text
O --- U <- upstream/main
 \
  F <- HEAD
```

你有兩種選擇.

不 rebase:

```text
O --- U <- upstream/main
 \
  F <- HEAD
```

適合:

```text
PR 還能 clean merge.
CI 沒有因為舊 base 失敗.
maintainer 沒要求更新 base.
```

rebase:

```text
O --- U <- upstream/main
       \
        F2 <- HEAD
```

指令:

```sh
git fetch upstream
git rebase upstream/main
git push --force-with-lease origin fix/msys2-ucrt-clock
```

適合:

```text
upstream 改到同一區域.
PR 出現 conflict.
CI 必須在新版 upstream/main 上驗證.
maintainer 要你 rebase.
```

不要為了 "看起來比較新" 而頻繁 rebase. Rebase 的價值是讓 PR diff 和 validation 對齊目前 base, 不是每天整理一次歷史.

## 常用檢查組合

看 commits:

```sh
git log --oneline upstream/main..HEAD
```

看 PR diff 檔案:

```sh
git diff upstream/main...HEAD --name-only
```

看 PR diff 內容:

```sh
git diff upstream/main...HEAD
```

看共同祖先:

```sh
git merge-base upstream/main HEAD
```

看圖:

```sh
git log --oneline --decorate --graph --all -n 30
```

## 本章學到的三件事

> [Summary]
> 1. `A..B` 常用來看 B 有但 A 沒有的 commits, 適合檢查 PR commit list.
> 2. `A...B` 在 `git diff` 裡表示從 `merge-base(A, B)` 比到 `B`, 更接近 GitHub PR diff.
> 3. upstream 前進後不一定要 rebase. 只有 conflict, CI, base freshness, or maintainer request 需要時再 rebase.

## 下一章

下一章處理 rebase 真的發生 conflict 時怎麼辦, 以及如何用 partial staging 和 reflog 把錯誤操作控制在可恢復範圍內.

讀: [ch06-rebase-conflict-and-recovery.md](ch06-rebase-conflict-and-recovery.md)

## Source Notes

```text
../0627_0013_git-operation-concepts.md
../0627_0026_pr-branch-concept.md
ch03-pr-branch-force-push.md
ch04-libiui-repo-policy.md
```
