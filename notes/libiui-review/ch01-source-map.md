# Ch01: Source Map

## 本章問題

上一章先回答 `libiui` 是什麼: 它是一個 pure C99 immediate-mode MD3 UI frontend. 這章要把那個抽象描述落到 repo layout:

```text
哪些檔案是 public API?
哪些檔案是 private runtime?
哪些檔案處理 input, layout, draw?
哪些檔案只是 backend?
tests/ 裡哪些是測試, 哪些是 demo?
```

這一步很重要, 因為如果你一開始只看檔名, 很容易把 `tests/example.c` 當成測試, 或把 `ports/sdl2.c` 當成 UI core. 這會讓後面判斷 bug scope 時混亂.

> [Scope]
> 本章建立 source tree map. 目標是知道每個目錄和主要檔案的責任邊界. 不逐行追 widget 實作, 不深入 build system, 不討論 Windows portability patch.

## 先退一步: C project 為什麼分 header 和 source

在 C project 裡, 先不要從 `.c` 開始亂讀. 更穩的問題是:

```text
誰是外部使用者能看到的 contract?
誰是內部實作細節?
誰只是 build 時被選進來的變體?
```

一般可以這樣分:

```text
.h files
  declarations, types, macros, API contracts.

.c files
  function bodies, private helpers, module implementation.

internal headers
  repo 內部共用, 不是 public API.
```

套到 `libiui`:

```text
include/iui.h
  public API. Application code include this.

include/iui-spec.h
  public MD3 constants and override points.

src/internal.h
  private shared runtime declarations. Not public API.

src/*.c
  implementation modules.

ports/*.c
  platform/backend implementation.
```

這個分法會避免一個常見誤解:

```text
Wrong:
  看到 function 就以為它是 library user 該直接呼叫的 API.

Better:
  先問它宣告在哪裡, include/iui.h 還是 src/internal.h?
```

## 第一張圖: 從使用者到 pixels

先把 source tree 轉成資料流:

```text
application
  includes include/iui.h
  owns app state and calls widgets every frame
        |
        v
libiui public API
  iui_begin_frame()
  iui_begin_window()
  iui_button()
  iui_slider()
  iui_end_frame()
        |
        v
src runtime
  context, input, layout, state, draw helpers, MD3 tokens
        |
        v
renderer callbacks
  draw_box()
  draw_text()
  set_clip_rect()
  draw_line()
  draw_circle()
  draw_arc()
        |
        v
ports backend
  SDL2 / WASM / headless / custom platform
```

Source map 的核心不是記住所有檔名, 而是記住這條邊界:

```text
include/
  defines what the outside world can ask libiui to do.

src/
  decides how libiui answers those requests.

ports/
  decides how drawing and input connect to a concrete platform.

tests/
  proves the above behavior and provides usage examples.
```

把這張圖再展開成 "檔案如何一起服務一個 button":

```text
Application code
  tests/example.c or user app
        |
        | calls public function
        v
include/iui.h
  declares iui_button()
        |
        | implementation selected by normal C linking
        v
src/basic.c
  iui_button()
  iui_button_styled()
        |
        +-----------------------------+
        |                             |
        v                             v
src/layout.c                    src/core.c + src/internal.h
  current layout rect             widget id
  window content area             focus state
  cursor advance                  input layer guard
        |                             field tracking
        |
        v
src/draw.c
  component state color
  state layer
  focus ring
  text drawing
  clip stack
        |
        v
iui_renderer_t callbacks
  draw_box()
  draw_text()
  set_clip_rect()
        |
        v
ports/sdl2.c or ports/headless.c or ports/wasm.c
  actual pixels or test framebuffer
```

同一個流程也可以用 "一個 button call 在做什麼" 來看:

```text
iui_button("Apply")
      |
      +--> identity
      |      hash label + id stack + rect
      |
      +--> geometry
      |      ask layout for current rect
      |
      +--> input
      |      mouse in rect?
      |      pressed this frame?
      |      activated by focused Enter?
      |
      +--> state
      |      default / hovered / pressed / focused / disabled
      |
      +--> render
      |      rounded rect
      |      state layer
      |      label text
      |      focus ring
      |
      +--> result
             return true when activation condition happened
```

