# rules/pc-app-python.md

Desktop app, Python only, Qt UI, single user, local data.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. Only what differs, narrows, or
adds. RFC 2119 keywords. **Default** = use it; **Alternative** = allowed with an ADR;
**Prohibited** = MUST NOT. The Prohibited column in §1 is the full prohibition list for this
archetype; the rest of the file states the rest as MUST / MUST NOT in place.

Wrong archetype if the UI is HTML (`pc-app-electron`), if two people share the data (a web
archetype), or if there is no window (a script).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Python | 3.12+, `.python-version`, uv-managed | — | 3.11 and below, system Python |
| Packaging | uv (`pyproject.toml`, `uv.lock`) | — | Poetry, pipenv, conda, bare pip |
| GUI | PySide6 | — | PyQt5/PyQt6 anywhere, including tests and scripts; Tkinter, wxPython, Kivy, Flet |
| UI construction | Qt Widgets in Python | Qt Designer `.ui` + `pyside6-uic` (generated `ui_*.py` committed, never hand-edited), QML | QtWebEngine as the UI |
| Persistence | SQLite via SQLAlchemy 2.0 (`Mapped[]`) | stdlib `sqlite3` for 1–2 tables | QtSql, pickle, shelve |
| Migrations | Ordered functions on `PRAGMA user_version` (§4.3) | — | `create_all()` on an existing DB; editing a shipped migration |
| Paths | `platformdirs` | — | Paths beside the executable or relative to cwd; writing into the install dir at runtime |
| Settings | JSON in `user_config_dir` | TOML, read-only (`tomllib` cannot write) | INI, pickle, `QSettings` |
| User credentials | `keyring` (OS credential store), declared as a runtime dependency | — | Credentials in the settings file, the DB, or a log line |
| Background work | `QThread` + worker `QObject` | `concurrent.futures`, results by signal | Blocking the GUI thread; touching widgets off it; killing a running `QThread` |
| Lint / format | Ruff | — | Black + isort + flake8 |
| Type check | mypy strict on `domain/`, `application/` | pyright | — |
| Tests | pytest + pytest-qt + pytest-cov | — | unittest-only |
| Build | `pyside6-deploy`, `mode = standalone` | PyInstaller for local iteration | `onefile` for distribution; cx_Freeze, py2exe, Briefcase; distributing a dev-machine build |
| Windows package | Inno Setup | — | — |
| macOS package | `.app` from `pyside6-deploy` | — | — |
| Linux package | standalone dir as `.tar.gz` | — | — |

---

## 2. Prerequisites

