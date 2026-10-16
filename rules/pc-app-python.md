# rules/pc-app-python.md

Archetype rule for desktop applications written in Python with a Qt UI.

**Precedence.** `GLOBAL_RULES.md` → `rules/pc-app-python.md` → `CLAUDE.md` → current task.
This document narrows `GLOBAL_RULES.md` and adds what only this archetype needs. It never
relaxes a global rule. Everything already stated there — task classes, phases, testing
principles, commit rules, security basics, the agent protocol — applies unchanged and is not
repeated here.

**RFC 2119.** MUST / MUST NOT / SHOULD / SHOULD NOT / MAY carry their RFC 2119 meaning.

**Selections** are marked **Default** (use without justification), **Alternative**
(allowed; record the reason as an ADR in `DECISIONS.md`), or **Prohibited** (MUST NOT be
used; listed because it is a plausible wrong answer).

---

## 1. Scope

Use this archetype when the product is a desktop program, written only in Python, with a
native window — not a web page. It runs on the user's machine, keeps its data there, and
needs no network after installation unless the PRD says otherwise.

It is the wrong archetype when:

- The UI is really a web page or needs HTML/CSS/JS rendering → `pc-app-electron`.
- More than one person uses the same data at the same time → a web archetype.
- The program has no window → it is a script or CLI; this document does not apply.

---

## 2. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Python | 3.12+, pinned in `.python-version`, installed by uv | — | 3.11 and below; a system Python |
| Packaging | uv (`pyproject.toml`, `uv.lock`) | — | Poetry, pipenv, conda, bare pip |
| GUI framework | **PySide6** (LGPLv3) | — | **PyQt6 / PyQt5** (GPL or paid licence); Tkinter, wxPython, Kivy, Flet |
| UI construction | Qt Widgets, written in Python | Qt Designer `.ui` compiled with `pyside6-uic`; QML | QtWebEngine as the UI |
| Persistence | SQLite via SQLAlchemy 2.0 (`Mapped[]` declarative) | stdlib `sqlite3` for one or two tables with no relations | QtSql; pickle / shelve as storage |
| Schema migrations | Ordered migration functions keyed on `PRAGMA user_version`, applied at startup (§6.3) | — | `create_all()` against an existing database as a migration |
| App data location | `platformdirs` | — | Paths beside the executable; paths relative to the working directory |
| Settings | TOML file in the user config directory | JSON | INI; pickle; `QSettings` |
| Long-running work | `QThread` + a worker `QObject` moved to it | `concurrent.futures`, with results delivered back by signal | Any work that blocks the GUI thread |
| Lint / format | Ruff | — | Black + isort + flake8 |
| Type check | mypy, strict on `domain/` and `application/` | pyright | — |
| Tests | pytest + pytest-qt + pytest-cov | — | unittest-only; QTest without pytest |
| Build | `pyside6-deploy` (Nuitka), `standalone` mode | PyInstaller — local iteration only; releasing with it needs an ADR | cx_Freeze, py2exe, auto-py-to-exe, Briefcase |
| Windows package | Inno Setup installer wrapping the standalone directory | — | — |
| macOS package | `.app` bundle produced by `pyside6-deploy` | — | — |
| Linux package | The standalone directory (`<name>.bin` inside), archived as `.tar.gz` | — | — |
| CI | GitHub Actions | — | — |

**Why PySide6, not PyQt.** The APIs are nearly identical, but PySide6 is the official Qt
binding under LGPLv3, so a closed-source program may ship against it. PyQt is GPL or a paid
commercial licence. Code copied from PyQt examples MUST be converted, not mixed in (§11).

**Why `standalone`, not `onefile`.** `onefile` unpacks itself to a temporary directory on
every launch: slower start, frequent antivirus false positives, and Qt libraries that the
user cannot replace. `standalone` leaves Qt as separate shared libraries, which is the
straightforward way to meet LGPL, and it is what the installer wraps anyway. macOS builds
are standalone regardless of the setting.

**Why not QtSql or QSettings.** Both put persistence inside the GUI toolkit, which the
dependency rule forbids for application logic. `pyside6-deploy` also refuses to build a
macOS app that uses QtSql. `QSettings` writes to the Windows registry by default, which
cannot be inspected, backed up, or diffed like a file.

**Licence notes.**

- PySide6 / Qt: every distributed build MUST include the Qt and PySide6 licence texts.
- Inno Setup: free to use, including for commercial software. Its authors *request* — but
  do not require — that commercial users buy a licence, defining a commercial user as a
  for-profit organisation or individual with annual revenue above USD 5,000. Public-sector
  and personal use fall outside that definition. A project that crosses the threshold
  records its choice in `DECISIONS.md`.

