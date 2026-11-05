# rules/web-api.md

Web app with a designed UI and a Python server. A Next.js frontend is built once into
static files; one FastAPI process serves those files and the JSON API from one container,
and keeps any durable data in PostgreSQL.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if nothing forces Python on the server (`web-ts`), if there is no server
logic at all (`web-static`), if it is an internal tool, dashboard or PoC with no need for a
designed UI (`web-python`), or if one person uses it on one machine with local data
(`pc-app-python`).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Python | 3.12+, `.python-version`, uv-managed | — | 3.11 and below, system Python |
| Python packaging | uv (`pyproject.toml` with `[build-system]`, `uv.lock`) at the repository root | — | Poetry, pipenv, conda, bare pip; a hand-maintained `requirements.txt` |
| Node.js | An LTS line, installed by pnpm, pinned in `frontend/package.json` `devEngines.runtime` | — | Odd-numbered (non-LTS) majors; `.nvmrc`, nvm, fnm, Volta as the version record |
| Node packaging | pnpm, standalone install, exact version in `packageManager`; `frontend/pnpm-lock.yaml` | — | npm, yarn, bun; Corepack; `npx` in scripts or docs |
| Backend | FastAPI, serving `/api/v1`, `/healthz` and the exported frontend | — | Flask, Django, Litestar; Starlette used directly; a second server process of any kind |
| API schemas | Pydantic v2, separate from ORM models | — | ORM models returned from routes |
| Frontend | Next.js 16, App Router, `output: "export"`, TypeScript `strict: true` | — | `output: "standalone"` or any Next.js server; server actions; route handlers; `proxy.ts`; Pages Router; plain JavaScript in new code |
| Styling | Tailwind CSS v4 via `@tailwindcss/postcss`; shadcn/ui, Radix base, copied into `frontend/src/components/ui/` | Plain CSS | `tailwind.config.js`, `@tailwind` directives; CSS-in-JS; full component kits installed as packages (MUI, Chakra, Mantine, Ant Design) |
| API contract | `openapi.json` exported from the backend; `openapi-typescript` generates `frontend/src/api/schema.d.ts`; both committed | — | Hand-written request or response types; editing either generated file |
| API client | `openapi-fetch`, typed by `schema.d.ts` | — | axios; a hand-written `fetch` wrapper per endpoint |
| Server state | TanStack Query v5 | — | SWR; `useEffect` + `fetch` to load a page's data |
| Client state | None — `useState`, `useReducer`, URL search params | Zustand, for state shared by distant client components | Redux, Redux Toolkit |
| Persistence | SQLAlchemy 2.0 (`Mapped[]`), sync `Session`. SQLite for development and tests; PostgreSQL via psycopg 3 in production when the app keeps durable data (§4.3) | SQLite on the disk of an Oracle Cloud Always Free VM | async SQLAlchemy; SQLModel; any other ORM; string-built SQL |
| Migrations | Alembic, `render_as_batch=True` on SQLite | — | `create_all()` against a deployed DB; editing a shipped revision; hand-applied schema changes |
| Config | Frozen dataclass built from `os.environ` in `config.py`; `.env` loaded in development by `uv run --env-file .env` | pydantic-settings | `python-dotenv` calls in application code; `os.environ` read anywhere but `config.py`; `process.env` or `NEXT_PUBLIC_*` in frontend code |
| Accounts | Users table in the app's own database; server-side session table, opaque session-ID cookie; passwords hashed with Argon2 via pwdlib | — | A vendor's auth service (Supabase Auth, Firebase Auth, Clerk, Auth0); passlib; JWT for browser sessions; tokens in `localStorage` or in a cookie scripts can read |
| User file uploads | None | Storage chosen by ADR | The container's filesystem on a host without a persistent disk |
| Background work | None — work finishes inside the request | FastAPI `BackgroundTasks` for best-effort, non-durable follow-ups; a durable job mechanism, chosen by ADR, for work whose loss is unacceptable | Threads or tasks started from a handler that must outlive it |
| Server | Uvicorn, one process | — | Multiple workers or processes on SQLite |
| Lint / format | Ruff; Biome | ESLint + Prettier, for the frontend | Black + isort + flake8; `next lint` (removed in Next.js 16) |
| Type check | mypy strict on `domain/`, `services/`; `tsc --noEmit` before `next build` | pyright | `typescript.ignoreBuildErrors` |
| Tests | pytest + pytest-cov + FastAPI `TestClient`; Vitest — `node` for plain TypeScript, browser mode (`@vitest/browser-playwright`, `vitest-browser-react`) for components; Playwright for pages | — | unittest-only; jsdom, happy-dom; Jest; Cypress |
| Third-party runtime resources | None: scripts, styles and fonts ship in the export; fonts through `next/font` | — | `<script>` or `<link>` to a CDN; tracking or analytics tags |
| Deploy artifact | One `Dockerfile`: a Node stage builds the export, a Python stage serves it | — | Two images; Node in the runtime image; Docker required for local development or tests; a release image built on a developer machine |
| Hosting | Render free web service, plus Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free VM | A separate frontend host (Cloudflare, Vercel, Netlify, a Render static site); Render's disk or Render free Postgres as the store of record; artificial traffic or CPU load to defeat a host's idle policy |

