# Ch00: What Is libiui

## 本章問題

你現在遇到的卡點很合理: 如果一開始就看 `Makefile`, `.config`, Kconfig, CI matrix, 會像是在拆一台機器的螺絲, 但還不知道這台機器原本要做什麼.

所以這章先不碰 build system 的細節. 我們先回答一個更基本的問題:

```text
libiui 是什麼?
它為什麼需要 immediate-mode?
它把 UI 問題拆成哪些階段?
這些階段在 repo 裡大概對應哪些目錄?
```

> [Scope]
> 這一章建立專案的 mental model. 目標不是熟悉每個 API, 而是讓你知道接下來讀 `include/`, `src/`, `ports/`, `tests/`, `Makefile` 時, 每個東西大概站在哪個位置.

## 先從最基本問題開始

一個 UI library 最基本要解決的是:

```text
input
  使用者現在做了什麼?

state
  這個 frame 裡哪些 widget 被 hover, pressed, focused, edited?

layout
  每個 widget 應該放在哪裡?

rendering
  要畫哪些 rectangle, text, line, circle, arc?

backend
  最後怎麼把 draw commands 送到 SDL2, browser canvas, framebuffer, or game engine?
```

如果用 graphics pipeline 直覺來看, `libiui` 不是 OpenGL/Vulkan 那種 GPU API, 而比較像 CPU-side UI frontend:

```text
Application code
  calls iui_button(), iui_slider(), iui_checkbox()
        |
        v
libiui core
  resolves input, layout, widget state, MD3 styling
        |
        v
renderer callbacks
  draw_box(), draw_text(), draw_line(), draw_circle(), set_clip_rect()
        |
        v
backend
  SDL2 / WASM Canvas / headless / custom target
```

換句話說, 它不是在管理 GPU resource, shader, command buffer. 它是在每一 frame 內, 把 UI 呼叫轉成平台無關的繪圖需求.

## naive 想法: UI library 就是畫按鈕

如果只用表面直覺看, 可能會以為:

```text
iui_button(ctx, "OK", ...)
  -> 畫一個矩形
  -> 畫文字
  -> 如果 mouse 在裡面就 return true
```

這個想法只對最小 toy UI 成立. 真正的 UI library 還要處理:

```text
same widget across frames
  這一 frame 是 hovered, pressed, focused, or activated?

layout cursor
  一個 widget 畫完, 下一個 widget 要放在哪裡?

window and clipping
  widget 超出 window 或 scroll viewport 時不能亂畫.

focus and keyboard
  mouse 之外還要支援 Tab, Enter, text input.

modal and overlay
  menu, dialog, dropdown 開啟時, 背後的 input 要被擋住.

backend independence
  library 不能假設底層一定是 SDL2 或 OpenGL.
```

這也是為什麼 repo 看起來不只是 `button.c` 和 `draw.c`. 它其實是在實作一個小型 UI runtime.

## retained-mode 和 immediate-mode 的差異

大多數傳統 GUI framework 比較接近 retained-mode:

```text
create Button object
  -> store it in widget tree
  -> framework owns tree
  -> event system later dispatches callback
```

ASCII 圖:

```text
Retained-mode

Application creates objects
        |
        v
Widget tree persists across time
        |
        v
Framework traverses tree for layout, input, render

Frame 1:  [Window [Button] [Slider]]
Frame 2:  [Window [Button] [Slider]]
Frame 3:  [Window [Button] [Slider]]
```

Immediate-mode 的想法不同:

```text
Every frame, application describes the UI again.
The widget exists as a function call in that frame.
Persistent data is either in user variables or compact internal state.
```

ASCII 圖:

```text
Immediate-mode

Frame 1:
  begin_frame()
  button("Apply")
  slider("Volume")
  end_frame()

Frame 2:
  begin_frame()
  button("Apply")
  slider("Volume")
  end_frame()
```

