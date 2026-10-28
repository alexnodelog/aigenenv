# rules/web-python.md

Web app, one Python process serves both the pages and the logic. Server-rendered, no Node,
no bundler, no `node_modules`.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if there is no server logic at all (`web-static`), if the UI needs a
JavaScript framework or real client-side state (`web-ts`, or `web-api` when the server must
also be Python), or if one person uses it on one machine with local data
(`pc-app-python`).

**Two tracks.** Pick one at project start and record it in `DECISIONS.md`. A repository
holds one track, never both.

- **Track A — Streamlit.** Every user is known by name: internal tools, dashboards, data
  apps, PoCs.
- **Track B — FastAPI + Jinja2 + htmx.** Anything else: anonymous visitors, the app's own
  accounts, a designed UI, or pages that must be addressable by URL.

Both tracks share `domain/`, `services/` and `adapters/` (§3). Moving from A to B rewrites
`ui/` into `web/` and replaces OIDC login with the app's own accounts (§4.4); the shared
layers stay. A move is recorded as an ADR. A Track A tool used for work is hosted like
Track B (§1 Hosting).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Python | 3.12+, `.python-version`, uv-managed | — | 3.11 and below, system Python |
| Packaging | uv (`pyproject.toml` with `[build-system]`, `uv.lock`) | — | Poetry, pipenv, conda, bare pip; a hand-maintained `requirements.txt` |
| Framework — Track A | Streamlit | — | Gradio, Dash, Panel, NiceGUI |
| Framework — Track B | FastAPI | — | Flask, Django, Litestar; Starlette used directly |
| HTML — Track B | Jinja2 via `Jinja2Templates`, autoescape on | — | HTML built in Python strings; `\|safe` or `Markup` on anything a user supplied |
| Interactivity — Track B | htmx 2.x, vendored as `static/vendor/htmx.min.js` | htmx 4; Alpine.js for local client state | React, Vue, Svelte, a bundler, npm; any `<script src>` loaded from a third-party host at runtime |
| Styling — Track B | Tailwind CSS v4 via its standalone CLI; compiled CSS committed, never hand-edited | Plain CSS | A Node-based asset pipeline; the Tailwind Play CDN; v3-style `tailwind.config.js` |
| Styling — Track A | Theme in `.streamlit/config.toml` | — | CSS or JS injected through `unsafe_allow_html` or a component to restyle the app |
| Charts — Track A | Streamlit built-ins, Plotly | — | — |
| Persistence | SQLAlchemy 2.0 (`Mapped[]`), sync `Session`. SQLite for development, tests, and hosts with a persistent disk; PostgreSQL via psycopg 3 in production when the host has no persistent disk and the app keeps durable data (§4.3) | PostgreSQL on a host that has a persistent disk | async SQLAlchemy; SQLModel; any other ORM; string-built SQL |
| Migrations | Alembic, `render_as_batch=True` on SQLite | — | `create_all()` against a deployed DB; editing a shipped revision; hand-applied schema changes |
| Schemas — Track B | Pydantic v2, separate from ORM models | — | ORM models passed to templates or returned from routes |
| Config | Frozen dataclass built from `os.environ` in `config.py`; `.env` loaded in development by `uv run --env-file .env` | pydantic-settings | `python-dotenv` calls in application code; `os.environ` read anywhere but `config.py` |
| Auth — Track A | `st.login` (OIDC, `streamlit[auth]`) + an email allowlist | — | A password form written in Streamlit; streamlit-authenticator |
| Accounts — Track B | Users table in the app's own database; server-side session table, opaque session-ID cookie; passwords hashed with Argon2 via pwdlib | — | Accounts held by a hosting or database vendor's auth service (Supabase Auth, Firebase Auth and the like); passlib; JWT for browser sessions; identity kept in a signed client-side cookie; tokens in `localStorage` |
| User file uploads | None | Storage chosen by ADR | The local disk of a host without a persistent disk |
| Background work | None — work finishes inside the request or rerun | FastAPI `BackgroundTasks` for best-effort, non-durable follow-ups | Threads or tasks started from a handler that must outlive it; a job-queue service (Celery, RQ) |
| Server — Track B | Uvicorn, one process | — | Multiple workers or processes on SQLite |
| Lint / format | Ruff | — | Black + isort + flake8 |
| Type check | mypy strict on `domain/`, `services/` | pyright | — |
| Tests | pytest + pytest-cov; FastAPI `TestClient` (Track B); `streamlit.testing.v1.AppTest` (Track A) | — | unittest-only |
| Deploy artifact | `Dockerfile` → container image | The host's native Python runtime building from `uv.lock` | Docker required for local development or tests; a release image built on a developer machine |
| Hosting | Render free web service, plus Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free VM; Streamlit Community Cloud — Track A, personal non-commercial projects with public data only | Render's local disk or Render free Postgres as the store of record; a work tool on Streamlit Community Cloud; artificial traffic or CPU load to defeat a host's idle policy |

