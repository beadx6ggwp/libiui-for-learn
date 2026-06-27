# PR Branch Concept

## Question

`PR branch` 是什麼? 為什麼有人說可以 force push PR branch, 但又說不要亂 force push?

## The Three Objects

先不要把 PR 想成一個神秘物件. 一個 GitHub PR 其實同時牽涉三個東西:

```text
1. local branch       你電腦上的 branch
2. remote branch      GitHub fork 上的 branch
3. GitHub PR object   GitHub 記錄的比較關係
```

以 Windows support 為例:

```text
local branch:   fix/windows-build-docs
remote branch:  origin/fix/windows-build-docs
GitHub PR:      compare sysprog21/libiui:main with beadx6ggwp/libiui-for-learn:fix/windows-build-docs
```

所以 `PR branch` 通常指的是 `head branch`, 也就是 PR 的來源 branch.

## Base And Head

Pull Request 的核心模型是:

```text
base branch <- head branch
目標 branch    來源 branch
```

對你的情境來說:

```text
base: sysprog21/libiui:main
head: beadx6ggwp/libiui-for-learn:fix/windows-build-docs
```

意思是: "請把 head branch 上相對於 base branch 多出來的改動, 合併回 base branch."

## Case 1: 開 PR 前

假設目前 `main` 跟 `upstream/main` 都在同一個 commit:

```text
O  eb8d9c8  <- upstream/main, origin/main, main
```

你從這裡切一個新 branch:

```sh
git switch -c fix/windows-build-docs
```

這時只是多一個 branch name 指到同一個 commit:

```text
O  eb8d9c8  <- upstream/main, origin/main, main, fix/windows-build-docs
```

接著你改 README, commit 一次:

```text
O  eb8d9c8  <- upstream/main, origin/main, main
|
W  add Windows docs  <- fix/windows-build-docs
```

這裡的 `W` 就是你想投稿的改動.

## Case 2: Push 後開 PR

你 push 這個 branch 到自己的 fork:

```sh
git push origin fix/windows-build-docs
```

GitHub 上會多一個 remote branch:

```text
O  eb8d9c8  <- upstream/main, origin/main, main
|
W  add Windows docs  <- fix/windows-build-docs, origin/fix/windows-build-docs
```

此時開 PR, GitHub 記住的是:

```text
PR #X
base = sysprog21/libiui:main -> O
head = beadx6ggwp/libiui-for-learn:fix/windows-build-docs -> W
```

PR 頁面顯示的 diff 不是一包死掉的 patch. 它是 GitHub 即時計算:

```text
diff = base..head = O..W
```

所以 head branch 變了, PR 內容就會變.

## Case 3: Reviewer 要你追加修改

假設 reviewer 說 README 還缺 MSYS2 package list. 你在同一個 branch 上再 commit:

```text
O  eb8d9c8  <- upstream/main, origin/main, main
|
W  add Windows docs
|
P  add MSYS2 packages  <- fix/windows-build-docs
```

普通 push 就可以:

```sh
git push origin fix/windows-build-docs
```

push 後:

```text
O  eb8d9c8  <- upstream/main, origin/main, main
|
W  add Windows docs
|
P  add MSYS2 packages  <- fix/windows-build-docs, origin/fix/windows-build-docs
```

GitHub PR 自動更新成:

```text
base = O
head = P
diff = O..P
```

這就是 "改同一個 PR branch, 不用重開 PR".

## Case 4: Amend 後為什麼需要 Force Push

另一種情況: reviewer 不是要你新增一個修補 commit, 而是希望 commit 本身更乾淨. 例如原本:

```text
O  eb8d9c8
|
W  add Windows docs
|
P  typo fix
```

如果你用 `git rebase -i` 或 `git commit --amend` 把 `P` 合回 `W`, 會產生一個新的 commit `W2`:

```text
before:

O
|
W
|
P  <- origin/fix/windows-build-docs

after local rewrite:

O
|
W2 <- fix/windows-build-docs
```

注意: `W2` 不是 `P` 往後長出來的 descendant. 對 Git 來說, 這不是 fast-forward.

所以普通 push 會被拒絕. 你要明確告訴 GitHub: "我要把 remote PR branch 從 `P` 改指到 `W2`."

```sh
git push --force-with-lease origin fix/windows-build-docs
```

force push 後:

```text
O
|
W2 <- fix/windows-build-docs, origin/fix/windows-build-docs
```

GitHub PR 仍然是同一個 PR:

```text
PR #X
base = O
head = W2
diff = O..W2
```

這就是 "force push PR branch". 它改的是 `head branch` 指向哪個 commit, 不是直接改 `upstream/main`.

## Case 5: 為什麼不要 Force Push Main

如果你 force push 的不是 PR branch, 而是 `main`, 風險就不同.

原本共享主線:

```text
O -> A -> B  <- origin/main
```

你 force push 成:

```text
O -> C       <- origin/main
```

其他人如果已經基於 `B` 工作:

```text
O -> A -> B  <- other-user-work
 \
  C          <- origin/main
```

大家的歷史就分裂了. 這就是為什麼一般說 "不要 force push main".

差別在於:

```text
PR branch:  通常只服務一個 PR, 可以為 review 整理 history
main:       共享主線, 很多人可能已經依賴它
```

## Case 6: PR #30 的形狀

`sysprog21/libiui#30` 不是 force push `main`. 它的形狀是:

```text
base: sysprog21/libiui:main
head: sysprog21/libiui:rendering
```

也就是:

```text
main      O ------------------------ M  <- merge result
           \                        /
rendering   R1 -> R2 -> force -> R3
```

GitHub event 顯示 jserv force push 的是 `rendering`, 也就是 PR branch. 最後 merge 到 `main` 的是整理後的結果.

所以這個操作比較像:

```text
1. 在 rendering branch 上整理 commit
2. 讓 PR 顯示乾淨 diff
3. merge 整理後的 branch 到 main
4. 刪掉 rendering branch
```

它不是:

```text
1. 把 main 的 public history 改掉
```

## Case 7: 對你的 repo 怎麼套用

你的 fork 可以分成幾種 branch:

```text
origin/main                    跟 upstream/main 對齊
learn/windows-first-attempt     保存第一次嘗試與 jserv 留言脈絡
learn/notes                     保存學習筆記
fix/windows-build-docs          真正準備送 PR 的 branch
```

投稿時:

```text
upstream/main
     |
     O  clean base
     |
     W  minimal Windows docs fix  <- fix/windows-build-docs, origin/fix/windows-build-docs
```

GitHub PR:

```text
base = sysprog21/libiui:main
head = beadx6ggwp/libiui-for-learn:fix/windows-build-docs
```

這時 maintainer review 的不是你的 `learn/notes`, 也不是 `learn/windows-first-attempt`. 他只看:

```text
diff = upstream/main..fix/windows-build-docs
```

## Mental Model

把 PR 想成一個會自動更新的比較器:

```text
PR = compare(base branch, head branch)
```

所以真正會改變 PR 內容的是 `head branch` 指到哪裡:

```text
head -> W   PR 顯示 O..W
head -> P   PR 顯示 O..P
head -> W2  PR 顯示 O..W2
```

`force push PR branch` 的意思就是: 重設這個 `head` 指標.

## Rule Of Thumb

- PR branch 是投稿用工作 branch.
- PR 會追蹤 head branch, 不是保存一次性的 patch.
- 普通追加 commit 用普通 push.
- amend, squash, rebase 後才需要 `--force-with-lease`.
- 可以整理自己的 PR branch, 但不要隨便 force push `main`.
- 不確定時先畫出 `base` 和 `head` 指到哪個 commit.