所以 `libiui` 的使用方式會像 README 裡的範例:

```c
iui_begin_frame(ctx, dt);
iui_begin_window(ctx, "Settings", 100, 100, 400, 300, 0);

if (iui_button(ctx, "Apply", IUI_ALIGN_LEFT))
    apply_settings();

iui_end_window(ctx);
iui_end_frame(ctx);
```

這裡沒有 `Button *button = new Button(...)`. `button` 是一個 frame 內的 function call. 這種模式對 resource-constrained environment 有吸引力, 因為它可以避開複雜 widget tree, heap allocation, object lifetime management.

## libiui 的核心定位

根據 `README.md`, 這個 project 的定位可以整理成:

```text
pure C99 immediate-mode UI library
  用 C 實作, 不依賴 C++ object system.

Material Design 3 implementation
  不是隨便畫幾個 widget, 而是把 MD3 dimensions, colors, state layers, motion tokens 寫進 spec.

zero heap allocation
  application 提供固定 buffer, library 在裡面建立 context.

renderer callback abstraction
  library 不直接碰 SDL2, OpenGL, Vulkan, DirectX.

configurable feature set
  透過 Kconfig 選 modules, backend, optional features.
```

這些點合在一起, 代表 `libiui` 的目標不是成為 Qt/GTK/Electron 替代品, 而是:

```text
在 embedded, game engine, lightweight desktop, browser/WASM 等環境,
提供一個 footprint 小, deterministic, backend-independent 的 MD3 UI frontend.
```

## 從 frame flow 看整份 repo

先把 runtime flow 畫出來:

```text
backend polls input
        |
        v
iui_update_mouse_pos()
iui_update_mouse_buttons()
iui_update_key()
iui_update_char()
        |
        v
iui_begin_frame()
        |
        v
application calls widgets
  iui_begin_window()
  iui_button()
  iui_slider()
  iui_menu_begin()
        |
        v
libiui computes:
  IDs
  layout rectangles
  hover/active/focus state
  MD3 colors and dimensions
  draw commands
        |
        v
iui_end_frame()
        |
        v
renderer callbacks draw pixels
```

這張圖可以對應到 repo:

```text
include/
  public API. 使用者看到的 interface.

src/
  UI runtime and widgets. input, layout, drawing, MD3 components.

ports/
  platform/backend boundary. SDL2, WASM, headless, and common port interface.

tests/
  API behavior, widget behavior, spec validation, demo programs.

configs/ + mk/ + Makefile
  build-time feature selection and compilation graph.
```

所以 build system 不是第一個概念, 它是第二層問題:

```text
先知道有哪些 runtime pieces.
再問 build system 如何選擇 pieces.
```

更具體地說, 一個 frame 可以想成這樣跑:

```text
Time ------------------------------------------------------------>

[backend event queue]
  mouse move, mouse down, key, text, scroll
          |
          v
[input update]
  iui_update_mouse_pos()
  iui_update_mouse_buttons()
  iui_update_key()
  iui_update_char()
          |
          v
[begin frame]
  iui_begin_frame(ctx, dt)
  reset per-frame state
  keep persistent interaction state
          |
          v
[application describes UI]
  iui_begin_window("Settings")
    iui_button("Apply")
    iui_slider("Volume")
    iui_checkbox("Enabled")
  iui_end_window()
          |
          v
[libiui resolves each widget]
  widget id
  layout rect
  input hit test
  hover / active / focus
  MD3 state layer
  draw commands
          |
          v
[end frame]
  iui_end_frame(ctx)
  finalize tracking
  flush batched draw commands if enabled
          |
          v
[backend presents pixels]
  SDL_RenderPresent()
  Canvas draw
  headless framebuffer
```

把這張圖想成 software renderer 的 frontend 也可以:

```text
UI call stream
  像 command stream, 但 command 是 widget calls.

layout and state resolution
  像 CPU-side primitive setup, 決定每個 widget 的 screen-space rect.

draw helpers
  像 rasterization 前的 draw command generation.

renderer callbacks
  像 backend command emission, 但目標可以是 SDL2, WASM, framebuffer.
```

再用一個 `iui_button()` 看局部流程:

```text
application
  if (iui_button(ctx, "Apply", IUI_ALIGN_LEFT))
      apply_settings();
        |
        v
libiui
  allocate button rect from current layout cursor
  expand hit region to MD3 touch target
  compare mouse position with rect
  compare mouse and keyboard state with current interaction rule
  choose visual state: default / hovered / pressed / focused
  draw rounded box, text, state layer, focus ring
  return true when this repo defines the button as activated
        |
        v
application
  runs apply_settings() only when interaction is submitted
```

這裡最重要的觀念是:

```text
Widget function is not just drawing.
Widget function is input + identity + layout + state + drawing + result.
```

## `include/` 是使用者看到的世界

`include/iui.h` 是 public API 的主要入口. 它暴露幾類東西:

```text
context lifecycle
  iui_min_memory_size()
  iui_make_config()
  iui_init()

frame lifecycle
  iui_begin_frame()
  iui_end_frame()

input update
  iui_update_mouse_pos()
  iui_update_mouse_buttons()
  iui_update_key()
  iui_update_char()

containers and layout
  iui_begin_window()
  iui_end_window()
  iui_flex()
  iui_grid_begin()

widgets
  iui_button()
  iui_slider()
  iui_checkbox()
  iui_tabs()
  iui_menu_begin()

drawing
  iui_draw_line()
  iui_draw_circle()
  iui_draw_arc()
```

`include/iui-spec.h` 則比較像 MD3 token table:

```text
button height
slider track height
FAB size
tab height
state layer alpha
WCAG contrast threshold
layout breakpoints
```

這是 `libiui` 和一般 toy immediate-mode UI 的不同點之一: 它不是只做互動, 它還試圖把 Material Design 3 的尺寸與語義固定成可測試的規格.

## `src/` 是 UI runtime

`src/` 可以先粗略分成幾種角色:

```text
core runtime
  context, frame lifecycle, input state, focus, modal, animation.

layout
  window, box, flex, grid, spacing, clipping.

draw
  rectangle, text, vector primitives, shadows, state layers.

widgets
  button, slider, textfield, checkbox, menu, dialog, tabs, navigation.

spec and validation
  MD3 constants, generated validation tables, test support.
```

更精準的 source map 要留到下一章, 因為那需要逐檔讀 `src/*.c`. 這章先建立一個粗模型:

```text
Widget function
  = input query
  + layout allocation
  + state transition
  + visual style
  + draw calls
  + return value
```

例如 `iui_button()` 不只是畫按鈕. 它需要知道:

```text
Where is the button?
Is mouse inside?
Was it pressed this frame?
Was it activated by mouse press or focused Enter?
Which MD3 state layer should be drawn?
Should keyboard focus affect it?
What should it return to application code?
```

## `ports/` 是 backend boundary

`libiui` 不應該直接依賴某個 graphics API. 如果它直接呼叫 SDL2 或 OpenGL, 就很難拿去 embedded framebuffer, WASM, or game engine.

所以 public API 有 `iui_renderer_t`:

```text
draw_box
draw_text
set_clip_rect
text_width
draw_line
draw_circle
draw_arc
```

這些是 library 需要的最小繪圖能力. Backend 的責任是把這些 callback 接到底層平台.

`ports/port.h` 又往上包了一層 application/backend interface:

```text
poll_events()
get_input()
begin_frame()
end_frame()
get_renderer_callbacks()
get_window_size()
get_dpi_scale()
clipboard operations
```

你可以把它想成:

```text
libiui core
  platform-independent UI logic

port backend
  platform-specific window, input, presentation, clipboard
```

