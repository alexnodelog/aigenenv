# rules/pc-app-python.md

Desktop application, Python only, Qt UI. Single user, local data.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. Only what differs, narrows, or
adds for this archetype. MUST / MUST NOT / SHOULD / MAY per RFC 2119. **Default** = use it;
**Alternative** = allowed with an ADR; **Prohibited** = MUST NOT.

Wrong archetype if the UI is HTML (`pc-app-electron`), if two people share the data (a web
archetype), or if there is no window (a script).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Python | 3.12+, `.python-version`, uv-managed | — | 3.11 and below, system Python |
| Packaging | uv (`pyproject.toml`, `uv.lock`) | — | Poetry, pipenv, conda, bare pip |
| GUI | PySide6 | — | PyQt5/PyQt6, Tkinter, wxPython, Kivy, Flet |
| UI construction | Qt Widgets in Python | Qt Designer `.ui` + `pyside6-uic`, QML | QtWebEngine as the UI |
| Persistence | SQLite via SQLAlchemy 2.0 (`Mapped[]`) | stdlib `sqlite3` for 1–2 tables | QtSql, pickle, shelve |
| Migrations | Ordered functions on `PRAGMA user_version` (§4.3) | — | `create_all()` on an existing DB |
| Paths | `platformdirs` | — | Paths beside the executable or relative to cwd |
| Settings | TOML in `user_config_dir` | JSON | INI, pickle, `QSettings` |
| Background work | `QThread` + worker `QObject` | `concurrent.futures`, results by signal | Blocking the GUI thread |
| Lint / format | Ruff | — | Black + isort + flake8 |
| Type check | mypy strict on `domain/`, `application/` | pyright | — |
| Tests | pytest + pytest-qt + pytest-cov | — | unittest-only |
| Build | `pyside6-deploy`, `mode = standalone` | PyInstaller for local iteration | `onefile` for distribution, cx_Freeze, py2exe, Briefcase |
| Windows package | Inno Setup | — | — |
| macOS package | `.app` from `pyside6-deploy` | — | — |
| Linux package | standalone dir as `.tar.gz` | — | — |

---

## 2. Prerequisites

Verify before the first build; `uv sync` does not reveal them.

| Item | Scope |
|---|---|
| C compiler | System. Windows: VS Build Tools (C++), MSVC on `PATH` via `vcvarsall.bat`. macOS: Xcode CLT. Linux: gcc/clang + ccache |
| nuitka, ordered-set, zstandard, patchelf (Linux) | Project `build` dependency group, locked (§5.1) |
| Inno Setup (`iscc` on `PATH`) | System, Windows only |
| `libxcb-cursor0`, `libxkbcommon-x11-0`, `libegl1`, `libgl1` | System, Linux only. Qt 6.5+ will not start without `libxcb-cursor0` |

---

## 3. Layout

```
<repo>/
├── pyproject.toml          deps, tool config, [tool.uv] required-environments
├── uv.lock
├── .python-version
├── src/<app>/
│   ├── main.py             entry point + composition root; the pyside6-deploy target
│   ├── pysidedeploy.spec   committed
│   ├── domain/             pure Python
│   ├── application/        use cases + ports.py (typing.Protocol)
│   ├── adapters/           SQLite, settings, filesystem, clock
│   │   └── migrations.py
│   ├── ui/                 windows, dialogs, widgets, item models
│   │   └── workers/
│   └── resources/          icons, licence texts
├── installer/windows.iss
├── tests/                  conftest.py, domain/, application/, adapters/, ui/
└── dist/                   gitignored
```

- `main.py` is the only place that creates `QApplication` and constructs adapters.
- `.venv/` and `dist/` MUST stay outside `src/<app>/`: `pyside6-deploy` packages everything
  under the entry file's directory.
- An app with no business rules MAY omit `domain/` and `application/`, and MUST NOT add
  them to match this layout. New top-level dirs under `src/<app>/` need an ADR.

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | stdlib, `domain/` | PySide6, SQLAlchemy, `platformdirs`, `adapters/`, `ui/` |
| `application/` | `domain/`, own `ports.py` | PySide6, SQLAlchemy, `adapters/`, `ui/` |
| `adapters/` | `application/ports.py`, `domain/`, SQLAlchemy, `platformdirs` | PySide6, `ui/` |
| `ui/` | `application/`, `domain/` types, PySide6 | SQLAlchemy, `adapters/` |

