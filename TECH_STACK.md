# TECH_STACK.md

**What this is.** The official technology selections for every project built in this
framework. `GLOBAL_RULES.md` decides *how* to build; this document decides *what to
build with*.

**Precedence.** This document is not loaded in project sessions; rules are written from
it, and what a project needs is copied into its rule files. A rule file MAY narrow a
choice made here. It MUST NOT widen one. Session precedence:
`GLOBAL_RULES.md` → `rules/<project>.md` → `CLAUDE.md` → current task.

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
| Agent Plugin | `rules/agent-plugin.md` | The deliverable is **installed into an AI coding tool**: skills, commands, hooks, a CLI, an MCP server, or several together |

| Module | File | Adds |
|---|---|---|
| AI / RAG | `rules/module-ai-rag.md` | LLM, embedding, vector search **at runtime** |
| Deployment | `rules/module-deployment.md` | Release automation, signing, distribution |

**Agent Plugin against the others.** Pick it when the thing being built is installed into
an AI coding tool and used through it — skills and commands, a CLI the user or the model
runs, an MCP server, or all of these from one repository. It has no window and no site of
its own. If the project needs its own window, it is a PC app; if it needs its own site or
API, it is a web archetype.

**Decision order for web projects.** Start at the top and stop at the first "yes":

1. Does anything have to happen on a server — including a form whose contents must be sent
   somewhere or stored? No → `web-static`.
2. Is the audience internal, or is it a dashboard / data tool / PoC? Yes → `web-python`.
3. Does the server genuinely need Python libraries (ML, scientific, pandas, a Python-only
   SDK)? No → `web-ts`. Yes → continue.
4. → `web-api`.

Choose `web-api` only after answering yes to step 3. It is the only archetype with two
toolchains — uv and pnpm — and two test stacks in one repository, and every task in it
pays for both.

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
| Node runtime | Node.js LTS, installed by pnpm, pinned in `package.json` `devEngines.runtime` | — | `.nvmrc` and separate version managers (nvm, fnm, Volta) as the version record |
| Node packaging | **pnpm**, standalone install, exact version in `package.json` `packageManager` (`pnpm-lock.yaml`) | — | npm, yarn; Corepack |
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
version pinning comes free and no PC needs a separate version manager installed.

**Why pnpm installs Node, not the other way round.** Since pnpm 11, pnpm ships as a
standalone executable and installs the Node.js version the project declares in
`devEngines.runtime`, the way uv installs Python. The older route — Corepack, bundled with
Node, fetching pnpm — ends with Node 25, which no longer ships Corepack. One tool installed
first, one file recording both versions.

---

## 4. Development Environment

### 4.1 Native first, Docker optional

Dependencies and language runtimes are reproduced by lockfiles, not by containers:

```
uv sync      # Python: exact deps + exact interpreter version
pnpm install # Node: exact deps + exact runtime version
```

System tools a lockfile cannot pin — compilers, SDKs, OS build tools — are listed per
archetype in §9 and pinned in CI. A project MUST start from a clean clone with those
commands alone, once §9's tools are installed. A project MUST NOT
require Docker to run its tests or its development server.

Docker Compose is RECOMMENDED for web archetypes whose deployment artifact is itself a
container. It is not used for `web-static`, desktop, or mobile archetypes, where GUI and
emulator toolchains make containers a net loss.

| Container runtime | Status |
|---|---|
| Docker Engine / Docker CE | Default |
| Podman Desktop, Rancher Desktop | Alternative — where Docker Desktop licensing or install policy blocks it |

Dev Container configuration is OPTIONAL. It MUST NOT become the only supported way to
develop a project.

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
| Shell | **Electron**, one of the three supported majors | — | — |
| Renderer UI | React + TypeScript | — | — |
| Build tooling | electron-vite | — | `vite-plugin-electron` (a different project with a similar name) |
| Styling | Tailwind CSS v4 + shadcn/ui | CSS Modules | — |
| Renderer state | None — React state | Zustand, for state shared by distant components | Redux, Redux Toolkit |
| Renderer data | Loaded through the preload bridge by the view that shows it | TanStack Query, for polling or optimistic updates | SWR |
| IPC | `contextBridge`, typed, explicitly enumerated channels | — | `nodeIntegration: true`, `contextIsolation: false`, exposing `ipcRenderer` wholesale |
| Persistence | SQLite via better-sqlite3 13+ in the main process; schema on `PRAGMA user_version` | — | Direct DB access from the renderer; `node:sqlite` while it is a release candidate |
| User credentials | Electron `safeStorage` | — | keytar |
| Packaging | electron-builder | Electron Forge | — |
| Auto-update | None | electron-updater, from GitHub Releases | — |
| Tests | Vitest — `node` for logic, browser mode in Chromium for renderer components; Playwright's Electron support for the whole app | — | jsdom, happy-dom; Spectron |

