# Ch03.0: Software Testing Foundations

## 本章問題

E2E test 常常像是在問:

```text
從使用者角度, 整條流程有沒有成功?
```

但開發端測試更常問:

```text
這個小單元的行為是否被清楚定義?
這個模組和相鄰模組接起來是否正確?
這個 bug 以後會不會回來?
這次改動有沒有破壞既有 contract?
```

所以在進入 GUI 測試前, 需要先補一層 software testing 的基礎 mental model. 不然你會看到 `mock renderer`, `headless backend`, `ASSERT_NEAR`, `test_button_state_sequence()` 時, 只覺得它們是零散技巧, 而不是同一套驗證方法的不同層級.

> [Scope]
> 本章不教特定 testing framework, 也不追 `libiui` 的每個 test case. 本章先建立開發端測試的基本概念: test purpose, unit/integration/e2e 差異, arrange-act-assert, test double, determinism, observability, regression. 下一章再把這些概念帶進 GUI framework testing.

## 測試不是證明程式完全正確

naive 想法是:

```text
有測試
  -> 程式正確

測試全過
  -> 沒有 bug
```

這是不對的. 測試不是數學證明. 測試比較像工程上的可重複檢查:

```text
Given a specific condition,
when we perform a specific action,
then this specific behavior should hold.
```

測試真正提供的是:

```text
executable expectation
  把你對行為的期待寫成可以執行的檢查.

regression guard
  防止同一個 bug 或同一類破壞再次出現.

design pressure
  迫使 code 有清楚 boundary, 可控制 input, 可觀察 output.

review evidence
  告訴 reviewer 你怎麼確認這個 patch 有效.
```

所以比較正確的心智模型是:

```text
Tests do not prove absence of bugs.
Tests make selected assumptions executable and repeatable.
```

ASCII 圖:

```text
human expectation
  "disabled slider should not change value"
        |
        v
test case
  create disabled slider
  simulate input
  assert value unchanged
        |
        v
future protection
  if someone breaks disabled behavior,
  this test fails.
```

## 最小測試結構: Arrange, Act, Assert

很多開發端測試都可以拆成三段:

```text
Arrange
  建立初始條件.

Act
  執行被測行為.

Assert
  檢查結果是否符合期待.
```

例如一個數值 function:

```text
Arrange:
  value = 27
  step = 10

Act:
  result = quantize(value, step)

Assert:
  result == 30
```

套到 `libiui` 的 slider:

```text
Arrange:
  create iui_context
  begin frame
  begin window

Act:
  value = iui_slider_ex(ctx, 27, 0, 100, 10, NULL)

Assert:
  value is near 30
```

GUI 會比較複雜, 但基本結構不變:

```text
Arrange:
  frame state, mouse position, widget state, mock renderer.

Act:
  call widget functions for this frame.

Assert:
  return value, internal state, draw calls, or pixels.
```

這就是為什麼測試讀起來常常會有很多 setup. Setup 不是雜訊, 它是在建構一個可控制的世界.

## Unit, integration, E2E 在切什麼

測試分層不是看工具名稱, 而是看你切的 boundary.

```text
Unit test
  只測一個小行為或小模組.
  dependencies 通常被隔離或簡化.

Integration test
  測多個模組接起來是否正確.
  重點是 boundary and contract.

E2E test
  從使用者流程或系統外部入口測整條路.
  最接近真實使用, 但也最慢, 最難定位失敗原因.
```

ASCII 圖:

```text
Unit
  [A] only

Integration
  [A] ---> [B] ---> [C]

E2E
  user/browser/process
        |
        v
  [UI] -> [app] -> [storage/network/backend]
```

它們各自回答不同問題:

```text
Unit test asks:
  這個小行為是否符合定義?

Integration test asks:
  這些模組接起來是否符合 contract?

E2E test asks:
  使用者可見流程是否成功?
```

對開發者來說, unit/integration tests 通常更常在改 code 時使用, 因為它們:

```text
更快
更靠近問題源頭
失敗時比較好定位
更適合保護小型行為規則
```

E2E 很重要, 但它通常不是唯一答案. 如果所有東西都只靠 E2E, 會出現這種問題:

```text
E2E failed
  是 UI broken?
  是 backend broken?
  是 network mock broken?
  是 timing issue?
  是 test data broken?
```

開發端測試希望把問題切小:

```text
small behavior fails
  -> likely local bug.

module boundary fails
  -> likely integration contract bug.

E2E fails
  -> likely full-system behavior bug, but needs further localization.
```

## 測試金字塔不是死規則

常見說法是 test pyramid:

```text
             E2E
            /   \
       integration
        /       \
       unit tests
```

它想表達的是:

```text
越下面:
  數量多, 跑得快, 定位容易.

越上面:
  數量少, 更接近真實使用, 但慢且脆弱.
```

但不要把它當宗教. 對不同系統, 合理比例不同.

例如 `libiui` 這種 C immediate-mode UI library:

```text
unit-like C tests
  很有價值, 因為 widget logic, layout, input state 可以直接控制.

headless integration tests
  很有價值, 因為可以跑 frame loop and renderer callbacks.

visual regression
  有價值, 但成本高, 需要 golden images.

browser-style E2E
  不是核心, 因為這不是 web app.
```

所以更好的問題不是:

```text
這算不算 unit test?
```

而是:

```text
這個測試切在哪個 boundary?
它控制了什麼 input?
它觀察了什麼 output?
失敗時能不能快速定位?
```

## Test double: dummy, stub, spy, mock, fake

開發端測試常常需要把真實 dependency 換掉. 這些替代品統稱 test double.

你可以先記這幾種:

```text
dummy
  只是填參數, 不會真的被用到.

stub
  回傳固定答案.

spy
  記錄發生了什麼, 讓 test 之後檢查.

mock
  預先設定期待, 驗證某些 call 是否發生.

fake
  有簡化實作的替代系統.
```

用 `libiui` 來對應:

```text
mock_text_width()
  類似 stub.
  給定字串, 用固定規則回傳寬度.

mock_draw_box()
  比較像 spy.
  它記錄 draw_box 被呼叫幾次, 最後一次 rect 是什麼.

ports/headless.c
  比較像 fake backend.
  它不是 SDL2 真視窗, 但有 framebuffer, input injection, draw stats.
```

ASCII 圖:

```text
real app
  widget -> SDL2 renderer -> window pixels

unit test
  widget -> mock renderer -> counters and last rect

headless test
  widget -> fake backend -> software framebuffer
```

這是開發端測試的一個核心技巧:

```text
If the real dependency is slow, unstable, external, or hard to observe,
replace it with a controlled test double.
```

## Control and observability

一個測試好不好, 很大程度取決於兩件事:

```text
control
  測試能不能精準控制初始狀態和 input?

observability
  測試能不能精準觀察 output or side effect?
```

如果兩者都差, 測試就會很脆弱:

```text
cannot control timing
cannot control random data
cannot observe internal result
only sees final full-system failure
```

開發端測試會努力把這兩件事做好:

```text
control:
  fixed buffer
  fixed frame delta time
  explicit mouse position
  explicit key input
  fixed config

observability:
  return value
  assert macros
  draw call counters
  last draw bounds
  framebuffer pixel
  screenshot diff
```

套到 `libiui`:

```text
control
  iui_update_mouse_pos(ctx, x, y)
  iui_update_mouse_buttons(ctx, pressed, released)
  iui_begin_frame(ctx, 1.0f / 60.0f)

observability
  clicked return value
  g_draw_box_calls
  g_last_box_x/y/w/h
  iui_headless_get_pixel()
  PNG screenshot
```

這也是 GUI 測試的重點: 你要能控制時間, input, rendering boundary, 才能穩定測 UI behavior.

## Regression test: 把 bug 變成 guard

開發端測試最實用的用法之一是 regression test.

流程:

```text
1. 發現 bug.
2. 寫一個會重現 bug 的 test.
3. 確認 test 在修正前會 fail.
4. 修 code.
5. 確認 test pass.
6. 以後這個 bug 回來, test 會 fail.
```

ASCII 圖:

```text
bug report
   |
   v
failing test
   |
   v
fix implementation
   |
   v
test passes
   |
   v
future regression guard
```

這個觀念對 PR 很重要. 如果你只說:

```text
I fixed it.
```

reviewer 還是不知道:

```text
你怎麼確認?
以後會不會再壞?
這個問題的最小重現條件是什麼?
```

更好的 PR evidence 是:

```text
This test fails before the patch and passes after the patch.
```

不是每個 patch 都一定要新增測試, 但你至少要能說明 validation path.

## 測試命名應該說出行為

好的測試名稱不是只說 function name, 而是說行為.

比較弱:

```text
test_slider()
test_button()
test_layout()
```