- No Qt type (`Signal`, `QObject`, `QDate`, `QModelIndex`) outside `ui/` and `main.py`.
- A slot calls one use case and renders the result. A decision about *what the application
  does*, taken inside a slot, is a defect.
- CI MUST fail if `domain/` or `application/` imports `PySide6` or `sqlalchemy`.

---

## 4. Runtime

### 4.1 GUI thread

- Work that can exceed ~100 ms runs on a worker. Widgets are touched only from the GUI
  thread; workers report by signal.
- Pattern: `QThread` + worker `QObject`, `moveToThread`, start from `thread.started`,
  `finished` → `thread.quit` → `deleteLater`. Hold a reference to thread and worker until
  `finished`.
- On close, stop running workers and wait on the threads before exiting.
- `processEvents()` loops and `time.sleep()` on the GUI thread are PROHIBITED.

### 4.2 Paths

- Data → `user_data_dir`, settings → `user_config_dir`, logs → `user_log_dir`; created on
  first run. The install directory is read-only at runtime.
- Bundled resources are located relative to the package (`importlib.resources`), never to
  cwd.

### 4.3 Persistence

- One `Engine` per process, created in `main.py`. A `Session` MUST NOT cross threads; each
  worker opens and closes its own.
- `PRAGMA foreign_keys=ON` on every connection.
- `adapters/migrations.py` holds an ordered list; startup applies each migration above the
  stored `user_version`, one transaction each, then writes the new version.
- Before migrating an existing file, copy the DB to a timestamped backup beside it.
- `user_version` higher than the app supports → refuse to open, tell the user to update.
- Shipped migrations are append-only. Fixes are new migrations.

### 4.4 Settings and errors

- Settings: typed dataclass, read once at startup, passed explicitly. Unknown keys logged
  and ignored, missing keys defaulted, a malformed file renamed aside and replaced. The app
  MUST start with a broken settings file.
- `tomllib` cannot write. Writing TOML needs a dependency (Blocking) and an ADR; otherwise
  use JSON.
- Released builds have no console: log to a rotating file in `user_log_dir`.
- `main.py` installs `sys.excepthook` before the window opens: log the traceback, show the
  user a plain message plus the log path. A slot that fails MUST leave state unchanged.

---

## 5. Build and package

### 5.1 Lock the toolchain

`pyside6-deploy` otherwise pip-installs Nuitka at build time, which fails or escapes
`uv.lock`.

- Declare nuitka and its helpers in a `build` group. Add `pip` there if `pyside6-deploy`
  still needs it importable.
- The Nuitka version in `pysidedeploy.spec` MUST match `uv.lock`.
- `[tool.uv] required-environments` MUST list every target OS.

### 5.2 Build

```
uv sync --locked --group build
uv run pyside6-deploy -c src/<app>/pysidedeploy.spec -f
```

- Import as `from PySide6.QtWidgets import ...`. A bare `import PySide6` breaks module
  detection and the build ships the wrong Qt libraries.
- `ui_*.py` from `pyside6-uic` is committed and regenerated, never hand-edited.
- `--dry-run` prints the Nuitka command without building.
- Build on every target OS during the first week, as a walking skeleton. Not at release.

### 5.3 Self-check

`main.py --self-check` starts the app, opens a DB in a temp dir, applies all migrations,
constructs the main window, closes it, exits 0 — or non-zero with the reason. Every build
handed to anyone MUST pass it on its own OS. Passing tests are not evidence that the
artifact starts.

### 5.4 Package

| OS | Output |
|---|---|
| Windows | `installer/windows.iss` → `<app>-<version>-setup.exe` |
| macOS | `<app>-<version>-macos.zip` |
| Linux | `<app>-<version>-linux-x86_64.tar.gz` |

- `version` in `pyproject.toml` is the single source; installer, About dialog and artifact
  names read it. `--self-check` prints it.
- Per-user install by default. No user data in the install directory. Uninstall MUST NOT
  delete the data directory without asking.
- Builds are unsigned: SmartScreen and Gatekeeper will warn. Document it for users.

