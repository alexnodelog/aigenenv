---
name: local-model-setup
description: Add llama-cpp-python and sqlite-vec to a pc-app-python project, pin their build dependencies, load sqlite-vec on each connection, run the model and parsers in watched child processes, install the runtime network guard, write agent-tool files safely, extend --self-check, the test setup and CI. Use when adding the AI/RAG module to a project, upgrading llama-cpp-python or sqlite-vec, switching to a GPU build, or editing pyproject.toml's build constraints, tests/model/conftest.py, the self-check, or the code that starts the model or parser child process, the network guard, or an agent tool that writes files.
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
3. Build the list. Take the `[build-system] requires` line from the llama-cpp-python
   source distribution's `pyproject.toml` (for 0.3.36: `scikit-build-core[pyproject]>=0.9.2`)
   and resolve it:

   ```bash
   printf 'scikit-build-core[pyproject]>=0.9.2\n' | uv pip compile - --no-header
   ```

   Pin every package it prints, and also `cmake` and `ninja`: scikit-build-core asks for
   them only on a machine that has none, so pinning them always keeps machines alike. A
   package that appears in the build log of step 5 but not in the list is added to it.

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
        except Exception as exc:  # noqa: BLE001
            # Loading was never switched on, so there is nothing to switch off; calling
            # enable_load_extension(False) here would fail the same way and lose the
            # connection.
            status.error = status.error or repr(exc)
            return
        try:
            sqlite_vec.load(dbapi_conn)
        except Exception as exc:  # noqa: BLE001 — every load failure is caught
            status.error = status.error or repr(exc)
        finally:
            # Not caught: if loading cannot be switched off, this connection must fail
            # rather than stay open with it on.
            dbapi_conn.enable_load_extension(False)
```

`status.error` is only ever set, never cleared: a later connection that loads the
extension leaves it as it is, so a pool with a failed connection keeps the index features
off until the app restarts.

Typical causes: `AttributeError` (this Python's `sqlite3` has no extension support),
`sqlite3.OperationalError` (loading not authorised, or the extension file not loadable).
Every exception is caught and the connection stays usable; `status.error` keeps the first
cause. Switching loading off is attempted whenever switching it on succeeded, and its
failure is the one exception that escapes. When switching on itself fails, loading was
never on: calling `enable_load_extension(False)` then fails the same way and would cost
the connection (Verified).

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

`adapters/netguard.py`. It is installed at module level, as the first statements of the
app's entry script, before any other import:

```python
from <app>.adapters.netguard import install_network_guard
install_network_guard()
```

A `spawn` child runs the entry script again as `__mp_main__` before it imports the
target's module, so the guard is in place in every child before any of its code runs
(Verified with `guard_installed()` evaluated at the target module's import). The model
and parser entry functions call `install_network_guard()` again as their first statement,
for a child started some other way.

```python
import ipaddress
import socket
import threading

_AF_UNIX = getattr(socket, "AF_UNIX", None)
_real = {name: getattr(socket, name) for name in (
    "getaddrinfo", "gethostbyname", "gethostbyname_ex", "gethostbyaddr", "getnameinfo")}
_allowed: set[tuple] = set()  # endpoints this process bound, or the adapter registered
_lock = threading.Lock()


class NetworkRefused(OSError):
    pass


def _loopback(host) -> bool:
    if host in ("", "localhost"):
        return True
    try:
        return ipaddress.ip_address(host).is_loopback
    except ValueError:
        return False


def _endpoint(family, kind, address) -> tuple:
    if family == _AF_UNIX:
        return (family, kind, address)
    host = "127.0.0.1" if address[0] in ("", "localhost") else address[0]
    return (family, kind, host, address[1])


def _check_connect(family, kind, address) -> None:
    if family != _AF_UNIX and not _loopback(address[0]):
        raise NetworkRefused(f"refused by the network guard: {address!r}")
    with _lock:
        known = _endpoint(family, kind, address) in _allowed
    if not known:
        raise NetworkRefused(f"refused by the network guard (not opened here): {address!r}")