---

## 3. Machine Prerequisites

What must exist before the first build. How to install each item belongs to the manuals.

| Requirement | Where | Note |
|---|---|---|
| Python 3.12+ | uv-managed | No separate installer |
| PySide6 | Project dependency | Ships the Qt runtime and the Qt tools (`pyside6-deploy`, `pyside6-uic`, `pyside6-rcc`, `pyside6-designer`). No separate Qt install |
| C compiler | **System — required for builds** | Nuitka compiles to C. Windows: Visual Studio Build Tools (C++ workload), and MSVC MUST be on `PATH` (run `vcvarsall.bat` or use a Developer prompt) before building. macOS: Xcode Command Line Tools. Linux: gcc or clang, plus ccache |
| Nuitka and its helper packages | Project `build` dependency group (§8.1) | Locked in `uv.lock`, not installed ad hoc |
| patchelf | `build` group, Linux only | Required by Nuitka on Linux |
| Inno Setup | System, Windows only | `iscc` on `PATH` |
| Qt runtime libraries | System, Linux only | `libxcb-cursor0` is mandatory since Qt 6.5; also `libxkbcommon-x11-0`, `libegl1`, `libgl1` and the other `libxcb-*` packages. Absent on minimal distros and CI images |

The C compiler is the most common reason a first build fails on a fresh machine, and nothing
in `uv sync` reveals that it is missing. Check it before the first build, not during it.

After setup, development and local builds work fully offline. Keep the uv cache; it is what
makes that true.

---

## 4. Project Layout

```
<repo>/
├── pyproject.toml          Dependencies, tool config, required-environments (§8.1)
├── uv.lock
├── .python-version
├── src/<app>/
│   ├── main.py             Entry point and composition root. The pyside6-deploy target
│   ├── pysidedeploy.spec   Build configuration. Committed
│   ├── domain/             Rules of the problem. Pure Python
│   ├── application/        Use cases, and the interfaces they need (ports.py)
│   ├── adapters/           SQLite, settings file, filesystem, clock — implementations
│   │   └── migrations.py   Ordered schema migrations (§6.3)
│   ├── ui/                 Windows, dialogs, widgets, item models
│   │   └── workers/        QObject workers run on QThreads
│   └── resources/          Icons, images, licence texts shipped with the app
├── installer/
│   └── windows.iss         Inno Setup script
├── tests/
│   ├── conftest.py
│   ├── domain/  application/  adapters/  ui/
└── dist/                   Build output. Gitignored
```

- `main.py` is the only place that creates `QApplication`, constructs adapters, and passes
  them into use cases and windows. Nothing else reaches for a global instance.
- `.venv/` and `dist/` MUST stay outside `src/<app>/`. `pyside6-deploy` treats the entry
  file's directory as the project and will try to package anything inside it.
- A program with no business rules — a single window around one file operation — MAY
  omit `domain/` and `application/`. It MUST NOT grow them to satisfy this layout.
- New top-level directories under `src/<app>/` need a reason in `DECISIONS.md`.

---

## 5. Architecture Boundaries

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | stdlib, other `domain/` modules | PySide6, SQLAlchemy, `platformdirs`, anything in `adapters/` or `ui/` |
| `application/` | `domain/`, its own `ports.py` | PySide6, SQLAlchemy, `adapters/`, `ui/` |
| `adapters/` | `application/ports.py`, `domain/`, SQLAlchemy, `platformdirs`, `tomllib` | PySide6 widgets, `ui/` |
| `ui/` | `application/`, `domain/` types, PySide6 | SQLAlchemy, `adapters/` directly |

- No Qt type crosses into `domain/` or `application/`: no `Signal`, `QObject`, `QDate`,
  `QModelIndex`. Signals exist in `ui/` only, where a window or a small presenter wraps a
  use case and emits.
- Ports are `typing.Protocol` classes in `application/ports.py`.
- A slot or event handler calls one use case and updates the UI with the result. A
  condition that decides *what the application does* inside a slot is a defect.

Add a CI check that fails if `domain/` or `application/` imports `PySide6` or `sqlalchemy`
(a grep in CI or a Ruff `banned-api` rule). A boundary that is only written down does not
hold.

---

## 6. Runtime Rules

### 6.1 The GUI thread

