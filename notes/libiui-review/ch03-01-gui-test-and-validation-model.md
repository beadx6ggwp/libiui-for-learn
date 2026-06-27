# Ch03.1: GUI Test and Validation Model

## 本章問題

你現在問的是很核心的問題: GUI framework 到底怎麼測?

一般 library test 很常像這樣:

```text
input value
  -> function
  -> return value
```

上一章先建立一般 software testing 的 mental model: arrange-act-assert, unit/integration/E2E boundary, test double, control, observability, regression. 現在可以把那些概念帶進 GUI framework.

GUI framework 不只是一個 value transform. 它包含 frame, input timing, layout rectangle, focus state, modal blocking, draw calls, sometimes pixels. 所以如果直接打開 `tests/test-slider.c` 或 `scripts/headless-test.py`, 會覺得它不像一般 unit test.

> [Scope]
> 本章建立 `libiui` 的 GUI testing mental model. 目標是理解 `tests/common.*`, `tests/main.c`, `tests/test-*.c`, `ports/headless.c`, `scripts/headless-test.py` 各自在測什麼. 不深入每個 widget 的完整實作, 不討論 CI 細節, 不處理 Windows build 問題.

## 先退一步: GUI 測試到底難在哪

naive 想法是:

```text
button()
  -> 看它有沒有畫出按鈕
```

這太粗. 因為一個 UI widget 的正確性通常不是單一輸出, 而是多個維度同時成立:

```text
geometry
  widget 在哪裡? width, height, clip 是否正確?

input
  mouse, key, text, scroll 怎麼進來?

time
  frame 1 press, frame 2 update, frame 3 release 之間 state 怎麼變?

state
  hover, pressed, focused, disabled, modal blocked 是否正確?

render output
  draw_box, draw_text, state layer, focus ring 是否被呼叫?

pixels
  如果有 framebuffer, 最後畫出來的顏色和圖像是否正確?

spec
  MD3 高度, touch target, alpha, duration, contrast 是否符合常數規格?
```

所以 GUI 測試不是只問:

```text
return value correct?
```

而是問:

```text
given this frame sequence,
does the UI runtime produce the expected state, draw calls, and pixels?
```

ASCII 圖:

```text
Frame N input
  mouse at (x, y)
  button pressed
        |
        v
widget call stream
  iui_begin_frame()
  iui_begin_window()
  iui_button()
  iui_end_frame()
        |
        v
observable outputs
  return value
  ctx state
  mock draw counters
  last bounds
  framebuffer pixels
```

這也是為什麼測試一個 GUI framework 通常會分層. 你不會每個 test 都真的打開視窗看畫面, 也不會只測 return value.

## libiui 的測試分成三層

這個 repo 的測試可以先分成三層:

```text
Layer 1: C unit tests
  tests/main.c
  tests/common.c
  tests/common.h
  tests/test-*.c

Layer 2: Headless backend tests
  ports/headless.c
  ports/headless.h
  scripts/headless-test.py

Layer 3: Spec and visual validation
  tests/test-spec.c
  src/md3-spec.dsl
  generated md3 validators
  screenshots and golden images
```

一張整體圖:

```text
                         +----------------------+
                         |      make check      |
                         +----------+-----------+
                                    |
                  +-----------------+-----------------+
                  |                                   |
                  v                                   v
        +--------------------+              +----------------------+
        |    libiui_test     |              | scripts/headless-    |
        | C unit test binary |              | test.py              |
        +---------+----------+              +----------+-----------+
                  |                                    |
                  v                                    v
        +--------------------+              +----------------------+
        | tests/test-*.c     |              | generated C program  |
        | tests/common.*     |              | headless backend     |
        +---------+----------+              +----------+-----------+
                  |                                    |
                  v                                    v
        +--------------------+              +----------------------+
        | mock renderer      |              | software framebuffer |
        | counters + bounds  |              | input injection      |
        +--------------------+              +----------------------+
```

這三層的差異:

```text
C unit tests
  快, 直接, 適合測 API behavior and internal state transition.

headless backend tests
  更接近真實 port, 適合測 frame loop, input injection, screenshots, pixel output.

spec validation
  確認 MD3 constants and generated validation rules 沒偏掉.
```

## `tests/common.*`: 測試世界的假 renderer

`tests/common.h` 和 `tests/common.c` 是 C unit tests 的基礎設施.

