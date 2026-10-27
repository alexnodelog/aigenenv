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
| Settings format | TOML | JSON — when the app writes settings back | INI, pickle |
| Long-running work | `QThread` + worker objects | `concurrent.futures` | Blocking the GUI thread |
| Tests | pytest + pytest-qt | — | — |
| Build | `pyside6-deploy` (Nuitka-based) | PyInstaller | — |
| Windows installer | Inno Setup | — | — |
| macOS / Linux | `.app` bundle / AppImage via `appimagetool` | Linux: `.tar.gz` of the standalone dir | — |

**Why PySide6 over PyQt6.** Both wrap Qt 6 with near-identical APIs. PySide6 is the
official Qt binding and is LGPL, so a closed-source application can ship against it when
linked dynamically. PyQt6 is GPL or a paid commercial licence, forcing either source
disclosure or a purchase.

**Build tradeoff.** `pyside6-deploy` wraps Nuitka and compiles to native code: smaller
binaries, faster startup, builds in minutes. PyInstaller bundles bytecode: builds in
seconds, larger output, trivially decompilable. Iterate with PyInstaller if build time
hurts; ship with `pyside6-deploy`.

**Build toolchain is locked.** Left alone, `pyside6-deploy` pip-installs Nuitka and its
helpers on first run. Under uv that either fails or installs versions `uv.lock` does not
record. The rule declares them in a locked `build` dependency group instead.

**Settings writer.** stdlib `tomllib` reads TOML but cannot write it. An app that saves its
own settings needs a writer, and JSON is the stdlib one.

**SQLite backup.** In WAL mode recent writes can sit in the `-wal` file, so copying the
`.db` file alone is not a consistent snapshot. `sqlite3.Connection.backup` is.

**AppImage.** `appimagetool` turns the standalone directory into an AppImage. Pin a tagged
release: the `continuous` release changes under the same URL. Licence not yet verified —
check before first use.

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

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | Streamlit | — | Gradio, Dash, Panel, NiceGUI |
| Charts | Streamlit built-ins, Plotly | — | — |
| State | `st.session_state` | — | — |
| Login | `st.login` (OIDC — Google, Microsoft, …) + an email allowlist | — | A password form written in Streamlit; streamlit-authenticator |

Streamlit reruns the entire script top to bottom on every interaction, keeps session state
per user *and per tab*, and gives almost no control over CSS. Its login tells the app *who*
the user is through an external identity provider; it has no accounts of its own and no
notion of roles or permissions. Those are design choices, not bugs, and they are fine for a
tool with a handful of known users. They are disqualifying for a public service. If the
project needs its own accounts, designed URLs, or a designed UI, it is Track B or it is
`web-ts`.

**Track B — FastAPI + server-rendered HTML.** Public-facing apps that do not need a
JavaScript framework.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | FastAPI | — | Flask, Django, Litestar |
| Templates | Jinja2, autoescape on | — | HTML built in Python strings |
| Interactivity | htmx 2.x, vendored into the repository | htmx 4; Alpine.js for local client state | React, Vue, a bundler; scripts loaded from a third-party CDN at runtime |
| Styling | Tailwind CSS v4 via its standalone CLI | Plain CSS | A Node-based asset pipeline; the Tailwind Play CDN |
| Schemas | Pydantic v2, separate from ORM models | — | ORM models as API response models |
| Accounts | Users table in the app's own database, server-side sessions | — | Accounts held by a hosting or database vendor's auth service |
| Password hashing | pwdlib (Argon2) | — | passlib — unmaintained, broken on Python 3.13+ |
| User file uploads | None by default; storage chosen by ADR | — | The local disk of a host without a persistent disk |
| Server | Uvicorn | — | — |
| Tests | pytest + httpx | — | — |

**Why htmx 2, not 4.** htmx 4.0 shipped in August 2026 with renamed events and changed
defaults. The htmx project keeps 2.x as its `latest` release until early 2027 and supports
it indefinitely, and coding agents have been trained on 2.x. Choosing 4 is an ADR, and the
agent must check which major version a project vendors before writing htmx code.

