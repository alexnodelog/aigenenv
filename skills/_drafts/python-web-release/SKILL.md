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
  PostgreSQL projects only: PostgreSQL service container, adapter tests +
  `alembic upgrade head` → Track B:
  regenerate stylesheet, `git diff --exit-code` → build image, smoke test.
- Tag job: all of the above → check the tag equals the `pyproject.toml` version → then
  one of two paths:
  - Nothing under `migrations/versions/` differs from the previous tag: trigger the host's
    deploy hook → run the post-deploy checks (§10).
  - Something differs: end the job without deploying. The deploy hook step does not run on
    this path. The human deploys by §7.

## 7. Deploy carrying a migration (rule §4.3, §5.4)

PostgreSQL only. On SQLite, `adapters/migrate.py` backs the file up itself at start.

1. Suspend the service on the host, and stop anything else that writes to the database.
   From here until step 4, nothing writes and no old instance is running.
2. Dump straight into encryption, so no plaintext file is written:
   `pg_dump --format=custom --no-owner "$DIRECT_URL" | <encrypt to the backup public key>
   > backup-<tag>.dump.enc`. On Neon use the **direct** (not pooled) URL, held on the
   human's machine only.
3. Restore into an empty database (a Neon branch or local PostgreSQL), decrypting into the
   pipe: `<decrypt> backup-<tag>.dump.enc | pg_restore --no-owner --exit-on-error -d
   "$EMPTY_URL"`. A failure stops the release; resume the service on the old version.
4. Deploy the tag so that the service comes back on the new version only.
5. Run the post-deploy checks (§10).
6. The dump holds personal data and any password hashes. Keep only the encrypted file,
   under the backup retention in `DECISIONS.md`; delete the restore-test database.

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
  non-zero exit. Store, retention and key location are an ADR.
- PostgreSQL: before encrypting, restore the dump into a PostgreSQL service container in
  the same job with `pg_restore --exit-on-error`, and check that its `alembic_version`
  matches the production revision and that every table the models define exists; either
  failing fails the job.
- Once, before the app holds real data: confirm that a failed run notifies the human, and
  restore an encrypted dump end to end, decrypting with the private key.

## 9. Rollback (rule §5.4)

1. Check first whether `migrations/versions/` changed between the previous tag and the
   deployed one.
2. No change: redeploy the previous tag through the host.
3. Changed: do not redeploy yet — the old code may not run on the new schema. Decide with
   the human: fix forward with a new revision, or restore the pre-deploy dump and then
   redeploy the previous tag. Restoring discards every write made since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. Run the post-deploy checks (§10).

## 10. Post-deploy checks (rule §5.4, §5.5)

Allow two minutes for a cold start, then:

1. Track B: `/healthz` answers 200 with the expected version and DB revision. Track A: the
   human confirms the deployed commit on the host's dashboard.
2. `/` answers 200.
3. The human signs in and opens one page that reads data. CI runs 1 and 2 only.

## Unverified — check before finalizing

- Tailwind CLI release asset names and download URL per OS.
- Exact `streamlit run` flags for address and port in the current version.
- Render deploy hook mechanics, and how to turn off auto-deploy.
- Render: how to deploy a tag to a suspended service so that it resumes on the new version
  without the old one serving first. Render feedback reports that resuming triggers a new
  build; which commit it builds is unknown. Until verified, no migration deploy is done
  on Render.
- `pg_restore` reading a custom-format dump from a pipe (no seekable file).
- Track B: the middleware or ASGI wrapper that counts body bytes as they arrive.
- htmx 2: `allowEval: false` and `includeIndicatorStyles: false` through the `htmx-config`
  meta tag leave no CSP violation on a page that uses `hx-headers`.
- `pg_dump` / `pg_restore` flags for Neon.
- GitHub's failure notification for scheduled workflows: who receives it.
- How a SQLite app on a host's disk is backed up off the host.
- Dockerfile pattern for copying a pinned uv binary.
