---
name: web-api-release
description: Set up the two toolchains, generate the API contract, configure next.config.ts for development and static export, serve the export from FastAPI, build the single image, run the smoke test, configure CI, deploy, back up and roll back a web-api app (FastAPI serving a Next.js static export) on Render with Neon. Use when creating a web-api project, when editing frontend/next.config.ts, the OpenAPI export, src/api/client.ts, the static mount in create_app, the Dockerfile, CI or backup workflows, when a release carries a file under migrations/versions/, when a deploy or a post-deploy check fails, or when rolling back.
---

# web-api-release

Procedures for `rules/web-api.md` §2, §4 and §5. `<app>` is the Python package under
`src/`; the Next.js app lives in `frontend/`. The pnpm version is the one in
`frontend/package.json` `packageManager`. Shell steps that pipe one command into another
run with `set -o pipefail`; without it a failed `pg_dump` piped into an encrypter exits 0
and leaves a useless file that looks like a backup.

## 1. Toolchain setup (rule §2)

1. Install pnpm standalone (`curl -fsSL https://get.pnpm.io | PNPM_VERSION=<v> sh -`, or on
   Windows `$env:PNPM_VERSION='<v>'; iwr https://get.pnpm.io/ps1 -useb | iex`). In
   `frontend/`, `pnpm install` installs the Node of `devEngines.runtime`.
2. `cd frontend && pnpm exec playwright install chromium`, once.
3. PostgreSQL projects: `pg_dump`, `pg_restore`, `psql` with a major version at or above
   the server's.
4. Apps whose data users cannot recreate: age, as in §10.

## 2. Frontend skeleton

```
pnpm create next-app@<exact version> frontend --ts --app --src-dir --tailwind --biome --use-pnpm --import-alias "@/*" --no-agents-md --no-agent-feedback --disable-git
```

- The generator writes `cacheComponents: true` and `partialPrefetching: true`; §3 replaces
  `next.config.ts` without them. They have no place in a static export.
- Keep the generated Tailwind setup (`@import "tailwindcss";` in `globals.css`, the
  `@tailwindcss/turbopack` loader).
- Add `devEngines.runtime` (exact LTS, `"onFail": "download"`), pin the `^` ranges to exact
  versions, and in `frontend/pnpm-workspace.yaml` deny the install scripts
  `ERR_PNPM_IGNORED_BUILDS` names unless the app fails without them:

  ```yaml
  allowBuilds:
    sharp: false
    unrs-resolver: false
  ```

- `pnpm add -E openapi-fetch @tanstack/react-query` and
  `pnpm add -D -E openapi-typescript shadcn`, then `pnpm exec shadcn init --base radix`.
- The generated layout loads Google fonts through `next/font/google` at build time. On a
  network that blocks it, use `next/font/local`.

## 3. `frontend/next.config.ts`

The export and the development forwarding must never be in the same configuration:
`output: "export"` is set only for builds, and the forwarding exists only for the
development server.

```ts
import type { NextConfig } from "next";
import { PHASE_DEVELOPMENT_SERVER } from "next/constants";

const BACKEND = "http://localhost:8000";

export default function config(phase: string): NextConfig {
  const base: NextConfig = {
    images: { unoptimized: true },
    trailingSlash: true,
    turbopack: {
      rules: { "*.css": { loaders: ["@tailwindcss/turbopack"], as: "*.css" } },
    },
  };
  if (phase === PHASE_DEVELOPMENT_SERVER) {
    return {
      ...base,
      skipTrailingSlashRedirect: true,
      async rewrites() {
        return [
          { source: "/api/:path*", destination: `${BACKEND}/api/:path*` },
          { source: "/healthz", destination: `${BACKEND}/healthz` },
        ];
      },
    };
  }
  return { ...base, output: "export" };
}
```

- `trailingSlash: true` makes the export write `items/index.html`, which the backend's
  static mount serves at `/items/`. `out/404.html` comes from `src/app/not-found.tsx`.
