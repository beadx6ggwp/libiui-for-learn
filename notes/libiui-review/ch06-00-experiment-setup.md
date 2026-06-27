# ch06-00 - Experiment Setup

## 本章要解決的問題

讀完 [ch05-public-api-tour.md](ch05-public-api-tour.md) 後, 下一步不是立刻寫正式 PR. 比較好的問題是:

```text
我要怎麼建立一個可以反覆實驗 libiui 的工作場?
```

這個問題同時牽涉三件事:

```text
git strategy
  實驗 code, 學習 notes, 正式 PR patch 要放在哪些 branch?

build flow
  這份 repo 的 make, Kconfig, .config, generated header 如何決定 build shape?

environment
  Windows 上目前是 PowerShell, MSYS2 UCRT64, SDL2, pkg-config, or another shell?
```

如果這三件事沒有先分清楚, 很容易把 "我想學 UI runtime" 變成 "我在修 Windows build", 或把 "正式 PR patch" 混進個人 notes.

## 先定義實驗的目的

這裡的 experiment 不是 upstream patch. 它的目的比較像 renderer project 裡的 golden model test:

```text
small controlled program
  -> exercise one API path
  -> observe state or draw output
  -> write down what was learned
  -> only later decide whether there is a PR-worthy issue
```

所以實驗場要能回答:

```text
我能不能初始化 iui_context?
我能不能用最小 renderer callbacks 觀察 draw calls?
我能不能注入 input event?
我能不能重現一個 widget behavior?
我能不能把這個 observation 轉成 test or PR evidence?
```

不要一開始就問:

```text
我要改哪一行送 PR?
```

因為現在更重要的是建立 observation pipeline.

## Git Strategy: 三種 Branch 不要混

目前 repo 可以分成三種用途:

```text
learn/repo-review
  personal notes, repo review, experiment records.

exp/*
  throwaway or structured experiments.
  may contain scratch programs, local build notes, temporary probes.

fix/*
  clean upstream PR branch.
  starts from upstream/main.
  contains only the minimal patch and its tests/docs.
```

圖:

```text
upstream/main
      |
      O--------------------O
       \
        \ learn/repo-review
         N---N---N
              \
               \ exp/minimal-app
                E---E

upstream/main
      |
      O
       \
        \ fix/msys2-build
         F
```

重點是:

```text
learn branch can remember thinking.
exp branch can be messy but local.
fix branch must be reviewable.
```

正式 PR 不應該從 `learn/repo-review` 開. 如果未來真的發現 bug, 正式 patch 應該這樣開:

```sh
git fetch upstream
git switch -c fix/some-focused-issue upstream/main
```

如果你同時想保留 learn branch 又想在 clean upstream tree 上做 PR, 可以用 `git worktree`:

```sh
git fetch upstream
git worktree add ..\libiui-fix-some-issue -b fix/some-issue upstream/main
```

mental model:

```text
same git repository
  -> two working directories
  -> one for notes and learning
  -> one for clean PR patch
```

這樣可以避免一直在同一個資料夾切 branch, 也比較不容易把 notes 帶進 PR diff.

## 實驗 Branch 什麼時候需要

不是每個實驗都要新 branch.

可以留在 `learn/repo-review` 的情況:

```text
只新增或修改 notes.
只做 read-only command investigation.
只記錄環境狀態, 沒有新增 scratch source.
```

適合開 `exp/*` 的情況:

```text
要新增 scratch C file.
要改 Makefile or config 只是為了測試想法.
要跑一連串可能產生 build artifacts 的操作.
要比較不同 backend or config.
```

適合開 `fix/*` 的情況:

```text
已經知道問題 root cause.
已經能用簡短 commands 重現.
patch scope 可以很小.
不需要把 personal notes 放進 commit.
```

簡單判斷:

```text
learning record?
  learn/repo-review

temporary proof?
  exp/*

upstream patch?
  fix/* from upstream/main
```

## Build Flow: 不要把 make 當黑盒

對這個 repo, `make` 不是單純把所有 `.c` 編起來. 它先看 config:

```text
configs/Kconfig + configs/defconfig
        |
        v
make defconfig
        |
        +--> .config
        |     used by Makefile
        |
        +--> src/iui_config.h
              used by C preprocessor
```

然後 Makefile 根據 `CONFIG_*` 選 source files:

```text
CONFIG_PORT_SDL2=y
  -> add ports/sdl2.c
  -> add SDL2 cflags/libs

CONFIG_PORT_HEADLESS=y
  -> add ports/headless.c

CONFIG_MODULE_INPUT=y
  -> add src/input.c
```