**Both tracks.**

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Database | SQLite | PostgreSQL — required when the host has no persistent disk (§6) | — |
| ORM | SQLAlchemy 2.0 (`Mapped[]` declarative) | — | SQLModel |
| Migrations | Alembic | — | Hand-applied schema changes; `create_all()` on a deployed database |
| Deploy artifact | Container image from a `Dockerfile` | The host's native Python runtime building from `uv.lock` | Docker required for local development or tests |

Sharing everything below the UI is what makes a move from Track A to Track B a rewrite of
the presentation layer only.

**Agent scoping rule for this archetype.** Business logic lives under `services/` and
contains no UI calls. Presentation lives in templates and route handlers and contains no
business logic. When several agents work in parallel here, each is scoped to one side of
that line. This split is what makes the archetype survivable as it grows.

### 5.5 Web — TypeScript

One language, one platform, one deployment.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| UI library | React | — | — |
| Framework | Next.js (routing, rendering, build) | — | — |
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
| UI library | React | — | — |
| Frontend framework | Next.js | — | — |
| Backend | FastAPI | — | — |
| API boundary | Versioned prefix `/api/v1` | — | Unversioned public API |
| Client types | Generated from OpenAPI via `openapi-typescript` | — | Hand-maintained duplicate types |
| Frontend → backend wiring | Same-origin proxy (see below) | Direct cross-origin call with an explicit CORS allowlist | Hard-coded backend URLs in source |
| Auth tokens | HTTP-only cookies, same-origin | Bearer tokens in memory | Tokens in `localStorage` |
| Cold start | Documented and handled in the UI | — | Pretending it will not happen |

**How the two deployments connect.** They do not talk to each other. The frontend is
static files on a CDN; the JavaScript runs in the user's browser and calls the backend
over HTTP. Only two settings join them: the backend's URL, injected into the frontend at
build time as an environment variable, and the frontend's origin, listed in the backend's
CORS allowlist. A mismatch between those two is the most common failure in this archetype.

**Prefer a same-origin proxy.** Configure the frontend host to forward `/api/*` to the
backend instead of having the browser call the backend's domain directly. Then every
request is same-origin from the browser's point of view: no CORS configuration, and
cookie-based sessions work without `SameSite=None` gymnastics. A project MAY instead call
the backend cross-origin, but then the CORS allowlist MUST be environment-driven and the
cookie policy MUST be documented.

**Cold start is not solved by either approach.** A free-tier backend that sleeps after
idle takes tens of seconds to wake. The first request MUST surface that state in the UI
rather than appearing broken. If always-on response time is a requirement, move the
backend to an always-free VM instead of a sleeping free tier.

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
| Android build | **Local** — `npx expo run:android`, or `eas build --local` | EAS cloud build | A pipeline that only works through the cloud |
| iOS build | EAS cloud build | Local on macOS + Xcode | — |
| Device testing | Android emulator | Physical device over USB; Expo Go on the same LAN | Depending on Expo Go as the only test path |
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
| `web-python` | Render (free web service) + Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free; **Streamlit Community Cloud — Track A, personal non-commercial practice only** | Render free: 512 MB RAM, sleeps after 15 min idle, about 1 min to wake. Neon Free: 0.5 GB per project, sleeps after 5 min idle and wakes on the next query |
| `web-ts` | Cloudflare Workers | **Vercel — non-commercial projects only** | Cloudflare D1 provides free SQLite |
| `web-api` | Cloudflare Pages (frontend) + Render (backend) + Neon Free Postgres when the backend keeps durable data | Oracle Cloud Always Free (both) | The only archetype spanning two platforms on free tiers |
| Desktop | GitHub Releases | — | — |
| Mobile | EAS Build → internal link → Google Play | — | — |

**Vercel.** Its free Hobby plan is restricted to non-commercial use. Any project that
might be monetized MUST NOT depend on it. Cloudflare carries no such restriction, which is
why it is the default here despite Vercel's first-party Next.js support.

