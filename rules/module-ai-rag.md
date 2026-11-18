# rules/module-ai-rag.md

An open-source model run inside the app on the user's own PC — chat, search, or an agent
over the user's own files — whose own code sends nothing outside the machine, run on a PC
with no network so that nothing else can (§4.6).

`GLOBAL_RULES.md` → `rules/<archetype>.md` and this file → `CLAUDE.md` → current task. The
archetype rule says how the app is built; this file says how it runs a model and searches
the user's documents. Where both speak, the stricter applies. This file holds only what
`GLOBAL_RULES.md` and the archetype rule do not already say. **Default** = use it;
**Alternative** = allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column
covers technology choices only; every other prohibition sits with the rule it belongs to.

Not this module: using an AI tool to write the code (`GLOBAL_RULES.md`), or a product that
calls a hosted model (no rule yet; the first such project decides it by ADR).

---

## 1. Stack

**Applies to** a project from the first commit that adds a model, an embedding, or a
vector search to the shipped product. Development happens on an online machine; the
product runs on a PC that may have no network at all.

| Archetype | Applies | Why |
|---|---|---|
| `pc-app-python` | Yes, every section | The Default runs the model in a child process with no port; the `llama-server` Alternative opens one, on 127.0.0.1 only |
| `pc-app-electron` | Not yet | No local model runtime has been chosen for Node; the first project that needs one decides it by ADR |
| `web-static`, `web-python`, `web-ts`, `web-api` | No | The model would run on a server, so the user's documents would leave their PC |
| `mobile` | No | No on-device runtime has been chosen |
| `agent-plugin` | No | The host's model does the work |

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Model runtime | llama-cpp-python in a child process the app starts (`multiprocessing` spawn), talking over pipes, no port | llama.cpp `llama-server`, started and stopped by the app, bound to 127.0.0.1, a new random API key per launch, its binary bundled in the installer | The model in the app's own process or thread; a resident daemon (Ollama); a server bound to any other address; a runtime's built-in model download |
| llama-cpp-python install | Built from the PyPI source distribution, CPU, hash in `uv.lock` | A GPU build; a prebuilt wheel from the project's own wheel index, declared as an explicit uv index | Any other third-party wheel or fork |
| Model files | GGUF chosen by the user; path and SHA-256 recorded (§4.2) | — | Pickle-based weights (`.bin`, `.pt`, `.pth`, `.pkl`); `trust_remote_code`; a model name or path fixed in code |
| Embeddings | A GGUF embedding model on the same runtime | — | A second runtime for embeddings only (sentence-transformers, PyTorch) |
| Vector store | sqlite-vec inside the app's SQLite file, exact version | — | A vector database server (Qdrant, Chroma, Milvus); sqlite-vss; vectors in a file apart from the documents they index (FAISS index files) |
| Document parsing | One parser per file type the app reads, chosen by ADR: runs offline, locked in `uv.lock`; memory watched with psutil (§4.4) | — | A parser that calls a service or starts processes of its own; OCR or conversion through a hosted API |
| Ingestion, chunking, retrieval | Project-owned code | — | A framework adopted for a single pipeline |
| Orchestration | None | LangChain or LlamaIndex: core package plus local integrations only, every model, embedding and store set explicitly in code | Their hosted-service integrations |

---

## 2. Prerequisites

Procedure: skill `local-model-setup`.

| Item | Scope |
|---|---|
| C compiler | Already required by `pc-app-python`. llama-cpp-python is compiled from source once per machine, CI included; uv's cache reuses that build until the lock or the build options change |
| GGUF model files — one chat model, one embedding model | Needed only to run `tests/model/`. Outside the repository; the human provides them and sets their paths in environment variables |
| GPU toolkit | Only with the GPU Alternative; the version is recorded in its ADR |
| `llama-server` binary | Only with that Alternative; an official llama.cpp release for each declared OS, its version and SHA-256 recorded in the ADR, bundled in the installer, never updated by the app |

---

## 3. Layout

Additions to the archetype layout:

```
<repo>/
├── src/<app>/
│   ├── application/
│   │   ├── ports.py         + LLMProvider, EmbeddingProvider, VectorStore, DocumentSource
│   │   └── rag/             chunking, retrieval, prompt assembly, agent loop — pure Python
│   └── adapters/
│       ├── model_llamacpp.py      LLMProvider and EmbeddingProvider: the worker side
│       ├── model_child.py         the model child process: the only module importing llama_cpp
│       ├── model_llamaserver.py   llama-server Alternative only, instead of the two above
│       ├── vectors_sqlitevec.py   VectorStore
│       ├── netguard.py            the runtime network guard (§4.6)
│       ├── children.py            starting and watching child processes
│       └── documents_fs.py        DocumentSource, one module per parser beside it
└── tests/
    ├── fixtures/documents/  made-up documents only
    └── model/               tests that need a real model file
```

| Port | Behind it | Job |
|---|---|---|
| `LLMProvider` | llama-cpp-python | Generate an answer, streamed, cancellable |
| `EmbeddingProvider` | The same runtime | Turn text into vectors |
| `VectorStore` | sqlite-vec | Store, delete and search vectors |
| `DocumentSource` | Folders the user has allowed | List and read source documents; refuse any path outside the allowed folders |

- `DocumentSource` is the only code that opens a user document. Tests feed it fixtures; the
  allowed-folder check lives in it and nowhere else.
- `tests/test_boundaries.py` also fails on `llama_cpp` imported outside
  `adapters/model_child.py`, on `sqlite_vec` imported from `domain/` or `application/`,
  and on any of `socket`, `_socket`, `urllib.request`, `http.client`, `requests`, `httpx`,
  `aiohttp`, `huggingface_hub`, `PySide6.QtNetwork` imported anywhere under `src/`. The exemptions: `adapters/netguard.py` imports `socket`, and with the
  `llama-server` Alternative its adapter (`adapters/model_llamaserver.py`) MAY import
  `http.client`.
- `*.gguf` is in `.gitignore`. No model file is committed, bundled into the installer, or
  stored under `tests/`.

---

## 4. Runtime

### 4.1 Model runtime

- Default: the model runs in one child process started with `multiprocessing` (`spawn`)
  and owned by one worker thread. The child holds one `Llama` instance per model file,
  chat and embedding alike; requests and streamed chunks travel over pipes, and no port is
  opened. The GUI thread never loads or calls a model.
- A native call cannot be interrupted, and a token limit is not a time limit. The worker
  therefore waits on every pipe read with a timeout recorded in `DECISIONS.md` (a longer
  one for loading a model). On expiry, or when the child dies, it terminates the child,
  marks the request failed, and starts a new child for the next request. The worker thread
  itself always ends.
- Generation streams (`stream=True`) and checks a cancel flag between chunks. Every call
  sets `max_tokens`. A cancel takes effect at the next chunk, and the child reports it as
  cancelled, never as done; if no chunk arrives within the timeout, the child is
  terminated as above. Closing the app does the same. Only an answer the child reports as
  done is shown as complete; a cancelled or failed one is shown as such.
- Every message from a child is read with a size limit and refused above it before it is
  decoded.
- With the `llama-server` Alternative, the same rules apply with the server in place of
  the child: the app checks the binary's SHA-256, starts it with `--host 127.0.0.1`, an
  unused port and a new random API key, and treats it as ready only when its health
  endpoint answers within the load timeout. Each HTTP call has its own timeout. A cancel
  closes the connection; if the health endpoint then does not answer within the timeout,
  the server is terminated and started again. It is terminated when the app closes.
- A prompt is measured with the model's own tokenizer before the call. Retrieved passages
  are dropped, lowest-ranked first, until the prompt fits `n_ctx` with room for
  `max_tokens`; the answer says how many were dropped. A prompt is never cut mid-passage.
- `verbose=False`, so llama.cpp writes nothing to a console a released build does not have.
- If the app keeps a history, each completed turn is committed before the next one
  starts, so a failure loses at most the turn in progress.

### 4.2 Model files

- Model and embedding model are chosen by the user through the UI: file path, its SHA-256,
  and the source the user named. They are rows in the app's database, not entries in the
  settings file.
- The first time a file is chosen, the user pastes the SHA-256 listed on the model's
  official download page; the dialog says so. The app computes the file's hash and
  refuses the file when the pasted value is empty or different. The computed hash is
  never accepted in place of the publisher's.
- With the hash, the app records the file's size and modification time. Before each load
  it compares both; if either differs, it computes the SHA-256 again on a worker, showing
  that it is checking the file, and refuses a file whose hash no longer matches. A change that keeps both — silent disk
  corruption — is not detected.
