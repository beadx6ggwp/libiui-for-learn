# Ch01: PR Thinking Model

## 本章問題

你現在不是只想知道 "怎麼開 PR". 你想知道的是: 在 `sysprog21/libiui` 這種 repo 裡, maintainer 到底怎麼判斷一個 PR 是不是值得 review, 值得合併?

naive 想法是:

```text
程式可以編過
diff 看起來不大
commit push 上去
PR body 簡單寫 update
```

這種做法可能偶爾過, 但不穩. 因為 reviewer 看的不是 "你有沒有改動", 而是你有沒有把問題拆清楚.

> [Scope]
> 本章建立 `libiui` PR 的基本思考模型. Commit message, PR body, docs, tests 會在後面三章細講.

## Maintainer 真正在 review 什麼

一個好的 PR 要能回答這條因果鏈:

```text
現象 -> root cause -> 設計選擇 -> 最小修正 -> 驗證方式 -> 後續可維護性
```

對 `libiui` 來說, 這條鏈通常會落在幾種邊界上:

```text
Kconfig boundary
  某個 config 關掉後, shared symbol 是否還能使用?

backend boundary
  SDL2, headless, wasm 是否各自維持正確假設?

widget state boundary
  immediate-mode state 是否在 frame 之間正確保存?

geometry/input boundary
  hit testing, clip rect, layout ownership 是否有明確 invariant?

documentation boundary
  文件是在解釋 upstream project, 還是在記錄個人環境?
```

如果 PR 只說 "修好了", 但沒有說是哪個 boundary 出錯, reviewer 仍然很難信任.

## Example 1: Build Fix 不是補一個 ifdef

`Fix demo compilation when calculator is disabled` 是小 patch:

```text
tests/example.c | 19 lines changed
```

它的價值不在 diff 大小, 而在問題邊界清楚:

```text
CONFIG_DEMO_CALCULATOR=n
  calculator demo 關掉

shared demo symbols
  其他 demo 仍然需要

root cause
  shared symbols 被放進 calculator-only boundary
```

所以好的 PR 說法不是:

```text
Fix build.
```

而是:

```text
When CONFIG_DEMO_CALCULATOR is disabled, other demo code still references shared window helpers. Those helpers were inside the calculator-only block, so the build fails. Move the shared helpers back to shared scope.
```

這種說法讓 maintainer 知道你在修 boundary, 不是隨機移 code.

## Example 2: Portability Fix 是移除錯誤假設

`Replace $(shell) with portable $(python)` 只改:

```text
configs/Kconfig
```

但它背後的問題是 portability:

```text
原本假設有 POSIX shell
Kconfig detection 使用 shell syntax
Windows or non-POSIX environment 會出問題
```

好的 portability PR 不應該只是:

```text
If Windows, do another branch.
```

更好的方向是:

```text
移除不必要的 POSIX shell assumption
改用 Kconfiglib already supports 的 Python path
```

這對你的 Windows/MSYS2 工作很重要. 不要先急著加平台分支. 先問:

```text
這是 Windows 特例?
還是原本 build system 假設太窄?
```

## Example 3: GUI Bug 要找 invariant

`Fix switch boundary hit testing (#27)` 是小 diff:

```text
src/input.c
src/internal.h
tests/test-tracking.c
```

這類 bug 的 naive 修法可能是調一個 magic number, 或用 numeric trick 讓邊界避開.

但 maintainer 更在意 invariant:

```text
相鄰 rectangles 的 shared boundary 到底屬於誰?
hit testing 用 closed interval 還是 half-open interval?
switch hit rect 應該 clamp 到哪個 layout height?
```

如果 PR 能說清楚 invariant, reviewer 才知道修正不只對一個點有效.

## Example 4: 大型 PR 要有證據和取捨

`Improve rendering: ink-bounds, draw batching, text cache` 是大型 PR:

```text
20 files changed
1306 insertions
421 deletions
benchmark code included
```

大型 PR 不能只說:

```text
Improve rendering performance.
```

它要說:

```text
hot path 是什麼
新的 interception point 是什麼
dirty region 如何計算
batching 什麼時候啟用
text cache 會改變什麼
benchmark 怎麼跑
tradeoff 是什麼
```

越大的 PR, 越需要讓 reviewer 知道你理解改動的成本.

## PR 類型和基本形狀

### Bug Fix

```text
symptom -> root cause -> smallest fix -> regression test
```

適合 first contribution. 風險低, review 範圍清楚.

### Build / Portability

```text
environment assumption -> failure -> portable replacement -> build validation
```

你的 Windows support 最接近這類.

### Feature

```text
use case -> API shape -> state model -> tests/demo/docs
```

如果是 widget, 還要接 MD3 spec, dimensions, accessibility, interaction state.

### Refactor

```text
bad boundary or duplication -> new invariant -> behavior preserved -> tests
```

不要只說 cleanup.

### Performance

```text
hot path -> current cost -> new mechanism -> benchmark -> tradeoff
```

沒有 benchmark 的 performance PR 很難 review.

## 本章學到的三件事

> [Summary]
> 1. `libiui` PR 要回答 problem -> root cause -> minimal fix -> validation, 不是只展示 diff.
> 2. 不同 PR 類型有不同證據需求: bug 要 regression, portability 要環境假設, performance 要 benchmark.
> 3. reviewer 會把 local symptom 拉回 shared invariant, 所以你要先找 boundary 和 root cause.

## 下一章

下一章看 commit: subject 怎麼寫, 一個 PR 應該幾個 commit, 什麼時候 rebase, 什麼 commit history 會讓 reviewer 比較容易信任.

讀: [ch02-commit-style.md](ch02-commit-style.md)
