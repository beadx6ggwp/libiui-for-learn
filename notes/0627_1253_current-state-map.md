# Current State Map

## Question

目前最大的問題不是單一技術點, 而是資訊量開始超過短期工作記憶:

```text
Git branch / fork / upstream
PR 規範和 commit 風格
libiui repo 架構
C build system
Windows / MSYS2 / PowerShell build
minimal app lab
future upstream PR
```

這些東西都相關, 但不是同一層問題. 如果全部混在一起看, 就會感覺像是每一條路都同時要做.

這篇 note 的目的不是新增技術內容, 而是建立目前狀態地圖:

```text
現在已經整理了什麼?
哪些只是學習筆記?
哪些是未驗證 hypothesis?
哪些東西要等 baseline 後再正式化?
下一步有哪些合理選擇?
```

## Current Branch State

目前主要工作 branch 是:

```text
learn/repo-review
```

它的角色是:

```text
保存個人學習筆記
保存 repo review
保存 Git / PR / build 的思考過程
```

目前不應該把它當成 upstream PR branch. 它可以很完整, 也可以包含個人學習脈絡, 但它不應該直接送給 `sysprog21/libiui` review.

目前 `git status --short --branch` 顯示:

```text
## learn/repo-review...origin/learn/repo-review [ahead 4]
```

這代表 local `learn/repo-review` 比 `origin/learn/repo-review` 多 4 個 commit. 工作樹目前沒有顯示 modified or untracked files. 看到的:

```text
warning: unable to access 'C:\Users\david/.config/git/ignore': Permission denied
```

比較像 Git 嘗試讀取使用者層級 ignore 檔時的權限問題, 不是這個 repo 目前筆記狀態本身的問題.

## Why It Feels Confusing

現在會亂, 不是因為方向錯, 而是因為你其實同時在做五條不同工作流:

```text
1. Git safety
   如何保留 learn notes, 又不污染 upstream PR.

2. Contribution style
   如何寫 commit, PR body, tests, validation, review response.

3. Repo review
   libiui 是什麼, source tree 怎麼分層, build/runtime/test 怎麼接.

4. Windows build investigation
   MSYS2 UCRT64, PowerShell, SDL2, pkg-config, Makefile shell helper.

5. Experiment lab
   未來怎麼用 minimal program or headless backend 觀察 libiui behavior.
```

naive 想法會是:

```text
我是不是應該趕快找一個 PR 題目?
```

但這個想法太早. 現在比較正確的問題是:

```text
我是否已經知道哪個現象是可重現的 problem,
哪個檔案是 root cause,
哪個 minimal patch 能修,
哪個 command 能 validation?
```

目前答案還不是完全確定. 所以現在還是在 `learn` and `exp` 的交界, 還沒到 `fix`.

## Note System Map

目前 `notes/` 已經形成三種層級.

### Timeline Notes

時間軸筆記保存當時的想法, 不要求最後答案完整正確:

```text
notes/0626_1705_project-thoughts-plan-questions.md
notes/0626_1723_upstream-pr-submission-plan.md
notes/0626_1726_jserv-pr-guidelines.md
notes/0626_1731_repo-state-cleanup-strategy.md
notes/0626_2320_github-fork-pr-model.md
notes/0627_0013_git-operation-concepts.md
notes/0627_0026_pr-branch-concept.md
notes/0627_0051_libiui-development-model.md
notes/0627_1217_windows-build-strategy-thinking.md
notes/0627_1222_msys-build-environment-thinking.md
```

這類筆記的價值是保留思考歷史:

```text
當時為什麼這樣想?
哪些問題還沒驗證?
哪些 branch 操作是為了保護資料?
```

### Topic Notes

主題資料夾保存整理後的教學版本:

```text
notes/libiui-git/
notes/libiui-contribution/
notes/libiui-review/
```

它們不是時間軸, 而是教學文章. 寫法應該更完整, 更像能回頭學的章節.

### Future Topic

目前還沒有正式建立:

```text
notes/windows-build/
```

原因是 Windows build 還沒完成 clean baseline. 目前只有思考筆記, 還不應該太早寫成正式教學. 否則容易把未驗證假設寫得像結論.

## Git Knowledge Track

`notes/libiui-git/` 已經整理的是 Git 操作安全感:

```text
ch01-fork-origin-upstream.md
  fork, origin, upstream, local branch, remote-tracking branch.

ch02-working-tree-branch-safety.md
  working tree, index, commit, reset, clean, restore, cherry-pick.

ch03-pr-branch-force-push.md
  PR branch, force push, review branch update.

ch04-libiui-repo-policy.md
  main, learn/*, fix/*, notes/ 的分工.

ch05-diff-range-and-merge-base.md
  GitHub PR diff, merge-base, A..B, A...B.

ch06-rebase-conflict-and-recovery.md
  rebase conflict, stash, reflog, recovery.

ch07-learning-branch-workflow.md
  learn branch 怎麼保存 notes, feature branch 旁邊的 notes 怎麼處理.
```

這條線的核心結論:

```text
learn/repo-review
  personal notes and review.

exp/*
  temporary experiment.

fix/*
  clean upstream PR branch from upstream/main.

main
  ideally follows upstream/main.
```

這條線目前足夠支撐下一步實驗. 不需要一直繼續補 Git 理論, 除非實際操作時又遇到不確定.

## Contribution Track

`notes/libiui-contribution/` 已經整理的是怎麼讓 PR 可 review:

```text
ch01-pr-thinking-model.md
  maintainer 看 patch 時在看什麼.

ch02-commit-style.md
  commit subject, commit scope, history cleanup.

ch03-pr-body-and-review.md
  PR body, review response, draft PR, force push review branch.

ch04-docs-tests-evidence.md
  docs, tests, screenshots, CI evidence.

ch05-pr-operation-playbook.md
  從 Git 操作接到 PR 操作.

ch06-upstream-contributing-guide.md
  對照 upstream CONTRIBUTING.md.
```

這條線的核心模型是:

```text
problem -> root cause -> minimal change -> validation -> maintainability
```

這代表現在不能直接把 `learn/windows-first-attempt` 當成 PR. 那次可以當線索, 但還缺:

```text
clean baseline
exact failure
root cause
minimal patch reasoning
validation commands
```

## Repo Review Track

`notes/libiui-review/` 是目前最主要的學習主線. 它不是要立刻找 bug, 而是先理解 repo 怎麼運作.

目前章節大致是:

```text
ch00-what-is-libiui.md
  libiui 是什麼, immediate-mode UI 的基本模型.

ch01-source-map.md
  include/, src/, ports/, tests/, assets/ 的責任分工.

ch02-build-flow.md
  ch02 系列入口.

ch02-00-c-build-system-foundations.md
  C build system 基礎, preprocessing, compilation, object file, linking.

ch02-01-libiui-build-flow.md
  libiui core build graph, Kconfig, .config, src/iui_config.h, Makefile source selection.

ch02-02-libiui-build-variants-and-validation.md
  generated files, demo, make check, CI, WASM, build failure diagnosis.

ch03-00-software-testing-foundations.md
  軟體測試基礎.

ch03-01-gui-test-and-validation-model.md
  GUI 測試, headless backend, mock renderer, validation evidence.

ch04-runtime-model.md
  immediate-mode UI runtime flow.

ch05-public-api-tour.md
  include/iui.h and include/iui-spec.h 的 public API.

ch06-00-experiment-setup.md
  實驗前的 Git strategy, build flow, environment probes.
```

最近重要調整是把 build flow 拆成:

```text
ch02-00
  先講 C build 基礎.

ch02-01
  再講 libiui core build graph.

ch02-02
  最後講 build variants and validation.
```

這個拆分是合理的, 因為你第一次接觸比較完整的 C build flow, 不能一上來就把 Kconfig, Makefile, generated header, SDL2, WASM, CI 全塞在同一篇.

## Current Technical Understanding

目前對 `libiui` 的理解可以先壓成這張圖:

```text
user code / demo / tests
        |
        v
include/iui.h public API
        |
        v
src/
  input, layout, widgets, draw, theme, text
        |
        v
renderer callbacks / backend boundary
        |
        +--> ports/sdl2.c
        +--> ports/headless.c
        +--> ports/wasm.c
```

build flow 則是:

```text
configs/Kconfig + configs/defconfig
        |
        v
make defconfig
        |
        +--> .config
        |     used by Makefile to select sources and flags
        |
        +--> src/iui_config.h
              used by C preprocessor

Makefile + mk/*.mk
        |
        v
compile selected .c -> .o
        |
        v
archive libiui.a
        |
        v
link demo / tests / wasm variant
```

一個重要修正是:

```text
libiui 不是 runtime 自己動態選 OS/backend.
它更像 build-time 根據 CONFIG_* 選 source files and backend.
```

所以 Windows 問題不只是 "C code 能不能編". 還包括:

```text
Makefile shell behavior
dependency detection
pkg-config / sdl2-config
compiler target
runtime DLL
POSIX API portability
```

## Windows Build Track

目前 Windows build 還在 hypothesis stage, 不是 conclusion stage.

已經記錄的兩篇 timeline note 是:

```text
notes/0627_1217_windows-build-strategy-thinking.md
notes/0627_1222_msys-build-environment-thinking.md
```

它們目前的核心判斷:

```text
不要直接沿用 learn/windows-first-attempt.
不要假裝它不存在.
先從 clean upstream/main baseline 重新驗證.
```

舊 branch:

```text
learn/windows-first-attempt
  historical evidence / hypothesis.
```

它可能指出:

```text
tests/example.c might have localtime_r / time_t portability issue.
```

但它還不能證明:

```text
upstream/main 在 MSYS2 UCRT64 下第一個 failure 是什麼.
PowerShell 失敗是不是 shell helper 問題.
localtime_r 是否是唯一 issue.
README 大段 Windows 教學是否適合 upstream.
```

目前比較合理的 baseline plan:

```text
1. 開 clean experiment branch or worktree from upstream/main.
2. 在 MSYS2 UCRT64 shell 跑:
   make defconfig
   make
   make check

3. 在 PowerShell 跑同樣命令.
4. 比較 failure layer.
5. 再決定是否開 notes/windows-build/.
6. 只有 root cause 清楚時, 才開 fix/* branch.
```

## Maintainer-Style Critique Of Current Thinking

目前的直覺可以整理成:

```text
libiui 比想像中複雜.
它雖然是 sysprog21 裡相對小的 project, 但流程完整.
它作為 immediate-mode UI core, 需要處理 input, layout, widget behavior, draw output, and backend boundary.
不同 OS, compiler, shell, backend, module, and config 會影響 build flow.
如果要提 Windows PR, 不能只是讓自己電腦能跑.
需要理解 build toolchain, repo 運作, contribution style, coding style, commit/PR 原則.
```

這個方向大致正確, 但如果用 maintainer 角度批判, 最危險的是把這句話想得太大:

```text
要先很了解整個跨平台相容機制, 才能針對 Windows 提出支援.
```

比較精確的版本應該是:

```text
我要理解到足以證明這個 Windows failure 的 root cause,
並確認我的修法不會破壞其他 backend / config / platform.
```

也就是:

```text
need enough understanding to support the patch
not need complete understanding before any patch
```

這裡的取捨很重要. 如果理解範圍太小, 會變成:

```text
it works on my machine
```

這不適合 upstream PR.

但如果理解範圍無限擴張, 又會變成:

```text
一直讀 repo, 一直整理筆記, 但沒有產生可驗證問題.
```

這也不是好的工程節奏.

更好的切法是 bounded investigation:

```text
1. 先跑 clean baseline.
2. 只追第一個可重現 failure.
3. 只讀和這個 failure 有關的 build path / source path.
4. 如果修法很小, 才開 fix/* branch.
5. 如果問題其實是環境使用方式, 就寫 notes/docs, 不急著 patch code.
```

從 maintainer 角度, 不好的 PR 會長得像:

```text
Add Windows support
  大量 README.
  順手改 test.
  沒有清楚說原本哪個 command fail.
  沒有區分 MSYS2 UCRT64 and PowerShell.
  沒有明確 root cause.
```

比較好的 PR 會長得像:

```text
tests: use portable localtime helper on Windows

Problem:
  `make check` fails on MSYS2 UCRT64 because ...

Root cause:
  `localtime_r` is POSIX, but UCRT does not provide it.

Change:
  add a small Windows-specific wrapper using `localtime_s`.

Validation:
  MSYS2 UCRT64: make defconfig, make, make check
  Linux or CI: existing check still passes
```

所以目前的修正後結論是:

```text
不是為了理解而理解,
而是為了切出 reviewer 能檢查的因果鏈.
```

這條因果鏈仍然是:

```text
problem -> root cause -> minimal change -> validation -> maintainability
```

## Branch Model Going Forward

目前最重要的是不要讓三種東西混在同一條線:

```text
learn/repo-review
  notes, source reading, thinking, article drafts.

exp/windows-baseline
  baseline build experiment, logs, temporary probes.

fix/*
  upstream-facing patch only.
```

ASCII 圖:

```text
upstream/main
      |
      O----------------O
       \
        \ learn/repo-review
         N---N---N---N

upstream/main
      |
      O
       \
        \ exp/windows-baseline
         E---E

upstream/main
      |
      O
       \
        \ fix/small-focused-issue
         F
```

`learn` 可以保存大量脈絡. `exp` 可以做 messy evidence. `fix` 只能留下 reviewer 需要看的 minimal patch.

## What Not To Do Now

現在先不要做這些事:

```text
不要從 learn/repo-review 開 PR.
不要把 notes/ 帶進 fix/* branch.
不要把 learn/windows-first-attempt 直接 cherry-pick 成 PR.
不要在還沒 baseline 前寫正式 notes/windows-build/ 教學.
不要一看到 Windows error 就立刻改 code.
不要把 PowerShell failure 和 MSYS2 UCRT64 failure 混成同一件事.
```

原因是這些都會破壞因果鏈:

```text
problem -> root cause -> minimal change -> validation
```

現在需要的是先觀察, 再分類, 再修.

## How To Organize Future Ideas

目前另一個問題是: 發想越來越多, 如果只靠時間軸檔名, 之後會很難知道當時到底遇到什麼問題, 為什麼做了某個決策.

可以考慮幾種整理方式.

### Option 1: Timeline First, Synthesis Later

這是目前最推薦的預設方式.

做法:

```text
1. 當下想法先放 `notes/MMDD_HHMM_title.md`.
2. 每篇 timeline note 都寫:
   Question
   Context
   Current Thinking
   Critique
   Decision
   Follow-ups

3. 等同一主題累積 2-4 篇後, 再整理到 topic folder.
4. topic article 末尾保留 Source Notes.
```

優點:

```text
保留當時的思考歷史.
不會太早把 hypothesis 寫成 conclusion.
之後可以回頭看觀念是怎麼修正的.
```

代價:

```text
需要定期整理.
timeline notes 會越來越多.
```

適合現在的狀態, 因為 Windows build 還沒 baseline, 很多想法仍然是 hypothesis.

### Option 2: Topic Folder As Soon As A Theme Appears

做法:

```text
一看到主題就開資料夾:

notes/windows-build/
notes/libiui-experiments/
notes/libiui-pr-cases/
```

優點:

```text
資料夾看起來很整齊.
同主題內容集中.
```

缺點:

```text
太早正式化.
容易把未驗證的推測寫成教學.
資料夾太多後, 反而不知道哪個是穩定結論.
```

這個方式現在不建議當預設, 但可以在 baseline 後使用.

### Option 3: Case-Based Index

做法:

建立一篇索引 note, 專門列出問題案例:

```text
Problem Case Index

Case 001: PowerShell make shell helper issue
  status: hypothesis
  source notes: ...
  related files: mk/deps.mk
  next probe: ...

Case 002: localtime_r on Windows UCRT
  status: historical evidence
  source notes: ...
  related files: tests/example.c
  next probe: ...
```

優點:

```text
很適合追蹤 PR-worthy issue.
每個 case 都可以轉成 problem -> root cause -> validation.
```

缺點:

```text
比較像 issue tracker.
如果現在 case 還不夠明確, 會有點重.
```

這個方式可以等跑完 baseline 後再做.

### Option 4: Decision Log

做法:

建立一篇簡短 decision log:

```text
Decision Log

2026-06-27:
  Decision:
    Do not use learn/windows-first-attempt as upstream PR directly.
  Reason:
    It is historical evidence, not clean baseline.
  Consequence:
    Need exp/windows-baseline from upstream/main.
```

優點:

```text
快速回顧為什麼走到現在.
避免重複討論同一個決策.
```

缺點:

```text
只能記決策, 不適合承載完整教學.
```