def _check_bind(family, address) -> None:
    if family != _AF_UNIX and not _loopback(address[0]):
        raise NetworkRefused(f"bind refused by the network guard: {address!r}")


def allow_endpoint(host: str, port: int) -> None:
    """Called by the llama-server adapter for the server it started: TCP, loopback."""
    if not _loopback(host):
        raise NetworkRefused(f"only loopback endpoints can be allowed: {host!r}")
    with _lock:
        _allowed.add(_endpoint(socket.AF_INET, socket.SOCK_STREAM, (host, port)))


class _GuardedSocket(socket.socket):
    def connect(self, address):
        _check_connect(self.family, self.type, address)
        return super().connect(address)

    def connect_ex(self, address):
        _check_connect(self.family, self.type, address)
        return super().connect_ex(address)

    def bind(self, address):
        _check_bind(self.family, address)
        super().bind(address)
        with _lock:
            _allowed.add(_endpoint(self.family, self.type, self.getsockname()))

    def sendto(self, data, *args):
        _check_connect(self.family, self.type, args[-1])
        return super().sendto(data, *args)

    def sendmsg(self, buffers, ancdata=(), flags=0, address=None):
        if address is not None:
            _check_connect(self.family, self.type, address)
            return super().sendmsg(buffers, ancdata, flags, address)
        return super().sendmsg(buffers, ancdata, flags)


def _name_guard(name):
    real = _real[name]

    def guarded(host, *args, **kwargs):
        target = host[0] if isinstance(host, tuple) else host
        if target is not None and not _loopback(target):
            raise NetworkRefused(f"{name} refused by the network guard: {host!r}")
        return real(host, *args, **kwargs)
    return guarded


def install_network_guard() -> None:
    socket.socket = _GuardedSocket                       # type: ignore[misc]
    for name in _real:
        setattr(socket, name, _name_guard(name))


def guard_installed() -> bool:
    return socket.socket is _GuardedSocket
```

What it covers, by API of the `socket` module. An endpoint is the address family, socket
type, address and port — not the port alone.

| Refused | Allowed |
|---|---|
| `connect`, `connect_ex`, `sendto`, `sendmsg` to any address that is not loopback | Endpoints this process bound itself: `socketpair` (Windows builds it on loopback; Linux uses an unbound Unix pair), `asyncio` |
| Loopback endpoints and Unix socket paths this process did not bind — another local service could relay outside — except the one the `llama-server` adapter registers with `allow_endpoint` | `multiprocessing` pipes; `spawn` |
| `bind` to any address that is not loopback | |
| `getaddrinfo`, `gethostbyname`, `gethostbyname_ex`, `gethostbyaddr`, `getnameinfo` for any host that is not loopback | Names and addresses of this machine |

`multiprocessing.Manager` and `Listener`/`Client` across processes connect to a Unix
socket another process bound, and are refused; the app uses pipes only. The guard is
Python-level: native code, and code importing `_socket` directly, are not seen (rule §4.6).

## 5. Child processes (rule §4.1, §4.4)

`adapters/children.py`. Two shapes: `run_once` for a parser, one document per process;
`ModelChild` for the model, one process that lives across requests. Both create their
pipes and process inside the `try` whose cleanup stops them, the child's pipe ends
included; the cleanup itself never raises, so a failure in `Process.start()` surfaces as
itself (Verified).

```python
import multiprocessing as mp
import pickle
import time

import psutil


class ChildFailed(Exception):
    """The child hung, died, or used too much memory; it has been stopped."""


class RequestFailed(Exception):
    """The child reported an error for one request; it is still running."""


class Cancelled(Exception):
    """The child confirmed a cancel; the answer is incomplete. It is still running."""


