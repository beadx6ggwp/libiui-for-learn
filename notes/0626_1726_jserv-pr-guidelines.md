# jserv/sysprog21 PR 規範與思路

## Question

我想理解的不是單次 Windows build fix 怎麼送，而是：面對 jserv/sysprog21 類型的 open-source project，應該怎麼思考、整理、提交 PR，才不是「把能動的 patch 丟上去」，而是可被 review、可被信任、可被合併的工程貢獻。

## Context

這篇不是官方規範文件，而是從以下材料歸納出來的工作模型：

- `sysprog21/libiui` closed PR #1-#30。
- 代表性 PR：#20 build fix、#22 portability fix、#25 WASM input fix、#27 external contributor bug fix、#28 static analysis、#30 rendering optimization。
- linked issues #19、#26。
- maintainer comments：不要用 fork 的 `main` branch 開 PR；用 `git rebase -i` 整理 commits 後 force push；UI/demo 類改動可補 screenshots。
- jserv `collab2026`：open-source contribution 重點是工程品質、真實問題、限制與取捨，不是單純累積 GitHub 足跡。

## Core Mental Model

一個好的 PR 不是「我改了什麼」，而是回答一條完整的因果鏈：

```text
現象 -> root cause -> 設計選擇 -> 最小修正 -> 驗證方式 -> review 後可維護性
```

naive 方法是：看到編譯錯就補一個 `#ifdef`，看到 UI 錯就調一個 magic number，看到 performance 慢就塞 cache。這種 patch 可能讓當下案例通過，但 reviewer 會問：

- 這是 root cause 嗎，還是在遮症狀？
- 會不會破壞其他 backend、config、platform？
- 有沒有測試或可重現步驟？
- 這個 abstraction 是必要的，還是過度設計？
- 以後維護者看到這段 code，能不能知道為什麼存在？

在 jserv/sysprog21 這類專案裡，PR 的價值通常不只在 code diff，而在你有沒有把問題拆到工程結構本身。

## What Maintainers Seem To Value

### 1. Problem-driven, not patch-driven

PR #20 不是單純移動幾個 symbol。它的問題是：`CONFIG_DEMO_CALCULATOR=n` 時，其他 demo 仍需要 `DEMO_WIN_X`、`DEMO_WIN_Y`、`get_demo_window_height`，但這些 shared symbols 被放進 calculator-only block。root cause 是 compile-time configuration boundary 錯了，所以修正是把 shared declarations 移回 shared scope。

可學到的規則：

- 先講 build/runtime failure。
- 再指出是哪個 module boundary、config boundary、state boundary 錯。
- 最後才講 patch。

### 2. Portability changes must remove assumptions

PR #22 把 Kconfig detection 裡的 POSIX shell dependency 改成 Kconfiglib `$(python,...)`。重點不是「Windows 編不過，所以加特例」，而是：build system 原本假設有 POSIX shell、`2>/dev/null`、`||`、`&&`，這個假設和 portability 目標衝突。

可學到的規則：

- portability PR 要寫清楚「原本假設什麼環境」。
- 好修正通常是移除錯誤假設，而不是堆平台分支。
- 如果一定要加平台分支，要把範圍縮到最小。

### 3. Reviewer 會追 root cause

PR #27 的初始方向是處理 switch touch rect 邊界，但 maintainer 指出更深層問題：`in_rect()` 用 closed interval，讓相鄰 rectangles 共用邊界點。正解不是用 `nextafterf()` 把邊界推開，而是改成 half-open interval `[x, x + width)` / `[y, y + height)`，再讓 switch hit rect clamp 到 `row_height`。

可學到的規則：

- 不要用 fragile numeric trick 掩蓋資料結構或幾何語義問題。
- GUI/input 類 bug 常常要回到 coordinate ownership：一個 pixel/point 到底屬於誰？
- reviewer 喜歡看到你把 local symptom 拉回 shared invariant。

### 4. Evidence matters

PR #30 是大型 rendering optimization，所以 body 裡有 feature list、benchmark、coverage、migration note。這種 PR 如果只說「比較快」是不夠的，因為它碰到 batching、dirty rect、text cache、backend behavior。

可學到的規則：

- 小 bug fix 要有 reproduce + test。
- performance PR 要有 benchmark method。
- rendering/UI PR 要有 screenshots 或 visual artifacts。
- API/refactor PR 要有 migration note 或 call-site impact。

### 5. AI review 可以輔助，但不能取代你的判斷

部分 PR 有 `cubic-dev-ai[bot]` review。這些 comments 有些指出實際 bug，例如 PR #30 的 text truncation、ink-bounds 太緊。但 maintainer review 仍然會補更深層的設計判斷，例如 PR #27 的 half-open interval。

可學到的規則：

- AI/bot comment 是 signal，不是 authority。
- 你回 PR 時要自己判斷它是否正確，並說明 root cause。
- 不要把 bot summary 當成自己的 reasoning。

## PR Categories and Expected Shape

### Bug Fix PR

適合 first contribution。形狀通常是：

```text
1 issue / 1 root cause / small diff / regression test or exact reproduction
```

PR body 應包含：

- What breaks?
- How to reproduce?
- Why does it break?
- Why is this fix the smallest correct fix?
- How was it validated?

### Build / Portability PR

適合你的 Windows/MSYS2 類問題。形狀通常是：

```text
environment assumption -> failure -> portable replacement -> build validation
```

要避免：

- 把個人環境教學塞進首頁。
- 宣稱完整 Windows support，但只驗證一個 demo。
- 只貼「我這裡可以」而沒有 exact compiler error。

### Feature PR

需要更完整設計。形狀通常是：

```text
use case -> API shape -> internal state model -> tests/demo -> docs/screenshots
```

