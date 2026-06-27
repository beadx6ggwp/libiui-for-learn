# Repository Guidelines

## Project Structure & Module Organization

`libiui` is a C99 immediate-mode UI library with renderer backends selected by Kconfig. Public API headers live in `include/` (`iui.h`, `iui-spec.h`). Core implementation lives in `src/`, with widget modules such as `basic.c`, `input.c`, `layout.c`, and generated MD3 validation files derived from `src/md3-spec.dsl`. Platform ports live in `ports/` (`sdl2.c`, `headless.c`, `wasm.c`). Tests, demos, and benchmarks live in `tests/`; helper scripts live in `scripts/`; build fragments live in `mk/`; configuration files live in `configs/`; browser/WASM assets live in `assets/web/`.

## Build, Test, and Development Commands

- `make defconfig` generates `.config` and `src/iui_config.h` from `configs/defconfig`.
- `make` builds `libiui.a` and enabled demo targets.
- `make -j$(nproc)` builds in parallel on POSIX shells.
- `make check` runs the C unit suite and Python headless tests with an isolated `.build/test` configuration.
- `make check-unit` runs only the compiled C test binary.
- `make check-headless` runs the Python renderer/interaction harness.
- `make indent` applies `clang-format` to `include/`, `src/`, `ports/`, and `tests/`.
- `make clean` removes build outputs; `make distclean` also removes generated config, generated MD3 files, and fetched Kconfig tools.

On Windows, use an MSYS2 UCRT64 shell with GCC, Make, SDL2, Python 3, and Git installed.

## Coding Style & Naming Conventions

Code is C99 and formatted with `.clang-format` based on Chromium: 4-space indentation, Linux braces, right-aligned pointers, and no tabs for C sources. Prefer small module-local helpers marked `static`; keep public names under the `iui_` prefix and public declarations in `include/iui.h`. Test files follow `tests/test-*.c`; generated `.inc` files should be produced by scripts, not hand-edited.

## Testing Guidelines

Add tests beside related coverage in `tests/test-*.c`, using `tests/common.c` and `tests/common.h` helpers where possible. Run `make check` before submitting behavioral changes. For renderer or MD3 spec changes, also review headless output and update `src/md3-spec.dsl` plus generated files through `make gen-md3`.

## Commit & Pull Request Guidelines

Recent history uses concise imperative or scoped subjects, for example `Fix WASM mouse by restoring g_wasm_ctx each frame` and `CI: Add Clang Static Analyzer`. Keep commits focused. Pull requests should describe the changed behavior, list test commands run, link issues when applicable, and include screenshots or headless artifacts for visual/UI changes.

## Agent-Specific Instructions

Avoid unrelated refactors. Do not edit generated files unless the matching generator or DSL change is included. Preserve Kconfig-driven module boundaries and native backend abstractions. When writing Traditional Chinese, use ASCII half-width punctuation in prose. Keep rough ideas and process notes as timestamped files under `notes/`; when a topic becomes stable enough to teach or reuse, synthesize it into a topic folder directly under `notes/`, such as `notes/libiui-git/`, with tutorial-style chapter files.