---

## 2. Prerequisites

`uv sync` and `pnpm install` neither install nor reveal these. Procedure: skill
`web-api-release`.

| Item | Scope |
|---|---|
| pnpm | System, standalone, installed first. The exact version in `packageManager` |
| Node.js | Installed by pnpm from `devEngines.runtime`. No separate Node installer |
| Chromium for tests | Project, downloaded once by `pnpm exec playwright install chromium`; tests then run offline |
| Docker or Podman | System. Optional on the developer's machine; required in CI (§5.5) |
| Host account | Render or Oracle Cloud — whichever `DECISIONS.md` names |
| Neon account | Account, only when the app keeps durable data. Before first use, confirm in Neon's current terms that the free plan permits commercial use, and confirm the data region if the app holds personal data; record both in `DECISIONS.md` |
| PostgreSQL client tools (`pg_dump`, `psql`) | System, PostgreSQL projects only. Major version at or above the server's, or `pg_dump` refuses to run |

---

## 3. Layout

```
<repo>/
├── pyproject.toml          backend deps, tool config, the app's version
├── uv.lock
├── .python-version
├── .env.example
├── Dockerfile              Node build stage → Python runtime stage
├── alembic.ini             if the app persists data
├── migrations/             Alembic env.py + versions/
├── openapi.json            generated from the backend, committed
├── src/<app>/
│   ├── main.py             create_app(settings) — entry point + composition root
│   ├── config.py           Settings dataclass; the only reader of os.environ
│   ├── domain/             pure Python
│   ├── services/           use cases + ports.py (typing.Protocol)
│   ├── adapters/           SQLAlchemy models + repositories, clock
│   │   └── migrate.py
│   └── api/                routers, Pydantic schemas, session and request guards, static serving
├── tests/                  conftest.py, test_boundaries.py, domain/, services/, adapters/,
│                           api/, smoke/
└── frontend/
    ├── package.json        packageManager, devEngines.runtime, scripts; no version
    ├── pnpm-lock.yaml
    ├── pnpm-workspace.yaml pnpm settings: allowBuilds
    ├── tsconfig.json       strict
    ├── next.config.ts      output: "export", trailingSlash, images.unoptimized
    ├── postcss.config.mjs
    ├── biome.json
    ├── components.json     shadcn/ui
    ├── vitest.config.ts    projects: plain TypeScript in Node, components in Chromium
    ├── playwright.config.ts
    ├── public/             copied into the export as-is, public
    ├── src/
    │   ├── app/            pages, layouts, not-found.tsx, globals.css
    │   ├── components/     app components; ui/ = shadcn/ui source
    │   └── api/            schema.d.ts (generated), client.ts, one hooks file per resource
    ├── tests/              unit/, browser/, e2e/, boundaries.test.ts
    └── out/                gitignored
```

- `main.py` is the only place that reads `Settings`, creates the `Engine`, and constructs
  adapters and services. Serve with `uvicorn <app>.main:create_app --factory`. Importing
  `main.py` MUST NOT read the environment or open a connection; tests call
  `create_app(settings)` directly.
- `frontend/` and the backend share no code and no files except `openapi.json`, which the
  backend writes and the frontend reads.
