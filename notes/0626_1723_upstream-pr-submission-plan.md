# Upstream PR Submission Plan

## Question

現在 fork 內有一個 Windows/MSYS2 UCRT64 build fix 和一大段個人化 README 教學。問題不是「能不能發 PR」，而是：

- 哪些內容適合送回 `sysprog21/libiui`？
- PR 要怎麼整理成 upstream maintainer 願意 review 的形狀？
- `notes/`、`AGENTS.md`、個人學習紀錄應該怎麼避免污染 PR？

## Context

目前本地 `HEAD` 相對 `upstream/main` 的差異只有：

```text
README.md       | 119 insertions
tests/example.c | 13 insertions, 1 deletion
```

`tests/example.c` 的實際修正是：

- Windows 下補 `localtime_r()` wrapper，內部呼叫 `localtime_s()`。
- 把 `localtime_r(&tv.tv_sec, &now_tm)` 改成先轉成 `time_t temp_sec`，再傳入 `localtime_r()`。

`README.md` 的差異則是把個人 MSYS2 UCRT64 安裝、修正流程、Pixel-Renderer 連結方向插到 upstream README 最前面。這對 learning fork 有用，但對 upstream PR 風險很高，因為它改變了專案首頁的定位，而且混入個人 HackMD 與個人學習脈絡。

## Sources Read

- GitHub API: `sysprog21/libiui` closed PR metadata, PR #1-#30。
- Selected PR discussion/review: #17, #20, #22, #25, #27, #28, #30；其中部分 inline review 來自 `cubic-dev-ai[bot]`，maintainer review 另行標示。
- Linked issues: #19, #26。
- Local git history: merge commits and `e722b54 Add Windows build instructions and documentation`。
- jserv `collab2026` HackMD: 重點是 open-source contribution 應該展示工程品質、真實問題、限制與取捨，而不是堆數量。

## Reasoning

### 1. Upstream PR 的接受形狀

已合併 PR 有幾種模式：

- **小型 bug fix**：PR #20 修 `CONFIG_DEMO_CALCULATOR=n` 時 example build fail。它先有 issue #19 描述可重現失敗，再用 1 個檔案、約 19 行 diff 修掉 shared symbols 被放進 calculator-only block 的 root cause。
- **portability/build fix**：PR #22 把 Kconfig toolchain detection 的 POSIX shell dependency 改成 Kconfiglib `$(python,...)`，body 清楚列出原問題、平台限制、替代方案。
- **外部 contributor PR**：PR #27 修 switch boundary hit testing。maintainer 明確提醒不要用 fork 的 `main` branch 開 PR，要用 feature branch；也要求用 `git rebase -i` 整理 commits 後 force push。review 不只看現象，還追到 root cause：closed interval hit-test 造成相鄰 rect 邊界重疊，所以正解是 half-open interval，而不是用 `nextafterf()` 補丁。
- **大型 rendering PR**：PR #30 有 feature list、benchmark、migration note，且 `cubic-dev-ai[bot]` 的 review comments 被處理後才合併。這種 PR 需要明確 evidence。

結論：你的 Windows fix 應該先做成 **小型 portability/build fix**，不要一開始包成「Windows 完整支援」或「文件重寫」。

### 2. 目前 commit 不適合直接 PR

`e722b54` 把 upstream-facing code fix 和 personal learning docs 混在一起。直接開 PR 會有三個問題：

- Review 範圍太雜：maintainer 要同時看 Windows compile fix、README 首頁重排、個人教學文字、Pixel-Renderer notes。
- README 插入點太重：119 行加在首頁最前面，會壓過原本專案定位。
- `tests/example.c` 裡有繁中 `//` 註解和 trailing space，風格上不像 upstream C code。

這不是「fork 名稱叫 `libiui-for-learn`」的問題。fork 名稱本身不影響 PR；真正影響的是 PR branch 的 diff 是否乾淨。

### 3. Windows fix 的 root cause 要先講精準

目前可說的 root cause 是：

- `tests/example.c` 使用 POSIX `localtime_r()`。
- MSYS2 UCRT64/Windows C runtime 提供的是 `localtime_s()`，而且參數順序和 POSIX `localtime_r()` 相反。
- `gettimeofday()` 的 `tv_sec` 型別在目標環境可能不是 `time_t`，直接把 `&tv.tv_sec` 傳給 `localtime_r()` 會觸發 pointer type mismatch 或 build failure。

但正式 PR 前還要補一份「可重現證據」：

```text
Environment:
- Windows version
- MSYS2 UCRT64
- gcc --version
- make --version
- SDL2 package version if relevant

Reproduce:
make distclean
make defconfig
make

Observed:
貼 exact compiler error
```

沒有 exact error log，就很容易變成「我這裡編不過，所以改一下」。

## Decision

第一個 upstream PR 建議只做：