這種圖的用法是: 之後你讀任何 widget, 都可以先問它落在這六個問題的哪一段:

```text
identity
geometry
input
state
render
result
```

## `include/`: public surface

`include/` 目前只有兩個主要 header:

```text
include/iui.h
include/iui-spec.h
```

`include/iui.h` 是使用者的主要入口. 它包含幾類 contract.

第一類是 context and memory:

```c
size_t iui_min_memory_size(void);
iui_config_t iui_make_config(void *buffer,
                             iui_renderer_t renderer,
                             float font_height,
                             const iui_vector_t *vector);
iui_context *iui_init(const iui_config_t *config);
```

這直接反映 `README.md` 的設計: application 提供 buffer, library 不主動 heap allocate.

第二類是 frame lifecycle:

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

這代表它是 immediate-mode. 你每一 frame 重新描述 UI.

第三類是 input update:

```c
void iui_update_mouse_pos(iui_context *ctx, float x, float y);
void iui_update_mouse_buttons(iui_context *ctx,
                              uint8_t pressed,
                              uint8_t released);
void iui_update_key(iui_context *ctx, int key);
void iui_update_char(iui_context *ctx, int codepoint);
```

這些 function 讓 backend 或 application 把 platform input 餵給 UI runtime.

第四類是 widgets and containers:

```text
iui_button()
iui_slider()
iui_checkbox()
iui_radio()
iui_textfield()
iui_card_begin()
iui_scroll_begin()
iui_menu_begin()
iui_dialog()
iui_tabs()
iui_nav_bar_item()
```

第五類是 drawing and renderer abstraction:

```c
typedef struct {
    void (*draw_box)(iui_rect_t rect, float radius, uint32_t color, void *user);
    void (*draw_text)(float x, float y, const char *text, uint32_t color, void *user);
    void (*set_clip_rect)(uint16_t min_x, uint16_t min_y,
                          uint16_t max_x, uint16_t max_y, void *user);
    float (*text_width)(const char *text, void *user);
    void (*draw_line)(...);
    void (*draw_circle)(...);
    void (*draw_arc)(...);
    void *user;
} iui_renderer_t;
```

這是 `libiui` 能支援 SDL2, WASM, headless, custom backend 的核心原因.

`include/iui-spec.h` 是 MD3 token table. 它存放:

```text
duration tokens
component dimensions
state layer alpha
touch targets
shape tokens
WCAG contrast thresholds
layout breakpoints
```

所以 `include/` 可以先理解成:

```text
iui.h
  behavior and API contract.

iui-spec.h
  Material Design 3 constants and override points.
```

## `src/internal.h`: private runtime map

`src/internal.h` 一開始就給了很有用的 dependency graph:

```text
core -> layout -> draw -> icons -> basic/input/container/modal -> complex
```

這句比檔名列表更重要. 它表示 source tree 的理解順序不是字母順序, 而是 dependency order.

可以畫成:

```text
core
  context, theme, focus, layers, field tracking, clipboard, IME
        |
        v
layout
  frame, window, box, grid, dirty rect
        |
        v
draw
  text, rect, clip, state layer, shadow, vector primitives, batching
        |
        v
base widgets
  basic.c, input.c, container.c, modal.c
        |
        v
complex widgets
  appbar.c, tabs.c, menu.c, dialog.c, navigation.c, pickers.c, searchbar.c
```

`internal.h` 也定義完整的 `struct iui_context`. 這不是 public API, 但它是理解 runtime 的關鍵. 裡面可以看到幾大區:

```text
renderer and theme
  iui_renderer_t renderer
  iui_theme_t colors

layout and current window
  iui_rect_t layout
  iui_window *current_window

input state
  mouse_pos
  key_pressed
  char_input
  mouse_pressed / held / released

modal and input layers
  modal
  input_layer
  input_capture
  focus_trap

animation and focus
  animation
  hover
  focused_widget_id

widget-specific state
  slider
  appbar
  dropdown
  scroll

large internal arrays
  windows
  focus_order
  id_stack
  string_buffer

performance systems
  dirty
  ink_bounds
  text_cache
  batch
  field_tracking
```