它提供幾件事:

```text
test counters
  g_tests_run
  g_tests_passed
  g_tests_failed

assert macros
  ASSERT_TRUE
  ASSERT_FALSE
  ASSERT_EQ
  ASSERT_NEAR

test window macros
  BEGIN_TEST_WINDOW
  END_TEST_WINDOW

mock renderer callbacks
  mock_draw_box
  mock_draw_text
  mock_set_clip
  mock_draw_line
  mock_draw_circle
  mock_draw_arc

context factory
  create_test_context()

interaction helpers
  test_simulate_click()
  test_simulate_click_frames()
  test_simulate_drag()
```

最關鍵的是 mock renderer. 一般 app 會提供 SDL2 or WASM renderer callback, 但 unit test 不需要真的畫到螢幕. 它只需要知道:

```text
這個 widget 有沒有要求畫 box?
最後畫的 box bounds 是什麼?
文字是否被要求畫出來?
clip rect 是否被設定?
vector primitives 是否被呼叫?
```

所以 mock renderer 把 draw calls 轉成可檢查的全域變數:

```text
mock_draw_box()
  g_draw_box_calls++
  g_last_box_x = rect.x
  g_last_box_y = rect.y
  g_last_box_w = rect.width
  g_last_box_h = rect.height
  g_last_box_color = color

mock_draw_text()
  g_draw_text_calls++
  g_last_text_content = text

mock_set_clip()
  g_set_clip_calls++
  g_last_clip_min_x = min_x
```

ASCII 圖:

```text
widget under test
  iui_slider_ex()
        |
        v
src/draw.c
  iui_emit_box()
        |
        v
iui_renderer_t.draw_box
        |
        v
mock_draw_box()
        |
        v
test can assert:
  g_draw_box_calls > 0
  g_last_box_w is expected
```

這是一個很常見的 GUI test strategy:

```text
不要真的看畫面.
先把 rendering boundary 換成 spy/mock.
用 draw calls 當成 observable evidence.
```

## C unit test 的基本形狀

一個 `libiui` C unit test 通常長這樣:

```text
TEST(name)
allocate fixed buffer
create_test_context()
iui_begin_frame()
iui_begin_window()
call widget or API
assert return value / draw call / internal state
iui_end_window()
iui_end_frame()
free buffer
PASS()
```

例如 `tests/test-slider.c` 裡的 basic slider test:

```text
create ctx
begin frame
begin window
value = iui_slider_ex(ctx, 50, 0, 100, 1, NULL)
assert value == 50
end window
end frame
```

這不是在測滑鼠拖曳, 而是在測:

```text
slider API can run inside a valid frame/window.
initial value remains stable.
basic rendering path does not break.
```

另一個 slider test 會測 quantization:

```text
value = iui_slider_ex(ctx, 23, 0, 100, 10, NULL)
expect 20

value = iui_slider_ex(ctx, 27, 0, 100, 10, NULL)
expect 30
```

這就是比較接近 pure function 的行為測試:

```text
input numeric value + step
  -> normalized slider value
```

但互動測試會開始引入 frame and input.

## Button test: 以 repo 的行為為準

`tests/test-input.c` 裡有 `test_button_state_sequence()`. 它測的是:

```text
mouse position inside button

Frame 1:
  no mouse button
  iui_button()
  expect clicked == false

Frame 2:
  mouse left pressed
  iui_button()
  expect clicked == true

Frame 3:
  mouse left released
  iui_button()
  expect clicked == false
```

ASCII timeline:

```text
Frame 1: idle
  mouse inside
  pressed = 0
  released = 0
  iui_button()
  clicked = false

Frame 2: pressed
  mouse inside
  pressed = LEFT
  released = 0
  iui_button()
  clicked = true

Frame 3: released
  mouse inside
  pressed = 0
  released = LEFT
  iui_button()
  clicked = false
```

這裡有一個重要學習點:

```text
不要把別的 GUI framework 的 click semantics 直接套進來.
```

有些 UI 系統會在 release inside widget 時 submit. 但目前這個 repo 的 button test 和 `src/basic.c` 的實作顯示: `iui_button_styled()` 在 `IUI_STATE_PRESSED` 時會設定 `clicked = true`, focused Enter 也會 activate.

所以寫測試或閱讀測試時, 要以 repo 自己的 tests and implementation 作為 truth:

```text
speculation:
  button should return true on release.

repo evidence:
  test_button_state_sequence expects true on pressed frame.
```

這種觀念對 contributor 很重要. PR 不是提出你覺得 GUI 應該怎樣, 而是先確認目前 project 定義的 behavior.

## Slider test: 測 value, draw, disabled, and bounds

`tests/test-slider.c` 展示了 GUI widget test 常見的幾個角度:

```text
basic
  value stays 50.

custom colors
  options can be passed without breaking.

labels
  start and end labels work.

value indicator
  value_format and indicator option work.

disabled
  disabled slider keeps value.

invalid bounds
  min > max or min == max should not corrupt result.

step quantization
  23 with step 10 becomes 20.
  27 with step 10 becomes 30.

clamping
  -50 becomes 0.
  150 becomes 100.

rendering
  g_draw_box_calls > 0.
```

可以把 slider test 分成兩類:

```text
numeric behavior
  clamping, quantization, invalid bounds.

UI behavior
  disabled interaction, draw calls, frame re-render stability.
```

ASCII 圖:

```text
iui_slider_ex(ctx, value, min, max, step, options)
        |
        +--> numeric normalization
        |      clamp to [min, max]
        |      quantize by step
        |
        +--> interaction
        |      mouse position
        |      drag state
        |      disabled guard
        |
        +--> rendering
               track box
               handle box
               value indicator
               optional labels
```

所以好的 GUI widget test 不只測一個 output. 它會拆成多個 observable behavior.

## Modal test: 測 input blocking

`tests/test-modal.c` 展示另一種 GUI-specific 測試: modal blocking.

modal 的重點不是畫一個 box, 而是:

```text
modal active 時, 背後 UI 不應該吃 input.
modal 裡面的 UI 仍然可以吃 input.
```

`test_modal_input_blocking()` 大概在測:

```text
1. mouse at (50, 50), pressed.
2. before modal:
     iui_get_component_state(bounds, false)
     expect IUI_STATE_PRESSED.
3. begin modal and end modal.
4. same bounds:
     iui_get_component_state(bounds, false)
     expect IUI_STATE_DEFAULT.
```

ASCII 圖:

```text
without modal

mouse press
    |
    v
background button bounds
    |
    v
state = PRESSED

with modal

mouse press
    |
    v
modal layer active
    |
    +----> inside modal bounds can process input
    |
    +----> background bounds blocked
             state = DEFAULT
```

這種 test 不是 pixel test. 它直接測 UI runtime 的 input routing rule.

這對你理解 GUI framework 很重要:

```text
GUI correctness includes what must not receive input.
```

## Spec test: 測設計規格不是測互動

`tests/test-spec.c` 是另一種 test. 它不是問 button 會不會 click, 而是問 MD3 constants 是否符合 spec.

例如:

```text
button height == 40dp
FAB size == 56dp
chip height == 32dp
slider track height == 4dp
tab height == 48dp
state layer hover alpha == 8%
duration token == expected milliseconds
WCAG contrast threshold == expected ratio
```

它還 include:

```text
src/md3-validate-gen.inc
tests/test-md3-gen.inc
```

這代表一部分 validation 是從 `src/md3-spec.dsl` 產生出來的.

這層測試很像 graphics 裡測 fixed pipeline constants:

```text
If token values are wrong,
all widgets may render consistently but still violate MD3.
```

所以它測的是:

```text
design token correctness
generated validator correctness
accessibility and contrast helper correctness
```

不是單一 widget 的互動.

## `tests/main.c`: 把 C tests 串成 test binary

`tests/main.c` 是 C unit test runner.

它做的事很直接:

```text
parse -v / -vv
create one context for demonstration tests
run_demo_tests(ctx)
run all unit test sections
print summary
return 1 if any failed
```

Test sections 包含:

```text
run_init_tests()
run_layout_tests()
run_widget_tests()
run_input_tests()
run_slider_ex_tests()
run_spec_tests()
run_focus_tests()
run_clip_tests()
run_field_tracking_tests()
run_overflow_tests()
run_navigation_tests()
run_bottom_sheet_tests()
run_box_tests()
```

所以 `libiui_test` 的結構是:

```text
tests/main.c
        |
        v
run_*_tests()
        |
        v
static test functions in tests/test-*.c
        |
        v
ASSERT_* / mock renderer / ctx state
        |
        v
pass or fail
```

