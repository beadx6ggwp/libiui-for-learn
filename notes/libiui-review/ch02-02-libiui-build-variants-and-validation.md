# Ch02-02: libiui Build Variants and Validation

## 本章問題

[ch02-01-libiui-build-flow.md](ch02-01-libiui-build-flow.md) 建立的是 core build graph:

```text
Kconfig decision
  -> .config and src/iui_config.h
  -> Makefile source selection
  -> .c files compile into .o
  -> libiui.a or executable
```

但看懂這條主線還不夠. `libiui` 真正難讀的地方在於它有多種 build variants:

```text
normal desktop build
  SDL2 backend, libiui.a, libiui_example.

test build
  headless backend, isolated test config, libiui_test, Python headless harness.

WASM build
  wasm backend, emcc toolchain, browser artifacts.

generated files
  Kconfig headers, MD3 validation includes, nyancat data, WASM output.
```

> [Scope]
> 本章講 `libiui` 的 build variants and validation model: generated files, demo target, `make check`, CI matrix, WASM flow, build failure diagnosis, and small review observations. 不重新講 `.c -> .o -> .a` 的基礎, 也不深入 widget runtime.

## 為什麼要從 ch02-01 拆出來

naive 做法是把所有 build 相關內容放在同一篇:

```text
Kconfig
Makefile
source selection
generated files
tests
CI
WASM
failure diagnosis
```

問題是這會把兩種問題混在一起:

```text
core build graph
  這次哪些 .c files and flags 會進 build?

validation and variants
  這個 repo 如何確認不同 build shape 都能工作?
```

拆開後閱讀順序比較清楚:

```text
ch02-01
  how build graph is formed.

ch02-02
  how build graph is varied, generated, tested, and diagnosed.
```

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

規則是:

```text
generated file 不應手改.
如果要改 generated output, 應改 source DSL or generator.
```

這跟 shader compiler, parser generator, asset pipeline 是同一類思想:

```text
source of truth
  -> generator
  -> generated code/data
  -> compiled into target
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

圖:

```text
tests/
  |
  +--> example.c
  |      interactive demo target
  |
  +--> test-*.c
  |      C unit tests
  |
  +--> common.c / common.h
         test support code
```

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

## 三種 build shape 放在一起看

```text
normal desktop build
  .config
  src/iui_config.h
  CONFIG_PORT_SDL2=y
  ports/sdl2.c
  libiui.a
  libiui_example

test build
  mk/test.mk overrides
  .build/test/iui_config.h
  CONFIG_PORT_HEADLESS=y
  ports/headless.c
  libiui_test
  .build/test/libiui.a
  scripts/headless-test.py

wasm build
  configs/wasm_defconfig
  CONFIG_PORT_WASM=y
  ports/wasm.c
  emcc/emar/emranlib
  libiui_example.js
  libiui_example.wasm
```

這三種不是同一個 build 的不同名字. 它們是不同 product/test shapes.

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
  scan-build make

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

CI 的意義是把:

```text
我本機剛好能編
```

提升成:

```text
幾個重要 build shape 都能編.
```

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
> 1. Generated files 的 source of truth 通常不是 output file 本身, 而是 Kconfig, DSL, generator script, or source asset.
> 2. `make check` 使用 isolated headless test config, 不等於一般 user build; 它驗證的是更廣的 API/test surface.
> 3. CI matrix 的目的不是重複 local build, 而是驗證 formatting, static analysis, sanitizers, module matrix, headless, and WASM build shapes.

## 下一章

讀完 build variants and validation 後, 下一步可以往 tests 展開:

```text
ch03-00-software-testing-foundations.md
  開發端測試基礎: unit/integration/E2E, test double, control, observability.

ch03-01-gui-test-and-validation-model.md
  GUI 測試怎麼控制 frame, input, renderer output, headless backend, and validation evidence.
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
mk/test.mk
.gitignore
.github/workflows/main.yml
.ci/common.sh
.ci/install-deps.sh
.ci/check-format.sh
.ci/check-newline.sh
.ci/build-wasm.sh
configs/wasm_defconfig
scripts/detect-compiler.py
README.md
CONTRIBUTING.md
```