---

## 2. Prerequisites

`uv sync` neither installs nor reveals these. Procedure: skill `python-web-release`.

| Item | Scope |
|---|---|
| Tailwind standalone CLI | System, Track B only. A tagged v4 release, never `latest`; version and SHA-256 recorded in `DECISIONS.md` |
| `static/vendor/htmx.min.js` | In the repository, Track B only. Version and SHA-256 recorded in `DECISIONS.md`. The filename carries no version |
| Docker or Podman | System. Optional on the developer's machine; required in CI (§5.5) |
| Host account | Render, Oracle Cloud, or Streamlit Community Cloud — whichever `DECISIONS.md` names |
| Neon account | Account, only when the app keeps durable data. Before first use, confirm in Neon's current terms that the free plan permits commercial use, and confirm the data region if the app holds personal data; record both in `DECISIONS.md` |
| PostgreSQL client tools (`pg_dump`, `psql`) | System, PostgreSQL projects only. Major version at or above the server's, or `pg_dump` refuses to run |
| OIDC client registration | Account, Track A with login. Client ID and secret go in `.streamlit/secrets.toml`, never in Git |

---

## 3. Layout

```
<repo>/
├── pyproject.toml          deps, tool config
├── uv.lock
├── .python-version
├── .env.example
├── Dockerfile
├── alembic.ini             if the app persists data
├── migrations/             Alembic env.py + versions/
├── .streamlit/             Track A: config.toml committed; secrets.toml gitignored,
│                           secrets.toml.example committed
├── assets/app.css          Track B: Tailwind input
├── src/<app>/
│   ├── main.py             Track B: create_app() — entry point + composition root
│   ├── streamlit_app.py    Track A: entry script + composition root
│   ├── config.py           Settings dataclass; the only reader of os.environ
│   ├── domain/             pure Python
│   ├── services/           use cases + ports.py (typing.Protocol)
│   ├── adapters/           SQLAlchemy models + repositories, HTTP clients, clock
│   │   └── migrate.py
│   ├── web/                Track B: routes/, templates/, templates/partials/, static/
│   └── ui/                 Track A: pages/
├── tests/                  conftest.py, test_boundaries.py, domain/, services/, adapters/,
│                           web/ or ui/, smoke/
└── dist/                   gitignored
```

- `main.py` or `streamlit_app.py` — whichever the track uses — is the only place that reads
  `Settings`, creates the `Engine`, and constructs adapters. The other one does not exist.
- Track B serves with `uvicorn <app>.main:create_app --factory`. Importing `main.py` MUST
  NOT read the environment or open a connection; tests call `create_app(settings)` directly.
- Track A builds its services once with `st.cache_resource` in `streamlit_app.py` and passes
  them, with the current user's identity, into each page as arguments. A page in `ui/`
  never constructs a service or reads `st.user` itself.
- An app with no business rules MAY omit `domain/` and `services/`, and MUST NOT add them
  to match this layout. A new top-level dir needs an ADR only if it changes a row of the
  import table below.

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | stdlib, `domain/` | Streamlit, FastAPI, Starlette, Jinja2, SQLAlchemy, httpx, `adapters/`, `web/`, `ui/` |
| `services/` | `domain/`, own `ports.py` | Streamlit, FastAPI, Starlette, Jinja2, SQLAlchemy, httpx, `adapters/`, `web/`, `ui/` |
| `adapters/` | `services/ports.py`, `domain/`, SQLAlchemy, httpx | Streamlit, FastAPI, `web/`, `ui/` |
| `web/`, `ui/` | `services/`, `domain/` types, the track's framework | SQLAlchemy, `adapters/` |

- No framework type (`Request`, `st.session_state`, `HTMLResponse`) outside `web/`, `ui/`
  and the composition root.
- A route handler or page parses input, calls one use case, and renders the result. A
  decision about *what the application does*, taken in a handler, page, or template, is a
  defect. Hiding a button in a template is not authorization; the use case checks.
- `tests/test_boundaries.py` walks the imports of `domain/` and `services/` and fails on any
  framework, SQLAlchemy, or httpx; it walks `web/` and `ui/` and fails on SQLAlchemy and
  `<app>.adapters`. That test is the boundary check CI and §8 refer to.

---

## 4. Runtime

### 4.1 Request and rerun model

