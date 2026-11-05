# rules/web-ts.md

Web app, TypeScript end to end. One Next.js server renders the pages and runs the logic;
it ships as a container and keeps any durable data in PostgreSQL.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if there is no server logic at all (`web-static`), if it is an internal
tool, dashboard or PoC with no need for a designed UI (`web-python`), if the server needs
Python libraries (`web-api`), or if one person uses it on one machine with local data
(`pc-app-python`).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Node.js | An LTS line, installed by pnpm, pinned in `package.json` `devEngines.runtime` | — | Odd-numbered (non-LTS) majors; `.nvmrc`, nvm, fnm, Volta as the version record |
| Packaging | pnpm, standalone install, exact version in `package.json` `packageManager`; `pnpm-lock.yaml` | — | npm, yarn, bun; Corepack; `npx` in scripts or docs |
| Language | TypeScript, `strict: true`, ES modules | — | Plain JavaScript in new code |
| Framework | Next.js 16, App Router, `output: "standalone"`, Node.js runtime | — | Pages Router; `output: "export"` (that is `web-static`); the Edge runtime (`runtime = "edge"`); OpenNext, vinext, or any adapter that runs the app outside a Node server |
| Styling | Tailwind CSS v4 via `@tailwindcss/postcss`; shadcn/ui, Radix base, copied into `src/components/ui/` | Plain CSS | `tailwind.config.js`, `@tailwind` directives; CSS-in-JS; full component kits installed as packages (MUI, Chakra, Mantine, Ant Design) |
| Client state | None — `useState`, `useReducer`, URL search params | Zustand, for state shared by distant client components | Redux, Redux Toolkit |
| Server data | Read in server components; changed by server actions | TanStack Query, for polling, infinite scroll, or optimistic updates | SWR; `useEffect` + `fetch` to load a page's data |
| Validation | Zod 4 (4.6.6 or later) | — | Unvalidated `FormData` or JSON reaching a use case |
| Persistence | PostgreSQL via Drizzle, its stable release line. `pg` (node-postgres) in production; PGlite (`@electric-sql/pglite`) in development and tests | — | A Drizzle beta or release candidate; SQLite, Cloudflare D1; Prisma; a vendor-specific driver (`@neondatabase/serverless`); `sql.raw` with any value from input |
| Migrations | `drizzle-kit generate`, SQL files in `drizzle/`, committed | — | `drizzle-kit push`; editing a shipped migration; hand-applied schema changes |
| Accounts | Better Auth: email and password, database sessions, Drizzle adapter | — | A vendor's auth service (Clerk, Auth0, Supabase Auth, Firebase Auth); Auth.js / NextAuth; Better Auth `session.cookieCache`; JWT browser sessions; tokens in `localStorage` |
| User file uploads | None | Storage chosen by ADR | The container's filesystem |
| Background work | None — work finishes inside the request | `after()` for best-effort, non-durable follow-ups; a durable job mechanism, chosen by ADR, for work whose loss is unacceptable | Timers or loops started from a request |
| Lint / format | Biome | ESLint + Prettier | `next lint` (removed in Next.js 16) |
| Type check | `tsc --noEmit` before `next build` | — | `typescript.ignoreBuildErrors` |
| Tests | Vitest — `node` for logic, browser mode (`@vitest/browser-playwright`, `vitest-browser-react`) for client components; Playwright for pages | — | jsdom, happy-dom; Jest; Cypress |
| Third-party runtime resources | None: scripts, styles and fonts ship in the build; fonts through `next/font` | — | `<script>` or `<link>` to a CDN; tracking or analytics tags |
| Deploy artifact | `Dockerfile` → container image | — | Docker required for local development or tests; a release image built on a developer machine |
| Hosting | Render free web service, plus Neon Free Postgres when the app keeps durable data | Oracle Cloud Always Free VM; Vercel Hobby — personal non-commercial projects only | Cloudflare Workers; Render's disk or Render free Postgres as the store of record; a work tool on Vercel Hobby; artificial traffic or CPU load to defeat a host's idle policy |

---

## 2. Prerequisites

`pnpm install` neither installs nor reveals these. Procedure: skill `next-web-release`.

