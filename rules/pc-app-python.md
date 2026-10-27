# rules/pc-app-python.md

Desktop app, Python only, Qt UI, single user, local data.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

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
| Settings | TOML in `user_config_dir`, read at startup only | JSON, only if the app writes settings back | INI, pickle, `QSettings` |
| User credentials | `keyring`, declared as a runtime dependency | — | Credentials in the settings file, the DB, or a log line |
| Background work | `QThread` + worker `QObject` | `concurrent.futures`, results by signal | Blocking the GUI thread; touching widgets off it; killing a running `QThread` |
| Lint / format | Ruff | — | Black + isort + flake8 |
| Type check | mypy strict on `domain/`, `application/` | pyright | — |
| Tests | pytest + pytest-qt + pytest-cov | — | unittest-only |
| Build | `pyside6-deploy`, `mode = standalone` | PyInstaller for local iteration | `onefile` for distribution; cx_Freeze, py2exe, Briefcase; distributing a dev-machine build |
| Windows package | Inno Setup | — | — |
| macOS package | `.app` from `pyside6-deploy` | — | — |
| Linux package | AppImage via `appimagetool` | `.tar.gz` of the standalone dir | — |

---

## 2. Prerequisites

`uv sync` neither installs nor reveals these. Setup: skill `pyside-release`.

| Item | Scope |
|---|---|
| C compiler | System. Windows: VS Build Tools (C++). macOS: Xcode CLT. Linux: gcc or clang |
| Nuitka and its helpers | Project `build` dependency group, locked (§5.1) |
| Inno Setup (`iscc`) | System, Windows only |
| `appimagetool` | System, Linux only, unless the ADR chose `.tar.gz`. A tagged release, never `continuous`; version and SHA-256 recorded in `DECISIONS.md` |
| Qt system libraries: `libxcb-cursor0`, `libxkbcommon-x11-0`, `libegl1`, `libgl1` | System, Linux only |

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
├── scripts/package.py
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
| `adapters/` | `application/ports.py`, `domain/`, SQLAlchemy | PySide6, `ui/`, `platformdirs` |
| `ui/` | `application/`, `domain/` types, PySide6 | SQLAlchemy, `adapters/` |

- No Qt type (`Signal`, `QObject`, `QDate`, `QModelIndex`) outside `ui/` and `main.py`.
- A slot calls one use case and renders the result. A decision about *what the application
  does*, taken inside a slot, is a defect.
- `tests/test_boundaries.py` walks the imports of `domain/` and `application/` and fails on
  `PySide6` or `sqlalchemy`. That test is the boundary check CI and §8 refer to.

---

## 4. Runtime

### 4.1 GUI thread

- Work that can exceed ~100 ms runs on a worker. Widgets are touched only from the GUI
  thread; workers report by signal.
- Pattern: `QThread` + worker `QObject`, `moveToThread`, start from `thread.started`,
  `finished` → `thread.quit` → `deleteLater`. Hold a reference to both until `finished`.
- Every blocking call inside a worker — socket, subprocess, lock, queue — takes its own
  timeout. A worker that cannot guarantee it terminates MUST NOT be written; a running
  `QThread` MUST NOT be killed or destroyed.
- On close, signal workers to stop and wait with a timeout. If it expires, the close is
  **cancelled**: ignore the close event, keep the window alive, tell the user the app is
  finishing, close for real when the last worker reports `finished`.
- A single file write replaces the target atomically: temp file on the **same filesystem**,
  then rename, original left intact if the rename fails. A single DB write is one
  transaction. Work spanning both MUST define its own recovery order.
- `processEvents()` loops and `time.sleep()` on the GUI thread are PROHIBITED.

### 4.2 Paths

- `main.py` is the only caller of `platformdirs`. It resolves `user_data_dir`,
  `user_config_dir` and `user_log_dir`, creates them if missing, and passes them to the
  adapters as one frozen `Paths` dataclass. An adapter that resolves its own path cannot be
  redirected, and `--self-check` (§5.3) depends on redirecting it.
- The install directory is read-only at runtime.
- Bundled resources are located relative to the package (`importlib.resources`), never cwd.

### 4.3 Persistence

- One `Engine` per process, created in `main.py`. A `Session` MUST NOT cross threads; each
  worker opens and closes its own.
- `PRAGMA foreign_keys=ON` and a non-zero `busy_timeout` on every connection. A write that
  still hits `SQLITE_BUSY` fails as an error; it MUST NOT be retried by re-running the use
  case.
- Each migration and its `user_version` write run in the **same** transaction.
- Before migrating an existing file, back it up beside the DB with
  `sqlite3.Connection.backup`; a file copy is PROHIBITED. Verify the backup opens and
  reports the old version before migrating.
- A failed migration: roll back, keep original and backup, do not advance `user_version`,
  refuse to open, and offer the user a way to open the backup's folder.
- `user_version` higher than the app supports → refuse to open, tell the user to update.
- Shipped migrations are append-only. Fixes are new migrations.

