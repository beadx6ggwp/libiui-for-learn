# Ch03: PR Body And Review

## 本章問題

你有乾淨 branch, 也有合理 commit subject. 接著問題是: PR body 要寫什麼? Reviewer 留 comment 時要怎麼回?

naive 想法是:

```text
PR title 已經說明了
diff 也看得到
body 寫簡單一點就好
review comment 照改就好
```

這會讓 reviewer 必須自己重建你的 problem, root cause, validation. 在 `libiui` 這類系統性小型 C project 裡, 這會增加 review 成本.

> [Scope]
> 本章講 PR body, review 回應, draft PR, branch hygiene. 測試證據和文件風格會在 Ch04 細講.

## PR Body 的任務

PR body 要讓 reviewer 快速回答:

```text
這個 PR 解決什麼問題?
為什麼現在的 code 會錯?
改動範圍為什麼這麼小或這麼大?
你怎麼驗證?
你刻意沒有處理什麼?
```

PR body 不是流水帳:

```text
I changed example.c and README.
```

它應該是因果鏈:

```text
Problem -> Root Cause -> Solution -> Validation -> Notes
```

## 建議模板

```markdown
## Problem

Describe the failing behavior, affected config, platform, or scenario.

## Root Cause

Explain the wrong assumption, boundary, invariant, or state transition.

## Solution

Explain the change and why it is the smallest correct fix.

## Validation

- command
- command
- screenshot / benchmark / environment if relevant

## Notes

Limitations, tradeoffs, or follow-ups intentionally left out.
```

對 jserv/sysprog21 類 review, `Root Cause` 和 `Validation` 特別重要. 它們證明你不是用運氣修過.

## Small Bug Fix PR Body

小 bug fix 不需要長篇大論, 但要精準.

例子:

```markdown
## Problem

The demo fails to build when CONFIG_DEMO_CALCULATOR is disabled.

## Root Cause

Shared demo window helpers were guarded by the calculator-only config block, but other demo paths still reference them.

## Solution

Move the shared helpers back to the common demo scope.

## Validation

- make defconfig
- make CONFIG_DEMO_CALCULATOR=n
```

這種 body 的重點是 boundary.

## Build / Portability PR Body

Windows/MSYS2 類問題要寫出 exact environment:

```text
OS
shell
toolchain
compiler version
SDL2 package source
exact command
exact failure
```

不要只寫:

```text
Windows build failed.
```

要寫:

```text
Under MSYS2 UCRT64, tests/example.c fails because the clock demo uses POSIX localtime_r(), while the target toolchain exposes localtime_s().
```

然後說明你的修正是不是:

```text
demo-local compatibility wrapper
build-system portability change
documentation-only clarification
```

這三種 scope 不應混在一起.

## Review 回應方式

好的回應要承認或反駁 reviewer 的技術點, 不要只說 "done".

比較好的模式:

```text
You are right. The issue is not the switch widget itself, but the ownership of the shared boundary in hit testing. I changed the rectangle test to use half-open intervals and updated the tracking test.
```

如果不同意 reviewer:

```text
I do not think this should be handled in the SDL2 backend, because the assumption is already present in the common input path. Moving the fix to the backend would leave the headless tests with the same invariant violation.
```

要用 evidence:

```text
test
benchmark
spec
counterexample
smaller invariant
```

不要用:

```text
It works on my machine.
AI suggested this.
I think it is cleaner.
```

## AI Review 的定位

`libiui` PR 裡有 bot review, 例如 `cubic-dev-ai[bot]`. 這些 comments 可能指出實際問題, 例如 PR #30 的 text truncation 或 ink-bounds 太 tight.

但 bot comment 不是權威. 正確做法是:

```text
1. 判斷 comment 是否有效.
2. 找 root cause.
3. 修正或解釋為何不修.
4. 用 test or reasoning 回覆.
```

不要把 bot output 當成自己的 reasoning.

## Branch Hygiene In PR Discussion

`libiui` PR 討論曾出現幾個重要 maintainer signal:

```text
avoid using default branch main for PR
use git rebase -i to refine commits
force push the PR branch after cleanup
screenshots can improve UI/demo documentation
```

這些不是形式主義. 它們的工程目的分別是:

```text
不用 main 開 PR
  避免 future PR 互相污染

rebase -i
  把 review noise 整理成 readable history

force push PR branch
  更新同一個 PR 的 head branch

screenshots
  降低 UI/demo review 成本
```

## Draft PR 何時有用

如果你還在確認:

```text
root cause 是否正確
CI 是否會過
maintainer 是否接受方向
large refactor 是否值得
```

可以用 draft PR. 但 draft PR 仍應該有基本 problem 和 current status.

不要把 draft PR 當成:

```text
我還沒想清楚, 先丟給 maintainer 幫我想
```

更好的 draft PR 是:

```text
I believe the root cause is X and the current patch validates Y. I am marking this as draft because I still need to verify Z.
```

## 本章學到的三件事

> [Summary]
> 1. PR body 的核心是 Problem, Root Cause, Solution, Validation, Notes.
> 2. Review 回覆要回到 invariant and evidence, 不要只說 done 或 works on my machine.
> 3. Branch hygiene, rebase, force push PR branch 都是為了降低 review 和 long-term history 成本.

## 下一章

下一章處理 docs, tests, screenshots, benchmarks. 也就是 PR body 裡的 validation 和 documentation 要怎麼具體化.

讀: [ch04-docs-tests-evidence.md](ch04-docs-tests-evidence.md)
