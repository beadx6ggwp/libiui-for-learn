# ch05 - Public API Tour

## 本章要解決的問題

讀完 [ch04-runtime-model.md](ch04-runtime-model.md) 之後, 我們已經追過 `iui_button()` 的 runtime path:

```text
input edge
  -> begin_frame
  -> begin_window
  -> layout rect
  -> widget id
  -> interaction state
  -> draw callbacks
  -> end_frame
```

但這只是從 implementation 往下追一個 widget. 接下來要反過來問:

```text
如果我是 libiui 的使用者或 backend 作者, include/iui.h 要我理解哪些概念?
```

這一章不把 `include/iui.h` 當成函式清單背誦. 目標是把 public API 分成幾層, 讓之後寫 minimal app lab, headless test lab, 或找 PR patch 時, 知道每一個函式大概站在哪個邊界上.

## 先不要從 Header 第一行讀到最後一行

naive 做法是:

```text
打開 include/iui.h
  -> 從 typedef 開始往下讀
  -> 遇到 widget API 就記名字
  -> 遇到 callback 就先跳過
```

這樣會很快迷路, 因為 `include/iui.h` 同時包含:

```text
compile-time limits
public structs and enums
renderer backend callbacks
context initialization
input injection
frame lifecycle
layout helpers
widgets
modal and input layers
focus and IME
theme and motion
accessibility helpers
performance helpers
```

它不是只給 application developer 看, 也不是只給 port/backend developer 看. 它是一份 public contract, 同時描述:

```text
application 怎麼把 UI call stream 送進 libiui
backend 怎麼把 platform input 送進 libiui
renderer 怎麼把 libiui 的 draw request 變成 pixels
widget state 哪些由 user 持有, 哪些由 iui_context 持有
```

所以比較好的讀法是先建立分層.

## Public API 的大圖

可以先把 `include/iui.h` 看成這張圖:

```text
        application code
   values, strings, widget state
             |
             v
  +-----------------------+
  |       include/iui.h    |
  | public API contract    |
  +-----------------------+
     |        |        |
     |        |        +------------------+
     |        |                           |
     v        v                           v
 iui_context frame/layout/widgets    renderer callbacks
 runtime UI state                    draw_box/draw_text/clip
     |                                      |
     v                                      v
    src/                             ports/sdl2.c
 core implementation                 ports/headless.c
                                     ports/wasm.c
```

對使用者來說, `include/iui.h` 的核心不是 "有哪些元件", 而是:

```text
誰持有 state?
誰負責餵 input?
誰負責產生每 frame 的 UI?
誰負責真正畫到畫面?
```

這四個問題比背 API 名稱重要.

## Layer 0: Compile-Time Limits and MD3 Spec

最前面一層不是 runtime API, 而是 compile-time configuration:

```c
#ifndef IUI_MAX_WINDOWS
#define IUI_MAX_WINDOWS 16
#endif

#ifndef IUI_ID_STACK_SIZE
#define IUI_ID_STACK_SIZE 8
#endif

#ifndef IUI_MAX_FOCUSABLE_WIDGETS
#define IUI_MAX_FOCUSABLE_WIDGETS 64
#endif
```

這些 macro 的意思是:

```text
libiui 傾向固定容量設計
  -> context 內部資料結構有上限
  -> 上限可在 include header 前 override
  -> 適合小型 UI library, test, embedded-like setting
```

`include/iui-spec.h` 是另一種 compile-time API. 它放的是 Material Design 3 的尺寸, duration, state alpha, elevation, accessibility constants:

```text
IUI_DURATION_SHORT_1
IUI_SLIDER_TRACK_HEIGHT
IUI_FAB_SIZE
IUI_ICON_BUTTON_TOUCH_TARGET
IUI_FOCUS_RING_WIDTH
IUI_WCAG_AA_NORMAL_TEXT
```

這表示 spec token 不是執行時從 JSON 或 theme file 載入, 而是 C preprocessor 階段就決定:

```text
application
  #define IUI_FAB_SIZE 64.f
  #include "iui.h"

compiler
  sees overridden token

libiui code
  compiles against that token
```

從 first principles 看, 這樣做的取捨是:

```text
好處:
  simple C API
  no parser
  no runtime allocation for spec tables
  easy for small static builds

代價:
  change token usually means rebuild
  not suited for runtime theme editor by default
  compile order matters
```

