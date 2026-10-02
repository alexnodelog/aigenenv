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

Every tool selected here MUST be free, **allow commercial use**, be installable on a
company-owned PC, and be actively maintained. Always-free offerings are preferred over
time-limited trials. A free tier that forbids commercial use is treated as *not free* for
any project that might ever be monetized.

---

## 2. Choosing an Archetype

A project picks **exactly one archetype** (what it deploys as) and **zero or more
modules** (what capabilities it has).

| Archetype | File | Pick it when |
|---|---|---|
| PC App — Python | `rules/pc-app-python.md` | Desktop tool, Python only, no web UI |
| PC App — Electron | `rules/pc-app-electron.md` | Desktop app whose UI is genuinely a web UI |
| Web — Static | `rules/web-static.md` | No server logic. Content, docs, calculators, portfolio |
| Web — Python | `rules/web-python.md` | Server logic in Python, no separate frontend build |
| Web — TypeScript | `rules/web-ts.md` | Real app UI, and nothing forces Python on the server |
| Web — Split | `rules/web-api.md` | Real app UI **and** Python needed on the server |
| Mobile | `rules/mobile.md` | iOS / Android application |

| Module | File | Adds |
|---|---|---|
| AI / RAG | `rules/module-ai-rag.md` | LLM, embedding, vector search **at runtime** |
| Deployment | `rules/module-deployment.md` | Release automation, signing, distribution |

**Decision order for web projects.** Start at the top and stop at the first "yes":

1. Is there any server logic at all? No → `web-static`.
2. Is the audience internal, or is it a dashboard / data tool / PoC? Yes → `web-python`.
3. Does the server genuinely need Python libraries (ML, scientific, pandas, a Python-only
   SDK)? No → `web-ts`. Yes → continue.
4. → `web-api`.

Choose `web-api` only after answering yes to step 3. It is the only archetype whose free
hosting spans two platforms, and every CORS problem, cold-start problem, and
duplicated deploy pipeline in this framework originates there.

**AI at build time is not a module.** Using Claude, Cursor, or Copilot to write the code is
universal and belongs to `GLOBAL_RULES.md`. The AI/RAG module applies only when the
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
as a single binary, and replaces pip, virtualenv, pipx, and pyenv at once — so Python
version pinning comes free and no PC needs a separate version manager installed. This
overrides the Poetry preference in the philosophy document; recorded as ADR-0001.

---

## 4. Development Environment

### 4.1 Native first, Docker optional

Reproducibility is guaranteed by lockfiles, not by containers:

```
uv sync      # Python: exact deps + exact interpreter version
pnpm install # Node: exact deps
```

A project MUST start from a clean clone with those commands alone. A project MUST NOT
require Docker to run its tests or its development server.

Docker Compose is RECOMMENDED for web archetypes whose deployment artifact is itself a
container. It is not used for `web-static`, desktop, or mobile archetypes, where GUI and
emulator toolchains make containers a net loss.

| Container runtime | Status |
|---|---|
| Docker Engine / Docker CE | Default |
| Podman Desktop, Rancher Desktop | Alternative — where Docker Desktop licensing or install policy blocks it |

Dev Container configuration is OPTIONAL. It MUST NOT become the only supported way to
develop a project. This overrides the "clone → docker compose up" workflow in the
philosophy document; recorded as ADR-0002.

### 4.2 Editor and agent orchestration

| Role | Default | Add when |
|---|---|---|
| Editor | VS Code + a CLI coding agent | — |
| Parallel agent workspace | Orca | Two or more agents need to work at once on separate features |
| Terminal agent multiplexer | Herdr | Long-running processes, remote machines, or agents that must watch each other's output |

Start with VS Code alone. Orca and Herdr overlap substantially — both give worktree
isolation and multi-pane agent sessions — so running all three at once is tool sprawl, not
rigour. Add the second tool only when a concrete limitation forces it.

These are recommendations, not requirements. The binding rules are in `GLOBAL_RULES.md`
and are stated in terms of behaviour, not tools: every parallel agent gets its own git
worktree; every agent gets an explicit directory scope; an agent that fails the same way
three times stops and asks a human. Any tool that satisfies those rules is acceptable, and
the rules survive these tools being replaced.

---

## 5. Archetype Stacks

### 5.1 PC App — Python

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| GUI framework | **PySide6** (Qt for Python, LGPLv3) | — | **PyQt6** — GPL or paid commercial only |
| UI construction | Qt Widgets, hand-written Python | Qt Designer `.ui` files, QML | — |
| Persistence | SQLite via SQLAlchemy | stdlib `sqlite3` for trivial cases | — |
| App data paths | `platformdirs` | — | Hard-coded paths, writing beside the executable |
| Settings format | TOML | JSON | INI, pickle |
| Long-running work | `QThread` + worker objects | `concurrent.futures` | Blocking the GUI thread |
| Tests | pytest + pytest-qt | — | — |
| Build | `pyside6-deploy` (Nuitka-based) | PyInstaller | — |
| Windows installer | Inno Setup | — | — |
| macOS / Linux | `.app` bundle / AppImage | — | — |

