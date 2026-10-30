# rules/web-static.md

Website with no server logic. Every page is an HTML file built once and served as-is from a
CDN; what runs, runs in the visitor's browser.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if anything must happen on a server: a form whose contents must arrive
somewhere, a call that needs a secret, a login, data shared between visitors
(`web-python` or `web-ts`). Wrong archetype if the UI keeps client-side state across views
or needs a component framework (`web-ts`).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Node.js | An LTS line, installed by pnpm, pinned in `package.json` `devEngines.runtime` | — | Odd-numbered (non-LTS) majors; `.nvmrc`, nvm, fnm, Volta as the version record |
| Packaging | pnpm, standalone install, exact version in `package.json` `packageManager`; `pnpm-lock.yaml` | — | npm, yarn, bun; Corepack; `npx` in scripts or docs |
| Language | TypeScript, `strict: true`, ES modules | — | Plain JavaScript in new code |
| Build | Vite, multi-page: one HTML file per URL | Astro with static output, for content-heavy sites built from Markdown | React, Vue, Svelte or any component framework, including as Astro islands; a client-side router; Astro server output or any adapter; webpack, Parcel |
| Styling | Tailwind CSS v4 via `@tailwindcss/vite` | Plain CSS | `tailwind.config.js`, `@tailwind` directives, a PostCSS config for Tailwind; the Tailwind Play CDN; CSS-in-JS |
| Interactivity | Standard DOM APIs | Alpine.js CSP build (`@alpinejs/csp`), bundled | Alpine's standard build (needs `unsafe-eval`); jQuery; htmx — there is no server to answer it |
| Third-party runtime resources | None: every script, stylesheet, font and image ships in `dist/` | — | `<script>` or `<link>` to a CDN; Google Fonts links; tracking or analytics tags |
| Browser storage | `localStorage` (§4.3) | IndexedDB | Cookies for application data |
| Lint / format | Biome | ESLint + Prettier | — |
| Type check | `tsc --noEmit` on both configs (§3); Astro: `astro check` | — | Treating a passing `vite build` as a type check |
| Tests | Vitest; DOM code runs in real Chromium through Vitest browser mode (`@vitest/browser-playwright`) | — | jsdom, happy-dom; Jest |
| Hosting | Cloudflare Workers, assets only: `wrangler.jsonc` with `assets` and no `main` | Netlify Free; GitHub Pages — public repository and non-commercial sites only | Vercel Hobby; a Worker script, Pages Functions or any edge function; form-backend services; `not_found_handling = "single-page-application"` |

---

## 2. Prerequisites

`pnpm install` neither installs nor reveals these. Procedure: skill `static-site-release`.

| Item | Scope |
|---|---|
| pnpm | System, standalone, installed first. The exact version in `packageManager` |
| Node.js | Installed by pnpm from `devEngines.runtime`. No separate Node installer |
| Cloudflare account | Account. Free plan |
| Cloudflare API token | Account. Stored only as a CI secret, scoped to deploying Workers on one account |
| Chromium for tests | Project, downloaded once by `pnpm exec playwright install chromium`; tests then run offline |

---

## 3. Layout

```
<repo>/
├── package.json          version, packageManager, devEngines.runtime, scripts
├── pnpm-lock.yaml
├── pnpm-workspace.yaml   pnpm settings: allowBuilds
├── tsconfig.json         whole project, strict
├── tsconfig.lib.json     src/lib/ only, no DOM library
├── vite.config.ts        build input (every page); Vitest projects: lib in Node, ui in Chromium
├── biome.json
├── wrangler.jsonc        assets only; no main
├── index.html            one HTML file per URL: <name>/index.html serves /<name>/
├── 404.html
├── <name>/index.html
├── public/               copied verbatim: _headers, _redirects, robots.txt, favicon, fonts
├── src/
│   ├── styles/app.css    Tailwind entry
│   ├── config.ts         the only reader of import.meta.env
│   ├── lib/              pure TypeScript: calculation, parsing, validation, formatting
│   ├── ui/               one module per page: export mount(root); reads and writes the DOM
│   └── entry/            one file per page, one line: mount(document)
├── tests/                lib/, ui/, smoke/, boundaries.test.ts
└── dist/                 gitignored; the whole deliverable
```

