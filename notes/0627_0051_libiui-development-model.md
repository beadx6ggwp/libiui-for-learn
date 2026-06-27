# libiui Development Model

## Purpose

這篇整理正式開發前需要理解的 `sysprog21/libiui` 開發概念. 重點不是只會下 Git 指令, 而是知道 upstream 期待什麼形狀的改動, commit, branch, test result.

觀察基準:

```text
upstream/main = eb8d9c83f828ac50412d7d51e7c52b2678c09450
date checked  = 2026-06-27
```

## Related Notes

這篇是正式開發前的入口筆記. 如果對 Git 或 PR 模型還不穩, 建議按這個順序讀:

```text
1. 0626_2320_github-fork-pr-model.md
   理解 fork, origin, upstream, remote-tracking branch.

2. 0627_0013_git-operation-concepts.md
   理解 reset, force push, branch pointer, commit reachability.

3. 0627_0026_pr-branch-concept.md
   理解 PR branch, base/head, ordinary push, force push PR branch.

4. 0627_0051_libiui-development-model.md
   理解 sysprog21/libiui 的 commit 風格, patch scope, test expectation.

5. 0626_1731_repo-state-cleanup-strategy.md
   對照目前這個 fork 的實際狀態和下一步決策.
```

定位可以這樣記:

```text
github-fork-pr-model          = GitHub fork 架構
git-operation-concepts        = Git 操作語意
pr-branch-concept             = GitHub PR 運作語意
libiui-development-model      = upstream 開發文化和實務
repo-state-cleanup-strategy   = 目前 repo 的狀態紀錄
```

## Repository Shape

`libiui` 是 C99 immediate-mode UI library. 主要邊界如下:

```text
include/     public API, especially iui.h and iui-spec.h
src/         core and widget modules
ports/       backend implementations, such as sdl2, headless, wasm
tests/       unit tests, demos, headless tests, benchmark code
mk/          build system fragments
configs/     Kconfig definitions and defconfig
scripts/     generators and test helpers
assets/web/  WebAssembly browser assets
```

正式開發時, 先判斷改動屬於哪個邊界. 不要把 unrelated cleanup, personal notes, build experiment, feature patch 混在同一個 PR.

## Upstream Branch And PR Pattern

從最近 history 看, upstream 常用 "one topic branch -> one PR -> merge to main":

```text
rendering        -> PR #30
widget-complete  -> PR #29
static-analysis  -> PR #28
fix-wasm         -> PR #25
portable-kconfig -> PR #22
fix-build        -> PR #20
```

概念圖:

```text
upstream/main:  O -------------------- M
                 \                    /
topic branch:     A -> B -> cleaned C
```

所以你的 fork 應該保持:

```text
origin/main                    跟 upstream/main 對齊
learn/windows-first-attempt     保存第一次嘗試和討論脈絡
learn/notes                     保存學習筆記
fix/windows-build-docs          真正投稿用 PR branch
```

PR branch 應該從乾淨的 `upstream/main` 切出來:

```sh
git fetch upstream
git switch main
git reset --hard upstream/main
git switch -c fix/windows-build-docs
```

## Commit Subject Style

最近 commit subject 的實際樣子:

```text
Improve rendering: ink-bounds, draw batching, text cache
Add small FAB, range slider, three-line list item
CI: Add Clang Static Analyzer
Fix switch boundary hit testing (#27)
Fix WASM mouse by restoring g_wasm_ctx each frame
Replace $(shell) with portable $(python)
Fix demo compilation when calculator is disabled
Remove legacy framebuffer code
Switch WASM backend to Canvas 2D direct rendering
Consolidate toggle widgets (checkbox/radio)
Break down iui_process_text_input_selection
```

可歸納成:

- 用 imperative verb 開頭: `Add`, `Fix`, `Improve`, `Replace`, `Remove`, `Switch`, `Consolidate`.
- Subject 簡短, 不加句點.
- 若是 CI 或 infrastructure, 可用 scope prefix, 例如 `CI: Add ...`.
- Subject 說明行為或目的, 不寫 vague message, 例如不要只寫 `Update README`.
- 小修可以很精準, 例如 `Fix demo compilation when calculator is disabled`.
- 大改仍要有單一主題, 例如 rendering batching 和 text cache 放在同一個 rendering topic.

