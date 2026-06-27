# Ch04: Runtime Model

## 本章問題

前面幾章已經知道:

```text
ch00
  libiui 是 immediate-mode UI library.

ch01
  include/, src/, ports/, tests/ 各自負責什麼.

ch02
  Kconfig/Makefile 如何選出要編的 modules and backend.

ch03
  測試如何控制 frame, input, renderer output, and headless backend.
```

現在要進入真正的 runtime 問題:

```text
application 呼叫 iui_button() 時, libiui 到底做了什麼?
input state 放在哪裡?
layout rect 從哪裡來?
widget identity 怎麼建立?
hover/pressed/focus 怎麼判斷?
draw call 怎麼送到 backend?
test 又怎麼知道它是對的?
```

> [Scope]
> 本章用 `iui_button()` 當 vertical slice. 目標是建立 runtime mental model: input update -> frame lifecycle -> window/layout -> widget identity -> component state -> draw helpers -> renderer callback -> tests. 不逐行解釋所有 widget, 不深入 slider drag, textfield editing, modal/dialog complex behavior.

## Naive 想法: immediate-mode 就是每 frame 重畫

如果只聽過 immediate-mode UI, 很容易得到一個過度簡化的想法:

```text
每一 frame 重新呼叫 widget.
所以 library 不需要保存 state.
button 只是畫一個 rect, mouse 在上面就 return true.
```

這只對 toy UI 成立. 真正的 immediate-mode UI 仍然需要 state, 只是它不保存一棵長期存在的 widget tree.

比較準確的模型是:

```text
application
  每 frame 重新描述 UI call stream.

libiui context
  保存跨 frame 必要的 runtime state:
    input edge
    mouse held
    hover animation
    focused widget id
    window positions
    modal state
    field tracking
    layout cursor
    clipping
    draw batching
```

也就是:

```text
Immediate-mode does not mean stateless.
Immediate-mode means widget tree is rebuilt from calls every frame.
```

ASCII 圖:

```text
Frame N:
  app calls:
    iui_begin_window("Test")
    iui_button("Apply")
    iui_end_window()

  libiui context keeps:
    mouse_pos
    mouse_pressed / held / released
    focused_widget_id
    hover.widget
    animation.widget
    windows[]

Frame N+1:
  app calls the same shape again.
  libiui matches it through label, rect, focus registration, and context state.
```

所以本章要回答的不是 "button 畫了什麼", 而是:

```text
一個 button call 如何從 application-level function call,
變成 layout decision, interaction state, draw commands, and testable behavior?
```

## 先看一個最小使用流程

典型使用方式是:

```c
iui_update_mouse_pos(ctx, x, y);
iui_update_mouse_buttons(ctx, pressed, released);

iui_begin_frame(ctx, dt);
iui_begin_window(ctx, "Test", 100, 100, 300, 200, 0);

if (iui_button(ctx, "Apply", IUI_ALIGN_CENTER))
    apply();

iui_end_window(ctx);
iui_end_frame(ctx);
```

整理成 graph:

```text
platform or test injects input
        |
        v
iui_update_mouse_pos()
iui_update_mouse_buttons()
        |
        v
iui_begin_frame()
        |
        v
iui_begin_window()
        |
        v
iui_button()
        |
        +--> compute rect
        +--> register focusable
        +--> expand touch target
        +--> get component state
        +--> draw background/text/focus
        +--> return clicked
        |
        v
iui_end_window()
        |
        v
iui_end_frame()
```

這裡的第一個重點:

```text
input update happens before the frame is interpreted.
widget call consumes the current frame's input edge.
iui_end_frame() cleans up frame-local edges for the next frame.
```

在目前 source 裡, `iui_update_mouse_buttons()` 寫入:

```text
ctx->mouse_pressed
ctx->mouse_released
ctx->mouse_held
```

而 `iui_end_frame()` 會呼叫 `iui_input_frame_begin(ctx)` 清掉:

```text
mouse_pressed
mouse_released
key_pressed
char_input
```

所以 `mouse_pressed` 是 frame edge, 不是長期狀態. `mouse_held` 才是跨 frame 的 held state.

## 這個 repo 的 button activation rule

先把行為講清楚, 避免拿其他 GUI framework 的直覺套進來.

`tests/test-input.c` 的 `test_button_state_sequence()` 驗證:

```text
idle frame:
  clicked == false

pressed frame:
  clicked == true

released frame:
  clicked == false
```

也就是目前 `libiui` 的 `iui_button()` 在 mouse press edge 那一 frame 回傳 true, 不是等 mouse release 才回傳 true.

