---
name: local-model-setup
description: Add llama-cpp-python and sqlite-vec to a pc-app-python project, pin their build dependencies, load sqlite-vec on each connection, run the model and parsers in watched child processes, install the runtime network guard, extend --self-check, the test setup and CI. Use when adding the AI/RAG module to a project, upgrading llama-cpp-python or sqlite-vec, switching to a GPU build, or editing pyproject.toml's build constraints, tests/model/conftest.py, the self-check, or the code that starts the model or parser child process, or the network guard.
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
2. `uv add "llama-cpp-python==<version>" "sqlite-vec==<version>" "psutil==<version>"` and
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
        except Exception as exc:  # noqa: BLE001 — every load failure is caught
            status.error = status.error or repr(exc)
            return
        try:
            sqlite_vec.load(dbapi_conn)
        except Exception as exc:  # noqa: BLE001
            status.error = status.error or repr(exc)
        finally:
            # Not caught: if loading cannot be switched off, this connection must fail.
            dbapi_conn.enable_load_extension(False)
```

`status.error` is only ever set, never cleared: a later connection that loads the
extension leaves it as it is, so a pool with a failed connection keeps the index features
off until the app restarts.

Typical causes: `AttributeError` (this Python's `sqlite3` has no extension support),
`sqlite3.OperationalError` (loading not authorised, or the extension file not loadable).
Every exception is caught, so the connection stays usable, extension loading is off
again, and `status.error` keeps the first cause for the log and the message the user
sees. The one exception that escapes is a failure to switch loading off: that connection
fails rather than stay open with loading enabled.

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

## 4. Network guard (rule §4.6)

`adapters/netguard.py`. `install_network_guard()` is the first statement of every process
entry point — the app's `main`, the model child, the parser child — before any other
import, so that `ssl` and every library subclass the guarded socket.

```python
import ipaddress
import socket

_AF_UNIX = getattr(socket, "AF_UNIX", None)
_real_getaddrinfo = socket.getaddrinfo


class NetworkRefused(OSError):
    pass


def _local(family, address) -> bool:
    if family == _AF_UNIX:
        return True
    host = address[0] if isinstance(address, tuple) else address
    if host in ("", "localhost"):
        return True
    try:
        return ipaddress.ip_address(host).is_loopback
    except ValueError:
        return False


def _check(family, address) -> None:
    if not _local(family, address):
        raise NetworkRefused(f"connection outside this machine refused: {address!r}")


class _GuardedSocket(socket.socket):
    def connect(self, address):
        _check(self.family, address)
        return super().connect(address)

    def connect_ex(self, address):
        _check(self.family, address)
        return super().connect_ex(address)

    def bind(self, address):
        _check(self.family, address)
        return super().bind(address)

    def sendto(self, data, *args):
        _check(self.family, args[-1])
        return super().sendto(data, *args)


def _getaddrinfo(host, *args, **kwargs):
    if host is not None and not _local(None, host):
        raise NetworkRefused(f"name lookup refused: {host!r}")
    return _real_getaddrinfo(host, *args, **kwargs)


def install_network_guard() -> None:
    """First statement of every process entry point, before any other import."""
    socket.socket = _GuardedSocket          # type: ignore[misc]
    socket.getaddrinfo = _getaddrinfo       # type: ignore[assignment]
```

It refuses connections, binds, datagrams and name lookups outside this machine, and
allows loopback and Unix sockets, which `multiprocessing`, `asyncio` and the `llama-server`
Alternative need. It is Python-level only: a socket opened by native code is not seen
(rule §4.6).

## 5. Child processes (rule §4.1, §4.4)

`adapters/children.py`. Two shapes: `run_once` for a parser, one document per process;
`ModelChild` for the model, one process that lives across requests. Both create their
pipes and process inside the `try` whose `finally` stops them, so a failure at any point
after creation cleans up.

```python
import multiprocessing as mp
import sys
import time

import psutil


class ChildFailed(Exception):
    """The child hung, died, or used too much memory; it has been stopped."""


class RequestFailed(Exception):
    """The child reported an error for one request; it is still running."""


def _stop(proc) -> None:
    if proc is not None and proc.is_alive():
        proc.terminate()
        proc.join(2)
        if proc.is_alive():
            proc.kill()
            proc.join(2)


