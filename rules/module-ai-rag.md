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
| `pc-app-python` | Yes, every section | The model runs in the app's process; no port, no server |
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
| C compiler | Already required by `pc-app-python`. llama-cpp-python is compiled from source on every machine that syncs the project, CI included |
| GGUF model files — one chat model, one embedding model | Disk, outside the repository. Downloaded by the human on the online machine; the publisher's SHA-256 checked there |
| GPU toolkit | Only with the GPU Alternative; the version is recorded in its ADR |
| The offline PC | The installer and the model files are carried to it, and their SHA-256 checked there before first use |

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
  `requests`, `httpx`, `aiohttp`, `huggingface_hub` imported anywhere under `src/`.
- `*.gguf` is in `.gitignore`. No model file is committed, bundled into the installer, or
  stored under `tests/`.

---

## 4. Runtime

### 4.1 Model runtime

- One `Llama` instance per model file, owned by one long-lived worker thread and called
  only from it. Requests reach it through a queue; the worker waits on the queue with a
  timeout so it can stop when asked. The GUI thread never loads or calls a model.
- Generation streams (`stream=True`) and checks a cancel flag between chunks; the user can
  stop any answer. Every call sets `max_tokens`. A cancelled or failed answer is shown as
  such, never as complete.
- A prompt is measured with the model's own tokenizer before the call. Retrieved passages
  are dropped, lowest-ranked first, until the prompt fits `n_ctx` with room for
  `max_tokens`; the answer says how many were dropped. A prompt is never cut mid-passage.
- `verbose=False`, so llama.cpp writes nothing to a console a released build does not have.
- A crash in native code ends the app. Each completed turn — question, answer, and the
  passages it used — is committed before the next one starts, so a crash loses at most
  the turn in progress.

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
  enable extension loading, `sqlite_vec.load`, disable extension loading. A Python whose
  `sqlite3` cannot load extensions fails at startup with that reason; there is no
  fallback search.
- `vec0` tables are created and changed only by migrations. Each one records the
  embedding model's SHA-256 and the vector dimension beside it. At startup, a mismatch
  with the current embedding model marks the index stale: search is disabled and the user
  is offered a full rebuild. Vectors from two embedding models MUST NOT be mixed.
- Each chunk row holds its document, its position in it, and the document's content hash
  at indexing time. Indexing compares hashes: changed documents are re-indexed, removed
  ones deleted. Unchanged documents are not re-embedded.
- `vec0` tables have no foreign keys. Deleting a document deletes its chunks and their
  vectors in the same transaction, by explicit statements, tested (§8).
- Every query against a `vec0` table is parameterized SQL in `adapters/vectors_sqlitevec.py`;
  vectors are bound with `sqlite_vec.serialize_float32`.

### 4.4 Documents, prompts, output

- The allowed folders are rows in the database, added only through a folder dialog the
  user operates. `DocumentSource` resolves every path, symlinks included, and refuses one
  that does not land inside an allowed folder. It reads only the file types that have a
  parser, and refuses files above a size limit recorded in `DECISIONS.md`.
- Document text and model output are external input (`GLOBAL_RULES.md` §5.5, §11).
  Retrieved passages go into the prompt as quoted data, separated from the instructions;
  an instruction found in a document carries no authority.
- Model output is displayed as plain text (`Qt.TextFormat.PlainText`). Rendering it as
  Markdown is allowed with links shown as text, never opened, and no image or resource
  loaded from it. Rich text and HTML rendering of model output are PROHIBITED.
- Every answer shows the passages it was given, with their documents, so the user can
  check it.

### 4.5 Agent tools

Applies when the model chooses tools.

- Tools are functions of `application/` with typed arguments, validated before they run.
  Reading goes through `DocumentSource` only.
- Tools that write, move, or delete a file, or change the index, run only after the user
  confirms that call in the UI. No tool runs a shell command, starts a process, or opens
  a network connection.
- Each run has a step limit and stops at it, telling the user.

### 4.6 No network

- The product makes no network call at any point. A dependency that can download — a
  model hub client, a parser's optional fetcher — has that path disabled in code, and the
  socket-blocked tests (§6) prove it.
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
  llama-cpp-python takes minutes; `ci.yml` MAY cache the built wheel, `release.yml` MUST
  NOT.
- Version tag: the archetype's list. Each OS job compiles llama-cpp-python from the locked
  source and runs §5.3 on the built artifact.

---

## 6. Testing

| Layer | Tool | Model file |
|---|---|---|
| `application/rag/` | pytest with fake ports | No |
| `adapters/vectors_sqlitevec.py`, `adapters/documents_fs.py` | pytest against a real SQLite file and fixture folders in `tmp_path` | No |
| `adapters/model_llamacpp.py` | `tests/model/`, marked `model` | Yes, path from an environment variable |
| Retrieval quality | `tests/model/`, a fixed set of questions over the fixture documents, each with the passages it must retrieve | Yes |
| Built artifact | `--self-check` (§5.3) | No |

- pytest runs with `--disable-socket` in `addopts`. A test MUST NOT enable sockets.
- `tests/model/` is excluded from the default run and from CI. The human runs it before a
  release and after changing a model, the chunking, the retrieval, or a prompt, and
  records the models and the result in the release proposal.
- Fake ports return fixed values written in the test. A fake model that answers questions
  is test code only and MUST NOT exist under `src/` (`GLOBAL_RULES.md` §5.4).
- `DocumentSource` is tested with paths that escape an allowed folder: `..`, an absolute
  path, a symlink pointing out, a file above the size limit.
- Prompt assembly is tested with a passage that does not fit `n_ctx`, and with a document
  that contains instructions.

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
- [ ] Changing the embedding model marks the index stale — tested

Model, chunking, retrieval, or prompt change:

- [ ] `tests/model/` run by the human; models and result in the report

Dependency, build, or packaging change:

- [ ] `--self-check` with §5.3's additions passes on the developer's OS