- A model file is opened only as GGUF, and only by the runtime: the Default's child
  process, or `llama-server --model`. `Llama.from_pretrained`, `llama-server`'s download
  options (`-hf`, `--hf-repo`), and every other code path that downloads are PROHIBITED.
- Changing the embedding model invalidates the index (§4.3).

### 4.3 Index

- The sqlite-vec extension is loaded on every connection by a SQLAlchemy `connect` event:
  enable extension loading, `sqlite_vec.load`, and disable extension loading again whether
  the load succeeded or not.
- Every load failure is caught inside that event and never raised from it; raising there
  makes every connection fail. The one exception: if extension loading cannot be switched
  off again, the error propagates and that connection fails rather than stay open with
  loading enabled. The cause goes into one status object owned by the adapter. A
  failure on any connection sets it, and nothing clears it until the app restarts — not a
  later connection that loads the extension. The application reads it: the features that need the index
  are disabled and the cause is logged and shown; everything else keeps working.
- There is no fallback when the index is unavailable: no vectors kept elsewhere, no
  brute-force search in Python, no other search's results shown in its place. A keyword
  search the spec asks for as a feature of its own is not a fallback.
- `vec0` tables are created and changed only by migrations. Each one records the
  embedding model's SHA-256 and the vector dimension beside it. At startup, a mismatch
  with the current embedding model marks the index stale: search is disabled and the user
  is offered a full rebuild. Vectors from two embedding models MUST NOT be mixed.
- Each chunk row holds its document, its position in it, and the document's content hash
  at indexing time. Indexing compares hashes: changed documents are re-indexed, removed
  ones deleted. Unchanged documents are not re-embedded.
- Re-indexing a document computes all its new chunks and embeddings first, then replaces
  its old chunks and vectors in one transaction. A failure at any step leaves the old
  entries searchable and the document listed as not indexed.
- `vec0` tables have no foreign keys. Deleting a document deletes its chunks and their
  vectors in the same transaction, by explicit statements, tested (§8).
- The total number of documents indexed and the number of chunks per document have
  limits recorded in `DECISIONS.md`. At either limit indexing stops for that document or
  run and tells the user why.
- Before writing each document's entries, indexing checks the free space on the
  database's disk and stops the run below a reserve recorded in `DECISIONS.md`, so the
  rest of the app can still write. A write that fails anyway — a full disk included —
  rolls back its transaction, keeps the old entries, and stops the run.
- Every query against a `vec0` table is parameterized SQL in `adapters/vectors_sqlitevec.py`;
  vectors are bound with `sqlite_vec.serialize_float32`.

### 4.4 Documents, prompts, output

- The allowed folders are rows in the database, added only through a folder dialog the
  user operates. `DocumentSource` resolves every path, symlinks included, and refuses one
  that does not land inside an allowed folder. It reads only the file types that have a
  parser, and refuses files above a size limit recorded in `DECISIONS.md`. The limit is
  enforced on the read itself — at most one byte past it is read, and then the file is
  refused — not only by a size check before opening.
- A parser can hang, crash, or exhaust memory on a malformed file, and a worker thread
  cannot be stopped (archetype §4.1). Parsing therefore runs in a child process started with
  `multiprocessing` (`spawn`), one document at a time. `DocumentSource` opens and checks
  the file in the app's process and passes the child its bytes, never a path. The child
  returns text only and never touches the database.
- The child refuses to send text longer than a maximum number of characters. The worker
  watches the child with a time limit and polls its resident memory (RSS) with `psutil`
  against an RSS limit. On expiry, crash, oversized output, or RSS above the limit, it
  terminates the child and marks that document as failed.
- Both limits are checked once per polling interval: a result arriving within one interval
  after the time limit is still accepted, and RSS can pass its limit between two polls.
  On Linux the child also sets an address-space limit (`RLIMIT_AS`), which fails a
  runaway allocation at once. It limits virtual address space, not RSS, so it is a
  separate, higher value. Outside Linux nothing stops memory between polls.
- The time, character, RSS and address-space limits and the interval are recorded in
  `DECISIONS.md`.
- A parser that starts processes of its own is not adopted.
- Out of scope: a process that can already write to the allowed folders or to a model file
  — a symlink swapped after the path check, a file replaced after its hash was computed.
  No defence against it is built.
