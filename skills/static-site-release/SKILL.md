---
name: static-site-release
description: Set up the toolchain, create the project skeleton (vite.config.ts, wrangler.jsonc, public/_headers, version.txt, tsconfig files), write the smoke test, configure CI, and deploy or roll back a web-static site on Cloudflare Workers. Use when creating a web-static project, editing vite.config.ts, astro.config.mjs, wrangler.jsonc, pnpm-workspace.yaml, public/_headers or tests/smoke/, when pnpm install fails with ERR_PNPM_IGNORED_BUILDS, when a CI workflow is added or changed, or when cutting, checking or rolling back a release.
---

# static-site-release

Procedures for `rules/web-static.md` §2 and §5. The pnpm version is always the one in
`package.json` `packageManager`; `<v>` below stands for it.

## 1. Machine setup (rule §2)

1. Install pnpm standalone, first. It needs no Node.
   - macOS / Linux: `curl -fsSL https://get.pnpm.io | PNPM_VERSION=<v> sh -`
   - Windows (PowerShell): `$env:PNPM_VERSION='<v>'; iwr https://get.pnpm.io/ps1 -useb | iex`
   - Open a new terminal. Check: `pnpm --version` prints `<v>`.
2. Node is not installed separately. `package.json` declares an exact LTS version:

   ```json
   "devEngines": {
     "runtime": { "name": "node", "version": "<exact LTS version>", "onFail": "download" }
   }
   ```

   `pnpm install` downloads it. Check: `pnpm exec node --version`.
3. `pnpm install`. If it fails with `ERR_PNPM_IGNORED_BUILDS`, follow §2.3.
4. `pnpm exec playwright install chromium`, once per machine; tests then run offline. On a
   network that blocks `cdn.playwright.dev` this step fails: report it; do not switch the
   tests to jsdom.

## 2. Project skeleton

Before any feature beyond it, the skeleton — `index.html`, `404.html`, `_headers` — is
deployed to the real host (rule §5.2).

### 2.1 Dependencies

```
pnpm add -D -E vite vitest @vitest/browser-playwright playwright tailwindcss @tailwindcss/vite typescript @biomejs/biome wrangler @types/node
pnpm exec biome init
```

### 2.2 `package.json` scripts

```json
"scripts": {
  "build": "tsc --noEmit -p tsconfig.json && tsc --noEmit -p tsconfig.lib.json && vite build",
  "test": "vitest run --project lib --project ui",
  "test:smoke": "vitest run --project smoke"
}
```

`test:smoke` runs only after `pnpm build`.

### 2.3 `pnpm-workspace.yaml`

Start with `allowBuilds: {}`. pnpm fails the install when a dependency's install script
is neither allowed nor denied. For each package `ERR_PNPM_IGNORED_BUILDS` names:

- Deny it (`false`) unless the site fails without its script. With the dependencies of
  §2.1, wrangler brings `esbuild` and `workerd`; neither is needed to build or deploy an
  assets-only site:

  ```yaml
  allowBuilds:
    esbuild: false
    workerd: false
  ```

- Allow it (`true`) only after confirming why it needs a script, and say so in the commit
  message.

### 2.4 `pages.ts` and `vite.config.ts`

`pages.ts` is the one list of pages; the build and the smoke test both import it.

```ts
// pages.ts
export const pages = ["index.html", "404.html"]; // add every page here
```

```ts
// vite.config.ts
import { readFileSync } from "node:fs";
import { resolve } from "node:path";
import tailwindcss from "@tailwindcss/vite";
import { playwright } from "@vitest/browser-playwright";
import type { Plugin } from "vite";
import { defineConfig } from "vitest/config";
import { pages } from "./pages.ts";

const root = import.meta.dirname;
const pkg = JSON.parse(readFileSync(resolve(root, "package.json"), "utf-8"));

function versionFile(): Plugin {
  return {
    name: "version-file",
    apply: "build",
    generateBundle() {
      this.emitFile({ type: "asset", fileName: "version.txt", source: `${pkg.version}\n` });
    },
  };
}

export default defineConfig({
  base: "/",
  plugins: [tailwindcss(), versionFile()],
  build: {
    rolldownOptions: {
      input: Object.fromEntries(pages.map((p) => [p, resolve(root, p)])),
    },
  },
  test: {
    projects: [
      {
        extends: true,
        test: {
          name: "lib",
          environment: "node",
          include: ["tests/lib/**/*.test.ts", "tests/boundaries.test.ts"],
        },
      },
      {
        extends: true,
        test: {
          name: "ui",
          include: ["tests/ui/**/*.test.ts"],
          browser: {
            enabled: true,
            headless: true,
            provider: playwright(),
            instances: [{ browser: "chromium" }],
          },
        },
      },
      {
        extends: true,
        test: { name: "smoke", environment: "node", include: ["tests/smoke/**/*.test.ts"] },
      },
    ],
  },
});
```

