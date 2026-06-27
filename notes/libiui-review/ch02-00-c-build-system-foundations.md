# Ch02-00: C Build System Foundations

## 本章問題

如果你以前只寫過:

```sh
gcc main.c -o main
```

那你其實只看過 C build 的最短路線. `libiui` 這種 repo 會讓你突然遇到:

```text
.o file
.a file
linker error
include path
library path
generated header
dependency file
Makefile target
```

這些不是額外雜訊. 它們都來自同一個基本問題:

```text
一個 C program 從 source code 變成可執行結果, 中間到底經過哪些階段?
```

> [Scope]
> 本章只講 GCC/C build 的共同基礎: preprocessing, compilation, assembling, linking, translation unit, header inclusion, object file, symbol, static library, dependency tracking, and Makefile basics. 下一篇 `ch02-01-libiui-build-flow.md` 才進入 `libiui` 的 Kconfig 和 repo-specific build design.

## Naive 想法: gcc 就是編譯器

你可能會先這樣想:

```text
gcc main.c -o app
  gcc 把 main.c 變成 app.
```

這句話作為初學口語可以, 但它把很多階段壓扁了. 比較精確的圖是:

```text
main.c
  |
  | preprocessing
  v
main.i
  |
  | compilation
  v
main.s
  |
  | assembling
  v
main.o
  |
  | linking
  v
app
```

GCC 常常像一個 driver. 你下 `gcc main.c -o app`, 它會幫你呼叫 preprocessor, compiler, assembler, linker. 所以你看到的是一條命令, 但裡面是多個 transformation.

## Stage 1: Preprocessing

preprocessing 處理的是 `#` 開頭的語言:

```c
#include <stdio.h>
#define N 3
#ifdef DEBUG
```

你可以用:

```sh
gcc -E main.c -o main.i
```

先看 preprocess 後的結果.

最重要的直覺是:

```text
#include 不是 link.
#include 比較像把 header 內容貼進目前的 .c file.
```

例如:

```text
main.c
  #include "foo.h"
  int main(void) { return foo(); }

foo.h
  int foo(void);
```

preprocess 後可以想成:

```text
int foo(void);
int main(void) { return foo(); }
```

所以 header 的角色通常是提供 declaration. 它讓 compiler 知道:

```text
有一個 function 叫 foo.
它回傳 int.
它不吃參數.
```

但 header 通常不提供 function body. `foo()` 的實作可能在 `foo.c`.

## Stage 2: Compilation

compilation 把 preprocess 後的 C code 變成 assembly:

```sh
gcc -S main.c -o main.s
```

這一層會做語法檢查, 型別檢查, optimization, 並產生目標 CPU 的 assembly. 對你目前讀 build system 來說, 不需要先懂每一行 assembly. 但要知道 compile error 通常發生在這一層:

```text
unknown type name
implicit declaration
missing header
syntax error
incompatible pointer type
```

這些錯誤代表 compiler 在處理某個 translation unit 時就已經不接受.

## Stage 3: Assembling

assembling 把 assembly 變成 object file:

```sh
gcc -c main.c -o main.o
```

`-c` 的意思是:

```text
compile and assemble only.
不要 link.
```

`main.o` 還不是 executable. 它是一個 object file, 裡面包含:

```text
machine code
symbols defined by this file
symbols referenced but not yet resolved
relocation information
debug information if enabled
```

用一個簡化圖看:

```text
main.c
  calls foo()
  defines main()
        |
        v
main.o
  defines: main
  needs:   foo
```

如果 `main.c` 呼叫 `foo()`, compiler 只需要知道 `foo()` 的 declaration. 它不需要在 compile `main.c` 時看到 `foo()` 的 body. 真正把 `foo` 接起來的是 linker.

## Translation Unit

C compiler 的基本編譯單位不是整個專案, 而是 translation unit.

```text
translation unit
  一個 .c file 加上它 include 進來的所有 header.
```

圖:

```text
main.c
  #include "foo.h"
  #include "bar.h"
        |
        v
translation unit for main.c
        |
        v
main.o
```

另一個檔案:

```text
foo.c
  #include "foo.h"
        |
        v
translation unit for foo.c
        |
        v
foo.o
```

這個觀念很重要. 因為一個 header 改了, 所有 include 它的 translation unit 都可能需要重編. 這就是 dependency tracking 的來源.

## Stage 4: Linking

linking 把多個 object files 合成 executable:

```sh
gcc main.o foo.o -o app
```

圖:

```text
main.o
  defines: main
  needs:   foo

foo.o
  defines: foo
  needs:   none

linker
  resolves main.o needs foo -> foo.o defines foo
        |
        v
app
```

常見 linker error:

```text
undefined reference to `foo`
```

意思不是 compiler 看不懂 `foo`. 它通常代表:

```text
compile 階段接受了 declaration.
link 階段找不到 definition.
```

所以這兩種錯誤要分開看:

```text
compile error
  某個 .c file 自己編不過.

link error
  每個 .c 可能都編過了, 但 object files 接不起來.
```

## Header 不是 Library

初學 C build 時很常混淆:

```text
#include <SDL2/SDL.h>
```

和:

```sh
-lSDL2
```

它們解決不同問題:

```text
header
  給 compiler 看 declaration.

library
  給 linker 找 definition.
```

圖:

```text
compile stage
  main.c + SDL.h
        |
        v
  main.o
  needs: SDL_Init

link stage
  main.o + libSDL2.a or SDL2.dll import library
        |
        v
  app
```

所以只裝 header 不夠, 只 link library 也不夠. 一般外部套件要同時提供:

```text
include path
  -I...

library path
  -L...

library name
  -l...
```

## `-I`, `-L`, and `-l`

這三個 flag 很容易一起出現:

```sh
gcc main.c -I/usr/include/SDL2 -L/usr/lib -lSDL2 -o app
```

它們分別是:

```text
-I/path
  compiler 搜尋 header 的路徑.

-L/path
  linker 搜尋 library 的路徑.

-lname
  linker 要找 libname.a or libname.so.
```

例如:

```text
-lSDL2
  may search for libSDL2.a
  may search for libSDL2.so
  platform dependent
```

這就是為什麼跨平台專案通常不會硬寫 SDL2 path. 它會用 `pkg-config` or `sdl2-config` 問系統:

```sh
pkg-config --cflags sdl2
pkg-config --libs sdl2
```

## Static Library: `.a`

如果每次 link 都手動列出很多 `.o`, 會很麻煩:

```sh
gcc app.o core.o layout.o draw.o input.o basic.o -o app
```

static library 的想法是先把一組 object files 打包:

```sh
ar rcs libiui.a core.o layout.o draw.o input.o basic.o
```

圖:

```text
core.o
layout.o
draw.o
input.o
basic.o
   |
   v
ar rcs
   |
   v
libiui.a
```

之後 executable 只需要 link:

```sh
gcc example.o libiui.a -o libiui_example
```

`libiui.a` 不是動態載入的 runtime plugin. 它比較像一包 object files, link 時 linker 會從裡面拿需要的 object code.

## 為什麼需要 Makefile

如果只有兩個檔案, 你可以手動打:

```sh
gcc -c main.c -o main.o
gcc -c foo.c -o foo.o
gcc main.o foo.o -o app
```

但一旦有很多檔案, 手動做會出現問題:

```text
你不知道哪些檔案需要重編.
你容易忘記某個 compiler flag.
你容易忘記某個 object file.
你不知道不同 target 該怎麼共用規則.
```

Makefile 的核心不是 "會編譯 C". 它的核心是描述 dependency graph:

```make
app: main.o foo.o
	$(CC) main.o foo.o -o app

main.o: main.c foo.h
	$(CC) -c main.c -o main.o

foo.o: foo.c foo.h
	$(CC) -c foo.c -o foo.o
```

圖:

```text
foo.h ----+--> main.o --+
main.c ---+             |
                        +--> app
foo.h ----+--> foo.o ---+
foo.c ----+
```

如果 `foo.h` 改了, `main.o` 和 `foo.o` 都可能要重編. 如果只有 `main.c` 改了, `foo.o` 不需要重編.

## Dependency File

手動寫所有 header dependency 很痛苦. GCC 可以幫忙產生 `.d` file:

```sh
gcc -MMD -MP -MF .build/main.d -c main.c -o .build/main.o
```

簡化後的 `.d` 可能長這樣:

```text
.build/main.o: main.c foo.h config.h
```

這就是 compiler 幫 Makefile 回答:

```text
這個 object file 實際上依賴哪些 source and header?
```

在 `libiui` 的 `mk/common.mk`, 你會看到:

```make
DEPFLAGS = -MMD -MP -MF $(BUILD_DIR)/$(notdir $*).d
```

現在你可以把它讀成:

```text
編譯時順便產生 header dependency map.
讓 make 知道 header 改動後該重編哪些 .o.
```

## Generated Header

一般 header 是人手寫的. 但有些 header 是 script 產生的:

```text
source of truth
  configs/Kconfig
        |
        v
generated header
  src/iui_config.h
```

generated header 的問題是:

```text
它不是 primary source.
它不能手改後當成正式修改.
它改了也會影響 include 它的 translation units.
```

所以 build system 需要知道:

```text
先產生 generated header.
再 compile include 它的 .c files.
generated header 改了, 相關 object files 要重編.
```

這就是為什麼 build graph 不只有 `.c -> .o`. 它還包含 generator step.

## 從錯誤訊息判斷壞在哪一層

看到 build failed 時, 先分類:

```text
preprocessing error
  找不到 header, macro 條件錯, generated header 不存在.

compile error
  C 語法, type, declaration, struct member, function signature 不符合.

link error
  undefined reference, duplicate symbol, missing library.

runtime/test error
  build 成功, 但執行結果不符合預期.
```

ASCII map:

```text
.c + .h
  |
  | preprocess
  v
.i
  |
  | compile
  v
.s
  |
  | assemble
  v
.o
  |
  | link with other .o and libraries
  v
executable
```

如果錯在上游階段, 下游就不會發生. 所以不要直接跳到 Makefile 猜測. 先問:

```text
錯誤是在找 header?
還是在 compile 某個 .c?
還是在 link symbols?
還是在跑 test?
```

## 和 libiui 的連接

現在回到 `libiui`, 你會比較容易看懂下一篇:

```text
libiui.a
  static library, 由多個 .o 打包.

libiui_example
  executable, link tests/example.o + libiui.a + backend libraries.

src/iui_config.h
  generated header, 給 C preprocessor 使用.

.config
  Makefile include 的 config, 用來決定 source selection and flags.

mk/common.mk
  generic compile/archive/link rules.

mk/deps.mk
  問系統 SDL2 的 -I, -L, -l.

mk/test.mk
  建立 isolated test build graph.
```

## 本章學到的三件事

> [Summary]
> 1. `gcc main.c -o app` 其實包住 preprocessing, compilation, assembling, and linking.
> 2. `.h` 給 compiler 看 declaration, `.o` 裝 compiled code and symbols, `.a` 是 object files 的 archive, linker 負責把 symbols 接起來.
> 3. Makefile 的核心是 dependency graph. 它管理哪些檔案要先產生, 哪些 object 要重編, 哪些 target 要 link.

## 下一章

接著讀 [ch02-01-libiui-build-flow.md](ch02-01-libiui-build-flow.md). 那篇會把本章的通用模型套回 `libiui`:

```text
C build primitives
  -> Makefile target graph
  -> Kconfig source selection
  -> generated config header
  -> backend source selection
```

再接著讀 [ch02-02-libiui-build-variants-and-validation.md](ch02-02-libiui-build-variants-and-validation.md), 看 generated files, demo target, `make check`, CI matrix, WASM flow, and build failure diagnosis.