- An app with no business rules MAY omit `domain/` and `services/`, and MUST NOT add them
  to match this layout. A new top-level dir needs an ADR only if it changes a row of the
  import tables below.

| Backend directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | stdlib, `domain/` | FastAPI, Starlette, Pydantic, SQLAlchemy, httpx, `adapters/`, `api/` |
| `services/` | `domain/`, own `ports.py` | FastAPI, Starlette, SQLAlchemy, httpx, `adapters/`, `api/` |
| `adapters/` | `services/ports.py`, `domain/`, SQLAlchemy, httpx | FastAPI, Starlette, `api/` |
| `api/` | `services/`, `domain/` types, FastAPI, Pydantic | SQLAlchemy, `adapters/` |

| Frontend directory | MAY import | MUST NOT import |
|---|---|---|
| `src/api/` | `schema.d.ts`, `openapi-fetch`, TanStack Query | `src/app/`, `src/components/` |
| `src/app/`, `src/components/` | `src/api/` hooks and types, React, `next/*` | `openapi-fetch`, `schema.d.ts` directly |

- No framework type (`Request`, `Response`, `Depends`) outside `api/` and `main.py`.
- A route handler parses input through its Pydantic schema, calls one use case, and
  returns the result. A decision about *what the application does*, taken in a handler or
  anywhere in `frontend/`, is a defect. The frontend decides how things look, never what
  is allowed.
- `fetch` and `openapi-fetch` appear only in `src/api/`. Nothing else in the frontend
  reaches the network.
- `tests/test_boundaries.py` walks the imports of `domain/` and `services/` and fails on
  any framework, SQLAlchemy or httpx, and walks `api/` and fails on SQLAlchemy and
  `<app>.adapters`. `frontend/tests/boundaries.test.ts` fails on any import that breaks the
  frontend table and on any `fetch(` outside `src/api/`. These two are the boundary check
  CI and §8 refer to.

---

## 4. Runtime

### 4.1 Request and rendering model

Backend — FastAPI runs each plain `def` handler on a worker thread.

- Route handlers are plain `def` using the sync `Session`. An `async def` handler runs on
  the event loop shared by every request, so it MUST NOT call anything that blocks — a sync
  DB call, a sync HTTP client, file I/O.
- One request is one transaction. The request-scoped session dependency commits after a
  successful handler and rolls back on any exception. Services and adapters MUST NOT call
  `commit()`.
- Every endpoint under `/api/v1` is a public HTTP endpoint, whether or not a page calls it.
  Each one validates its input and resolves the current user itself; the use case checks
  what that user may do. Hiding a page or a button is not authorization.
- `GET` and `HEAD` never change state.
- Every string and list field in an API schema has a maximum length, and every list
  endpoint a maximum page size. A request body of up to 1 MiB (1,048,576 bytes) is
  accepted; one byte more is refused with 413, unless an ADR raises the limit for one
  endpoint. Bytes are counted as they arrive, also when the request has no
  `Content-Length`, and the request is refused as soon as the count passes the limit,
  without the body being read whole.
- Cross-site request forgery: every state-changing request MUST carry the custom header
  the API client in `src/api/client.ts` sets on every request, and is refused with 403
  without it, or when its `Origin` header is present and is not the app's own origin from
  `Settings`. A browser cannot add a custom header to a cross-site request without CORS
  approval, so this defence holds only while the API sends no CORS headers.
- An expected failure — invalid input, a rule the use case refused — returns a 4xx JSON
  body naming the field or the reason, which the page shows next to the field. Only
  unexpected errors are 500.
- Work that outlives the request does not run in the web process: a free-tier host stops
  it on spin-down without warning. `BackgroundTasks` is only for effects whose loss is
  acceptable — never one that costs money or writes to a third party.

Frontend — pages are HTML files built once by `next build`; everything after the first
paint happens in the browser.

- Server components run only during `next build`, when no backend exists. They MUST NOT
  fetch, read the session, or read application data. Every read of application data
  happens in a client component through a hook in `src/api/`.
- `"use client"` goes on the smallest component that needs it, never on a layout.
- Who the user is comes from the API on each page load. The frontend MUST NOT keep
  identity or session data in `localStorage`, `sessionStorage`, or a readable cookie.
