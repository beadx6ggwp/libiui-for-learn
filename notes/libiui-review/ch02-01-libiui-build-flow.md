# Ch02-01: libiui Build Flow

## 本章問題

讀完 [ch02-00-c-build-system-foundations.md](ch02-00-c-build-system-foundations.md) 後, 你已經知道一個 C project 的底層 build primitive:

```text
.c + .h -> .o -> .a or executable
```

現在回到 `libiui`. 這個 repo 的困難不是 GCC 不一樣, 而是它不是單一固定 binary. 它要同時回答:

```text
這次要編哪些 widget module?
這次要用 SDL2, headless, 還是 WASM backend?
哪些 C source 應該被加入 library?
哪些 compiler flags 由 config 決定?
哪些 header 或 .inc 是 generator 產生的?
測試應該跟使用者目前的 .config 綁在一起嗎?
CI 要怎麼確認不同 build shape 都沒壞?
```

> [Scope]
> 本章講 `libiui` 的 repo-specific build flow: `make defconfig`, `.config`, `src/iui_config.h`, Kconfig, `Makefile`, `mk/*.mk`, generated files, tests, CI, and WASM. 不深入 widget runtime model, backend implementation, or API design.

## 為什麼 naive Makefile 不夠

對小專案來說, 你可以寫:

```make
app: main.o foo.o bar.o
	$(CC) -o app main.o foo.o bar.o
```

但 `libiui` 不適合只用固定 source list. 原因不是形式複雜, 而是專案本身有多種產品形狀:

```text
desktop demo
  SDL2 backend + example binary.

headless test
  memory-only backend + deterministic tests.

WASM browser build
  wasm backend + Emscripten flags + browser assets.

reduced embedded-style build
  關掉部分 widgets/features, 減少 binary size.
```

如果把所有東西都無條件編進去, 會出現幾個問題:

```text
1. SDL2 不存在的環境會被迫 build SDL2 code.
2. WASM build 不能使用 native SDL2 assumption.
3. 關掉 module 後, source 仍被編進去, 裁切就失去意義.
4. 測試需要完整 API surface, 但 user build 可能故意關掉 module.
5. compiler flags scattered 到命令列, CI, README, local scripts, 很容易不一致.
```

所以 `libiui` 最終選擇了這個方向:

```text
Kconfig
  管理 feature/backend/build option.

Makefile
  根據 CONFIG_* 選 source files, target, flags, generated files.

src/iui_config.h
  讓 C code 也能看見同一份 CONFIG_*.

mk/*.mk
  把通用 compile/archive/link/test/toolchain 規則拆出來.

CI matrix
  用多種 build shape 驗證不是只有 default build 能過.
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

make check
  -> .build/test/iui_config.h
  -> libiui_test
  -> .build/test/libiui.a
  -> scripts/headless-test.py
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
          .build/*.o -> libiui.a -> demo/test artifacts
```

重點是:

```text
.config
  給 Makefile 用. 它決定 target, source list, flags, dependencies.

src/iui_config.h
  給 C preprocessor 用. 它讓 source code 可以 #ifdef CONFIG_*.
```

為什麼需要兩份? 因為 Makefile 和 C compiler 活在不同階段:

```text
Makefile stage
  還沒編譯 C. 這時要先決定哪些 .c file 要被送進 compiler.

C preprocessing stage
  已經選好某個 .c file. 這時 code 內部需要知道 feature 是否啟用.
```

如果只有 `src/iui_config.h`, Makefile 很難在 compile 前選 source list. 如果只有 `.config`, C code 又看不到 `CONFIG_*`. 所以同一份 Kconfig decision 需要輸出到兩個世界.

## Kconfig 是什麼問題的答案

先不要把 Kconfig 想成 Linux kernel 專用工具. 從第一性原理看, Kconfig 是一種 feature selection language.

它要解決的問題是:

```text
有很多 boolean/string choice.
有些 option 依賴另一些 option.
有些 option 只能三選一.
有些 option 的 default 取決於工具鏈或外部套件.
最後要產生一份穩定 config.
```

如果不用 Kconfig, 你可能會把設定散落成:

```sh
make PORT=sdl2 ENABLE_BASIC=1 ENABLE_INPUT=1 USE_SANITIZER=0
```

這在小專案可以接受, 但一多就會失控:

```text
命令列太長.
option 之間的 dependency 不清楚.
default value 不集中.
CI 和使用者文件容易不同步.
source code 和 Makefile 可能讀到不同設定.
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

Examples
  CONFIG_DEMO_EXAMPLE
  CONFIG_DEMO_CALCULATOR
  ...

Build Options
  CONFIG_OPTIMIZE_SIZE
  CONFIG_DEBUG_SYMBOLS
  CONFIG_LTO
  CONFIG_SANITIZERS
```

所以 `make defconfig` 的本質是:

```text
把一組 project default decision materialize 成本機 build 可以使用的 config artifacts.
```

具體命令由 `Makefile` 定義:

```sh
python3 tools/kconfig/defconfig.py --kconfig configs/Kconfig configs/defconfig
python3 tools/kconfig/genconfig.py --header-path src/iui_config.h configs/Kconfig
```

如果 `tools/kconfig/kconfiglib.py` 不存在, `make defconfig` 會先 clone `Kconfiglib` 到 `tools/kconfig/`. 這代表第一次跑可能會碰到 network dependency. 之後它就是 local tool.

## Makefile 如何把 CONFIG 變成 source list

核心 library target 是:

```make
target.a-y := libiui.a
```

這可以讀成:

```text
要產生一個 static library target: libiui.a.
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

這行是理解整個 build flow 的小關鍵. 假設:

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

所以 source graph 不是固定的:

```text
CONFIG_MODULE_* --------+
                       v