這告訴你一個重點:

```text
Immediate-mode 不代表完全沒有 state.
它代表 widget tree 不長期存在.
但 context 仍保存跨 frame 必要的 interaction state.
```

例如 slider drag, textfield focus, hover animation, modal layer, dirty rectangles 都需要跨 frame.

## `src/core.c`: context and cross-cutting systems

`src/core.c` 很大, 但不要把它想成所有 UI 的入口. 它比較像 runtime foundation:

```text
motion and easing
  iui_ease()
  iui_motion_apply()
  iui_motion_progress()

vector font support
  iui_text_width_vec()
  iui_draw_text_vec()

context lifecycle
  iui_min_memory_size()
  iui_make_config()
  iui_config_is_valid()
  iui_init()

theme
  iui_theme_light()
  iui_theme_dark()
  iui_set_theme()

ID and focus
  iui_push_id()
  iui_pop_id()
  iui_register_focusable()
  iui_focus_next()
  iui_focus_prev()

accessibility and contrast
  iui_contrast_ratio()
  iui_theme_validate_contrast()
  iui_a11y_*()

input layers and modal blocking
  iui_push_layer()
  iui_should_process_input()
  iui_begin_input_capture()

field tracking
  iui_register_textfield()
  iui_register_slider()
```

如果用 GPU pipeline 類比, `core.c` 不是 draw stage, 而像 runtime state manager:

```text
it owns context-wide state
it provides helpers used by many widget modules
it enforces invariants across frames
```

## `src/layout.c`: frame, window, and geometry allocation

`src/layout.c` 是 `begin_frame`, `begin_window`, layout cursor, box/grid system 的主要位置.

代表 function:

```text
iui_begin_frame()
iui_begin_window()
iui_end_window()
iui_end_frame()
iui_box_begin()
iui_box_next()
iui_box_end()
iui_grid_begin()
iui_grid_next()
iui_grid_end()
iui_get_layout_rect()
iui_require_content_width()
```

這個檔案回答的是:

```text
current frame 是否開始?
current window 是誰?
widget 要放在哪個 rectangle?
下一個 widget 的 cursor 如何前進?
window auto-size 如何知道 content 需要多寬?
```

可以把它看成 UI frontend 的 primitive assembler:

```text
application calls widgets in order
        |
        v
layout.c gives each widget a rectangle
        |
        v
widget module uses that rectangle for input and drawing
```

## `src/draw.c`: drawing primitives and visual helpers

`src/draw.c` 不只是呼叫 renderer callback. 它還處理很多 visual system:

```text
text width cache
  iui_get_text_width()
  iui_text_cache_*()

text and divider
  iui_text()
  iui_divider()

vector primitives
  iui_draw_line()
  iui_draw_circle()
  iui_draw_arc()

component visual state
  iui_get_component_state()
  iui_get_state_color()
  iui_draw_state_layer()

elevation and shadow
  iui_draw_shadow()
  iui_draw_elevated_box()

clipping
  iui_push_clip()
  iui_pop_clip()
  iui_is_clipped()

batching and ink bounds
  iui_batch_*()
  iui_ink_bounds_*()
```

它的責任是把 UI runtime 需要的視覺語義, 轉成 renderer callback 可以消化的基本 draw operation.

也就是:

```text
MD3 visual rule
  hover layer, shadow, rounded rect, text width
        |
        v
draw helper
        |
        v
renderer callback
```

## Base widget modules

Base widget modules 是最常被 application 直接用到的元件.

`src/basic.c`:

```text
iui_segmented()
iui_slider()
iui_slider_ex()
iui_range_slider()
iui_button()
iui_button_styled()
```

它負責最基礎的 clickable/control widgets.

`src/input.c`:

```text
iui_textfield()
iui_edit_with_selection()
iui_textfield_with_selection()
iui_switch()
iui_checkbox()
iui_radio()
iui_dropdown()
```

它處理 text editing, selection, keyboard navigation, switch/checkbox/radio/dropdown.

`src/container.c`:

```text
iui_card_begin()
iui_card_end()
iui_progress_linear()
iui_progress_circular()
iui_snackbar()
iui_scroll_begin()
iui_scroll_end()
iui_bottom_sheet_begin()
iui_tooltip()
iui_badge_number()
iui_banner()
iui_table_*()
iui_carousel_*()
```

