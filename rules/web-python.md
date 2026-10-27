# rules/web-python.md

Web app, one Python process serves both the pages and the logic. Server-rendered, no Node,
no bundler, no `node_modules`.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. Only what differs, narrows, or
adds. RFC 2119 keywords. **Default** = use it; **Alternative** = allowed with an ADR;
**Prohibited** = MUST NOT. The §1 table's Prohibited column covers technology choices only;
every other prohibition is a MUST NOT where the rule it belongs to lives.

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

Both tracks share every layer below the UI (§3), so moving from A to B rewrites `ui/` into
`web/` and nothing else. That is the boundary's purpose; a move is an ADR, not a rewrite.

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
| Persistence | SQLite via SQLAlchemy 2.0 (`Mapped[]`), sync `Session` | PostgreSQL via SQLAlchemy + psycopg 3 — when the host has no persistent disk (§4.3); async SQLAlchemy | SQLModel; any other ORM; string-built SQL |
| Migrations | Alembic, `render_as_batch=True` on SQLite | — | `create_all()` against a deployed DB; editing a shipped revision; hand-applied schema changes |
| Schemas — Track B | Pydantic v2, separate from ORM models | — | ORM models passed to templates or returned from routes |
| Config | Frozen dataclass built from `os.environ` in `config.py`; `.env` loaded in development by `uv run --env-file .env` | pydantic-settings | `python-dotenv` calls in application code; `os.environ` read anywhere but `config.py` |
| Auth — Track A | `st.login` (OIDC, `streamlit[auth]`) + an email allowlist | — | A password form written in Streamlit; streamlit-authenticator |
| Auth — Track B | Server-side session table, opaque session-ID cookie; passwords hashed with Argon2 via pwdlib | — | passlib; JWT for browser sessions; identity kept in a signed client-side cookie; tokens in `localStorage` |
| Background work | None — work finishes inside the request or rerun | FastAPI `BackgroundTasks` for best-effort, non-durable follow-ups | Threads or tasks started from a handler that must outlive it; a job-queue service (Celery, RQ) — that is a second deployment |
| Server — Track B | Uvicorn, one process | — | Multiple workers or processes on SQLite |
| Lint / format | Ruff | — | Black + isort + flake8 |
| Type check | mypy strict on `domain/`, `services/` | pyright | — |
| Tests | pytest + pytest-cov; FastAPI `TestClient` (Track B); `streamlit.testing.v1.AppTest` (Track A) | — | unittest-only |
| Deploy artifact | `Dockerfile` → container image | The host's native Python runtime building from `uv.lock` | Docker required for local development or tests; a release image built on a developer machine |
| Hosting | Render free web service — only when the server keeps no durable data | Oracle Cloud Always Free VM; Streamlit Community Cloud — Track A only, non-commercial, public non-sensitive data only | Render free disk or Render free Postgres as the store of record; artificial traffic or CPU load to defeat a host's idle policy |

---

## 2. Prerequisites

Verify before the first build; `uv sync` does not install or reveal these.

| Item | Scope |
|---|---|
| Tailwind standalone CLI | System, Track B only. Pin a tagged v4 release, never `latest`; record the version and SHA-256 in `DECISIONS.md` |
| `static/vendor/htmx.min.js` | In the repository, Track B only. Record the version and SHA-256 in `DECISIONS.md`. The filename carries no version |
| Docker or Podman | System. Optional on the developer's machine; required in CI (§5.5) |
| Host account | Render, Oracle Cloud, or Streamlit Community Cloud — whichever `DECISIONS.md` names |
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
  to match this layout. A new top-level dir needs an ADR only if it changes the boundaries
  below.

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
  `<app>.adapters`. That test is the boundary check the CI job and the DoD mean.

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
  it on spin-down without warning. `BackgroundTasks` is for effects whose loss is
  acceptable, and nothing that costs money or writes to a third party (`GLOBAL_RULES.md`
  repeated-operations rule).

### 4.2 Paths

- The container's filesystem is read-only to the app except one data directory, `DATA_DIR`,
  read from `Settings`. An adapter MUST NOT resolve a path of its own; it receives it.
- On a host without a persistent disk, nothing written to `DATA_DIR` survives a restart.
  User uploads and SQLite files MUST NOT be kept there on such a host.
- Templates, static files and bundled resources are located relative to the package
  (`importlib.resources` or `Path(__file__)`), never the working directory.

### 4.3 Persistence

- **Where durable data may live.** Render's free web service loses its filesystem on every
  restart, redeploy and spin-down, and its free Postgres expires 30 days after creation
  with no backups. An app that keeps durable data therefore takes one of two paths, recorded
  as an ADR: SQLite on the persistent disk of an Oracle Cloud Always Free VM, or PostgreSQL
  on a managed free tier whose terms permit commercial use and do not expire the database.
  Check the provider's current terms before choosing; free tiers change yearly.
- One `Engine` per process, created in the composition root. A `Session` MUST NOT cross
  threads or requests.
- SQLite: `PRAGMA foreign_keys=ON`, WAL, and a non-zero `busy_timeout` on every connection,
  set in a connect event. A write that still hits `SQLITE_BUSY` fails as an error; it MUST
  NOT be retried by re-running the use case.
