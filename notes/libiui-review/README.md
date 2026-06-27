# libiui Review Notes

## 這個資料夾在解決什麼

現在不應該直接進入第一個 PR. 更穩的路線是先建立一條 `learn/*` branch, 把個人 notes 和系統性 repo review 保存下來, 再從這個理解出發去做 Windows build 調查和 codebase 小切口搜尋.

也就是:

```text
learn branch
  保存學習筆記和 repo review

libiui-review
  釐清整份 repo 怎麼運作, 怎麼編譯, 各模組怎麼接起來

windows-build investigation + codebase review
  先收集 Windows 相關 build 問題, 再判斷是否形成可投稿 patch

fix/* branch
  從 upstream/main 開, 只放正式 PR patch
```

> [Scope]
> 這裡不是 upstream PR 文件, 也不是一般 Git 教材. 這裡是正式開發前的 repo review 路線圖, 目標是先理解 `sysprog21/libiui` 的架構和 build flow, 再決定第一個乾淨 PR.

## 為什麼先做 Learn Branch

naive 做法是:

```text
既然目標是 PR, 就直接從目前 main 修改 code.
```

問題是目前 repo 同時有三種不同性質的內容:

```text
personal learning notes
  AGENTS.md
  notes/
  repo review records

upstream-facing investigation
  Windows build evidence
  source reading notes
  bug reproduction

upstream PR patch
  code or docs change meant for sysprog21/libiui
```

如果這三種內容都在 `main` 或同一條 PR branch 混在一起, 之後會很難判斷:

```text
哪些是要保留給自己看的?
哪些只是調查證據?
哪些真的要送 upstream?
```

所以比較好的 branch 分工是:

```text
main
  keep clean, follows upstream/main

learn/*
  personal notes, review records, experiments

fix/*
  upstream PR branch, one focused patch
```

## 建議 Branch

可以建立:

```sh
git switch -c learn/repo-review
```

然後把目前 notes 保存進這條 branch:

```sh
git add AGENTS.md notes/
git commit -m "Add libiui learning and review notes"
git push -u origin learn/repo-review
```

圖:

```text
O <- main, origin/main, upstream/main
 \
  N <- learn/repo-review
       contains AGENTS.md and notes/
```

這樣做的意義是:

```text
notes 不再只是 untracked files.
main 仍可回到 upstream baseline.
正式 PR 不會從 learn/repo-review 開.
```

正式 PR 仍然要從 `upstream/main` 開:

```sh
git fetch upstream
git switch -c fix/msys2-ucrt-clock upstream/main
```

## Repo Review 的核心問題

要 review 整份 repo, 不是從檔案列表開始背. 要從幾個系統問題切入:

```text
Project orientation:
  libiui 是什麼?
  immediate-mode UI 在這個 repo 裡怎麼被拆成 input, layout, draw, backend?

Public API:
  include/iui.h 暴露哪些概念?
  src/ 裡哪些 module 實作這些概念?

Runtime model:
  immediate-mode UI state 放在哪裡?
  input, layout, draw, widget state 如何流動?

Backend boundary:
  core library 和 ports/sdl2.c, ports/headless.c, ports/wasm.c 怎麼分工?

Tests and demos:
  tests/ 哪些是 unit test?
  哪些是 demo?
  headless test 如何驗證 UI 行為?

Contribution surface:
  哪些改動是小而可 review 的 patch?
  哪些其實需要先討論 design?

Build:
  make defconfig 到 make 之間產生了什麼?
  Kconfig 和 .config 如何決定 backend, demo, test?
```

## 建議 Review 順序

第一輪只建立地圖, 不追細節. 章節順序應該先從 project orientation 開始, 不要一開始就進 build system:

```text
0. What is libiui
   專案定位, immediate-mode UI, frame flow, source tree 大方向.

1. Source map
   include/, src/, ports/, tests/ 的責任分工.

2.0. C build system foundations
   GCC preprocessing, compilation, assembling, linking, object files, static libraries, dependency files.

2.1. libiui build flow
   Makefile, mk/, configs/, Kconfig, generated files.

3.0. Software testing foundations
   unit/integration/E2E, arrange-act-assert, test doubles, control, observability.

3.1. GUI test and validation model
   tests/common.*, tests/main.c, test-*.c, ports/headless.c, scripts/headless-test.py.

4. Runtime model
   用 `iui_button()` 做 vertical slice, 追 input, frame, layout, widget state, draw callbacks, tests.

5. Public API
   include/iui.h, include/iui-spec.h.

6. Minimal app lab
   用最小 renderer callbacks, context buffer, frame loop 建立可實驗的 UI program.

7. Headless test lab
   用 headless backend 和 input injection 驗證 widget behavior and draw output.

8. Core modules
   src/input.c, src/layout.c, src/basic.c, src/menu.c, src/fab.c.

9. Render and backend boundary
   ports/sdl2.c, ports/headless.c, ports/wasm.c, assets/web/.

10. Existing contribution style
   commit history, PR discussion, CI workflow.
```