- Every query renders its loading state and its error state. An error MUST NOT render as
  an empty list or a blank page.
- A component that calls `useSearchParams()` sits inside a `<Suspense>` boundary, or the
  export fails to build.
- Data reaches the DOM through JSX. `dangerouslySetInnerHTML` with any value that is not a
  compile-time constant is PROHIBITED. A URL from data goes into `href` or `src` only after
  its scheme is checked to be `http:` or `https:`.

### 4.2 Paths

- Every page is a file in `out/`. A path segment that varies with data needs
  `generateStaticParams()`, which is fixed at build time, so record IDs travel in query
  parameters (`/items/?id=3`), never in path segments.
- `trailingSlash: true`, so each page exports as `<path>/index.html`, a URL without the
  slash redirects to the one with it, and links are written with it.
- Paths under `/api/` are answered only by the API. An unknown `/api/` path answers a JSON
  404, never the HTML 404 page. Any other unknown path answers 404 with the page built from
  `not-found.tsx`, never a redirect to the home page.
- A published page URL is a contract. The export cannot redirect, so renaming one adds a
  permanent redirect to the backend in the same commit.
- HTML responses carry `Cache-Control: no-cache`. Files under `/_next/static/` are
  content-hashed and carry a year-long `immutable` cache. A browser holding stale HTML after
  a deploy requests script files that no longer exist.
- The container's filesystem is read-only to the app except one data directory, `DATA_DIR`,
  read from `Settings`; on a host without a persistent disk nothing written there survives
  a restart. The export's location is `FRONTEND_DIR` in `Settings`. An adapter MUST NOT
  resolve a path of its own.
- Everything in `frontend/public/` and in the export is served to anyone.

### 4.3 Persistence

- On a host without a persistent disk — Render's free web service — durable data goes to
  PostgreSQL, Neon Free by default; Render's free Postgres is never the store of record. An
  app with no durable data uses SQLite in a temp dir, or no database.
- The app reaches the database only through `DATABASE_URL` in `Settings`. Moving to another
  PostgreSQL provider changes that variable and nothing in the code.
- Development and the default test run use SQLite, offline. A PostgreSQL project also runs
  the adapter tests and `alembic upgrade head` against a PostgreSQL service container in CI,
  and a migration MUST pass on both. A local PostgreSQL MAY be used but MUST NOT be
  required. SQL only one engine accepts stays in `adapters/` and is covered by the
  PostgreSQL run.
- One `Engine` per process, created in `main.py`. A `Session` MUST NOT cross threads or
  requests. On Neon, the pool points at the pooled endpoint, is sized for Neon Free's
  connection limit, and sets a connect timeout and PostgreSQL `statement_timeout`, 10 s
  each unless an ADR names other values: the first query after Neon has slept is slow, not
  failed, and no query hangs a request.
- SQLite: `PRAGMA foreign_keys=ON`, WAL, and a non-zero `busy_timeout` on every connection,
  set in a connect event. A write that still hits `SQLITE_BUSY` fails as an error; it MUST
  NOT be retried by re-running the use case.
- Alembic revisions are generated with `--autogenerate`, then read and corrected by hand.
  An autogenerated revision MUST NOT be applied unread: it misses renames, data moves and
  server defaults.
- `adapters/migrate.py` runs at container start, before the server. If revisions are
  pending on SQLite, it first backs the file up beside it with `sqlite3.Connection.backup`
  — a file copy is PROHIBITED, since in WAL mode the `.db` file alone is not a consistent
  snapshot — verifies the backup opens at the old revision, then runs `alembic upgrade
  head`. On failure it exits non-zero, the server does not start, and original and backup
  stay.
- A database at a revision the code does not know → refuse to start, log why.
- PostgreSQL: the app cannot dump its own database. Before a deploy that carries a
  migration, the human suspends the service on the host, takes a `pg_dump`, confirms it
  restores into an empty database, and deploys; the service serves again only on the new
  version. The web service is the only writer to the database; anything else that writes
  is stopped too. No write lands after the dump, and no old instance runs during the
  migration.
  §5.4 stops CI for this. Neon's point-in-time restore is not the backup.