## Patch Size And Scope

upstream 能接受不同大小的 patch, 但重點是 scope 要清楚.

小修例子:

```text
Fix switch boundary hit testing (#27)
3 files changed, 8 insertions, 4 deletions
```

build portability 例子:

```text
Replace $(shell) with portable $(python)
1 file changed, configs/Kconfig
```

feature 例子:

```text
Add small FAB, range slider, three-line list item
include/iui.h
include/iui-spec.h
src/basic.c
src/fab.c
src/list.c
src/md3-spec.dsl
```

大型 rendering 例子:

```text
Improve rendering: ink-bounds, draw batching, text cache
20 files changed, includes benchmark code
```

判斷原則:

- bug fix: 儘量小, 附近測試一起改.
- build or portability fix: 不順手重構功能邏輯.
- widget feature: public API, implementation, spec, tests 要一起想.
- generated output: 只有 generator 或 DSL 變更時才動.

## Code Style

`.clang-format` 顯示主要規則:

```text
BasedOnStyle: Chromium
IndentWidth: 4
UseTab: Never
BreakBeforeBraces: Linux
PointerAlignment: Right
```

開發時應該:

- C code 用 4-space indentation.
- 不用 tab.
- public API 保持 `iui_` prefix.
- module-local helper 儘量 `static`.
- 不把 backend-specific logic 洩漏到 core module.
- Kconfig-controlled module 要維持條件編譯邊界.

## Build And Test Expectations

本機至少知道這些命令:

```sh
make defconfig
make
make check
make check-unit
make check-headless
make indent
```

CI 對 code-related changes 會檢查:

```text
formatting
newline at EOF
Clang Static Analyzer
unit tests on Ubuntu and macOS
headless tests
ASan build
minimal/basic/full build matrix
WASM build
```

所以 PR description 應該列出你跑過的 test command. 如果沒有跑完整, 要明確說原因.

## How To Prepare A Real Patch

建議流程:

```text
1. 從 issue 或具體 build failure 定義問題.
2. 找到最小 repo 邊界, 例如 README, Makefile, ports/sdl2.c, tests/example.c.
3. 先做可驗證的小修.
4. 補測試或至少跑相關 build.
5. 用一個清楚 commit subject.
6. push 到 fix/... PR branch.
7. PR description 寫 problem, solution, tests.
```

不要在投稿 branch 放:

```text
notes/
個人學習紀錄
第一次嘗試的大包 commit
不相關 refactor
未說明原因的 formatting churn
```

## For The Windows Support Work

`jserv` 在 `e722b54` 的 commit comment 提到可以考慮送 Windows support PR. 這代表方向可行, 但不代表第一次 commit 的形狀適合直接投稿.

比較好的投稿形狀:

```text
base: upstream/main
branch: fix/windows-build-docs
scope: Windows build instructions or one verified Windows build fix
```

如果只是文件:

```text
README.md
```

如果需要改 build:

```text
Makefile
mk/*.mk
configs/Kconfig
ports/sdl2.c
tests/example.c
```

但不要一次把 "documentation", "build system", "demo behavior", "personal notes" 全部混在同一個 PR.

## Pre-PR Checklist

提交前確認:

- `git status` 只包含這個 PR 應該有的檔案.
- branch 是從 `upstream/main` 切出.
- commit subject 符合 upstream 風格.
- 沒有 `notes/` 或 personal learning files.
- 沒有無關 formatting.
- 有跑過對應測試, 或清楚記錄未跑原因.
- PR description 能回答: problem 是什麼, solution 改了什麼, tests 跑了什麼.

## Current Repo State

目前這個 fork 的乾淨開發基準是:

```text
main         = eb8d9c83f828ac50412d7d51e7c52b2678c09450
origin/main  = eb8d9c83f828ac50412d7d51e7c52b2678c09450
upstream/main = eb8d9c83f828ac50412d7d51e7c52b2678c09450
```

`e722b54` 已由下面 branch 保留:

```text
learn/windows-first-attempt
origin/learn/windows-first-attempt
```

因此正式開發可以從乾淨 `upstream/main` 重新切 PR branch, 不需要把第一次嘗試直接送出去.
