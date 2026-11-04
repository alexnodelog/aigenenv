---
name: web-api-release
description: Set up the two toolchains, generate the API contract, run development, build the single image, run the smoke test, configure CI, and deploy a web-api app (FastAPI serving a Next.js static export). Use when creating or changing the Dockerfile, next.config.ts, the OpenAPI export, CI workflows, or when cutting a release.
---

# web-api-release (draft)

Raw procedures moved out of `rules/web-api.md`. The rules they serve stay in that file
(§2, §4, §5). This draft leaves `_drafts/` only when the Unverified list at the end is
empty.

## 1. Toolchain setup (rule §2)

- Install pnpm standalone first; `pnpm install` in `frontend/` then installs the Node
  version in `devEngines.runtime`.
- `cd frontend && pnpm exec playwright install chromium`, once.
- PostgreSQL projects: install `pg_dump` / `psql` with a major version at or above the
  server's.

## 2. API contract (rule §5.2)

```
uv run python -m <app>.api.export_openapi > openapi.json
cd frontend && pnpm exec openapi-typescript ../openapi.json -o src/api/schema.d.ts
```

`export_openapi` builds the app with test placeholder settings and prints
`app.openapi()`; it opens no connection.

`src/api/client.ts` creates one `openapi-fetch` client with base URL `/` and sets the
custom forgery-defence header (rule §4.1) on every request.

## 3. Development (rule §5.2)

Two terminals:

```
uv run --env-file .env uvicorn <app>.main:create_app --factory --reload --port 8000
cd frontend && pnpm dev
```

`next.config.ts` sets `output: "export"` only for `next build`, and returns
`rewrites()` forwarding `/api/:path*` and `/healthz` to `http://localhost:8000` only in
development. The backend runs without `FRONTEND_DIR` in development and serves the API
only.

## 4. Static serving (rule §4.2)

In `create_app`, in this order:

1. API routers under `/api/v1`, and `/healthz`.
2. A catch-all route `/api/{path:path}` returning a JSON 404.
3. If `FRONTEND_DIR` is set: `app.mount("/", StaticFiles(directory=..., html=True))`.
   With `html=True`, Starlette serves `index.html` for a directory, redirects a directory
   URL without a trailing slash to the one with it, and serves `404.html` with status 404
   on a miss.

A middleware sets the security headers of rule §4.4 on every response, and the cache
headers of rule §4.2 by path: `/_next/static/` → `public, max-age=31536000, immutable`;
`text/html` → `no-cache`.

## 5. Image (rule §5.1, §5.2)

```
docker build -t <app>:<version> .
```

- Node stage: base image by digest, install the pinned pnpm, `pnpm install
  --frozen-lockfile`, `pnpm build` → `frontend/out/`.
- Python stage: base image by digest, copy the pinned uv binary, `uv sync --locked
  --no-dev`, copy `frontend/out/` to `FRONTEND_DIR`, non-root user.
- Start: `python -m <app>.adapters.migrate && uvicorn <app>.main:create_app --factory
  --host 0.0.0.0 --port $PORT --forwarded-allow-ips <host proxy>`.

## 6. Smoke test (rule §5.3)

Start the image with a temp `DATA_DIR`, an empty database and placeholder secrets → poll
`/healthz` → run the checks listed in rule §5.3 → stop the container, timing the stop.

## 7. Test configuration

```toml
[tool.pytest.ini_options]
markers = ["integration: needs network or an external system"]
addopts = "-m 'not integration'"
```

`frontend/playwright.config.ts` `webServer`: run `pnpm build`, then start the backend with
`FRONTEND_DIR=frontend/out`, `DATA_DIR` and `DATABASE_URL` in a fresh temp dir, and wait
for `/healthz`.

## 8. CI (rule §5.5)

- Push / PR job, in order: `uv sync --locked` → Ruff → mypy → pytest → `alembic check` →
  fail if a file under `migrations/versions/` that exists on `main` was modified or
  deleted → PostgreSQL service container: adapter tests + `alembic upgrade head` →
  regenerate the contract (§2), `git diff --exit-code` → `pnpm install --frozen-lockfile`
  → Biome → `pnpm build` → Vitest → Playwright → build image, smoke test.
- Tag job: all of the above → check the tag equals the `pyproject.toml` version → if
  anything under `migrations/versions/` differs from the previous tag, stop → trigger the
  host's deploy hook → poll `/healthz` for the version, allowing two minutes for a cold
  start → `GET /`.

## 9. Deploy carrying a migration (rule §4.3, §5.4)

PostgreSQL only. On SQLite, `adapters/migrate.py` backs the file up itself at start.

1. Record the row counts of the main tables, then
   `pg_dump --format=custom --no-owner "$DIRECT_URL" > backup-<tag>.dump`. On Neon use the
   **direct** (not pooled) URL, held on the human's machine only.
2. Restore into an empty database (a Neon branch or local PostgreSQL):
   `pg_restore --no-owner --exit-on-error -d "$EMPTY_URL" backup-<tag>.dump`; the restored
   row counts match the recorded ones. If they differ, users wrote during the dump: suspend
   the service on the host, and repeat from step 1.
3. Deploy the tag.
4. Confirm the version and `/`.
5. The dump holds personal data and password hashes. Keep it encrypted, or delete it once
   the release is confirmed.

## 10. Scheduled backup (rule §4.3)

Only for an app holding data its users cannot recreate.

- PostgreSQL: GitHub Actions workflow on a schedule. Install the PostgreSQL client at or
  above the server's major; `pg_dump` with the direct URL from the secret
  `BACKUP_DATABASE_URL`, set on the backup job's environment only.
- SQLite on a VM disk: CI cannot reach it. The method is an ADR.
- Either way: encrypt the dump with the public key named in the ADR (the private key is
  never in CI or in the store), upload it to the chosen off-host store, fail the job on any
  non-zero exit. Confirm once that a failed run notifies the human, and restore an
  encrypted dump once end to end.

## 11. Rollback (rule §5.4)

1. Check whether `migrations/versions/` changed between the previous tag and the deployed
   one.
2. No change: redeploy the previous tag through the host.
3. Changed: do not redeploy yet — the old code may not run on the new schema. Decide with
   the human: fix forward with a new revision, or restore the pre-deploy dump and then
   redeploy the previous tag. Restoring discards every write made since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. After rollback, repeat the post-deploy checks of §9.

## Unverified — check before finalizing

- Next.js 16: `rewrites()` returned only in development while `output: "export"` is set
  only for `next build` — that `next dev` honours it and `next build` emits no warning.
- `next.config.ts` phase detection: `PHASE_DEVELOPMENT_SERVER` vs `NODE_ENV`.
- FastAPI: a catch-all `/api/{path:path}` route registered before the `/` static mount
  takes precedence for every method.
- Static export of `not-found.tsx` lands as `out/404.html` with `trailingSlash: true`.
- Render: which forwarded header carries the client address, and the value for
  `--forwarded-allow-ips`.
- Render deploy hook mechanics, and how to turn off auto-deploy.
- `pg_dump` / `pg_restore` flags for Neon.
- Dockerfile pattern for copying a pinned uv binary and installing a pinned pnpm in the
  Node stage.
- GitHub's failure notification for scheduled workflows: who receives it.
