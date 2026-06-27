# libiui Git Contribution Notes

## 這個資料夾在解決什麼

這個資料夾不是一般 Git 教材, 也不是時間軸紀錄. 它把前面零散的 Git, GitHub fork, PR branch, force push, repo cleanup 問題, 統合成一組 "如何安全地為 `sysprog21/libiui` 做貢獻" 的教學文章.

一開始真正卡住的不是某一條指令, 而是你要同時滿足兩個目標:

```text
保留自己的學習歷程
提交給 upstream 一個乾淨 PR
```

為了做到這件事, 你必須理解多層概念:

```text
GitHub fork
origin / upstream
local branch / remote-tracking branch
working tree / index / commit
reset / clean / force push
PR base / head
learning branch / upstream PR branch
```

如果直接背指令, 很容易變成每次操作前都不確定會不會刪資料, 或不確定 PR 會不會把 notes 帶給 maintainer. 所以這組文章按照 "準備 libiui upstream PR" 需要的概念依賴排序.

> [Scope]
> 這裡的目標不是學完整 Git. 目標是讓你在 `D:\Github\libiui-for-learn` 裡能安全地保留學習歷程, 同時為 `sysprog21/libiui` 準備乾淨的 upstream PR.

## 建議閱讀順序

```text
ch01-fork-origin-upstream.md
  先建立 libiui fork contribution 的整體模型, 區分 origin, upstream, local branch, remote-tracking branch.

ch02-working-tree-branch-safety.md
  再理解準備 PR 前最容易誤用的 working tree, index, commit, reset, clean, restore, cherry-pick.

ch03-pr-branch-force-push.md
  接著理解 GitHub PR 的 base/head 模型, 以及 reviewer 要你修改時該如何更新 PR branch.

ch04-libiui-repo-policy.md
  接著把前面模型套到目前 repo, 決定 main, learn/*, fix/*, notes/ 在 libiui contribution workflow 裡各自的用途.

ch05-diff-range-and-merge-base.md
  再理解 `A..B`, `A...B`, `merge-base`, GitHub PR diff, 以及 upstream/main 前進後要不要 rebase.

ch06-rebase-conflict-and-recovery.md
  接著處理 rebase conflict, partial staging, tracking branch, stash, reflog recovery.

ch07-learning-branch-workflow.md
  最後整理 `learn/repo-review` 的操作 flow, 包含何時同步 upstream, feature branch 旁邊的 notes 怎麼保存, 以及 PR branch 如何避免混入 notes.
```

## 和時間軸筆記的關係

時間戳筆記保存的是當時怎麼想:

```text
../0626_2320_github-fork-pr-model.md
../0627_0013_git-operation-concepts.md
../0627_0026_pr-branch-concept.md
../0626_1731_repo-state-cleanup-strategy.md
```

這個資料夾保存的是整理後的教學版本. 也就是:

```text
timestamped notes = 思考歷史和操作證據
notes/libiui-git/ = libiui contribution workflow 教學文章
```

不要用這個資料夾取代時間軸筆記. 兩者用途不同.

## 這組文章的寫法

每章都盡量遵守同一個結構:

```text
1. 本章要回答的問題
2. 章節範圍
3. naive 想法
4. naive 想法為什麼失敗
5. 正確模型
6. 對 libiui-for-learn / sysprog21/libiui 的具體操作
7. 本章學到的三件事
8. 下一章連結
```

這是參考 `learning-note` 的方式, 但保留 Markdown, 因為這個 repo 的 notes 目前以 `.md` 為主.
