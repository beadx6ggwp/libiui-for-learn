# MSYS Build Environment Thinking

## Question

現在要理解 `libiui` 的 Windows build, 不能只問:

```text
Windows 能不能 build?
```

這個問題太粗. 真正要拆成:

```text
哪個 shell?
哪個 compiler?
哪個 C runtime?
哪個 package manager?
哪個 dependency discovery tool?
哪個 build system assumption?
```

目前我的環境配置大致參考:

```text
https://hackmd.io/@beadx6ggwp/B1pTXpKQZg
```

這份 HackMD 比較像個人 Windows C/C++ 開發環境總表, 包含 `winget`, MSYS2 UCRT64, `pacman`, PATH, VS Code IntelliSense, and MSYS/MINGW/UCRT64 差異. 它可以當背景資料, 但不能直接當作 `libiui` 的正式 build guide.

## Context

HackMD 目前記錄的主路線是:

```text
winget install VS Code / MSYS2 / Git
  -> open ucrt64.exe
  -> pacman -Syu
  -> pacman install UCRT64 toolchain and CMake
  -> add C:\msys64\ucrt64\bin to PATH
  -> VS Code 設定 compilerPath and includePath
```

這背後其實有幾層:

```text
Windows OS
  提供 Win32 API, file path rules, process launching, DLL loading.

Shell
  PowerShell, cmd.exe, MSYS2 bash.

Toolchain
  GCC from MSYS2 UCRT64.

C runtime
  UCRT, not old MSVCRT, not MSYS runtime.

Package manager
  pacman provides gcc, make, SDL2, pkg-config.

Editor layer
  VS Code IntelliSense has separate include/compiler settings.
```

如果這些層混在一起, 就會出現一種錯覺:

```text
我裝了 gcc, 所以 make 一定會正常.
```

但實際上不一定. `make` 本身還會呼叫 shell, shell 還會解析 recipes and `$(shell ...)`, dependency discovery 還會用 `pkg-config` or `sdl2-config`.

## Why This Matters For libiui

`libiui` 看起來是普通 C project:

```text
make defconfig
make
make check
```

但這個 "普通 make" 其實包含:

```text
Kconfig
  產生 .config and src/iui_config.h.

Makefile
  根據 CONFIG_* 選 src files, ports, demos, tests.

mk/deps.mk
  用 shell helper 找 SDL2 cflags/libs.

pkg-config / sdl2-config
  提供 dependency include path and linker flags.

compiler/linker
  真正把 C files 變成 executable.
```

所以 Windows build 問題可能發生在任何一層:

```text
shell cannot parse Makefile helper
dependency tool cannot run
SDL2 package not found
compiler cannot find headers
linker cannot find libs
C source uses POSIX-only API
runtime cannot find DLL
```

這些 failure 的解法完全不同. 如果沒有先分層, 就會很容易把症狀修成偶然可跑, 但不知道 root cause.

## MSYS2 Environments: Need A Mental Model

MSYS2 不是單一東西. 它比較像一組 environment:

```text
MSYS
  提供類 Unix shell and tools.
  用 MSYS runtime.
  適合跑 build tools, 不一定適合產出 standalone Windows app.

MINGW64
  GCC/MinGW 64-bit environment.
  runtime 偏舊 MSVCRT.

UCRT64
  GCC/MinGW 64-bit environment.
  runtime 是 Universal C Runtime.
  對現代 Windows 比較合理.

CLANG64
  LLVM/Clang based environment.
```

目前我的主環境是 UCRT64. 這代表:

```text
compiler path:
  C:\msys64\ucrt64\bin\gcc.exe

package prefix:
  mingw-w64-ucrt-x86_64-*

expected shell:
  usually MSYS2 UCRT64 shell, not arbitrary PowerShell session.

C runtime:
  UCRT.
```

這也代表安裝 SDL2 時應該用同一個 prefix:

```sh
pacman -S mingw-w64-ucrt-x86_64-SDL2
```

不要混成:

```text
UCRT64 gcc + MINGW64 SDL2
MSYS shell gcc + UCRT64 SDL2
MSVC + pacman SDL2
```

混用時, include path, library ABI, runtime DLL, and pkg-config metadata 都可能不一致.

## PowerShell vs MSYS2 Shell

目前最容易混淆的是:

```text
PowerShell 裡可以找到 gcc/make/pkg-config
```

不等於:

```text
Makefile 裡所有 POSIX shell 語法都能正常跑
```

原因是 GNU Make 在 Windows 上執行 `$(shell ...)` or recipe 時, 可能使用 `cmd.exe` as shell. 如果 Makefile 裡有:

```make
for pkg in $(2); do \
    if command -v $${pkg}-config >/dev/null 2>&1; then \
        ...
    fi; \
done
```

這是 POSIX shell 語法. 在 `cmd.exe` 裡不一定能解析.

目前已觀察到:

```text
pkg was unexpected at this time.
-v was unexpected at this time.
```

這個 failure 比較像 shell parsing issue, 不像 gcc or SDL2 missing.

所以之後要分開驗證:

```text
MSYS2 UCRT64 shell baseline
  make defconfig
  make
  make check

PowerShell baseline
  same commands
  compare where it fails
```

如果 MSYS2 UCRT64 shell 能過, 但 PowerShell 不行, 那問題是 Windows entry shell support. 它可能是 docs issue or Makefile portability issue, 不一定是 core code issue.

## VS Code IntelliSense Is A Separate Layer

HackMD 裡提到:

```text
compiler can find headers because PATH is configured.
IntelliSense does not automatically read system PATH.
VS Code needs includePath and compilerPath.
```

這點很重要, 因為 IDE 紅線不等於 build failure.

可以分成:

```text
actual build
  gcc, make, pkg-config, linker.

editor analysis
  VS Code C/C++ extension, includePath, compilerPath, IntelliSense mode.
```

如果 VS Code 顯示找不到 SDL2 header, 但 `make` 能 build, 那可能是 editor config issue. 如果 `make` 也找不到 header, 才是 build dependency issue.

## libiui-Specific Questions

接下來要驗證的不是一般 C++ 環境是否配置成功, 而是 `libiui` 的 build assumptions 是否和我的環境吻合:

```text
1. `make defconfig` 在 MSYS2 UCRT64 shell 是否能產生 .config and src/iui_config.h?
2. Kconfig 的 SDL2 detection 是否使用 pkg-config?
3. Makefile 的 SDL2 cflags/libs 是否走 sdl2-config or pkg-config?
4. `make` 是否能 build libiui.a and libiui_example?
5. `make check` 是否能跑 headless tests?
6. 如果 PowerShell 失敗, failure 是 shell parsing, path, dependency, compile, link, or runtime?
7. 如果 compile 到 `tests/example.c` 才失敗, 是否和 `localtime_r` / `time_t` 有關?
```

這些問題要用 command output 回答, 不是靠猜.

## Where This Should Live Later

目前這篇只是 timeline thinking note. 之後可以整理成正式資料夾:

```text
notes/windows-build/
```

可能章節:

```text
ch00-windows-c-build-environments.md
  Windows build problem 的分層: shell, toolchain, runtime, package manager, build system.

ch01-msys2-ucrt64-mental-model.md
  MSYS / MINGW64 / UCRT64 / CLANG64 差異, pacman package prefix, PATH.

ch02-libiui-windows-baseline.md
  clean upstream/main on MSYS2 UCRT64 shell 的實測 command and output.

ch03-powershell-vs-msys2-shell.md
  PowerShell 入口與 Makefile POSIX shell helper 的差異.

ch04-localtime-portability-case.md
  如果確認 `localtime_r` 是真正 issue, 再整理成 portable C patch case study.
```

但現在不要太早正式化. 因為還沒做 clean baseline.

## Decision

目前決策:

```text
先把 MSYS2/Windows build 的分層模型記在 timeline note.
暫時不寫正式 `notes/windows-build/` 教學.
下一步先做 clean baseline, 再決定哪些內容值得整理成教學章節.
```

推薦順序:

```text
1. MSYS2 UCRT64 shell baseline.
2. PowerShell baseline.
3. Minimal app lab if build layer清楚.
4. 若發現穩定 Windows issue, 再開 notes/windows-build/.
```

## Follow-ups

要做 baseline 時, 先記錄:

```text
shell name
PATH head
where gcc
where make
where python3
where pkg-config
gcc -dumpmachine
gcc --version
make --version
python3 --version
pkg-config --modversion sdl2
make defconfig
make
make check
```

如果失敗, 不要先改 code. 先記錄:

```text
command
shell
exact error
which layer failed
suspected root cause
next probe
```

這樣後面才可能形成乾淨 PR:

```text
problem
root cause
minimal patch
validation
```