- `skipTrailingSlashRedirect` in development only: without it the development server
  answers every `/api/...` call with a 308 to the same path plus `/` before forwarding it.
- Phase is decided by `PHASE_DEVELOPMENT_SERVER`, never by `NODE_ENV`.

## 4. API contract (rule §5.2)

```
uv run python -m <app>.api.export_openapi > openapi.json
cd frontend && pnpm exec openapi-typescript ../openapi.json -o src/api/schema.d.ts
```

`export_openapi` builds the app with test placeholder settings and prints
`json.dumps(app.openapi())`; it opens no connection.

`frontend/src/api/client.ts` creates the one client, base URL `/`, with the custom
forgery-defence header of rule §4.1 on every request:

```ts
import createClient from "openapi-fetch";
import type { paths } from "./schema";

export const api = createClient<paths>({ baseUrl: "/", headers: { "<header name>": "<value>" } });
```

Pages read data through TanStack Query v5 hooks in `src/api/` that call `api`.

## 5. Development

Two terminals:

```
uv run --env-file .env uvicorn <app>.main:create_app --factory --reload --port 8000
cd frontend && pnpm dev
```

The backend runs without `FRONTEND_DIR` in development and serves the API only. Open the
frontend on the development server's port; `/api/...` and `/healthz` reach the backend
through §3's forwarding.

## 6. Static serving and client address (rule §4.1, §4.2, §4.4)

In `create_app`, in this order:

1. API routers under `/api/v1`, and `/healthz`.
2. A catch-all for every method, answering a JSON 404:

   ```python
   @app.api_route("/api/{path:path}", methods=["GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS"])
   def api_not_found(path: str) -> JSONResponse:
       return JSONResponse({"detail": "Not Found"}, status_code=404)
   ```

3. If `FRONTEND_DIR` is set: `app.mount("/", StaticFiles(directory=..., html=True))`.
   Routes registered before the mount win for every method. The mount serves
   `index.html` for a directory, answers `/items` with a 307 to `/items/`, and serves
   `404.html` with status 404 for any other path.

A middleware sets the security headers of rule §4.4 on every response, and the cache
headers of rule §4.2 by path: `/_next/static/` → `public, max-age=31536000, immutable`;
`text/html` → `no-cache`.

Request body limit (rule §4.1): `src/<app>/web/body_limit.py`, a plain ASGI middleware that
counts bytes as they arrive, with or without `Content-Length`, and answers 413 as soon as
the count passes 1,048,576. Register it with `app.add_middleware(BodySizeLimit)`.

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

The tests of rule §6 run it against Uvicorn as well as `TestClient`;
`curl -H 'Transfer-Encoding: chunked' --data-binary @file` sends a body without
`Content-Length`.

Behind Render's proxy:

- Uvicorn runs with `--forwarded-allow-ips '*'`, so the `https` scheme reaches `Secure`
  cookies and redirects. The container is reachable only through Render's proxy.
- With `'*'`, Uvicorn takes the client address from the left end of `X-Forwarded-For`,
  which the client can write, and Render keeps a client-sent `X-Forwarded-For`. The
  sign-in counters therefore MUST NOT use `request.client`. One function in `web/` reads
  the address from the header named in `Settings`, `true-client-ip` on Render, set by the
  CDN in front of it.
- Walking skeleton check: wrong-password sign-ins from two networks show two addresses;
  sign-ins carrying a made-up `X-Forwarded-For` and a made-up `True-Client-IP` do not
  change the counted address. If either fails, stop: no header is trusted, and the human
  decides.

## 7. Image (rule §5.1, §5.2)