- The GUI thread MUST NOT block. Anything that can take longer than roughly 100 ms — file
  scans, large queries, network, parsing — runs on a worker.
- Widgets MUST be touched only from the GUI thread. A worker reports progress and results
  by emitting signals; it never calls a widget method.
- Pattern: create a `QThread`, create a worker `QObject`, `worker.moveToThread(thread)`,
  start work from `thread.started`, connect the worker's `finished` to `thread.quit` and
  both objects' cleanup to `deleteLater`. Keep a reference to the thread and the worker
  until `finished`; a garbage-collected `QThread` that is still running crashes the
  process.
- On window close, running workers MUST be asked to stop and the threads waited on. Do not
  let the process exit with a thread still running.
- `QApplication.processEvents()` loops and `time.sleep()` on the GUI thread are PROHIBITED
  as a way to keep the UI responsive.

### 6.2 Files and paths

- User data → `platformdirs.user_data_dir(<app>)`. Settings →
  `user_config_dir`. Logs → `user_log_dir`. Created on first run if missing.
- The installation directory is read-only at runtime. On Windows it is under
  `Program Files`, and writes there fail or are silently redirected.
- Resources shipped with the app are located relative to the package (`importlib.resources`
  or `Path(__file__)`), never relative to the working directory, which is arbitrary when
  the app is launched from a shortcut.

### 6.3 Persistence and migrations

- One SQLAlchemy `Engine` per process, created in `main.py`. A `Session` MUST NOT cross
  threads: each worker opens its own session from the shared `sessionmaker` and closes it.
- The database file lives in `user_data_dir`. Enable `PRAGMA foreign_keys=ON` on every
  connection; SQLite leaves it off by default.
- Schema version is `PRAGMA user_version`. `adapters/migrations.py` holds an ordered list
  of migration functions; at startup the app applies every migration above the stored
  version, each in its own transaction, then sets the new version.
- Before applying any migration to an existing file, the app MUST copy the database file to
  a timestamped backup beside it. This is the restore path the global rule requires for
  migrations — on a user's machine there is nobody to take a backup by hand.
- If the stored version is **higher** than the app knows, the app MUST refuse to open the
  database and tell the user to update. An older build writing to a newer schema corrupts
  data silently.
- Migrations are append-only. A shipped migration is never edited; a fix is a new one.

### 6.4 Settings

- Settings are a typed dataclass read once at startup and passed explicitly.
- Unknown keys are ignored with a logged warning; missing keys take defaults; a malformed
  file is renamed aside and replaced with defaults, and the user is told. The app MUST
  start even when the settings file is broken.
- stdlib `tomllib` reads TOML but cannot write it. If the app writes settings, writing TOML
  requires adding a dependency (Blocking, per the global execution protocol) and an ADR;
  otherwise use the JSON alternative.

### 6.5 Errors and logging

- A released build has no console. Logging MUST go to a rotating file in `user_log_dir`
  (`logging.handlers.RotatingFileHandler`).
- `main.py` installs `sys.excepthook` before the window opens. An uncaught exception is
  logged with its traceback and shown to the user as a dialog with a plain message and the
  log file location — never the traceback itself.
- An exception inside a slot does not stop Qt's event loop; the app keeps running in
  whatever state the slot left behind. Slots that change data MUST leave state unchanged
  on failure, and the excepthook makes the failure visible.

---

## 7. Testing

### 7.1 Where each kind of test goes

| Layer | Tool | Qt in the test? |
|---|---|---|
| `domain/`, `application/` | pytest, with fakes for ports | No. These tests MUST NOT import PySide6 |
| `adapters/` | pytest against a real SQLite file in `tmp_path` | No |
| `ui/` | pytest-qt (`qtbot`) | Yes |
| Packaged build | Self-check run (§8.3) | Yes, in the built artifact |

Most tests belong in the first two rows. A UI test proves that a click reaches the right
use case and that the result is displayed — not the business rule, which is already tested
without Qt.

