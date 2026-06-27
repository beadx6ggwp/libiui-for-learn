# Ch02: Commit Style

## 本章問題

你要送 `libiui` PR 時, commit 不是只給 Git 記錄用. Commit 是 reviewer 和未來 maintainer 讀 history 的單位.

naive 想法是:

```text
commit message 只是描述我做了什麼
先 push, 之後 reviewer 反正看 diff
fix typo / address comment 也沒關係
```

但在 upstream repo 裡, commit history 是長期文件. 它要讓半年後的人能理解:

```text
這個改動解決了什麼問題?
為什麼用這種方式?
它屬於哪個 module or boundary?
```

> [Scope]
> 本章只講 commit subject, commit body, commit scope, history cleanup. PR body 放到 Ch03.

## Upstream 明文規則

`CONTRIBUTING.md` 是 commit style 的 primary source. 它引用 "How to Write a Git Commit Message", 並要求:

```text
subject 和 body 用空白行分開
subject 約 50 字
subject 首字母大寫
subject 不用句點結尾
使用 imperative mood
body wrap around 72 columns
body 解釋 what and why, not how
```

所以這裡不是在套 Conventional Commits. 我們後面觀察 upstream history, 是為了找出這個 repo 實際常用的 subject 形狀, 不是取代 `CONTRIBUTING.md`.

## 先觀察 upstream subject

`sysprog21/libiui` 最近 history 有這些 subject:

```text
Fix WASM mouse by restoring g_wasm_ctx each frame
Fix switch boundary hit testing (#27)
Fix demo compilation when calculator is disabled
Replace $(shell) with portable $(python)
CI: Add Clang Static Analyzer
Add small FAB, range slider, three-line list item
Improve rendering: ink-bounds, draw batching, text cache
Remove legacy framebuffer code
Switch WASM backend to Canvas 2D direct rendering
Consolidate toggle widgets (checkbox/radio)
Break down iui_process_text_input_selection
```

可以歸納:

```text
imperative verb
short subject
no trailing period
scope prefix only when useful
describe behavior or intent
avoid process noise
```

常見開頭:

```text
Fix
Add
Improve
Replace
Remove
Switch
Consolidate
Break down
CI:
```

## 好 subject 的特徵

好的 subject 讓 reviewer 立刻知道改動類型:

```text
Fix demo compilation when calculator is disabled
```

它回答:

```text
type: Fix
object: demo compilation
condition: calculator is disabled
```

另一個例子:

```text
Replace $(shell) with portable $(python)
```

它回答:

```text
type: Replace
old assumption: $(shell)
new mechanism: portable $(python)
```

這比 `Update Kconfig` 好, 因為它直接說出方向和原因.

## 不好的 subject

避免:

```text
Update README
Fix bug
Windows stuff
Try build fix
Address review
Fix typo
notes and code
```

問題是這些 subject 沒有 root cause, scope, or review value.

如果你的 commit subject 只能寫成 `Update files`, 通常代表 PR scope 還沒想清楚.

## Commit scope 要單一

一個 commit 應該是一個可理解的工程動作.

好例子:

```text
Fix MSYS2 UCRT64 demo clock build
```

可能包含:

```text
tests/example.c
```

不好例子:

```text
Fix Windows build and add learning notes
```

它混了:

```text
upstream-facing build fix
personal learning notes
documentation that may not belong upstream
```

這會讓 reviewer 無法只 review code fix.

## 一個 PR 幾個 commit?

小 bug fix:

```text
one PR -> one commit
```

例子:

```text
Fix switch boundary hit testing (#27)
```

feature or larger change:

```text
one PR -> a few semantic commits
```

但每個 commit 要能獨立說明:

```text
Add API surface
Implement internal state
Add tests or demo
Update docs
```

不要保留:

```text
fix typo
try again
address comment
oops
```

這些是 review 過程的暫存歷史, 不是 upstream history.

## 什麼時候需要 commit body

subject 能說清楚的小修, commit body 可以省略.

需要 body 的情況:

```text
root cause 不容易從 subject 看出
behavior tradeoff 需要解釋
跨 backend or config boundary
修的是 portability assumption
有 test or benchmark detail 值得留在 history
```

commit body 可以用:

```text
Problem:
Root cause:
Solution:
Validation:
```

但不要把 PR body 完整複製進每個 commit. Commit body 是給未來讀 history 的人, PR body 是給當下 reviewer 的完整 context.

## Rebase 是整理歷史, 不是炫技

PR review 過程常會產生:

```text
Add Windows build fix
Fix typo
Address review
Try localtime_s
Fix format
```

final review 前, maintainer 可能希望你用:

```sh
git rebase -i upstream/main
```

把它整理成:

```text
Fix MSYS2 UCRT64 demo clock build
```

然後:

```sh
git push --force-with-lease origin fix/msys2-ucrt-clock
```

這不是隱藏錯誤. 這是把臨時開發歷史整理成可維護 history.

## Windows PR 的 commit subject

如果你要處理 `e722b54` 裡的 `tests/example.c`, subject 不應該寫:

```text
Add Windows build instructions and documentation
```

因為真正要送的可能不是 instructions and documentation, 而是 demo build fix.

更合理:

```text
Fix MSYS2 UCRT64 demo clock build
```

如果後來是文件 PR:

```text
Document Windows build requirements
```

這兩件事應該分 PR, 不要用同一個 commit 混在一起.

## 本章學到的三件事

> [Summary]
> 1. Commit subject 是長期 history 的入口, 要用 imperative, short, scoped 的形式描述工程動作.
> 2. 一個 commit 不應混合 upstream fix, personal notes, unrelated docs, formatting churn.
> 3. `git rebase -i` 是為了把 review 過程中的 noise 整理成 readable upstream history.

## 下一章

下一章看 PR body 和 review 回應. Commit 說的是 history, PR body 說的是 reviewer 當下需要的 context.

讀: [ch03-pr-body-and-review.md](ch03-pr-body-and-review.md)

## Source Notes

```text
../../CONTRIBUTING.md
```