```dockerfile
FROM node:<major>-<variant>@sha256:<digest> AS frontend
RUN npm install -g pnpm@<packageManager version>
WORKDIR /frontend
COPY frontend/package.json frontend/pnpm-lock.yaml frontend/pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile
COPY frontend/ ./
RUN NEXT_TELEMETRY_DISABLED=1 pnpm build

FROM python:<minor from .python-version>-slim@sha256:<digest>
COPY --from=ghcr.io/astral-sh/uv:<uv version>@sha256:<digest> /uv /uvx /bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev --no-install-project
COPY . .
RUN uv sync --locked --no-dev
COPY --from=frontend /frontend/out /app/frontend-out
RUN useradd --create-home --uid 10001 app
USER app
ENV PATH="/app/.venv/bin:$PATH" FRONTEND_DIR=/app/frontend-out
CMD ["sh", "-c", "python -m <app>.adapters.migrate && uvicorn <app>.main:create_app --factory --host 0.0.0.0 --port $PORT --forwarded-allow-ips '*'"]
```

- The Node stage installs pnpm with npm only to bootstrap the exact `packageManager`
  version inside the image; the project itself never runs npm.
- `.dockerignore` lists `frontend/node_modules`, `frontend/.next`, `frontend/out`, `.venv`,
  `.env*`.

## 8. Smoke test and tests (rule §5.3, §6)

Smoke test: start the image with `-e PORT=8000`, a fresh temp `DATA_DIR`, an empty
database and placeholder secrets → poll `/healthz` every second up to 60 s → run the
checks of rule §5.3 → `docker stop` within 10 s, timed.

```toml
[tool.pytest.ini_options]
markers = ["integration: needs network or an external system"]
addopts = "-m 'not integration'"
```

`frontend/playwright.config.ts` `webServer`: run `pnpm build`, then start the backend with
`FRONTEND_DIR=frontend/out`, `DATA_DIR` and `DATABASE_URL` in a fresh temp dir, and wait
for `/healthz`.

## 9. CI (rule §5.5)

Every `uses:` is pinned to a full commit SHA. pnpm comes from `pnpm/setup` (v3,
`install: false`, run in `frontend/`). Steps that pipe use `shell: bash`, which GitHub runs
with `-eo pipefail`.

- Push / PR, in order: `uv sync --locked` → Ruff → mypy → pytest → `alembic check` → fail
  if a file under `migrations/versions/` that exists on `main` was modified or deleted →
  PostgreSQL projects: a PostgreSQL service container, the adapter tests and
  `alembic upgrade head` against it → regenerate the contract (§4), `git diff
  --exit-code` → `pnpm install --frozen-lockfile` → `pnpm exec playwright install
  --with-deps chromium` → Biome → `pnpm build` → Vitest → Playwright → build the image →
  smoke test.
- Tag job: all of the above → fail unless the tag equals `v` + `pyproject.toml` `version`
  → then one of two paths:
  - Nothing under `migrations/versions/` differs from the previous tag: call the deploy
    hook for the tagged commit, with `RENDER_DEPLOY_HOOK_URL` set on this step only:
    `curl -fsS -X POST "$RENDER_DEPLOY_HOOK_URL&ref=$GITHUB_SHA"`. Then §14.
  - Something differs: end without calling the hook, and say that the release needs §11.

## 10. Keys for backups

Once, on the human's machine: install a tagged release of `FiloSottile/age`;
`age-keygen -o backup-key.txt` prints the public key (`age1…`). The file holds the private
key: keep it on that machine and one offline copy, never in the repository, CI or the
backup store. Record the release, its SHA-256 and the public key in `DECISIONS.md` (skill
`adr`). Losing the private key makes every backup unreadable.

## 11. Deploy carrying a migration (rule §4.3, §5.4)

PostgreSQL. On SQLite, `adapters/migrate.py` backs the file up itself at start.

Render refuses a deploy hook call while a service is suspended (409), and does not
document which commit it builds on resume. So the human migrates the database first,
while nothing runs; after that, any older commit Render starts finds a revision it does
not know and refuses to start (rule §4.3).