| Item | Scope |
|---|---|
| pnpm | System, standalone, installed first. The exact version in `packageManager` |
| Node.js | Installed by pnpm from `devEngines.runtime`. No separate Node installer |
| Chromium for tests | Project, downloaded once by `pnpm exec playwright install chromium`; tests then run offline |
| Docker or Podman | System. Optional on the developer's machine; required in CI (§5.5) |
| Host account | Render or Oracle Cloud — whichever `DECISIONS.md` names |
| Neon account | Account, only when the app keeps durable data. Before first use, confirm in Neon's current terms that the free plan permits commercial use, and confirm the data region if the app holds personal data; record both in `DECISIONS.md` |
| PostgreSQL client tools (`pg_dump`, `psql`) | System, only when the app keeps durable data. Major version at or above the server's, or `pg_dump` refuses to run |

---

## 3. Layout

```
<repo>/
├── package.json          version, packageManager, devEngines.runtime, scripts
├── pnpm-lock.yaml
├── pnpm-workspace.yaml   pnpm settings: allowBuilds
├── tsconfig.json         whole project, strict
├── tsconfig.core.json    src/domain/ and src/services/ only, no DOM library
├── next.config.ts        output: "standalone", headers(), redirects()
├── postcss.config.mjs
├── biome.json
├── components.json       shadcn/ui
├── drizzle.config.ts
├── vitest.config.ts      projects: logic in Node, client components in Chromium
├── playwright.config.ts
├── Dockerfile
├── .env.example
├── drizzle/              generated migrations + meta/, committed
├── public/               served as-is, public
├── src/
│   ├── app/              routes: page, layout, error, actions.ts beside the route;
│   │                     api/auth/[...all]/route.ts, api/health/route.ts; globals.css
│   ├── components/       app components; ui/ = shadcn/ui source
│   ├── instrumentation.ts  register(): calls server.ts once at server start
│   ├── proxy.ts          sets the per-request CSP nonce (§4.4); nothing else
│   ├── config.ts         the only reader of process.env
│   ├── server.ts         composition root: builds the pool, adapters, services
│   ├── domain/           pure TypeScript
│   ├── services/         use cases + ports.ts
│   └── adapters/         db/ (schema.ts, auth-schema.ts, client.ts, migrate.ts), auth.ts, clock
├── tests/                unit/, adapters/, browser/, e2e/, smoke/, boundaries.test.ts
└── .next/                gitignored
```

- `src/server.ts` is the only module that imports `config.ts`, creates the connection pool,
  and constructs adapters and services. It builds them once per process and exports a
  function that returns them. Route files call that function; they construct nothing.
- `server.ts`, `config.ts` and everything in `adapters/` start with `import "server-only"`.
  A client component that reaches any of them fails the build.
- `adapters/db/migrate.ts` is the one exception to both rules above. It runs under plain
  Node, outside Next.js, so it has no `server-only` import, reads `DATABASE_URL` itself,
  and imports only Drizzle's migrator and the two drivers. `pnpm db:migrate` and the
  container run this same file.
- An app with no business rules MAY omit `domain/` and `services/`, and MUST NOT add them
  to match this layout. A new top-level dir needs an ADR only if it changes a row of the
  import table below.

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | `domain/` | React, `next/*`, Drizzle, `pg`, PGlite, Better Auth, Zod, `adapters/`, `app/`, `components/` |
| `services/` | `domain/`, own `ports.ts`, Zod | React, `next/*`, Drizzle, `pg`, PGlite, Better Auth, `adapters/`, `app/`, `components/` |
| `adapters/` | `services/ports.ts`, `domain/`, Drizzle, `pg`, PGlite, Better Auth | React, `next/*`, `app/`, `components/` |
| `app/`, `components/` | `services/` types, `domain/` types, `server.ts`, React, `next/*` | Drizzle, `pg`, PGlite, `adapters/` |
| Files with `"use client"` | `components/`, `domain/` types, React | `server.ts`, `config.ts`, `services/`, `adapters/` |

- No framework type (`NextRequest`, `FormData`, `ReadonlyHeaders`) outside `app/`,
  `components/` and `server.ts`.
- A server action or route handler parses input with Zod, calls one use case, and returns
  or renders the result. A decision about *what the application does*, taken in an action,
  handler, page or component, is a defect.
