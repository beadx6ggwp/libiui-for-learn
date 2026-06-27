# Ch04: libiui Contribution Branch Policy

## 本章問題

前面三章建立的是為 `libiui` contribution 服務的 Git/PR 模型. 現在要回答這個 repo 的實際政策:

```text
main 應該拿來做什麼?
learn/windows-first-attempt 要不要保留?
notes/ 要不要進 Git?
未來 upstream PR branch 要從哪裡開?
第一個 Windows support PR 應該怎麼整理?
```

> [Scope]
> 本章是 `D:\Github\libiui-for-learn` 的 repo-specific policy. 它不是所有專案都必須照做, 但它適合目前這個 fork 的學習需求, 以及向 `sysprog21/libiui` 提交乾淨 PR 的需求.

## 目前已確認的狀態

目前乾淨基準是:

```text
main          = eb8d9c83f828ac50412d7d51e7c52b2678c09450
origin/main   = eb8d9c83f828ac50412d7d51e7c52b2678c09450
upstream/main = eb8d9c83f828ac50412d7d51e7c52b2678c09450
```

第一次嘗試 commit:

```text
e722b54d289b3a833ba5eafde2ce60668bbfcbcf
Add Windows build instructions and documentation
```

已由以下 branch 保存:

```text
learn/windows-first-attempt
origin/learn/windows-first-attempt
```

這表示:

```text
main 現在是乾淨的.
e722b54 沒有消失.
jserv 在該 commit 上的 comment 還有公開上下文.
```

## 為什麼 e722b54 不該直接當 PR

`e722b54` 的問題不是完全沒有價值. 它的問題是混合了不同用途:

```text
README.md
  Windows/MSYS2 教學
  個人學習方向
  Pixel-Renderer 後續想法

tests/example.c
  可能有真正 upstream-facing build fix
```

open-source PR 需要的是:

```text
one problem
one root cause
one focused patch
clear validation
```

所以 `e722b54` 的定位應該是:

```text
first attempt
learning history
public context
not direct PR branch
```

## Branch 分工

建議政策:

```text
main
  keep clean
  stay close to upstream/main
  use as stable baseline

upstream/main
  original project baseline
  fetch from it
  never commit directly on it

origin/main
  your fork default branch
  keep clean if possible

learn/*
  learning notes
  first attempts
  experiments
  context preservation

fix/*
  upstream PR branches
  one focused change per branch

notes/
  thinking records and topic teaching notes
  should not enter upstream PR by accident
```

最短版:

```text
main follows upstream.
learn/* preserves learning.
fix/* submits PR.
notes stay out of PR.
```

## Notes 的位置

現在採用兩種 notes:

```text
notes/MMDD_HHMM_short-title.md
  時間軸想法
  問題
  操作紀錄
  暫時推理

notes/<topic>/
  整理後的教學文章
  例如 notes/libiui-git/
```

這種命名直接對應主題. 因為整理後的內容不一定只屬於正式開發前知識, 也不只是行政分類. 它是某個主題的教學整理.

## 第一個乾淨 Windows PR

不要從 `learn/windows-first-attempt` 開. 從 `upstream/main` 開:

```sh
git fetch upstream
git switch -c fix/msys2-ucrt-clock upstream/main
```

如果只需要拿 `e722b54` 裡的 `tests/example.c`:

```sh
git restore --source=e722b54 -- tests/example.c
git diff -- tests/example.c
```

接著手動整理:

```text
remove personal comments
match upstream C style
keep patch minimal
record exact build failure
record validation command
```

如果最後判斷是 documentation PR, 就只動 upstream 需要的 documentation. 如果是 build fix, 就只動必要的 source 或 build files. 不要把 README 教學, code fix, personal plan 混成一個 PR.

## 開 PR 前固定檢查

在 PR branch 上:

```sh
git status --short --branch
git log --oneline upstream/main..HEAD
git diff upstream/main...HEAD --stat
git diff upstream/main...HEAD --name-only
git diff upstream/main...HEAD -- README.md notes AGENTS.md
```

你要看到:

```text
branch name is fix/*
commit list is short
diff paths are upstream-facing
notes/ and AGENTS.md are absent
```

如果最後一個指令有輸出, 代表 learning content 進了 PR branch.

## Commit Message 風格

upstream 最近 commit subject 類型:

```text
Fix WASM mouse by restoring g_wasm_ctx each frame
Fix switch boundary hit testing (#27)
Replace $(shell) with portable $(python)
CI: Add Clang Static Analyzer
Add small FAB, range slider, three-line list item
```

可歸納:

```text
imperative verb
short subject
no trailing period
scope prefix only when useful, such as CI:
describe behavior, not your local process
```

比較適合的 subject:

```text
Fix MSYS2 UCRT64 demo build
Document Windows build requirements
```

不適合:

```text
Update files
Windows stuff
notes and fix
first try
```

## 如果 notes 要版本控制

可以開 learning branch:

```sh
git switch -c learn/notes main
git add AGENTS.md notes/
git commit -m "Add libiui learning notes"
git push origin learn/notes
```

但不要:

```text
merge learn/notes into fix/*
open upstream PR from learn/notes
```

learning branch 的目的不是 upstream review. 它是保存自己的思考脈絡.

## 如果想完全隔離 PR workspace

用 `git worktree`:

```sh
git worktree add ..\libiui-pr-windows upstream/main
```

得到:

```text
D:\Github\libiui-for-learn
  learning workspace
  notes and topic folders

D:\Github\libiui-pr-windows
  clean PR workspace
  no notes pollution
```

這是最不容易混淆的做法. 缺點只是多一個資料夾.

## 本章學到的三件事

> [Summary]
> 1. `main` 現在應該維持乾淨, `learn/*` 用來保存學習和第一次嘗試, `fix/*` 才是 upstream PR branch.
> 2. `e722b54` 應該當作 first attempt 和 context, 不應該直接當 PR 起點.
> 3. 開 PR 前一定要檢查 `notes/`, `AGENTS.md`, personal README 內容沒有進入 diff.

## 下一章

下一章補上 PR diff 的核心概念: `A..B`, `A...B`, `merge-base`, 以及為什麼本機檢查 PR 時常用 `git diff upstream/main...HEAD`.

讀: [ch05-diff-range-and-merge-base.md](ch05-diff-range-and-merge-base.md)

## Source Notes

```text
../0626_1731_repo-state-cleanup-strategy.md
../0627_0051_libiui-development-model.md
../0626_1723_upstream-pr-submission-plan.md
../0626_1726_jserv-pr-guidelines.md
```
