# Ch06: Upstream CONTRIBUTING Guide

## 本章問題

前面幾章用 commit history, PR 討論, CI, 以及 `libiui` 的 code shape 來推導貢獻方式. 但還有一個更直接的來源:

```text
CONTRIBUTING.md
```

如果忽略這份文件, 前面的結論就容易像是臆測. 正確做法是把來源分成兩層:

```text
upstream explicit rules
  CONTRIBUTING.md 明文要求

local interpretation
  我們根據 history, PR, CI, repo structure 推導出的實作工作流
```

> [Scope]
> 本章不重複整份 `CONTRIBUTING.md`. 它只整理對第一個 libiui PR 最有影響的規則, 並說明它們如何回扣到 Ch01-Ch05.

## 最重要的結論

`CONTRIBUTING.md` 的核心不是某一條格式規則, 而是:

```text
small coherent changes
clear discussion for non-trivial work
style consistency
test before committing
commit history readable months later
```

這和前面整理出的 PR 因果鏈一致:

```text
problem -> root cause -> minimal change -> validation -> maintainability
```

差別是 `CONTRIBUTING.md` 是 upstream 明文規範, 所以前面章節應該以它為地基, 再用 commit history 和 PR discussion 補細節.

## Issues: 什麼時候先討論

`CONTRIBUTING.md` 明確建議, bug report, feature request, substantial PR 應先開 GitHub Issue 討論. 目的不是形式流程, 而是避免投入大量 effort 後才發現方向不符合 project design.

對 `libiui` 來說:

```text
minor typo, small refactor, small docs/comments
  通常不需要先開 issue

new feature, larger refactor, architecture decision, uncertain design
  先開 issue or draft PR 比較合理
```

這補強 Ch03 的 draft PR 觀念: draft PR 不是把問題丟給 maintainer, 而是帶著目前 reasoning 去確認方向.

## Style: 先跟工具對齊

`CONTRIBUTING.md` 明文列出三類 formatter:

```text
clang-format 20+
shfmt
black 25.1.0
```

並提供整體格式化入口:

```sh
make format
```

所以 Ch04 的 evidence 不應只寫:

```text
make passed
```

對 code-related PR, 你還要知道自己是否碰到:

```text
C formatting
shell formatting
Python formatting
newline at EOF
```

如果 PR 只改 C source, 最少要確認 `.clang-format` 和 project style. 如果改 `scripts/*.py`, 就要注意 `black`. 如果改 `.ci/*.sh`, 就要注意 `shfmt`.

## Commit Rules: 不是 Conventional Commits

`CONTRIBUTING.md` 引用 "How to Write a Git Commit Message" 的七條規則:

```text
subject 和 body 用空白行分開
subject 約 50 字
subject 首字母大寫
subject 不用句點結尾
使用 imperative mood
body wrap around 72 columns
body 解釋 what and why, not how
```

它沒有要求 Conventional Commits. 所以我們前面建議:

```text
Fix MSYS2 UCRT64 demo clock build
Document Windows build requirements
```

比下面更貼近 upstream:

```text
fix: fix windows build
docs: update docs
```

這補強 Ch02 的結論: subject 要像 upstream history, 也要符合 `CONTRIBUTING.md` 的 commit message rules.

## Commit Scope: 一個 coherent change

`CONTRIBUTING.md` 明確要求 related changes grouped together, 並避免 work-in-progress commits. 這直接支持我們對 `e722b54` 的判斷:

```text
README personal learning context
Windows build notes
tests/example.c possible code fix
```

這些不應直接變成一個 upstream PR commit. 它們可以保留在 `learn/*`, 但正式 `fix/*` branch 要拆成 coherent upstream patch.

## Docs: English Dialect And Audience

`CONTRIBUTING.md` 要求 documentation, comments, materials 使用 American English. 這代表 upstream docs 要寫成:

```text
initialize
color
behavior
```

而不是:

```text
initialise
colour
behaviour
```

這也再次區分:

```text
notes/
  personal learning, Traditional Chinese is fine

upstream docs
  user-facing English, en_US, reproducible, neutral
```

所以 Ch04 說 `notes/` 不該直接進 upstream PR, 不是個人偏好, 而是 audience 不同.

## 對第一個 PR 的實際 Checklist

送 PR 前, 最低限度要能回答:

```text
Issue:
  這是 minor fix, 還是需要先 issue/draft PR?

Scope:
  commit 是否只包含 coherent upstream change?

Style:
  是否符合 C, shell, or Python formatting?

Commit:
  subject 是否 imperative, short, no trailing period?
  body 是否只在需要時解釋 what and why?

Validation:
  是否跑了和改動相符的 tests or build commands?

Docs:
  upstream docs 是否使用 neutral English and en_US?
  personal notes 是否沒有進 PR diff?
```

## 和前面章節的關係

```text
Ch01 PR Thinking Model
  補上 upstream 明確要求 substantial work 先討論.

Ch02 Commit Style
  補上 upstream 明確 commit message rules.

Ch03 PR Body And Review
  補上 issue-first and discussion-first 的背景.

Ch04 Docs, Tests, And Evidence
  補上 formatter, test before commit, en_US docs.

Ch05 PR Operation Playbook
  補上為什麼要把 notes 留在 learn/*, fix/* 只保留 coherent patch.
```

## 本章學到的三件事

> [Summary]
> 1. `CONTRIBUTING.md` 是 upstream 明文規範, commit history 和 PR discussion 是補充 evidence.
> 2. `libiui` 不要求 Conventional Commits. 它要求 readable imperative subject, coherent commits, and useful body when needed.
> 3. Personal notes 和 upstream docs 的 audience 不同, 所以 notes 應留在 `learn/*`, upstream PR 只放可 review 的 code, tests, or docs.

## Source Notes

```text
../../CONTRIBUTING.md
ch01-pr-thinking-model.md
ch02-commit-style.md
ch03-pr-body-and-review.md
ch04-docs-tests-evidence.md
ch05-pr-operation-playbook.md
```
