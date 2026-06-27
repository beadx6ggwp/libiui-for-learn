# Ch04: Documentation, Tests, And Evidence

## 本章問題

很多 PR 卡住不是因為 code 完全錯, 而是因為 evidence 不夠. Reviewer 不知道你的 patch 在哪個環境有效, 也不知道它是否會破壞其他 config, backend, or UI state.

naive 想法是:

```text
我本機可以編過
我在 PR body 說 tested
文件多寫一點總是好的
```

對 `libiui` 來說, 這還不夠. Evidence 要對應 PR 類型, documentation 也要分清楚 upstream docs 和 personal notes.

> [Scope]
> 本章整理 `libiui` PR 裡 tests, screenshots, benchmarks, docs 的放法. 它也說明為什麼你的 learning notes 不該直接放進 upstream PR.

## CI 告訴你 maintainer 在意什麼

`.github/workflows/main.yml` 對 code-related changes 會跑:

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

這表示 `libiui` 不只在意 "default config 能不能編". 它在意:

```text
multiple OS
multiple configs
headless validation
WASM build
static analysis
formatting
```

所以你在 PR body 裡列 validation 時, 要盡量對應改動範圍.

## Validation 要跟 PR 類型對應

### Small Bug Fix

需要:

```text
reproduction
regression test if practical
make check or targeted test
```

例子:

```text
Validation:
- make check
- added test for half-open hit rect boundary
```

### Build / Portability Fix

需要:

```text
exact environment
exact failing command
exact compiler or linker error
fixed command result
```

Windows/MSYS2 類 PR 應包含:

```text
MSYS2 environment: UCRT64
compiler: gcc version
packages: mingw-w64-ucrt-x86_64-SDL2, make, python
command: make defconfig && make
failure before: exact error
result after: build or test command
```

不要只寫:

```text
Tested on Windows.
```

### UI / Demo PR

需要:

```text
screenshot
before/after if visual behavior changed
which demo or component
which config
```

Maintainer 曾提到可以把 screenshots 放在 `assets` and reference them in `README.md`. 這不是裝飾. 對 UI project 來說, screenshot 是 review evidence.

### Performance PR

需要:

```text
benchmark command
hardware or runtime environment
before/after numbers
what workload represents
tradeoff
```

PR #30 的形狀值得學: rendering optimization 不只描述 feature, 也附 benchmark and coverage. 大型 performance PR 沒有 numbers, reviewer 很難知道價值是否大於風險.

## Documentation: upstream docs vs personal notes

你的 `notes/` 很重要, 但它是 learning asset, 不是 upstream docs.

要分清楚:

```text
personal learning notes
  notes/
  AGENTS.md
  project plans
  thought process
  not for upstream PR

upstream documentation
  README.md
  docs intended for all users
  neutral tone
  reproducible commands
  no personal learning context
```

`e722b54` 的 README 問題在這裡. 它有 Windows/MSYS2 資訊, 但也包含個人學習脈絡和 Pixel-Renderer 想法. 這不適合直接放進 upstream README.

如果要送 Windows docs, 要整理成 upstream user-facing form:

```text
Supported environment
Required packages
Build commands
Known limitations
Validation status
```

不要寫成:

```text
我為了學習而做了什麼
我未來想接 Pixel-Renderer
我的本機 setup 隨筆
```

## README 修改要小心

README 是 project front page. 修改它的門檻比改個 notes 文件高.

適合 README 的內容:

```text
official quick start
supported platform
minimal build command
demo screenshot
link to deeper docs
```

不適合 README 的內容:

```text
personal debug diary
long environment essay
uncertain workaround
future personal project plan
```

如果 Windows support 還只是 partial, 不要寫成 full support.

可以寫:

```text
Windows builds are known to work under MSYS2 UCRT64 with SDL2 after installing ...
```

但前提是你真的驗證過.

## Patch Scope 和 Evidence 要一致

如果 PR title 是:

```text
Fix MSYS2 UCRT64 demo build
```

那 evidence 應該圍繞:

```text
MSYS2 UCRT64 build failure
why it fails
the exact fix
build validation
```

不要同時塞:

```text
README teaching section
notes about fork workflow
future Pixel-Renderer integration
unrelated formatting
```

如果 title 是:

```text
Document Windows build requirements
```

那 evidence 應該圍繞:

```text
which environment was tested
why users need these steps
commands are minimal and reproducible
limitations are clear
```

## 開 PR 前的 Evidence Checklist

```text
Problem:
  Can I reproduce it?

Root cause:
  Can I point to the wrong assumption or boundary?

Scope:
  Are all changed files necessary?

Validation:
  Did I run commands that match the change?

Docs:
  Is this upstream-facing, not personal notes?

Artifacts:
  Does UI/demo/performance need screenshot or benchmark?

Limitations:
  Did I say what this PR does not solve?
```

## 本章學到的三件事

> [Summary]
> 1. Validation must match PR type: bug needs reproduction/test, portability needs exact environment, UI needs screenshots, performance needs benchmark.
> 2. `notes/` and personal learning records are not upstream docs. README changes must be neutral, reproducible, and project-facing.
> 3. Evidence is part of the patch. Without it, reviewer must guess whether the fix is correct and maintainable.

## 下一章

前面四章談的是 PR 內容和 evidence. 下一章把這些要求接到實際 Git 操作, 也就是如何從 `upstream/main` 開乾淨 branch, 抽出最小 patch, 檢查 personal notes 沒有進 diff, 並在 review 後更新同一個 PR.

讀: [ch05-pr-operation-playbook.md](ch05-pr-operation-playbook.md)

## 後續主題

如果要繼續整理, 下一個主題可以是:

```text
notes/windows-build/
  專門整理 MSYS2 UCRT64 build, localtime_r/localtime_s, SDL2 package, exact validation.

notes/libiui-windows-pr/
  專門為第一個 Windows support PR 寫 problem, root cause, patch plan, validation.
```

## Source Notes

```text
../0626_1726_jserv-pr-guidelines.md
../0627_0051_libiui-development-model.md
../0626_1723_upstream-pr-submission-plan.md
.github/workflows/main.yml
```
