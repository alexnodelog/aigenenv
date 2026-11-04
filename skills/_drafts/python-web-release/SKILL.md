---
name: python-web-release
description: Set up the build toolchain, build the image, run the smoke test, configure CI, and deploy a web-python app (Streamlit or FastAPI). Use when creating or changing the Dockerfile, CI workflows, Tailwind/htmx pins, or when cutting a release.
---

# python-web-release (draft)

Raw procedures moved out of `rules/web-python.md`. The rules they serve stay in that file
(§2, §4.3, §5). This draft leaves `_drafts/` only when the Unverified list at the end is
empty.

## 1. Toolchain setup (rule §2)

- Tailwind standalone CLI: download a tagged v4 release, never `latest`. Record version and
  SHA-256 in `DECISIONS.md`.
- htmx: download a 2.x release, save as `src/<app>/web/static/vendor/htmx.min.js`. Record
  version and SHA-256 in `DECISIONS.md`.
- PostgreSQL projects: install `pg_dump` / `psql` with a major version at or above the
  server's.

## 2. Stylesheet (Track B, rule §5.2)

```
tailwindcss -i assets/app.css -o src/<app>/web/static/app.css --minify
```

## 3. Image (rule §5.1, §5.2)

```
docker build -t <app>:<version> .
```

Start command, after `adapters/migrate.py`:

- Track B: `uvicorn <app>.main:create_app --factory --host 0.0.0.0 --port $PORT --forwarded-allow-ips <host proxy>`
- Track A: `streamlit run src/<app>/streamlit_app.py --server.headless true --server.address 0.0.0.0 --server.port $PORT`

## 4. Smoke test (rule §5.3)

Start the image with a temp `DATA_DIR`, an empty database and placeholder secrets → poll
`/healthz` (Track B) or `/_stcore/health` (Track A; 200 when ready, 503 otherwise) → `GET /`
→ Track A: check DB revision → stop the container.

## 5. pytest configuration

```toml
[tool.pytest.ini_options]
markers = ["integration: needs network or an external system"]
addopts = "-m 'not integration'"
```

## 6. CI (rule §5.5)

- Push / PR job, in order: `uv sync --locked` → Ruff → mypy → pytest → `alembic check` →
  fail if a file under `migrations/versions/` that exists on `main` was modified or deleted →
  PostgreSQL service container: adapter tests + `alembic upgrade head` → Track B:
  regenerate stylesheet, `git diff --exit-code` → build image, smoke test.
- Tag job: all of the above → check tag equals `pyproject.toml` version → if an Alembic
  anything under `migrations/versions/` differs from the previous tag, stop → trigger the host's deploy hook → poll
  `/healthz` for the version, allowing two minutes for a cold start → `GET /`.

## 7. Deploy carrying a migration (rule §4.3, §5.4)

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
5. The dump holds personal data and any password hashes. Keep it encrypted, or delete it
   once the release is confirmed.

## 8. Scheduled backup (rule §4.3)

Only for an app holding data its users cannot recreate.

- PostgreSQL: GitHub Actions workflow on a schedule. Install the PostgreSQL client at or
  above the server's major; `pg_dump` with the direct URL from the secret
  `BACKUP_DATABASE_URL`, set on the backup job's environment only, never on the app or
  another job.
- SQLite: the database lives on the host's disk, which CI cannot reach. The method is an
  ADR.
- Either way: encrypt the dump with the public key named in the ADR (the private key is
  never in CI or in the store), upload it to the chosen off-host store, fail the job on any
  non-zero exit. Confirm once that a failed run notifies the human, and restore an
  encrypted dump once end to end. Store, retention and key location are an ADR.

## 9. Rollback (rule §5.4)

1. Check first whether `migrations/versions/` changed between the previous tag and the
   deployed one.
2. No change: redeploy the previous tag through the host.
3. Changed: do not redeploy yet — the old code may not run on the new schema. Decide with
   the human: fix forward with a new revision, or restore the pre-deploy dump and then
   redeploy the previous tag. Restoring discards every write made since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. After rollback, repeat the post-deploy checks of §7.

## Unverified — check before finalizing

- Tailwind CLI release asset names and download URL per OS.
- Exact `streamlit run` flags for address and port in the current version.
- Render deploy hook mechanics, and how to turn off auto-deploy.
- `pg_dump` / `pg_restore` flags for Neon.
- GitHub's failure notification for scheduled workflows: who receives it.
- How a SQLite app on a host's disk is backed up off the host.
- Dockerfile pattern for copying a pinned uv binary.
