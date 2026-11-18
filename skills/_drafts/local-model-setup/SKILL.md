---
name: local-model-setup
description: Add llama-cpp-python and sqlite-vec to a pc-app-python project, pin their build dependencies, load sqlite-vec on each connection, extend --self-check, the test setup and CI. Use when adding the AI/RAG module to a project, upgrading llama-cpp-python or sqlite-vec, switching to a GPU build, or editing pyproject.toml's build constraints, tests/model/conftest.py or the self-check.
---

# local-model-setup (draft)

Procedures for `rules/module-ai-rag.md`. The rules they serve stay in that file (§2, §4,
§5). Downloading model files and installing on an offline PC are the
human's work and belong in the manual, not here. This draft leaves `_drafts/` only when the Unverified list at the end is empty.

Verified 2026-11-13 on Linux x86_64, uv 0.11.32, uv-managed CPython 3.12.13:
llama-cpp-python 0.3.36 built from the PyPI source distribution; sqlite-vec 0.1.9 loaded,
inserted, deleted and searched; pytest-socket 0.8.1 installed.

## 1. Add the dependencies (rule §5.1)

1. Confirm the C compiler the archetype already requires: `gcc --version`, `clang
   --version`, or on Windows the VS Build Tools C++ workload.
2. `uv add "llama-cpp-python==<version>" "sqlite-vec==<version>"` and
   `uv add --dev pytest-socket`. Exact versions only.
3. Find the build dependencies llama-cpp-python pulled in. In this verification they were
   `scikit-build-core`, `packaging`, `pathspec`, `cmake`, `ninja` (`cmake` and `ninja` only
   when the system has none). Pin each in `pyproject.toml`:

   ```toml
   [tool.uv]
   build-constraint-dependencies = [
     "scikit-build-core==1.1.2",
     "packaging==26.3",
     "pathspec==1.1.1",
     "cmake==4.4.4",
     "ninja==1.13.2",
   ]
   ```

   The versions above are the ones resolved on the verification date, not a
   recommendation; take the current ones from `uv pip compile` of
   `scikit-build-core[pyproject]` and from the build log.
4. `uv lock`, then check that `uv.lock` lists them under `build-constraints`.
5. Clean build: `uv cache clean llama-cpp-python`, then `uv sync --locked`. Expect several
   minutes.

## 2. GPU build (rule §1 Alternative, ADR first)

Set `CMAKE_ARGS` before syncing — `-DGGML_CUDA=on` (NVIDIA), `-DGGML_METAL=on` (macOS),
`-DGGML_VULKAN=on` — then clean-build as in §1, step 5. Record the toolkit version and the OSes
covered in the ADR.

## 3. Load sqlite-vec on every connection (rule §4.3)

In the adapter that creates the `Engine`:

```python
import sqlite_vec
from sqlalchemy import event

@event.listens_for(engine, "connect")
def _load_vec(dbapi_conn, _record):
    dbapi_conn.enable_load_extension(True)
    try:
        sqlite_vec.load(dbapi_conn)
    finally:
        dbapi_conn.enable_load_extension(False)
```

`AttributeError` (no `enable_load_extension`) or `sqlite3.OperationalError` (loading not
authorised or the file not loadable) means this Python cannot use sqlite-vec: report it,
per the rule. Extension loading is switched off again on every path.

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

Inside the archetype's `--self-check`, before the window opens:

```python
import socket

def _block_sockets() -> None:
    def _refuse(*_a, **_k):
        raise RuntimeError("network access in self-check")
    socket.socket = _refuse          # type: ignore[assignment]
    socket.create_connection = _refuse  # type: ignore[assignment]
```

Then: `import llama_cpp` and print `llama_cpp.__version__`; open the temp database through
the normal `Engine` (which loads sqlite-vec), print `vec_version()`, create a `vec0` table
with dimension 3, insert one vector, search for it, and fail if it is not returned.

## 5. Test setup (rule §6)

`pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = "--disable-socket -m 'not model'"
markers = ["model: needs a real model file; run by the human"]
```

`tests/model/conftest.py` provides the model paths through a session fixture that calls
`pytest.fail` when `APP_TEST_CHAT_MODEL` or `APP_TEST_EMBED_MODEL` is unset or the file is
missing. A fixture, not a module-level check: the default run deselects these tests and
must not fail on collection. The human runs them with
`uv run pytest -m model -rs tests/model/` and copies the summary line into the release
proposal.

## 6. CI (rule §5.5)

`ci.yml` may cache the built llama-cpp-python wheel through the uv cache keyed on
`uv.lock`; `release.yml` restores no cache and compiles in every OS job.

## Unverified

- macOS: whether a uv-managed Python's `sqlite3` can load extensions. Sources disagree.
  `--self-check` on a macOS build settles it; if it cannot, the fix goes into the rule.
- `pyside6-deploy` (Nuitka): whether the llama-cpp-python shared libraries and the
  sqlite-vec loadable file are included without extra options, and which options are
  needed if not.
- Windows: the llama-cpp-python source build with the VS Build Tools the archetype
  requires.
- Whether `uv` rebuilds llama-cpp-python when `CMAKE_ARGS` changes, or reuses a cached CPU
  build.
- pytest-socket together with pytest-qt's offscreen `QApplication`.
