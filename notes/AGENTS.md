# Notes Guidelines

## Scope

This folder is for rough ideas, learning traces, design notes, and implementation thinking related to `libiui`, immediate-mode UI, software rendering, and Pixel-Renderer integration. Notes may be exploratory, but they should still be easy to search and reuse later.

## Folder Organization

Use two note types:

- `notes/MMDD_HHMM_short-title.md` for timeline notes, rough ideas, questions, process records, and temporary reasoning.
- `notes/<topic>/` for tutorial-style articles synthesized from multiple timeline notes. Use this when a topic is stable enough to teach or reuse, such as `notes/libiui-git/`, `notes/libiui-contribution/`, `notes/libiui-review/`, or `notes/windows-build/`.

Do not force every rough note into a topic folder immediately. First record thoughts in timestamped notes. When a topic becomes clear and reusable, synthesize it into a teaching article under the matching topic folder with links back to the source notes when useful.

## Filename Convention

Create new timeline notes with this format:

```text
MMDD_HHMM_short-title.md
```

Examples:

```text
0626_2130_draw-list-clipping.md
0626_2145_input-state-model.md
0627_1015_pixel-renderer-port-plan.md
```

Rules:

- Use Taiwan local time (`Asia/Taipei`) for `MMDD_HHMM`.
- Use 24-hour time for `HHMM`.
- Keep the title short, descriptive, and lowercase.
- Prefer ASCII words joined by hyphens in the title segment.
- Avoid spaces, punctuation-heavy names, and vague titles such as `note.md` or `idea.md`.
- If two notes are created in the same minute, add a numeric suffix: `0626_2130_draw-list-clipping-2.md`.
- Topic-folder articles may use chapter-style or topic-based filenames without timestamps, for example `notes/libiui-git/ch01-fork-origin-upstream.md`, `notes/libiui-contribution/ch02-commit-style.md`, or `notes/windows-build/README.md`.

## Note Structure

Start each note with a clear title:

```markdown
# Draw List Clipping
```

Use short sections when helpful:

- `## Question` for the problem being investigated.
- `## Context` for repo files, prior decisions, or constraints.
- `## Reasoning` for first-principles analysis and failed naive approaches.
- `## Decision` for the current conclusion.
- `## Follow-ups` for concrete next actions.

## Writing Style

Write notes in Traditional Chinese by default. Use ASCII half-width punctuation in Chinese prose. Keep code identifiers, file paths, APIs, compiler flags, and algorithm names in English. Prefer concrete references such as `src/draw.c`, `ports/sdl2.c`, or `tests/test-input.c` over abstract descriptions.

When comparing approaches, state the constraint first, then explain why a naive method fails, then record the better direction.

For topic-folder articles, write primarily in Traditional Chinese as complete teaching articles, not short summaries. Use a learning-note style: problem-driven opening, scope box, naive idea, why it fails, correct model, concrete examples, ASCII diagrams, commands, a three-point summary, and source-note links.

For Git operation teaching, include ASCII diagrams for branch pointers, working tree, index, local branches, remote-tracking branches, and remote branches. The diagram should make clear which pointer moves, which files are only in `WT`, which files are staged in `Index`, and whether the command affects `origin`, `upstream`, or only local state.

When proposing next steps, avoid presenting a single path as if it is the only option. Provide 2-4 viable choices, explain when each is appropriate, note the tradeoffs, and then identify the recommended default path with the reason.

For commits, suggest the staging scope and commit message, but let the user run `git add`, `git commit`, and `git push` unless they explicitly ask Codex to perform those commands.