def _receive(conn, proc, deadline, *, interval, watched=None, mem_limit=None):
    """Wait for one message; raise ChildFailed on time, death, or memory."""
    while True:
        if conn.poll(interval):
            try:
                return conn.recv()
            except EOFError:  # the child died: a closed pipe polls as readable
                raise ChildFailed(f"exited with {proc.exitcode}") from None
        if not proc.is_alive():
            raise ChildFailed(f"exited with {proc.exitcode}")
        if mem_limit is not None:
            try:
                if watched.memory_info().rss > mem_limit:
                    raise ChildFailed("memory limit")
            except psutil.NoSuchProcess:
                raise ChildFailed("exited") from None
        if time.monotonic() > deadline:
            raise ChildFailed("time limit")


# --- one-shot child: a parser, one document per process -----------------------

def run_once(target, payload, *, timeout, mem_limit, interval=0.2):
    """Run target(conn, payload) in a new child and return the one message it sends."""
    ctx = mp.get_context("spawn")
    proc = recv = send = None
    try:
        recv, send = ctx.Pipe(duplex=False)
        proc = ctx.Process(target=target, args=(send, payload), daemon=True)
        proc.start()
        send.close()
        send = None
        watched = psutil.Process(proc.pid)
        deadline = time.monotonic() + timeout
        result = _receive(recv, proc, deadline, interval=interval,
                          watched=watched, mem_limit=mem_limit)
        proc.join(interval)
        return result
    finally:
        _stop(proc)
        for conn in (recv, send):
            if conn is not None:
                conn.close()


# --- long-lived child: the model ---------------------------------------------

class ModelChild:
    """The model process. Used only from the model worker thread."""

    def __init__(self, target, args, *, load_timeout, chunk_timeout, interval=0.2):
        self._target, self._args = target, args
        self._load_timeout, self._chunk_timeout = load_timeout, chunk_timeout
        self._interval = interval
        self._proc = self._to_child = self._from_child = None

    def start(self) -> None:
        ctx = mp.get_context("spawn")
        try:
            child_in, self._to_child = ctx.Pipe(duplex=False)
            self._from_child, child_out = ctx.Pipe(duplex=False)
            self._proc = ctx.Process(target=self._target,
                                     args=(child_in, child_out, *self._args), daemon=True)
            self._proc.start()
            child_in.close()
            child_out.close()
            deadline = time.monotonic() + self._load_timeout
            if _receive(self._from_child, self._proc, deadline,
                        interval=self._interval) != ("ready",):
                raise ChildFailed("no ready message")
        except BaseException:
            self.stop()
            raise

    def request(self, message, cancelled=lambda: False):
        """Yield the chunks of one answer. On any failure the child is stopped."""
        if self._proc is None or not self._proc.is_alive():
            self.start()
        error = None
        try:
            self._to_child.send(("request", message))
            cancel_sent = False
            while True:
                if cancelled() and not cancel_sent:
                    self._to_child.send(("cancel",))
                    cancel_sent = True
                deadline = time.monotonic() + self._chunk_timeout
                kind, *body = _receive(self._from_child, self._proc, deadline,
                                       interval=self._interval)
                if kind == "chunk":
                    yield body[0]
                elif kind == "done":
                    break
                elif kind == "error":
                    error = body[0]
                    break
        except BaseException:  # includes the worker abandoning the generator
            self.stop()
            raise
        if error is not None:
            raise RequestFailed(error)

    def stop(self) -> None:
        _stop(self._proc)
        for conn in (self._to_child, self._from_child):
            if conn is not None:
                conn.close()
        self._proc = self._to_child = self._from_child = None


def serve(inbox, outbox, load, answer):
    """Child side of ModelChild. load() -> model; answer(model, message) yields chunks."""
    model = load()
    outbox.send(("ready",))
    while True:
        try:
            kind, *body = inbox.recv()
        except EOFError:
            return
        if kind != "request":
            continue
        try:
            for chunk in answer(model, body[0]):
                outbox.send(("chunk", chunk))
                if inbox.poll(0) and inbox.recv() == ("cancel",):
                    break
            outbox.send(("done",))
        except Exception as exc:
            outbox.send(("error", repr(exc)))
