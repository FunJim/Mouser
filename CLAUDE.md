# Repository Guidelines

## What this is

Mouser is a cross-platform (Windows / macOS / Linux) Logitech mouse remapper that speaks HID++ 2.0 directly. PySide6 (Qt6 / QML Material) UI + a Python core engine. No external package manager — pip + venv (`.venv/`).

## Commands

- **Run from source:** `python main_qml.py` (add `--start-hidden` to launch into the tray)
- **Tests:** `python -m unittest discover -s tests` (this project uses **unittest, not pytest**)
- **Single test:** `python -m unittest tests.test_<module>.<TestClass>.<test_method>`
- **Compile-check:** `python -m py_compile <file>` — CI runs this on `main_qml.py`, `core/*.py`, `ui/*.py`
- **QML lint:** `pyside6-qmllint <file>.qml` (CI runs this on every `.qml`)
- **Build Windows:** `build.bat` (or `build.bat --clean`)
- **Build macOS:** `./build_macos_app.sh` — set `MOUSER_SIGN_IDENTITY=<SHA-1>` for real signing; unset = ad-hoc
- **Build Linux:** `pyinstaller Mouser-linux.spec --noconfirm` (needs `libhidapi-dev`)

There is **no Python formatter or linter configured** — CI only runs `py_compile` + `qmllint` + the test suite. Don't introduce Black/Ruff/etc. without asking.

## Verification before declaring done

Run the unittest suite and `py_compile` any file you changed. Both must pass.

## Architecture pointers

- `main_qml.py` is the entry point (~1300 lines). It orchestrates startup, tray, Qt Material, and PyInstaller path quirks.
- `core/` — HID++ device handling, per-platform mouse hooks, app detection, config, engine.
- `ui/` — PySide6 backend bridge + QML.
- `tests/` — 25 unittest files, heavy use of `unittest.mock` and `SimpleNamespace` stubs; no fixtures, no test data files.

## Things that bite if you don't know them

- **Mouse hooks are per-platform**: Windows `WH_MOUSE_LL`, macOS `CGEventTap`, Linux `evdev` + `uinput`. Changing hook behavior usually means touching all three implementations and their tests (which are stubbed per-platform).
- **HID++ settings must be re-applied on every reconnect / wake.** SmartShift (`0x2110`/`0x2111`), DPI, etc. are replayed whenever a device reconnects. New device settings need to participate in this replay path.
- **Config schema is versioned.** `core/config.py` migrates older configs forward. Schema changes require a new migration step — don't break old configs.
- **PyInstaller bundling.** `main_qml.py` uses `getattr(sys, "frozen", False)` to switch path resolution (macOS `.app/Contents/Resources` vs. Windows/Linux `_internal/`) and manually sets `QML2_IMPORT_PATH` / `QT_PLUGIN_PATH`. If you touch startup paths, test both source and frozen builds.
- **Startup is deferred.** Core engine start is wrapped in `QTimer.singleShot(0, ...)` so the window renders before HID++ binding (which can block). Don't move HID++ binding back into the synchronous startup path.
- **macOS Accessibility permission is required** for the keystroke/gesture path — see `readme_mac_osx.md`. Logitech Options+ must not be running simultaneously (HID++ conflict).
- **Debugging hangs:** `kill -SIGUSR1 <pid>` dumps all thread stacks (handler is registered in `main_qml.py`).

## Commit / PR style

Scoped conventional commits — `feat(scope): …`, `fix(macos): …`, `test(core): …`, `build: …`. Keep the scope. Prefer small, focused commits over bundled ones. Recent history is a good reference (`git log --oneline`).

## Per-subsystem instructions

Subdirectory CLAUDE.md files (e.g., `core/CLAUDE.md`, `ui/CLAUDE.md`) can be added later for module-specific guidance — they load automatically when Claude works in those directories. Ask if you want me to scaffold one.