Track A — Streamlit reruns the whole script on every interaction, in one thread per
session, and every session shares the process.

- Work that can exceed ~100 ms is cached with `st.cache_data` (explicit `ttl`) or placed
  behind a button or form. It MUST NOT run at the top of the script on every rerun.
- A write, a send, or any external effect runs only in a widget callback (`on_click`,
  `on_change`) or a form submit, never in top-level script flow, where the next rerun repeats
  it.
- `st.cache_resource` holds only objects that are process-wide, thread-safe, and the same
  for every user: the `Engine`, a loaded model. User data in it is visible to every user.
- `st.cache_data` is shared across users too. A cached function returning user-specific data
  MUST take the user's identity as an argument.
- Module-level mutable state is PROHIBITED: it is shared by every session.
- `st.session_state` holds UI state only. It is lost on refresh and is separate per tab; it
  is never the source of truth for data.
- Polling uses `st.fragment(run_every=...)`. `while True` loops and `time.sleep()` in the
  script are PROHIBITED.

Track B — one request, one thread from Uvicorn's pool.

- Route handlers are plain `def` using the sync `Session`; FastAPI runs them in a
  threadpool. An `async def` handler MUST NOT call anything that blocks — a sync DB call, a
  sync HTTP client, file I/O — because it stalls every other request. Mixing the two styles
  in one handler is a defect.
- One request is one transaction. The request-scoped session dependency commits after a
  successful handler and rolls back on any exception. Services and adapters MUST NOT call
  `commit()`.
- `GET` and `HEAD` never change state. Every state-changing request is `POST`, `PUT`,
  `PATCH` or `DELETE` and carries a CSRF token, sent by htmx through `hx-headers` on
  `<body>`.
- A route that answers htmx returns a fragment from `templates/partials/` when the
  `HX-Request` header is present. Any URL a user can land on — directly, by bookmark, or
  through `hx-push-url` — MUST render a full page when requested without that header.
- Work that outlives the request does not run in the web process: a free-tier host stops
  it on spin-down without warning. `BackgroundTasks` is only for effects whose loss is
  acceptable — never one that costs money or writes to a third party.

### 4.2 Paths

- The container's filesystem is read-only to the app except one data directory, `DATA_DIR`,
  read from `Settings`. An adapter MUST NOT resolve a path of its own; it receives it.
- On a host without a persistent disk, nothing written to `DATA_DIR` survives a restart.
  SQLite files and user uploads MUST NOT be kept there on such a host.
- Templates, static files and bundled resources are located relative to the package
  (`importlib.resources` or `Path(__file__)`), never the working directory.

### 4.3 Persistence

- On a host without a persistent disk — Render's free web service — durable data goes to
  PostgreSQL, Neon Free by default; Render's free Postgres is never the store of record.
  SQLite on the disk of an Oracle Cloud Always Free VM is the Alternative. An app with no
  durable data uses SQLite in a temp dir, or no database.
- The app reaches the database only through `DATABASE_URL` in `Settings`. Moving to another
  PostgreSQL provider changes that variable and nothing in the code.
- Development and the default test run use SQLite, offline. A PostgreSQL project also runs
  the adapter tests and `alembic upgrade head` against a PostgreSQL service container in CI,
  and a migration MUST pass on both. A local PostgreSQL MAY be used but MUST NOT be required
  to develop or to run the default tests. SQL only one engine accepts stays in `adapters/`
  and is covered by the PostgreSQL run.
- One `Engine` per process, created in the composition root. A `Session` MUST NOT cross
  threads or requests.
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
- PostgreSQL: the app cannot dump its own database. Before a deploy that carries a
  migration, the human takes a `pg_dump` and confirms it restores into an empty database;
  §5.4 stops CI for this. The provider's point-in-time restore is not the backup.
- An app holding data its users cannot recreate MUST take a backup on a schedule and keep
  it off the database host. A restore from that backup into an empty database MUST have
  succeeded once before the app holds real data. A failed backup run MUST fail a scheduled
  CI job, not only write a log line. Schedule and method are recorded in `DECISIONS.md`.
- A database at a revision the code does not know → refuse to start, log why.
- Shipped revisions are append-only. Fixes are new revisions.

### 4.4 Settings, credentials, errors

- Settings: a frozen dataclass built once at startup by `config.py` from the environment,
  validated there, passed explicitly. A missing or invalid required variable stops startup
  with a message naming the variable, never its value. A server MUST NOT start on guessed
  defaults.
- Track A's `[auth]` section is the one deployment secret not in `.env.example`: `st.login`
  reads it from `.streamlit/secrets.toml`, which is gitignored, documented by
  `secrets.toml.example`, and supplied by the host as a secret file.