- An app holding data its users cannot recreate MUST take a backup on a schedule and keep
  it off the database host. A restore from that backup into an empty database MUST have
  succeeded once before the app holds real data. Backups are encrypted before they leave
  the CI job, with a key kept apart from the store that holds them. A failed backup run
  MUST fail a scheduled CI job whose failure notification reaches the human. Each run
  restores its own dump into an empty database and checks, as a minimum proof that the
  dump restores, its Alembic revision and that every table the models define exists,
  before keeping it; a failed check fails the run.
  The data loss users can accept, the schedule that follows from it, retention, store and
  key location are recorded in `DECISIONS.md`.
- Shipped revisions are append-only: a file under `migrations/versions/`, once on `main`,
  is never modified or deleted. Fixes are new revisions.

### 4.4 Settings, credentials, errors

- Settings: a frozen dataclass built once by `config.py` from the environment, validated
  there, passed explicitly. A missing or invalid required variable stops startup with a
  message naming the variable, never its value. A server MUST NOT start on guessed
  defaults.
- The export has no runtime configuration. It reaches the API at its own origin, so no
  variable — secret or not — is compiled into the frontend, and `next build` MUST succeed
  with no environment variables set.
- A server has no OS credential store, so this archetype names none. A login password is
  only ever verified, so the app stores only its Argon2 hash. Storing an end user's own
  third-party credential needs an ADR naming how it is encrypted at rest and with which
  deployment key.
- No mail provider is selected. A feature that sends mail — password reset, address
  verification, notifications — needs an ADR naming the provider and its free-tier terms
  before it is built. Until then, a password reset is done by an administrator.
- Sessions: a random session ID in a cookie with `HttpOnly`, `Secure` and `SameSite=Lax`,
  the row server-side, rotated on login, deleted on logout, expiring after an idle
  time of 24 hours and an absolute time of 30 days, unless an ADR names other values.
- Failed logins are counted in the database, so the limits survive a restart. Two
  separate counters, unless an ADR names other values:
  - Account and client address together: after 5 consecutive failures the pair is refused
    for 15 minutes. Its count resets on success and when the refusal expires. Keying by the
    pair means an attacker elsewhere cannot lock the owner out.
  - Client address alone: failures in the last 15 minutes, rolling, never reset. While
    there are 20 or more, the address may make one sign-in attempt per minute; it is
    slowed, not blocked, so users sharing the address can still sign in.
- The start of a refusal or a slowdown is logged once as a warning; the attempts it
  refuses are not logged one by one.
- A sign-in response is the same whether or not the account exists, and a sign-in for an
  unknown account still runs one Argon2 verification, so both paths cost the same.
- Behind the host's TLS proxy, Uvicorn MUST trust that proxy's forwarded headers
  (`--forwarded-allow-ips`, set per host), or `Secure` cookies break and the rate limit
  sees one client address. The walking skeleton (§5.2) confirms two clients are counted
  separately, and that an `X-Forwarded-For` header the client sets itself does not change
  the address counted.
- The backend sets, on every response, `X-Content-Type-Options: nosniff`,
  `Referrer-Policy: strict-origin-when-cross-origin`, and this Content-Security-Policy:
  `default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';
  img-src 'self' data:; font-src 'self'; connect-src 'self'; object-src 'none';
  base-uri 'self'; form-action 'self'; frame-ancestors 'none'`. The export cannot set
  headers. Exported pages carry inline scripts and no server issues a nonce, so
  `script-src` needs `'unsafe-inline'`, and this policy does not stop an injected inline
  script from running. Such a script can call the API as the signed-in user; the policy
  only limits what it can load from or send to other origins. The defence against
  injection is §4.1's rules on how data reaches the DOM. A hash-based policy needs every
  exported page's inline scripts hashed after each build and served as per-page headers;
  it needs an ADR. Changing a directive
  needs an ADR.
- Logs go to stdout, one line per event, with a request ID. The app writes no log files.
- An error response carries the request ID and a message for the user, never SQL, a
  traceback, or an exception's text. `debug` is off outside development.

---

## 5. Build and package

Procedure: skill `web-api-release`.

### 5.1 Lock the toolchain

- `uv.lock` pins Python dependencies. `packageManager` and `devEngines.runtime` (with
  `onFail: "download"`) are the single source for the pnpm and Node versions; CI installs
  from both. Every install outside a developer's first setup uses `uv sync --locked` and
  `pnpm install --frozen-lockfile`.
