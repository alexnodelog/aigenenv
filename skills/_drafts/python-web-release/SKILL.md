---
name: python-web-release
description: Set up the toolchain (Tailwind standalone CLI, vendored htmx), write the Dockerfile, the body-size limit and the smoke test, configure CI, deploy, back up, and roll back a web-python app (Streamlit or FastAPI + htmx) on Render with Neon. Use when creating or changing the Dockerfile, assets/app.css, static/vendor/htmx.min.js, the request body limit, tests/smoke/, CI or backup workflows, when a release carries a file under migrations/versions/, when a deploy or a post-deploy check fails, or when rolling back.
---

# python-web-release

Procedures for `rules/web-python.md` §2, §4.3 and §5. `<app>` is the package name under
`src/`. Shell steps that pipe one command into another run with `set -o pipefail`; without
it a failed `pg_dump` piped into an encrypter exits 0 and leaves a useless file that looks
like a backup.

## 1. Toolchain setup (rule §2)

### 1.1 Tailwind standalone CLI (Track B)

1. Pick a tagged v4 release of `tailwindlabs/tailwindcss`, never `latest`. Assets:
   `tailwindcss-linux-x64`, `tailwindcss-linux-arm64`, `tailwindcss-macos-arm64`,
   `tailwindcss-macos-x64`, `tailwindcss-windows-x64.exe`, and `sha256sums.txt`.
2. Download from `https://github.com/tailwindlabs/tailwindcss/releases/download/<tag>/<asset>`.
3. Compare its SHA-256 with the line in that release's `sha256sums.txt`. Record the tag
   and the SHA-256 in `DECISIONS.md` (skill `adr`).
4. Check: `tailwindcss --help` prints the release version.

### 1.2 htmx (Track B)

1. Take `dist/htmx.min.js` from the npm package `htmx.org` at an exact 2.x version (for
   example from `npm pack htmx.org@<version>`, or the release on GitHub).
2. Save it as `src/<app>/web/static/vendor/htmx.min.js`. The filename carries no version.
3. Record the version and its SHA-256 in `DECISIONS.md`.

### 1.3 PostgreSQL client (PostgreSQL projects)

Install `pg_dump`, `pg_restore`, `psql` with a major version at or above the server's.
Check: `pg_dump --version`, and `psql "$DIRECT_URL" -c 'SHOW server_version'`.

## 2. Stylesheet (Track B, rule §5.2)

`assets/app.css`, with `@source` paths relative to this file:

```css
@import "tailwindcss";
@source "../src/<app>/web/templates";

.htmx-indicator { opacity: 0; }
.htmx-request .htmx-indicator,
.htmx-request.htmx-indicator { opacity: 1; transition: opacity 200ms ease-in; }
```

The indicator rules replace the ones htmx would inject inline, which the CSP blocks.

```
tailwindcss -i assets/app.css -o src/<app>/web/static/app.css --minify
```

Commit the output with the template or `app.css` change that caused it.

## 3. Page head (Track B, rule §4.4)

Every page:

```html
<meta name="htmx-config" content='{"allowEval":false,"includeIndicatorStyles":false}'>
<script src="/static/vendor/htmx.min.js"></script>
<link rel="stylesheet" href="/static/app.css">
...
<body hx-headers='{"X-CSRF-Token": "{{ csrf_token }}"}'>
```

With this meta tag, htmx 2 runs under the rule's CSP with no violation and still sends
`hx-headers`. Without it, htmx injects an inline style and the browser reports a
`style-src` violation.

## 4. Request body limit (Track B, rule §4.1)

`src/<app>/web/body_limit.py`, a plain ASGI middleware. It counts bytes as they arrive,
with or without `Content-Length`, and answers 413 as soon as the count passes the limit:

```python
from starlette.types import ASGIApp, Message, Receive, Scope, Send

MAX_BODY_BYTES = 1_048_576


class _BodyTooLarge(Exception):
    pass


class BodySizeLimit:
    def __init__(self, app: ASGIApp, max_bytes: int = MAX_BODY_BYTES) -> None:
        self.app = app
        self.max_bytes = max_bytes

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        for name, value in scope["headers"]:
            if name == b"content-length" and value.isdigit() and int(value) > self.max_bytes:
                await self._reject(send)
                return
        received = 0
        response_started = False

        async def counting_receive() -> Message:
            nonlocal received
            message = await receive()
            if message["type"] == "http.request":
                received += len(message.get("body", b""))
                if received > self.max_bytes:
                    raise _BodyTooLarge
            return message

        async def tracking_send(message: Message) -> None:
            nonlocal response_started
            if message["type"] == "http.response.start":
                response_started = True
            await send(message)

        try:
            await self.app(scope, counting_receive, tracking_send)
        except _BodyTooLarge:
            if response_started:
                raise
            await self._reject(send)

    async def _reject(self, send: Send) -> None:
        await send({"type": "http.response.start", "status": 413,
                    "headers": [(b"content-type", b"text/plain"), (b"connection", b"close")]})
        await send({"type": "http.response.body", "body": b"Request body too large"})
```

Register it in `create_app` with `app.add_middleware(BodySizeLimit)`. A route the ADR
allows more passes its own `max_bytes` through a separate mount. The four tests of rule
§6 run against Uvicorn as well as `TestClient`; `curl -H 'Transfer-Encoding: chunked'
--data-binary @file` sends a body without `Content-Length`.

## 5. Image (rule §5.1, §5.2)

```dockerfile
FROM python:<minor from .python-version>-slim@sha256:<digest>
COPY --from=ghcr.io/astral-sh/uv:<uv version>@sha256:<digest> /uv /uvx /bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project
COPY . .
RUN uv sync --locked --no-dev
RUN useradd --create-home --uid 10001 app && mkdir -p /data && chown app /data
USER app
ENV PATH="/app/.venv/bin:$PATH" DATA_DIR=/data
CMD ["sh", "-c", "python -m <app>.adapters.migrate && <start command>"]
```

- Find a digest with `docker buildx imagetools inspect <image>:<tag>`.
- Start command, Track B:
  `uvicorn <app>.main:create_app --factory --host 0.0.0.0 --port $PORT --forwarded-allow-ips <host proxy>`
- Start command, Track A:
  `streamlit run src/<app>/streamlit_app.py --server.headless true --server.address 0.0.0.0 --server.port $PORT`
- `docker build -t <app>:<version> .`

## 6. Smoke test (rule §5.3)

