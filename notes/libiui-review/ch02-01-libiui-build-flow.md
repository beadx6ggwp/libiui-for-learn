# Ch02-01: libiui Build Graph

## 本章問題

讀完 [ch02-00-c-build-system-foundations.md](ch02-00-c-build-system-foundations.md) 後, 你已經知道一個 C project 的底層 build primitive:

```text
.c + .h -> .o -> .a or executable
```

現在回到 `libiui`. 這個 repo 的難點不是 GCC 變成另一種東西, 而是同一份 source tree 可以被切成多種 build shape:

```text
desktop demo
  SDL2 backend + example executable.

headless test
  memory-only backend + deterministic tests.

WASM browser build
  wasm backend + Emscripten flags + browser artifacts.

reduced build
  disable some modules/features to reduce binary size.
```

> [Scope]
> 本章只建立 `libiui` 的 core build graph mental model: Kconfig decision, `.config`, `src/iui_config.h`, Makefile source selection, `mk/common.mk`, toolchain flags, and SDL2 dependency flags. Generated files, `make check`, CI matrix, WASM deployment, and build failure diagnosis moved to [ch02-02-libiui-build-variants-and-validation.md](ch02-02-libiui-build-variants-and-validation.md).

## 先校正一個容易混在一起的想法

你可能會先這樣理解:

```text
libiui 是跨平台 UI 渲染前端.
它接收 OS events, 產生 draw commands, 再交給不同 backend.
所以不同 OS, backend, module 都會影響編譯流程.
```

這段有一半是對的. 但如果用 maintainer/reviewer 的角度看, 它還不夠精確.

對的部分是:

```text
libiui runtime 確實需要跨平台邊界.
OS input and renderer backend 不能寫死在 core UI code 裡.
SDL2, headless, WASM 確實代表不同 backend.
module/feature 也確實會改變 build result.
```

不夠精確的部分是:

```text
你把 runtime architecture 和 build-time selection 放在同一層講.
```

runtime architecture 問的是:

```text
程式跑起來後, event 如何進入 UI core?
widget 如何更新 state?
draw request 如何交給 backend?
```

build-time selection 問的是:

```text
在編譯前, 這次要把哪些 .c files, generated headers, compiler flags, external libs, and targets 放進 build graph?
```

runtime view:

```text
OS events
   |
   v
backend port
   |
   v
iui_update_*()
   |
   v
UI core + widgets
   |
   v
renderer callbacks
   |
   v
pixels / framebuffer / canvas
```

build-time view:

```text
configs/Kconfig + configs/defconfig
   |
   v
.config + src/iui_config.h
   |
   v
Makefile expands CONFIG_*
   |
   v
selected .c files + flags + libs
   |
   v
.o files -> libiui.a -> demo/test/wasm artifacts
```

兩張圖會互相影響, 但不要混用.

如果 runtime 需要 SDL2 backend, build-time 就要選:

```text
CONFIG_PORT_SDL2=y
  -> compile ports/sdl2.c
  -> add SDL2 cflags
  -> add SDL2 linker flags
```

但 build system 本身不處理 OS event. 它只是在決定:

```text
這次 binary 要包含哪個能處理 OS event 的 backend implementation.
```

所以更精確的說法是:

```text
libiui 的 source tree 支援多種 runtime shape.
Build system 的任務是把 config decision 轉成某一個 concrete build graph.
Kconfig 管理 decision.
Makefile 把 decision 展開成 source selection, flags, generated files, targets, and tests.
```

## 為什麼 naive Makefile 不夠

對小專案來說, 你可以寫:

```make
app: main.o foo.o bar.o
	$(CC) -o app main.o foo.o bar.o
```

但 `libiui` 不適合只用固定 source list. 如果把所有東西都無條件編進去, 會出現幾個問題:

```text
1. SDL2 不存在的環境會被迫 build SDL2 code.
2. WASM build 不能使用 native SDL2 assumption.
3. 關掉 module 後, source 仍被編進去, 裁切就失去意義.
4. 測試需要完整 API surface, 但 user build 可能故意關掉 module.
5. compiler flags scattered 到命令列, CI, README, local scripts, 很容易不一致.
```