- Alembic revisions are generated with `--autogenerate`, then read and corrected by hand.
  An autogenerated revision MUST NOT be applied unread: it misses renames, data moves and
  server defaults, and on SQLite every `ALTER` needs batch mode.
- `adapters/migrate.py` runs at container start, before the server: if revisions are
  pending on SQLite, it first backs the file up beside it with
  `sqlite3.Connection.backup` — not a file copy, since in WAL mode the `.db` file alone is
  not a consistent snapshot — verifies the backup opens at the old revision, then runs
  `alembic upgrade head`. On failure it exits non-zero, the server does not start, and
  original and backup stay.
- PostgreSQL: the app cannot dump its own database. A deploy carrying a migration is a
  Blocking step (`GLOBAL_RULES.md` §4) that names the dump taken before it.
- A backup on the same VM does not survive losing the VM. An app holding durable data
  SHOULD copy backups off the host on a schedule, method recorded in `DECISIONS.md`.
- A database at a revision the code does not know → refuse to start, log why.
- Shipped revisions are append-only. Fixes are new revisions.

### 4.4 Settings, credentials, errors

- Settings: a frozen dataclass built once at startup by `config.py` from the environment,
  passed explicitly. Values are validated there. A missing or invalid required variable
  stops startup with a message naming the variable, never its value. Unlike a desktop app,
  a server MUST NOT start on guessed defaults.
- Deployment secrets — database URL, OIDC secret, session-signing key — come from the
  environment and are listed in `.env.example`. Track A's `[auth]` section is the one
  exception: `st.login` reads it from `.streamlit/secrets.toml`, which is gitignored,
  documented by `secrets.toml.example`, and supplied by the host as a secret file.
- There is no OS credential store on a server. End-user passwords are stored only as
  Argon2 hashes, never logged, never placed in session state. Storing an end user's own
  third-party credential — their API token, their mail password — needs an ADR naming how it
  is encrypted at rest and with which deployment key.
- Track B sessions: a random session ID in a cookie with `HttpOnly`, `Secure` and
  `SameSite=Lax`, the row server-side, rotated on login, deleted on logout. Behind the host's
  TLS proxy, Uvicorn MUST trust that proxy's forwarded headers (`--forwarded-allow-ips`, set
  per host), or `Secure` cookies and `url_for` produce `http://` links.
- Track A: `st.login` establishes who the user is, not what they may do. The composition
  root checks the email against the allowlist from `Settings` before any page runs and
  stops otherwise. A Track A app on the public internet without login MUST hold only
  public data.
- Logs go to stdout, structured, one line per event, with a request ID. The host collects
  them; the app writes no log files.
- Errors shown to a user name what failed and what to do. They MUST NOT contain a
  traceback, SQL, an internal identifier, or a file path. Track B: an exception handler
  renders a generic page — or a fragment for htmx requests — with the request ID, and
  `debug` is off outside development. Track A: production config sets
  `[client] showErrorDetails = "none"`; its default shows full tracebacks to every viewer.
- A failed request or callback leaves no partial write: the transaction rolls back.

---

## 5. Build and package

### 5.1 Lock the toolchain

- `uv.lock` pins Python dependencies. The `Dockerfile` base image uses the minor version in
  `.python-version`, pinned by digest, and copies a pinned uv version — a moving `latest`
  tag makes the image built at release differ from the one CI tested.
- Track B: the Tailwind binary and the vendored htmx file are pinned per §2. Changing
  either is a dependency change.
- If the host builds natively instead (the Alternative in §1), its Python version is set to
  match `.python-version` and its build command is `uv sync --locked`.

### 5.2 Build

```
# Track B only — regenerate the committed stylesheet
tailwindcss -i assets/app.css -o src/<app>/web/static/app.css --minify

docker build -t <app>:<version> .
```

- Tailwind v4 is configured in `assets/app.css` (`@import "tailwindcss";`, `@source` lines
  pointing at `web/templates/`). There is no `tailwind.config.js`.
- The image installs with `uv sync --locked --no-dev`, runs as a non-root user, and starts
  with `adapters/migrate.py` then the server, listening on `0.0.0.0` and the port the host
  passes in `PORT`:
  - Track B: `uvicorn <app>.main:create_app --factory --forwarded-allow-ips <host proxy>`
  - Track A: `streamlit run src/<app>/streamlit_app.py --server.headless true`
- Deploy a walking skeleton — one page, `/healthz`, the real host — in the first week.
  Port, proxy, cold start and secrets surface there. Leaving the first real deploy until
  release is PROHIBITED.

### 5.3 Self-check

Track B exposes `GET /healthz`: 200 with the version from `pyproject.toml` and the DB
revision when the database answers and is at head; 503 otherwise. Track A uses Streamlit's
`/_stcore/health` for liveness; its revision check runs in the smoke test.

The smoke test (`tests/smoke/`) starts the built image, waits for health, requests `/`,
stops the container, and exits 0 — or non-zero with the reason.