**Why PySide6 over PyQt6.** Both wrap Qt 6 with near-identical APIs. PySide6 is the
official Qt binding and is LGPL, so a closed-source application can ship against it when
linked dynamically. PyQt6 is GPL or a paid commercial licence, forcing either source
disclosure or a purchase.

**Build tradeoff.** `pyside6-deploy` wraps Nuitka and compiles to native code: smaller
binaries, faster startup, builds in minutes. PyInstaller bundles bytecode: builds in
seconds, larger output, trivially decompilable. Iterate with PyInstaller if build time
hurts; ship with `pyside6-deploy`.

### 5.2 PC App — Electron

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Shell | **Electron** | **Tauri 2** — when installer size or memory is a stated product requirement | — |
| Renderer UI | React + TypeScript | — | — |
| Build tooling | electron-vite | — | — |
| Styling | Tailwind CSS + shadcn/ui | CSS Modules | — |
| Renderer state | Zustand | — | Redux Toolkit at personal scale |
| IPC | `contextBridge`, typed, explicitly enumerated channels | — | `nodeIntegration: true`, `contextIsolation: false`, exposing `ipcRenderer` wholesale |
| Persistence | better-sqlite3 in the main process | — | Direct DB access from the renderer |
| Packaging | electron-builder | Electron Forge | — |
| Tests | Vitest + Playwright | — | — |

**Why Electron stays the default.** Tauri's smaller binaries and lower idle memory are
real, but it moves custom native behaviour into Rust and pushes WebView compatibility
testing onto the developer, while Electron keeps the whole application in one language
with a mature packaging and auto-update ecosystem. For one developer maintaining a project
for years, a second language costs more than a hundred megabytes on disk.

### 5.3 Web — Static

No server process. Everything is built once and served from a CDN.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Language | HTML, CSS, TypeScript | — | Untyped JavaScript |
| Build | Vite | Astro — content-heavy sites with many pages | A framework chosen before a need for one exists |
| Styling | Tailwind CSS | Plain CSS | — |
| Interactivity | Standard DOM APIs | Alpine.js | React for a page with three buttons |
| Forms / backend calls | Hosting platform's form handling or edge function | — | — |

**Promotion path.** When this archetype needs real server logic, it becomes `web-python` or
`web-ts`. Do not grow a backend inside a static site through edge functions; that is the
signal to change archetype.

### 5.4 Web — Python

One Python process serves everything. No Node, no bundler, no `node_modules`.

Two tracks. Both deploy identically, so a project MAY start on one and move to the other.

**Track A — Streamlit.** Internal tools, dashboards, data apps, PoCs.

| Category | Default |
|---|---|
| Framework | Streamlit |
| Charts | Streamlit built-ins, Plotly |
| State | `st.session_state` |

Streamlit reruns the entire script top to bottom on every interaction, keeps session state
per user *and per tab*, offers no routing or authentication, and gives almost no control
over CSS. Those are design choices, not bugs, and they are fine for a tool with a handful
of known users. They are disqualifying for a public service. If the project needs login,
URL routing, or a designed UI, it is Track B or it is `web-ts`.

**Track B — FastAPI + server-rendered HTML.** Public-facing apps that do not need a
JavaScript framework.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | FastAPI | — | — |
| Templates | Jinja2 | — | — |
| Interactivity | htmx | Alpine.js for local client state | React, Vue, a bundler |
| Styling | Tailwind CSS via its standalone CLI | Plain CSS | A Node-based asset pipeline |
| ORM | SQLAlchemy 2.0 (`Mapped[]` declarative) | — | — |
| Schemas | Pydantic v2, separate from ORM models | — | ORM models as API response models |
| Migrations | Alembic | — | Hand-applied schema changes |
| Server | Uvicorn | — | — |
| Tests | pytest + httpx | — | — |

**Agent scoping rule for this archetype.** Business logic lives under `services/` and
contains no UI calls. Presentation lives in templates and route handlers and contains no
business logic. When several agents work in parallel here, each is scoped to one side of
that line. This split is what makes the archetype survivable as it grows.

### 5.5 Web — TypeScript

One language, one platform, one deployment.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | Next.js | — | — |
| Server logic | Route handlers / server actions | — | A separate backend service — that is `web-api` |
| Styling | Tailwind CSS + shadcn/ui | — | — |
| Client state | Zustand | — | — |
| Server state | TanStack Query | — | — |
| Database | Cloudflare D1 (SQLite) | PostgreSQL | — |
| ORM | Drizzle | — | — |
| Tests | Vitest + Playwright | — | — |

### 5.6 Web — Split

Next.js frontend, FastAPI backend, deployed separately. Backend is identical to §5.4
Track B minus the templates.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Frontend | Next.js | — | — |
| Backend | FastAPI | — | — |
| API boundary | Versioned prefix `/api/v1` | — | Unversioned public API |
| Client types | Generated from OpenAPI via `openapi-typescript` | — | Hand-maintained duplicate types |
| CORS | Explicit allowlist, environment-driven | — | `allow_origins=["*"]` in production |
| Cold start | Documented and handled in the UI | — | Pretending it will not happen |

