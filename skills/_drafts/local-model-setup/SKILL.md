---
name: local-model-setup
description: Add llama-cpp-python and sqlite-vec to a pc-app-python project, pin their build dependencies, load sqlite-vec on each connection, extend --self-check, the test setup and CI. Use when adding the AI/RAG module to a project, upgrading llama-cpp-python or sqlite-vec, switching to a GPU build, or editing pyproject.toml's build constraints, tests/model/conftest.py or the self-check.
---

# local-model-setup (draft)

Procedures for `rules/module-ai-rag.md`. The rules they serve stay in that file (§2, §4,
§5). This skill adds only what the AI/RAG module changes; the general toolchain,
packaging, `--self-check` and CI procedures are in `pyside-release`. Downloading model
files and installing on an offline PC are the human's work and belong in the manual, not
here. This draft leaves `_drafts/` only when the Unverified list at the end is empty.

## 1. Add the dependencies (rule §5.1)

1. Confirm the C compiler the archetype already requires: `gcc --version`, `clang
   --version`, or on Windows the VS Build Tools C++ workload.
2. `uv add "llama-cpp-python==<version>" "sqlite-vec==<version>"` and
   `uv add --dev pytest-socket`. Exact versions only.
3. Starting list: `uv pip compile` of `scikit-build-core[pyproject]`. Then do a clean build
   with verbose output (step 5 with `-v`). The packages installed into the build
   environment are the authoritative list: `cmake` and `ninja` appear there only when the
   system has none, and compile does not show them. Every package in that list is pinned:

   ```toml
   [tool.uv]
   build-constraint-dependencies = [
     # example values from 2026-11-13; use the current list from the build log
     "scikit-build-core==1.1.2",
     "packaging==26.3",
     "pathspec==1.1.1",
     "cmake==4.4.4",
     "ninja==1.13.2",
   ]
   ```

4. `uv lock`, then check that `uv.lock` lists them under `build-constraints`.
5. Clean build: `uv cache clean llama-cpp-python`, then `uv sync --locked`. Expect several
   minutes. Repeat it after any change to the list; the log MUST show no build-environment
   package missing from the list.

## 2. GPU build (rule §1 Alternative, ADR first)

1. Pick one backend for the OS: `-DGGML_CUDA=on` (NVIDIA), `-DGGML_METAL=on` (macOS) or
   `-DGGML_VULKAN=on`. Never more than one.
2. `uv cache clean llama-cpp-python`, then set the variable for the sync command only:
   bash `CMAKE_ARGS="-DGGML_CUDA=on" uv sync --locked`; PowerShell
   `$env:CMAKE_ARGS="-DGGML_CUDA=on"; uv sync --locked; Remove-Item Env:CMAKE_ARGS`.
3. Confirm the result is a GPU build:
   `uv run python -c "import llama_cpp; print(llama_cpp.llama_supports_gpu_offload())"`
   MUST print `True`. A CPU build prints `False`.
4. The ADR records the backend, the toolkit version and the OSes covered. Whether uv
   reuses a cached CPU build when only `CMAKE_ARGS` changes is unverified; step 2's clean
   and step 3's check make the answer irrelevant. The first GPU project records what it
   observed in its ADR.

## 3. Load sqlite-vec on every connection (rule §4.3)

In the adapter that creates the `Engine`. A failure MUST NOT escape the event: an
exception raised there makes every connection fail, and the rule keeps the rest of the
database working.

```python
import sqlite3
import sqlite_vec
from sqlalchemy import event


class VecStatus:
    """Read by the application to enable or disable index features."""
    error: str | None = None


def install_vec_loader(engine, status: VecStatus) -> None:
    @event.listens_for(engine, "connect")
    def _load_vec(dbapi_conn, _record):
        try:
            dbapi_conn.enable_load_extension(True)
        except (AttributeError, sqlite3.OperationalError) as exc:
            status.error = repr(exc)
            return
        try:
            sqlite_vec.load(dbapi_conn)
        except sqlite3.OperationalError as exc:
            status.error = repr(exc)
        finally:
            dbapi_conn.enable_load_extension(False)
```

`AttributeError`: this Python's `sqlite3` has no extension support. `OperationalError`:
loading not authorised, or the extension file not loadable. Either way the connection
stays usable, extension loading is off again, and `status.error` keeps the cause for the
log and the message the user sees.

A `vec0` table is created by a migration:

```sql
CREATE VIRTUAL TABLE chunk_vectors USING vec0(
  chunk_id INTEGER PRIMARY KEY,
  embedding FLOAT[<dimension>]
);
```

Search binds the query vector with `sqlite_vec.serialize_float32(vector)`:

```sql
SELECT chunk_id, distance FROM chunk_vectors
WHERE embedding MATCH :query AND k = :k
```

## 4. Self-check additions (rule §5.3)

These run wherever the archetype's `--self-check` runs, the packaged build included
(`pyside-release`). Inside it, before the window opens and before any other import of
the app's modules:

```python
import socket


class _NoSocket(socket.socket):
    def __new__(cls, *_a, **_k):
        raise RuntimeError("network access in self-check")


def _refuse(*_a, **_k):
    raise RuntimeError("network access in self-check")


def _block_sockets() -> None:
    socket.socket = _NoSocket                # type: ignore[misc]
    socket.create_connection = _refuse       # type: ignore[assignment]
    socket.getaddrinfo = _refuse             # type: ignore[assignment]
```

`socket.socket` is replaced by a subclass, not a function: `ssl` subclasses it on import,
and a function there makes `import ssl` fail with `TypeError`. This is a Python-level
guard only — it does not see native code or a child process (rule §4.6). The self-check
starts no child process.

Then: `import llama_cpp` (this loads its shared library) and print
`llama_cpp.__version__`; open the temp database through the normal `Engine`, fail if
`VecStatus.error` is set, print `vec_version()`, create a `vec0` table with dimension 3,
insert one vector, search for it, and fail if it is not returned.

## 5. Test setup (rule §6)

Merge into the archetype's existing pytest settings; do not replace them. Add
`--disable-socket -m 'not model'` to the existing `addopts`, and the `model` entry to the
existing `markers`:

```toml
[tool.pytest.ini_options]
addopts = "<existing options> --disable-socket -m 'not model'"
markers = [
  # existing markers stay
  "model: needs a real model file; run by the human",
]
```

`tests/model/conftest.py` holds one session fixture with `autouse=True` that calls
`pytest.fail` when `APP_TEST_CHAT_MODEL` or `APP_TEST_EMBED_MODEL` is unset or the file is
missing. Autouse, so no model test can skip it by not requesting it; a fixture, not a
module-level check, so the default run, which deselects these tests, does not fail on
collection. The human runs them with `uv run pytest -m model -rs tests/model/` and copies
the summary line into the release proposal.

Tests for §3, with `sqlite_vec.load` replaced by a function that raises
`sqlite3.OperationalError`: a connection still opens, `SELECT 1` works, `VecStatus.error`
holds the cause, and `SELECT load_extension('x')` is refused (loading switched off again).

## 6. CI (rule §5.5)

- `ci.yml` builds the CPU Default only. It MAY cache the built llama-cpp-python wheel; the
  cache key includes the OS, the architecture, the Python version and the hash of
  `uv.lock` (which pins llama-cpp-python and the build constraints).
- GPU builds are never cached and are not in `ci.yml`. A GPU ADR that wants CI adds a
  separate job for its declared OS and toolkit, running §2 steps 2–3.
- `release.yml` restores no cache and compiles in every OS job.

## Verified

2026-11-13, Linux x86_64, uv 0.11.32, uv-managed CPython 3.12.13: llama-cpp-python 0.3.36
built from the PyPI source distribution; sqlite-vec 0.1.9 loaded, inserted, deleted and
searched; pytest-socket 0.8.1 installed.

2026-11-14, same platform, sqlite-vec 0.1.9 with SQLAlchemy: §3's loader with a simulated
load failure — connection open, `SELECT 1` works, `load_extension` refused afterwards.
§4's guard: `import ssl` and `urllib.request` succeed after it, a connection attempt is
refused; the function-replacement form makes `import ssl` raise `TypeError`.
`llama_supports_gpu_offload` present in llama-cpp-python 0.3.36 (source).

## Unverified

- macOS: whether a uv-managed Python's `sqlite3` can load extensions. Sources disagree.
  `--self-check` on a macOS build settles it; if it cannot, the fix goes into the rule.
- `pyside6-deploy` (Nuitka): whether the llama-cpp-python shared libraries and the
  sqlite-vec loadable file are included without extra options, and which options are
  needed if not.
- Windows: the llama-cpp-python source build with the VS Build Tools the archetype
  requires.
- Which line of uv's verbose build output lists the build-environment packages (§1,
  step 3).
- pytest-socket together with pytest-qt's offscreen `QApplication`.