第二輪才針對 Windows build 和第一個 PR 問題深入. 這裡先不要預設只有某一個錯誤, 而是把 Windows 上遇到的 build 差異都列成候選問題:

```text
environment and shell
  MSYS2 UCRT64, MINGW64, Git Bash, PowerShell, or other setup.

toolchain and packages
  compiler, make, python, SDL2 package source.

observed failures
  exact command, exact target, exact error message.

known case
  localtime_r / localtime_s is one observed issue, not the whole topic.

build boundary
  which config enables the failing target, and whether the issue belongs to code, Makefile, docs, or environment.

validation
  commands that prove the problem is fixed or documented accurately.
```

## 2 + 4 要怎麼接

你前面選的 `2 + 4` 可以變成:

```text
2. Windows build 調查
  建立 notes/windows-build/
  先收集 environment, command, failure, suspected root cause, validation.
  不先假設只有 MSYS2 UCRT64 或 localtime_r/localtime_s.

4. codebase 小切口搜尋
  在 repo review 時找 small upstream-facing patches.
  例如 portability fix, test gap, docs gap, demo build issue.
```

它們不是兩條分開路線. 比較好的順序是:

```text
repo review
  -> 找到 build flow 和 target boundary
  -> Windows build 調查有根據
  -> 第一個 PR scope 更小
```

## 產出物規劃

可以先產出這幾篇:

```text
notes/libiui-review/ch00-what-is-libiui.md
  libiui 是什麼, immediate-mode UI mental model, repo 大方向.

notes/libiui-review/ch01-source-map.md
  include/, src/, ports/, tests/ 的責任分工.

notes/libiui-review/ch02-build-flow.md
  Build flow 入口, 說明 ch02-00 and ch02-01 的閱讀順序.

notes/libiui-review/ch02-00-c-build-system-foundations.md
  GCC/C build 基礎, preprocessing, compilation, object files, linking, static libraries, dependency files.

notes/libiui-review/ch02-01-libiui-build-flow.md
  make defconfig, .config, src/iui_config.h, generated files, build targets.

notes/libiui-review/ch03-00-software-testing-foundations.md
  開發端測試基礎, unit/integration/E2E, test double, control and observability.

notes/libiui-review/ch03-01-gui-test-and-validation-model.md
  GUI 測試模型, C unit tests, mock renderer, headless backend, screenshots, validation evidence.

notes/libiui-review/ch04-runtime-model.md
  immediate-mode UI 的 input, layout, draw, widget state flow.

notes/libiui-review/ch05-public-api-tour.md
  include/iui.h and include/iui-spec.h 的 public API 分類與使用者視角.

notes/libiui-review/ch06-minimal-app-lab.md
  用最小 renderer callbacks, context buffer, frame lifecycle, and widgets 建立實驗場.

notes/libiui-review/ch07-headless-test-lab.md
  用 headless backend, input injection, and observable output 練習驗證 UI behavior.

notes/windows-build/README.md
  Windows build 調查入口, 先列環境, 失敗案例, 待確認問題, validation plan.

notes/windows-build/cases.md
  逐步收集具體案例. `localtime_r` / `localtime_s` 只是其中一個候選案例.
```

## 本階段不要做什麼

不要:

```text
從 learn/repo-review 開 upstream PR.
把 notes/ merge 到 fix/*.
還沒理解 build flow 就先寫 README 大段 Windows 教學.
還沒重現 failure 就先修 code.
把 first attempt commit e722b54 直接 cherry-pick 成正式 PR.
```

原因是 upstream reviewer 要看的不是你的學習歷程, 而是:

```text
problem
root cause
minimal patch
validation
maintainability
```

learn branch 的價值是幫你建立這條因果鏈, 不是直接送出去.

## 本章學到的三件事

> [Summary]
> 1. 下一步應該先建立 `learn/repo-review`, 把 notes 和 repo review 系統性保存起來.
> 2. Repo review 要先釐清 build flow, source map, runtime model, backend boundary, tests and demos, 再挑 PR.
> 3. Windows build 調查和 codebase 小切口搜尋要接在 repo review 後面, 先收集問題再決定 PR scope, 正式 PR branch 仍然從 `upstream/main` 開.

## Links

```text
../libiui-git/
../libiui-contribution/
../0626_1705_project-thoughts-plan-questions.md
../0627_0051_libiui-development-model.md
```