所以實驗前要先知道:

```text
目前 .config 是什麼?
目前 src/iui_config.h 是什麼?
我要測的是 SDL2 app, headless test, or pure public API path?
```

如果只是想理解 public API 和 frame loop, 不一定要先跑 SDL2 window. 可以先用 headless or mock renderer, 因為它比較 deterministic.

## 目前環境快照

本章目前在 Windows + PowerShell 下檢查到:

```text
branch
  learn/repo-review
  ahead origin/learn/repo-review by 1 commit

remotes
  origin   https://github.com/beadx6ggwp/libiui-for-learn.git
  upstream https://github.com/sysprog21/libiui.git

compiler
  C:\msys64\ucrt64\bin\gcc.exe
  target: x86_64-w64-mingw32
  version: GCC 15.2.0, MSYS2 project

make
  C:\msys64\ucrt64\bin\make.exe
  GNU Make 4.4.1, x86_64-w64-mingw32

python
  C:\msys64\ucrt64\bin\python3.exe
  Python 3.12.12

SDL2 detection
  pkg-config --modversion sdl2 -> 2.32.10
  pkg-config --cflags sdl2 works
  pkg-config --libs sdl2 works

shell state
  COMSPEC=C:\WINDOWS\system32\cmd.exe
  SHELL not set
  MAKESHELL not set
  sh.exe exists at C:\msys64\usr\bin\sh.exe
  bash.exe exists at C:\msys64\usr\bin\bash.exe
  sh/bash are not currently on PATH
```

目前 `.config` 已存在, 而且是 SDL2 desktop build shape:

```text
CONFIG_HAVE_SDL2=y
CONFIG_PORT_SDL2=y
# CONFIG_PORT_HEADLESS is not set
CONFIG_DEMO_EXAMPLE=y
many modules and features enabled
```

這代表現在不是 clean "no config" state. 之後實驗時要記得:

```text
changing .config changes what make builds.
src/iui_config.h should match .config.
```

## Windows 上目前看到的 Build 風險

`mk/deps.mk` 的 dependency helper 用 POSIX shell 語法:

```make
define dep
$(shell \
    for pkg in $(2); do \
        if command -v $${pkg}-config >/dev/null 2>&1; then \
            $${pkg}-config --$(1) $(DEVNULL); \
        elif command -v $(PKG_CONFIG) >/dev/null 2>&1; then \
            $(PKG_CONFIG) --$(1) $$pkg $(DEVNULL); \
        fi; \
    done \
)
endef
```

這段在 Unix shell 裡合理, 但在目前 PowerShell 入口下, GNU Make 可能透過 `cmd.exe` 解讀 `$(shell ...)`. dry-run 時已經看到:

```text
pkg was unexpected at this time.
-v was unexpected at this time.
```

這不代表 `gcc` 不存在, 也不代表 SDL2 不存在. 目前更像是:

```text
toolchain exists
pkg-config works
but Makefile shell helper may be parsed by the wrong shell
```

另一個觀察是 `sdl2-config` 本身是沒有副檔名的 shell script:

```text
C:\msys64\ucrt64\bin\sdl2-config
  starts with #!/bin/sh
```

在 PowerShell 直接執行:

```sh
sdl2-config --version
```

會出現 access denied. 這是 Windows entry shell 問題, 不是 SDL2 library 本身不存在.

因此 Windows build 調查要分清楚:

```text
dependency detection by Kconfig
  configs/Kconfig uses pkg-config, currently can detect SDL2.

dependency flags by Makefile
  mk/deps.mk prefers sdl2-config, then pkg-config.
  this path depends on shell behavior.

actual compile/link
  only after dependency flags are correct, gcc/linker errors才有意義.
```

## 實驗前的 Read-Only Probe

每次進入新的 shell 或新機器, 先跑 read-only probes:

```sh
git status --short --branch
git remote -v
where.exe gcc
where.exe make
where.exe python3
where.exe pkg-config
where.exe sdl2-config
gcc -dumpmachine
gcc --version
make --version
python3 --version
pkg-config --modversion sdl2
pkg-config --cflags sdl2
pkg-config --libs sdl2
```

再看 config:

```sh
type .config
type src\iui_config.h
```

如果只想看 make 會做什麼, 先 dry-run:

```sh
make -n V=1 libiui.a
make -n V=1 libiui_example
make -n V=1 check
```

但 dry-run 不是完整驗證. 它能暴露 parse-time 問題, 不能證明真的能 compile/link/run.

## 實際 Build 前的兩條路

目前有兩條合理路線.

