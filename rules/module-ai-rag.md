# rules/module-ai-rag.md

An open-source model run inside the app on the user's own PC — chat, search, or an agent
over the user's own files — with nothing it reads leaving the machine.

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
| `pc-app-python` | Yes, every section | The Default runs the model in the app's process, with no port; the `llama-server` Alternative opens one, on 127.0.0.1 only |
| `pc-app-electron` | Not yet | No in-process runtime has been chosen for Node; the first project that needs one decides it by ADR |
| `web-static`, `web-python`, `web-ts`, `web-api` | No | The model would run on a server, so the user's documents would leave their PC |
| `mobile` | No | No on-device runtime has been chosen |
| `agent-plugin` | No | The host's model does the work |

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Model runtime | llama-cpp-python, in the app's own process | llama.cpp `llama-server`, started and stopped by the app, bound to 127.0.0.1, a new random API key per launch | A resident daemon (Ollama); a server bound to any other address; a runtime's built-in model download |
| llama-cpp-python install | Built from the PyPI source distribution, CPU, hash in `uv.lock` | A GPU build; a prebuilt wheel from the project's own wheel index, declared as an explicit uv index | Any other third-party wheel or fork |
| Model files | GGUF chosen by the user; path and SHA-256 recorded (§4.2) | — | Pickle-based weights (`.bin`, `.pt`, `.pth`, `.pkl`); `trust_remote_code`; a model name or path fixed in code |
| Embeddings | A GGUF embedding model on the same runtime | — | A second runtime for embeddings only (sentence-transformers, PyTorch) |
| Vector store | sqlite-vec inside the app's SQLite file, exact version | — | A vector database server (Qdrant, Chroma, Milvus); sqlite-vss; vectors in a file apart from the documents they index (FAISS index files) |
| Document parsing | One parser per file type the app reads, chosen by ADR: runs offline, locked in `uv.lock` | — | A parser that calls a service; OCR or conversion through a hosted API |
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
| `llama-server` binary | Only with that Alternative; an official llama.cpp release, its version and SHA-256 recorded in the ADR, never updated by the app |

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
│       ├── model_llamacpp.py      LLMProvider and EmbeddingProvider
│       ├── vectors_sqlitevec.py   VectorStore
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
- `tests/test_boundaries.py` also fails on `llama_cpp` or `sqlite_vec` imported from
  `domain/` or `application/`, and on any of `socket`, `urllib.request`, `http.client`,
  `requests`, `httpx`, `aiohttp`, `huggingface_hub`, `PySide6.QtNetwork` imported anywhere
  under `src/`. The one exemption: with the `llama-server` Alternative, its adapter
  (`adapters/model_llamaserver.py`) MAY import `http.client`.
- `*.gguf` is in `.gitignore`. No model file is committed, bundled into the installer, or
  stored under `tests/`.

---

## 4. Runtime

### 4.1 Model runtime

- One `Llama` instance per model file, owned by one long-lived worker thread and called
  only from it. Requests reach it through a queue; the worker waits on the queue with a
  timeout so it can stop when asked. The GUI thread never loads or calls a model.
- Generation streams (`stream=True`) and checks a cancel flag between chunks. Every call
  sets `max_tokens`. A cancelled or failed answer is shown as such, never as complete.
- A native model call cannot be interrupted from Python. It still ends: prompt processing
  is bounded by `n_ctx` and generation by `max_tokens`, so the worker meets the archetype's
  termination rule without a timeout on the call itself. A cancel therefore takes effect
  at the next chunk, or after prompt processing finishes; until then the UI shows that it
  is stopping. Closing the app waits for the current chunk the same way. After a cancel,
  the same `Llama` instance serves the next request.
- With the `llama-server` Alternative, the app starts the server as a child process with
  `--host 127.0.0.1`, an unused port and a new random API key, and terminates it on close
  and on any timeout. Each HTTP call to it is blocking I/O with its own timeout; a cancel
  closes the connection and, if the server does not recover, restarts it.
- A prompt is measured with the model's own tokenizer before the call. Retrieved passages
  are dropped, lowest-ranked first, until the prompt fits `n_ctx` with room for
  `max_tokens`; the answer says how many were dropped. A prompt is never cut mid-passage.
- `verbose=False`, so llama.cpp writes nothing to a console a released build does not have.
- A crash in native code ends the app. If the app keeps a history, each completed turn is
  committed before the next one starts, so a crash loses at most the turn in progress.

### 4.2 Model files

- Model and embedding model are chosen by the user through the UI: file path, its SHA-256,
  and the source the user named. They are rows in the app's database, not entries in the
  settings file.
- The app computes the SHA-256 on a worker before every load, and refuses to load a file
  whose hash differs from the recorded one. The first time a file is chosen, the app
  shows the computed hash and the user confirms it against the publisher's value.
- A file is opened only as GGUF through llama-cpp-python. `Llama.from_pretrained` and
  every other code path that downloads are PROHIBITED.
- Changing the embedding model invalidates the index (§4.3).

### 4.3 Index

- The sqlite-vec extension is loaded on every connection by a SQLAlchemy `connect` event:
  enable extension loading, `sqlite_vec.load`, and disable extension loading again whether
  the load succeeded or not.
- A load failure is caught inside that event and never raised from it; raising there makes
  every connection fail. The cause goes into one status object owned by the adapter, which
  each new connection updates. The application reads it: the features that need the index
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
- Every query against a `vec0` table is parameterized SQL in `adapters/vectors_sqlitevec.py`;
  vectors are bound with `sqlite_vec.serialize_float32`.