- A content-only site has no `src/lib/`, `src/ui/` or `src/entry/`, and MUST NOT add them
  to match this layout. A new top-level dir needs an ADR only if it changes the boundaries
  below.
- A page's HTML loads exactly one script: its `src/entry/<page>.ts`. Importing a `ui/`
  module MUST NOT touch the DOM; only `mount()` does.
- Astro: pages live in `src/pages/`; `src/lib/` and `src/ui/` keep the same boundaries.

| Directory | MAY import | MUST NOT import or use |
|---|---|---|
| `src/lib/` | `src/lib/`, packages that need no DOM | `src/ui/`, `src/config.ts`, Alpine; DOM globals, `fetch`, `localStorage` |
| `src/ui/` | `src/lib/`, `src/config.ts`, DOM APIs, Alpine | `src/entry/` |
| `src/entry/` | its own page's `src/ui/` module | anything else |

- `tsconfig.lib.json` includes only `src/lib/`, with no `DOM` in `lib` and `types: []`, so
  DOM use in `lib/` fails the type check. `tests/boundaries.test.ts` fails on any import in
  `src/lib/` that leaves `src/lib/` or names Alpine. These two are the boundary check CI and
  §8 refer to.
- A `ui/` handler reads input, calls `lib/`, and renders the result. A calculation or rule
  written in `ui/` is a defect.

---

## 4. Runtime

### 4.1 Page and script model

- Every URL is an HTML file the build emits. Routing in the browser — History API
  rewrites, hash routes — is PROHIBITED: it is a change of archetype.
- Content is in the HTML. A content page MUST be readable with JavaScript disabled; a page
  that cannot work without it says so in `<noscript>`.
- Scripts arrive only as `<script type="module" src>` from the build. Inline `<script>`,
  `on*=` attributes and `javascript:` URLs are PROHIBITED; the CSP (§4.4) blocks them.
- Data reaches the DOM through `textContent`, `setAttribute` and `createElement`.
  `innerHTML`, `outerHTML` and `insertAdjacentHTML` with any string that is not a
  compile-time constant are PROHIBITED.
- A `<form>` computes and shows its result in the page: it has no `action`, and its `submit`
  handler calls `preventDefault()`.
- Controls are native `<button>`, `<a>`, `<input>`, `<select>`; every input has a `<label>`.
  A clickable `<div>` is a defect.
- `fetch` reaches only public HTTPS endpoints that need no key and send CORS headers, each
  listed in the CSP `connect-src`. Every call has a timeout (`AbortSignal.timeout`) and a
  visible failure state, and its response is validated by `lib/` before use.

### 4.2 Paths

- Pages are `<name>/index.html`, served at `/<name>/`. Internal links use that form,
  trailing slash included; the host redirects every other form.
- Assets are referenced through Vite (an import, or a `/src/...` path in HTML) or from
  `public/`. Vite `base` is `/`, or the sub-path a GitHub Pages project site requires. A
  relative `base` (`./`) is PROHIBITED: the 404 page and nested pages resolve assets
  against the wrong directory.
- `404.html` exists at the root and is served with status 404
  (`not_found_handling = "404-page"`). A fallback that answers unknown paths with 200 hides
  every broken link.
- A published URL is a contract. Renaming or merging a page adds a `301` line to
  `public/_redirects` in the same commit, pointing at the page that replaced it; the file's
  default status is 302. A page removed with no replacement answers 404 — it is not
  redirected to the home page.
- Everything in `dist/` is public, including what preview deployments upload. There is no
  access control: a draft, an internal note or an "unlisted" page MUST NOT be in it.

### 4.3 Persistence

- The site keeps no data on any server and nothing about its visitors.
- Browser storage holds preferences and work in progress. Data the user cannot recreate
  MUST NOT live only there: browsers clear it. The page offers export to a file and import
  from one, or the project is not static.
- Keys are prefixed with the site's name. Each stored value is JSON carrying a `schema`
  number and is read through a `lib/` parser that validates it.
- A value the page cannot read — malformed, or a `schema` newer than it knows, as after a
  rollback — stays untouched under its key. The page runs on defaults in memory, does not
  write to that key, and tells the user, offering the stored value for export. Only the
  user's discard, confirmed in a second step, replaces it. It MUST NOT break the page.
- A change to a stored shape increments `schema`, and `lib/` converts the previous shape.
- Storage access can throw — private browsing, a full quota, storage disabled. The page
  catches it and keeps working without persistence.