- `tsconfig.core.json` includes only `src/domain/` and `src/services/`, with no `DOM` in
  `compilerOptions.lib`, so DOM use there fails the type check. `tests/boundaries.test.ts` fails on any
  import that breaks the table. These two are the boundary check CI and §8 refer to.

---

## 4. Runtime

### 4.1 Request and rendering model

- Components are server components unless they need state, effects, or browser APIs.
  `"use client"` goes on the smallest component that needs it, never on a page or layout.
- Everything passed as a prop to a client component is sent to the browser. Pass the
  fields the component displays, never a database row, a session object, or a config
  value.
- Every server action and route handler is a public HTTP endpoint, whether or not a page
  links to it. Each one validates its input and resolves the current user itself; the use
  case checks what that user may do. Hiding a control, or a check in `proxy.ts`, is not
  authorization.
- Browser-initiated changes go through server actions, which Next.js checks against the
  request's origin. Apart from the Better Auth mount, which protects its own endpoints, a
  route handler MUST NOT change state for a cookie-authenticated browser request. Route
  handlers serve that mount, the health check, and callers that are not browsers.
- `GET` never changes state.
- Every string and array field in a Zod input schema has a maximum length, and every list
  a maximum page size. A request body of up to 1 MiB (1,048,576 bytes) is accepted; one
  byte more is refused, unless an ADR raises the limit for one action or route. Server
  actions keep Next.js's `bodySizeLimit` at that value; a route handler that reads a body
  counts bytes as they arrive, also without `Content-Length`, and refuses as soon as the
  count passes the limit, without reading the body whole.
- An expected failure — invalid input, a rule the use case refused — is returned from the
  action as a value and shown next to the field. Only unexpected errors are thrown.
- Data reaches the DOM through JSX. `dangerouslySetInnerHTML` with any value that is not
  a compile-time constant is PROHIBITED.
- Cache Components (`cacheComponents`, `"use cache"`) stay off; turning them on is an ADR.
  A page that reads the session, a cookie or the database renders per request.
- Work that outlives the request does not run in the web process: a free-tier host stops
  it on spin-down without warning. `after()` is only for effects whose loss is acceptable —
  never one that costs money or writes to a third party.

### 4.2 Paths

- Every page and API URL is a route under `src/app/`. A published URL is a contract: renaming one adds
  a permanent entry to `redirects()` in `next.config.ts` in the same commit.
- An unknown path answers 404 through `not-found.tsx`, never a redirect to the home page.
- The app writes no files at runtime. Next.js's own `.next/cache` is the only exception
  and is lost on every restart.
- Everything in `public/` is served to anyone, at its own path.

### 4.3 Persistence

- Durable data lives in PostgreSQL, Neon Free by default. An app with no durable data has
  no database.
- The app reaches the database only through `DATABASE_URL`. `adapters/db/client.ts`
  chooses the driver by its scheme: `pglite:` opens PGlite (development, tests),
  `postgres:` opens a `pg` pool (CI and production). Moving to another PostgreSQL
  provider changes the variable and nothing in the code.
- Development and the default test run use PGlite, offline. CI also runs the adapter
  tests and the migrations against a PostgreSQL service container, and a migration MUST
  pass on both. A local PostgreSQL MAY be used but MUST NOT be required.
- One pool per process, created in `server.ts`, sized for Neon Free's connection limit
  and pointed at Neon's pooled endpoint. The pool's connection timeout and PostgreSQL
  `statement_timeout` are both set, 10 s each unless an ADR names other values, so the
  first query after Neon has slept is slow, not failed, and no query hangs a request.
- A use case that writes more than once runs those writes in one `db.transaction`. A
  failed action leaves no partial write.
- Migrations are generated by `drizzle-kit generate`, then read and corrected by hand
  before commit: it asks about renames interactively and can turn a rename into a drop
  and an add.
- Better Auth's tables are generated by its CLI into `adapters/db/auth-schema.ts`,
  committed, and never edited by hand. A change to the Better Auth configuration or its
  plugins regenerates that file in the same commit, followed by `drizzle-kit generate`.
- An app with no database has no `migrate.ts` and no `drizzle/`; its health check reports
  the version only.
- `adapters/db/migrate.ts` runs at container start, before the server. On failure it exits
  non-zero, the server does not start, and the database is left at its previous migration.
  A database holding migrations the code does not know → refuse to start, log why.