這個可以和 Option 1 搭配.

## Recommended Organization Style

目前建議不要急著開很多新資料夾. 先採用:

```text
timeline notes
  保存發想, 疑問, baseline log, maintainer critique.

current-state-map
  每隔一段時間整理目前所有線路.

topic folders
  只放已經穩定到可以教人的內容.

case index
  等 baseline 產生後再開.

decision log
  如果同一類決策反覆出現, 再補.
```

可以用這個判斷:

```text
還在想?
  timeline note.

已經能教?
  topic folder article.

已經能重現?
  case index.

已經做選擇?
  decision log.
```

## Reasonable Next Options

### Option A: Clean Windows Baseline

這是目前最推薦.

做法:

```sh
git fetch upstream
git worktree add ..\libiui-exp-windows -b exp/windows-baseline upstream/main
```

然後在新 worktree 裡跑:

```text
MSYS2 UCRT64 shell:
  make defconfig
  make
  make check

PowerShell:
  make defconfig
  make
  make check
```

優點:

```text
產生新事實.
能判斷 Windows build 真正第一個 failure.
能決定 windows-build topic 是否值得開.
```

代價:

```text
需要記錄 command output.
可能會遇到環境問題, 要耐心分類.
```

### Option B: Baseline Log Template First

如果還沒準備好跑 build, 可以先開一篇 timeline note 當 log template:

```text
shell
PATH/tool locations
command
expected result
actual result
failure layer
hypothesis
next probe
```

優點是實驗時比較不會亂. 缺點是它不會產生新 evidence.

### Option C: Minimal App Lab First

繼續 `notes/libiui-review/ch06-01-minimal-app-lab.md`, 用 renderer callbacks 建立最小 frame loop.

優點:

```text
更靠近 immediate-mode UI 和 Pixel-Renderer 方向.
可以理解 runtime behavior.
```

缺點:

```text
如果 build baseline 不清楚, 實驗可能被環境問題打斷.
```

### Option D: Read Ch02 Again Before Running Build

如果 build flow 還不穩, 可以先重讀:

```text
ch02-00-c-build-system-foundations.md
ch02-01-libiui-build-flow.md
ch02-02-libiui-build-variants-and-validation.md
```

並邊讀邊用 `make -n V=1` 對照實際 target.

優點是觀念更穩. 缺點是如果一直不跑 baseline, 就沒有新事實可以修正筆記.

## Recommended Default Path

目前推薦順序:

```text
1. 讀這篇 current state map, 先把線路分清楚.
2. 若 notes 已經 commit/push 到 learn/repo-review, 開 clean worktree.
3. 在 exp/windows-baseline 跑 MSYS2 UCRT64 baseline.
4. 再跑 PowerShell baseline.
5. 把結果寫進新的 timeline note.
6. 根據結果決定:
   - 開 notes/windows-build/
   - 回頭做 minimal app lab
   - 或開 fix/* branch 處理小 patch
```

換句話說, 目前最有價值的不是再新增一堆理論筆記, 而是產生第一組 clean baseline evidence.

## Decision

目前決策暫定:

```text
learn/repo-review
  繼續作為筆記和 repo review branch.

windows-build
  暫時維持 timeline notes, 等 baseline 後再正式開 topic folder.

minimal app lab
  是下一條學習路線, 但最好在 Windows build baseline 之後做.

upstream PR
  暫時還沒開始. 必須等 problem/root cause/validation 清楚後, 再從 upstream/main 開 fix/* branch.
```

## 本篇學到的三件事

> [Summary]
> 1. 目前混亂的原因是多條工作流同時存在: Git safety, contribution style, repo review, Windows build, and experiment lab.
> 2. `learn/repo-review` 是個人學習和 review branch, `exp/*` 才適合 baseline 實驗, `fix/*` 才是正式 upstream PR branch.
> 3. 下一步最推薦先做 clean Windows baseline, 產生 command output and failure classification, 再決定是否整理成 `notes/windows-build/` 或開 PR patch.

## Links

```text
notes/AGENTS.md
notes/libiui-git/README.md
notes/libiui-contribution/README.md
notes/libiui-review/README.md
notes/0627_1217_windows-build-strategy-thinking.md
notes/0627_1222_msys-build-environment-thinking.md
```
