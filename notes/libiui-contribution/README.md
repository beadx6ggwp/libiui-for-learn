# libiui Contribution Notes

## 這個資料夾在解決什麼

`notes/libiui-git/` 解決的是 branch, reset, PR branch, force push 這類操作問題. 這個資料夾解決的是另一件事: 當 branch 已經乾淨後, 你要怎麼把一個 patch 寫成 `sysprog21/libiui` maintainer 願意 review 的貢獻.

也就是:

```text
libiui-git
  怎麼安全操作 repo

libiui-contribution
  怎麼把問題, commit, PR body, docs, tests 寫清楚
```

> [Scope]
> 這裡不是一般 open-source 禮儀筆記. 目標是根據 `sysprog21/libiui` 的 commit history, PR 討論, CI 設定, 以及 jserv/sysprog21 的協作風格, 整理出對這個 repo 可用的貢獻寫作方式.

## 建議閱讀順序

```text
ch01-pr-thinking-model.md
  先建立 maintainer review 一個 patch 時在看什麼.

ch02-commit-style.md
  再看 commit subject, commit scope, commit history 要怎麼整理.

ch03-pr-body-and-review.md
  接著看 PR body, review 回應, draft PR, force push review branch 的方式.

ch04-docs-tests-evidence.md
  再看文件, screenshots, test commands, benchmark, CI evidence 要怎麼放.

ch05-pr-operation-playbook.md
  最後把前面 PR 寫作要求接到實際 Git 操作: fetch upstream, 開 fix/* branch, 抽 patch, 檢查 diff, push, review update.
```

## 這組文章的核心模型

好的 PR 不是 "我改了什麼". 好的 PR 是一條可檢查的因果鏈:

```text
problem -> root cause -> minimal change -> validation -> maintainability
```

這條鏈如果斷掉, reviewer 會開始懷疑:

```text
你修的是 root cause 還是 symptom?
這個改動會不會破壞別的 backend or config?
這個文件是 upstream 需要的, 還是個人學習筆記?
這個 commit history 以後還能不能讀?
```

## Source Materials

這組文章主要整理自:

```text
../0626_1726_jserv-pr-guidelines.md
../0627_0051_libiui-development-model.md
../0626_1723_upstream-pr-submission-plan.md
../0626_1731_repo-state-cleanup-strategy.md
../libiui-git/

upstream/main commit history
.github/workflows/main.yml
.build/pr-details/
.build/collab2026.md
```

如果未來 upstream 有新 PR 風格, 可以再回來更新這組文章.