所以 `libiui` 選擇這個方向:

```text
Kconfig
  管理 feature/backend/build option.

Makefile
  根據 CONFIG_* 選 source files, target, flags, generated files.

src/iui_config.h
  讓 C code 也能看見同一份 CONFIG_*.

mk/*.mk
  把通用 compile/archive/link/test/toolchain 規則拆出來.
```

## 總流程圖

最常見路線:

```text
make defconfig
  -> tools/kconfig/
  -> .config
  -> src/iui_config.h

make
  -> .build/*.o
  -> libiui.a
  -> libiui_example
```

完整一點的 graph:

```text
configs/Kconfig + configs/defconfig
        |
        | make defconfig
        v
      .config --------------------+
        |                         |
        | genconfig.py            | included by Makefile
        v                         v
src/iui_config.h           Makefile sees CONFIG_*
        |                         |
        +------------+------------+
                     |
                     v
          selected src/ + ports/ + tests/
                     |
                     v
          .build/*.o -> libiui.a -> demo artifacts
```

這張圖可以拆成四個時間點:

```text
time 0: make parses Makefile
  -include .config
  include mk/toolchain.mk
  include mk/deps.mk
  include mk/test.mk
  declare target.a-y, target-y, *_files-y

time 1: config generation
  make defconfig
    -> tools/kconfig/defconfig.py
    -> .config
    -> tools/kconfig/genconfig.py
    -> src/iui_config.h

time 2: build graph expansion
  CONFIG_MODULE_BASIC=y
    -> libiui.a_files-y += src/basic.c
  CONFIG_PORT_SDL2=y
    -> libiui.a_files-y += ports/sdl2.c
    -> ask SDL2 cflags/libs

time 3: compiler/linker actions
  selected .c -> .build/*.o
  .build/*.o -> libiui.a
  tests/example.o + libiui.a + backend libs -> libiui_example
```

## `.config` and `src/iui_config.h`

重點是:

```text
.config
  給 Makefile 用.
  決定 target, source list, flags, dependencies.

src/iui_config.h
  給 C preprocessor 用.
  讓 source code 可以 #ifdef CONFIG_*.
```

為什麼需要兩份? 因為 Makefile and C compiler 活在不同階段:

```text
Makefile stage
  還沒編譯 C.
  這時要先決定哪些 .c file 要被送進 compiler.

C preprocessing stage
  已經選好某個 .c file.
  這時 code 內部需要知道 feature 是否啟用.
```

同一個 decision 會投影成兩種 artifact:

```text
Kconfig decision
        |
        +--> .config
        |     build-system projection
        |
        +--> src/iui_config.h
              C-preprocessor projection
```

如果只有 `src/iui_config.h`, Makefile 很難在 compile 前選 source list. 如果只有 `.config`, C code 又看不到 `CONFIG_*`.

## Kconfig 是什麼問題的答案

先不要把 Kconfig 想成 Linux kernel 專用工具. 從 first principles 看, Kconfig 是一種 feature selection language.

它要解決的問題是:

```text
有很多 boolean/string choice.
有些 option 依賴另一些 option.
有些 option 只能三選一.
有些 option 的 default 取決於工具鏈或外部套件.
最後要產生一份穩定 config.
```

Kconfig 把這些 decision 集中在 `configs/Kconfig`:

```text
Toolchain Configuration
  compiler type, emcc availability.

Dependency Detection
  SDL2 availability.

Backend Selection
  CONFIG_PORT_SDL2
  CONFIG_PORT_WASM
  CONFIG_PORT_HEADLESS

Core Modules
  CONFIG_MODULE_BASIC
  CONFIG_MODULE_INPUT
  CONFIG_MODULE_CONTAINER
  ...

Feature Toggles
  CONFIG_FEATURE_ACCESSIBILITY
  CONFIG_FEATURE_VECTOR
  CONFIG_FEATURE_ICONS
  ...

Build Options
  CONFIG_OPTIMIZE_SIZE
  CONFIG_DEBUG_SYMBOLS
  CONFIG_LTO
  CONFIG_SANITIZERS
```

