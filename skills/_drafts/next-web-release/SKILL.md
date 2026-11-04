---
name: next-web-release
description: Set up, build, deploy and roll back a web-ts project (Next.js standalone container on Render, Neon PostgreSQL, Better Auth, Drizzle). Use for first setup, the walking skeleton, a release, a migration deploy, a backup, or a rollback.
---

# next-web-release

Items marked **[verify]** were not confirmed against the installed CLI. This draft leaves
`_drafts/` only when none remain.

## 1. First setup on a developer machine

1. Install pnpm standalone. Confirm: `pnpm --version` matches `packageManager`.
2. `pnpm install` — pnpm downloads Node from `devEngines.runtime`. Confirm: `pnpm exec node --version`.
3. `pnpm exec playwright install chromium` (once; tests then run offline).
4. Copy `.env.example` to `.env.local`. Set `DATABASE_URL=pglite:./.data/dev` **[verify the
   URL form `client.ts` parses]**, `BETTER_AUTH_SECRET` from `openssl rand -base64 32`,
   `BETTER_AUTH_URL=http://localhost:3000`. `.data/` and `.env.local` are gitignored.
5. `pnpm db:migrate`, then `pnpm dev`. `db:migrate` runs `src/adapters/db/migrate.ts`
   against `DATABASE_URL` — the same entry the container runs at start. It applies only
   pending migrations and exits non-zero on failure. Script:
   `"db:migrate": "node --env-file-if-exists=.env.local src/adapters/db/migrate.ts"`
   **[verify: Node runs the `.ts` file by type stripping, so `migrate.ts` may import only
   packages and relative paths with `.ts` extensions — no `@/` aliases]**.

## 2. New project skeleton

1. `pnpm create next-app@<exact version> <name> --ts --app --src-dir --tailwind --use-pnpm
   --no-eslint` **[verify flag names on Next.js 16]**. Remove anything it adds that §1
   prohibits.
2. Add `packageManager`, `devEngines.runtime` (`onFail: "download"`), `pnpm-workspace.yaml`
   with `allowBuilds`.
3. `next.config.ts`: `output: "standalone"`, `headers()` with the §4.4 set, `redirects()`.
4. Biome: `pnpm add -D -E @biomejs/biome`, `pnpm exec biome init`.
5. shadcn/ui: `pnpm add -D -E shadcn`, `pnpm exec shadcn init --base radix` **[verify CLI v4
   flags]**.
6. Drizzle: `pnpm add drizzle-orm pg @electric-sql/pglite`, `pnpm add -D -E drizzle-kit
   @types/pg`. `drizzle.config.ts` points `schema` at `src/adapters/db/` and `out` at
   `drizzle/`, dialect `postgresql`.
7. Better Auth: `pnpm add better-auth`, `pnpm add -D -E auth` **[verify the CLI package
   name is `auth`]**. Configure `drizzleAdapter(db, { provider: "pg" })`,
   `emailAndPassword: { enabled: true }`, `rateLimit: { storage: "database" }`, the
   client-IP header for the host **[verify option path: `advanced.ipAddress`]**, and the
   `nextCookies()` plugin last. Mount `toNextJsHandler(auth)` at
   `src/app/api/auth/[...all]/route.ts`.
8. Generate: `pnpm exec auth generate --output src/adapters/db/auth-schema.ts`
   **[verify flag]**, then `pnpm exec drizzle-kit generate`. Read the SQL before committing.
9. Vitest: two projects — `unit` (`environment: "node"`, `tests/unit`, `tests/adapters`) and
   `browser` (`@vitest/browser-playwright`, Chromium, `tests/browser`, `vitest-browser-react`).
10. Playwright, after `pnpm build`: `webServer` runs `pnpm start` on the built app with a temp PGlite
    `DATABASE_URL` and placeholder secrets.

## 3. Dockerfile shape

Multi-stage. Base: `node:<major>-<variant>@sha256:<digest>`.