### 7.2 Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
qt_api = "pyside6"
markers = [
  "integration: needs network or an external system; excluded by default",
]
addopts = "-m 'not integration'"
```

```python
# tests/conftest.py — must run before any QApplication exists
import os
os.environ.setdefault("QT_QPA_PLATFORM", "offscreen")
```

`offscreen` runs widget tests with no display, on any OS, locally and in CI. A developer
who wants to watch a test sets `QT_QPA_PLATFORM` explicitly to override it.

### 7.3 pytest-qt rules

- Every widget created in a test is registered with `qtbot.addWidget(widget)`, so it is
  closed and freed after the test.
- Tests never create their own `QApplication`. pytest-qt provides one.
- Waiting is done with `qtbot.waitSignal(signal, timeout=...)`, `qtbot.waitSignals`, or
  `qtbot.waitUntil(predicate, timeout=...)` — always with a timeout. `time.sleep()` and
  `qtbot.wait(ms)` used to "let things settle" are PROHIBITED; they are the source of
  flaky UI tests.
- Assert what the user would see: label text, enabled state, row count, visibility. Not
  private attributes.
- A modal dialog in a test blocks forever. Every `QMessageBox` / `QFileDialog` static call
  reached by a test MUST be replaced with `monkeypatch` to return the answer the test
  needs.
- Worker logic is tested as a plain function on the test thread. Only the wiring — that the
  worker's signal reaches the window — is tested across threads, with `waitSignal`.
- Warnings logged by Qt during a test (`qWarning`, `qCritical`) are captured by pytest-qt.
  A test that produces Qt warnings is investigated, not ignored.

---

## 8. Build and Packaging

### 8.1 Lock the build toolchain

On first run, `pyside6-deploy` installs Nuitka and its helper packages into the current
environment with pip. In a uv project that either fails (uv environments do not include pip
by default) or installs versions that `uv.lock` knows nothing about. Therefore:

- Nuitka and the helper packages `pyside6-deploy` expects (`ordered-set`, `zstandard`, and
  `patchelf` on Linux) MUST be declared in a `build` dependency group and locked. If
  `pyside6-deploy` still requires pip to be importable, add `pip` to the same group rather
  than seeding it by hand.
- The Nuitka version written in `pysidedeploy.spec` MUST match the one in `uv.lock`.
- `pyproject.toml` MUST declare `[tool.uv] required-environments` for every target OS, so
  that `uv lock` rejects a PySide6 release that lacks a wheel for one of them instead of
  letting the gap surface on that OS's build.

### 8.2 Build

```
uv sync --locked --group build
uv run pyside6-deploy -c src/<app>/pysidedeploy.spec -f
```

- `pysidedeploy.spec` is committed. Set `mode = standalone` and point the output directory
  at the gitignored `dist/`. Generated deployment files are not committed.
- Code imports Qt as `from PySide6.QtWidgets import ...`. A bare `import PySide6` defeats
  `pyside6-deploy`'s detection of which Qt modules to bundle.
- If Qt Designer is used, `ui_*.py` files are generated by `pyside6-uic`, committed, and
  never edited by hand. Regenerate them; do not patch them.
- `pyside6-deploy --dry-run` prints the Nuitka command without running it. Use it to
  inspect what will be bundled before a slow build.
- The first build on each target OS happens in the first week of the project, as part of a
  walking skeleton — not at release time. Builds of uv-managed projects with
  `pyside6-deploy` have had platform-specific failures; finding one late means finding it
  with a deadline.

### 8.3 Self-check

`main.py` accepts `--self-check`: it starts the application, opens a database in a
temporary directory, applies all migrations, constructs the main window, closes it, and
exits with status 0 — or non-zero with the reason on stderr and in the log.

Passing tests do not show that the built artifact starts. Missing data files, an unbundled
Qt plugin, or an import that only works from source all appear here and nowhere else. Every
build that will be handed to anyone MUST pass `--self-check` on the OS it was built for.

### 8.4 Package

| OS | Input | Output |
|---|---|---|
| Windows | standalone directory | `installer/windows.iss` → `<app>-<version>-setup.exe` |
| macOS | `.app` | `<app>-<version>-macos.zip` |
| Linux | standalone directory | `<app>-<version>-linux-x86_64.tar.gz` |

- Version has one source: `version` in `pyproject.toml`. The installer script, the window
  title or About dialog, and the artifact names read it from there. The self-check prints
  it, so a mismatch is caught in CI.
- The installer installs per-user by default and never writes user data into the
  installation directory.
- Uninstalling MUST NOT delete the user's data directory without asking.
- Signing and notarization are out of scope for this archetype rule. Unsigned builds work,
  but Windows SmartScreen and macOS Gatekeeper warn or block on first launch; tell users how
  to proceed rather than hiding it.

---

## 9. Release

A local build is for the developer's own machine. Anything given to another person is built
by CI from a tagged commit.

**On every push and pull request** — Linux runner: `uv sync --locked`, Ruff, mypy, the
default pytest run with `QT_QPA_PLATFORM=offscreen`, and the boundary check from §5. The
Qt runtime libraries from §3 are installed in the job.

**On a version tag** — a matrix of Windows, macOS, and Linux runners. Each runner: sync,
run the full test suite, build (§8.2), run `--self-check` on the result, package (§8.4),
and attach the artifact to the GitHub Release for that tag. The release is published only
if every OS succeeds.

- Each job verifies its toolchain before building: the compiler, and on Windows `iscc` on
  `PATH`. Never assume a runner image ships a tool; install it if it is absent.
- Cost: GitHub-hosted runners are free for public repositories. Private repositories on the
  free plan get 2,000 minutes a month, and Windows and macOS minutes count at 2× and 10×.
  Nuitka builds are long, so the three-OS matrix runs on tags only.

---

## 10. Prohibited in This Archetype

In addition to the Prohibited column in §2:

- Importing from `PyQt5` or `PyQt6` anywhere, including tests and scripts.
- `QtWebEngine` used to draw the application's UI. If the UI is HTML, the archetype is
  wrong.
- Any PySide6 or SQLAlchemy import in `domain/` or `application/`.
- Calling widget methods from a non-GUI thread, directly or through a Python `threading`
  thread.
- `processEvents()` loops or `time.sleep()` on the GUI thread.
- Writing anything into the installation directory or the working directory at runtime.
- `onefile` mode for a distributed build.
- Distributing an artifact that was built on a developer's machine.
- Editing generated `ui_*.py` files.
- Editing a migration that has shipped.
- Opening a database whose `user_version` is newer than the app supports.

---

## 11. Where Agents Go Wrong

These are the mistakes an agent makes most often in this archetype. When reviewing an
agent's change, check these first.

1. **PyQt idioms in PySide6 code.** `pyqtSignal`, `pyqtSlot`, `exec_()`, short-form enums
   copied from PyQt5 examples. PySide6 uses `Signal`, `Slot`, `exec()`, and fully qualified
   enums (`Qt.AlignmentFlag.AlignLeft`). A mixed import compiles in some places and fails
   at runtime in others.
2. **Blocking work in a slot.** A button handler that reads a large file or runs a query
   directly. The window freezes and the OS marks it "not responding".
3. **Updating widgets from a worker.** Works most of the time in testing, crashes
   unpredictably in use. Every cross-thread update goes through a signal.
4. **Losing the `QThread` reference.** A thread created as a local variable is collected
   while running and takes the process down.
5. **Business rules in widgets.** Validation and decisions written in `ui/` because that is
   where the button is. Move them into a use case and test them without Qt.
6. **Session shared across threads.** A SQLAlchemy session created on the GUI thread and
   used inside a worker raises SQLite's same-thread error, or worse, appears to work.
7. **`QSettings` or a file beside the executable** for settings, because that is what most
   examples show.
8. **Tests that hang.** A modal dialog left unpatched, or a `waitSignal` without a timeout.
   A hung test is a failure; the agent MUST NOT wait it out or rerun it hoping it passes.
9. **`sleep` to fix a flaky UI test.** The flakiness is a missing wait on a specific
   signal or condition.
10. **"Tests pass, so the build works."** The build is a separate claim, proven only by
    `--self-check` on the built artifact.
11. **Letting `pyside6-deploy` install things.** An unlocked Nuitka appears in the
    environment and the next machine builds with a different one.
12. **`import PySide6`** at the top of a module. `pyside6-deploy` then cannot tell which Qt
    modules to bundle.
13. **Reaching for PyInstaller when Nuitka fails.** Switching build tools to escape an
    error is the "switch architecture to escape the error" the three-strikes rule forbids.
    PyInstaller is an Alternative with an ADR, not an escape hatch.
14. **`create_all()` as a migration.** It creates missing tables and silently ignores every
    changed column.

---

## 12. Definition of Done — Additions

On top of the global Definition of Done:

**Every task touching this archetype**

- [ ] No PySide6 or SQLAlchemy import in `domain/` or `application/` (boundary check passes)
- [ ] No GUI-thread blocking added; long work runs on a worker and reports by signal
- [ ] UI tests use `qtbot.addWidget`, timed waits only, and no unpatched modal dialog

**Additionally, when the task changes the schema**

- [ ] A new migration appended; no shipped migration edited
- [ ] Tested from the previous schema version with real data in `tmp_path`, including the
      backup file being created

**Additionally, when the task changes packaging, dependencies, resources, or `main.py`**

- [ ] A build on at least one OS passes `--self-check`; the OSes not built are reported as
      not run