這裡的 common pattern 是 container 或 compound UI surface. 它們通常不只是單一按鈕, 而是會改變 layout, clipping, or overlay behavior.

`src/modal.c`:

```text
iui_begin_modal()
iui_end_modal()
iui_close_modal()
iui_is_modal_active()
```

它提供 modal blocking 的基本機制, 讓 menu, dialog, bottom sheet 這類 overlay 可以擋住背後 input.

## Complex widget modules

Complex widgets 通常建立在 base runtime, layout, draw helpers 上.

```text
src/appbar.c
  top app bar and action icons.

src/fab.c
  floating action button and icon buttons.

src/chips.c
  assist, filter, input, suggestion chips.

src/list.c
  one-line, two-line, three-line list items.

src/tabs.c
  primary and secondary tabs.

src/navigation.c
  navigation rail, navigation bar, drawer, bottom app bar, side sheet.

src/menu.c
  menu lifecycle and menu items.

src/dialog.c
  alert dialog and fullscreen dialog.

src/pickers.c
  date picker and time picker.

src/searchbar.c
  search bar and search view.
```

這些檔案通常有共同結構:

```text
state object or parameters
        |
        v
compute layout rectangle
        |
        v
query input / focus / modal layer
        |
        v
draw MD3 visual states
        |
        v
return changed / clicked / selected
```

所以之後如果要讀單一 widget, 不要只看它畫了什麼. 要同時看:

```text
identity
  widget id 怎麼產生?

geometry
  rect 從哪裡來?

interaction
  hover, active, focus, disabled 怎麼判斷?

state ownership
  state 放 ctx, user struct, or pointer parameter?

rendering
  最後呼叫哪些 draw helpers?
```

## Font, icons, and generated MD3 data

有幾個檔案不屬於一般 widget module, 但會被 draw/widget 依賴:

```text
src/font.c
src/font.h
src/glyphs-data.inc
src/icons.c
src/md3-spec.dsl
src/md3-validate.h
src/md3-flags-gen.inc
src/md3-validate-gen.inc
```

`font.*` 和 `glyphs-data.inc` 支援 built-in vector font. 這符合 README 說的 "no FreeType, no external font files".

`icons.c` 提供內建 icon drawing. 很多 MD3 components 會用到 icon, 例如 FAB, app bar, textfield.

`md3-spec.dsl` 和 generated inc files 用在 MD3 validation. 這些檔案和 `tests/test-spec.c` 關係很密切.

要注意:

```text
generated .inc files
  不是一般手寫邏輯入口.
  之後讀 spec validation 時再回來看.
```

## `ports/`: backend boundary

`ports/` 的核心是 `ports/port.h`. 它定義 `iui_port_t`:

```text
init()
shutdown()
poll_events()
get_input()
begin_frame()
end_frame()
get_renderer_callbacks()
get_vector_callbacks()
get_delta_time()
get_window_size()
get_dpi_scale()
clipboard operations
```

每個 backend 都會提供同名 global:

```c
const iui_port_t g_iui_port = { ... };
```

而 build system 只會選一個 backend 編進來:

```text
CONFIG_PORT_SDL2
  ports/sdl2.c

CONFIG_PORT_WASM
  ports/wasm.c

CONFIG_PORT_HEADLESS
  ports/headless.c
```

這裡的邊界很關鍵:

```text
src/
  platform-independent UI logic.

ports/
  platform-specific event, framebuffer, renderer, clipboard, presentation.
```

`ports/sdl2.c`:

```text
desktop backend
SDL window and renderer
mouse, keyboard, text input, scroll
draw rounded rect, line, circle, arc
HiDPI scaling
```

`ports/wasm.c`:

```text
browser backend
Emscripten interop
JS event hooks
Canvas-oriented rendering path
```

`ports/headless.c`:

```text
test backend
software framebuffer
input injection
screenshot and pixel inspection
shared memory hooks
```

`ports/port-sw.h` and `ports/headless-shm.h`:

```text
shared software rendering helpers and headless/shared-memory support.
```