**Durable data on free hosting.** Render's free web service loses its filesystem on every
restart, redeploy, and spin-down, so a SQLite file or uploaded file kept there is lost.
Render's free Postgres expires 30 days after creation and has no backups. Any backend on
Render's free tier that keeps durable data MUST store it outside Render, in Neon Free
Postgres by default. The app reaches the database only through its connection URL, so
moving to another Postgres changes one environment variable.

Neon was chosen over Supabase Free. Supabase pauses a free project after 7 days of low
activity, and the database stays down until someone restores it by hand from the
dashboard; an internal tool left unused for a week would appear broken. Supabase's bundled
login and file storage would also place user accounts inside one vendor. Neon provides
Postgres only and wakes on its own.

Before a project first uses Neon, confirm in Neon's current terms that the free plan
permits commercial use, and confirm the data region if the app holds personal data. Neither
was verified when this selection was made.

**Streamlit Community Cloud.** Its terms prohibit business or commercial use and the
processing of sensitive personal data. It is listed only for personal learning projects
with public data. A work tool, including any public-sector tool, MUST NOT be hosted there.

**Idle policies.** Free tiers sleep, pause, or reclaim idle resources by design. A project
MUST NOT generate artificial traffic or CPU load to defeat that policy. It breaks silently
when the pinger stops, and it risks the account. If always-on behaviour is a requirement,
choose a host that provides it.

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

## 9. Installation Checklist

This section lists **what must exist** on a machine for each archetype. It does not explain
how to install anything — that belongs to `manuals/`. Treat it as the completeness check:
if a project needs something that is not listed here, this document is wrong and must be
amended before the project starts.

Scope note: this is the target stack. Everything the archetype will ever need through its
final deployed state is listed, including items only needed at release time.

### 9.1 Base — required for every archetype

| Requirement | Scope | Note |
|---|---|---|
| Git | System | — |
| GitHub account | Account | Free. Public repo required for GitHub Pages |
| GitHub CLI (`gh`) | System | Releases, PRs, workflow runs from the terminal |
| VS Code | System | — |
| CLI coding agent | System | — |
| uv | System | Required even for TypeScript-only projects |
| pre-commit | Tool (`uv tool install`) | — |

**Why uv is in the base list.** pre-commit is a Python application. Even a project with no
Python code needs a Python tool runner to install the hooks, and uv provides that without
a separate Python installation.

### 9.2 PC App — Python

| Requirement | Scope | Note |
|---|---|---|
| Python 3.12+ | Managed by uv | No separate installer needed |
| PySide6 | Project dependency | Bundles the Qt runtime **and** the Qt tools (`pyside6-designer`, `pyside6-uic`, `pyside6-rcc`). No separate Qt installation |
| C toolchain | **System — required** | Nuitka compiles to C. Windows: MSVC Build Tools. macOS: Xcode Command Line Tools. Linux: gcc or clang, plus ccache |
| nuitka, ordered-set, zstandard, patchelf | Project `build` group, locked | Installed by `uv sync --group build`; patchelf on Linux only. Never left to `pyside6-deploy` to install |
| Inno Setup | System — Windows only | Installer packaging |
| Qt runtime libraries | System — Linux only | `libxcb-cursor0`, `libxkbcommon-x11-0`, `libegl1`, `libgl1`; absent on minimal distros and server images. Qt 6.5+ will not start without `libxcb-cursor0` |
| `appimagetool` | System — Linux only | Tagged release, version and SHA-256 recorded in `DECISIONS.md`. Not needed if the project ships `.tar.gz` |

The C toolchain is the single most common cause of a first build failing on a fresh
machine. Nothing in `uv sync` reveals that it is missing.

### 9.3 PC App — Electron

| Requirement | Scope | Note |
|---|---|---|
| Node.js LTS | System | — |
| pnpm | System (via corepack) | — |
| Native module build toolchain | **System — required** | better-sqlite3 is rebuilt against Electron's ABI. Windows: MSVC Build Tools + Python. macOS: Xcode Command Line Tools. Linux: build-essential |
| Playwright browsers | Project (`pnpm exec playwright install`) | Downloaded, not bundled |
| Code signing certificate | Account — paid, optional | Unsigned Windows builds trigger SmartScreen warnings; unsigned macOS builds require a Gatekeeper override. Defer until distribution matters |