- Dependency install scripts run only for packages named in `allowBuilds` in
  `pnpm-workspace.yaml`. Allowing every package, or setting `minimumReleaseAge` to 0 to get
  a just-published version, is PROHIBITED.
- Both `Dockerfile` base images are pinned by digest: the Node stage to the major in
  `devEngines.runtime` with the pnpm version in `packageManager`, the Python stage to the
  minor in `.python-version` with a pinned uv. A moving tag makes the image built at
  release differ from the one CI tested.
- `openapi-typescript` and the shadcn CLI are locked devDependencies, run with `pnpm exec`.

### 5.2 Build

- `openapi.json` is written by a backend command, without starting a server, and
  `schema.d.ts` is generated from it. A change to any route or API schema regenerates both
  in the same commit.
- `pnpm build` runs `tsc --noEmit`, then `next build`, producing `out/`. `next.config.ts`
  sets `images.unoptimized`: the default image loader needs a server.
- Tailwind v4 is configured in `src/app/globals.css` (`@import "tailwindcss";`) and loaded
  through `@tailwindcss/postcss`.
- In development, the Next.js dev server forwards `/api/` and `/healthz` to the local
  backend. That forwarding exists only in development; the export contains none.
- The image copies `out/` from the Node stage to `FRONTEND_DIR`, installs with
  `uv sync --locked --no-dev`, runs as a non-root user, runs `adapters/migrate.py` before
  the server, and listens on `0.0.0.0` and the port the host passes in `PORT`, with
  `--forwarded-allow-ips` (§4.4).
- A project that will be deployed deploys a walking skeleton — one page that reads one
  endpoint, sign-in, the health check, the real host and database — before any feature
  beyond it is built. Leaving the first real deploy until release is PROHIBITED.

### 5.3 Self-check

`GET /healthz` answers 200 with the version from `pyproject.toml` and the DB revision when
the database answers and is at head; 503 otherwise. It reveals nothing else.

The smoke test (`tests/smoke/`) starts the built image against an empty database, waits
for health, and exits 0 — or non-zero naming each failure. It checks that:

- `/` answers 200 with HTML and `Cache-Control: no-cache`;
- an unknown page path answers 404 with HTML, and an unknown `/api/v1/` path answers 404
  with JSON;
- every header §4.4 requires is present with its required value;
- a state-changing API request without the custom header of §4.1 is refused;
- the image holds no `.env*` file, and the project's secret check passes over the image's
  filesystem as well as the tree;
- the container stops within 10 s.

`DATA_DIR` is a fresh temp dir, every secret is a test placeholder, and the database is a
new SQLite file — or, for a PostgreSQL project in CI, an empty database in the service
container. Timeouts: healthy within 60 s, each request within 10 s. A timeout fails the
test.

### 5.4 Package

| Output | Where it runs |
|---|---|
| Container image from the `Dockerfile` at a tagged commit | Render, built by the host from that tag; or an Oracle VM, pulled or built there |

- `version` in `pyproject.toml` is the single source; `/healthz`, the image tag and the Git
  tag read it. `frontend/package.json` carries no version. CI fails a tag that does not
  match.
- Auto-deploy on push is off. CI triggers the host's deploy for the tagged commit after
  every check passes — except when anything under `migrations/versions/` differs from the
  previous tag. CI then stops before deploying, and the human deploys after the backup in
  §4.3.
- Rollback redeploys the previous tag, and only when nothing under `migrations/versions/`
  changed since it; otherwise the human decides first (`GLOBAL_RULES.md` §12).
- `README.md` tells users the first visit after idle may take about a minute.

### 5.5 CI

- Push / PR, backend: `uv sync --locked`, Ruff, mypy, pytest offline including the boundary
  check; `alembic check` and a failure if a revision file that exists on `main` was
  modified or deleted, if the app persists data. A PostgreSQL project adds the adapter
  tests and migrations against a PostgreSQL service container.
- Push / PR, contract: regenerate `openapi.json` and `schema.d.ts`; fail on any diff.
- Push / PR, frontend: `pnpm install --frozen-lockfile`, install Chromium on the runner,
  Biome, `pnpm build`, Vitest in
  Node and Chromium including the boundary check, Playwright against the backend serving
  `out/`.