def _stop(proc) -> None:
    """Never raises: it runs in cleanup paths, where it must not hide the first error."""
    if proc is None:
        return
    try:
        if proc.is_alive():
            proc.terminate()
            proc.join(2)
            if proc.is_alive():
                proc.kill()
                proc.join(2)
    except Exception:  # noqa: BLE001 — a process that never started, or already reaped
        pass


def _close(*conns) -> None:
    for conn in conns:
        if conn is not None:
            try:
                conn.close()
            except OSError:
                pass


def _receive(conn, proc, deadline, *, interval, max_bytes, watched=None, rss_limit=None):
    """Wait for one message; raise ChildFailed on time, death, or memory.

    Limits are checked once per interval: a message that arrives up to one interval
    after the deadline is still accepted, and memory can pass rss_limit between checks.
    A message longer than max_bytes is refused before it is unpickled.
    """
    while True:
        if conn.poll(interval):
            try:
                return pickle.loads(conn.recv_bytes(max_bytes))
            except OSError:  # longer than max_bytes
                raise ChildFailed(f"message over {max_bytes} bytes") from None
            except EOFError:  # the child died: a closed pipe polls as readable
                proc.join(interval)
                raise ChildFailed(f"exited with {proc.exitcode}") from None
        if not proc.is_alive():
            raise ChildFailed(f"exited with {proc.exitcode}")
        if rss_limit is not None:
            try:
                if watched.memory_info().rss > rss_limit:
                    raise ChildFailed("memory limit")
            except psutil.NoSuchProcess:
                raise ChildFailed("exited") from None
        if time.monotonic() > deadline:
            raise ChildFailed("time limit")


# --- one-shot child: a parser, one document per process -----------------------

def run_once(target, payload, *, timeout, rss_limit, max_bytes, interval=0.2):
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
        result = _receive(recv, proc, deadline, interval=interval, max_bytes=max_bytes,
                          watched=watched, rss_limit=rss_limit)
        proc.join(interval)
        return result
    finally:
        _stop(proc)
        _close(recv, send)


# --- long-lived child: the model ---------------------------------------------

class ModelChild:
    """The model process. Used only from the model worker thread."""

    def __init__(self, target, args, *, load_timeout, chunk_timeout,
                 max_bytes=1 << 20, interval=0.2):
        self._target, self._args = target, args
        self._max_bytes = max_bytes
        self._load_timeout, self._chunk_timeout = load_timeout, chunk_timeout
        self._interval = interval
        self._proc = self._to_child = self._from_child = None

    def start(self) -> None:
        ctx = mp.get_context("spawn")
        child_in = child_out = None
        try:
            child_in, self._to_child = ctx.Pipe(duplex=False)
            self._from_child, child_out = ctx.Pipe(duplex=False)
            self._proc = ctx.Process(target=self._target,
                                     args=(child_in, child_out, *self._args), daemon=True)
            self._proc.start()
            _close(child_in, child_out)
            child_in = child_out = None
            deadline = time.monotonic() + self._load_timeout
            if _receive(self._from_child, self._proc, deadline, interval=self._interval,
                        max_bytes=self._max_bytes) != ("ready",):
                raise ChildFailed("no ready message")
        except BaseException:
            _close(child_in, child_out)
            self.stop()
            raise

    def request(self, message, cancelled=lambda: False):
        """Yield the chunks of one answer.

        Ends normally only for a complete answer. Raises Cancelled or RequestFailed with
        the child still running, or ChildFailed with it stopped.
        """
        if self._proc is None or not self._proc.is_alive():
            self.start()
        outcome = None
        try:
            self._to_child.send(("request", message))
            cancel_sent = False
            while True:
                if cancelled() and not cancel_sent:
                    self._to_child.send(("cancel",))
                    cancel_sent = True
                deadline = time.monotonic() + self._chunk_timeout
                kind, *body = _receive(self._from_child, self._proc, deadline,
                                       interval=self._interval, max_bytes=self._max_bytes)
                if kind == "chunk":
                    yield body[0]
                elif kind == "done":
                    break
                elif kind == "cancelled":
                    outcome = Cancelled()
                    break
                elif kind == "error":
                    outcome = RequestFailed(body[0])
                    break
        except BaseException:  # includes the worker abandoning the generator
            self.stop()
            raise
        if outcome is not None:
            raise outcome

    def stop(self) -> None:
        _stop(self._proc)
        _close(self._to_child, self._from_child)
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
            status = ("done",)
            for chunk in answer(model, body[0]):
                outbox.send(("chunk", chunk))
                if inbox.poll(0) and inbox.recv() == ("cancel",):
                    status = ("cancelled",)
                    break
            outbox.send(status)
        except Exception as exc:
            outbox.send(("error", repr(exc)))