### 5.5 CI

- Push / PR, Linux: `uv sync --locked`, Ruff, mypy, pytest (offscreen), boundary check.
  Install the §2 Linux Qt libraries in the job.
- Version tag, Windows + macOS + Linux matrix: test → build → `--self-check` → package →
  attach to the Release. Publish only if all three succeed. Verify compiler and `iscc`
  before building.
- The three-OS matrix runs on tags only.

---

## 6. Testing

| Layer | Tool | Qt |
|---|---|---|
| `domain/`, `application/` | pytest with fake ports | MUST NOT import PySide6 |
| `adapters/` | pytest against a real SQLite file in `tmp_path` | no |
| `ui/` | pytest-qt `qtbot` | yes |
| Built artifact | `--self-check` | yes |

Most tests are in the first two rows. A UI test proves a click reaches the right use case
and the result is displayed — not the rule itself.

```toml
[tool.pytest.ini_options]
qt_api = "pyside6"
markers = ["integration: needs network or an external system"]
addopts = "-m 'not integration'"
```

```python
# tests/conftest.py — before any QApplication exists
import os
os.environ.setdefault("QT_QPA_PLATFORM", "offscreen")
```

- Every widget created in a test gets `qtbot.addWidget(widget)`. Tests never create their
  own `QApplication`.
- Wait with `qtbot.waitSignal`, `waitSignals`, or `waitUntil`, always with a timeout.
  `time.sleep()` and `qtbot.wait(ms)` to "let things settle" are PROHIBITED.
- Assert what the user sees: text, enabled state, row count, visibility.
- Every `QMessageBox` / `QFileDialog` static call a test reaches MUST be monkeypatched; an
  unpatched modal dialog hangs the suite.
- Worker logic is tested as a plain function on the test thread; only the signal wiring is
  tested across threads.
- Qt warnings captured during a test are investigated, not ignored.

---

## 7. Prohibited

Beyond the Prohibited column in §1:

- Importing `PyQt5` / `PyQt6` anywhere, including tests and scripts.
- QtWebEngine as the application UI.
- PySide6 or SQLAlchemy in `domain/` or `application/`.
- Touching widgets from a non-GUI thread.
- `processEvents()` loops or `time.sleep()` on the GUI thread.
- Writing into the install directory or cwd at runtime.
- `onefile` for a distributed build; distributing an artifact built on a dev machine.
- Hand-editing generated `ui_*.py`; editing a shipped migration.
- Opening a DB whose `user_version` exceeds what the app supports.

---

## 8. Where agents go wrong

Check these first when reviewing an agent's change here.

1. PyQt idioms in PySide6: `pyqtSignal`, `pyqtSlot`, `exec_()`, short-form enums. PySide6
   uses `Signal`, `Slot`, `exec()`, `Qt.AlignmentFlag.AlignLeft`.
2. Blocking work inside a slot.
3. Updating widgets from a worker instead of emitting a signal.
4. `QThread` held in a local variable, collected while running.
5. Validation and decisions written in `ui/` because that is where the button is.
6. One `Session` shared across threads.
7. `QSettings` or a settings file beside the executable.
8. Unpatched modal dialog, or `waitSignal` without a timeout → hung test. A hung test is a
   failure, not something to rerun.
9. `sleep` added to fix a flaky UI test instead of waiting on the actual signal.
10. "Tests pass, so the build works." Only `--self-check` on the artifact shows that.
11. Letting `pyside6-deploy` pip-install Nuitka.
12. `import PySide6` at module top, breaking module detection.
13. Switching to PyInstaller to escape a Nuitka error — that is the architecture switch the
    three-strikes rule forbids. PyInstaller for release needs an ADR.
14. `create_all()` used as a migration.

---

## 9. Definition of Done — additions

Every task here:

- [ ] Boundary check passes
- [ ] No new GUI-thread blocking; long work on a worker, results by signal
- [ ] UI tests use `addWidget`, timed waits, no unpatched modal

Schema change:

- [ ] New migration appended, none edited; tested from the previous version with the backup
      file created

Packaging, dependencies, resources, or `main.py` change:

- [ ] `--self-check` passes on at least one OS; the others reported as not run