- Then build the image and run the smoke test against it.
- Version tag: all of the above → trigger the deploy (unless §5.4 stops it) → confirm that
  `/healthz` reports the tagged version and `/` answers 200.
- The deploy hook URL is set only on the deploy step; no other job or step can read it.

---

## 6. Testing

| Layer | Tool | Framework |
|---|---|---|
| `domain/`, `services/` | pytest with fake ports | MUST NOT import FastAPI |
| `adapters/` | pytest against a real SQLite file in `tmp_path`, schema from `alembic upgrade head`; PostgreSQL projects repeat this in CI (§4.3) | no |
| `api/` | `TestClient` on `create_app(settings)` | yes |
| Plain TypeScript (formatting, mapping in `src/api/`) | Vitest, `node` | no |
| Components | Vitest browser mode, Chromium, `vitest-browser-react`; data passed as props | React only |
| Pages, sign-in, flows | Playwright against the backend serving `out/`, fresh SQLite database | yes |
| Built image | smoke test (§5.3) | yes |

Most tests are in the first three rows. A component or page test proves an input reaches
the right endpoint and the result is shown — not the rule itself.

- Tests construct `Settings` explicitly. A test that passes only because the developer's
  `.env` is present is a defect. Dependency overrides are set per test and cleared after
  it.
- Every endpoint's use case states who may call it. One that requires a session is tested
  without one; one limited to the owner of the data is tested as another user; one limited
  to a role is tested without it — each is refused. A public endpoint is tested without a
  session and succeeds.
- A state-changing request without the custom header, and one with a foreign `Origin`, are
  each refused (§4.1).
- Pages are tested against the real backend, never a mocked API: drift between the two
  halves is what those tests exist to catch. Each Playwright run starts the backend on a
  fresh database in a temp dir and creates its users through sign-up.
- Controls are found by role and label text, never by CSS selector or id. Assert what the
  user sees: text, `disabled`, `aria-invalid`, the URL after navigation.
- One test submits input containing markup and asserts it comes back as text.
- A Content-Security-Policy violation reported in the browser fails the Playwright run.
- A request body of exactly 1 MiB is accepted and one of 1 MiB + 1 byte is refused with
  413, each with and without `Content-Length`.
- Waiting real time — `time.sleep()`, `setTimeout`, `page.waitForTimeout` — is PROHIBITED;
  wait for an element, a URL, or a response.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. TanStack Query v4 idioms: positional `useQuery(key, fn)`, `onSuccess` / `onError` on a
   query, `cacheTime`, `isLoading` for "no data yet". v5 takes one options object, has no
   query callbacks, and uses `gcTime` and `isPending`.
2. Pydantic v1 and older FastAPI idioms: `class Config: orm_mode = True`, `.dict()`,
   `@app.on_event("startup")`. Pydantic v2 uses `model_config = ConfigDict(from_attributes=True)`
   and `.model_dump()`; FastAPI uses a `lifespan` function.
3. Next.js export idioms from before version 14: the `next export` command, and
   `getStaticProps` / `getServerSideProps`, which belong to the Pages Router. The App
   Router exports with `output: "export"`.
4. Switching to `output: "standalone"`, adding a route handler, or running a Node server to
   escape an export limit — a path segment, a redirect, a header. That changes the
   archetype's shape, which needs an ADR, not a workaround.
5. Turning on CORS, or putting the backend's address in frontend code, to get past a
   development forwarding problem. CORS breaks §4.1's forgery defence; an address compiled
   into the export ties the build to one host.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes on both halves
- [ ] Every new or changed endpoint validates its input and has the access tests of §6
- [ ] An API change regenerated `openapi.json` and `schema.d.ts` in the same commit

Schema change:

- [ ] New Alembic revision appended and read by hand, none edited; upgraded from the
      previous head on a database with rows — on PostgreSQL too, for a PostgreSQL
      project; `alembic check` clean; a failing migration leaves the revision and rows
      unchanged and the backup opens

Dependencies, `Dockerfile`, `next.config.ts`, `main.py`, or `config.py` change:

- [ ] `pnpm build` passes from a frozen install with no environment variables; image builds
      and the smoke test (§5.3) passes
