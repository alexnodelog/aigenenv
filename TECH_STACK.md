# TECH_STACK.md

**What this is.** The official technology selections for every project built in this
framework. `GLOBAL_RULES.md` decides *how* to build; this document decides *what to
build with*.

**Precedence.** `GLOBAL_RULES.md` → `TECH_STACK.md` → `rules/<project>.md` → current task.
A project rule MAY narrow a choice made here. It MUST NOT widen one.

**RFC 2119.** MUST / MUST NOT / SHOULD / SHOULD NOT / MAY carry their RFC 2119 meaning.

**How to read a selection.** Every entry is one of three things:

| Marking | Meaning |
|---|---|
| **Default** | Use this unless the project rule says otherwise. No justification needed. |
| **Alternative** | Allowed, but the reason MUST be recorded in the project's `DECISIONS.md`. |
| **Prohibited** | MUST NOT be used. Listed because it is a plausible-looking wrong answer. |

---

## 1. Selection Principles

Priority order, highest first:

1. Maintainability
2. Simplicity
3. Consistency
4. Developer Experience
5. AI Readability
6. Vendor Independence
7. Long-term Sustainability
8. Extensibility
9. Performance
10. Trend

Performance and trends MUST NEVER outweigh maintainability.

Every tool selected here MUST be free, allow commercial use, be installable on a
company-owned PC, and be actively maintained. Always-free services are preferred over
time-limited free tiers.

---

## 2. Project Types

A project picks **exactly one archetype** (what it deploys as) and **zero or more
modules** (what capabilities it has).

| Archetype | File | Use when |
|---|---|---|
| PC App — Python | `rules/pc-app-python.md` | Desktop tool, Python-only, no web UI needed |
| PC App — Electron | `rules/pc-app-electron.md` | Desktop app with a rich web UI or heavy Node ecosystem use |
| Web — Monolithic | `rules/web-monolithic.md` | Web service deployed as one unit (default web choice) |
| Web — Full-Stack | `rules/web-fullstack.md` | Frontend and backend deployed and scaled separately |
| Mobile | `rules/mobile.md` | iOS / Android application |

| Module | File | Adds |
|---|---|---|
| AI / RAG | `rules/module-ai-rag.md` | LLM, embedding, and vector search **at runtime** |
| Deployment | `rules/module-deployment.md` | Release automation, signing, distribution |

**AI at build time is not a module.** Using Claude, Cursor, or Copilot to write the code
is universal and belongs to `GLOBAL_RULES.md`. The AI/RAG module applies only when the
shipped product itself calls a model. A photo-classification desktop app written with AI
assistance but running purely offline is `pc-app-python` with **no** AI module.

---

## 3. Common Stack

Applies to every archetype.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Python runtime | Python 3.12+ | — | Python 3.11 and below |
| Python packaging | **uv** (`uv.lock`, `.python-version`) | — | Poetry, pipenv, bare pip in projects |
| Node runtime | Node.js LTS (`.nvmrc`) | — | — |
| Node packaging | **pnpm** (`pnpm-lock.yaml`) | — | npm, yarn |
| Frontend language | TypeScript, `strict: true` | — | Plain JavaScript in new code |
| Python lint + format | Ruff | — | Black + isort + flake8 (superseded) |
| Python typing | mypy, strict on domain code | pyright | — |
| TS lint + format | Biome | ESLint + Prettier | — |
| Python tests | pytest, pytest-cov | — | unittest-only |
| TS tests | Vitest | Jest (mobile only) | — |
| Config / secrets | `.env` + committed `.env.example` | — | Secrets in source or in Git |
| Documentation | Markdown | MkDocs | — |
| API specification | OpenAPI (generated, never hand-written) | — | — |
| Version control | Git + GitHub | — | — |
| CI | GitHub Actions | — | — |
| Commits | Conventional Commits | — | — |
| Pre-commit | pre-commit (lint, format, type check) | — | `--no-verify` |

**Why uv instead of Poetry.** uv resolves and installs an order of magnitude faster, ships
as a single binary, and replaces pip, virtualenv, pipx, and pyenv at once — which means
Python version pinning comes free and no PC needs a separate version manager installed.
This overrides the Poetry preference stated in the original philosophy document; the
reason is recorded as ADR-0001.

---

## 4. Development Environment

**Native first. Docker optional.**

Reproducibility is guaranteed by lockfiles, not by containers:

```
uv sync      # Python: exact deps + exact interpreter version
pnpm install # Node: exact deps
```

A project MUST start from a clean clone with those commands alone. A project MUST NOT
require Docker in order to run its tests or its development server.

Docker Compose is **RECOMMENDED for web archetypes only**, where the deployment artifact
is itself a container and local/production parity has real value. It is **not used** for
desktop and mobile archetypes, where GUI and emulator toolchains make containers a net
loss.