Verify before the first build; `uv sync` does not install or reveal these.

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
│   ├── adapters/           SQLite, settings, credentials, filesystem, clock
│   │   └── migrations.py
│   ├── ui/                 windows, dialogs, widgets, item models
│   │   └── workers/
│   └── resources/          icons, licence texts
├── installer/windows.iss
├── tests/                  conftest.py, test_boundaries.py, domain/, application/, adapters/, ui/
└── dist/                   gitignored
```

- `main.py` is the only place that creates `QApplication` and constructs adapters.
- `.venv/` and `dist/` MUST stay outside `src/<app>/`: `pyside6-deploy` packages everything
  under the entry file's directory.
- An app with no business rules MAY omit `domain/` and `application/`, and MUST NOT add them
  to match this layout. A new top-level dir needs an ADR only if it changes the boundaries
  below.

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | stdlib, `domain/` | PySide6, SQLAlchemy, `platformdirs`, `adapters/`, `ui/` |
| `application/` | `domain/`, own `ports.py` | PySide6, SQLAlchemy, `adapters/`, `ui/` |
| `adapters/` | `application/ports.py`, `domain/`, SQLAlchemy, `platformdirs` | PySide6, `ui/` |
| `ui/` | `application/`, `domain/` types, PySide6 | SQLAlchemy, `adapters/` |

- No Qt type (`Signal`, `QObject`, `QDate`, `QModelIndex`) outside `ui/` and `main.py`.
- A slot calls one use case and renders the result. A decision about *what the application
  does*, taken inside a slot, is a defect.
- `tests/test_boundaries.py` walks the imports of `domain/` and `application/` and fails on
  `PySide6` or `sqlalchemy`. That test is the boundary check the CI job and the DoD mean.

---

## 4. Runtime

### 4.1 GUI thread

- Work that can exceed ~100 ms runs on a worker. Widgets are touched only from the GUI
  thread; workers report by signal.
- Pattern: `QThread` + worker `QObject`, `moveToThread`, start from `thread.started`,
  `finished` → `thread.quit` → `deleteLater`. Hold a reference to both until `finished`.
- On close, signal workers to stop and wait with a timeout. If it expires, the close is
  **cancelled**: ignore the close event, keep the window alive, tell the user the app is
  finishing, close for real when the last worker reports `finished`.
- A single file write replaces the target atomically: temp file on the **same filesystem**,
  then rename, original left intact if the rename fails. A single DB write is one
  transaction. Work spanning both MUST define its own recovery order — neither mechanism
  makes a cross-store operation atomic.
- `processEvents()` loops and `time.sleep()` on the GUI thread are PROHIBITED.

### 4.2 Paths

- Data → `user_data_dir`, settings → `user_config_dir`, logs → `user_log_dir`; created on
  first run. The install directory is read-only at runtime.
- Bundled resources are located relative to the package (`importlib.resources`), never cwd.

### 4.3 Persistence

- One `Engine` per process, created in `main.py`. A `Session` MUST NOT cross threads; each
  worker opens and closes its own.
- `PRAGMA foreign_keys=ON` and a non-zero `busy_timeout` on every connection. A write that
  still hits `SQLITE_BUSY` fails as an error; it MUST NOT be retried by re-running the use
  case, which would duplicate work.
- `adapters/migrations.py` holds an ordered list; startup applies each migration above the
  stored `user_version`, with the version write in the **same** transaction, so a crash
  cannot leave the version and the schema disagreeing.
- Before migrating an existing file, back it up beside the DB with
  `sqlite3.Connection.backup`, not a file copy — in WAL mode the `.db` file alone is not a
  consistent snapshot. Verify the backup opens and reports the old version first.
- A failed migration: roll back, keep original and backup, do not advance `user_version`,
  refuse to open, and offer the user a way to open the backup's folder.
- `user_version` higher than the app supports → refuse to open, tell the user to update.
- Shipped migrations are append-only. Fixes are new migrations.

### 4.4 Settings, credentials, errors

- Settings: typed dataclass, read once at startup, passed explicitly. The file is external
  input — wrong types and values outside the bounds declared on the dataclass take defaults
  with a logged warning, as do unknown and missing keys.
- A malformed file is renamed aside (never deleted) and replaced. If the rename or write
  fails, run on in-memory defaults and tell the user settings cannot be saved. The app MUST
  start either way.
- A desktop app holds no deployment secrets, so nothing here comes from the environment.
  Credentials the user supplies at runtime go to `keyring`. If the credential store is
  unavailable, features needing a secret are disabled with a clear error — never stored
  somewhere else instead.
- Released builds have no console: log to a rotating file in `user_log_dir`.
- `main.py` installs `sys.excepthook` before the window opens: log the traceback, show the
  user a plain message of what failed and what to do. The dialog MUST NOT contain the
  traceback, SQL, internal identifiers, or a file path — it offers a control that opens the
  log directory instead.
- A failed slot MUST leave no partial write in the DB or on disk. The window MAY be left
  showing stale data as long as the user is told the action failed.

---

## 5. Build and package

### 5.1 Lock the toolchain

`pyside6-deploy` otherwise pip-installs Nuitka at build time, which fails under uv or
escapes `uv.lock`.

- Declare nuitka and its helpers in a `build` group; add `pip` there if `pyside6-deploy`
  still needs it importable. The version in `pysidedeploy.spec` MUST match `uv.lock`.
- `[tool.uv] required-environments` MUST list every declared target OS.

### 5.2 Build

```
uv sync --locked --group build
uv run pyside6-deploy -c src/<app>/pysidedeploy.spec -f
```

- Import as `from PySide6.QtWidgets import ...`. A bare `import PySide6` breaks module
  detection and the build ships the wrong Qt libraries.
- `--dry-run` prints the Nuitka command without building.
- Build once on every declared target OS in the first week, as a walking skeleton. Leaving
  the first cross-platform build until release is PROHIBITED.

### 5.3 Self-check

`main.py --self-check` starts the app, opens a DB in a temp dir, applies all migrations,
runs the event loop until the window reports ready, closes it, exits 0 — or non-zero with
the reason. Startup and shutdown each run under a timeout and an overall deadline; a
timeout is a failure, not a pass. It MUST NOT read or write the real `user_data_dir`.

Passing tests are not evidence that the artifact starts.

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
- Version tag, one job per declared target OS: test → build → `--self-check` → package →
  attach to the Release. Publish only if every declared OS succeeds. Verify the compiler,
  and `iscc` on Windows, before building.
- The matrix runs on tags only. Local builds are for the developer's own machine; anything
  handed to another person comes from a tag and passes `--self-check` on its own OS.

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

- Every widget created in a test gets `qtbot.addWidget`. Tests never create their own
  `QApplication`.
- Wait with `qtbot.waitSignal`, `waitSignals`, or `waitUntil`, always with a timeout.
  `time.sleep()` and `qtbot.wait(ms)` to "let things settle" are PROHIBITED.
- Assert what the user sees: text, enabled state, row count, visibility.
- Every `QMessageBox` / `QFileDialog` static call a test reaches MUST be monkeypatched; an
  unpatched modal dialog hangs the suite.
- Worker logic is tested as a plain function on the test thread; only the signal wiring is
  tested across threads.
- Qt warnings captured during a test are investigated, not ignored.

---

## 7. Where agents go wrong

Check these first when reviewing a change here.

1. PyQt idioms in PySide6: `pyqtSignal`, `pyqtSlot`, `exec_()`, short-form enums. PySide6
   uses `Signal`, `Slot`, `exec()`, `Qt.AlignmentFlag.AlignLeft`.
2. Blocking work inside a slot; widgets updated from a worker instead of by signal.
3. `QThread` held in a local variable, collected while running.
4. Validation and decisions written in `ui/` because that is where the button is.
5. One `Session` shared across threads.
6. `QSettings` or a settings file beside the executable.
7. Unpatched modal dialog, or `waitSignal` without a timeout → hung test. A hung test is a
   failure, not something to rerun; `sleep` is not the fix for a flaky one.
8. "Tests pass, so the build works." Only `--self-check` on the artifact shows that.
9. Letting `pyside6-deploy` pip-install Nuitka; `import PySide6` at module top.
10. Switching to PyInstaller to escape a Nuitka error — the three-strikes rule's MUST NOT
    list covers switching to escape an error.
11. `create_all()` used as a migration.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes
- [ ] No new GUI-thread blocking; long work on a worker, results by signal
- [ ] UI tests use `addWidget`, timed waits, no unpatched modal

Schema change:

- [ ] New migration appended, none edited; tested from the previous version, and tested
      failing mid-migration — original schema and rows unchanged, version unchanged, backup
      opens

Packaging, dependencies, resources, or `main.py` change:

- [ ] `--self-check` passes on the developer's OS during the task; anything handed to
      another person goes through §5.5 and passes on each declared target OS