**Why Electron stays the default.** Tauri's smaller binaries and lower idle memory are
real, but it moves custom native behaviour into Rust and pushes WebView compatibility
testing onto the developer, while Electron keeps the whole application in one language
with a mature packaging and auto-update ecosystem. For one developer maintaining a project
for years, a second language costs more than a hundred megabytes on disk. Tauri shares
none of Electron's build, IPC, packaging or test tooling, so it is not an Alternative row
here: a project that needs it writes an ADR and its own rule.

**Why no state or data library by default.** Same reasoning as `web-ts`: the renderer
asks the main process over IPC, which is local and does not sleep, so loading and error
states stay small. Each library is added when a concrete need from the Alternative column
appears.

**Why better-sqlite3 13, not `node:sqlite`.** From 13.0 better-sqlite3 is an N-API module
and ships prebuilt binaries that load in both Node and Electron, so it is no longer
rebuilt against Electron's ABI on every upgrade, and the same binary runs the adapter
tests under plain Node. `node:sqlite` needs no native module at all, but in Node 24 it is
Stability 1.2, release candidate. Reconsider when it is stable.

**Why `safeStorage`.** It is Electron's own API: the key lives in the OS store (Keychain,
DPAPI, a Linux secret service) and the app keeps only ciphertext. keytar, the former
choice, is archived. On Linux without a secret service `safeStorage` falls back to a
hard-coded key; the rule treats that as no credential store.

### 5.3 Web — Static

No server process. Everything is built once and served from a CDN; what runs, runs in the
visitor's browser.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Language | HTML, CSS, TypeScript | — | Untyped JavaScript |
| Build | Vite, multi-page | Astro, static output — content-heavy sites with many pages | A component framework or client-side router; Astro server output or adapters |
| Styling | Tailwind CSS v4 | Plain CSS | — |
| Interactivity | Standard DOM APIs | Alpine.js, CSP build (`@alpinejs/csp`) | React for a page with three buttons; Alpine's standard build |
| Third-party runtime resources | None — everything ships in the build | — | CDN scripts, hosted web fonts, tracking tags |
| Tests | Vitest; DOM code runs in real Chromium through Vitest browser mode (Playwright provider) | — | jsdom, happy-dom |

**Promotion path.** A form whose contents must be sent somewhere or stored, a call that needs
a secret, or a login is server logic, and the project becomes `web-python` or `web-ts`. Do
not grow a backend inside a static site through edge functions or Worker scripts; that is
the signal to change archetype. A form that computes and shows a result in the page is not
server logic.

**Why a real browser for DOM tests.** A simulated DOM such as jsdom is faster and needs no
browser download, but which behaviours it gets wrong — layout, focus, applied CSS — cannot
be told from the code, so no rule can say when it is safe. Pure logic is tested in Node;
everything that touches the page runs in Chromium. One browser download, once.

**Why Alpine's CSP build.** The standard build evaluates expressions from strings, which a
Content-Security-Policy without `'unsafe-eval'` blocks. The CSP build works under that
policy.

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
| Tests | pytest + `streamlit.testing.v1.AppTest` | — | — |

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
| Tests | pytest + FastAPI `TestClient` (built on httpx) | — | — |

**Why htmx 2, not 4.** htmx 4.0 shipped in August 2026 with renamed events and changed
defaults. The htmx project keeps 2.x as its `latest` release until early 2027 and supports
it indefinitely, and coding agents have been trained on 2.x. Choosing 4 is an ADR, and the
agent must check which major version a project vendors before writing htmx code.

**Both tracks.**

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Database | SQLite for development, tests, and hosts with a persistent disk; PostgreSQL in production when the host has no persistent disk and the app keeps durable data (§6) | PostgreSQL on a host that has a persistent disk | — |
| PostgreSQL driver | psycopg 3 | — | — |
| ORM | SQLAlchemy 2.0 (`Mapped[]` declarative) | — | SQLModel |
| Migrations | Alembic | — | Hand-applied schema changes; `create_all()` on a deployed database |
| Deploy artifact | Container image from a `Dockerfile` | The host's native Python runtime building from `uv.lock` | Docker required for local development or tests |
| Background work | None — work finishes inside the request or rerun | FastAPI `BackgroundTasks` for best-effort follow-ups | A job-queue service (Celery, RQ) |

Sharing everything below the UI is what makes a move from Track A to Track B a rewrite of
the presentation layer only.

**Where the database runs.** The code reads one connection URL, so each stage uses the
database that suits it: SQLite on the developer's PC, offline and with no setup; PostgreSQL
in a CI service container, which catches the differences between the two engines before
deploy; Neon in production. Local development and the default tests therefore never need
the production database. A developer MAY run PostgreSQL locally — in a container, or as a
separate Neon project for development — but a project MUST NOT require it.

**Why no job queue.** A queue needs a broker and a separate worker process: two more
services to host and keep alive on free tiers that have room for one. Work that must
survive the request is out of scope for this archetype until an ADR names where it runs.