1. Start the image with `-e PORT=8000`, a fresh temp dir mounted as `DATA_DIR`, a new
   SQLite file (or the CI service container's empty database) and placeholder secrets.
2. Poll health every second, up to 60 s: Track B `GET /healthz`; Track A
   `GET /_stcore/health`, which answers `ok` with 200 when ready and 503 otherwise.
3. `GET /` within 10 s.
4. Track B: every header of rule §4.4 with its required value. Track A: query the database
   for its Alembic revision and compare with `alembic heads`.
5. `docker stop` within 10 s. Any timeout fails the test.

## 7. Test configuration

```toml
[tool.pytest.ini_options]
markers = ["integration: needs network or an external system"]
addopts = "-m 'not integration'"
```

Track A page test:

```python
from streamlit.testing.v1 import AppTest


def page(service, user):
    import streamlit as st          # imports inside: only the source is run
    from <app>.ui.pages import orders
    orders.render(service, user)


def test_orders_page():
    at = AppTest.from_function(page, args=(FakeOrderService(), "user@example.com"))
    at.run(timeout=10)
    assert not at.exception
```

Objects passed through `args` and `kwargs` reach the function as they are; nothing else
from the test module does.

## 8. CI (rule §5.5)

Every `uses:` is pinned to a full commit SHA. Steps that pipe use `shell: bash`, which on
GitHub runners runs with `-eo pipefail`; a step with no `shell:` runs without pipefail.

- Push / PR job, in order: `uv sync --locked` → Ruff → mypy → pytest → `alembic check` →
  fail if a file under `migrations/versions/` that exists on `main` was modified or
  deleted → PostgreSQL projects: a PostgreSQL service container, the adapter tests and
  `alembic upgrade head` against it → Track B: regenerate the stylesheet (§2) and
  `git diff --exit-code` → build the image → smoke test (§6).
- Tag job: all of the above → fail unless the tag equals `v` + `pyproject.toml` `version`
  → then one of two paths:
  - Nothing under `migrations/versions/` differs from the previous tag: call the deploy
    hook for the tagged commit, with `RENDER_DEPLOY_HOOK_URL` set on this step only:
    `curl -fsS -X POST "$RENDER_DEPLOY_HOOK_URL&ref=$GITHUB_SHA"`. 200 means the deploy
    started. Then §12.
  - Something differs: end the job without calling the hook, and say that the release
    needs §9.
- The deploy hook URL is a secret: anyone holding it can deploy.

## 9. Deploy carrying a migration (rule §4.3, §5.4)

PostgreSQL. On SQLite, `adapters/migrate.py` backs the file up itself at start.

Render refuses a deploy hook call while a service is suspended (409), and does not
document which commit it builds on resume. So the human migrates the database first,
while nothing runs; after that, any older commit Render starts finds a revision it does
not know and refuses to start (rule §4.3).

On the human's machine, at the tag, with `DIRECT_URL` the database owner's **direct**
(not pooled) Neon URL, held on this machine only:

1. `git checkout v<version>` and `uv sync --locked`.
2. Suspend the service on Render, and stop anything else that writes to the database.
   Check that the site no longer answers.
3. Dump into encryption, so no plaintext file is written. The ADR names the encryption
   tool and key; with `age`:

   ```bash
   set -o pipefail
   pg_dump --format=custom --no-owner "$DIRECT_URL" | age -r "<backup public key>" > backup-v<version>.dump.age
   ```

4. Restore into an empty database (a new Neon branch, or a local PostgreSQL):

   ```bash
   set -o pipefail
   age -d -i <private key file> backup-v<version>.dump.age | pg_restore --no-owner --no-acl --exit-on-error -d "$EMPTY_URL"
   ```

   A non-zero exit stops the release: resume the service; it comes back on the old
   version with the database unchanged.
5. Migrate production: `DATABASE_URL="$DIRECT_URL" uv run python -m <app>.adapters.migrate`.
   On failure, check `alembic current` against the previous head. Unchanged: resume and
   report. Changed: stop; restoring the dump is the human's decision (§11).
6. Resume the service on Render.
7. Call the deploy hook with `&ref=<tagged commit SHA>`. 200 means the deploy started;
   409 means the service is still suspended.
8. §12.
9. Keep only the encrypted dump, under the retention in `DECISIONS.md`. Delete the
   restore-test database or branch.

## 10. Scheduled backup (rule §4.3)

Only for an app holding data its users cannot recreate.

- PostgreSQL: a GitHub Actions workflow on a `schedule`. Install the PostgreSQL client at or
  above the server's major. `BACKUP_DATABASE_URL`, the direct URL, is set on the backup
  job's environment only.
- In the same job, with `shell: bash`: dump into the encryption of §9 step 3; restore the
  encrypted file into a PostgreSQL service container with `pg_restore --no-owner --no-acl
  --exit-on-error`; fail unless its `alembic_version` equals production's and every table
  the models define exists; upload the encrypted file to the store the ADR names. The
  private key is never in CI or in the store.
- Failure notifications for a scheduled workflow go to the person who last changed its
  `cron` line, or who last re-enabled it. That person is the human who must receive them;
  record who in `DECISIONS.md`.
- In a public repository, GitHub disables scheduled workflows after 60 days without
  repository activity, and a disabled workflow sends no failure notice. Keep the backup
  workflow in a private repository, or record in `DECISIONS.md` that the human checks the
  store for a recent dump at an interval no longer than the acceptable data loss. A commit
  or API call made only to keep the schedule alive is PROHIBITED, as any artificial
  activity is.
- SQLite on a VM disk: CI cannot reach it. The method is an ADR.
- Once, before the app holds real data: make a run fail on purpose and confirm the
  notification arrives; restore an encrypted dump end to end with the private key.

## 11. Rollback (rule §5.4)

1. Check whether `migrations/versions/` changed between the previous tag and the deployed
   one.
2. No change: call the deploy hook with `&ref=<previous tag's commit SHA>`.
3. Changed: do not redeploy yet; the old code refuses to start on the new revision. Decide
   with the human: fix forward with a new revision, or restore the pre-deploy dump and
   then redeploy the previous tag. Restoring discards every write since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. §12.

## 12. Post-deploy checks (rule §5.4, §5.5)

Allow two minutes for a cold start, then:

1. Track B: `/healthz` answers 200 with the tag's version and the DB revision at head.
   Track A: the human confirms on the Render dashboard that the live deploy is the tagged
   commit.
2. `/` answers 200.
3. The human signs in and opens one page that reads data. CI runs 1 and 2 only.

A failed check stops everything: report it with the rollback options of §11. No retry and
no redeploy.