```

- Limits are checked once per `interval`: a message that arrives within one interval
  after the deadline is still accepted, and memory can pass `rss_limit` between checks.
- A message is read with `recv_bytes(max_bytes)` and unpickled only after its length is
  known, so an oversized result never enters the app's memory. Once `poll` reports data,
  the read waits for the whole message; a child that froze in the middle of writing one
  is not covered — messages are small (one chunk, or text under the character limit).
- `ChildFailed`: the child hung, died, sent too much, or passed the memory limit; it has
  been stopped, and the next `request` starts a new one. `RequestFailed`: the model raised
  for one request. `Cancelled`: the child confirmed a cancel. With both, the child keeps
  running. A `request` that ends without raising is a complete answer — the only one the
  UI may show as complete.
- Cancel: the worker passes `cancelled`, checked between chunks; the child stops at its
  next chunk and answers `cancelled`. A child that sends nothing within `chunk_timeout`
  is stopped.
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

The parser child's entry point. Two different memory limits:

- `rss_limit`: resident memory, polled by `run_once` on every OS.
- `address_space_limit`: Linux only, set by the child with `resource.RLIMIT_AS` as a hard
  cap on virtual address space, so a runaway allocation fails in the child before the
  next poll. It is not a resident-memory limit: libraries reserve address space they never
  touch, so it is set well above `rss_limit` — the ADR records both and the documents they
  were tried on. Windows has no standard-library equivalent; macOS was not checked.

```python
def parse_main(conn, payload):
    install_network_guard()
    data, parse, max_chars, address_space_limit = payload  # parse: module-level function
    if sys.platform == "linux":
        import resource
        resource.setrlimit(resource.RLIMIT_AS, (address_space_limit, address_space_limit))
    text = parse(data)
    if len(text) > max_chars:
        conn.send(("error", f"text over {max_chars} characters"))
    else:
        conn.send(("text", text))
```

## 6. Agent tool writes (rule §4.5)

`application/` decides what to write; this adapter writes it. A new file is created with
`open(..., "xb")`, so a name that appears between the check and the write is refused, not
overwritten. An overwrite takes the `Target` captured when the confirmation was shown and
refuses if the file changed since; it writes a temporary file in the same folder and
replaces the original with `os.replace`, so a crash leaves either the old file or the new
one.

```python
import os
import tempfile
from dataclasses import dataclass
from pathlib import Path


class WriteRefused(Exception):
    pass


@dataclass(frozen=True)
class Target:
    """What the user confirmed: the path and the file as it was when shown."""
    path: Path
    size: int
    mtime_ns: int
    inode: int


def snapshot(path: Path) -> Target:
    st = path.stat()
    return Target(path, st.st_size, st.st_mtime_ns, st.st_ino)


def create_new(path: Path, data: bytes) -> None:
    """A new file: fails if the name exists by the time it is written (no-clobber)."""
    try:
        with open(path, "xb") as f:
            f.write(data)
    except FileExistsError:
        raise WriteRefused(f"{path} now exists; an overwrite needs confirmation") from None


