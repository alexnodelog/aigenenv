---
name: next-web-release
description: Set up, build, deploy and roll back a web-ts project (Next.js standalone container on Render, Neon PostgreSQL, Better Auth, Drizzle). Use for first setup, the walking skeleton, a release, a migration deploy, a backup, or a rollback.
---

# next-web-release

Items marked **[verify]** were not confirmed against the installed CLI. Confirm them in
step 7 of the framework build before this skill leaves `_drafts/`.

## 1. First setup on a developer machine

1. Install pnpm standalone. Confirm: `pnpm --version` matches `packageManager`.
2. `pnpm install` — pnpm downloads Node from `devEngines.runtime`. Confirm: `pnpm exec node --version`.
3. `pnpm exec playwright install chromium` (once; tests then run offline).
4. Copy `.env.example` to `.env.local`. Set `DATABASE_URL=pglite:./.data/dev` **[verify the
   URL form `client.ts` parses]**, `BETTER_AUTH_SECRET` from `openssl rand -base64 32`,
   `BETTER_AUTH_URL=http://localhost:3000`. `.data/` and `.env.local` are gitignored.
5. `pnpm db:migrate`, then `pnpm dev`.

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
10. Playwright: `webServer` runs `pnpm start` on the built app with a temp PGlite
    `DATABASE_URL` and placeholder secrets.

## 3. Dockerfile shape

Multi-stage. Base: `node:<major>-<variant>@sha256:<digest>`.

1. **deps** — install pnpm at the `packageManager` version; `pnpm install --frozen-lockfile`.
2. **build** — `NEXT_TELEMETRY_DISABLED=1 pnpm build` with no secrets in the environment.
3. **run** — copy `.next/standalone/`, then `public/` → `./public` and `.next/static/` →
   `./.next/static`; copy `drizzle/` and the compiled `migrate` entry **[verify how
   `migrate.ts` is compiled into the standalone image]**; `USER node`;
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
   1. `pg_dump --format=custom --no-owner "$DIRECT_URL" > backup-<tag>.dump` using Neon's
      **direct** (not pooled) URL.
   2. Restore into an empty database (a Neon branch or local PostgreSQL):
      `pg_restore --no-owner -d "$EMPTY_URL" backup-<tag>.dump`; check row counts of the
      main tables.
   3. Trigger the deploy hook by hand.
4. After deploy: `/api/health` reports the tagged version; `/` answers 200; sign in once.

## 7. Scheduled backup

GitHub Actions workflow on a schedule: install the PostgreSQL client at or above the
server's major, `pg_dump` with the direct URL from a secret, upload the dump to the chosen
off-Neon store, fail the job on any non-zero exit. The store and retention are an ADR.

## 8. Rollback

1. Redeploy the previous tag through the host.
2. If a migration ran since that tag, the old code may not run on the new schema. Stop and
   decide with the human: restore the pre-deploy dump, or fix forward with a new migration.
3. After rollback, repeat the post-deploy checks of §6.4.