### 4.4 Settings, credentials, errors

- Settings: typed dataclass, read once at startup, passed explicitly. Wrong types, values
  outside the bounds declared on the dataclass, and unknown or missing keys take defaults
  with a logged warning.
- A malformed file is renamed aside (never deleted) and replaced. If the rename or write
  fails, the app runs on in-memory defaults and tells the user settings cannot be saved. The
  app MUST start either way.
- A desktop app has no deployment secrets and ships no `.env.example`. Every secret it holds
  is an end-user credential and goes to `keyring`.
- Released builds have no console: log to a rotating file in `user_log_dir`.
- `main.py` installs `sys.excepthook` before the window opens. It logs the traceback and
  shows the user what failed and what to do, with a control that opens the log directory.
- A failed slot MUST leave no partial write in the DB or on disk. The window MAY show stale
  data as long as the user is told the action failed.

---

## 5. Build and package

Commands, per-OS steps and CI job setup: skill `pyside-release`.

### 5.1 Lock the toolchain

- Nuitka and its helpers are declared in a locked `build` dependency group.
  `pyside6-deploy` MUST NOT install packages at build time. The Nuitka version in
  `pysidedeploy.spec` MUST match `uv.lock`.
- `[tool.uv] required-environments` MUST list every declared target OS.

### 5.2 Build

- PySide6 is imported by submodule (`from PySide6.QtWidgets import ...`). A bare
  `import PySide6` is PROHIBITED: the build ships the wrong Qt libraries.
- **Declared target OSes** are the ones `README.md` lists under its install section. That
  list is the single source for `required-environments`, the first-week builds, and the CI
  matrix; adding or dropping an OS means editing it there first.
- Build once on every declared target OS in the first week, as a walking skeleton — on CI
  where the developer has no such machine. Leaving the first cross-platform build until
  release is PROHIBITED.

### 5.3 Self-check

`main.py --self-check` starts the app, applies all migrations, runs the event loop until the
window reports ready, closes it, exits 0 — or non-zero with the reason.

- It MUST NOT touch the user's environment: it builds the `Paths` of §4.2 from
  subdirectories of one temp dir and disables credential-store access. A test asserts that
  a `--self-check` run creates nothing under the real directories.
- Timeouts: ready within 30 s, shutdown within 10 s, whole run within 60 s. A timeout
  exits non-zero.
- It prints the app version.

### 5.4 Package

- Packaging is done only by `scripts/package.py`, run after the build on each OS. CI's
  package step calls that script; inline packaging commands are PROHIBITED.
- `version` in `pyproject.toml` is the single source; installer, About dialog and artifact
  names read it.
- Per-user install by default. Uninstall MUST NOT delete the data directory without asking.
- Builds are unsigned. `README.md` tells users to expect SmartScreen and Gatekeeper
  warnings.

### 5.5 CI

- Push / PR: one Linux job — `uv sync --locked`, Ruff, mypy, pytest offscreen including the
  boundary check.
- Version tag: one job per declared target OS — test, build, `--self-check`, package. The
  release is published only if every OS job succeeds.

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

- pytest-qt runs with `qt_api = "pyside6"`, and `QT_QPA_PLATFORM=offscreen` is set before
  any `QApplication` exists.
- Every widget created in a test gets `qtbot.addWidget`. Tests never create their own
  `QApplication`.
- Wait with `qtbot.waitSignal`, `waitSignals`, or `waitUntil`, always with a timeout.
  `time.sleep()` and `qtbot.wait(ms)` to "let things settle" are PROHIBITED.
- Assert what the user sees: text, enabled state, row count, visibility.
- Every `QMessageBox` / `QFileDialog` static call a test reaches MUST be monkeypatched.
- Worker logic is tested as a plain function on the test thread; only the signal wiring is
  tested across threads.
- Qt warnings captured during a test are investigated, not ignored.

---

## 7. Where agents go wrong

The mistakes agents most often make here. Each breaks the rule in the section named.

1. PyQt idioms in PySide6: `pyqtSignal`, `pyqtSlot`, `exec_()`, short-form enums. PySide6
   uses `Signal`, `Slot`, `exec()`, `Qt.AlignmentFlag.AlignLeft`.
2. Blocking work inside a slot; widgets updated from a worker instead of by signal (§4.1).
3. `QThread` held in a local variable, collected while running (§4.1).
4. Validation and decisions written in `ui/` because that is where the button is (§3).
5. One `Session` shared across threads (§4.3).
6. `QSettings` or a settings file beside the executable (§1).
7. A hung test — unpatched modal, untimed wait — rerun instead of fixed (§6).
8. Letting `pyside6-deploy` install Nuitka; `import PySide6` at module top (§5.1, §5.2).
9. Switching to PyInstaller to escape a Nuitka error. A build error is not grounds for the
   Alternative's ADR.
10. An adapter calling `platformdirs` on its own (§4.2).

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
      another person goes through §5.5