所以 `iui-spec.h` 也算 public API, 只是它的使用時機在 compile time, 不在 frame loop 裡.

## Layer 1: Context, Memory, and Backend Contract

public API 裡最重要的 opaque type 是:

```c
typedef struct iui_context iui_context;
```

opaque 的意思是 application 看不到 `struct iui_context` 內部欄位. 使用者只能透過 public function 操作它:

```text
application
  can hold iui_context*
  cannot access ctx->layout directly
  cannot mutate ctx->input directly
```

這讓 library 可以保留 implementation freedom. 但 `libiui` 又不是完全把記憶體交給自己處理. 初始化路線是:

```c
size_t iui_min_memory_size(void);
iui_config_t iui_make_config(void *buffer,
                             iui_renderer_t renderer,
                             float font_height,
                             const iui_vector_t *vector);
iui_context *iui_init(const iui_config_t *config);
```

這裡的 mental model 是:

```text
application owns memory buffer
  -> iui_make_config connects buffer and renderer callbacks
  -> iui_init places runtime context into that buffer
  -> application keeps iui_context*
```

圖:

```text
malloc or static buffer
        |
        v
  iui_config_t
  buffer + renderer + font_height + vector
        |
        v
     iui_init()
        |
        v
    iui_context*
```

`iui_renderer_t` 是 backend boundary:

```c
typedef struct {
    void (*draw_box)(iui_rect_t rect,
                     float radius,
                     uint32_t srgb_color,
                     void *user);
    void (*draw_text)(float x,
                      float y,
                      const char *text,
                      uint32_t srgb_color,
                      void *user);
    void (*set_clip_rect)(uint16_t min_x,
                          uint16_t min_y,
                          uint16_t max_x,
                          uint16_t max_y,
                          void *user);
    float (*text_width)(const char *text, void *user);
    ...
    void *user;
} iui_renderer_t;
```

這不是 widget API, 而是 draw backend API. `src/` 不應該知道 SDL2 window, browser canvas, or headless framebuffer 的細節. 它只知道:

```text
draw a rounded box
draw text
set clip rectangle
measure text width
maybe draw vector line/circle/arc
```

backend 則把這些 callback 實作成:

```text
ports/sdl2.c
  draw to SDL surface/window

ports/wasm.c
  draw to web canvas / wasm target

ports/headless.c
  draw to software framebuffer for tests
```

這是後面讀 backend boundary 的核心.

## Layer 2: Input Injection

GUI library 不能自己知道 OS 發生什麼事. backend 必須把 platform event 轉成 libiui 的 input state:

```c
void iui_update_mouse_pos(iui_context *ctx, float x, float y);
void iui_update_mouse_buttons(iui_context *ctx,
                              uint8_t pressed,
                              uint8_t released,
                              uint8_t held);
void iui_update_scroll(iui_context *ctx, float dx, float dy);
void iui_update_key(iui_context *ctx, int key);
void iui_update_char(iui_context *ctx, int codepoint);
void iui_update_modifiers(iui_context *ctx, uint8_t modifiers);
```

資料流是:

```text
OS event
  -> SDL2 event / browser event / headless injected event
  -> iui_update_*
  -> ctx input state
  -> widgets read input during frame
```

這裡有一個重要觀念: input update 通常發生在 widget calls 之前. 如果順序反過來, widget 這一 frame 看到的 input 就會落後或錯位.

```text
correct frame
  poll events
  update input
  begin_frame
  call widgets
  end_frame

bad frame
  begin_frame
  call widgets
  update input
  end_frame
```

在 immediate-mode UI 裡, input timing 很重要. 因為 widget 不是常駐物件, 它每 frame 重新執行一次 function call, 只能根據當下 `ctx` state 做判斷.

## Layer 3: Frame and Window Lifecycle

frame lifecycle 是 public API 的主幹:

```c
void iui_begin_frame(iui_context *ctx, float delta_time);

bool iui_begin_window(iui_context *ctx,
                      const char *name,
                      float x,
                      float y,
                      float width,
                      float height,
                      uint32_t options);

void iui_end_window(iui_context *ctx);
void iui_end_frame(iui_context *ctx);
```

一個最小流程長這樣:

```text
while app is running:
  poll platform events
  iui_update_*(ctx, ...)

  iui_begin_frame(ctx, dt)

  if iui_begin_window(ctx, "Main", 20, 20, 400, 300, 0):
      iui_text(ctx, IUI_ALIGN_LEFT, "Hello")
      if iui_button(ctx, "Run", IUI_ALIGN_LEFT):
          do_action()
      iui_end_window(ctx)

  iui_end_frame(ctx)
```

這裡的第一性原理是:

```text
UI 每 frame 都要被重新描述
  -> begin_frame 清理/準備 per-frame 狀態
  -> begin_window 建立目前 layout and clip context
  -> widget calls 依序消耗 layout
  -> end_window 收尾 window scope
  -> end_frame 做 cleanup and finalization
```

如果少了 begin/end pairing, library 就不知道目前 widget 應該屬於哪個 window, clip stack 應該如何還原, per-frame state 何時清掉.

## Layer 4: Layout and Identity

public layout API 有幾類:

```text
simple flow:
  iui_newline()
  iui_get_layout_rect()
  iui_get_window_rect()
  iui_get_remaining_height()

box layout:
  iui_box_begin()
  iui_box_next()
  iui_box_end()

grid layout:
  iui_grid_begin()
  iui_grid_next()
  iui_grid_end()

adaptive layout:
  iui_size_class()
  iui_layout_columns()
  iui_layout_margin()
  iui_layout_gutter()
```

從 ch04 可知, widget 需要 layout rect 才能做三件事:

```text
hit testing
  mouse 是否在 touch target 內?

draw placement
  box/text/icon 要畫在哪裡?

identity
  label + position + id stack 形成 widget id
```

identity 相關 API 是:

```c
void iui_push_id(iui_context *ctx, const char *id);
void iui_pop_id(iui_context *ctx);
```

這代表 public API 不只管理畫面排版, 也管理 widget identity namespace.

圖:

```text
window layout cursor
        |
        v
   widget asks for rect
        |
        +--> hit test
        +--> draw calls
        +--> id = label + rect + id stack
```

如果同一個 window 裡有重複 label, 或 dynamic list 裡 position 會變, 就要開始思考 `iui_push_id()` 這類 API. 這跟 React key, ImGui ID stack 是同一類問題: UI call stream 每 frame 重建, 所以要有穩定 identity.

## Layer 5: Widgets and State Ownership

`include/iui.h` 最長的部分是 widgets. 但 widgets 可以分成幾種呼叫型態.

第一種是 pure display:

```c
void iui_text(iui_context *ctx,
              iui_text_alignment_t alignment,
              const char *string,
              ...);

void iui_divider(iui_context *ctx);
```

它們主要消耗 layout and emit draw calls, 不需要 user-owned state.

第二種是 simple interaction:

```c
bool iui_button(iui_context *ctx,
                const char *label,
                iui_text_alignment_t alignment);
```

這種 API 回傳 "這一 frame 發生了什麼". 使用者不保存 button object, 而是在 frame loop 裡處理 return value.

第三種是 value pointer:

```c
bool iui_checkbox(iui_context *ctx, const char *label, bool *checked);
bool iui_radio(iui_context *ctx,
               const char *label,
               int *group_value,
               int button_value);
void iui_slider(iui_context *ctx,
                const char *label,
                float min_value,
                float max_value,
                float step,
                float *value,
                const char *fmt);
```

這種 API 說明一個 immediate-mode UI 的關鍵: application state 仍然在 application 端.

```text
application owns checked/value/group_value
  -> widget reads current value
  -> widget may update value this frame
  -> application keeps value for next frame
```

第四種是 user-provided state struct:

```text
iui_edit_state
iui_scroll_state
iui_menu_state
iui_dialog_state
iui_date_picker_state
iui_time_picker_state
iui_search_view_state
iui_nav_drawer_state
iui_side_sheet_state
iui_carousel_state
```

這些不是 "違反 immediate-mode". 它們是在承認某些 widget 的互動狀態太複雜, 不適合只靠 bool return 或 value pointer 表達.

可以這樣分:

```text
library-owned state:
  input edge
  hover/active/focus
  window and layout stacks
  clip and modal layers

user-owned state:
  checkbox checked
  slider value
  text cursor and selection
  scroll offset
  menu open position
  dialog selected button

backend-owned state:
  renderer user pointer
  platform window
  framebuffer or canvas
  clipboard integration
```