## Headless backend: 不開視窗也能跑 GUI

C unit tests 用 mock renderer. Headless backend 則更接近真實 backend, 但不需要開 GUI 視窗.

`ports/headless.h` 暴露測試用 API:

```text
iui_headless_set_max_frames()
iui_headless_get_frame_count()
iui_headless_inject_input()
iui_headless_inject_click()
iui_headless_inject_key()
iui_headless_inject_text()
iui_headless_get_framebuffer()
iui_headless_get_pixel()
iui_headless_save_screenshot()
iui_headless_get_stats()
```

`ports/headless.c` 的 renderer callbacks 會:

```text
headless_draw_box()
  stats.draw_box_calls++
  rasterize rounded rect into framebuffer if enabled

headless_draw_line()
  stats.draw_line_calls++
  rasterize line

headless_set_clip_rect()
  stats.set_clip_calls++
  update raster clip
```

這和 mock renderer 的差別:

```text
mock renderer
  record last call only.
  fast and simple.
  good for unit tests.

headless backend
  owns framebuffer.
  can inject input through port interface.
  can save screenshot.
  good for integration and visual tests.
```

ASCII 圖:

```text
headless test program
        |
        v
g_iui_port = headless backend
        |
        v
iui_port_apply_input()
        |
        v
libiui widgets
        |
        v
headless renderer callbacks
        |
        v
software framebuffer + stats
        |
        +--> assert stats
        +--> get pixel
        +--> save screenshot
```

這就是 GUI 測試常見的 "headless rendering" 概念:

```text
run the UI without a real window,
but still execute real frame loop and rendering backend.
```

## `scripts/headless-test.py`: 動態產生小型 GUI test program

`scripts/headless-test.py` 更進一步. 它不是只跑既有 C test files, 而是用 Python 產生小型 C program.

它裡面有 `TestCase`:

```text
name
description
code
state_vars
inject_code
validate_code
min_box_calls
```

例如 button test concept 是:

```text
state_vars:
  static int click_count = 0;

widget code:
  if (iui_button(ctx, "Click Me", IUI_ALIGN_LEFT))
      click_count++;

inject:
  if (frame == 1)
      iui_headless_inject_click(port, 80, 50);

validate:
  if (frame == 4)
      test_passed = (click_count >= 1);
```

這其實是在測:

```text
input injected at frame 1
  -> port transfers input to libiui
  -> widget receives interaction
  -> application state changes
  -> later frame validates final state
```

`TEST_TEMPLATE` 的 frame loop 大概是:

```text
while (g_iui_port.poll_events(port)) {
    frame = iui_headless_get_frame_count(port);

    inject input for selected frame

    g_iui_port.get_input(port, &in);
    iui_port_apply_input(ctx, &in);

    g_iui_port.begin_frame(port);
    iui_begin_frame(ctx, dt);
    iui_begin_window(ctx, "Test", 10, 10, 380, 280, 0);

    widget code under test

    iui_end_window(ctx);
    iui_end_frame(ctx);
    g_iui_port.end_frame(port);

    validate if frame is ready
}
```

ASCII timeline:

```text
Frame 0
  render initial UI

Frame 1
  inject click or key
  apply input
  render widget

Frame 2-3
  allow runtime state and animation to settle

Frame 4
  validate app state
  print passed:1
```

這和 C unit test 最大差異是:

```text
C unit test
  directly calls libiui functions with mock renderer.

headless Python test
  builds a mini app and runs it through backend-like frame loop.
```

## Visual regression: 真的比較 pixels

`scripts/headless-test.py` 也支援 visual regression:

```text
--golden
  generate golden images.

--visual
  render current screenshot and compare to golden.

--tolerance
  allow small per-channel pixel difference.
```

流程:

```text
run test with screenshot
        |
        v
headless framebuffer saved as PNG
        |
        v
load current image
load golden image
        |
        v
compare width, height, RGBA pixels
        |
        v
pass if differences within tolerance
```

這種測試比 draw call counters 更強, 但成本也更高:

```text
pros
  can catch actual visual changes.

cons
  more brittle.
  platform differences can cause noise.
  needs golden image management.
```

所以 visual regression 通常不是每個小邏輯都用. 它適合用在:

```text
rendering backend
complex widget appearance
layout regression
MD3 visual conformance
```

## 不同 test type 應該測什麼

可以用這張決策表:

```text
Pure API behavior
  use C unit test.
  example: clamp value, invalid input, null safety.

Frame state transition
  use C unit test with multiple begin_frame/end_frame.
  example: hover, focus, stale field tracking.

Draw call happened
  use mock renderer counters.
  example: g_draw_box_calls > 0.

Backend integration
  use headless backend.
  example: input injection through iui_port_t, framebuffer stats.

Pixel-level rendering
  use headless screenshot and visual regression.
  example: compare current PNG to golden.

Design token correctness
  use spec test.
  example: IUI_BUTTON_HEIGHT == 40.
```

ASCII map:

```text
Question you ask
        |
        +-- is this just a value rule?
        |       -> C unit test
        |
        +-- does this need frame timing?
        |       -> multi-frame C unit test
        |
        +-- does this need backend input?
        |       -> headless backend test
        |
        +-- does this need actual pixels?
        |       -> visual regression
        |
        +-- does this need MD3 spec constants?
                -> spec validation test
```

## 為什麼這對 PR 很重要

對 upstream PR 來說, reviewer 想知道的不只是 "我改了 code". 他會想知道:

```text
problem:
  哪個行為不對?

root cause:
  是 input, layout, draw, backend, build, or spec?

patch:
  修哪一層?

validation:
  哪個 test 或 command 證明它被修好?
```

所以理解測試模型能幫你決定 validation evidence:

```text
修改 widget numeric logic
  add or update tests/test-slider.c.

修改 modal input routing
  add or update tests/test-modal.c.

修改 MD3 token
  update tests/test-spec.c or generated validator.

修改 backend rendering
  use headless test, screenshot, or port-specific validation.

修改 build or config
  use make defconfig, make check, and specific target build evidence.
```

如果未來你做 Windows build PR, 測試證據就不一定是新增 GUI test. 可能是:

```text
make defconfig
make
make check
specific failing target now builds
possibly headless-test.py if behavior can run without SDL2
```

但如果你改的是 UI runtime 行為, 就應該回到上面的 test layer 去補 coverage.

## 建議閱讀順序

要理解測試, 我建議這樣讀:

```text
1. tests/common.h
   看 test macros, mock renderer, context factory.

2. tests/common.c
   看 mock renderer 如何記錄 draw calls.

3. tests/main.c
   看 test sections 如何被串起來.

4. tests/test-input.c
   看 mouse/key/text input and button state sequence.

5. tests/test-slider.c
   看 value rule, draw call, disabled behavior.

6. tests/test-spec.c
   看 MD3 constants and generated validation.

7. ports/headless.h + ports/headless.c
   看 headless backend 能 inject input, record stats, save screenshots.

8. scripts/headless-test.py
   看 Python 如何產生 mini app, compile, run, validate, compare images.
```

## 本章學到的三件事

> [Summary]
> 1. GUI 測試不是只測 return value, 而是控制 frame, input, layout, state, draw calls, pixels, and spec constants.
> 2. `libiui` 的 C unit tests 用 mock renderer 觀察 draw calls and bounds, headless backend 則用 software framebuffer and input injection 測更接近真實 app 的流程.
> 3. 寫 PR validation 時要先判斷改動屬於 API behavior, frame state, backend rendering, visual output, or MD3 spec, 再選對應測試層.

## 下一章

下一章讀 [ch04-runtime-model.md](ch04-runtime-model.md), 從一個 widget 的完整生命週期往下追:

```text
input update
  iui_update_mouse_buttons()

frame start
  iui_begin_frame()

layout allocation
  src/layout.c

widget logic
  src/basic.c or src/input.c

draw helper
  src/draw.c

backend callback
  ports/headless.c or ports/sdl2.c

test evidence
  tests/test-*.c
```

之後再開 `ch05-public-api-tour.md`, 從 `include/iui.h` 的 API group 開始讀. 目前 runtime model 已經接上測試篇建立的觀察方法.

## Source Files Read

```text
tests/main.c
tests/common.h
tests/common.c
tests/test-slider.c
tests/test-input.c
tests/test-spec.c
tests/test-modal.c
ports/headless.h
ports/headless.c
scripts/headless-test.py
src/basic.c
notes/libiui-review/README.md
notes/libiui-review/ch02-build-flow.md
notes/libiui-review/ch02-00-c-build-system-foundations.md
notes/libiui-review/ch02-01-libiui-build-flow.md
```
