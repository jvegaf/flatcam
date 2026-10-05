# AGENTS.md

FlatCAM Evo — PyQt6 CAM app (Gerber/Excellon → G-Code) for PCB milling.
Architecture details live in `CLAUDE.md`; read it before touching structure. This file covers only what it gets wrong or omits.

## Environment: docs are stale, trust the toolchain

`README.md` and `CLAUDE.md` both say **Python 3.11 + conda/mamba + `requirements.txt`**. That is out of date.

The repo has migrated to **uv** (commit `618fe8e3 uv environment`):

- `pyproject.toml` → `requires-python = ">=3.14"`, build backend `uv_build`
- `.python-version` → `3.14`, committed
- `uv.lock` + `.venv/` present; `environment.yml`/`requirements.txt` are legacy leftovers

```bash
uv sync                          # install/update deps from uv.lock
uv run --no-sync python flatcam.py   # run the app
uv run --no-sync python tests/test_gerber_parser.py
```

`gdal` and `rasterio` have no wheels for every platform — if `uv sync` fails to build, that is why. Don't "fix" it by downgrading Python.

## Two entrypoints; only one is real

- **`flatcam.py`** (repo root) — the actual application entrypoint. Must be run from the repo root; it does flat imports (`from appMain import App`) because `camlib.py`, `appMain.py`, `appDatabase.py` etc. live at the root, not in a package.
- **`src/flatcam/__init__.py`** — a `uv_build` scaffold stub. Its `main()` only prints `Hello from flatcam!`. The `[project.scripts] flatcam = "flatcam:main"` console script therefore does **not** launch the app. Don't treat the repo as `src/`-layout and don't wire new code to that entrypoint.

## Tests: run individually, one is broken

There is **no pytest installed, no `conftest.py`, no tox/CI** (`.github/` does not exist). Tests are `unittest`-style with a hand-rolled `run_all_tests()` + `sys.exit()` `__main__` block, so pytest collection semantics do not apply.

```bash
uv run --no-sync python tests/test_appmain.py          # source-text assertions, passes
uv run --no-sync python tests/test_gerber_parser.py    # 18 passed
uv run --no-sync python tests/test_vispy_batch.py      # 19 tests, OK
uv run --no-sync python tests/test_pdf_hole_detection.py
```

**`test_pdf_hole_detection.py` is already broken** — line 5 hardcodes `sys.path.insert(0, 'D:/1.Development/FlatCAM_EVO/.worktrees/refactor-parsepdf')`, a developer's Windows path. It exits 1 with `ModuleNotFoundError: No module named 'appParsers'`. This is pre-existing, not your regression. `CLAUDE.md` wrongly lists it as a working command.

`tests/test_files/` holds fixture Gerbers (`simple_line.gbr`, `region_test.gbr`, `flash_test.gbr`). `tests/test_appmain.py` asserts against **regex-extracted `appMain.py` source text**, so cosmetic refactors that move methods will break it.

## i18n: the convention nobody can guess

All 174 UI-bearing modules open with this exact block, which installs `_` into `builtins` via `gettext`:

```python
import gettext
import appTranslation as fcTranslate
import builtins

fcTranslate.apply_language('strings')
if '_' not in builtins.__dict__:
    _ = gettext.gettext
```

- Wrap user-facing strings in `_("...")`. There is no `tr()` helper — `rg 'tr\('` returns nothing.
- **Never wrap a value that gets persisted, compared, serialized, or emitted as G-Code.** Existing code carries the comment `protection against having this translated or loading a project with translated values` wherever this matters. Pattern to copy: `addItems([_("Around"), _("Over")])` for display, then `idx = 0 if area_dict["strategy"] == 'around' else 1` against the **raw** stored string.
- Translations live in `locale/<lang>/LC_MESSAGES/`, template in `locale_template/strings.pot`.
- Runtime settings are **not** in the repo: QSettings org `Open Source` / app `FlatCAM_EVO`, plus `~/.FlatCAM` for logs and data.

## Plugin layout: filenames differ from the docs

`CLAUDE.md` says MVC plugins use `Tool.py` / `ToolUI.py` / `ToolGen.py`. **Those files do not exist** — `find appPlugins -name ToolUI.py` returns nothing. The actual split, one folder per tool:

```
appPlugins/ToolPaint/  → Paint.py    PaintUI.py    PaintGen.py
appPlugins/ToolNCC/    → Ncc.py      NccUI.py      NccGen.py
appPlugins/ToolPdf/    → PdfImport.py             (not split)
```

Everything else is still a single `ToolName.py` with controller + UI classes together. New tools should follow the `<Name>/<Name>.py` convention and match the existing prefix.

All tools inherit `AppTool` (`appTool.py`); lifecycle is `__init__(app)` → `install()` → `run()` → `connect_signals()` / `set_tool_ui()`, with `ui_connect()` / `ui_disconnect()` kept in pairs. Editors mirror this under `appEditors/*/`. Preprocessors auto-register via the `ABCPreProcRegister` metaclass — do not manually register them.

## Repo hygiene gotchas

- **These files are enormous — never read them whole.** `appMain.py`, `camlib.py`, and `appDatabase.py` are each 150–350 KB; `CHANGELOG.md` is ~500 KB. Use `rg -n` with bounded context or grep the symbol you need.
- **`errors.txt` at the repo root is a stray untracked crash log**, referenced nowhere in the code and not gitignored. Don't treat it as input, and don't commit it.
- `appTranslation.py` is imported by 174 files — the most-imported module. Changing it breaks the whole UI.
- Current work happens on branch `dev`; history is Bitbucket-era (`Merged in <branch> (pull request #NN)`) though remotes are GitHub.
- `Makefile` targets (`install`, `remove`) do a **system-wide or `~/.local/share/applications` install** and will `rm -rf /usr/share/flatcam-beta`. They are packaging, not dev tasks — don't reach for them while iterating.