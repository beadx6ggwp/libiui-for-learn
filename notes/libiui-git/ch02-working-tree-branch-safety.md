# Ch02: Safe Local Operations Before libiui PR

## 本章問題

你已經知道 `origin`, `upstream`, branch, remote-tracking branch 不同. 下一個問題是: 在準備 `libiui` PR 前, Git 指令到底會改哪裡?

這很重要, 因為同樣看起來像 "清理 repo" 的指令, 風險完全不同:

```text
git fetch upstream
git reset --hard upstream/main
git clean -fd
git restore --source=e722b54 -- tests/example.c
git push --force-with-lease origin main
```

它們改的層次不同. 如果分不清楚, 就會不知道 notes 會不會被刪, old commit 會不會消失, PR branch 會不會被污染.

> [Scope]
> 本章只講本機狀態和指令風險, 目標是避免把 notes, first attempt, 或 unrelated README 內容帶進 `libiui` PR. PR base/head 和 force push PR branch 的協作語意放到 Ch03.

## 先建立三層本機狀態

在本機 repo 裡, 至少要分三層:

```text
working tree
  你檔案系統上看到的實際檔案

index / staging area
  git add 後準備進下一個 commit 的內容

commit history
  已經被 Git 記錄下來的 snapshots
```

`git status --short` 可以判斷檔案在哪一層:

```text
?? notes/
  untracked, 還沒有進 Git

 M src/input.c
  tracked file 在 working tree 被修改, 還沒 staged

M  src/input.c
  tracked file 的修改已 staged
```

這對你的 notes 很關鍵. `?? notes/` 代表它還不屬於任何 commit, 也不是任何 branch 的一部分.

## Naive 想法: reset 會把整個資料夾變回去

很自然會以為:

```text
git reset --hard upstream/main
會把整個資料夾變成 upstream/main 的樣子
所以 notes 也會消失
```

這個說法只對一半.

`reset --hard` 會重設 tracked files, 也就是已經在 Git 管理中的檔案. 但它不會刪 untracked files.

所以:

```text
tracked README.md change
  reset --hard 會丟掉

untracked notes/
  reset --hard 不會刪
```

真正會刪 untracked files 的是:

```sh
git clean -fd
```

## Reset 的真正語意

`git reset` 的核心是移動目前 local branch pointer.

三種常見模式:

```text
git reset --soft <commit>
  移動 branch pointer
  保留 index
  保留 working tree

git reset --mixed <commit>
  移動 branch pointer
  重設 index
  保留 working tree

git reset --hard <commit>
  移動 branch pointer
  重設 index
  重設 tracked working tree
```

對這個 repo 來說:

```sh
git reset --hard upstream/main
```

意思是:

```text
讓目前 branch 指到 upstream/main 指向的 commit.
tracked files 變成 upstream/main 的內容.
tracked local modifications 被丟掉.
untracked files 保留.
```

所以之前你執行 `reset --hard upstream/main` 後, 時間軸 notes 還在, 是符合 Git 模型的.

## Clean 的危險

`git clean` 專門處理 untracked files.

```sh
git clean -fd
```

會刪:

```text
untracked files
untracked directories
```

如果 `notes/` 還是 `??`, 它會被刪.

永遠先 dry run:

```sh
git clean -fdn
```

`-n` 代表 preview. 它只列出會刪什麼, 不真的刪.

可以這樣記:

```text
reset --hard
  對 tracked files 危險

clean -fd
  對 untracked files 危險
```

## Read-Only 指令要先養成習慣

在任何有風險操作前, 先跑 read-only commands:

```sh
git status --short --branch
git diff --stat
git diff --cached --stat
git branch -vv
git log --oneline --decorate --graph --all -n 30
```

你要先回答:

```text
我現在在哪個 branch?
有沒有 tracked modifications?
有沒有 staged changes?
有沒有 untracked notes?
HEAD 附近的 commit graph 長怎樣?
```

如果這些都不知道, 就不要先下 `reset --hard` 或 `clean -fd`.

## 從舊 commit 拿一個檔案

`e722b54` 這個第一次嘗試 commit 混了兩種東西, 所以它是很好的 libiui PR 拆分案例:

```text
README.md
  個人教學和學習脈絡

tests/example.c
  可能有 upstream-facing Windows build fix
```

naive 做法是整個 cherry-pick:

```sh
git cherry-pick e722b54
```

問題是這會把 `README.md` 也帶進 PR branch.

對 `libiui` contribution 來說, 更好的做法是只拿需要的檔案:

```sh
git switch -c fix/msys2-ucrt-clock upstream/main
git restore --source=e722b54 -- tests/example.c
git diff -- tests/example.c
```

這樣你只把 `tests/example.c` 放進 working tree, 接著可以手動整理 patch.

## 如果真的需要拆整個 commit

有時你想先套整個 commit 再刪掉不要的部分. 那就用 no-commit cherry-pick:

```sh
git cherry-pick -n e722b54
```

`-n` 代表不直接產生 commit.

接著把不該進 PR 的檔案還原:

```sh
git restore --source=upstream/main --staged --worktree README.md
git status
git diff --stat
```

這比直接 cherry-pick 安全, 因為你還有機會在 commit 前檢查 diff.

## Commit Reachability

另一個常見恐懼是:

```text
我 force push 或 reset 後, e722b54 會不會變孤兒節點?
```

判斷方式:

```sh
git branch -a --contains e722b54
```

如果輸出包含:

```text
learn/windows-first-attempt
remotes/origin/learn/windows-first-attempt
```

表示這個 commit 仍然被 branch 可達, 不是 orphan.

如果真的失手, 可以查:

```sh
git reflog
```

找到 commit 後立刻建立 branch:

```sh
git branch rescue/<name> <commit>
```

但 `reflog` 是最後防線, 不是日常工作流. 日常上, 改 history 前先建 branch 才是穩定做法.

## 本章學到的三件事

> [Summary]
> 1. `reset --hard` 會丟 tracked local changes, 但不刪 untracked notes.
> 2. `git clean -fd` 會刪 untracked files, 所以要先用 `git clean -fdn` preview.
> 3. 從混合 commit 重用內容時, 優先拿單一檔案或用 `cherry-pick -n`, 不要無腦整個 cherry-pick.

## 下一章

下一章處理 GitHub PR. 你會看到為什麼 PR branch 可以被更新, 為什麼 reviewer 要你改時不用重開 PR, 以及為什麼 force push PR branch 和 force push `main` 是不同風險.

讀: [ch03-pr-branch-force-push.md](ch03-pr-branch-force-push.md)

## Source Notes

```text
../0627_0013_git-operation-concepts.md
../0626_1731_repo-state-cleanup-strategy.md
../0627_0026_pr-branch-concept.md
```