### 4.4 Documents, prompts, output

- The allowed folders are rows in the database, added only through a folder dialog the
  user operates. `DocumentSource` resolves every path, symlinks included, and refuses one
  that does not land inside an allowed folder. It reads only the file types that have a
  parser, and refuses files above a size limit recorded in `DECISIONS.md`.
- A parser can hang or crash on a malformed file, and a worker thread cannot be stopped
  (archetype §4.1). Parsing therefore runs in a child process started with
  `multiprocessing` (`spawn`), one document at a time. `DocumentSource` opens and checks
  the file in the app's process and passes the child its bytes, never a path. The child
  returns text only and never touches the database.
- The worker waits on the child with a time limit, and accepts at most a maximum text
  length; both are recorded in `DECISIONS.md`. On expiry, crash, or oversized output it
  terminates the child and marks that document as failed. Memory is bounded only through
  the file size limit; a project that needs more records it by ADR.
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
  checks reads. A new file never replaces an existing one; a name that already exists is
  an overwrite and needs confirmation.
- No tool runs a shell command, starts a process, or opens a network connection.
- A tool that changes, moves, or deletes an existing file, or changes the index, runs
  only after the user confirms that call. The confirmation shows the tool, every target
  path, and the action.
- The loop that lets the model call tools has a maximum number of model calls, set in the
  spec. At the limit it stops and tells the user; it never continues unbounded.

### 4.6 No network

- No connection leaves the machine. The only socket the product may open is the
  `llama-server` Alternative's connection to 127.0.0.1, from its adapter alone. A
  dependency that can download — a model hub client, a parser's optional fetcher — has
  that path disabled in code.
- What the checks cover: the boundary check (§3) catches network libraries imported by the
  app's own code; the socket-blocked tests (§6) and `--self-check` (§5.3) catch Python code
  that opens a socket on the paths they run.
- What they do not cover: native code and child processes. Every new dependency is
  therefore reviewed for network use before it is added, and the release is run once on a
  PC with no network before it is handed to anyone.
- Real user documents never reach the development machine or the coding agent's context.
  Development, tests, and bug reproduction use made-up documents under
  `tests/fixtures/documents/`. A bug seen only with a real document is reproduced by the
  human on the offline PC and described in words or as a made-up equivalent.

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

### 5.3 Self-check

Additions to the archetype's `--self-check`, in its temp directory, with no model file:

- Import `llama_cpp` and print its version — proves the native library loads.
- Load sqlite-vec into the temp database, print `vec_version()`, create a `vec0` table,
  insert one vector, find it — proves the extension ships and can be loaded.
- With all sockets blocked for the whole run.

### 5.4 Package

- Model files are never inside the installer. `README.md` lists the models the app was
  tested with, their sources and SHA-256, and how to choose them in the app.
- `README.md` also states that the app makes no network calls, and how to carry the
  installer and models to an offline PC and check them there.

### 5.5 CI

- Push / PR: the archetype's list, with pytest under `--disable-socket`. Compiling
  llama-cpp-python takes minutes; `ci.yml` MAY cache the CPU build, keyed on the OS, the
  architecture, the Python version and the hash of `uv.lock`. GPU builds and
  `release.yml` MUST NOT use a cache.
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

- pytest runs with `--disable-socket` in `addopts`. A test MUST NOT enable sockets. Only
  the `llama-server` adapter's tests under `tests/model/` run with
  `--allow-hosts=127.0.0.1`.
- A cancel during generation is tested: the answer is marked cancelled and the next
  request on the same model succeeds.
- Agent tools are tested with calls the spec does not allow — an unlisted tool, a path
  outside the allowed folders, a new file whose name exists — each refused by the executor.
- `tests/model/` is excluded from the default run and from CI. The human runs it before a
  release and after changing a model, the chunking, the retrieval, or a prompt. Run
  explicitly, a missing model file is a failure, never a skip. The release task's report gives
  the models' SHA-256 and the passed, failed and skipped counts; a skip means the run did
  not happen.
- Fake ports return fixed values written in the test. A fake model that answers questions
  is test code only and MUST NOT exist under `src/` (`GLOBAL_RULES.md` §5.4).
- `DocumentSource` is tested with paths that escape an allowed folder: `..`, an absolute
  path, a symlink pointing out, a file above the size limit.
- Prompt assembly is tested with a passage that does not fit `n_ctx`, and with a document
  that contains instructions.
- sqlite-vec is tested failing to load: connections still open, ordinary queries work,
  the index features report the cause, and extension loading is off again.
- The parsing child process is tested with a fake parser that hangs and one that crashes:
  each is terminated, the document marked failed, and the next document still parsed.

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
   skipping the hash check because hashing is slow, `allow_dangerous_deserialization=True`
   on a pickled index.
5. Truncating a prompt by characters, or raising `n_ctx` past what the model was trained
   for, to make an overflow error disappear.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check passes, including the network imports of §3
- [ ] Test suite passes with sockets blocked; no test enables them
- [ ] No real user document in the repository, the fixtures, or the task's report

Index or schema change:

- [ ] Deleting a document leaves no chunk or vector behind — tested
- [ ] A re-index failing mid-document leaves the old entries searchable — tested
- [ ] Changing the embedding model marks the index stale — tested

Model, chunking, retrieval, or prompt change:

- [ ] `tests/model/` run by the human: models' SHA-256, passed / failed / skipped counts,
      zero skipped

Dependency, build, or packaging change:

- [ ] `--self-check` with §5.3's additions passes on the developer's OS