**Agent scoping rule for this archetype.** Business logic lives under `services/` and
contains no UI calls. Presentation lives in templates and route handlers and contains no
business logic. When several agents work in parallel here, each is scoped to one side of
that line. This split is what makes the archetype survivable as it grows.

### 5.5 Web — TypeScript

One language, one process, one deployment. Next.js runs as a Node.js server in a
container, the same way `web-python` runs.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| UI library | React | — | — |
| Framework | Next.js, App Router, `output: "standalone"` | — | Pages Router in new code; vinext or any other reimplementation of the Next.js API |
| Server logic | Server components, server actions, route handlers | — | A separate backend service — that is `web-api` |
| Styling | Tailwind CSS v4 + shadcn/ui | — | — |
| Client state | None — React state and URL search params | Zustand, for state shared across distant client components | — |
| Server state | None — data is read in server components | TanStack Query, for polling, infinite scroll, or optimistic updates | — |
| Database | PostgreSQL. PGlite (Postgres compiled to WASM, in-process) for development and tests; Neon Free in production | — | SQLite and Cloudflare D1 — Drizzle schemas are written per engine, so one codebase holds one engine |
| ORM | Drizzle, `drizzle-kit` migrations | — | Prisma |
| Validation | Zod 4 | — | Hand-written parsing of request input |
| Accounts | Better Auth, tables in the app's own database through its Drizzle adapter | — | Accounts held by a vendor's auth service (Clerk, Supabase Auth, Firebase Auth, Auth0); Auth.js / NextAuth for new projects |
| Deploy artifact | Container image from a `Dockerfile` | — | Docker required for local development or tests |
| Tests | Vitest — `node` for logic, browser mode in Chromium for client components; Playwright for whole pages | — | jsdom, happy-dom |

**Why a Node server, not Cloudflare Workers.** The Workers Free plan stops a request at
10 ms of CPU time and rejects a Worker larger than 3 MB compressed. Cloudflare's own
documentation puts server rendering and authentication at 10–20 ms per request, and a
Next.js bundle can cross 3 MB on an adapter upgrade alone. On the free plan an ordinary
Next.js page fails at random; making it reliable costs a monthly fee. Next.js
self-hosted as a Node server is supported by the Next.js project itself, so the same
image runs on any host that runs a container. It also puts `web-ts` on the same host,
database and backup procedure as `web-python`.

**Why not vinext.** Cloudflare now recommends vinext for Next.js on Workers. It is a
reimplementation of the Next.js API on Vite, not Next.js, released as 1.0 in September
2026. It does not remove the Workers Free limits, and coding agents trained on Next.js
cannot tell where its behaviour differs.

**Why PGlite for development.** Drizzle has no engine-neutral schema, so the database
must be PostgreSQL at every stage. PGlite is PostgreSQL in WASM, installed as an ordinary
dependency: no server, no Docker, offline. CI also runs the database tests against a real
PostgreSQL service container, which catches what PGlite does differently.

**Why no client-state or data-fetching library by default.** With the App Router, a page
reads its data on the server and sends finished HTML. A client cache or a global store
duplicates that path. Each is added when a concrete need from the Alternative column
appears.

**Why Better Auth.** Password hashing, session rotation and login rate limiting are the
code a one-person project can least afford to get wrong, and the least likely to be
caught by tests. Better Auth stores accounts in the app's own tables, so the user data
never leaves the project's database. The Auth.js team recommends it for new projects.
Vercel acquired it in July 2026; it remains MIT-licensed. Its tables are ordinary
Drizzle tables, so replacing it later is a data migration, not a vendor exit.

### 5.6 Web — Split

A Next.js frontend built as a static export, and a FastAPI backend that serves both that
export and the API, from one container. The backend is §5.4 Track B without templates,
htmx, or the Tailwind standalone CLI. The frontend uses the toolchain of §5.5 without a
Node server.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| UI library | React | — | — |
| Frontend framework | Next.js, App Router, `output: "export"` | — | A Next.js server (`output: "standalone"`) beside the backend; Pages Router in new code |
| Styling | Tailwind CSS v4 + shadcn/ui | — | — |
| Backend | FastAPI, serving `/api/v1` and the exported files | — | — |
| API boundary | Versioned prefix `/api/v1` | — | Unversioned public API |
| Client types | Generated from OpenAPI via `openapi-typescript` | — | Hand-maintained duplicate types |
| API client | `openapi-fetch`, typed by the generated file | — | axios; a hand-written `fetch` wrapper per endpoint |
| Server state | TanStack Query v5 | — | SWR; `useEffect` + `fetch` to load a page's data |
| Client state | None — React state and URL search params | Zustand, for state shared across distant client components | Redux, Redux Toolkit |
| Frontend → backend wiring | Same origin by construction: one process serves both | — | A separate frontend host; a proxy in front of the backend; CORS in any deployed environment; backend URLs in source |
| Accounts | As §5.4 Track B: users table, server-side sessions, HTTP-only cookie | — | Tokens in `localStorage`; bearer tokens handed to the browser; JWT browser sessions |
| Deploy artifact | One container image: a Node stage builds the export, a Python stage serves it | — | Two images; Docker required for local development or tests |
| Tests | Backend as §5.4 Track B; frontend as §5.5, with Playwright against the backend serving the export | — | jsdom, happy-dom |