Cross-building installers for another OS is **not** supported practice here. Use a GitHub
Actions matrix with real runners per OS instead of Wine or equivalents.

### 9.4 Web — Static

| Requirement | Scope | Note |
|---|---|---|
| Node.js LTS, pnpm | System | — |
| wrangler | Project dependency | Cloudflare deploys and local preview |
| Cloudflare account | Account | Free |

### 9.5 Web — Python

| Requirement | Scope | Note |
|---|---|---|
| Python 3.12+ | Managed by uv | — |
| Tailwind standalone CLI | System — Track B only | A single binary, pinned to a tagged v4 release. Keeps Node out of this archetype entirely |
| Docker or Podman | System — optional | Local parity with the deployed container |
| htmx | Vendored file — Track B only | Version and checksum recorded in `DECISIONS.md` |
| Render account | Account | Free tier sleeps after idle |
| Neon account | Account — only when the app keeps durable data | Free, no card. Check terms and region first (§6) |
| Streamlit Community Cloud account | Account — Track A, personal practice only | Free, non-commercial (§6) |

No Node.js. If a project in this archetype reaches for npm, it has outgrown the archetype
and should move to `web-ts` or `web-api`.

### 9.6 Web — TypeScript

| Requirement | Scope | Note |
|---|---|---|
| Node.js LTS, pnpm | System | — |
| wrangler | Project dependency | Also runs D1 locally; no separate database install |
| Playwright browsers | Project | — |
| Cloudflare account | Account | Free |

### 9.7 Web — Split

Everything in §9.5 (Track B backend) plus everything in §9.6, minus wrangler's D1 usage.
Additionally:

| Requirement | Scope | Note |
|---|---|---|
| Render account | Account | Backend |
| Cloudflare account | Account | Frontend and `/api/*` proxy |
| Docker or Podman | System — optional | Backend parity |

### 9.8 Mobile

| Requirement | Scope | Note |
|---|---|---|
| Node.js LTS, pnpm | System | — |
| Expo CLI, EAS CLI | Run via `npx` | No global install |
| Expo account | Account | Free, ~30 builds/month |
| Android Studio + Android SDK + emulator image | **System — required** | Local builds and the emulator. The only build path that survives a restricted network |
| JDK 17 | **System — required** | Gradle. Android Studio bundles a JDK, but set `JAVA_HOME` explicitly rather than relying on it |
| Expo Go app | Phone — optional | Fast iteration convenience. Requires the phone and the PC on the same reachable network |
| Watchman | System — macOS only | — |
| Maestro CLI | System | Requires Java |
| Google Play developer account | Account — one-time fee | Only at store release |
| macOS + Xcode + Apple Developer membership | System + Account — deferred | iOS only. EAS Build compiles iOS in the cloud, so no Mac is required for builds — but App Store submission requires the paid membership |

**Why the local toolchain is required rather than optional.** Expo is often described as
cloud-first, but `npx expo run:android` and `eas build --local` run the entire Android
build on the developer's machine. That path is the default here because it is the only one
that works when outbound access to a build service is blocked, when a personal phone may
not join the network, or when client isolation prevents a phone from reaching the PC's
bundler. Gradle still fetches dependencies from Maven on a first build, but once the cache
is warm the loop is local. EAS cloud build remains available and is the right choice when
the network allows it — it just cannot be the foundation.

iOS is the exception and stays deferred: a local iOS build requires macOS and Xcode, so
for iOS the cloud build is the only option short of buying a Mac.

### 9.9 Modules