- Document text and model output are external input (`GLOBAL_RULES.md` §5.5, §11).
  Retrieved passages go into the prompt as quoted data, separated from the instructions;
  an instruction found in a document carries no authority.
- Model output is displayed as plain text (`Qt.TextFormat.PlainText`). Rendering it as
  Markdown is allowed with links shown as text, never opened, and no image or resource
  loaded from it. Rich text and HTML rendering of model output are PROHIBITED.

### 4.5 Agent tools

Applies when the model chooses tools.

Which tools exist, their limits, and where they may write are decided in the app's spec.
This section is the floor every such app keeps.

- Tools are functions of `application/` with typed arguments, validated before they run.
  Reading goes through `DocumentSource` only.
- Every permission is enforced by the code that executes the tool, never by instructions
  in the prompt: which tools this run may call, which paths each may read or write, and
  which calls need the user's confirmation. A call that fails any check is refused and
  reported to the model as refused.
- Writes go only to the folders the spec allows, checked the same way `DocumentSource`
  checks reads. Writing to a name that already exists is an overwrite and needs
  confirmation like any other change to an existing file.
- A new file is created so that the create fails if the name exists by then; it never
  overwrites. An overwrite acts only on the file that was confirmed: if its size,
  modification time or identity changed since, it is refused. It writes a temporary file
  in the same folder and atomically replaces the original, so a crash leaves the old file
  or the new one.
- Each write has a size limit, each run a limit on the total written, and no write goes
  below the disk reserve of §4.3; all are recorded in `DECISIONS.md`.
- No tool runs a shell command, starts a process, or opens a network connection.
- A tool that changes, moves, or deletes an existing file, or changes the index, runs
  only after the user confirms that call. The confirmation shows the tool, every target
  path, the action, and the content: for a new file what will be written, for an
  overwrite the difference from the current file.
- The loop that lets the model call tools has a maximum number of model calls, recorded in
  `DECISIONS.md`. At the limit it stops and tells the user; it never continues unbounded.

### 4.6 No network

- No connection leaves the machine, and none reaches another local service, which could
  relay it outside. The app's own code opens only one kind of network connection: the
  `llama-server` Alternative's adapter, to the port of the server it started. A dependency
  that can download — a model hub client, a parser's optional fetcher — has that path
  disabled in code.
- At run time the network guard is installed by the first statements of the app's entry
  script, at module level, before any other import; a `spawn` child runs that script
  before anything else, so the guard is in place in every child before its code runs.
  Child entry functions install it again as their first statement. Through the `socket`
  module's API it refuses: connections, datagrams and binds to any address that is not
  loopback; connections to any loopback endpoint or Unix socket path the process did not
  bind itself — an endpoint being family, type, address and port — except the one
  endpoint the `llama-server` adapter registers; and name lookups (`getaddrinfo`,
  `gethostbyname`, `gethostbyname_ex`, `gethostbyaddr`, `getnameinfo`) for any host that
  is not loopback. `multiprocessing` pipes and endpoints the process bound itself stay
  allowed; `multiprocessing` managers and listeners across processes do not, and are
  not used.
- Before release: the boundary check (§3) catches network libraries imported by the app's
  own code; the socket-blocked tests (§6) and `--self-check` (§5.3) catch Python code that
  opens a socket on the paths they run.
- What none of these cover: native code — a parser or llama.cpp compromised by a crafted
  file, a library's C extension — and code that imports `_socket` directly. A child process isolates hangs, crashes and memory; it
  is not a sandbox, and runs with the user's permissions. Against native code the
  protection is a PC with no network. Every new dependency is reviewed for network use
  before it is added, and the release is run once on a PC with no network before it is
  handed to anyone.
- Real user documents never reach the development machine or the coding agent's context.
  Development, tests, and bug reproduction use made-up documents under
  `tests/fixtures/documents/`. A bug seen only with a real document is reproduced by the
  human on the offline PC and described in words or as a made-up equivalent.

### 4.7 Settings recorded in `DECISIONS.md`

Every limit above is a project value. No code path uses one that is not recorded, and a
release is not proposed while any is missing.