- A server has no OS credential store, so this archetype names none. A login password is
  only ever verified, so the app stores only its Argon2 hash and never places it in session
  state. Storing an end user's own third-party credential — their API token, their mail
  password — needs an ADR naming how it is encrypted at rest and with which deployment key.
- No mail provider is selected. A feature that sends mail — password reset, address
  verification, notifications — needs an ADR naming the provider and its free-tier terms
  before it is built. Until then, a password reset is done by an administrator.
- Track B sessions: a random session ID in a cookie with `HttpOnly`, `Secure` and
  `SameSite=Lax`, the row server-side, rotated on login, deleted on logout, expiring after
  a stated idle time and a stated absolute time. Failed logins are rate-limited per account
  and per client address, and a failed login says the same thing whether or not the account
  exists. Behind the host's TLS proxy, Uvicorn MUST trust that proxy's forwarded headers
  (`--forwarded-allow-ips`, set per host), or `Secure` cookies and `url_for` produce
  `http://` links.
- Track A: `st.login` establishes who the user is, not what they may do. The composition
  root checks the email against the allowlist from `Settings` before any page runs and
  stops otherwise. A Track A app on the public internet without login MUST hold only
  public data.
- Logs go to stdout, one line per event, with a request ID. The app writes no log files.
- User-facing errors also never contain SQL. Track B: an exception handler renders a
  generic page — or a fragment for htmx requests — with the request ID, and `debug` is off
  outside development. Track A: production config sets
  `[client] showErrorDetails = "none"`; the default, `"full"`, shows tracebacks to every
  viewer.
- A failed request or callback leaves no partial write: the transaction rolls back.

---

## 5. Build and package

Procedure: skill `python-web-release`.

### 5.1 Lock the toolchain

- `uv.lock` pins Python dependencies. The `Dockerfile` base image uses the minor version in
  `.python-version`, pinned by digest, and copies a pinned uv version — a moving `latest`
  tag makes the image built at release differ from the one CI tested.
- Track B: the Tailwind binary and the vendored htmx file are pinned per §2. Changing
  either is a dependency change.
- If the host builds natively instead (the Alternative in §1), its Python version is set to
  match `.python-version` and its build command is `uv sync --locked`.

### 5.2 Build

- Track B: Tailwind v4 is configured in `assets/app.css` (`@import "tailwindcss";`,
  `@source` lines pointing at `web/templates/`). There is no `tailwind.config.js`. The
  committed stylesheet is regenerated in the same commit as any template or `app.css`
  change.
- The image installs with `uv sync --locked --no-dev`, runs as a non-root user, runs
  `adapters/migrate.py` before the server, and listens on `0.0.0.0` and the port the host
  passes in `PORT`. Track B passes `--forwarded-allow-ips` (§4.4); Track A runs headless.
- A project that will be deployed deploys a walking skeleton — one page, the health
  check, the real host — before any feature beyond it is built. Port, proxy, cold start and
  secrets surface there. Leaving the first real deploy until release is PROHIBITED.

### 5.3 Self-check

Track B exposes `GET /healthz`: 200 with the version from `pyproject.toml` and the DB
revision when the database answers and is at head; 503 otherwise. Track A uses Streamlit's
built-in `/_stcore/health` for liveness; its revision check runs in the smoke test.

The smoke test (`tests/smoke/`) starts the built image, waits for health, requests `/`,
stops the container, and exits 0 — or non-zero with the reason.

- It MUST NOT touch real data or real secrets: `DATA_DIR` is a fresh temp dir, the database
  is a new SQLite file migrated from empty — or, for a PostgreSQL project in CI, an empty
  database in the service container — and every secret is a test placeholder. Login is not
  exercised against a real identity provider.
- Timeouts: healthy within 60 s, `/` within 10 s, stopped within 10 s. A timeout fails the
  test.

### 5.4 Package

| Output | Where it runs |
|---|---|
| Container image from the `Dockerfile` at a tagged commit | Render, built by the host from that tag; or an Oracle VM, pulled or built there |
| `uv.lock` at a tagged commit | Streamlit Community Cloud — personal projects only — which reads `uv.lock` directly |

- `version` in `pyproject.toml` is the single source; `/healthz`, the image tag and the Git
  tag read it. CI fails a tag that does not match.
- Auto-deploy on push is off. CI triggers the host's deploy for the tagged commit after
  every check passes — except when the tag adds an Alembic revision since the previous tag.
  CI then stops before deploying, and the human deploys after the backup in §4.3. Container
  start runs migrations, so this stop is what makes the migration a Blocking step rather
  than a written intention.