- The app cannot dump its own database. Before a deploy that carries a migration, the human
  suspends the service on the host, takes a `pg_dump`, confirms it restores into an empty
  database, and deploys; the service serves again only on the new version. The web
  service is the only writer to the database; anything else that writes is stopped too.
  No write lands after the dump, and no old instance runs during the migration. §5.4 stops
  CI for this. Neon's point-in-time restore is not the backup.
- An app holding data its users cannot recreate MUST take a backup on a schedule and keep
  it off Neon. A restore from that backup into an empty database MUST have succeeded once
  before the app holds real data. Backups are encrypted before they leave the
  CI job, with a key kept apart from the store that holds them. A failed backup run MUST
  fail a scheduled CI job whose failure notification reaches the human. Each run restores
  its own dump into an empty database and checks, as a minimum proof that the dump
  restores, its latest applied migration and that every table the schema defines exists,
  before keeping it; a failed check fails the run. The data loss users can accept, the
  schedule that follows from it, retention, store and key location are recorded in
  `DECISIONS.md`.
- Shipped migrations are append-only: a migration's SQL and snapshot, once on `main`,
  are never modified or deleted, and generated metadata such as `meta/_journal.json`
  only gains entries at its end. Fixes are new migrations.

### 4.4 Settings, credentials, errors

- `src/config.ts` builds a frozen settings object from `process.env`, validated with Zod.
  `instrumentation.ts` calls `server.ts` in `register()`, so a missing or invalid variable
  stops the server at start with a message naming the variable, never its value.
- `next build` MUST succeed without `DATABASE_URL` or any secret. A page that would read
  the database or a secret at build time renders per request instead.
- `NEXT_PUBLIC_*` values are compiled into the browser bundle at build time and are never
  secret. Every other variable is read at runtime only.
- Better Auth: `BETTER_AUTH_SECRET` (at least 32 characters) and `BETTER_AUTH_URL` come
  from the environment; secure cookies in production; its built-in rate limiting on,
  stored in the database (`rateLimit.storage = "database"`) so it survives a restart.
- Sessions expire after an idle time of 24 hours and an absolute time of 30 days, unless
  an ADR names other values. Better Auth's `expiresIn` sets the idle time; the app's own
  session reader refuses and revokes a session older than the absolute time.
- Failed sign-ins are counted in the database. Better Auth's built-in limiter counts every
  request per address and path, so two separate counters are added through its hooks,
  unless an ADR names other values:
  - Account and client address together: after 5 consecutive failures the pair is refused
    for 15 minutes. Its count resets on success and when the refusal expires. Keying by the
    pair means an attacker elsewhere cannot lock the owner out.
  - Client address alone: failures in the last 15 minutes, rolling, never reset. While
    there are 20 or more, the address may make one sign-in attempt per minute; it is
    slowed, not blocked, so users sharing the address can still sign in.
- The start of a refusal or a slowdown is logged once as a warning; the attempts it
  refuses are not logged one by one.
- A sign-in response is the same whether or not the account exists, and a sign-in for an
  unknown account costs the same password-hash work as one for a known account.
- Rate limiting needs the real client address. The header the host uses for it is set in
  Better Auth's configuration and recorded in `DECISIONS.md`, and the walking skeleton
  (§5.2) confirms two clients are counted separately, and that an `X-Forwarded-For` header
  the client sets itself does not change the address counted.
- No mail provider is selected. A feature that sends mail — address verification,
  password reset, notifications — needs an ADR naming the provider and its free-tier terms
  before it is built. Until then, a password reset is done by an administrator.
- A server has no OS credential store, so this archetype names none. Storing an end user's
  own third-party credential needs an ADR naming how it is encrypted at rest and with which
  deployment key.
- Every route segment that renders data has an `error.tsx`, and the root has
  `global-error.tsx`. They show what failed and what to try, and the digest Next.js
  attaches, never the message of a server error.
- `next.config.ts` `headers()` sets, for every path, `X-Content-Type-Options: nosniff` and
  `Referrer-Policy: strict-origin-when-cross-origin`.
