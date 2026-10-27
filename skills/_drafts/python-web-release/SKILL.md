---
name: python-web-release
description: Set up the build toolchain, build the image, run the smoke test, configure CI, and deploy a web-python app (Streamlit or FastAPI). Use when creating or changing the Dockerfile, CI workflows, Tailwind/htmx pins, or when cutting a release.
---

# python-web-release (draft)

Raw procedures moved out of `rules/web-python.md`. The rules they serve stay in that file
(§2, §4.3, §5). To be completed and verified in step 7.

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
  PostgreSQL service container: adapter tests + `alembic upgrade head` → Track B:
  regenerate stylesheet, `git diff --exit-code` → build image, smoke test.
- Tag job: all of the above → check tag equals `pyproject.toml` version → if an Alembic
  revision was added since the previous tag, stop → trigger the host's deploy hook → poll
  `/healthz` for the version, allowing two minutes for a cold start → `GET /`.

## 7. Deploy carrying a migration (rule §4.3, §5.4)

1. `pg_dump` the production database.
2. Restore the dump into an empty database; confirm it opens and has the expected tables.
3. Deploy the tag.
4. Confirm the version and `/`.

## Unverified — check before finalizing

- Tailwind CLI release asset names and download URL per OS.
- Exact `streamlit run` flags for address and port in the current version.
- Render deploy hook mechanics, and how to turn off auto-deploy.
- `pg_dump` / `pg_restore` flags for Neon.
- Dockerfile pattern for copying a pinned uv binary.