所以如果後面遇到 Windows SDL2 build 問題, 你要先問:

```text
是 src/ UI logic 問題?
是 ports/sdl2.c backend 問題?
是 Makefile/Kconfig detection 問題?
是 Windows C library portability 問題?
```

source map 的價值就在這裡.

## `tests/`: tests, demos, and evidence

`tests/` 不是單純 test-only. 它有三種角色.

第一種是 test harness:

```text
tests/main.c
tests/common.c
tests/common.h
```

`tests/main.c` 負責跑全部 test sections:

```text
run_init_tests()
run_layout_tests()
run_widget_tests()
run_input_tests()
run_spec_tests()
run_navigation_tests()
run_bottom_sheet_tests()
...
```

`tests/common.*` 提供:

```text
mock renderer callbacks
test context factory
BEGIN_TEST_WINDOW / END_TEST_WINDOW
click and drag simulation helpers
assert macros
section reporting
```

第二種是 behavior tests:

```text
test-init.c
test-layout.c
test-box.c
test-widget.c
test-input.c
test-slider.c
test-chip.c
test-modal.c
test-navigation.c
test-bottomsheet.c
test-scroll.c
test-focus.c
test-clip.c
test-tracking.c
test-overflow.c
```

這些檔案是最適合拿來理解 expected behavior 的地方. 例如:

```text
test-slider.c
  slider bounds, disabled behavior, drag interaction, click-to-value.

test-input.c
  keyboard editing, selection, UTF-8 helpers, button state sequence.

test-tracking.c
  textfield and slider stale state cleanup across frames.

test-overflow.c
  stack, buffer, coordinate, field tracking overflow guards.
```

第三種是 spec and demo:

```text
test-spec.c
  MD3 dimension, state layer, duration, easing, WCAG, generated validation.

example.c
  interactive demo and API usage reference.

bench-render.c
  rendering benchmark scenarios.

pixelwall-demo.c
forthsalon-demo.c
  optional demo workloads.
```

所以讀 `tests/` 時要分清楚:

```text
test-*.c
  expected behavior and regression coverage.

example.c
  how a real application uses the API.

bench/demo files
  visual or performance-oriented examples.
```

## `assets/`, `scripts/`, `configs/`, `mk/`, `.ci/`

這些不是 runtime source, 但會影響開發和驗證.

```text
assets/web/
  WASM demo page and JS glue.

scripts/
  compiler detection, headless test runner, local WASM server, generated data scripts.

configs/
  Kconfig and defconfig variants.

mk/
  Makefile fragments for toolchain, dependencies, common rules, tests.

.ci/
  GitHub Actions helper scripts.
```

目前你可以先把它們放在第二層:

```text
source map
  understand code ownership first.

build map
  then understand how configs select these files.
```

`ch02-build-flow.md` 會處理第二層, 並拆成 GCC/C build 基礎與 `libiui` build flow 兩篇.

## 一張更完整的 operation map

把 `include/`, `src/`, `ports/`, `tests/` 放在同一張圖, 會比較像這樣:

```text
                       +----------------------+
                       |      user app        |
                       |  tests/example.c     |
                       +----------+-----------+
                                  |
                                  | include public API
                                  v
                       +----------------------+
                       |    include/iui.h     |
                       | API + public types   |
                       +----------+-----------+
                                  |
                                  | implemented by
                                  v
+----------------------+   +------+-------------------+   +----------------------+
| include/iui-spec.h   |-->|          src/            |-->|     iui_renderer_t   |
| MD3 constants        |   | runtime + widgets        |   | callback interface   |
+----------------------+   |                          |   +----------+-----------+
                           | core.c                   |              |
                           | layout.c                 |              | provided by
                           | draw.c                   |              v
                           | basic.c / input.c        |   +----------------------+
                           | container.c / modal.c    |   |        ports/        |
                           | complex widgets          |   | SDL2 / WASM / headless|
                           +------+-------------------+   +----------+-----------+
                                  |                                  |
                                  | validated by                     | emits pixels/events
                                  v                                  v
                       +----------------------+          +----------------------+
                       |        tests/        |          | display or framebuffer|
                       | behavior + spec      |          | or browser canvas     |
                       +----------------------+          +----------------------+
```