| Requirement | Scope | Note |
|---|---|---|
| Ollama | System — AI/RAG, local models | — |
| Model weights | Disk — AI/RAG | Multi-gigabyte downloads. Budget disk space and first-run time |
| CUDA toolkit / GPU drivers | System — AI/RAG, optional | CPU inference works; GPU is a speed choice |
| sentence-transformers, FAISS | Project dependencies | — |
| Model provider API key | Account — AI/RAG, hosted models only | Usage-based cost. Never committed |

### 9.10 OS build toolchain summary

The items below are the OS-level prerequisites that package managers cannot install for
you. Everything else in this document installs through uv, pnpm, or a project dependency.

| OS | Install | Needed for |
|---|---|---|
| Windows | Visual Studio Build Tools (C++ workload) | `pc-app-python` (Nuitka), `pc-app-electron` (native modules) |
| Windows | Inno Setup | `pc-app-python` installers |
| macOS | Xcode Command Line Tools | `pc-app-python`, `pc-app-electron` |
| macOS | Watchman | `mobile` |
| Linux | build-essential (gcc), ccache | `pc-app-python`, `pc-app-electron` |
| Linux | Qt runtime libraries | `pc-app-python` |
| Any | JDK 17 | `mobile` — Gradle builds and Maestro |
| Any | Android Studio + Android SDK | `mobile` — local builds and emulator |

### 9.11 Total cost at the target state

| Item | Cost | Triggered by |
|---|---|---|
| Everything in §9.1 through §9.10 not listed below | Free, commercial use permitted | — |
| Google Play developer account | One-time fee | Publishing an Android app to the store |
| Apple Developer membership | Yearly | Publishing an iOS app — deferred |
| Windows / macOS code signing certificate | Yearly, optional | Distributing desktop builds without OS warnings |
| Hosted model API usage | Per-use | AI/RAG module with hosted models instead of Ollama |
| Oracle Cloud Always Free | Free, card required for identity verification | Replacing a sleeping free-tier backend with an always-on one |

Nothing else in this stack requires payment at any stage, including the final deployed
state. Vercel's free tier is excluded from the stack as a default precisely because its
non-commercial restriction would break that guarantee.

### 9.12 Network dependence

Every archetype needs network access once, to install. What differs is what happens after
that. On a restricted or filtered network, the column that matters is the middle one.

| Archetype | First setup | Development loop | Release |
|---|---|---|---|
| `pc-app-python` | Network | **Fully offline** | Local build; network only to upload the artifact |
| `pc-app-electron` | Network | **Fully offline** | Local build; network only to upload the artifact |
| `web-static` | Network | **Fully offline** | Network — the platform is the product |
| `web-python` | Network | **Fully offline** | Network |
| `web-ts` | Network | Offline, except `wrangler` features that proxy to Cloudflare | Network |
| `web-api` | Network | **Fully offline** | Network |
| `mobile` | Network, including Gradle's first dependency fetch | **Offline once the Gradle cache is warm** | Android: local. iOS: network |
| AI/RAG module | Network for model download | **Fully offline with Ollama** | — |

Two rules follow from this table.

A project's test suite MUST run offline. Tests that reach the network are integration
tests, MUST be marked as such, and MUST be excluded from the default test run. CI is the
only place where network availability may be assumed.

Dependency caches are treated as assets, not scratch space. The uv cache, the pnpm store,
and the Gradle cache turn a one-time network requirement into permanent offline capability
and MUST NOT be cleared casually. Every package manager in this stack has an offline mode
(`uv sync --offline`, `pnpm install --offline`, `gradle --offline`), and a developer on a
restricted network should know they exist.

**The desktop archetypes are the safest choice under network restriction.** Nothing in
`pc-app-python` or `pc-app-electron` requires a network after setup, through to a signed,
shippable artifact. When a project could reasonably be either a desktop app or a web
service and the target environment is locked down, that is a real argument for desktop.

---

## 10. Changing This Document

A technology selection changes only when:

1. The reason is written as an ADR in the affected project's `DECISIONS.md`, and
2. If the change should apply to all future projects, this document is edited in the same
   commit.

Git history is the version record. This document carries no version number, and neither do
its filenames. A replaced selection is deleted here and survives in the ADR that replaced
it.