圖:

```text
                 +------------------+
                 |   application    |
                 | values + states  |
                 +--------+---------+
                          |
                          v
frame calls: iui_button / iui_slider / iui_menu / ...
                          |
                          v
                 +------------------+
                 |   iui_context    |
                 | runtime UI state |
                 +--------+---------+
                          |
                          v
                 renderer callbacks
```

這個 state split 是讀 libiui widget API 時最重要的概念.

## Layer 6: Containers, Overlays, and Input Blocking

有些 API 不是單一 widget, 而是會改變後續 widget 的 context:

```text
iui_card_begin() / iui_card_end()
iui_scroll_begin() / iui_scroll_end()
iui_begin_modal() / iui_end_modal()
iui_push_layer() / iui_pop_layer()
iui_push_clip() / iui_pop_clip()
```

這類 API 的共同點是:

```text
begin/push
  save current context
  install a new local context
  affect following widget calls

end/pop
  restore previous context
```

圖:

```text
window
  button A
  push_clip(scroll rect)
    scroll content
    button B
  pop_clip()
  button C
```

如果 `button B` 被 scroll clip 裁切, 或 modal 開啟後 background button 不能點, 這些行為就不是 button 自己單獨決定的. 它們來自 context 裡的 clip stack, modal state, and input layer system.

所以 debug overlay 類問題時, 不能只看 widget implementation. 要看:

```text
current clip stack
active modal
input layer depth
blocking regions
focused layer
```

public API 中這些函式就是為這個 boundary 暴露出來:

```c
bool iui_push_clip(iui_context *ctx, iui_rect_t rect);
void iui_pop_clip(iui_context *ctx);

void iui_begin_modal(iui_context *ctx, const char *id);
void iui_end_modal(iui_context *ctx);

int iui_push_layer(iui_context *ctx, int z_order);
void iui_pop_layer(iui_context *ctx);
bool iui_should_process_input(iui_context *ctx, iui_rect_t bounds);
```

## Layer 7: Focus, Text Input, IME, and Clipboard

Text input 比 button 複雜, 因為它涉及:

```text
keyboard focus
cursor and selection
composition text
copy and paste
platform clipboard
```

public API 因此有一組 focus and text infrastructure:

```text
iui_set_focus()
iui_has_focus()
iui_clear_focus()
iui_focus_next()
iui_focus_prev()
iui_get_focused_id()
iui_has_any_focus()
```

IME and clipboard:

```text
iui_update_composition()
iui_commit_composition()
iui_ime_is_composing()
iui_ime_get_text()
iui_set_clipboard_callbacks()
iui_clipboard_copy()
iui_clipboard_paste()
```

這裡的 boundary 很清楚:

```text
platform owns native IME and clipboard events
  -> backend translates them into iui calls
  -> libiui text widgets consume composition/clipboard state
  -> application still owns the text buffer
```

這也是為什麼 `iui_clipboard_t` 裡有 `void *user`. C API 常用這個 pattern 把 backend-specific context 帶進 callback:

```text
callback function pointer + void *user
  = object-like behavior in C
```

## Layer 8: Theme, Motion, and Material Design 3

`libiui` 不是純 layout library, 它也帶了 Material Design 3 的 visual vocabulary:

```text
iui_theme_t
iui_theme_light()
iui_theme_dark()
iui_set_theme()

iui_easing_t
iui_motion_config
iui_ease()
iui_motion_apply()
iui_motion_progress()

iui_get_component_state()
iui_get_state_color()
iui_draw_shadow()
iui_draw_elevated_box()
```

這些 API 可以理解成兩層:

```text
semantic layer:
  primary, surface, outline, error, shadow
  focused, hovered, pressed, disabled

render helper layer:
  convert semantic state into color/elevation/motion
  emit draw calls through renderer callbacks
```

對 graphics / renderer 背景來說, 可以把它想成 fixed UI shading model:

```text
component state + theme token + spec token
  -> concrete color, radius, alpha, duration
  -> draw_box / draw_text / draw_line
```

這跟 GPU pipeline 的概念不是一樣的技術層, 但有類似的資料轉換味道: semantic input 最後必須變成 concrete draw primitives.