1. **deps** — install pnpm at the `packageManager` version; `pnpm install --frozen-lockfile`.
2. **build** — `NEXT_TELEMETRY_DISABLED=1 pnpm build` with no secrets in the environment.
3. **run** — copy `.next/standalone/`, then `public/` → `./public` and `.next/static/` →
   `./.next/static`; copy `drizzle/` and `src/adapters/db/migrate.ts`, and make sure Drizzle's migrator and
   the `pg` driver resolve from the run stage — standalone output keeps only the files
   the app itself traces **[verify which install or copy achieves this]**. The smoke test
   proves it: the image must migrate an empty database before health turns 200; `USER node`;
   `ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 HOSTNAME=0.0.0.0`;
   `CMD` runs migrate, then `node server.js`.

Check the image size and that `.env*` is absent: `docker run --rm <image> ls -a`.

## 4. Neon

1. Read the current terms: commercial use on the free plan, and the data region if the app
   holds personal data. Record both in `DECISIONS.md`.
2. Create the project in that region. Use the **pooled** connection string with
   `sslmode=require` as `DATABASE_URL`.
3. Note the connection limit of the plan; set the `pg` pool `max` below it.

## 5. Render

1. New Web Service from the repository, runtime Docker, instance type Free.
2. Auto-Deploy: off. Copy the Deploy Hook URL into the GitHub secret `RENDER_DEPLOY_HOOK`
   on the deploy job's environment only.
3. Health Check Path: `/api/health`.
4. Environment: `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL` (the public
   `https://` URL), and every other variable in `.env.example`.
5. Find which header carries the client address behind Render's proxy **[verify:
   `X-Forwarded-For` vs a Render-specific header]**, set it in Better Auth, record it in
   `DECISIONS.md`.
6. Walking skeleton check: sign up from two different networks and confirm the rate
   limiter counts them separately.

## 6. Release

1. Bump `version` in `package.json`; commit; tag `v<version>`; push the tag.
2. CI runs every check (§5.5 of the rule). If the tag adds a file under `drizzle/` since the
   previous tag, CI stops before deploying and says so.
3. Migration release only — the human:
   1. Record the row counts of the main tables, then
      `pg_dump --format=custom --no-owner "$DIRECT_URL" > backup-<tag>.dump` with Neon's
      **direct** (not pooled) URL, held on the human's machine only.
   2. Restore into an empty database (a Neon branch or local PostgreSQL):
      `pg_restore --no-owner --exit-on-error -d "$EMPTY_URL" backup-<tag>.dump`; the
      restored row counts match the recorded ones. If they differ, users wrote during the
      dump: suspend the service on the host, and repeat from step 1.
   3. The dump holds password hashes and personal data. Keep it encrypted, or delete it
      once the release is confirmed.
   3. Trigger the deploy hook by hand.
4. After deploy: `/api/health` reports the tagged version; `/` answers 200; sign in once.

## 7. Scheduled backup

Only for an app holding data its users cannot recreate (rule §4.3).

GitHub Actions workflow on a schedule: install the PostgreSQL client at or above the
server's major, `pg_dump` with Neon's direct URL from the secret `BACKUP_DATABASE_URL` —
set on the backup job's environment only, never on the app or another job. Encrypt the
dump with the public key named in the ADR (the private key is never in CI or in the store),
upload it to the chosen off-Neon store, fail the job on any non-zero exit. Confirm once
that a failed run notifies the human **[verify GitHub's notification for scheduled
workflows]**, and restore an encrypted dump once end to end. Store, retention and key
location are an ADR.

## 8. Rollback

1. Check first whether `drizzle/` changed between the previous tag and the deployed one.
2. No change: redeploy the previous tag through the host.
3. Changed: do not redeploy yet — the old code may not run on the new schema. Decide with
   the human: fix forward with a new migration, or restore the pre-deploy dump and then
   redeploy the previous tag. Restoring discards every write made since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. After rollback, repeat the post-deploy checks of §6.4.