### 5.7 Mobile

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | React Native via **Expo** (managed) | — | Bare React Native without a stated reason |
| Navigation | Expo Router | React Navigation | — |
| State | Zustand | — | — |
| Local storage | expo-sqlite | AsyncStorage for key-value only | — |
| Secure storage | expo-secure-store | — | Tokens in AsyncStorage |
| Unit tests | Jest (`jest-expo`) | — | — |
| E2E | Maestro | — | — |
| Build | EAS Build | — | — |
| OTA updates | EAS Update | — | Shipping native changes as OTA |

**Distribution path**, in order:

1. **Internal** — EAS `preview` profile, `distribution: internal`, Android APK by link. No
   store account. The free EAS plan covers roughly 30 builds a month.
2. **Google Play** — one-time developer registration fee, AAB from the `production` profile.
3. **iOS** — deferred. Requires a yearly Apple Developer membership. No stack change is
   needed when that day comes: EAS Build compiles iOS in the cloud, so a Mac is not
   required.

Target floor: Android 7+, iOS 16.4+.

---

## 6. Hosting

All selections below are free tiers that permit commercial use, except where noted.

| Archetype | Default | Alternative | Notes |
|---|---|---|---|
| `web-static` | Cloudflare Pages | GitHub Pages, Netlify | Cloudflare: unlimited bandwidth, 500 builds/month, 100 sites. GitHub Pages requires a public repo. Netlify now meters a monthly credit allowance |
| `web-python` | Render (free web service) | Streamlit Community Cloud (Track A), Oracle Cloud Always Free | Render free: 512 MB RAM, sleeps after idle, 30–60 s cold start. Move to Oracle Always Free when always-on matters |
| `web-ts` | Cloudflare Workers | **Vercel — non-commercial projects only** | Cloudflare D1 provides free SQLite |
| `web-api` | Cloudflare Pages (frontend) + Render (backend) | Oracle Cloud Always Free (both) | The only archetype spanning two platforms on free tiers |
| Desktop | GitHub Releases | — | — |
| Mobile | EAS Build → internal link → Google Play | — | — |

**Vercel.** Its free Hobby plan is restricted to non-commercial use. Any project that
might be monetized MUST NOT depend on it. Cloudflare carries no such restriction, which is
why it is the default here despite Vercel's first-party Next.js support.

**Platform independence.** Hosting is the single most volatile item in this document —
free-tier terms change yearly. Business logic MUST NOT call hosting-provider-specific APIs.
Deployment configuration lives in its own files and MUST be replaceable without touching
application code. Verify current free-tier terms before starting any new project; do not
trust this table's limits without checking.

---

## 7. Module Stacks

### 7.1 AI / RAG Module

Applies only when the shipped product calls a model at runtime.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Local LLM runtime | Ollama | llama.cpp | — |
| Hosted LLM | Provider-agnostic, behind an adapter | — | Provider SDK types in business logic |
| Embeddings | sentence-transformers (local) | Hosted embedding API behind an adapter | — |
| Vector store | FAISS | sqlite-vec, pgvector, Qdrant | — |
| Ingestion / chunking | Project-owned code | — | A framework adopted for a single pipeline |
| Orchestration | None by default | LangChain / LlamaIndex with a recorded reason | — |

Four interfaces MUST exist and MUST be owned by the application, not imported from a
vendor: `LLMProvider`, `EmbeddingProvider`, `VectorStore`, `Storage`. Swapping Ollama for a
hosted API, or FAISS for pgvector, MUST touch only the adapter behind that interface.

A project MAY ship exactly one implementation per interface. The rule requires the seam to
exist, not that alternatives be built in advance.

**On orchestration frameworks.** LangChain and LlamaIndex solve problems that appear at
team scale and add a large, fast-moving dependency surface in exchange. A personal
retrieval pipeline is usually under 200 lines of owned, readable code. Start without one.

### 7.2 Deployment Module

| Target | Mechanism |
|---|---|
| Web | GitHub Actions → platform deploy hook or container image |
| Desktop | GitHub Actions matrix build → GitHub Releases |
| Mobile | EAS Build → internal link, then store submission |
| Versioning | Semantic Versioning, Git tags, generated from Conventional Commits |

Release artifacts MUST be produced by CI, never from a developer's machine.

---

## 8. Cross-Cutting Architecture Requirements

Business logic MUST NOT directly import or reference:

- SQLite, PostgreSQL, D1, or any database driver
- FAISS, Qdrant, pgvector, or any vector store client
- OpenAI, Ollama, Gemini, or any model provider SDK
- Any hosting or cloud provider SDK
- Any web framework, GUI framework, or UI library

Each goes through an application-owned interface. Replacing any of them MUST be confined
to adapter code. If it is not, the boundary has failed, and fixing it is part of the
change rather than a follow-up task.

---

## 9. Changing This Document

A technology selection changes only when:

1. The reason is written as an ADR in the affected project's `DECISIONS.md`, and
2. If the change should apply to all future projects, this document is edited in the same
   commit.

Git history is the version record. This document carries no version number, and neither do
its filenames. A replaced selection is deleted here and survives in the ADR that replaced
it.