## Layer 9: Accessibility and Performance Helpers

`include/iui.h` 後面還有 accessibility and performance surface:

```text
WCAG contrast helpers
iui_a11y_* callbacks and node metadata

iui_batch_enable()
iui_dirty_*
iui_ink_bounds_*
iui_text_cache_*
```

第一輪 repo review 不需要立刻深入這一層, 但要知道它們存在. 原因是 upstream-facing patch 有時候不是 widget 外觀錯誤, 而是:

```text
contrast ratio 不足
focus metadata 缺失
text measurement cache invalidation
dirty region 判斷錯
batching 造成 draw order 問題
```

如果後面要做比較成熟的 PR, 這些 helper 會是判斷 "這個 bug 屬於哪一層" 的線索.

## 從使用者角度整理 API Reading Order

如果現在要重新讀 `include/iui.h`, 建議順序不是 file order, 而是:

```text
1. context and config
   iui_context, iui_config_t, iui_renderer_t, iui_init()

2. frame lifecycle
   iui_begin_frame(), iui_begin_window(), iui_end_window(), iui_end_frame()

3. input injection
   iui_update_mouse_*, iui_update_key(), iui_update_char()

4. simple widgets
   iui_text(), iui_button(), iui_checkbox(), iui_slider()

5. layout and identity
   iui_newline(), box, grid, iui_push_id(), iui_pop_id()

6. stateful components
   textfield, scroll, menu, dialog, date/time picker, search, navigation

7. overlay and input control
   modal, layer, clip, focus, input capture

8. style systems
   theme, motion, state color, elevation, iui-spec.h

9. advanced integration
   IME, clipboard, accessibility, performance
```

這個順序符合學習路線:

```text
先能跑一個 minimal app
  -> 再理解 input and layout
  -> 再理解 stateful widgets
  -> 再理解 overlay/focus
  -> 最後才看 a11y/performance
```

## 和 ch04 的連接

ch04 用 `iui_button()` 追的是 implementation path:

```text
iui_button()
  -> iui_button_styled()
  -> layout
  -> widget id
  -> input hit test
  -> draw callbacks
```

本章從 public API 看, 同一件事可以反過來說:

```text
application owns frame loop
  -> frame loop calls iui_button()
  -> iui_button consumes current ctx input/layout state
  -> returns one-frame behavior signal
  -> emits draw callbacks through configured renderer
```

這兩個方向要合起來看:

```text
public API view
  what contract does user see?

implementation view
  how does src/ satisfy that contract?
```

之後做 PR 時, 也要用這個方式拆問題:

```text
bug in public contract?
  maybe include/iui.h docs or API behavior issue

bug in implementation?
  maybe src/basic.c, src/layout.c, src/event.c

bug in backend?
  maybe ports/sdl2.c, ports/headless.c, ports/wasm.c

bug in build/test route?
  maybe Makefile, mk/, configs/, scripts/
```

## 下一步: Experiment Setup

下一章先讀 [ch06-00-experiment-setup.md](ch06-00-experiment-setup.md). 目標不是做正式 PR, 而是先把可反覆實驗的工作場整理出來:

```text
git strategy
  learn/*, exp/*, fix/* 不要混.

build flow
  .config, src/iui_config.h, Makefile, Kconfig 要先對上.

environment
  Windows/MSYS2/PowerShell 的 shell boundary 要先確認.
```

整理完後, 再進 [ch06-01-minimal-app-lab.md](ch06-01-minimal-app-lab.md), 把本章的 public API contract 變成可觀察的 program.

## 本章學到的三件事

> [Summary]
> 1. `include/iui.h` 不是單純 widget list, 而是 application, backend, renderer, and widget state ownership 之間的 public contract.
> 2. 讀 public API 時要先分層: compile-time spec, context/config, input, frame lifecycle, layout, widgets, overlay/focus, theme/motion, a11y/perf.
> 3. immediate-mode 不代表沒有 state. `libiui` 把 runtime UI state 放在 `iui_context`, 把 application values and complex widget states 留給使用者, 把 platform/rendering state 留給 backend.

## Source Files Read

```text
include/iui.h
include/iui-spec.h
notes/libiui-review/ch04-runtime-model.md
notes/libiui-review/README.md
```