- Rollback is a redeploy of the previous tag.
- `README.md` tells users the first visit after idle may be slow.

### 5.5 CI

- Push / PR: `uv sync --locked`, Ruff, mypy, pytest offline including the boundary check,
  `alembic check` if the app persists data. A PostgreSQL project adds the adapter tests and
  migrations against a PostgreSQL service container. Track B regenerates the stylesheet and
  fails on any diff. Then the smoke test — against the built image, or against `uv run` on
  a native-runtime host.
- Version tag: all of the above → trigger the deploy (unless §5.4 stops it) → confirm the
  deployed version and that `/` responds. Track B reads the version from `/healthz`. Track A
  has no version endpoint, so the human confirms the deployed commit on the host's
  dashboard.

---

## 6. Testing

| Layer | Tool | Framework |
|---|---|---|
| `domain/`, `services/` | pytest with fake ports | MUST NOT import Streamlit or FastAPI |
| `adapters/` | pytest against a real SQLite file in `tmp_path`, schema from `alembic upgrade head`; PostgreSQL projects repeat this in CI (§4.3) | no |
| `web/` — Track B | `TestClient` on `create_app(settings)` | yes |
| `ui/` — Track A | `AppTest.from_function`, fake services passed through `args` / `kwargs` | yes |
| Built image | smoke test (§5.3) | yes |

Most tests are in the first two rows. A route or page test proves the input reaches the
right use case and the result is shown — not the rule itself.

- Tests construct `Settings` explicitly. A test that passes only because the developer's
  `.env` is present is a defect.
- Track B: dependency overrides are set per test and cleared after it. Assert status code,
  the text the user sees, and htmx response headers. Every htmx route is tested with and
  without `HX-Request`.
- Track B: one test renders user input containing `<script>` and asserts it comes back
  escaped.
- Track B: a state-changing request without a CSRF token is rejected.
- Track A: `AppTest.from_function` runs the function's *source* as a standalone script. The
  function MUST do its own imports and receive everything else through `args` / `kwargs`;
  a closure variable or a name from the test module does not exist inside it.
- Track A: every `AppTest.run()` takes a timeout, and the test asserts `at.exception` is
  empty before anything else. Tests never contact an OIDC provider; the identity is an
  argument.
- `time.sleep()` to wait for anything is PROHIBITED.

---

## 7. Where agents go wrong

The mistakes agents most often make here. Each breaks the rule in the section named.

1. Tailwind v3 habits: `tailwind.config.js`, `@tailwind base;`, `npm install tailwindcss`.
   This archetype is v4 and the standalone binary (§1, §5.2).
2. htmx versions mixed: 2 and 4 differ in event names, attribute inheritance, and whether
   error responses are swapped. Check the vendored file's major version before writing any
   `hx-` attribute or event listener (§2).
3. A DB write or paid call at the top level of a Streamlit script, repeated on every rerun
   (§4.1).
4. User data in `st.cache_resource`, in `st.cache_data` without the user in its arguments, or
   in a module global — one user sees another's data (§4.1).
5. `async def` handlers calling the sync `Session` or a sync HTTP client (§4.1).
6. A SQLite file or uploads on Render's free disk; Render free Postgres as the only copy of
   the data; a scheduled ping to keep a free service awake (§1, §4.3).
7. Supabase Auth, Firebase Auth or a similar service proposed for login because it is
   quicker (§1).
8. Logic in route handlers, pages or templates; a hidden button treated as authorization
   (§3).
9. `|safe`, `Markup`, or an f-string building HTML from user input (§1).
10. An autogenerated Alembic revision applied unread; `create_all()` at startup (§4.3).
11. "The deploy command succeeded, so it works." Check `/healthz` for the version (§5.5).
12. Reaching for npm, React, or a CDN script to escape an htmx or Streamlit limit. That is a
    change of archetype, which needs an ADR, not a workaround.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes
- [ ] Track A: no effect reachable by a rerun outside a callback; no user data in a shared
      cache or global. Track B: no blocking call in `async def`; no `GET` that writes
- [ ] Tests construct `Settings` explicitly

Schema change:

- [ ] New Alembic revision appended and read by hand, none edited; upgraded from the
      previous head on a database with rows — on PostgreSQL too, for a PostgreSQL
      project; `alembic check` clean; a failing migration leaves the revision and rows
      unchanged and the backup opens

Auth, session, or access change:

- [ ] Track B: the CSRF rejection test (§6) passes

Dependencies, `Dockerfile`, static assets, or composition root change:

- [ ] Image builds and the smoke test (§5.3) passes; Track B stylesheet regenerated