比較好:

```text
test_slider_ex_step_quantization()
test_slider_disabled_no_interaction()
test_button_state_sequence()
test_modal_input_blocking()
test_window_autosize_expands_to_content()
```

原因是測試失敗時, 名稱就是第一份錯誤報告:

```text
FAIL test_slider_disabled_no_interaction
  -> disabled slider interaction rule broke.

FAIL test_modal_input_blocking
  -> modal layer no longer blocks background input.
```

這對開發者定位問題很有幫助.

## 測試也會有壞味道

測試不是越多越好. 壞測試會拖慢開發.

常見壞味道:

```text
too broad
  一個測試檢查太多行為, 失敗時不知道哪裡壞.

too coupled to implementation
  測 private detail 太深, 小重構就壞.

too flaky
  跟 timing, random, environment 綁太重.

unclear assertion
  fail 時不知道期待是什麼.

only snapshots
  只比較大塊 output, 但不知道語義是什麼.
```

開發端測試要盡量做到:

```text
focused
  每個 test 有清楚行為目標.

deterministic
  同一個 input 每次得到同一個結果.

diagnostic
  fail 時能幫你定位.

maintainable
  不因為無關 refactor 大量破裂.
```

## 和 E2E 的關係

你工作上接觸的 E2E 測試不是錯, 只是它回答的是不同問題.

可以這樣比較:

```text
E2E test
  outside-in.
  模擬使用者流程.
  高信心, 但慢, 失敗原因較遠.

developer-side unit/integration test
  inside-out or boundary-focused.
  驗證小行為和模組 contract.
  低成本, 失敗原因較近.
```

對一個 UI library 來說:

```text
E2E-like:
  headless mini app runs multiple frames and input injection.

developer unit:
  call iui_slider_ex() directly and assert value.

integration:
  widget -> layout -> draw helper -> mock renderer.

visual:
  widget -> headless framebuffer -> screenshot compare.
```

所以你可以把 `libiui` 的測試看成一個分層系統:

```text
small behavior
  C unit tests.

module connection
  C tests with mock renderer.

backend-like execution
  headless backend tests.

actual visual output
  screenshot and golden comparison.
```

## 帶回 libiui

現在先不要急著把所有測試都分類得很漂亮. 對 contributor 更有用的是問這幾個問題:

```text
1. 這個改動改的是哪一層?
   value rule, input state, layout, drawing, backend, build, or spec?

2. 最小可觀察結果是什麼?
   return value, draw call, bounds, pixel, command output?

3. 能不能控制重現條件?
   fixed frame, fixed mouse input, fixed config, fixed backend?

4. 失敗時能不能定位?
   test name and assertion 是否指出行為破壞?
```

以 `libiui` 為例:

```text
修改 slider value rule
  add C unit test in tests/test-slider.c.

修改 modal input blocking
  add C unit test in tests/test-modal.c.

修改 renderer backend
  use headless backend or port-specific validation.

修改 MD3 spec token
  update tests/test-spec.c or generated validator.

修改 build system
  validate make defconfig, make, make check, target-specific build.
```

這就是下一章 GUI 測試篇會接上的地方:

```text
一般測試問題:
  How do we control input and observe output?

GUI 測試問題:
  How do we control frame/input/layout/rendering/pixels?
```

## 本章學到的三件事

> [Summary]
> 1. 開發端測試不是證明程式完全正確, 而是把特定行為期待變成 executable and repeatable checks.
> 2. Unit, integration, E2E 的差異在於 boundary, control, observability, failure localization, 不是單純工具名稱.
> 3. Mock renderer, headless backend, screenshot comparison 都是 test double / observability 的具體形式, 只是分別服務不同層級的測試問題.

## 下一章

下一章讀 [ch03-01-gui-test-and-validation-model.md](ch03-01-gui-test-and-validation-model.md). 它會把本章的概念帶進 GUI framework:

```text
software testing foundation
  arrange-act-assert
  unit/integration/e2e
  test double
  control and observability

GUI testing in libiui
  frame sequence
  input injection
  mock renderer
  headless backend
  visual regression
  MD3 spec validation
```

## Source Files Read

```text
tests/common.h
tests/common.c
tests/main.c
tests/test-slider.c
tests/test-input.c
tests/test-modal.c
ports/headless.h
scripts/headless-test.py
notes/libiui-review/ch03-01-gui-test-and-validation-model.md
```