### 4.4 Settings, credentials, errors

- Build-time settings come from `import.meta.env.VITE_*`, read only in `src/config.ts` and
  validated there for presence, type and format; a missing or empty required value, or
  one in the wrong format, fails the build. Every `VITE_` value ships to every visitor,
  so none of them is a secret. Widening Vite's `envPrefix` is PROHIBITED.
- The site has no deployment secrets at runtime. The host's deploy token is the project's
  only secret and exists only in CI.
- There are no accounts and no credentials. A password checked in JavaScript, content hidden
  by CSS or script, and an unguessable URL are not access control; content that needs it
  needs a server.
- `public/_headers` sets, for `/*`, a Content-Security-Policy with exactly these
  directives: `default-src 'self'`, `script-src 'self'`, `style-src 'self'`,
  `img-src 'self' data:`, `font-src 'self' data:` (Vite inlines small assets as `data:`
  URLs), `object-src 'none'`, `base-uri 'self'`, `form-action 'none'`,
  `frame-ancestors 'none'`, and `connect-src` listing exactly the origins the site fetches
  (`'none'` if it fetches nothing); no `'unsafe-inline'` or `'unsafe-eval'` anywhere. Also
  `X-Content-Type-Options: nosniff` and `Referrer-Policy: strict-origin-when-cross-origin`.
- A host that cannot set response headers is allowed only for a content-only site; there
  the CSP goes in a `<meta http-equiv>` on every page, without `frame-ancestors`, which a
  meta tag cannot carry.
- Invalid input is reported next to its field, in words. An unexpected exception is caught
  at `mount()` and the page shows what failed and what to try. No error-reporting service
  is used by default.

---

## 5. Build and package

Procedure: skill `static-site-release`.

### 5.1 Lock the toolchain

- `packageManager` and `devEngines.runtime` (with `onFail: "download"`) are the single
  source for the pnpm and Node versions; CI installs from both. Every install outside a developer's first setup uses
  `pnpm install --frozen-lockfile`.
- Dependency install scripts run only for packages named in `allowBuilds` in
  `pnpm-workspace.yaml`. Allowing every package, or setting `minimumReleaseAge` to 0 to get
  a just-published version, is PROHIBITED.
- wrangler is a locked devDependency and every deploy runs that copy.

### 5.2 Build

- `pnpm build` runs `tsc --noEmit` on both configs, then `vite build`. Vite strips types
  without checking them.