| Setting | Unit | Applies to |
|---|---|---|
| Model load timeout, chunk timeout | seconds | Model child, `llama-server` (§4.1) |
| Child message size | bytes | Every child (§4.1) |
| Document size | bytes | `DocumentSource` (§4.4) |
| Parse time limit, polling interval | seconds | Parser child (§4.4) |
| Parsed text length | characters | Parser child (§4.4) |
| RSS limit; address-space limit (Linux) | bytes | Parser child (§4.4) |
| Documents indexed; chunks per document | count | Indexing (§4.3) |
| Disk reserve | bytes | Indexing, tool writes (§4.3, §4.5) |
| Tool write size; total per run | bytes | Agent tools (§4.5) |
| Model calls per run | count | Agent loop (§4.5) |

---

## 5. Build and package

Procedure: skill `local-model-setup`.

### 5.1 Lock the toolchain

- llama-cpp-python and sqlite-vec are pinned to exact versions. Every build dependency of
  llama-cpp-python (scikit-build-core, CMake, and what they pull in) is pinned in
  `[tool.uv] build-constraint-dependencies`, which `uv.lock` records. An unpinned build
  dependency is fetched at its newest version on every clean build.
- A change to any of these is a dependency change. sqlite-vec is pre-1.0: an upgrade
  rebuilds a copy of a real index and compares search results before release.

### 5.2 Build

- The default build is CPU. A GPU build is the Alternative; its ADR names the toolkit
  version and the declared target OSes it covers.
- The native libraries of llama-cpp-python and the sqlite-vec extension are package data
  the build MUST include. A missing one is caught by §5.3, not at a user's first question.
- With the `llama-server` Alternative, the build bundles the binary recorded in the ADR for
  each declared OS, and checks its SHA-256.

### 5.3 Self-check

Additions to the archetype's `--self-check`, in its temp directory, with no model file:

- Start the model child process without a model and have it report `llama_cpp`'s version
  — proves `spawn` works in the packaged build and the native library loads.
- With the `llama-server` Alternative instead: check the bundled binary's SHA-256, start
  it without a model, and get an answer from its health endpoint.
- Load sqlite-vec into the temp database, print `vec_version()`, create a `vec0` table,
  insert one vector, find it — proves the extension ships and can be loaded.
- Under the network guard, in every child too; fail unless a connection to an outside
  address is refused — proves the guard is installed in the packaged build.

### 5.4 Package

- Model files are never inside the installer. `README.md` lists the models the app was
  tested with, their sources and SHA-256, and how to choose them in the app.
- `README.md` also states that the app's own code opens no connection that leaves the
  machine — with the `llama-server` Alternative, that it talks to it over 127.0.0.1 only;
  that native code compromised by a crafted document or model file is stopped only by
  running on a PC with no network, and that this risk remains on a networked PC; and how
  to carry the installer and models to an offline PC and check them there.

### 5.5 CI

- Push / PR: the archetype's list, with pytest under `--disable-socket`. Compiling
  llama-cpp-python takes minutes; `ci.yml` MAY cache the CPU build, built with
  `CMAKE_ARGS` unset and keyed on the OS, the architecture, the runner image version, the
  Python version and the hash of `uv.lock`. GPU builds and `release.yml` MUST NOT use a
  cache.
- Version tag: the archetype's list. Each OS job compiles llama-cpp-python from the locked
  source and runs §5.3 on the built artifact.

---

## 6. Testing

| Layer | Tool | Model file |
|---|---|---|
| `application/rag/` | pytest with fake ports | No |
| `adapters/vectors_sqlitevec.py`, `adapters/documents_fs.py` | pytest against a real SQLite file and fixture folders in `tmp_path` | No |
| `adapters/model_llamacpp.py` | `tests/model/`, marked `model` | Yes, path from an environment variable |
| Retrieval quality | `tests/model/`, a fixed set of questions over the fixture documents. Each lists the IDs of its expected passages and a `k`; it passes when at least one expected passage is in the top `k` | Yes |
| Built artifact | `--self-check` (§5.3) | No |

- pytest runs with `--disable-socket` in `addopts`. A test MUST NOT enable sockets. The
  one exception: the `llama-server` adapter's tests under `tests/model/` run with
  `--allow-hosts=127.0.0.1` added, which still refuses every other address.
- The model child is tested with fake children that hang and that exit: the request
  fails, the child is terminated, and the next request starts a new one.
- A cancel during generation is tested: the child reports it as cancelled, the answer is
  not shown as complete, and the next request on the same model succeeds. A child message
  over the size limit is refused.