On the human's machine, at the tag, with `DIRECT_URL` the database owner's **direct** (not
pooled) Neon URL, held on this machine only:

1. `git checkout v<version>` and `uv sync --locked`.
2. Suspend the service on Render, and stop anything else that writes to the database.
   Check that the site no longer answers.
3. Dump into encryption:

   ```bash
   set -o pipefail
   pg_dump --format=custom --no-owner "$DIRECT_URL" | age -r "<public key>" > backup-v<version>.dump.age
   ```

4. Restore into an empty database (a new Neon branch, or a local PostgreSQL):

   ```bash
   set -o pipefail
   age -d -i backup-key.txt backup-v<version>.dump.age | pg_restore --no-owner --no-acl --exit-on-error -d "$EMPTY_URL"
   ```

   A non-zero exit stops the release: resume the service; it comes back on the old
   version with the database unchanged.
5. Migrate production: `DATABASE_URL="$DIRECT_URL" uv run python -m <app>.adapters.migrate`.
   On failure, check `alembic current` against the previous head. Unchanged: resume and
   report. Changed: stop; restoring the dump is the human's decision (§13).
6. Resume the service on Render.
7. Call the deploy hook with `&ref=<tagged commit SHA>`. 200 means the deploy started; 409
   means the service is still suspended.
8. §14.
9. Keep only the encrypted dump, under the retention in `DECISIONS.md`. Delete the
   restore-test database or branch.

## 12. Scheduled backup (rule §4.3)

Only for an app holding data its users cannot recreate. A GitHub Actions workflow on a
`schedule`; install the PostgreSQL client at or above the server's major, and the age
release checked against its SHA-256. `BACKUP_DATABASE_URL`, the direct URL, is set on the
backup job's environment only. CI holds only the public key, so it checks the dump before
encrypting it, in steps with `shell: bash`:

1. `pg_dump --format=custom --no-owner "$BACKUP_DATABASE_URL" > "$RUNNER_TEMP/db.dump"`
2. `pg_restore --no-owner --no-acl --exit-on-error` of that file into a PostgreSQL service
   container; fail unless its `alembic_version` equals production's and every table the
   models define exists.
3. `age -r "<public key>" -o "$RUNNER_TEMP/db.dump.age" "$RUNNER_TEMP/db.dump"`, then
   upload `db.dump.age` to the store the ADR names.
4. Delete `$RUNNER_TEMP/db.dump` in a step with `if: always()`.

- Failure notifications for a scheduled workflow go to the person who last changed its
  `cron` line, or who last re-enabled it. Record who that is in `DECISIONS.md`.
- In a public repository, GitHub disables scheduled workflows after 60 days without
  repository activity, and a disabled workflow sends no failure notice. Keep the backup
  workflow in a private repository, or record that the human checks the store for a
  recent dump at an interval no longer than the acceptable data loss. A commit or API
  call made only to keep the schedule alive is PROHIBITED.
- SQLite on a VM disk: CI cannot reach it. The method is an ADR.
- Once, before the app holds real data: make a run fail on purpose and confirm the
  notification arrives; restore an encrypted dump end to end with the private key.

## 13. Rollback (rule §5.4)

1. Check whether `migrations/versions/` changed between the previous tag and the deployed
   one.
2. No change: call the deploy hook with `&ref=<previous tag's commit SHA>`.
3. Changed: do not redeploy yet; the old code refuses to start on the new revision. Decide
   with the human: fix forward with a new revision, or restore the pre-deploy dump and
   then redeploy the previous tag. Restoring discards every write since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. §14.

## 14. Post-deploy checks (rule §5.3, §5.4)

Allow two minutes for a cold start, then:

1. `/healthz` answers 200 with the tag's version and the DB revision at head.
2. `/` answers 200 with HTML.
3. The human signs in and opens one page that reads data. CI runs 1 and 2 only.

A failed check stops everything: report it with the options of §13. No retry and no
redeploy.