Use `import.meta.dirname`, not `__dirname`, and keep the `.ts` extension on local
imports: Vite warns that the other forms break under its upcoming native config loader.

### 2.5 TypeScript

- `tsconfig.json`: `"strict": true`, `"module": "ESNext"`, `"moduleResolution": "bundler"`,
  `"noEmit": true`, `"allowImportingTsExtensions": true`, `"lib": ["ES2023", "DOM"]`,
  `"types": ["vite/client", "node"]`; includes `src`, `tests`, `vite.config.ts`, `pages.ts`.
- `tsconfig.lib.json`: the same compiler options except `"lib": ["ES2023"]`,
  `"types": []`; `"include": ["src/lib"]`.

### 2.6 Styles and pages

- `src/styles/app.css`: `@import "tailwindcss";`
- Each HTML file links `/src/styles/app.css` and loads its one `src/entry/<page>.ts` with
  `<script type="module" src>`.
- A UI test loads the real page:

  ```ts
  import html from "../../about/index.html?raw";
  import { mount } from "../../src/ui/about";

  const doc = new DOMParser().parseFromString(html, "text/html");
  document.body.innerHTML = doc.body.innerHTML; // test setup only, never in src/
  mount(document);
  ```

### 2.7 `wrangler.jsonc`

```jsonc
{
  "name": "<site-name>",
  "account_id": "<account id>",
  "compatibility_date": "<today>",
  "workers_dev": true,
  "preview_urls": false,
  "assets": {
    "directory": "./dist",
    "html_handling": "auto-trailing-slash",
    "not_found_handling": "404-page"
  }
}
```

No `main`. `workers_dev` and `preview_urls` are written out even though they are
wrangler's defaults, so a changed default cannot publish preview uploads.

### 2.8 `public/_headers`

`connect-src 'none'` until the site fetches something, then exactly those origins:

```
/*
  Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self' data:; connect-src 'none'; object-src 'none'; base-uri 'self'; form-action 'none'; frame-ancestors 'none'
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

Vite's build emits no inline script or style, so the policy needs no relaxation.

### 2.9 Check the skeleton

`pnpm build`, then `pnpm test:smoke`, then `pnpm exec wrangler deploy --dry-run`. It needs
no account and lists the files it would upload.

## 3. Astro variant

- `pnpm create astro` with the minimal template. No adapter.
- `astro.config.mjs`:

  ```js
  import { readFileSync, writeFileSync } from "node:fs";
  import { fileURLToPath } from "node:url";
  import tailwindcss from "@tailwindcss/vite";
  import { defineConfig } from "astro/config";

  const pkg = JSON.parse(readFileSync(new URL("./package.json", import.meta.url), "utf-8"));

  const versionFile = {
    name: "version-file",
    hooks: {
      "astro:build:done": ({ dir }) => {
        writeFileSync(fileURLToPath(new URL("version.txt", dir)), `${pkg.version}\n`);
      },
    },
  };

  export default defineConfig({
    output: "static",
    build: { inlineStylesheets: "never" },
    integrations: [versionFile],
    vite: { plugins: [tailwindcss()] },
  });
  ```

  `inlineStylesheets: "never"` is required: by default Astro inlines small stylesheets as
  `<style>`, which the CSP blocks.
- Scripts: `"check": "astro check && tsc --noEmit -p tsconfig.lib.json"`,
  `"build": "astro build"`. Output goes to `dist/`, so §2.7, §2.8, §6 and §7 stay as they
  are.
- `vitest.config.ts` reads Astro's config:

  ```ts
  import { getViteConfig } from "astro/config";
  export default getViteConfig({ test: { projects: [/* the three projects of §2.4 */] } });
  ```

## 4. Cloudflare account and token

1. Create a free Cloudflare account. Copy the account ID into `wrangler.jsonc`; it is an
   identifier, not a secret.
2. Create a custom API token: permission **Account → Workers Scripts → Edit**, account
   resources limited to this one account. Do not use the "Edit Cloudflare Workers"
   template: it grants more than an assets-only deploy needs.
3. Add `CLOUDFLARE_API_TOKEN` as a GitHub Actions secret. It is the only secret. List it in
   `.env.example` with a placeholder.
4. Do not connect the repository to Cloudflare's Git builds.
5. If a deploy fails with `Authentication error [code: 10000]`, stop and report the API path
   the error names. Do not widen the token without the human.

## 5. CI workflow

Every `uses:` is pinned to a full commit SHA with the tag as a comment. Look the SHA up with
`git ls-remote https://github.com/<owner>/<action> refs/tags/<tag>^{}`.