如果是 UI widget，要說它對應哪個 MD3 spec、尺寸/狀態/accessibility 怎麼驗證。

### Refactor PR

最容易被質疑，因為 refactor 本身不一定創造 value。形狀應該是：

```text
現有重複/錯誤 invariant -> 新 boundary -> call-site impact -> behavior preserved
```

不要只說「clean up」或「make it better」。要說清楚它降低了哪個維護成本，或修掉哪個 shared bug source。

### Performance PR

形狀通常是：

```text
hot path -> current complexity/cost -> new algorithm/cache/batching -> benchmark -> tradeoff
```

PR #30 有一個值得學的點：它不只報快，也承認 noop backend 下 ink-bounds tracking overhead 會顯著，並解釋為何 real backend 的 dirty-rect saving 可以抵消。這就是取捨。

## Branch and Commit Hygiene

### Do not open PR from fork `main`

maintainer 在 PR #27 明確提醒過：不要用 default branch (`main`) 開 PR。原因很實際：

- fork `main` 通常混有個人筆記、README 修改、實驗 commit。
- review 時難以確認 diff 是否只屬於這個 PR。
- 後續要開第二個 PR 時會互相污染。

建議流程：

```bash
git fetch upstream
git switch -c fix-something upstream/main
```

### Keep one PR about one idea

一個 PR 最好只有一條主線。例如：

- `Fix demo compilation when calculator is disabled`
- `Replace $(shell) with portable $(python)`
- `Fix WASM mouse by restoring g_wasm_ctx each frame`

不要把 code fix、README 教學、個人 note、unrelated formatting 放一起。

### Commit message

觀察 `libiui` history，常見格式是 concise imperative subject：

```text
Fix WASM mouse by restoring g_wasm_ctx each frame
Replace $(shell) with portable $(python)
CI: Add Clang Static Analyzer
Improve rendering: ink-bounds, draw batching, text cache
```

如果 commit body 有價值，應該寫 problem/root cause/test，不要寫流水帳。

目前 `libiui` history 沒看出強制 DCO `Signed-off-by`。若 project 沒要求，不必主動加；若 maintainer 要求，再補。

### Rebase before final review

maintainer 在 PR #27 建議用：

```bash
git rebase -i
```

目的不是炫技，而是把 review 過程中的修補 commit 整理成 readable history。常見目標：

- 1 個小 bug fix -> 1 commit。
- 大 feature -> 幾個有語意的 commits，每個可獨立閱讀。
- 不留下 `fix typo`、`try again`、`address comment` 這類 noise。

## PR Body Template

通用模板：

```markdown
## Problem

What breaks, where, and under what condition?

## Root Cause

Which assumption, state transition, boundary, or algorithm is wrong?

## Solution

What changed? Why is this the smallest correct fix?

## Validation

- command 1
- command 2
- screenshot / benchmark / exact environment if relevant

## Notes

Any limitation, tradeoff, or follow-up intentionally left out of this PR.
```

對 jserv/sysprog21 類 review，`Root Cause` 和 `Validation` 最重要。它們讓 reviewer 知道你不是用運氣修過。

## Before Opening a PR

先問自己：

- 我能用一句話說出這個 PR 的 problem 嗎？
- 我能指出 root cause 在哪個 file/function/config boundary 嗎？
- 我能重現 before failure 嗎？
- 我有 after validation 嗎？
- diff 裡有沒有 unrelated files？
- 有沒有個人筆記、IDE 設定、learning README 混進去？
- commit history 是否可讀？
- 如果 reviewer 問「為什麼不是另一種做法」，我答得出來嗎？

如果其中一題答不出來，通常代表還不該開 PR，或至少要開 draft PR。

## How To Respond To Review

不要把 review 當成「對方挑毛病」。在這種專案裡，review 是把 patch 拉回工程不變量的過程。

好的回覆方式：

```text
You are right. The actual ambiguity is that `in_rect()` used closed intervals,
so adjacent rows both owned the shared boundary. I changed it to half-open
intervals and simplified the switch clamp to `row_height`.
```

不好的回覆方式：

```text
It works on my machine.
I changed it because AI suggested it.
I think this is cleaner.
```

如果不同意 reviewer，要用 evidence 回答：

- 反例。
- benchmark。
- spec。
- failing/passing test。
- simpler invariant。

## Applying This To Your Current libiui Fork

目前你的 `e722b54` 是 learning fork commit，不是 upstream-ready PR。原因：

- `README.md` 插入 119 行個人 Windows 教學，會改變 upstream README 首頁定位。
- `tests/example.c` 的修正方向可能合理，但要整理成英文註解、最小 diff、無 trailing whitespace。
- `notes/` 和 `AGENTS.md` 是你的學習/協作系統，不應該進 upstream PR。

正確路線：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
```

然後只重做 `tests/example.c` 的最小 patch。PR body 不說「我整理 Windows 教學」，只說：

```text
MSYS2 UCRT64 build fails because the clock demo assumes POSIX `localtime_r()`.
This patch adds a demo-local `_WIN32` compatibility wrapper backed by
`localtime_s()` and converts `tv_sec` to `time_t` before calling it.
```

README/Windows 教學如果要送，等 code fix 合併後再開第二個 PR，且應改成 neutral upstream docs，而不是個人學習筆記。

## Follow-ups

- [ ] 把這篇當成 PR checklist，每次開 PR 前先填 `Problem / Root Cause / Solution / Validation`。
- [ ] 針對目前 Windows issue，先保存 exact compiler error。
- [ ] 建立乾淨 feature branch，不從 fork `main` 開 PR。
- [ ] 將 learning notes 保留在 fork-only branch 或本地，不混入 upstream PR。
- [ ] 若想做更高價值貢獻，優先找「可重現 bug + 小範圍 root cause + 可補 test」的題目。