- Agent tools are tested with calls the spec does not allow — an unlisted tool, a path
  outside the allowed folders, a new file whose name exists — each refused by the executor.
- Every call that needs confirmation is tested three ways: before confirmation nothing
  changes; after confirmation exactly the shown targets change, with the shown content;
  after refusal nothing changes. A create over a name that appeared after the check, and
  an overwrite of a file changed after confirmation, are refused with the file unchanged.
  The write size, run total and disk reserve limits refuse a write that passes them.
- The network guard is tested by calling each API it covers directly: every refused case
  of §4.6, including a loopback port and a Unix socket opened by another process, the same
  port on another loopback address, and UDP to a bound TCP port; and what stays allowed —
  endpoints the process bound, the registered `llama-server` endpoint, `socketpair`,
  `ssl`, `asyncio`, `spawn` pipes. A spawned child reports that the guard was installed
  before its module was imported.
- Child-process cleanup is tested with a start that fails inside `Process.start()`: the
  original error surfaces, and no process or pipe is left open.
- sqlite-vec loading is also tested with a `sqlite3` that cannot load extensions: the
  connection stays usable and the index features report it.
- The index limits are tested: indexing stops at the document and chunk limits and below
  the disk reserve, and a write that fails mid-document keeps the old entries.
- The model-file check is tested: an unchanged file loads without rehashing; a changed size
  or modification time triggers a rehash, and a changed hash refuses the load.
- `tests/model/` is excluded from the default run and from CI. The human runs it before a
  release and after changing a model, the chunking, the retrieval, or a prompt. Run
  explicitly, a missing model file is a failure, never a skip. The release task's report gives
  the models' SHA-256 and the passed, failed and skipped counts; a skip means the run did
  not happen.
- Fake ports return fixed values written in the test. A fake model that answers questions
  is test code only and MUST NOT exist under `src/` (`GLOBAL_RULES.md` §5.4).
- `DocumentSource` is tested with paths that escape an allowed folder: `..`, an absolute
  path, a symlink pointing out, and a file above the size limit, including one that
  grows past it after the size check.
- Prompt assembly is tested with a passage that does not fit `n_ctx`, and with a document
  that contains instructions.
- sqlite-vec is tested failing to load: connections still open, ordinary queries work,
  the index features report the cause, extension loading is off again, and a later
  connection that loads it does not re-enable the index features.
- The parsing child process is tested with fake parsers that hang, crash, return too much
  text, and allocate past the memory limit: each is terminated, the document marked
  failed, and the next document still parsed.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. Older formats and libraries recalled from training: GGML `.bin` model files from before
   GGUF; sqlite-vss instead of sqlite-vec; LangChain imports from before its packages were
   split (`langchain.vectorstores`, `langchain.llms`).
2. Answering a failed llama-cpp-python build by switching to Ollama, an HTTP runtime, or a
   wheel from an unvetted index. A build error is not grounds for any Alternative's ADR.
3. Fetching a model so a test can run: `Llama.from_pretrained`, `huggingface-cli download`,
   a URL in a fixture. A test that needs a model goes under `tests/model/` and waits for
   the human's file.
4. Turning a failing check off: `@pytest.mark.enable_socket`, removing `--disable-socket`,
   removing or delaying `install_network_guard()` because a library fails under it,
   skipping the hash check because hashing is slow, `allow_dangerous_deserialization=True`
   on a pickled index.
5. Truncating a prompt by characters, or raising `n_ctx` past what the model was trained
   for, to make an overflow error disappear.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check passes, including the network imports of §3
- [ ] Test suite passes with sockets blocked; only the `llama-server` adapter's model
      tests allow 127.0.0.1, and nothing allows any other address
- [ ] Every process entry point installs the network guard first
- [ ] No real user document in the repository, the fixtures, or the task's report
- [ ] Every value in §4.7 that the task's code uses is recorded in `DECISIONS.md`

Index or schema change:

- [ ] Deleting a document leaves no chunk or vector behind — tested
- [ ] A re-index failing mid-document leaves the old entries searchable — tested
- [ ] Changing the embedding model marks the index stale — tested

Model, chunking, retrieval, or prompt change:

- [ ] `tests/model/` run by the human: models' SHA-256, passed / failed / skipped counts,
      zero skipped

Dependency, build, or packaging change:

- [ ] `--self-check` with §5.3's additions, for the runtime the project uses, passes on
      the developer's OS