`make defconfig` 的本質是:

```text
把一組 project default decision materialize 成本機 build 可以使用的 config artifacts.
```

這裡的 "default" 不等於 "永遠正確". 它只是 repo 提供的一組常用選擇:

```text
configs/defconfig
  desktop-friendly default.
  usually selects SDL2 and full feature set when dependencies exist.

configs/wasm_defconfig
  browser/WASM-oriented default.
  selects WASM port and size optimization.

test config
  generated by mk/test.mk.
  forces headless and broad API coverage.
```

## Makefile 如何把 CONFIG 變成 source list

核心 library target 是:

```make
target.a-y := libiui.a
```

always included sources:

```text
src/core.c
src/event.c
src/layout.c
src/draw.c
src/font.c
```

optional module sources 用這種寫法:

```make
libiui.a_files-$(CONFIG_MODULE_BASIC) += src/basic.c
```

如果:

```text
CONFIG_MODULE_BASIC=y
```

Make variable 展開後就等於:

```make
libiui.a_files-y += src/basic.c
```

如果:

```text
CONFIG_MODULE_BASIC=n
```

就會變成:

```make
libiui.a_files-n += src/basic.c
```

而 `mk/common.mk` 收集 object 時只看 `*_files-y`, 所以 `*_files-n` 不會進 target.

圖:

```text
CONFIG_MODULE_BASIC=y
       |
       v
libiui.a_files-y += src/basic.c
       |
       v
src/basic.c -> .build/basic.o -> libiui.a

CONFIG_MODULE_BASIC=n
       |
       v
libiui.a_files-n += src/basic.c
       |
       v
not selected
```

backend selection 也是同一個概念:

```text
CONFIG_PORT_SDL2
  add ports/sdl2.c
  add SDL2 cflags/libs

CONFIG_PORT_HEADLESS
  add ports/headless.c

CONFIG_PORT_WASM
  add ports/wasm.c
  use Emscripten-specific flags when CC is emcc
```

source graph 不是固定的:

```text
always included core
  src/core.c
  src/event.c
  src/layout.c
  src/draw.c
  src/font.c
        |
        v
  libiui.a_files-y
        |
        +-------------------------+
        |                         |
        v                         v
optional modules              selected backend
  src/basic.c                   ports/sdl2.c
  src/input.c                   ports/headless.c
  src/container.c               ports/wasm.c
  src/list.c
  src/menu.c
  ...
        |                         |
        +------------+------------+
                     |
                     v
                 .build/*.o
                     |
                     v
                  libiui.a
```

Reviewer 會在意這件事:

```text
source file selected
  means it is compiled into this build shape.

source file exists in repo
  does not mean it is compiled in every build shape.
```

因此看 bug 時要問:

```text
bug exists in source file?
or bug exists in selected build shape?
```

## `mk/common.mk`: 把宣告變成規則

主 `Makefile` 比較像 project-specific declaration:

```text
有哪些 target.
每個 target 有哪些 source.
每個 target 需要哪些 include path, flags, libs, dependencies.
```

`mk/common.mk` 則是 generic build engine:

```text
src/foo.c   -> .build/foo.o
tests/foo.c -> .build/foo.o
ports/foo.c -> .build/foo.o

target.a-y  -> ar rcs
target-y    -> cc link
```

實際 compile rule 大概是:

```text
$(CC) $(CFLAGS) $(GLOBAL_EXTRA_CFLAGS) \
  -MMD -MP -MF .build/foo.d \
  -Iinclude -Isrc -Itests -Iports -Iexternals \
  -c -o .build/foo.o src/foo.c
```

這幾個 flag 可以先這樣讀:

```text
-Iinclude
  增加 header search path.

-c
  只 compile, 不 link.

-o .build/foo.o
  指定 object output.

-MMD -MP -MF .build/foo.d
  讓 compiler 產生 dependency file.
```

dependency file 可以先想成 compiler 幫 make 寫的小地圖:

```text
.build/foo.o: src/foo.c include/iui.h src/internal.h src/iui_config.h
```

## `mk/toolchain.mk`: compiler flags 也要被管理

compiler 不是只有一種:

```text
native gcc or clang
  build desktop executable.

emcc
  build WebAssembly target.

cross compiler
  build for different architecture or sysroot.
```

`mk/toolchain.mk` 做的是:

```text
1. 選 compiler.
2. 偵測 compiler type.
3. 根據 Kconfig 組出 CFLAGS and LDFLAGS.
```

例如:

```text
CONFIG_OPTIMIZE_SIZE -> -Os
else                 -> -O2
CONFIG_DEBUG_SYMBOLS -> -g
CONFIG_LTO           -> -flto
CONFIG_SANITIZERS    -> -fsanitize=address,undefined
```

這個 design 的用意是把 build policy 集中:

```text
不要每個人自己在命令列亂加 flags.
不要 CI 和 local build 用不同語意的 option.
不要讓 source code 決定 optimization or sanitizer policy.
```

## `mk/deps.mk`: 外部套件不能硬編

SDL2 flags 不應該寫死成:

```text
-I/usr/include/SDL2 -lSDL2
```

原因很直接:

```text
Linux, macOS, MSYS2 的 install path 可能不同.
不同發行版提供的 config tool 可能不同.
Emscripten or headless build 不該依賴 SDL2.
```

`mk/deps.mk` 用兩條常見路徑找 dependency:

```text
sdl2-config
  如果存在, 優先用.

pkg-config
  否則 fallback to pkg-config.
```

這裡又分成兩種 detection:

```text
Kconfig detection
  configs/Kconfig asks:
    does pkg-config --exists sdl2 work?
  result:
    CONFIG_HAVE_SDL2=y or n

Makefile dependency flags
  mk/deps.mk asks:
    what cflags/libs should be used for sdl2?
  result:
    -I..., -L..., -l...
```

圖:

```text
dependency availability
  pkg-config --exists sdl2
        |
        v
  CONFIG_HAVE_SDL2=y
        |
        v
  CONFIG_PORT_SDL2 can be selected

dependency flags
  sdl2-config --cflags/libs
  or pkg-config --cflags/libs sdl2
        |
        v
  compiler sees SDL2 headers
  linker sees SDL2 libraries
```

這兩件事不要混在一起:

```text
HAVE_SDL2=y
  只代表 Kconfig 認為 SDL2 exists.

correct cflags/libs
  才代表 compiler/linker 能真的使用 SDL2.
```

在 Windows/MSYS2 上, 這點特別重要. 因為 `pkg-config` 可能能找到 SDL2, 但 `sdl2-config` 可能是 shell script, 在不同 shell entry 下不一定能直接執行.

## 本章學到的三件事

> [Summary]
> 1. `ch02-01` 的核心不是 runtime event/draw flow, 而是 build-time selection 如何把 config decision 變成 concrete build graph.
> 2. `.config` 給 Makefile 選 target/source/flags, `src/iui_config.h` 給 C code 做 `#ifdef CONFIG_*`.
> 3. `libiui.a_files-$(CONFIG_*)` 的重點是 source selection: 檔案存在於 repo 不代表它會被每個 build shape 編進去.

## 下一章

接著讀 [ch02-02-libiui-build-variants-and-validation.md](ch02-02-libiui-build-variants-and-validation.md). 那篇接續本章的 build graph, 說明 generated files, demo target, `make check`, CI matrix, WASM flow, and build failure diagnosis.

## Source Files Read

```text
Makefile
mk/common.mk
mk/toolchain.mk
mk/deps.mk
configs/Kconfig
configs/defconfig
configs/wasm_defconfig
.config
src/iui_config.h
```