```

- `ChildFailed`: the child hung, died, or passed the memory limit; it has been stopped,
  and the next `request` starts a new one. `RequestFailed`: the model raised for one
  request; the child keeps running.
- Cancel: the worker passes `cancelled`, checked between chunks; the child stops at its
  next chunk. A child that sends nothing within `chunk_timeout` is stopped.
- A worker that stops iterating a `request` before it ends abandons it, and the child is
  stopped: an answer half-read is not resumed.
- `target`, `load` and `answer` are module-level functions in their own modules
  (`adapters/model_child.py`, the parser modules), because `spawn` imports them in the
  child. The app's entry point calls `multiprocessing.freeze_support()` after the guard.

The model child's entry point:

```python
def model_main(inbox, outbox, chat_path, embed_path):
    install_network_guard()
    from llama_cpp import Llama

    def load():
        return {
            "chat": Llama(model_path=chat_path, verbose=False),
            "embed": Llama(model_path=embed_path, embedding=True, verbose=False),
        }

    def answer(models, message):
        ...  # stream=True and max_tokens on every call; yield each text chunk

    serve(inbox, outbox, load, answer)
```

The parser child's entry point. The memory limit is polled by `run_once` on every OS; on
Linux the child also sets it as a hard limit with `resource`, which works there in a
`spawn` child (Verified). Windows has no standard-library equivalent, and macOS was not
checked.

```python
def parse_main(conn, payload):
    install_network_guard()
    data, parse, max_chars, mem_limit = payload  # parse: a module-level function
    if sys.platform == "linux":
        import resource
        resource.setrlimit(resource.RLIMIT_AS, (mem_limit, mem_limit))
    text = parse(data)
    if len(text) > max_chars:
        conn.send(("error", f"text over {max_chars} characters"))
    else:
        conn.send(("text", text))
```

## 6. Self-check additions (rule §5.3)

These run wherever the archetype's `--self-check` runs, the packaged build included
(`pyside-release`). The self-check runs under the same guard as the product (§4), in the
app and in every child it starts. Then:

- Connect to `192.0.2.1` (a documentation-only address) and fail unless the guard refuses
  it — proves the guard is installed in the packaged build.
- Start a `ModelChild` whose `load` imports `llama_cpp` and returns its version instead of
  a model; fail unless it answers — proves `spawn` and the native library work in the
  packaged build.
- Open the temp database through the normal `Engine`, fail if `VecStatus.error` is set,
  print `vec_version()`, create a `vec0` table with dimension 3, insert one vector, search
  for it, and fail if it is not returned.

## 7. Test setup (rule §6)

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
the summary line into the release task's report. With the `llama-server` Alternative, its
adapter's tests run with `--allow-hosts=127.0.0.1` added to that command; the
`--disable-socket` from `addopts` stays, and every other address is still refused.

Tests for §3, with `sqlite_vec.load` replaced by a function that raises
`sqlite3.OperationalError`: a connection still opens, `SELECT 1` works, `VecStatus.error`
holds the cause, and `SELECT load_extension('x')` is refused (loading switched off again).

Tests for §5 use fake `target`, `load` and `answer` functions in a test module: a parser
that hangs, exits, returns too much text, and allocates past the limit; a model answer
that hangs, exits, raises, and streams normally, with a cancel after the first chunk and
a request after each failure.

## 8. CI (rule §5.5)

- `ci.yml` builds the CPU Default only, with `CMAKE_ARGS` unset. It MAY cache the built
  llama-cpp-python wheel; the cache key includes the OS, the architecture, the runner
  image version (`$ImageOS` and `$ImageVersion` on GitHub-hosted runners), the Python
  version and the hash of `uv.lock` (which pins llama-cpp-python and the build
  constraints).
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

2026-11-15, same platform: pytest-socket with `--disable-socket --allow-hosts=127.0.0.1` —
a loopback connection succeeds, an external one is refused; `--disable-socket` alone
refuses both. A §4-style loop with psutil 7.2.2 ends a child that hangs (time limit) and
one that allocates past the limit (memory limit), reports one that exits without
answering (a closed pipe polls as readable and raises `EOFError`), and returns a normal
child's result.
`resource.setrlimit(RLIMIT_AS)` also works in a Linux spawn child.

2026-11-16, same platform, the code of §4 and §5 as written: the guard allows a loopback
connection, `socketpair`, `ssl`, an `asyncio` loop and `spawn` with one-way and duplex
pipes, and refuses an external IP, an external name, a bind to `0.0.0.0` and an external
UDP send. `run_once` returns a normal parser's text and reports hang, memory and exit.
`ModelChild` streams an answer; after a cancel and after a `RequestFailed` the same process
serves the next request; after a hang and after an exit it is stopped and the next
request starts a new one.

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
- `spawn` child processes in the `pyside6-deploy` build, with `freeze_support()`.
- The guard and `psutil` on Windows and macOS.
- `$ImageVersion` on every GitHub-hosted OS image the project uses.
