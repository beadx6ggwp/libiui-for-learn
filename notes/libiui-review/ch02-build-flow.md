# Ch02: Build Flow

## 本章定位

這個章節現在拆成兩篇. 原因是 `build flow` 其實有兩層問題:

```text
level 1:
  GCC 怎麼把 C source 變成 object file, static library, and executable?

level 2:
  libiui 怎麼用 Makefile, Kconfig, generated headers, backend selection, tests, and CI 管理多種 build shape?
```

如果直接從 `Makefile` 和 Kconfig 開始, 你會看到很多規則, 但不知道它們在保護什麼. 所以先讀 C build 基礎, 再回到 repo-specific build.

## Reading Order

```text
ch02-00-c-build-system-foundations.md
  GCC/C 編譯流程: preprocessing, compilation, assembling, linking, headers, object files, static libraries, dependency files, and Makefile basics.

ch02-01-libiui-build-flow.md
  libiui 的具體 build flow: Kconfig, .config, src/iui_config.h, CONFIG_* source selection, mk/*.mk, generated files, tests, CI, and WASM.
```

## 為什麼要拆

naive 想法是:

```text
先把 libiui 的 Makefile 看完就會懂 build.
```

問題是 `Makefile` 不是 build 的起點. 它只是把底層 compiler/linker 的變換組織起來. 如果還不熟:

```text
.c 和 .h 的差別
translation unit
object file
symbol
linker
static library
-I, -L, -l
dependency file
```

那 `libiui` 的 `target.a-y`, `libiui.a_files-y`, `CONFIG_PORT_SDL2`, `-MMD -MP`, `src/iui_config.h` 就會像任意規則. 拆章後閱讀順序比較接近:

```text
compiler/linker mental model
  -> generic build system problem
  -> libiui-specific Makefile and Kconfig design
```

## 本章學到的三件事

> [Summary]
> 1. `ch02` 現在是 build flow 的入口, 不是主要教學本文.
> 2. 先讀 `ch02-00`, 建立 GCC/C build 的底層模型.
> 3. 再讀 `ch02-01`, 才能理解 `libiui` 為什麼需要 Kconfig, generated headers, isolated test build, and CI matrix.