- Every page, `404.html` included, is listed in `build.rolldownOptions.input`
  (`rollupOptions` is Vite 8's deprecated alias). The dev server serves any HTML file; the
  build emits only listed ones.
- Tailwind v4 is configured in `src/styles/app.css` (`@import "tailwindcss";`). There is no
  `tailwind.config.js`.
- Astro: `astro check` replaces the `tsconfig.json` check (`tsconfig.lib.json` is still
  checked), and `astro build` replaces `vite build`. Every file in `src/pages/` is a page,
  so there is no input list; wherever this file says "build input", Astro reads
  `src/pages/`.
- Fonts are self-hosted, from a package or `public/`.
- `dist/` stays within the host's per-deployment limits — on Workers Free, 20,000 files and
  25 MiB per file. Media that does not fit needs an ADR naming where it lives.
- A project that will be deployed deploys a walking skeleton — `index.html`, `404.html`,
  `_headers`, the real host — before any feature beyond it is built.

### 5.3 Self-check

The smoke test (`tests/smoke/`) runs after `pnpm build`, against `dist/` on disk, offline,
and exits 0 — or non-zero naming each failure. It checks that:

- every page in the build input, and `404.html`, exists in `dist/`;
- every local reference — `href`, `src`, `srcset` in built HTML; `url()` and `@import` in
  built CSS — resolves to a file in `dist/`;
- no built HTML contains an inline `<script>` or an `on*=` attribute;
- no URL on another origin appears in HTML `src`, `srcset` or `<link href>`, or in built
  CSS `url()` or `@import`;
- `dist/_headers` sets every header and directive §4.4 requires, with its required value;
- `dist/` holds no `.map`, `.env*` or Markdown file, and the project's secret check passes
  over `dist/` as well as the tree;
- `dist/version.txt` holds the `version` from `package.json`;
- `dist/` is within the limits of §5.2.

### 5.4 Package

| Output | Where it runs |
|---|---|
| `dist/`, built by CI at a tagged commit | Cloudflare Workers, uploaded by `wrangler deploy` |

- `version` in `package.json` is the single source; `version.txt` and the Git tag read it.
  CI fails a tag that does not match.
- The host's Git-connected builds stay off. CI deploys the tagged commit only if it is on
  `main` and every check passes.
- Rollback is `wrangler rollback` to the previous version, or a redeploy of the previous
  tag. It also rolls back `_redirects`, so a URL renamed since then breaks again.

### 5.5 CI

- Push / PR: `pnpm install --frozen-lockfile`, Biome, the type checks and build of §5.2,
  Vitest including the boundary check, the smoke test.
- Version tag: all of the above → deploy → confirm that `/version.txt` matches the tag, `/`
  answers 200, a random path answers 404 with an HTML body, and the CSP header on `/` equals
  the one in `_headers`.
- The deploy token is set only on the deploy step; no other job or step can read it.
  Install, build and test run third-party code.

---

## 6. Testing

| Layer | Tool | DOM |
|---|---|---|
| `src/lib/` | Vitest, `node` environment | MUST NOT be available |
| `src/ui/` | Vitest browser mode, Chromium | yes, a real browser |
| Built site | smoke test (§5.3) | — |

Most tests are in the first row. A UI test proves an input reaches the right `lib/` function
and the result is shown — not the rule itself.

- A UI test loads the page's real HTML file into the document, then calls `mount()`. Markup
  written by hand in a test is PROHIBITED: the test then passes against a page that does not
  exist.
- Controls are found by role and label text, never by CSS selector or id, so an input
  without a working `<label>` fails its test.
- Assert what the user sees: text, `disabled`, `hidden`, `aria-invalid`.
- `fetch` is stubbed per test and restored after it. `localStorage` is cleared before each
  test, and the path where storage throws is tested.
- Time is controlled with fake timers. Waiting real time is PROHIBITED.
- One test renders input containing markup and asserts it comes back as text.

---

## 7. Where agents go wrong

The mistakes agents most often make here. Each breaks the rule in the section named.

1. Reaching for React, a router, or a component framework for a few interactions (§1,
   §4.1).
2. Tailwind v3 habits: `tailwind.config.js`, `@tailwind base;`, a PostCSS config (§1,
   §5.2).
3. An API key in a `VITE_` variable or in source, "because it is only a public key" (§4.4).
4. A password prompt or hidden section treated as access control (§4.4).
5. `innerHTML` built from user input or a fetched response (§4.1).
6. A new page that works in the dev server and is missing from `dist/` — not in the build
   input (§5.2).
7. An SPA not-found fallback, or no `404.html`, so broken links answer 200 (§4.2).
8. A CDN `<script>`, a Google Fonts `<link>`, or an analytics tag pasted into a page (§1).
9. A Worker script or edge function added to make a form submit or hide a key. That is a
   change of archetype, which needs an ADR, not a workaround (§1).
10. pnpm 9/10 settings (`onlyBuiltDependencies`) or "allow all builds" to silence the
    install-script error; `.nvmrc`, Corepack, `npm install` or `npx` in docs and CI (§1,
    §5.1).
11. "`vite build` passed, so the types are fine" (§5.2).
12. A page renamed without a `_redirects` line (§4.2).
13. "The deploy command succeeded, so it works." Check `/version.txt` and the 404 (§5.5).
14. Switching UI tests to jsdom "because it is faster" (§1, §6).

---

## 8. Definition of Done — additions

Every task here:

- [ ] Build and smoke test pass
- [ ] No script, stylesheet, font or image from another origin; every `fetch` origin is in
      `connect-src`

Any change under `src/lib/`, `src/ui/` or `src/entry/`:

- [ ] Boundary check (§3) passes
- [ ] No inline script, `on*=` attribute, or non-constant string passed to `innerHTML`

New, renamed, or removed page:

- [ ] In the build input (Astro: in `src/pages/`); a renamed URL answers 301 to its
      replacement

Stored-data shape change:

- [ ] `schema` incremented; tested reading the previous shape, a malformed value, a newer
      `schema` (left untouched), and storage that throws

Dependencies, build config, `wrangler.jsonc`, `_headers`, or `_redirects` change:

- [ ] `pnpm build` and the smoke test pass from a frozen install