def overwrite(confirmed: Target, data: bytes) -> None:
    """Replace a confirmed file atomically, only if it is still the file that was shown."""
    if snapshot(confirmed.path) != confirmed:
        raise WriteRefused(f"{confirmed.path} changed after it was confirmed")
    fd, tmp = tempfile.mkstemp(dir=confirmed.path.parent, prefix=".tmp-")
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(data)
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp, confirmed.path)
    except BaseException:
        Path(tmp).unlink(missing_ok=True)
        raise
```

## 7. Self-check additions (rule §5.3)

These run wherever the archetype's `--self-check` runs, the packaged build included
(`pyside-release`). The self-check runs under the same guard as the product (§4), in the
app and in every child it starts. Then:

- Connect to `192.0.2.1` (a documentation-only address) and fail unless the guard refuses
  it, and have each child report `guard_installed()` — proves the guard is installed in
  the packaged build, in every process.
- Start a `ModelChild` whose `load` imports `llama_cpp` and returns its version instead of
  a model; fail unless it answers — proves `spawn` and the native library work in the
  packaged build.
- Open the temp database through the normal `Engine`, fail if `VecStatus.error` is set,
  print `vec_version()`, create a `vec0` table with dimension 3, insert one vector, search
  for it, and fail if it is not returned.

## 8. Test setup (rule §6)

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
that hangs, exits, raises, and streams normally, with a cancel after the first chunk
(which must raise `Cancelled`) and a request after each failure; a parser that sends more
than `max_bytes`; and an argument that cannot be pickled, so that
`Process.start()` fails and the original error surfaces with nothing left open.

Tests for §6: a create over an existing name is refused and the file unchanged; an
overwrite of a file changed after `snapshot` is refused and the file unchanged; a
confirmed overwrite leaves no temporary file.

Tests for §4 call each API in its table directly — the refused ones with an outside
address or name and with a loopback port opened by another process, the allowed ones
with endpoints this process bound, `socketpair`, `ssl`, an `asyncio` loop and a `spawn`
pipe. Refused cases include the same port on another loopback address, UDP to a bound TCP
port, and a Unix socket bound by another process. A connection factory (`connect_args={"factory": ...}`) whose
`enable_load_extension` raises tests the three paths of §3.

## 9. CI (rule §5.5)

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
refuses both. A child-watching loop with psutil 7.2.2 ends a child that hangs (time limit) and
one that allocates past the limit (memory limit), reports one that exits without
answering (a closed pipe polls as readable and raises `EOFError`), and returns a normal
child's result.
`resource.setrlimit(RLIMIT_AS)` also works in a Linux spawn child.

2026-11-16 (third pass), same platform, the code of §4, §5 and §6 as now written: a
Unix socket and a loopback port bound by another process are refused, and the port
accepted once registered with `allow_endpoint`; the same port on `127.0.0.2` and UDP to a
bound TCP port are refused; the process's own Unix socket and loopback port work; the
guard is installed in a `spawn` child before the target's module is imported. A cancel
raises `Cancelled` and the same child serves the next request; a parser message over
`max_bytes` is refused. `create_new` refuses an existing name; `overwrite` refuses a file
changed after `snapshot`, replaces atomically otherwise, and leaves no temporary file.

2026-11-16 (second pass), same platform, the code of §3, §4 and §5 as now written:
every refused API in §4's table refuses, including a loopback port opened by another
process, and accepts it once registered; every allowed one works; `getfqdn` returns
without a lookup. A `Process.start()` failure (unpicklable argument) surfaces as
`PicklingError` with the process and pipes cleaned up. The loader keeps the connection
when `enable_load_extension` is missing or raises, and fails it when switching off fails.
The §1 compile command resolves `scikit-build-core 1.1.2`, `packaging 26.3`, `pathspec
1.1.1`.

2026-11-16 (first pass), same platform: the guard allows a loopback
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
