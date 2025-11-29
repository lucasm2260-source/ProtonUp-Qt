<!-- Copilot instructions for contributors and AI agents -->
# ProtonUp-Qt — Copilot instructions

Purpose: Quickly orient an AI coding agent to the repository layout, conventions, runtime flows, and developer commands so it can be productive immediately.

- Quick dev commands
  - Install deps: `python -m pip install -r requirements.txt`
  - Run from source: `python -m pupgui2` (or install editable: `python -m pip install -e .` to enable the `protonup-qt` entry point)
  - Run tests: `pytest -q` (test files live in `tests/`)
  - Build AppImage: install `appimage-builder` and run `appimage-builder` (see `AppImageBuilder.yml`)

- Big picture (what to read first)
  - `pupgui2/pupgui2.py`: The main GUI orchestrator. `MainWindow` loads the `.ui` via `pkgutil.get_data`, wires dialogs and threads, and coordinates installs.
  - `pupgui2/ctloader.py`: Dynamically discovers `ctmod_*` modules under `pupgui2/resources/ctmods/` and constructs ctobj installers.
  - `pupgui2/resources/ctmods/ctmod_*.py`: Each compatibility tool provides `CT_NAME`, `CT_LAUNCHERS`, `CT_DESCRIPTION` and a `CtInstaller` (commonly subclassing a generic installer like `ctmod_00protonge.CtInstaller`). Example: `ctmod_protonem.py`.
  - `pupgui2/util.py`, `pupgui2/constants.py`: Utility helpers and constants used across the app (install locations, feature flags, TEMP_DIR, etc.).

- Key runtime flows (how an install happens)
  1. User triggers add/install in the UI (see `btn_add_version_clicked` in `pupgui2/pupgui2.py`).
  2. `PupguiInstallDialog` emits a signal with the selected compat tool.
  3. `MainWindow.install_compat_tool` enqueues the tool and notifies `InstallWineThread`.
  4. `InstallWineThread` invokes `ctobj['installer'].get_tool(...)` (implemented by each `CtInstaller`).
  5. Progress is communicated via Qt signals such as `download_progress_percent` and optional `message_box_message`.

- Project-specific conventions & patterns
  - ctmods naming: `ctmod_<id>.py` under `pupgui2/resources/ctmods/`. Each module must export `CT_NAME` and a `CtInstaller` class.
  - `CtInstaller` subclasses often reuse a base installer (see `ctmod_00protonge.py`), overriding `CT_URL`, `CT_INFO_URL`, and format specifics.
  - UI files are packaged and loaded with `pkgutil.get_data(__name__, 'resources/ui/<file>.ui')`. Do not assume file paths on disk in runtime.
  - Inter-thread communication uses Qt signals/slots. When adding new signals, ensure they are connected in `MainWindow` (see loader loop in `__init__`).
  - Translations: `.ts` sources in `i18n/`, compiled `.qm` in `pupgui2/resources/i18n/`.

- Integration points & external dependencies
  - Windows/Steam parsing: `pupgui2/steamutil.py` uses the `vdf` and `steam` packages (git-based deps in `setup.cfg`).
  - Desktop integration: `pupgui2/dbusutil.py` uses DBus — Linux-only behavior; CI and packaging target POSIX (see `setup.cfg` classifiers).
  - Many ctmods fetch from GitHub/GitLab release APIs — maintainers configured tokens via env vars or config helpers (`PUPGUI_GHA_TOKEN`, `PUPGUI_GLA_TOKEN`).

- Tests & fixtures
  - Tests are pytest-based in `tests/` and use fixtures in `fixtures/` (e.g. `fixtures/networkutil`). Run `pytest -q`.
  - When changing network calls, update or mock fixtures accordingly.

- When editing or adding a ctmod
  - Follow the existing `ctmod_*` modules: export `CT_NAME`, `CT_LAUNCHERS`, `CT_DESCRIPTION`, and provide a `CtInstaller` class.
  - Prefer subclassing an existing installer (e.g. `ctmod_00protonge.CtInstaller`) unless behavior differs fundamentally.
  - Use Qt signals for progress/messages so the `MainWindow` UI can respond without tight coupling.

- Files to inspect when debugging or extending
  - `pupgui2/pupgui2.py` — main app flow and UI wiring
  - `pupgui2/ctloader.py` — plugin discovery and loading
  - `pupgui2/resources/ctmods/` — examples of compatibility tool modules
  - `pupgui2/util.py` and `pupgui2/constants.py` — install locations and helpers
  - `pupgui2/pupgui2installdialog.py` — dialog logic for selecting versions/releases

- Edge cases and platform notes
  - The project targets Linux (Flatpak/AppImage). DBus and Flatpak detections assume POSIX-style environments; behavior on Windows may differ or be limited.
  - Resource loading via package data means files must be included in packaging (`include_package_data = True` in `setup.cfg`).

- Next steps and requests
  - If you want a `ctmod` template file or an expanded `CtInstaller` API summary, say which you'd prefer and I'll add it.

If anything above is unclear or you'd like more detail (for example: a summary of the `CtInstaller` API surface, or a sample ctmod template), tell me which area to expand and I'll iterate.