- It MUST NOT touch real data or real secrets: `DATA_DIR` is a fresh temp dir, the database
  is a new SQLite file migrated from empty, and every secret is a test placeholder. Login
  is not exercised against a real identity provider.
- Timeouts: healthy within 60 s, `/` within 10 s, stopped within 10 s. A timeout fails the
  test.

### 5.4 Package

| Output | Where it runs |
|---|---|
| Container image from the `Dockerfile` at a tagged commit | Render, built by the host from that tag; or an Oracle VM, pulled or built there |
| `uv.lock` at a tagged commit | Streamlit Community Cloud, which reads `uv.lock` directly |

- `version` in `pyproject.toml` is the single source; `/healthz`, the image tag and the Git
  tag read it. CI fails a tag that does not match.
- Auto-deploy on push is off. CI triggers the host's deploy for the tagged commit after
  every check passes (§5.5).
- Rollback is a redeploy of the previous tag. It is safe only if that version runs against
  the current schema; a migration that breaks this is stated before the deploy.
- A free Render service sleeps after 15 minutes idle and takes about a minute to wake.
  `README.md` tells users the first visit may be slow.

### 5.5 CI

- Push / PR: `uv sync --locked`, Ruff, mypy, pytest (offline), boundary check,
  `alembic check` if the app persists data. Track B also regenerates the stylesheet and
  fails on `git diff --exit-code`. Then build the image and run the smoke test.
- Version tag: all of the above → trigger the deploy → confirm `/healthz` reports the tagged
  version and `/` responds. Allow two minutes for a cold start before failing. On failure,
  stop and report; never redeploy or retry automatically.
- Anything a person other than the developer uses comes from a tag that passed this job.

---

## 6. Testing

| Layer | Tool | Framework |
|---|---|---|
| `domain/`, `services/` | pytest with fake ports | MUST NOT import Streamlit or FastAPI |
| `adapters/` | pytest against a real SQLite file in `tmp_path`, schema from `alembic upgrade head` | no |
| `web/` — Track B | `TestClient` on `create_app(settings)` | yes |
| `ui/` — Track A | `AppTest.from_function`, fake services passed as arguments | yes |
| Built image | smoke test (§5.3) | yes |

Most tests are in the first two rows. A route or page test proves the input reaches the
right use case and the result is shown — not the rule itself.

```toml
[tool.pytest.ini_options]
markers = ["integration: needs network or an external system"]
addopts = "-m 'not integration'"
```

- Tests construct `Settings` explicitly. A test that passes only because the developer's
  `.env` is present is a defect.
- Track B: dependency overrides are set per test and cleared after it. Assert status code,
  the text the user sees, and htmx response headers. Every htmx route is tested with and
  without `HX-Request`.
- Track B: one test renders user input containing `<script>` and asserts it comes back
  escaped.
- Access control, both tracks: unauthenticated access is refused; user A cannot read or
  change user B's record by guessing its ID; Track B rejects a state-changing request
  without a CSRF token.
- Track A: every `AppTest.run()` takes a timeout, and the test asserts `at.exception` is
  empty before anything else. Tests never contact an OIDC provider; the identity is an
  argument.
- `time.sleep()` to wait for anything is PROHIBITED.

---

## 7. Where agents go wrong

Check these first when reviewing a change here.

1. Tailwind v3 habits: `tailwind.config.js`, `@tailwind base;`, `npm install tailwindcss`.
   This archetype is v4 and the standalone binary.
2. htmx versions mixed: htmx 2 and 4 differ in event names, attribute inheritance, and
   whether error responses are swapped. Check the vendored file's major version before
   writing any `hx-` attribute or event listener.
3. A DB write or paid call at the top level of a Streamlit script, repeated on every rerun.
4. User data in `st.cache_resource`, in `st.cache_data` without the user in its arguments, or
   in a module global — one user sees another's data.
5. `async def` handlers calling the sync `Session` or a sync HTTP client.
6. A SQLite file or uploads on Render's free disk; Render free Postgres as the only copy of
   the data.
7. Logic in route handlers, pages or templates; a hidden button treated as authorization.
8. `|safe`, `Markup`, or an f-string building HTML from user input.
9. An autogenerated Alembic revision applied unread; `create_all()` at startup.
10. "The deploy command succeeded, so it works." Check `/healthz` for the version.
11. Reaching for npm, React, or a CDN script to escape an htmx or Streamlit limit. That is a
    change of archetype, which needs an ADR, not a workaround.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes
- [ ] Track A: no effect reachable by a rerun outside a callback; no user data in a shared
      cache or global. Track B: no blocking call in `async def`; no `GET` that writes
- [ ] Tests construct `Settings` explicitly and run offline

Schema change:

- [ ] New Alembic revision appended and read by hand, none edited; upgraded from the
      previous head on a database with rows; `alembic check` clean; a failing migration
      leaves the revision and rows unchanged and the backup opens

Auth, session, or access change:

- [ ] Unauthenticated, cross-user and CSRF tests (§6) pass

Dependencies, `Dockerfile`, static assets, or composition root change:

- [ ] Image builds and the smoke test (§5.3) passes; Track B stylesheet regenerated