路線 A: 用 MSYS2 UCRT64 shell 做正式 build validation.

```text
優點:
  最接近 MSYS2 package expectation.
  sh/bash usually on PATH.
  Unix-style scripts such as sdl2-config behave normally.

代價:
  要明確記錄是在 MSYS2 UCRT64 shell, not PowerShell.
```

路線 B: 讓 PowerShell 入口也能穩定 build.

```text
優點:
  使用者可直接在目前 IDE terminal 執行.
  Windows onboarding 更友善.

代價:
  可能牽涉 Makefile shell portability.
  需要小心區分 local workaround and upstream-worthy patch.
```

目前不應該立刻改 Makefile. 比較好的實驗順序是:

```text
1. 在 MSYS2 UCRT64 shell 跑官方路線.
2. 在 PowerShell 跑同樣命令, 記錄差異.
3. 判斷差異是文件問題, environment issue, or Makefile portability issue.
4. 只有 root cause 很清楚時, 才開 fix/* branch.
```

## Minimal App Lab 要怎麼接

真正的 minimal app lab 應該分兩階段.

第一階段不需要 SDL2 window:

```text
mock renderer or headless renderer
  -> allocate iui_context
  -> call begin_frame/begin_window/widgets/end_frame
  -> record draw callbacks
```

這能先回答 public API 是否理解正確:

```text
renderer callbacks 何時被呼叫?
text_width 對 layout 有什麼影響?
button return value 何時變 true?
layout cursor 如何前進?
```

第二階段才接 platform backend:

```text
SDL2 backend
  -> actual window
  -> real input events
  -> software renderer to window surface
```

這樣分是因為:

```text
如果最小 renderer callbacks 都不懂, 直接開 SDL2 window 只會多一層噪音.
如果 headless behavior 已經懂, SDL2 問題就更容易定位到 backend or environment.
```

## 實驗資料夾要怎麼放

目前可以先不新增正式 `experiments/` 目錄. 因為 upstream repo 未必需要這個目錄, 而 learn branch 也不一定要長期保存 scratch code.

建議策略:

```text
notes/libiui-review/
  保存教學與實驗紀錄.

temporary local files
  只在 exp/* branch 上建立.
  如果只是 disposable, 不一定 commit.

正式測試
  如果最後變成 PR, 移到 tests/ or scripts/.
  只在 fix/* branch 上提交.
```

如果之後確定需要長期保存 learning experiments, 再考慮:

```text
notes/libiui-experiments/
  experiment writeups.

experiments/
  only in learn branch, not upstream PR.
```

先不要急著建立, 因為現在真正需要的是穩定實驗流程, 不是多一個資料夾.

## 下一章: Minimal App Lab

下一章可以寫 [ch06-01-minimal-app-lab.md](ch06-01-minimal-app-lab.md). 它應該回答:

```text
不用 SDL2 window, 只靠 renderer callbacks, 能不能跑一個最小 libiui frame?
```

預計路線:

```text
1. define a tiny renderer that logs draw_box, draw_text, set_clip_rect.
2. allocate buffer with iui_min_memory_size().
3. build iui_config_t with iui_make_config().
4. call iui_init().
5. run one frame with iui_begin_frame().
6. open one window.
7. call iui_text() and iui_button().
8. close window and frame.
9. inspect callback log.
```

這會把 ch05 的 public API contract 轉成可以觀察的 program.

## 本章學到的三件事

> [Summary]
> 1. 實驗場要先分清楚 `learn/*`, `exp/*`, and `fix/*`: learning notes, temporary proof, and upstream PR patch 不要混在同一條 branch.
> 2. `libiui` 的 build 不是單純 `make all`: `.config` 給 Makefile 選 source/flags, `src/iui_config.h` 給 C code 做 conditional compilation.
> 3. 目前 Windows 環境有 UCRT64 gcc/make/pkg-config/SDL2, 但 PowerShell entry shell 下 Makefile 的 POSIX shell helper 可能有風險, 所以要先記錄 shell boundary 再開始正式 build validation.

## Source Files and Commands Read

```text
Makefile
mk/deps.mk
configs/Kconfig
.config
src/iui_config.h
notes/libiui-review/ch05-public-api-tour.md

git status --short --branch
git remote -v
where.exe gcc
where.exe make
where.exe python python3 pkg-config sdl2-config bash
gcc -dumpmachine
gcc --version
make --version
python3 --version
pkg-config --modversion sdl2
pkg-config --cflags sdl2
pkg-config --libs sdl2
make -n V=1 libiui.a
make -n V=1 libiui_example
make -n V=1 check
```