再把 "測試怎麼進來" 畫出來:

```text
tests/main.c
  run_slider_ex_tests()
        |
        v
tests/test-slider.c
  creates test context via tests/common.c
        |
        v
mock renderer callbacks
  record draw calls and widget bounds
        |
        v
src/basic.c
  iui_slider_ex()
        |
        +--> src/layout.c
        +--> src/draw.c
        +--> src/core.c state/tracking
        |
        v
assert expected value, state, or interaction result
```

這說明 `tests/` 的角色不是 "另外一個世界". 它是用 controlled context 和 mock renderer, 去觀察 `src/` 是否產生正確行為.

## 一個可用的讀 code 順序

如果現在要正式讀 source, 我建議不要從最大檔案 `core.c` 逐行讀到尾. 先用這條 route:

```text
1. include/iui.h
   看使用者能呼叫什麼.

2. tests/example.c
   看 real usage pattern.

3. tests/test-slider.c or tests/test-widget.c
   看一個 widget 的 expected behavior.

4. src/basic.c
   追 iui_button() or iui_slider() 的實作.

5. src/layout.c + src/draw.c
   補上 widget rectangle 和 draw helper.

6. src/core.c + src/internal.h
   回頭理解 context state, focus, layer, tracking.

7. ports/sdl2.c or ports/headless.c
   看 backend 如何把 renderer callback 落地.
```

ASCII route:

```text
API surface
  include/iui.h
        |
        v
usage example
  tests/example.c
        |
        v
expected behavior
  tests/test-slider.c
        |
        v
widget implementation
  src/basic.c
        |
        +------> src/layout.c
        |
        +------> src/draw.c
        |
        v
runtime state
  src/internal.h + src/core.c
        |
        v
backend
  ports/sdl2.c / ports/headless.c
```

這條路線比直接看 build system 更接近你原本的 graphics intuition:

```text
API call
  -> expected behavior
  -> rectangle allocation
  -> state transition
  -> draw primitive
  -> backend implementation
```

## 本章學到的三件事

> [Summary]
> 1. `include/` 是 public contract, `src/internal.h` 和 `src/*.c` 是 private runtime and implementation, `ports/` 是 backend boundary.
> 2. `src/internal.h` 的 dependency order `core -> layout -> draw -> icons -> base widgets -> complex widgets` 比檔名字母順序更適合當閱讀路線.
> 3. `tests/` 同時包含 behavior tests, spec validation, demo and benchmark, 所以它也是理解 API usage 和 expected behavior 的重要入口.

## 下一章

下一章讀 [ch02-build-flow.md](ch02-build-flow.md), 把這章的 source map 接到 build system:

```text
source map asks:
  repo 有哪些 pieces?

build flow asks:
  這一次 config 要選哪些 pieces 編進去?
```

`ch02` 內部建議先讀 [ch02-00-c-build-system-foundations.md](ch02-00-c-build-system-foundations.md), 再讀 [ch02-01-libiui-build-flow.md](ch02-01-libiui-build-flow.md). 讀完後, 再進入 [ch03-00-software-testing-foundations.md](ch03-00-software-testing-foundations.md), 先補開發端測試基礎. 接著讀 [ch03-01-gui-test-and-validation-model.md](ch03-01-gui-test-and-validation-model.md), 理解 GUI 測試如何控制 frame, input, mock renderer, and headless backend. 之後讀 [ch04-runtime-model.md](ch04-runtime-model.md), 追一個 widget 從 input 到 draw 的完整路線.

## Source Files Read

```text
README.md
include/iui.h
include/iui-spec.h
src/internal.h
src/core.c
src/layout.c
src/draw.c
src/basic.c
src/input.c
src/container.c
src/modal.c
src/appbar.c
src/fab.c
src/chips.c
src/list.c
src/navigation.c
src/tabs.c
src/menu.c
src/dialog.c
src/pickers.c
src/searchbar.c
ports/port.h
ports/sdl2.c
ports/headless.c
ports/wasm.c
tests/main.c
tests/common.c
tests/common.h
tests/test-*.c
tests/example.c
tests/bench-render.c
```