```yaml
permissions:
  contents: read
```

On push / PR:

1. `actions/checkout`
2. `pnpm/setup` (v3): installs the pnpm of `packageManager` and the Node of
   `devEngines.runtime`. Set `install: false`; the next step installs.
3. `pnpm install --frozen-lockfile`
4. `pnpm exec playwright install --with-deps chromium`
5. `pnpm exec biome ci .`
6. `pnpm build` (runs both type checks)
7. `pnpm test`
8. `pnpm test:smoke`

On tag `v*`, all of the above, then:

1. Fail if the tag is not `v` + `package.json` `version`, or if the tagged commit is not on
   `main`: `git fetch origin main` then `git merge-base --is-ancestor <sha> origin/main`.
2. `pnpm exec wrangler deploy`, with `CLOUDFLARE_API_TOKEN` in the `env:` of this step only.
3. The checks of §6.
4. On any failure: stop and report. No retry and no redeploy. A failed §6 check means the
   wrong site may be live: the report names the failed check and the rollback command of
   §7, and the human decides.

## 6. Post-deploy check

Against the URL `wrangler deploy` printed:

- `GET /version.txt`: body equals the tag's version
- `GET /`: 200
- `GET /<random string>/`: 404 with a non-empty HTML body
- `GET /`: the `Content-Security-Policy` value equals the `/*` entry in `dist/_headers`

When the check fails intermittently, suspect the check before the host: run it against a
known-good page.

## 7. Rollback

Rolling back is deploying: the human approves it.

1. `pnpm exec wrangler deployments list` shows the versions.
2. `pnpm exec wrangler rollback <version-id> -m "<reason>"`. Without an ID it returns to
   the previous version. Or re-run the tag workflow for the previous tag.
3. Repeat §6 with the previous version number.
4. Report any URL renamed since that version: its `_redirects` line is gone with the
   rollback.

## 8. Smoke test outline

`tests/smoke/`, reads `dist/` only; no server, no network.

1. Read the page list from `pages.ts` (Astro: the files in `src/pages/`).
2. Collect references: `href`, `src`, `srcset` from every built `.html`; `url(...)` and
   `@import` from every built `.css` (CSS paths resolve relative to the CSS file). Skip
   `#…`, `data:`, `mailto:`, `tel:`. Fail on another origin. Map `/x/` to
   `dist/x/index.html` and `/a.css` to `dist/a.css`; fail if the file is missing.
3. Fail on a `<script>` without `src`, a `<style>` element, a `style=` attribute, or any
   `on[a-z]+=` attribute. The CSP blocks all four, so the page would break silently.
4. Parse the `/*` block of `dist/_headers`: every directive rule §4.4 requires, with its
   required value; no `'unsafe-inline'` or `'unsafe-eval'`; `nosniff`;
   `Referrer-Policy: strict-origin-when-cross-origin`.
5. Fail on any `*.map`, `.env*` or `*.md` file in `dist/`. Run the project's secret scanner
   over `dist/`.
6. `dist/version.txt` equals `package.json` `version`.
7. Fail above 20,000 files or on any file above 25 MiB.