core source list -> libiui.a_files-y -> .build/*.o -> libiui.a
                       ^
CONFIG_PORT_* ----------+
```

這就是為什麼改 code 時不能只問 "這個檔案能不能編過". 你還要問:

```text
這個 symbol 是否只在某個 CONFIG_* 下存在?
關掉 module 後, include 或 call site 是否還成立?
換 backend 後, platform assumption 是否還成立?
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

這裡有幾個值得先懂的 flag:

```text
-Iinclude
  增加 header search path.

-c
  只 compile, 不 link.

-o .build/foo.o
  指定 object output.

-MMD -MP -MF .build/foo.d
  讓 compiler 產生 dependency file. 如果 foo.c include 的 header 改了, make 可以知道 foo.o 要重編.
```

dependency file 可以先想成 compiler 幫 make 寫的小地圖:

```text
.build/foo.o: src/foo.c include/iui.h src/internal.h src/iui_config.h
```

這讓 build system 不需要手動列出每個 `.h` dependency.

## `mk/toolchain.mk`: 為什麼 compiler flags 也要被管理

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

如果 `.config` 選了:

```text
CONFIG_PORT_WASM=y
```

而 user 沒有手動指定 `CC`, `mk/toolchain.mk` 會嘗試自動切到:

```text
CC := emcc
AR := emar
RANLIB := emranlib
```

這不是為了複雜化. 這是在承認 WASM build 其實不是 native build 的變種, 而是另一個 target platform.

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

所以 SDL2 backend 的 build graph 是:

```text
CONFIG_PORT_SDL2=y
       |
       +--> ports/sdl2.c
       |
       +--> $(call dep,cflags,sdl2)
       |
       +--> $(call dep,libs,sdl2)
```

這也是 Windows/MSYS2 之後要調查的邊界:

```text
MSYS2 UCRT64 有沒有 sdl2-config?
pkg-config 是否能找到 sdl2?
Kconfig 的 HAVE_SDL2 判斷和 Makefile 的 dep(sdl2) 是否一致?
```

目前先記錄為 investigation point, 不先假設答案.

## Generated Files: 為什麼有些檔案不該手改

generated file 的第一性原理是:

```text
某些資訊的 source of truth 不適合直接手寫 C.
所以先寫更高階或更集中的描述, 再由 script 產出 C include file.
```

`libiui` 目前有幾類 generated artifacts:

```text
Kconfig:
  .config
  src/iui_config.h

MD3 DSL:
  src/md3-flags-gen.inc
  src/md3-validate-gen.inc
  tests/test-md3-gen.inc

Nyancat:
  tests/nyancat-data.h

WASM:
  assets/web/libiui_example.js
  assets/web/libiui_example.wasm
```

以 MD3 validation 為例:

```text
src/md3-spec.dsl
scripts/gen-md3-validate.py
        |
        v
src/md3-flags-gen.inc
src/md3-validate-gen.inc
tests/test-md3-gen.inc
```

這個設計背後的想法是:

```text
Material Design 3 spec rules 應該集中描述.
runtime validation 和 tests 應該從同一份 spec 產生.
避免 source code 和 tests 各自手寫後 drift.
```

所以規則是:

```text
generated file 不應手改.
如果要改 generated output, 應改 source DSL or generator.
```

## Demo Target: 為什麼 tests/ 不只是 tests

如果:

```text
CONFIG_DEMO_EXAMPLE=y
```

Makefile 會加入:

```text
target-y += libiui_example
```

demo source 是:

```text
tests/example.c
```

部分 demo source 依 config 追加:

```text
CONFIG_DEMO_PIXELWALL
  tests/pixelwall-demo.c

CONFIG_DEMO_FORTHSALON
  tests/forthsalon-demo.c
```

這代表 `tests/` 在這個 repo 裡不只是 unit tests:

```text
tests/example.c
  interactive demo app.

tests/test-*.c
  C unit test sources.

tests/common.c
  test helper and mock renderer support.

scripts/headless-test.py
  Python headless UI behavior test harness.
```

所以如果 failure 發生在 `tests/example.c`, 它可能是 demo build issue, 不一定是 core library issue.

## `make check`: 測試 build 不等於 user build

naive 想法:

```text
make check 就是拿目前 .config 跑測試.
```

`libiui` 不是這樣. `mk/test.mk` 會在 test goal 下強制:

```text
enable all modules.
enable important test features.
select CONFIG_PORT_HEADLESS=y.
use isolated build dir .build/test.
generate .build/test/iui_config.h.
```

圖:

```text
normal make
  .config
  src/iui_config.h
  .build/*.o
  libiui.a
  libiui_example

make check
  forced test CONFIG_*
  .build/test/iui_config.h
  .build/test/*.o
  libiui_test
  .build/test/libiui.a
  scripts/headless-test.py
```

為什麼要這樣? 因為 user `.config` 可能把某些 module 關掉. 如果測試完全依賴 user config, 那就可能出現:

```text
某個 widget 壞了.
但使用者本機 config 剛好沒啟用它.
make check 沒測到.
```

headless backend 的價值是 deterministic:

```text
不需要真的開視窗.
不依賴 mouse/keyboard real device.
可以人工注入 input.
可以檢查 draw calls, framebuffer, widget state.
適合 CI.
```

所以這裡有一個很重要的區分:

```text
normal build
  validates chosen product config.

test build
  validates broad API surface under headless backend.
```

如果你改的是 config-dependent or backend-dependent code, 只說 `make check` 過了可能不夠. 你可能還要補:

```text
make defconfig
make

or targeted config build
or WASM build
or Windows/MSYS2 build
```

## CI Matrix: 為什麼 local build 過了還不等於 PR 準備好

`.github/workflows/main.yml` 不是只跑 `make`. 它大致覆蓋:

```text
coding-style
  .ci/check-newline.sh
  .ci/check-format.sh

static-analysis
  make defconfig
  make clean
  scan-build-20 make

unit-tests
  Ubuntu and macOS
  make defconfig
  make
  make check

headless-tests
  make defconfig
  make check-headless

sanitizers
  make defconfig
  echo CONFIG_SANITIZERS=y >> .config
  make check

build-matrix
  make defconfig
  append module config
  make libiui.a

wasm-build
  .ci/build-wasm.sh build
  .ci/build-wasm.sh deploy-prep
```

CI 的意義是把 "我本機剛好能編" 提升成 "幾個重要 build shape 都能編".

對這個 repo 來說, review evidence 應該思考:

```text
我改的是 core source?
  至少 make check.

我改的是 backend?
  跑對應 backend build or test.

我改的是 config/build files?
  跑 make defconfig, make, make check, 也看 CI matrix 是否受影響.

我改的是 WASM path?
  跑 WASM build flow or 至少說明本機沒有 emcc.

我改的是 formatting-sensitive C code?
  跑 make indent or check format.
```

## WASM Flow: 另一個 target platform

WASM 不是單純 `CC=emcc make` 這麼簡單. CI flow 是:

```sh
make defconfig
python3 tools/kconfig/defconfig.py --kconfig configs/Kconfig configs/wasm_defconfig
python3 tools/kconfig/genconfig.py --header-path src/iui_config.h configs/Kconfig
CC=emcc AR=emar RANLIB=emranlib make
```

`configs/wasm_defconfig` 指定:

```text
CONFIG_PORT_WASM=y
CONFIG_OPTIMIZE_SIZE=y
```

其他 option 由 Kconfig defaults 補齊.

部署準備階段會把:

```text
assets/web/index.html
assets/web/iui-wasm.js
libiui_example.js
libiui_example.wasm
```

整理到:

```text
deploy/
```

這個 flow 之後要和 `scripts/serve-wasm.py`, `assets/web/index.html`, and `ports/wasm.c` 一起看. 那比較接近 backend/runtime review, 不在本章深入.

## 讀 Build Failure 的方法

看到 build failure 時, 不要只看最後一行. 先判斷它壞在哪一層:

```text
config generation failed
  Kconfig, tools/kconfig, Python, dependency detection.

compile failed
  某個 .c -> .o 失敗. 看 include path, CONFIG_*, missing declaration, compiler flag.

archive failed
  ar rcs libiui.a 失敗. 通常是 object list or filesystem 問題.

link failed
  symbol unresolved, duplicate symbol, missing external lib, wrong ldflags.

test failed
  executable 能 build, 但 behavior expectation 不符.

headless script failed
  C library build 可能成功, 但 Python harness, generated mini app, input simulation, or validation 出問題.
```

ASCII map:

```text
Kconfig -> .config -> source selection -> compile -> archive -> link -> run/test
   |          |              |              |          |        |       |
   v          v              v              v          v        v       v
config    Make vars      wrong .c       C error     ar err  ld err  behavior
error     wrong          chosen
```

這個分類很重要. 因為同樣是 "build failed", 修法完全不同.

## Formatting Target 的觀察

`CONTRIBUTING.md` 提到:

```sh
make format
```

但目前 `Makefile` 實際有:

```sh
make indent
```

CI 則跑:

```sh
.ci/check-format.sh
```

這裡可能是文件和 Makefile target 的差異. 目前先記錄為 review observation, 不立刻視為 bug:

```text
Observation:
  CONTRIBUTING.md mentions make format, but Makefile defines indent.

Possible follow-up:
  confirm whether make format should exist, or docs should say make indent.
```

這是 future small docs/build PR candidate, 但需要先確認 maintainer expectation.

## 本章學到的三件事

> [Summary]
> 1. `libiui` 使用 Kconfig 是因為它不是單一固定產品, 而是由 module, feature, backend, demo, and toolchain option 組成的多種 build shape.
> 2. `.config` 給 Makefile 選 target/source/flags, `src/iui_config.h` 給 C code 做 `#ifdef CONFIG_*`.
> 3. `make check` 使用 isolated headless test config, 不等於一般 user build; CI 會額外檢查 format, static analysis, sanitizer, build matrix, and WASM.

## 下一章

讀完 build flow 後, 下一步可以往 runtime model 或 tests 展開:

```text
ch03-00-software-testing-foundations.md
  開發端測試基礎: unit/integration/E2E, test double, control, observability.

ch03-01-gui-test-and-validation-model.md
  GUI 測試怎麼控制 frame, input, renderer output, headless backend, and validation evidence.

ch04-runtime-model.md
  input -> layout -> widget state -> draw command -> backend.
```

## Open Questions

```text
1. CONTRIBUTING.md 的 make format 和 Makefile 的 indent 是否需要對齊?
2. Windows/MSYS2 下 Kconfig 的 HAVE_SDL2 和 Makefile dep(sdl2) 是否都能正確找到 SDL2?
3. README 寫 make check SANITIZERS=1, 但 CI 用 echo CONFIG_SANITIZERS=y >> .config. 需要確認 SANITIZERS=1 是否仍是有效入口.
4. Generated MD3 files 被 .gitignore 忽略. 需要確認 release or CI 中是否總是由 generator 產生, 以及開發者是否要手動跑 make gen-md3.
```

## Source Files Read

```text
Makefile
mk/common.mk
mk/toolchain.mk
mk/deps.mk
mk/test.mk
configs/Kconfig
configs/defconfig
configs/wasm_defconfig
.config
src/iui_config.h
.gitignore
.github/workflows/main.yml
.ci/common.sh
.ci/install-deps.sh
.ci/check-format.sh
.ci/check-newline.sh
.ci/build-wasm.sh
scripts/detect-compiler.py
README.md
CONTRIBUTING.md
```
