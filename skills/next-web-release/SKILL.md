---
name: next-web-release
description: Set up, build, deploy, back up and roll back a web-ts project (Next.js standalone container on Render, Neon PostgreSQL, Better Auth, Drizzle). Use when creating a web-ts project, when editing next.config.ts, src/proxy.ts, drizzle.config.ts, the Better Auth configuration or its hooks, src/adapters/db/migrate.ts, the Dockerfile or CI and backup workflows, when pnpm install fails with ERR_PNPM_IGNORED_BUILDS, when a release carries a file under drizzle/, when a deploy or a post-deploy check fails, or when rolling back.
---

# next-web-release

Procedures for `rules/web-ts.md` §2, §4 and §5. The pnpm version is the one in
`package.json` `packageManager`. Shell steps that pipe one command into another run with
`set -o pipefail`; without it a failed `pg_dump` piped into an encrypter exits 0 and
leaves a useless file that looks like a backup.

## 1. First setup on a developer machine

1. Install pnpm standalone (`curl -fsSL https://get.pnpm.io | PNPM_VERSION=<v> sh -`, or on
   Windows `$env:PNPM_VERSION='<v>'; iwr https://get.pnpm.io/ps1 -useb | iex`). Check:
   `pnpm --version` equals `packageManager`.
2. `pnpm install`. pnpm downloads Node from `devEngines.runtime`. Check:
   `pnpm exec node --version`.
3. `pnpm exec playwright install chromium`, once; tests then run offline.
4. Copy `.env.example` to `.env.local`: `DATABASE_URL=pglite:./.data/dev`,
   `BETTER_AUTH_SECRET` from `openssl rand -base64 32`,
   `BETTER_AUTH_URL=http://localhost:3000`. `.data/` and `.env.local` are gitignored.
5. `pnpm db:migrate`, then `pnpm dev`.

## 2. New project skeleton

### 2.1 Create

```
pnpm create next-app@<exact version> <name> --ts --app --src-dir --tailwind --biome --use-pnpm --import-alias "@/*" --no-agents-md --no-agent-feedback --disable-git
```

Then:

- In `next.config.ts`, delete `cacheComponents` and `partialPrefetching` if the generator
  wrote them (it does with `--yes`). With them on, a root layout that reads `headers()`
  fails the build, and the nonce CSP of rule §4.4 cannot work: Next.js documents nonces
  as incompatible with Partial Prerendering.
- Keep the generated Tailwind setup: `@import "tailwindcss";` in `src/app/globals.css`
  and the `@tailwindcss/turbopack` loader in `next.config.ts`. There is no PostCSS
  config.
- Add to `package.json`: `"type": "module"`, and `devEngines.runtime` with an exact LTS
  version and `"onFail": "download"`. `packageManager` is already set.
- Pin the generated `^` ranges to exact versions.

### 2.2 `pnpm-workspace.yaml`

pnpm fails the install when a dependency's install script is neither allowed nor denied.
Deny each package `ERR_PNPM_IGNORED_BUILDS` names unless the app fails without its
script. With the dependencies of this skill:

```yaml
allowBuilds:
  sharp: false
  unrs-resolver: false
  esbuild: false
```

`esbuild` works without its install script; drizzle-kit and §3 use it.

### 2.3 `next.config.ts`

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  experimental: { serverActions: { bodySizeLimit: 1048576 } },
  turbopack: {
    rules: { "*.css": { loaders: ["@tailwindcss/turbopack"], as: "*.css" } },
  },
  async headers() {
    return [{
      source: "/:path*",
      headers: [
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
      ],
    }];
  },
  async redirects() {
    return [];
  },
};

export default nextConfig;
```

`bodySizeLimit` counts the raw body, with or without `Content-Length`: exactly 1,048,576
bytes is accepted, one more is refused. Multipart boundaries and part headers count.

### 2.4 Nonce CSP (rule §4.4)

`src/proxy.ts`:

```ts
import { type NextRequest, NextResponse } from "next/server";

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");
  const csp = [
    "default-src 'self'",
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data:",
    "font-src 'self'",
    "connect-src 'self'",
    "object-src 'none'",
    "base-uri 'self'",
    "form-action 'self'",
    "frame-ancestors 'none'",
  ].join("; ");
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-nonce", nonce);
  requestHeaders.set("Content-Security-Policy", csp);
  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set("Content-Security-Policy", csp);
  return response;
}