測試流程可以簡化成:

```text
mouse_pos inside button

Frame 1:
  pressed=0, released=0
  iui_button() -> false

Frame 2:
  pressed=IUI_MOUSE_LEFT, released=0
  iui_button() -> true

Frame 3:
  pressed=0, released=IUI_MOUSE_LEFT
  iui_button() -> false
```

這件事很重要, 因為你之後看 PR 或 bug 時不能假設:

```text
button activation must happen on release
```

你要先看 repo 的 current behavior and tests.

## Frame Lifecycle: 每一 frame 先重設哪些東西

`src/layout.c` 的 `iui_begin_frame()` 做的是 frame setup. 它不是只標記 "開始畫". 它會更新或重設一批 per-frame state:

```text
current_window = NULL
delta_time = current dt
animation.t advances
hover.t advances
cursor_blink advances
modal frame count updates
input layer double buffer swaps
field tracking registration arrays reset
focus_count = 0
focus_index = -1
appbar state resets
mouse click clears keyboard focus
box layout state resets
draw batch count resets
ink bounds resets
```

可以畫成:

```text
iui_begin_frame(ctx, dt)
        |
        +--> prepare input/layer systems
        +--> reset per-frame focus registration
        +--> reset per-frame layout helpers
        +--> advance animation timers
        +--> clear current_window
        v
ready to interpret this frame's UI calls
```

這裡有一個 immediate-mode 的關鍵:

```text
focus_order is rebuilt every frame.
```

每個 focusable widget 在這一 frame 被呼叫時, 會重新 register. 這樣 `iui_end_frame()` 才知道這一 frame 有哪些 widget 可被 Tab focus.

## Window Lifecycle: layout cursor 從這裡開始

`iui_begin_window()` 會用 window name hash 找到或建立 `iui_window`. 它還會:

```text
apply auto-size from previous frame
handle resize and dragging
draw window surface
initialize ctx->layout
draw title
push window content clip
```

對 `iui_button()` 來說, 最重要的是這段:

```text
ctx->layout = window content area
```

也就是 button 不是自己決定要放在整個 screen 哪裡. 它讀取目前的 layout cursor:

```text
ctx->layout.x
ctx->layout.y
ctx->layout.width
ctx->layout.height
ctx->row_height
ctx->padding
```

簡化圖:

```text
iui_begin_window("Test", 100, 100, 300, 200)
        |
        v
window rect:
  x=100, y=100, w=300, h=200
        |
        v
content layout:
  x = window.x + padding * 2
  y = title bottom + padding
  width = window.width - padding * 4
  height = row_height
        |
        v
next widget uses ctx->layout
```

`iui_end_window()` 之後會:

```text
update min_height and min_width
assert clip balance
clear current_window
pop clip
reset layout modes
```

所以 window 是一個 frame-local interpretation boundary:

```text
inside window:
  widgets can draw and consume layout.

outside window:
  many widgets early-return or draw nothing.
```

`tests/test-widget.c` 也有測到 widgets outside window 不應產生 draw calls.

## Widget Step 1: public API wrapper

Public API 在 `include/iui.h`:

```c
bool iui_button(iui_context *ctx,
                const char *label,
                iui_text_alignment_t alignment);
```

註解寫的是:

```text
Returns true if the button was pressed this frame.
```

實作在 `src/basic.c`:

```c
bool iui_button(iui_context *ctx,
                const char *label,
                iui_text_alignment_t alignment)
{
    return iui_button_styled(ctx, label, alignment, IUI_BUTTON_TONAL);
}
```

所以 `iui_button()` 只是 default style wrapper. 真正 runtime slice 在:

```text
iui_button_styled()
```

## Widget Step 2: guard current window

`iui_button_styled()` 一開始:

```c
if (!ctx->current_window || !label)
    return false;
```

這個 guard 的意義是:

```text
button 必須在 window 內呼叫.
label 不能是 NULL.
```

也就是 `libiui` 的 widget call 不是獨立畫圖函式. 它依賴目前 frame 中的 window/layout context.

ASCII:

```text
current_window == NULL
        |
        v
iui_button() returns false
no draw
no layout consumption
```

這和 `tests/test-widget.c` 的 outside-window 測試吻合.

## Widget Step 3: compute geometry

button 需要先算 visual rect:

```text
button_rect
  實際畫出來的 button bounds.

touch_rect
  擴大後的 input hit target.
```

一般 flow layout 下:

```text
text_width = iui_get_text_width(ctx, label)
btn_height = min(IUI_BUTTON_HEIGHT, ctx->row_height)
corner = btn_height * 0.5
button_width = text_width + 2 * padding
button_y = layout.y + vertical centering
button_x = depends on alignment
```

圖:

```text
ctx->layout
+------------------------------------------------+
|                                                |
|          +--------------------------+          |
|          |          Apply           |          |
|          +--------------------------+          |
|                                                |
+------------------------------------------------+

IUI_ALIGN_CENTER:
  button.x = layout.x + layout.width / 2 - button.width / 2
```

grid mode 則不同:

```text
if ctx->in_grid:
  button_rect = ctx->layout
  corner = ctx->layout.height * 0.5
```

所以同一個 widget function 會根據 layout mode 決定 geometry. 這也是 source map 裡說 `layout.c` 和 widget modules 密切耦合的原因.

## Widget Step 4: identity

Immediate-mode 每 frame 重新呼叫 widget, 那 library 要怎麼知道 "這個 button" 是哪一個?

`libiui` 用:

```c
uint32_t widget_id = iui_widget_id(label, button_rect);
```

`iui_widget_id()` 目前是:

```text
hash(label) XOR hash(rect.x, rect.y)
```

也就是:

```text
same label + same approximate position
  -> same widget id

same label + different position
  -> different widget id
```

圖:

```text
Frame N:
  label="OK", rect=(200,150,...)
        |
        v
  widget_id = H("OK") ^ H(200,150)

Frame N+1:
  label="OK", rect=(200,150,...)
        |
        v
  same widget_id
```

這讓 focus/navigation 可以跨 frame 指到相同 widget.

但 source comment 也寫出 tradeoff:

```text
如果 dynamic list reorder 導致位置變了, ID 也會變.
這時應考慮 iui_push_id()/iui_pop_id() 這類 stable id stack.
```

這是 immediate-mode UI 的核心問題之一:

```text
widget identity must be reconstructed from call-site data.
```

## Widget Step 5: focus registration

button 會呼叫:

```c
iui_register_focusable(ctx, widget_id, button_rect, corner);
bool is_focused = iui_widget_is_focused(ctx, widget_id);
```

`iui_register_focusable()` 會把這個 frame 看到的 focusable widget 放進:

```text
ctx->focus_order[]
ctx->focus_rects[]
ctx->focus_corners[]
ctx->focus_count
```

`iui_begin_frame()` 會先把 `focus_count` 重設成 0. 所以 focus list 每 frame 重建:

```text
Frame N:
  Button1 registers index 0
  Button2 registers index 1
  Button3 registers index 2
  end_frame processes focus navigation

Frame N+1:
  focus_count reset
  visible widgets register again
```

`tests/test-focus.c` 的 multiple widgets 測試就是這個模型:

```text
Frame 1:
  render Button1, Button2, Button3
  call iui_focus_next()
  end_frame selects Button1

Frame 2:
  render same buttons again
  call iui_focus_next()
  end_frame selects Button2
```

所以 keyboard focus 不是 widget object 自己保存, 而是:

```text
focused_widget_id persists in ctx.
focus_order is rebuilt each frame from current widget calls.
```

## Widget Step 6: touch target

MD3 button visual height 是 40dp, 但 touch target 要至少 48dp. `iui_button_styled()` 會做:

```c
iui_rect_t touch_rect = button_rect;
iui_expand_touch_target_h(&touch_rect, IUI_BUTTON_MIN_TOUCH_TARGET);
```

這代表:

```text
visual rect
  用來畫 button.

touch rect
  用來判斷 hover/press.
```

ASCII:

```text
touch_rect 48dp high
+--------------------------------+
|                                |
|  button_rect 40dp high         |
|  +--------------------------+  |
|  |          Apply           |  |
|  +--------------------------+  |
|                                |
+--------------------------------+
```

這也解釋了測試或 debugging 時不能只看最後一個 drawn box. Hit target 可能比 visual box 大.

## Widget Step 7: component state

button 取得 interaction state:

```c
iui_state_t state = iui_get_component_state(ctx, touch_rect, false);
```

`src/draw.c` 的 `iui_get_component_state()` 大致是:

```text
if disabled:
  return DISABLED

if modal active and not rendering modal:
  return DEFAULT

hovered = mouse_pos inside bounds

if hovered and mouse_pressed left:
  return PRESSED

if hovered:
  return HOVERED

return DEFAULT
```

圖:

```text
mouse_pos inside touch_rect?
        |
        +-- no --> IUI_STATE_DEFAULT
        |
        +-- yes
              |
              +-- mouse_pressed left? --> IUI_STATE_PRESSED
              |
              +-- otherwise -----------> IUI_STATE_HOVERED
```

這裡要注意兩件事:

```text
1. button 的 mouse activation 使用 mouse_pressed edge.
2. iui_get_component_state() 是很多 widget 共用的 state helper.
```

所以 button 的 clicked rule 不是散落在測試裡猜測, 而是:

```text
iui_update_mouse_buttons()
  sets ctx->mouse_pressed

iui_get_component_state()
  sees mouse_pressed + hover -> IUI_STATE_PRESSED

iui_button_styled()
  sees IUI_STATE_PRESSED -> clicked = true
```

## Widget Step 8: keyboard activation

mouse 之外, button 也支援 focus + Enter:

```c
if (is_focused && (ctx->key_pressed == IUI_KEY_ENTER)) {
    clicked = true;
    ctx->key_pressed = IUI_KEY_NONE;
    ctx->animation = {.widget = (void *) label};
}
```

這裡有兩個設計點:

```text
1. keyboard activation 依賴 focused_widget_id.
2. key_pressed 被 consume, 避免同一 frame 被後續 widget 重複使用.
```

所以 input edge 有不同處理:

```text
mouse_pressed
  由 frame end 清掉.

key_pressed
  可能被 widget 主動 consume.
```

## Widget Step 9: visual style and animation

button style 決定:

```text
background color
text color
border color
hover layer
```

`iui_button()` default 是 `IUI_BUTTON_TONAL`. 其他 style 包含:

```text
filled
outlined
text
elevated
tonal
```

當 state 是 pressed:

```c
clicked = true;
ctx->animation = {.widget = (void *) label};
```

下一段會依照:

```text
ctx->animation.widget == label
```

做 press animation:

```text
blend color
shrink/expand rect by ease impulse
```

hover 則使用:

```text
ctx->hover.widget == label
```

去控制 hover transition.

這裡有一個值得記住的 state ownership pattern:

```text
button does not allocate Button object.
button writes small runtime state into ctx->animation and ctx->hover.
```

## Widget Step 10: draw commands

最後 button 會畫:

```text
focus ring, if focused
background box or state layer
outline, if outlined style
label text
MD3 validation touch target
```

主要呼叫:

```text
iui_draw_focus_ring()
iui_emit_box()
iui_draw_rect_outline()
iui_internal_draw_text()
IUI_MD3_TRACK_BUTTON()
```

`iui_emit_box()` 是內部統一入口:

```text
update ink bounds
if batching enabled:
  add rect command to batch
else:
  call ctx->renderer.draw_box()
```

圖:

```text
iui_button_styled()
        |
        +--> iui_emit_box()
        |       |
        |       +--> ctx->renderer.draw_box(...)
        |
        +--> iui_internal_draw_text()
                |
                +--> ctx->renderer.draw_text(...)
                or vector font fallback
```

這裡就是 backend boundary:

```text
src/basic.c
  decides what should be drawn.

renderer callback
  decides how it becomes pixels or test evidence.
```

## Renderer callback 到 backend

`tests/common.c` 的 mock renderer 會記錄:

```text
g_draw_box_calls
g_draw_text_calls
g_last_box_x/y/w/h
g_last_text_content
```

這讓 C unit tests 可以不開視窗也觀察:

```text
button 是否產生 draw call?
text 是否被畫?
bounds 是否合理?
```

headless backend 則更接近 real backend:

```text
ports/headless.c
  headless_draw_box()
    increments stats
    rasterizes to software framebuffer

  headless_set_clip_rect()
    updates software rasterizer clip

  framebuffer
    can be used for screenshot or visual regression
```

所以同一個 `iui_emit_box()` 可以走到兩種觀察方式:

```text
C unit test mock renderer
  record calls and last parameters.

headless backend
  rasterize into framebuffer and collect stats.
```

這正好接上 ch03 的 testing model:

```text
return value
draw call
bounds
pixel/framebuffer
```

## 一張完整 button vertical slice

把上面串起來:

```text
test or backend
  iui_update_mouse_pos(ctx, x, y)
  iui_update_mouse_buttons(ctx, pressed, released)
        |
        v
src/event.c
  ctx->mouse_pos
  ctx->mouse_pressed
  ctx->mouse_held
  ctx->mouse_released
        |
        v
src/layout.c
  iui_begin_frame()
    reset current_window
    reset focus registration
    advance animation/hover
        |
        v
  iui_begin_window()
    find/create window
    set ctx->layout
    push clip
        |
        v
include/iui.h
  iui_button(ctx, "Apply", IUI_ALIGN_CENTER)
        |
        v
src/basic.c
  iui_button_styled()
    compute button_rect
    widget_id = hash(label) ^ hash(position)
    register focusable
    expand touch_rect
    state = iui_get_component_state()
    clicked = state == PRESSED or focused Enter
    emit box/text/focus
        |
        v
src/draw.c + src/internal.h
  iui_get_component_state()
  iui_emit_box()
  iui_internal_draw_text()
        |
        v
renderer callbacks
  draw_box()
  draw_text()
        |
        v
tests/common.c or ports/headless.c or ports/sdl2.c
  mock record / framebuffer / real display
        |
        v
src/layout.c
  iui_end_window()
  iui_end_frame()
    process focus navigation
    clear input edges
```

這張圖是之後讀其他 widget 的模板.

## 用 pipeline 直覺重新理解

如果用 graphics pipeline 的直覺看, `iui_button()` 像一個 CPU-side UI primitive setup:

```text
Application call stream
  like command stream.

layout rect
  like screen-space primitive bounds.

input hit test
  like coverage test, but for interaction.

component state
  like shading state selected by hover/press/focus.

draw helpers
  generate backend-independent draw commands.

renderer callbacks
  backend-specific emission.
```

但也要注意這個類比的邊界:

```text
GPU pipeline consumes geometry and produces pixels.
UI runtime consumes input + layout + state and produces draw commands + behavior result.
```

所以 button 的 return value 是 UI runtime 多出來的東西:

```text
draw output:
  pixels or draw calls.

behavior output:
  clicked == true/false.
```

測試也必須同時看這兩種輸出.

## Debug Runtime 問題時要問的順序

之後看到 bug, 不要直接問 "button 壞了". 先沿著 slice 分層:

```text
1. Input edge 正確嗎?
   iui_update_mouse_buttons() 有沒有在 frame 前呼叫?
   mouse_pressed, mouse_held, mouse_released 分別是什麼?

2. Layout rect 正確嗎?
   current_window 是否存在?
   ctx->layout 是否是預期位置?
   grid/flow mode 是否不同?

3. Widget identity 穩定嗎?
   label 是否重複?
   position 是否每 frame 變動?
   需要 iui_push_id()/iui_pop_id() 嗎?

4. State helper 判斷對嗎?
   mouse_pos 是否在 touch_rect?
   modal/input layer 是否 block input?
   disabled 是否為 true?

5. Draw command 有送出嗎?
   iui_emit_box() 是否呼叫 renderer.draw_box?
   batching/clip 是否影響觀察?

6. Test 觀察點對嗎?
   是要 assert return value?
   draw call count?
   last bounds?
   framebuffer pixel?
```

這個順序比直接讀整個 `src/core.c` 更有效, 因為它沿著實際資料流走.

## 本章學到的三件事

> [Summary]
> 1. Immediate-mode UI 不是沒有 state, 而是每 frame 重建 widget call stream, 並把必要的 interaction/window/focus/layout state 放在 `iui_context`.
> 2. `iui_button()` 的 runtime path 是 input edge -> frame/window setup -> layout rect -> widget id -> focus registration -> touch target -> component state -> draw callbacks -> end-frame cleanup.
> 3. 目前 `libiui` 的 button click rule 是 press frame returns true, release frame returns false; 這由 `tests/test-input.c` 驗證, 不能直接套用其他 GUI framework 的 release-click 直覺.

## 下一章

接下來先讀 [ch05-public-api-tour.md](ch05-public-api-tour.md).

ch04 是從 implementation side 追 `iui_button()` 的 runtime path. ch05 反過來從使用者視角整理 `include/iui.h` and `include/iui-spec.h`, 看 public API 如何描述 context, renderer callbacks, input, frame lifecycle, layout, widgets, overlays, focus, theme, and performance surface.

讀完 ch05 後, 再進入 minimal app lab 會比較合理. 因為那時已經知道哪些 API 是 app 必須呼叫的 contract, 哪些 API 是 backend 必須提供的 boundary.

## Source Files Read

```text
include/iui.h
src/event.c
src/layout.c
src/basic.c
src/draw.c
src/core.c
src/internal.h
tests/common.h
tests/common.c
tests/test-input.c
tests/test-focus.c
tests/test-widget.c
ports/headless.c
notes/libiui-review/ch03-00-software-testing-foundations.md
notes/libiui-review/ch03-01-gui-test-and-validation-model.md
```