```text
Title: Fix MSYS2 UCRT64 clock demo build
```

Scope:

- 只改 `tests/example.c`，或加一個非常小的 demo-local compatibility helper。
- 不改 root `README.md`。
- 不加 `notes/`、`AGENTS.md`。
- 不提 Pixel-Renderer。
- 不宣稱完整 Windows support，只說修正 MSYS2 UCRT64 下 clock demo/example build 的 compile issue。

建議 patch 方向：

- 把 compatibility wrapper 寫成英文註解，且保持 C99 style。
- 如果只在 `tests/example.c` 使用，就先放 demo-local `static inline` helper。
- 如果發現其他檔案也需要同樣處理，再考慮抽到 shared portability header；不要為單點修正過度抽象。

README 若真的要提交，應該是第二個 PR，而且要改成短而中性的 upstream docs，例如：

```text
docs/windows-msys2.md
```

或 README 中極短的 platform note。不要把個人 HackMD、個人化 rename `make.exe` 建議、Pixel-Renderer integration 放進 upstream PR。

## Branch Hygiene

不要從目前 fork `main` 直接開 PR。建立乾淨 branch：

```bash
git fetch upstream
git switch -c fix-msys2-ucrt-clock upstream/main
```

然後只套用最小 code patch。完成後確認：

```bash
git diff upstream/main..HEAD --stat
git diff upstream/main..HEAD -- README.md notes AGENTS.md
```

第二個指令應該沒有輸出，代表個人文件沒有進 PR。

提交前整理 commit：

```bash
git status
git add tests/example.c
git commit -m "Fix MSYS2 UCRT64 clock demo build"
git log --oneline upstream/main..HEAD
```

如果中途產生多個修正 commit，開 PR 前用：

```bash
git rebase -i upstream/main
```

整理成 1 個清楚 commit，再 push feature branch：

```bash
git push origin fix-msys2-ucrt-clock
```

## Validation Checklist

至少要有：

```bash
make distclean
make defconfig
make
```

能跑的話再補：

```bash
make check
./libiui_example.exe
```

如果 `make check` 在 Windows 因 headless/Python/SDL2 環境失敗，要誠實寫出「未通過原因」或「未執行原因」。不要假裝全測過。

更好的第二階段 PR 是加 Windows CI，例如 `windows-latest` + MSYS2 UCRT64 build-only job。但這應該獨立於 first fix，否則 review surface 會變大。

## PR Body Draft

````markdown
## Problem

`tests/example.c` fails to build on MSYS2 UCRT64 because the clock demo uses POSIX `localtime_r()`, while the Windows/UCRT runtime provides `localtime_s()` with a different argument order. The `tv_sec` field from `struct timeval` also needs to be converted to `time_t` before passing its address to the time conversion helper.

## Root Cause

The demo assumes POSIX time conversion semantics. MSYS2 UCRT64 exposes a Windows-compatible runtime API, so the POSIX call is unavailable or type-incompatible in this build environment.

## Solution

Add a small `_WIN32` compatibility wrapper for `localtime_r()` in `tests/example.c`, backed by `localtime_s()`, and convert `tv.tv_sec` to a `time_t` temporary before calling it.

## Validation

- `make distclean`
- `make defconfig`
- `make`
- `./libiui_example.exe` on MSYS2 UCRT64
````

如果有 exact compiler error，可以在 `Problem` 後面加：

````markdown
Before this patch, MSYS2 UCRT64 reported:

```text
<paste exact compiler error>
```
````

## Notes / AGENTS Strategy

`notes/` 和 `AGENTS.md` 是 learning fork 的協作基礎，不應該進 upstream PR。可選策略：

- **最簡單**：讓它們留在目前 learning branch，不從這個 branch 開 upstream PR。
- **乾淨 PR 策略**：所有 upstream PR 都從 `upstream/main` 新開 feature branch，只 cherry-pick 或手動套用目標 patch。
- **個人筆記不進 git**：若不想被 status 干擾，可把 `notes/` 放進 `.git/info/exclude`。這只影響本機，不會改 repo。
- **保留學習歷程**：如果希望 notes 也被版本控制，放在 fork-only branch，例如 `learn/notes`，不要 merge 到 PR branch。

## Follow-ups

- [ ] 在 MSYS2 UCRT64 重新跑一次 `make distclean && make defconfig && make`，保存 exact error log。
- [ ] 從 `upstream/main` 開乾淨 branch，不使用 fork `main`。
- [ ] 重寫 `tests/example.c` patch：英文註解、無 trailing whitespace、最小 diff。
- [ ] 確認 `README.md`、`notes/`、`AGENTS.md` 不在 PR diff。
- [ ] 跑 validation checklist。
- [ ] 用上面的 PR body draft 開 draft PR。
- [ ] 如果 maintainer 接受方向，再考慮第二 PR：Windows/MSYS2 docs 或 Windows CI。