**Why one container.** Two hosts need a proxy to stay same-origin, and Cloudflare's
`_redirects` cannot proxy to an external domain, so that proxy becomes Worker code. Two
hosts also mean two deploy pipelines and a frontend that can run against an API version it
was not built with. Served by the backend, the export is same-origin by construction: no
CORS, no proxy, one image, one tag, one version. The cost is that the pages sleep with the
backend, so the first visit after idle waits about a minute, as in `web-python` and
`web-ts`.

**What a static export gives up.** Dynamic route segments without `generateStaticParams()`,
rewrites, redirects, `headers()`, cookies, server actions, and route handlers that read the
request. Record IDs travel in query parameters (`/items?id=3`), not in path segments.
Redirects and response headers are set by the backend.

**Why TanStack Query is the default here and not in `web-ts`.** In `web-ts` a page reads
its data on the server. Here every read happens in the browser, against an API that may be
waking up, so loading, error, retry and cache invalidation are needed on every page. That
is the library's whole job.

**Cold start.** `README.md` says the first visit after idle is slow. If always-on response
time is a requirement, move to an always-free VM rather than keeping the service awake.

### 5.7 Mobile

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Framework | React Native via **Expo**, Continuous Native Generation: `android/` and `ios/` are generated, not committed | — | Bare React Native without a stated reason; committed, hand-edited native directories |
| Navigation | Expo Router | React Navigation | — |
| State | None — `useState`, `useReducer` | Zustand, for state shared by distant screens | Redux, Redux Toolkit |
| Local storage | expo-sqlite | AsyncStorage for key-value only | — |
| Secure storage | expo-secure-store | — | Tokens in AsyncStorage |
| Unit tests | Jest (`jest-expo`) | — | — |
| E2E | Maestro | — | — |
| pnpm linker | `isolated` (pnpm's default) | `hoisted`, when a named library fails under `isolated` | `enableGlobalVirtualStore`: Metro cannot resolve packages stored outside the project |
| CLI | Expo CLI from the `expo` package, run as `pnpm expo` | EAS CLI, only with an EAS alternative below: `pnpm dlx eas-cli@<exact version>` | `npx`; `eas-cli` as a project dependency (Expo advises against it); a global install |
| Development build | Local — `pnpm expo run:android`, with `expo-dev-client` | EAS cloud build | Expo Go as the only test path |
| Android release build | CI — GitHub Actions runs `expo prebuild` and Gradle, signed with the upload key held in CI secrets | EAS cloud build — personal non-commercial projects only | A release built on a developer machine; `eas build --local` on Windows |
| iOS build | Deferred | EAS cloud build; local on macOS + Xcode | — |
| Device testing | Android emulator | Physical device over USB; Expo Go on the same LAN | — |
| OTA updates | None | EAS Update — personal non-commercial projects only | Shipping native changes as OTA |

**Why EAS is not the default.** EAS is Expo's hosted build, update and submit service.
Its free plan is limited to pre-revenue startups, proofs of concept, and hobby and student
projects, and a commercial project earning more than US$1,500 a month must move to a paid
plan. Under §1 that counts as not free, the same as Vercel Hobby. The free plan also caps
builds at 15 Android and 15 iOS a month and updates at 1,000 monthly active users. The Expo
SDK itself is MIT-licensed and needs no account; only EAS does. GitHub Actions produces the
same Android artifact with no account beyond GitHub.

**Why not `eas build --local`.** It still requires an Expo login, and Expo supports it only
on macOS and Linux. On Windows, `pnpm expo run:android` is the local build.

**Distribution path**, in order:

1. **Internal** — the signed release APK that CI attaches to a GitHub Release, for the
   developer and people close to them. No store account.
2. **Google Play** — one-time developer registration fee, AAB built by CI. A personal
   account created after 13 November 2023 must run a closed test with at least 12 testers
   opted in for 14 continuous days before it can apply for production access. An
   organization account is exempt but needs a D-U-N-S number. Play App Signing uses the
   key Google generates — the default most new apps take — so a Play build cannot update
   a GitHub APK. Testers move to Play's internal testing track; anyone on a GitHub APK
   exports their data and reinstalls from Play once. Handing Play the project's own key
   instead is for apps that keep shipping outside Play as well.
3. **iOS** — deferred. Requires a yearly Apple Developer membership and either a Mac or EAS
   cloud build, so choosing it reopens the EAS question above.

**Android developer verification.** Google is extending to every country in 2027 a
requirement that apps installed on certified Android devices, sideloaded APKs included,
come from a verified developer who has registered the app's package name and signing key.
A free limited-distribution account covers up to 20 devices without government ID.
Installing over ADB during development is not affected. The package name and the signing
key are therefore permanent: an app distributed outside Play cannot ship an update once
its key is lost.

Target floor: set by the Expo SDK (Android 7+, iOS 16.4+ when this was written). A project
never lowers it.

### 5.8 Agent Plugin

The deliverable is **installed into an AI coding tool** rather than run on its own. It has up
to three components, and a project takes only the ones it needs.

| Component | When |
|---|---|
| **A. Instruction bundle** | Always. Skills, agents, commands, hooks — Markdown the host reads |
| **B. Executable** | Only when instructions cannot do the work: a CLI, an MCP server, or both from one codebase |
| **C. Distribution** | Always. How a user installs A, B, or both |

**The dividing line.** A skill tells the host's model what to do with what it already has.
An executable is needed when the capability itself does not exist in the host: parsing a
binary file format, OCR, a network API with authentication, anything requiring a library.
Most requests that sound like "add a small MCP server" are one skill.

**The preferred shape when both exist.** The program does the work and is published to a
package registry. The skill stays thin and invokes it (`npx -y <pkg>`, `uvx <pkg>`). The
plugin is a subdirectory of the program's repository with its own manifest. This keeps one
codebase, one release, and a bundle that still works when the program is absent — it can
say so instead of failing.

**`npx` and `uvx` in the bundle.** The launch command in the bundle runs on the user's
machine, where pnpm cannot be assumed, so the bundle's MCP configuration, its skill text,
and the README's install section use `npx -y` or `uvx`. That is not a breach of the pnpm
rule: it governs the project's own scripts, CI, and developer documentation.

#### A. Instruction bundle

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Content format | Markdown with YAML front matter. Every skill and agent file carries at least a `description` — a missing one produces load errors | — | Instructions embedded in code strings |
| Layout | Components at the plugin root: `skills/<name>/SKILL.md`, `agents/`, `hooks/hooks.json`, `.mcp.json`. Only the manifest sits in `.claude-plugin/plugin.json` | — | **Components inside `.claude-plugin/`** — the host reads that directory as manifest only. New files in `commands/` — the host documents commands as the older format that skills supersede. A `CLAUDE.md` at the plugin root — the host does not load it; instructions are skills. A top-level `bin/` — host-specific, and claude.ai and Cowork refuse to install a plugin that has one; executables ship through npm or PyPI |
| Host manifests | The Claude Code manifest is canonical; manifests for other hosts are generated from it | One manifest when only one host is supported | Hand-maintained copies that drift |
| Hooks | Hooks check, block, and log | — | A hook that auto-approves a tool call |
| Bootstrap scripts | **None.** Markdown only | For a script that must run *before* any runtime exists: PowerShell on Windows and POSIX `sh` elsewhere, as a matched pair, with the reason recorded | A script where a skill is enough |
| Validation | `claude plugin validate <path> --strict`, which checks the manifests, front matter, `hooks/hooks.json` and `.mcp.json`; plus a check that every internal link resolves, which validate does not do | — | Releasing without validation |
| Behaviour tests | **Scenario records**: for each skill, a fixed request and the expected behaviour, in Markdown. Written before the skill is added or its behaviour changed, and run then; re-run on each supported host before a release. Not run per commit | — | "It worked in my session" with no record; a host-specific eval tool as the only record |

**Folder name to check.** At least one host treats a root `rules/` directory as its own
rule location. A plugin storing rule files there MUST confirm no supported host claims the
name.

**Hosts differ** in how they install plugins, how they prompt for approval, and how closely
their model follows instructions. A plugin names the hosts it supports and keeps a scenario
record per host.

Scenario records are integration tests: they need a live host and are not deterministic, so
they are excluded from the default test run.

#### B. Executable — CLI and MCP server

One codebase. The CLI and the MCP server are two front doors onto the same core functions,
which know about neither.

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Language | TypeScript on Node.js LTS — `npx` makes it runnable with nothing installed | Python, when the work needs Python libraries; run with `uvx` | A language the target users cannot run without a toolchain |
| MCP SDK (TypeScript) | `@modelcontextprotocol/server` (SDK v2) | `@modelcontextprotocol/sdk` (v1) for an existing server; it receives fixes for at least six months past the v2 release | Framework wrappers adopted before the official SDK proves insufficient |
| MCP SDK (Python) | `mcp` 2.x; the server class is `MCPServer` (FastMCP was its v1 name) | — | `mcp<2` in a new server; the separate `fastmcp` package |
| Transport | **stdio** — local, subprocess-isolated, what an AI coding tool launches | Streamable HTTP, only for a server that must be remote | Legacy HTTP+SSE |
| Tool surface | Few tools, each doing one thing, with descriptions written for a model to choose between | — | Exposing every internal function as a tool |
| Secrets | **None.** An executable that needs a user's credential for another service is decided by ADR before it is built | — | Credentials in the server's config file or its repository |
| Filesystem reach | Only paths the caller passes in | — | Reading or writing outside them without being asked |
| Offline operation | An environment switch that disables all outbound traffic | — | Silent network calls from a tool the user believes is local |
| Testing | pytest or Vitest against the core functions, which are plain code | `pnpm dlx @modelcontextprotocol/inspector@<exact version>` for manual tool checks | Testing only through the host's model |
| Optional heavy dependencies | Declared optional, with the feature degrading and saying so when absent | — | A multi-hundred-megabyte install that every user pays for |

**Why stdio is the default.** An AI coding tool launches the server as a subprocess on the
same machine. There is no port, no origin, and no authorization layer to get wrong, and the
process dies with the client.

**Why the core is separate from both front doors.** The same functions serve the CLI, the
MCP tools, and the tests. Only the adapters know about argument parsing or JSON-RPC — §8
applies here as it does anywhere else.

#### C. Distribution

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Bundle | GitHub repository with `.claude-plugin/marketplace.json` at its root | Listing in a third-party marketplace, in addition | Distributing files by hand |
| Executable | npm for TypeScript, PyPI for Python, runnable with `npx` / `uvx` and no prior install. Published by CI with trusted publishing (OIDC) and no stored token | — | Install instructions that require cloning and building; a long-lived publish token |
| MCP discovery | An entry in the official MCP registry, published by CI with `mcp-publisher` logged in through GitHub OIDC | — | — |
| Client registration | A `setup` command that writes the MCP configuration for whichever clients are installed | Documented manual configuration per client | Editing a client's config without saying what changed |
| Versioning | Semantic Versioning, Git tags, from Conventional Commits. The bundle pins the executable to the exact version released with it, so a published package reaches users only when the marketplace pin moves. The marketplace entry pins the bundle to the release tag and its commit SHA, so users never install from `main` | — | A skill pinned to `latest` or to a version range; a marketplace entry that follows a branch |
| Documentation | README stating the prerequisite, the supported hosts, each host's install command, and the manual fallback | — | Install steps only in a wiki |
| Secrets | None anywhere in the bundle | — | Tokens in plugin files |

**The first npm version.** npm accepts a trusted publisher only for a package that already
exists; PyPI accepts one in advance. The first npm version is therefore published by CI with
a short-lived granular token, which is revoked once trusted publishing is configured.

**Why the marketplace pins a commit.** Every finished task is merged into `main`, so `main`
holds work that has not been released. Pinning the entry to the release tag and its SHA means
a user receives only what was approved for release. A release is the tag and the pin update
together; if the tag's CI fails, the pin is not moved and users keep the previous version.

---

## 6. Hosting

All selections below are free tiers that permit commercial use, except where noted.

| Archetype | Default | Alternative | Notes |
|---|---|---|---|
| `web-static` | Cloudflare Workers, static assets only (no Worker script) | Netlify Free; GitHub Pages — public repository, non-commercial sites only | Workers Free: static asset requests free and unlimited; 20,000 files per deployment, 25 MiB per file. Netlify Free: 300 credits/month shared by deploys, bandwidth and requests; every site pauses until the next cycle when they run out. GitHub Pages: terms exclude online businesses, e-commerce and commercial SaaS |
| `web-python` | Render (free web service) + Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free; **Streamlit Community Cloud — Track A, personal non-commercial practice only** | Render free: 512 MB RAM, sleeps after 15 min idle, about 1 min to wake. Neon Free: 1 GB per project, sleeps after 5 min idle and wakes on the next query |
| `web-ts` | Render (free web service) + Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free; **Vercel Hobby — personal non-commercial projects only** | Same limits as `web-python`. Cloudflare Workers Free is not listed: see §5.5 |
| `web-api` | Render (free web service — one container serving the frontend and the API) + Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free | Same limits as `web-python`. One service: one cold start, one draw on the workspace's 750 free instance hours a month |
| Desktop | GitHub Releases | — | — |
| Mobile | GitHub Actions build → GitHub Releases (internal) → Google Play | EAS Build, EAS Update — personal non-commercial projects only | EAS Free: 15 Android builds a month, updates to 1,000 monthly active users; commercial projects excluded (§5.7) |
| `agent-plugin` | GitHub repository as the marketplace source; npm or PyPI for an executable; the official MCP registry for a server | — | No hosting service and no account cost. Publishing to npm, PyPI, and the MCP registry is free |

**Cloudflare Workers, not Pages.** Cloudflare directs new projects to Workers static
assets and adds new features only there; Pages keeps working but is no longer the target.
An assets-only Worker has no script, so it serves files the way Pages does, with
`_headers` and `_redirects` supported.

**Vercel.** Its free Hobby plan is restricted to non-commercial use. Any project that
might be monetized MUST NOT depend on it. Cloudflare carries no such restriction, which is
why it is not the default here despite Vercel's first-party Next.js support.

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

Neon Free's instant restore reaches back 6 hours, capped at 1 GB of change history. It
undoes a mistake noticed the same morning; it is not a backup. An app holding data its
users cannot recreate keeps its own dumps off Neon.

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
| Mobile | GitHub Actions build → GitHub Releases, then Google Play |
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
| pnpm | System — standalone | Installed first. Exact version in `packageManager` |
| Node.js LTS | Installed by pnpm | Version in `devEngines.runtime`. No separate Node installer |
| C toolchain | System — only when better-sqlite3 has no prebuilt binary for the platform | better-sqlite3 13+ ships N-API prebuilds. A source build needs Windows: MSVC Build Tools + Python. macOS: Xcode Command Line Tools. Linux: build-essential |
| Chromium for renderer tests | Project (`pnpm exec playwright install chromium`) | Downloaded once. Whole-app tests drive the Electron binary itself and need no browser |
| Code signing certificate | Account — paid, optional | Unsigned Windows builds trigger SmartScreen warnings; unsigned macOS builds require a Gatekeeper override. Defer until distribution matters |

Cross-building installers for another OS is **not** supported practice here. Use a GitHub
Actions matrix with real runners per OS instead of Wine or equivalents.

### 9.4 Web — Static

| Requirement | Scope | Note |
|---|---|---|
| pnpm | System — standalone | Installed first. Exact version in `packageManager` |
| Node.js LTS | Installed by pnpm | Version in `devEngines.runtime`. No separate Node installer |
| wrangler | Project dependency | Cloudflare deploys and local preview |
| Chromium for tests | Project (`pnpm exec playwright install chromium`) | Downloaded once; tests then run offline |
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
| OIDC client registration | Account — Track A with login | Google, Microsoft or another provider; client ID and secret kept out of Git |
| PostgreSQL client tools (`pg_dump`, `psql`) | System — PostgreSQL projects only | Major version at or above the server's, or `pg_dump` refuses to run |

No Node.js. If a project in this archetype reaches for npm, it has outgrown the archetype
and should move to `web-ts` or `web-api`.

### 9.6 Web — TypeScript

| Requirement | Scope | Note |
|---|---|---|
| pnpm | System — standalone | Installed first. Exact version in `packageManager` |
| Node.js LTS | Installed by pnpm | Version in `devEngines.runtime`. No separate Node installer |
| PGlite | Project dependency | The development and test database. No PostgreSQL install |
| Chromium for tests | Project (`pnpm exec playwright install chromium`) | Downloaded once; tests then run offline |
| Docker or Podman | System — optional | Local parity with the deployed container |
| Render account | Account | Free tier sleeps after idle |
| Neon account | Account — only when the app keeps durable data | Free, no card. Check terms and region first (§6) |
| PostgreSQL client tools (`pg_dump`, `psql`) | System — only when the app keeps durable data | Major version at or above the server's |

### 9.7 Web — Split

Everything in §9.5 except the Track A, Tailwind standalone CLI and htmx rows, plus the
pnpm, Node.js and Chromium rows of §9.6. One Render account hosts the whole app; no other
host account is needed.

### 9.8 Mobile

| Requirement | Scope | Note |
|---|---|---|
| pnpm | System — standalone | Installed first. Exact version in `packageManager` |
| Node.js LTS | Installed by pnpm | Version in `devEngines.runtime`, on a line the Expo SDK supports. No separate Node installer |
| Expo CLI | Project dependency (`expo`) | Run as `pnpm expo`. No global install |
| Android Studio + Android SDK + emulator image | **System — required** | Local builds and the emulator. SDK Platform 36 for current Expo SDKs. The only build path that survives a restricted network |
| JDK 17 | **System — required** | Gradle. Android Studio bundles a JDK, but set `JAVA_HOME` explicitly rather than relying on it |
| Maestro CLI | System | Requires Java 17+. Native Windows install supported |
| Upload keystore | Project-owned file, outside the repository | Created once, backed up off the machine. CI receives it and its passwords as secrets |
| Expo Go app | Phone — optional | Fast iteration convenience. Requires signing in to an Expo account, and the phone and the PC on the same reachable network |
| Expo account, EAS CLI | Account — only with an EAS alternative (§5.7) | Free plan excludes commercial projects |
| Google Play developer account | Account — one-time fee | Only at store release. New personal accounts face the 12-tester rule (§5.7) |
| Android developer verification | Account — distributing outside Play, from 2027 | Free limited-distribution account up to 20 devices |
| macOS + Xcode + Apple Developer membership | System + Account — deferred | iOS only. App Store submission requires the paid membership |

**Why the local toolchain is required rather than optional.** `pnpm expo run:android` runs
the whole Android build on the developer's machine, with no Expo account. It is the only
development path that works when outbound access to a build service is blocked, when a
personal phone may not join the network, or when client isolation prevents a phone from
reaching the PC's bundler. Gradle still fetches dependencies from Maven on a first build,
but once the cache is warm the loop is local. Releases are built by CI
(`GLOBAL_RULES.md` §12), never on this machine.

iOS is the exception and stays deferred: a local iOS build requires macOS and Xcode, and
the cloud route is EAS (§5.7).

### 9.9 Agent Plugin

| Requirement | Scope | Note |
|---|---|---|
| Base items in §9.1 | As listed | — |
| Each supported host tool | System, one per host | Needed to test the bundle, and each carries its own account or subscription for its model |
| pnpm | System — standalone; required for a TypeScript executable | As in §9.6. Also needed by the MCP Inspector regardless of the server's language |
| Node.js LTS | Installed by pnpm | Version in `devEngines.runtime`. No separate Node installer |
| npm account | Account — publishing a TypeScript executable | Free |
| PyPI account | Account — publishing a Python executable | Free |
| MCP Inspector | Run via `pnpm dlx` at an exact version, not installed | Manual tool checks against a local server |
| `mcp-publisher` | CI only — publishing an MCP server | A tagged release, version and SHA-256 recorded in `DECISIONS.md` |
| PowerShell / POSIX `sh` | Included with the OS | Only for the pre-runtime bootstrap case |
| OS build toolchain | **None**, unless a dependency ships native code | Native modules fall back to §9.3's requirements |

Lightest archetype to set up, heaviest to test: verifying behaviour means running a real
session on every host the plugin claims to support.

### 9.10 Modules

| Requirement | Scope | Note |
|---|---|---|
| Ollama | System — AI/RAG, local models | — |
| Model weights | Disk — AI/RAG | Multi-gigabyte downloads. Budget disk space and first-run time |
| CUDA toolkit / GPU drivers | System — AI/RAG, optional | CPU inference works; GPU is a speed choice |
| sentence-transformers, FAISS | Project dependencies | — |
| Model provider API key | Account — AI/RAG, hosted models only | Usage-based cost. Never committed |

### 9.11 OS build toolchain summary

The items below are the OS-level prerequisites that package managers cannot install for
you. Everything else in this document installs through uv, pnpm, or a project dependency.

| OS | Install | Needed for |
|---|---|---|
| Windows | Visual Studio Build Tools (C++ workload) | `pc-app-python` (Nuitka); `pc-app-electron` only without a prebuilt native binary |
| Windows | Inno Setup | `pc-app-python` installers |
| macOS | Xcode Command Line Tools | `pc-app-python`; `pc-app-electron` only without a prebuilt native binary |
| Linux | build-essential (gcc), ccache | `pc-app-python`; `pc-app-electron` only without a prebuilt native binary |
| Linux | Qt runtime libraries | `pc-app-python` |
| Any | JDK 17 | `mobile` — Gradle builds and Maestro |
| Any | Android Studio + Android SDK | `mobile` — local builds and emulator |

### 9.12 Total cost at the target state

| Item | Cost | Triggered by |
|---|---|---|
| Everything in §9.1 through §9.11 not listed below | Free, commercial use permitted | — |
| Google Play developer account | One-time fee | Publishing an Android app to the store |
| Apple Developer membership | Yearly | Publishing an iOS app — deferred |
| Windows / macOS code signing certificate | Yearly, optional | Distributing desktop builds without OS warnings |
| Hosted model API usage | Per-use | AI/RAG module with hosted models instead of Ollama |
| Oracle Cloud Always Free | Free, card required for identity verification | Replacing a sleeping free-tier backend with an always-on one |

Nothing else in this stack requires payment at any stage, including the final deployed
state. Vercel's free tier is excluded from the stack as a default precisely because its
non-commercial restriction would break that guarantee.

### 9.13 Network dependence

Every archetype needs network access once, to install. What differs is what happens after
that. On a restricted or filtered network, the column that matters is the middle one.

| Archetype | First setup | Development loop | Release |
|---|---|---|---|
| `pc-app-python` | Network | **Fully offline** | CI build (`GLOBAL_RULES.md` §12) |
| `pc-app-electron` | Network | **Fully offline** | CI build (`GLOBAL_RULES.md` §12) |
| `web-static` | Network | **Fully offline** | Network — the platform is the product |
| `web-python` | Network | **Fully offline** | Network |
| `web-ts` | Network | **Fully offline** | Network |
| `web-api` | Network | **Fully offline** | Network |
| `mobile` | Network, including Gradle's first dependency fetch | **Offline once the Gradle cache is warm** | Android: CI. iOS: network, deferred |
| `agent-plugin` | Network | **Fully offline** for the executable's own tests and the Inspector; **network to test the bundle**, since the host's model is a cloud service | Network |
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