這個分界之後看 Windows build 或 SDL2 問題會很重要. 因為不是所有問題都屬於 UI core. 有些問題是在 backend, 有些是在 build detection, 有些是在 platform C library 差異.

## `tests/` 不只是測試資料夾

在這個 repo 裡, `tests/` 同時包含:

```text
unit tests
  test-button, test-slider, test-layout, test-focus, etc.

spec validation
  test-spec.c, generated MD3 validation cases.

demo application
  tests/example.c is used to build libiui_example.

benchmark or visual examples
  bench-render.c, pixelwall-demo.c, forthsalon-demo.c.
```

所以之後不要把 `tests/` 只理解成 "test code". 它也是理解 API usage 的入口.

## 為什麼 build system 之後才看

現在可以回頭理解為什麼 `ch02-build-flow.md` 需要存在.

如果 repo 是單一固定 binary, build flow 可能很簡單:

```text
gcc src/*.c -o app
```

但 `libiui` 不是固定形狀. 它會依照 config 選:

```text
which modules?
  basic, input, navigation, overlay, data, etc.

which backend?
  SDL2, WASM, headless, custom.

which optional features?
  animation, accessibility, vector drawing.

which generated files?
  MD3 validation data, nyancat data, config header.
```

所以 build system 是在回答:

```text
這一次要把哪一個 libiui 變體編出來?
```

Kconfig 的直覺可以先這樣記:

```text
Kconfig
  問題清單. 例如要不要 SDL2? 要不要 animation?

.config
  給 Makefile 看的答案表.

src/iui_config.h
  給 C preprocessor 看的答案表.

Makefile
  根據答案表決定要編哪些 .c files and flags.
```

因此 build flow 不是不重要, 而是不應該作為第一個入口. 先知道 `libiui` 的 runtime pieces, build flow 才有意義.

## 建議閱讀順序

接下來 repo review 可以改成:

```text
ch00-what-is-libiui.md
  先建立專案定位, frame flow, source tree 大方向.

ch01-source-map.md
  逐步整理 include/, src/, ports/, tests/ 的責任分工.

ch02-build-flow.md
  Build flow 入口, 先導向 GCC/C build 基礎, 再導向 libiui-specific build.

ch02-00-c-build-system-foundations.md
  先理解 preprocessing, compilation, object files, linking, static libraries, dependency files.

ch02-01-libiui-build-flow.md
  再回來理解 Makefile, Kconfig, .config, iui_config.h.

ch03-00-software-testing-foundations.md
  先補開發端測試基礎: unit/integration/E2E, test double, control, observability.

ch03-01-gui-test-and-validation-model.md
  再理解 GUI 測試怎麼控制 frame, input, renderer output, headless backend, and validation evidence.

ch04-runtime-model.md
  深入 input -> layout -> widget state -> draw.
```

也就是:

```text
what is it?
  -> where is each part?
  -> how is it built?
  -> how is it validated?
  -> how does it run?
```

## 本章學到的三件事

> [Summary]
> 1. `libiui` 是 pure C99 immediate-mode MD3 UI library, 目標是 small footprint, deterministic memory, backend independence.
> 2. 整體資料流可以先看成 input -> frame lifecycle -> widgets -> layout/state/draw -> renderer callbacks -> backend.
> 3. Build system 應該放在第二層理解: 先知道 repo pieces, 再問 Kconfig/Makefile 如何選擇要編進去的 pieces.

## 下一章

下一章讀 [ch01-source-map.md](ch01-source-map.md), 目標是把這章的粗略目錄理解變成更具體的檔案地圖:

```text
include/iui.h
  public API surface.

src/internal.h
  private runtime state.

src/*.c
  modules and widgets.

ports/*.c
  backend implementations.

tests/*.c
  behavior examples and validation.
```

## Source Files Read

```text
README.md
include/iui.h
include/iui-spec.h
src/core.c
src/internal.h
ports/port.h
```
