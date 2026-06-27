# Windows Build Strategy Thinking

## Question

現在卡住的問題不是單純 "Windows 要怎麼 build", 而是:

```text
要不要沿用 learn/windows-first-attempt 的舊結果?
還是先假裝沒有舊結果, 從 upstream/main 重新當正常使用者 build 一次?
MSYS2 UCRT64 是必要環境, 還是普通 Windows shell 也應該能用?
```

這比較偏思考筆記, 先不要放進正式 `notes/libiui-review/` 教學章節. 等真正驗證完環境與 failure 後, 再整理成 `windows-build` or `libiui-review` 的教學文章.

## Context

目前已知 branch:

```text
learn/repo-review
  正在保存 repo review, Git 教學, build/runtime/API 理解筆記.

learn/windows-first-attempt
  舊的第一次 Windows 嘗試.
  commit: e722b54 Add Windows build instructions and documentation
```

`learn/windows-first-attempt` 只改了:

```text
README.md
tests/example.c
```

其中 `tests/example.c` 的重點是:

```text
_WIN32 下補 localtime_r() wrapper.
把 tv.tv_sec 轉成 time_t temp_sec.
```

README 則新增大量 MSYS2 UCRT64 build 教學. 但那次是比較隨意地請 Claude 改到能跑, 不是從 baseline failure, root cause, minimal patch, validation evidence 的流程做出來.

所以它應該被視為:

```text
evidence / hypothesis / historical attempt
```

而不是:

```text
clean baseline
official build guide
upstream-ready patch
```

## Why Not Directly Continue The Old Branch

naive 做法:

```text
既然 windows-first-attempt 似乎能跑, 就沿用它繼續修.
```

問題是這會把幾件事混在一起:

```text
1. 正常 upstream/main 在 Windows 上到底會不會 build?
2. 如果不能 build, 第一個 failure 是 shell, SDL2, Makefile, or C portability?
3. localtime_r 是唯一問題, 還是只是其中一個後面才會遇到的問題?
4. README 大段新增是否符合 upstream reviewer 想看的 scope?
5. tests/example.c 的 patch 是否足夠乾淨, portable, and minimal?
```

如果沒有先重新驗證, 很容易把舊 branch 的結果當成事實. 這不符合比較嚴謹的 PR 流程.

## Better Mental Model

現在比較合理的做法是把舊結果降級成線索:

```text
upstream/main baseline
  -> run normal build as a user
  -> observe exact failure
  -> classify failure layer
  -> compare with windows-first-attempt
  -> decide whether old patch addresses the real root cause
```

圖:

```text
upstream/main
    |
    v
fresh Windows build attempt
    |
    +--> succeeds
    |      old patch may be obsolete or environment-specific.
    |
    +--> fails at shell / Makefile helper
    |      investigate PowerShell vs MSYS2 shell boundary.
    |
    +--> fails at SDL2 detection/linking
    |      investigate pkg-config, sdl2-config, library path.
    |
    +--> fails at localtime_r/time_t
           compare with learn/windows-first-attempt.
```

## Hypotheses To Test

目前可以列成三個 hypothesis:

```text
H1:
  In MSYS2 UCRT64 shell, upstream/main can build normally.

H2:
  In PowerShell with UCRT64 tools on PATH, make may fail because Makefile shell helpers are parsed by cmd.exe.

H3:
  After shell and dependency detection are correct, tests/example.c may still fail because Windows UCRT lacks POSIX localtime_r.
```

`learn/windows-first-attempt` 主要只對 H3 有參考價值. 它不能證明 H1 or H2.

## Environment Question

這裡要把 "Windows" 拆成不同 execution environments:

```text
MSYS2 UCRT64 shell
  Windows 上的 GNU-like build environment.
  有 gcc, make, sh/bash, pkg-config, SDL2 packages.
  最適合先當 Windows baseline.

PowerShell + MSYS2 UCRT64 bin on PATH
  IDE terminal 常見用法.
  可能找到 gcc/make/pkg-config, 但 Makefile 的 $(shell ...) 可能由 cmd.exe 解讀.
  目前 dry-run 已看到 `pkg was unexpected at this time.`

Pure Windows / MSVC / nmake
  目前不像 repo 原本支援路線.
  Makefile, Kconfig, pkg-config, SDL2 flow 都偏 GNU/MSYS2.
```

所以現在不要急著說 "Windows 不能 build" or "Windows 能 build". 要先說清楚:

```text
哪一種 Windows shell?
哪一套 compiler?
哪一套 make?
SDL2 由哪個 package manager 提供?
```

## Decision For Now

目前決策:

```text
不直接沿用 learn/windows-first-attempt.
不假裝它完全不存在.
先從 upstream/main or clean exp branch 重新驗證 baseline.
把舊 branch 當成 H3 的 historical evidence.
```

比較建議的流程:

```text
1. 建 clean experiment branch or worktree from upstream/main.
2. 在 MSYS2 UCRT64 shell 跑 official-ish route:
   make defconfig
   make
   make check

3. 在 PowerShell 跑同樣 commands, 記錄差異.
4. 如果出現 localtime_r/time_t failure, 再對照 learn/windows-first-attempt.
5. 只有 root cause 很清楚時, 才開 fix/* branch.
```

## Branch Strategy

可以使用:

```text
learn/repo-review
  保存這類思考筆記.

exp/windows-baseline
  做重新驗證 upstream/main 的實驗.
  可以 messy, 但最好記錄 command and output.

fix/windows-localtime
  只有確認 localtime_r/time_t 是真正 upstream-facing issue 時才開.
```

如果不想在同一個資料夾一直切 branch, 可以之後考慮 `git worktree`:

```sh
git fetch upstream
git worktree add ..\libiui-exp-windows -b exp/windows-baseline upstream/main
```

這樣:

```text
current repo
  learn/repo-review notes.

new worktree
  clean upstream/main experiment.
```

## Follow-ups

下一步可以選:

```text
Option A: Baseline first
  先開 clean experiment branch/worktree, 用 MSYS2 UCRT64 shell 跑 make defconfig/make/make check.
  優點是最嚴謹.

Option B: Minimal app lab first
  先不碰 SDL2 window, 用 mock renderer 做 ch06-01.
  優點是能先理解 public API and runtime.

Option C: Windows shell investigation first
  專門研究 PowerShell + GNU Make + POSIX shell helper 的問題.
  優點是直接面對目前 dry-run 看到的風險.
```

目前推薦:

```text
Option A -> Option B -> Option C
```

原因是先建立 baseline 才知道什麼是真問題. Minimal app lab 可以並行理解 runtime. PowerShell shell issue 可能會變成後續 docs or Makefile portability topic, 但現在不應該先假設它就是第一個 PR.