| Container runtime | Status |
|---|---|
| Docker Engine / Docker CE | Default |
| Podman Desktop, Rancher Desktop | Alternative — use where Docker Desktop licensing or install policy blocks it |

Dev Container configuration is OPTIONAL and MAY be added for web archetypes. It MUST NOT
become the only supported way to develop a project.

This overrides the "clone → docker compose up" workflow in the original philosophy
document; the reason is recorded as ADR-0002.

---

## 5. Archetype Stacks

### 5.1 PC App — Python

For desktop tools built entirely in Python.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| GUI framework | **PySide6** (Qt for Python, LGPLv3) | — | **PyQt6** — GPL or paid commercial only |
| UI construction | Qt Widgets, hand-written Python | Qt Designer `.ui` files, QML | — |
| Persistence | SQLite via SQLAlchemy | stdlib `sqlite3` for trivial cases | — |
| App data paths | `platformdirs` | — | Hard-coded paths, writing next to the executable |
| Settings format | TOML | JSON | INI, pickle |
| Async / long work | `QThread` + worker objects | `concurrent.futures` | Blocking the GUI thread |
| Tests | pytest + pytest-qt | — | — |
| Build | `pyside6-deploy` (Nuitka-based) | PyInstaller | — |
| Windows installer | Inno Setup | — | — |
| macOS / Linux | `.app` bundle / AppImage | — | — |

**Why PySide6 over PyQt6.** Both wrap Qt 6 with near-identical APIs. PySide6 is the
official Qt project binding and is LGPL, so a closed-source application can ship against
it when linked dynamically. PyQt6 is GPL or a paid commercial licence, which forces either
source disclosure or a purchase. That difference alone decides it.

**Build tradeoff.** `pyside6-deploy` wraps Nuitka, which compiles to native code: smaller
binaries, faster startup, builds measured in minutes. PyInstaller bundles bytecode:
builds in seconds, larger output, trivially decompilable. Use PyInstaller during
iteration if build time hurts; ship with `pyside6-deploy`.

Docker: not used.

---

### 5.2 PC App — Electron

For desktop apps whose UI is genuinely a web UI.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Shell | **Electron** | **Tauri 2** — when installer size or memory is a stated product requirement | — |
| Language | TypeScript | — | — |
| Renderer UI | React | — | — |
| Build tooling | electron-vite | — | — |
| Styling | Tailwind CSS | CSS Modules | — |
| Components | shadcn/ui | — | — |
| Renderer state | Zustand | — | Redux Toolkit for personal-scale apps |
| IPC | `contextBridge` over a typed, explicitly enumerated channel list | — | `nodeIntegration: true`, `contextIsolation: false`, exposing `ipcRenderer` wholesale |
| Persistence | better-sqlite3 in the main process | — | Direct DB access from the renderer |
| Packaging | electron-builder | Electron Forge | — |
| Tests | Vitest (unit) + Playwright (E2E) | — | — |

**Why Electron stays the default.** Tauri produces far smaller binaries and lower idle
memory, and those numbers are real. But it moves custom native behaviour into Rust and
pushes WebView compatibility testing onto the developer, while Electron keeps the whole
application in one language with a mature packaging and auto-update ecosystem. For a
single developer maintaining a project for years, adding a second language costs more than
a hundred megabytes on disk. Choose Tauri when small footprint is a requirement someone
actually asked for — not by default.

Docker: not used.

---

### 5.3 Web — Monolithic

Default choice for web services. One deployable unit: FastAPI serves both the API and the
pre-built frontend assets.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Backend framework | FastAPI | — | — |
| ORM | **SQLAlchemy 2.0** (`Mapped[]` declarative style) | SQLModel — single-user tools and throwaway PoCs only | — |
| Schemas | Pydantic v2, defined separately from ORM models | — | Reusing ORM models as API response models |
| Migrations | Alembic | — | Schema changes applied by hand |
| Database | SQLite | PostgreSQL | SQLite-specific SQL in business logic |
| Frontend | React + Vite | — | — |
| Styling | Tailwind CSS + shadcn/ui | — | — |
| Frontend state | Zustand | TanStack Query for server state | — |
| Server | Uvicorn | — | — |
| Tests | pytest + httpx (backend), Vitest + Playwright (frontend) | — | — |
| Packaging | Single Docker image | — | — |

**Why SQLAlchemy over SQLModel.** SQLModel collapses the ORM table and the API schema into
one class, which reads well and removes duplication — but that is exactly the leak
`GLOBAL_RULES.md` forbids: a column rename becomes an API contract change, and persistence
concerns reach the delivery layer. Restoring the boundary means writing separate Pydantic
schemas anyway, at which point SQLModel's advantage is gone. It also remains a `0.0.x`
release with a deliberately narrow SQLAlchemy version pin that has repeatedly blocked
upgrades of the dependency underneath it. SQLAlchemy 2.0's typed declarative style is
concise enough that the ergonomic gap no longer justifies the coupling.