export const config = {
  matcher: [{
    source: "/((?!api|_next/static|_next/image|favicon.ico).*)",
    missing: [
      { type: "header", key: "next-router-prefetch" },
      { type: "header", key: "purpose", value: "prefetch" },
    ],
  }],
};
```

The root layout is `async` and reads `(await headers()).get("x-nonce")`. That makes every
page render per request: `next build` lists every route as `ƒ (Dynamic)`. A route shown
as `○ (Static)` has no nonce, and its scripts are blocked.

The generated layout loads Google fonts through `next/font/google`, which downloads them
at build time. On a network that blocks it, use `next/font/local` with the font files in
the repository.

### 2.5 Tools

- shadcn/ui: `pnpm add -D -E shadcn`, then `pnpm exec shadcn init --base radix`.
- Drizzle: `pnpm add -E drizzle-orm pg @electric-sql/pglite` and
  `pnpm add -D -E drizzle-kit esbuild @types/pg`.
- Better Auth: `pnpm add -E better-auth` and `pnpm add -D -E auth`. `auth` is the Better
  Auth CLI, released with the same version as `better-auth`. `@better-auth/cli` is its
  old name; do not add it.

### 2.6 Database files

`drizzle.config.ts` lists the schema files by name. A glob such as
`src/adapters/db/*.ts` also loads `migrate.ts` and `client.ts`, and drizzle-kit fails on
their top-level `await`.

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  dialect: "postgresql",
  schema: ["./src/adapters/db/schema.ts", "./src/adapters/db/auth-schema.ts"],
  out: "./drizzle",
});
```

`src/adapters/db/client.ts` chooses the driver by the URL scheme (rule §4.3): `pglite:` +
a directory opens `new PGlite(dir)` with `drizzle-orm/pglite`; anything else opens a
`pg.Pool` with `drizzle-orm/node-postgres`.

`src/adapters/db/migrate.ts` runs under plain Node, which strips the types itself, so it
imports only packages and relative paths with a `.ts` extension, never `@/` aliases.
PGlite is imported only when the URL asks for it, so the production bundle (§3) does not
need it:

```ts
import { drizzle as drizzlePg } from "drizzle-orm/node-postgres";
import { migrate as migratePg } from "drizzle-orm/node-postgres/migrator";
import pg from "pg";

const url = process.env.DATABASE_URL;
if (!url) {
  console.error("DATABASE_URL is not set");
  process.exit(1);
}
const migrationsFolder =
  process.env.MIGRATIONS_DIR ?? new URL("../../../drizzle", import.meta.url).pathname;

if (url.startsWith("pglite:")) {
  const { PGlite } = await import("@electric-sql/pglite");
  const { drizzle } = await import("drizzle-orm/pglite");
  const { migrate } = await import("drizzle-orm/pglite/migrator");
  const client = new PGlite(url.slice("pglite:".length));
  await migrate(drizzle({ client }), { migrationsFolder });
  await client.close();
} else {
  const pool = new pg.Pool({ connectionString: url, max: 1 });
  await migratePg(drizzlePg({ client: pool }), { migrationsFolder });
  await pool.end();
}
console.log("migrations applied");
```

Script: `"db:migrate": "node --env-file-if-exists=.env.local src/adapters/db/migrate.ts"`.
It applies only pending migrations and exits non-zero on failure.

### 2.7 Better Auth

`src/adapters/auth/auth.ts`:

```ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { nextCookies } from "better-auth/next-js";

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: "pg", schema }),
  emailAndPassword: { enabled: true },
  rateLimit: { enabled: true, storage: "database" },
  session: { expiresIn: 60 * 60 * 24 },                     // idle time, rule §4.4
  advanced: { ipAddress: { ipAddressHeaders: ["<header from §5 step 5>"] } },
  hooks: { before: signInBefore, after: signInAfter },
  plugins: [nextCookies()],                                 // always last
});
```

- `ipAddressHeaders` defaults to `x-forwarded-for`. Better Auth skips a header holding
  more than one address unless `advanced.ipAddress.trustedProxies` lists the proxies.
- Mount `toNextJsHandler(auth)` at `src/app/api/auth/[...all]/route.ts`, wrapped so the
  body is counted as it arrives and refused past 1 MiB (rule §4.1).
- Better Auth already hashes the given password when the email is unknown, and returns
  the same `INVALID_EMAIL_OR_PASSWORD` error, so both paths cost the same.

Sign-in counters (rule §4.4): a table in `adapters/db/schema.ts` for the account+address
pair and for the address. The hooks, from `better-auth/api`:

```ts
import { APIError, createAuthMiddleware } from "better-auth/api";

const signInBefore = createAuthMiddleware(async (ctx) => {
  if (ctx.path !== "/sign-in/email") return;
  // read ctx.body.email and the address header; refuse (throw new APIError(...)) or slow
  // down according to the counters
});

const signInAfter = createAuthMiddleware(async (ctx) => {
  if (ctx.path !== "/sign-in/email") return;
  const failed = ctx.context.returned instanceof APIError;
  // failed: record a failure for the pair and the address; succeeded: reset the pair
});
```

Absolute session lifetime: Better Auth has no option for it. The session reader in
`server.ts` reads the session, and revokes and refuses one whose `createdAt` is older than
the absolute time.

Generate the tables, then the migration, and read the SQL before committing:

```
pnpm exec auth generate --config src/adapters/auth/auth.ts --output src/adapters/db/auth-schema.ts --yes
pnpm exec drizzle-kit generate
```

### 2.8 Tests

- Vitest, two projects: `unit` (`environment: "node"`, `tests/unit`, `tests/adapters`) and
  `browser` (`@vitest/browser-playwright`, Chromium, `tests/browser`,
  `vitest-browser-react`).
- Playwright, after `pnpm build`: `webServer` copies `.next/static` and `public` into
  `.next/standalone` and runs `node .next/standalone/server.js` with a temp PGlite
  `DATABASE_URL` and placeholder secrets. A fixture fails the test on any
  `securitypolicyviolation` event or CSP console error.

## 3. Dockerfile

Multi-stage. Base: `node:<major from devEngines.runtime>-<variant>@sha256:<digest>`.

1. **deps:** install pnpm at the `packageManager` version; `pnpm install
   --frozen-lockfile`.
2. **build:** `NEXT_TELEMETRY_DISABLED=1 pnpm build` with no secrets in the environment.
   Then bundle the migrator into one file; standalone output does not contain Drizzle,
   so `migrate.ts` cannot run there as it is:

   ```
   pnpm exec esbuild src/adapters/db/migrate.ts --bundle --platform=node --format=esm \
     --target=node<major> --external:@electric-sql/pglite --external:drizzle-orm/pglite \
     --external:drizzle-orm/pglite/migrator --external:pg-native \
     "--banner:js=import { createRequire } from 'node:module'; const require = createRequire(import.meta.url);" \
     --outfile=build/migrate.mjs
   ```

   The banner is required: without it the bundle fails with `Dynamic require of "events"
   is not supported`.
3. **run:** copy `.next/standalone/`, then `public/` to `./public` and `.next/static/` to
   `./.next/static`; copy `build/migrate.mjs` and `drizzle/`. `USER node`;
   `ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 HOSTNAME=0.0.0.0
   MIGRATIONS_DIR=/app/drizzle`; `CMD ["sh", "-c", "node migrate.mjs && node server.js"]`.

Check that `.env*` is absent: `docker run --rm <image> ls -a`.

## 4. Neon

1. Read the current terms: commercial use on the free plan, and the data region if the app
   holds personal data. Record both in `DECISIONS.md` (skill `adr`).
2. Create the project in that region. `DATABASE_URL` is the **pooled** connection string
   with `sslmode=require`. The **direct** string is for `pg_dump`, `pg_restore` and §7 step 5
   only: they fail over the pooled endpoint.
3. Note the plan's connection limit; set the `pg` pool `max` below it.

## 5. Render

1. New Web Service from the repository, runtime Docker, instance type Free.
2. Auto-Deploy: Off. Copy the deploy hook URL into the GitHub secret
   `RENDER_DEPLOY_HOOK_URL`, set on the deploy step only. Anyone holding it can deploy.
3. Health Check Path: `/api/health`.
4. Environment: `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL` (the public
   `https://` URL), and every other variable in `.env.example`.
5. Client address: Render does not document it. Render's proxy appends to a client-sent
   `X-Forwarded-For` instead of replacing it, so that header can be forged. Start with
   `true-client-ip`, set by the CDN in front of Render. Record the header in
   `DECISIONS.md`.
6. Walking skeleton check: sign in with wrong passwords from two networks and confirm the
   counters show two addresses; then send sign-ins carrying a made-up `X-Forwarded-For`
   and a made-up `True-Client-IP`, and confirm the counted address does not change. If
   either check fails, stop: no header is trusted, and the human decides.

## 6. Release

1. Bump `version` in `package.json`; commit; tag `v<version>`; push the tag after the
   human approves it.
2. CI runs every check (rule §5.5), then one of two paths:
   - Nothing under `drizzle/` differs from the previous tag: call the deploy hook for the
     tagged commit, `curl -fsS -X POST "$RENDER_DEPLOY_HOOK_URL&ref=$GITHUB_SHA"`, then
     §9 checks 1 and 2.
   - Something differs: end without calling the hook, and say that the release needs §7.

## 7. Deploy carrying a migration (rule §4.3, §5.4)

Render refuses a deploy hook call while a service is suspended (409), and does not
document which commit it builds on resume. So the human migrates the database first,
while nothing runs; after that, any older commit Render starts finds migrations it does
not know and refuses to start (rule §4.3).

On the human's machine, at the tag, with `DIRECT_URL` the database owner's direct Neon
URL, held on this machine only:

1. `git checkout v<version>` and `pnpm install --frozen-lockfile`.
2. Suspend the service on Render, and stop anything else that writes to the database.
   Check that the site no longer answers.
3. Dump into encryption with the public key of §8:

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
5. Migrate production: `DATABASE_URL="$DIRECT_URL" node src/adapters/db/migrate.ts`. On
   failure, compare `drizzle.__drizzle_migrations` with the previous tag's `drizzle/`.
   Unchanged: resume and report. Changed: stop; restoring the dump is the human's decision
   (§10).
6. Resume the service on Render.
7. Call the deploy hook with `&ref=<tagged commit SHA>`. 200 means the deploy started; 409
   means the service is still suspended.
8. §9.
9. Keep only the encrypted dump, under the retention in `DECISIONS.md`. Delete the
   restore-test database or branch.

## 8. Scheduled backup

Only for an app holding data its users cannot recreate (rule §4.3).

Keys, once, on the human's machine: install a tagged release of `FiloSottile/age`;
`age-keygen -o backup-key.txt` prints the public key (`age1…`). The file holds the private
key: keep it on that machine and one offline copy, never in the repository, CI or the
backup store. Record the release, its SHA-256 and the public key in `DECISIONS.md`.
Losing the private key makes every backup unreadable.

A GitHub Actions workflow on a `schedule`. Install the PostgreSQL client at or above the
server's major, and the age release, checked against its SHA-256.
`BACKUP_DATABASE_URL`, the direct URL, is set on the backup job's environment only. CI
holds only the public key, so it checks the dump before encrypting it, in steps with
`shell: bash` (GitHub runs those with `-eo pipefail`):

1. `pg_dump --format=custom --no-owner "$BACKUP_DATABASE_URL" > "$RUNNER_TEMP/db.dump"`
2. `pg_restore --no-owner --no-acl --exit-on-error` of that file into a PostgreSQL service
   container; fail unless its latest migration equals production's and every table the
   schema defines exists.
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
- Once, before the app holds real data: make a run fail on purpose and confirm the
  notification arrives; restore an encrypted dump end to end with the private key.

## 9. Post-deploy checks

Allow two minutes for a cold start, then:

1. `/api/health` answers 200 with the tag's version and the latest migration.
2. `/` answers 200.
3. The human signs in and opens one page that reads data. CI runs 1 and 2 only.

A failed check stops everything: report it with the options of §10. No retry and no
redeploy.

## 10. Rollback

1. Check whether `drizzle/` changed between the previous tag and the deployed one.
2. No change: call the deploy hook with `&ref=<previous tag's commit SHA>`.
3. Changed: do not redeploy yet; the old code refuses to start on the new schema. Decide
   with the human: fix forward with a new migration, or restore the pre-deploy dump and
   then redeploy the previous tag. Restoring discards every write since the dump. Before
   choosing it, dump the current database, state what the restore would lose, and get the
   human's explicit approval of that loss.
4. §9.

## 11. CI runner setup

Every `uses:` is pinned to a full commit SHA. Set up pnpm with `pnpm/setup` (v3,
`install: false`), which installs the pnpm of `packageManager` and the Node of
`devEngines.runtime`; then `pnpm install --frozen-lockfile`, then
`pnpm exec playwright install --with-deps chromium`, before Vitest and Playwright. An app
with no database skips the PostgreSQL service container and the migration checks.