- `proxy.ts` generates a fresh nonce per request and sets, on the request and on the
  response, this Content-Security-Policy: `default-src 'self'; script-src 'self'
  'nonce-<nonce>' 'strict-dynamic'; style-src 'self' 'unsafe-inline'; img-src 'self'
  blob: data:; font-src 'self'; connect-src 'self'; object-src 'none'; base-uri 'self';
  form-action 'self'; frame-ancestors 'none'`, adding `'unsafe-eval'` to `script-src` in
  development only. Next.js applies the nonce to the scripts it renders. A nonce exists
  only when a page renders per request, so the root layout reads it through `headers()`,
  which makes every page render per request. `style-src` keeps `'unsafe-inline'` because
  server-rendered `style` attributes take no nonce. Loosening a directive needs an ADR.
- Logs go to stdout, one line per event. The app writes no log files.

---

## 5. Build and package

Procedure: skill `next-web-release`.

### 5.1 Lock the toolchain

- `packageManager` and `devEngines.runtime` (with `onFail: "download"`) are the single
  source for the pnpm and Node versions; CI installs from both. Every install outside a
  developer's first setup uses `pnpm install --frozen-lockfile`.
- Dependency install scripts run only for packages named in `allowBuilds` in
  `pnpm-workspace.yaml`. Allowing every package, or setting `minimumReleaseAge` to 0 to get
  a just-published version, is PROHIBITED.
- The `Dockerfile` base image uses the Node major in `devEngines.runtime`, pinned by
  digest, and installs the pnpm version in `packageManager`. A moving tag makes the image
  built at release differ from the one CI tested.
- `drizzle-kit`, the Better Auth CLI and the shadcn CLI are locked devDependencies, run
  with `pnpm exec`.

### 5.2 Build

- `pnpm build` runs `tsc --noEmit` on both configs, then `next build`.
- Tailwind v4 is configured in `src/app/globals.css` (`@import "tailwindcss";`) and loaded
  through `@tailwindcss/postcss`.
- The image copies the standalone output together with `public/` and `.next/static/` —
  standalone output omits both — installs nothing at runtime, runs as a non-root user,
  sets `NEXT_TELEMETRY_DISABLED=1`, runs `migrate.ts` (if the app has a database), then
  `node server.js` on `0.0.0.0`
  and the port the host passes in `PORT`.
- A project that will be deployed deploys a walking skeleton — one page, sign-in, the
  health check, the real host and database — before any feature beyond it is built.
  Leaving the first real deploy until release is PROHIBITED.

### 5.3 Self-check

`GET /api/health` answers 200 with the version from `package.json` and the latest applied
migration when the database answers and is at the code's latest migration; 503 otherwise.
It reveals nothing else. An app with no database reports the version only (§4.3).

The smoke test (`tests/smoke/`) starts the built image against an empty PostgreSQL
database, waits for health, and exits 0 — or non-zero naming each failure. It checks that:

- `/` answers 200, and an unknown path answers 404 with an HTML body;
- two requests to `/` carry different CSP nonces;
- every header §4.4 requires is present with its required value;
- the image holds no `.env*` file, and the project's secret check passes over the image's
  filesystem as well as the tree;
- the container stops within 10 s.

It uses a fresh database and test placeholders for every secret. Timeouts: healthy within
60 s, each request within 10 s. A timeout fails the test.

### 5.4 Package

| Output | Where it runs |
|---|---|
| Container image from the `Dockerfile` at a tagged commit | Render, built by the host from that tag; or an Oracle VM, pulled or built there |

- `version` in `package.json` is the single source; `/api/health`, the image tag and the
  Git tag read it. CI fails a tag that does not match.
- Auto-deploy on push is off. CI triggers the host's deploy for the tagged commit after
  every check passes — except when anything under `drizzle/` differs from the previous
  tag. CI then stops before deploying, and the human deploys after the backup in §4.3.
- Rollback redeploys the previous tag, and only when nothing under `drizzle/` changed since
  it; otherwise the human decides first (`GLOBAL_RULES.md` §12).
- `README.md` tells users the first visit after idle may take about a minute.

### 5.5 CI

- Push / PR: `pnpm install --frozen-lockfile`, install Chromium on the runner, Biome, both
  type checks, Vitest in Node and
  Chromium including the boundary check; a failure if the change breaks the append-only rule
  of §4.3 — new migration files and new journal entries pass; `drizzle-kit check` and a `drizzle-kit generate`
  that produces no new file, the adapter tests and migrations against a PostgreSQL service
  container, `pnpm build`, Playwright against the built app, then the smoke test against
  the built image.