Docker: recommended.

---

### 5.4 Web — Full-Stack

Frontend and backend as separate deployables, communicating over a versioned REST
boundary. Choose this only when separate deployment or separate scaling is an actual
requirement; otherwise use Monolithic.

Backend is identical to §5.3. Differences:

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Frontend framework | Next.js | — | — |
| Rendering | Server Components where they help; static export where possible | — | — |
| API boundary | Versioned prefix `/api/v1` | — | Unversioned public API |
| Client types | Generated from OpenAPI via `openapi-typescript` | — | Hand-maintained duplicate type definitions |
| CORS | Explicit allowlist | — | `allow_origins=["*"]` in production |
| Frontend hosting | Vercel | Static host + CDN | — |
| Backend hosting | Oracle Cloud Always Free | Self-hosted Linux | Anything requiring a paid tier at MVP scale |

Applications MUST NOT call hosting-provider-specific APIs from business logic.

Docker: recommended for the backend.

---

### 5.5 Mobile

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | React Native via **Expo** (managed workflow) | — | Bare React Native without a stated reason |
| Language | TypeScript | — | — |
| Navigation | Expo Router | React Navigation | — |
| State | Zustand | — | — |
| Local storage | expo-sqlite | AsyncStorage for key-value only | — |
| Secure storage | expo-secure-store | — | Tokens in AsyncStorage |
| Networking | fetch + generated OpenAPI types | — | — |
| Unit tests | Jest (`jest-expo`) | — | — |
| E2E | Maestro | — | — |
| Build | EAS Build | — | — |
| OTA updates | EAS Update | — | Shipping native changes as OTA |

**Distribution path.** Three stages, in order:

1. **Internal** — EAS Build `preview` profile, `distribution: internal`, Android APK by
   shareable link. No store account needed. The free EAS plan covers roughly 30 builds a
   month, which is ample at personal scale.
2. **Google Play** — one-time developer registration fee, AAB via the `production` profile.
3. **iOS** — deferred. Requires a yearly Apple Developer membership. No stack change is
   needed when that day comes: EAS Build compiles iOS in the cloud, so a Mac is not
   required.

Target floor: Android 7+, iOS 16.4+.

Docker: not used.

---

## 6. Module Stacks

### 6.1 AI / RAG Module

Applies only when the shipped product calls a model at runtime.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Local LLM runtime | Ollama | llama.cpp | — |
| Hosted LLM | Provider-agnostic, behind an adapter | — | Provider SDK types in business logic |
| Embeddings | sentence-transformers (local) | Hosted embedding API behind an adapter | — |
| Vector store | FAISS | sqlite-vec, pgvector, Qdrant | — |
| Chunking / ingestion | Project-owned code | — | Wholesale framework adoption for a single pipeline |
| Orchestration | None by default | LangChain / LlamaIndex only with a recorded reason | — |

Four interfaces MUST exist and MUST be owned by the application, not imported from a
vendor: `LLMProvider`, `EmbeddingProvider`, `VectorStore`, `Storage`. Swapping Ollama for a
hosted API, or FAISS for pgvector, MUST touch only the adapter implementing that
interface.

A project MAY ship exactly one implementation per interface. The rule requires the seam to
exist, not that alternatives be built in advance.

**On orchestration frameworks.** LangChain and LlamaIndex solve problems that appear at
team scale and add a large, fast-moving dependency surface in exchange. For a personal
project, a retrieval pipeline is usually under 200 lines of owned, readable code. Start
without one.

### 6.2 Deployment Module

| Target | Mechanism |
|---|---|
| Web | GitHub Actions → container image → Oracle Cloud Always Free or self-hosted |
| Desktop | GitHub Actions matrix build → GitHub Releases |
| Mobile | EAS Build → internal link, then store submission |
| Versioning | Semantic Versioning, tagged in Git, generated from Conventional Commits |

Release artifacts MUST be produced by CI, never from a developer's machine.

---

## 7. Cross-Cutting Architecture Requirements

Business logic MUST NOT import or reference, directly:

- SQLite, PostgreSQL, or any database driver
- FAISS, Qdrant, pgvector, or any vector store client
- OpenAI, Ollama, Gemini, or any model provider SDK
- Any cloud provider SDK
- Any web framework, GUI framework, or UI library

Access to each of these goes through an application-owned interface. Replacing any of them
MUST be confined to adapter code. If it is not, the boundary has failed and fixing it is
part of the change, not a follow-up task.

---

## 8. Changing This Document

A technology selection changes only when:

1. The reason is written down as an ADR in the affected project's `DECISIONS.md`, and
2. If the change should apply to all future projects, this document is edited in the same
   commit.

Git history is the version record. This document carries no version number, and neither do
its filenames. A selection that was replaced is deleted here and survives in the ADR that
replaced it.