- Version tag: all of the above → trigger the deploy (unless §5.4 stops it) → confirm that
  `/api/health` reports the tagged version and `/` answers 200.
- The deploy hook URL is set only on the deploy step; no other job or step can read it.

---

## 6. Testing

| Layer | Tool | Framework |
|---|---|---|
| `domain/`, `services/` | Vitest, `node`, fake ports | MUST NOT import React or Next.js |
| `adapters/` | Vitest, `node`, in-memory PGlite migrated from `drizzle/`; PostgreSQL in CI (§4.3) | no |
| Client components | Vitest browser mode, Chromium, `vitest-browser-react` | React only |
| Pages, server actions, sign-in | Playwright against `next build` output, PGlite database | yes |
| Built image | smoke test (§5.3) | yes |

Most tests are in the first two rows. A component or page test proves an input reaches the
right use case and the result is shown — not the rule itself.

- Server components that are `async` cannot be rendered by Vitest; they are covered by
  Playwright. Logic is never left in a server component to dodge a unit test.
- Each Playwright run starts the built app on a fresh PGlite database in a temp dir and
  creates its users through sign-up. Tests construct settings explicitly; a test that
  passes only because the developer's `.env` is present is a defect.
- Every server action's use case states who may call it. One that requires a session is
  tested without one; one limited to the owner of the data is tested as another user; one
  limited to a role is tested without it — each is refused. A public action is tested
  without a session and succeeds.
- Route handlers: the health check is tested without credentials and succeeds; a handler
  for non-browser callers is tested without and with wrong credentials, and both are
  refused. The Better Auth mount is covered by the Playwright sign-up and sign-in flow.
- Controls are found by role and label text, never by CSS selector or id.
- Assert what the user sees: text, `disabled`, `aria-invalid`, the URL after a redirect.
- One test submits input containing markup and asserts it comes back as text.
- A Content-Security-Policy violation reported in the browser fails the Playwright run.
- A request body of exactly 1 MiB is accepted and one of 1 MiB + 1 byte is refused, each
  with and without `Content-Length`, for server actions and for each route handler that
  reads a body.
- Waiting real time — `setTimeout`, `page.waitForTimeout` — is PROHIBITED; wait for an
  element, a URL, or a response.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. Next.js 14/15 idioms: `middleware.ts` exporting `middleware`, synchronous `cookies()`,
   `headers()`, `params` and `searchParams`. Next.js 16 uses `proxy.ts` exporting `proxy`,
   and every one of those is awaited.
2. Zod 3 idioms: `z.string().email()`, `message:` in options, `.flatten()`. Zod 4 uses
   `z.email()`, `error:`, and `z.flattenError()`.
3. Old setup commands: `@better-auth/cli` (the CLI is now the `auth` package), per-package
   `@radix-ui/react-*` imports (shadcn/ui now imports the unified `radix-ui` package), a
   `tailwind.config` path in `components.json`.
4. Copying the testing setup from the Next.js documentation, which uses jsdom and
   `@testing-library/react`. Here client components run in real Chromium through
   `vitest-browser-react`, whose `render` is awaited.
5. Drizzle v1 release-candidate idioms in a project on the stable line, or the reverse:
   `defineRelations()`, one folder per migration with no `meta/_journal.json`. Check the
   installed `drizzle-orm` major and the shape of `drizzle/` before writing relations or
   migration tooling.
6. Moving a page to the Edge runtime, or the app to Cloudflare Workers or Vercel, to escape
   a cold start or a slow page. That is a change of hosting, which needs an ADR, not a
   workaround.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes
- [ ] No server data reaches a client component beyond the fields it displays
- [ ] Every new or changed server action and route handler validates its input and has
      the access tests of §6

Schema change:

- [ ] New migration generated and read by hand, none edited; upgraded from the previous
      migration on a database with rows — on PGlite and on PostgreSQL; `drizzle-kit check`
      clean; a failing migration leaves the previous schema and rows unchanged
- [ ] Better Auth configuration changed → `auth-schema.ts` regenerated in the same commit

Dependencies, `Dockerfile`, `next.config.ts`, `server.ts`, or `config.ts` change:

- [ ] `pnpm build` passes from a frozen install with no secrets set; image builds and the
      smoke test (§5.3) passes
