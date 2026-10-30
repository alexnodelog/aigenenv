# static-site-release (draft)

Procedure extracted from `rules/web-static.md` §2 and §5. Draft for step 7: frontmatter
(`name`, `description`) is added then. Items marked **[verify]** were not confirmed against
current tool documentation when this was written.

## 1. Machine setup

1. Install pnpm standalone, first — it does not need Node.
   - macOS / Linux: `curl -fsSL https://get.pnpm.io/install.sh | sh -`
   - Windows (PowerShell): `Invoke-WebRequest https://get.pnpm.io/install.ps1 -UseBasicParsing | Invoke-Expression`
   - **[verify]** both commands, and how to install the exact `packageManager` version.
   - Check: `pnpm --version` equals the `packageManager` version.
2. Node is not installed separately. `package.json` declares it:
   ```json
   "devEngines": {
     "runtime": { "name": "node", "version": "<LTS version>", "onFail": "download" }
   }
   ```
   `pnpm install` downloads that Node. **[verify]** accepted version syntax (exact vs range).
   Check: `pnpm exec node --version`.
3. `pnpm install`. If it stops on an install-script error, add only the named package to
   `allowBuilds` in `pnpm-workspace.yaml`, after confirming why it needs a script.
4. `pnpm exec playwright install chromium` — once per machine; tests then run offline.

## 2. Project skeleton (walking skeleton, first week)

- `vite.config.ts` — build input and test projects:
  ```ts
  import { defineConfig } from "vitest/config";
  import tailwindcss from "@tailwindcss/vite";
  import { playwright } from "@vitest/browser-playwright";
  import { resolve } from "node:path";

  const pages = ["index.html", "404.html"]; // add every page here

  export default defineConfig({
    base: "/",
    plugins: [tailwindcss()],
    build: {
      rolldownOptions: {
        input: Object.fromEntries(pages.map((p) => [p, resolve(__dirname, p)])),
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
  **[verify]** Vitest 4 `projects` and browser-provider syntax; whether `smoke` should be a
  separate script so `vitest run` before `build` does not fail on a missing `dist/`.
- UI test loading a real page: `import html from "../../about/index.html?raw"`, put its
  `<body>` into `document.body`, then call the page's `mount()`. **[verify]** `?raw` on
  HTML files in browser mode.
- `src/styles/app.css`: `@import "tailwindcss";`
- `version.txt`: a small Vite plugin (`generateBundle` → `emitFile`) writes
  `package.json` `version`. **[verify]** hook name under Rolldown.
- `tsconfig.lib.json`: `"include": ["src/lib"]`, `"lib": ["ES2023"]`, `"types": []`.
- `wrangler.jsonc`:
  ```jsonc
  {
    "name": "<site-name>",
    "compatibility_date": "<today>",
    "assets": {
      "directory": "./dist",
      "html_handling": "auto-trailing-slash",
      "not_found_handling": "404-page"
    }
  }
  ```
  No `main`. **[verify]** whether to set `workers_dev` / `preview_urls` explicitly.
- `public/_headers` (template; adjust `connect-src` to the origins the site fetches):
  ```
  /*
    Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'; connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'none'; frame-ancestors 'none'
    X-Content-Type-Options: nosniff
    Referrer-Policy: strict-origin-when-cross-origin
  /assets/*
    Cache-Control: public, max-age=31536000, immutable
  ```
  **[verify]** whether Vite's build output needs any `style-src` relaxation.
- First deploy of the skeleton follows §4, then the human opens the `*.workers.dev` URL.

## 3. Cloudflare account and token

1. Create a free Cloudflare account; note the account ID.
2. Create an API token with only the permission needed to deploy Workers on that account.
   **[verify]** exact permission name (Account → Workers Scripts → Edit).
3. Add `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` as GitHub Actions secrets. List
   both in `.env.example` with placeholders. Never in `wrangler.jsonc` or any committed file.
4. Do not connect the repository to Cloudflare's Git builds.

## 4. CI workflow

- On push / PR:
  1. checkout
  2. set up pnpm from `packageManager` — no separate Node setup step; pnpm installs Node from
     `devEngines.runtime`. **[verify]** current action (`pnpm/setup` v2 for pnpm 11+) and
     whether it runs the runtime download.
  3. `pnpm install --frozen-lockfile`
  4. `pnpm exec playwright install --with-deps chromium` (cache the browser directory)
  5. `pnpm exec biome ci .`
  6. `pnpm exec tsc --noEmit -p tsconfig.json && pnpm exec tsc --noEmit -p tsconfig.lib.json`
  7. `pnpm exec vitest run --project lib --project ui`
  8. `pnpm build`
  9. `pnpm exec vitest run --project smoke`
- On tag `v*`: all of the above, then
  1. fail if the tag ≠ `v` + `package.json` `version`
  2. `pnpm exec wrangler deploy`, the two secrets set in `env:` of this step only — never at
     workflow or job level
  3. post-deploy check (§5)
  4. on any failure: stop, report, no retry, no redeploy

## 5. Post-deploy check

Against the deployed URL:

- `GET /version.txt` → body equals the tag's version
- `GET /` → 200
- `GET /<random-string>/` → 404 and a non-empty HTML body
- `GET /` → `Content-Security-Policy` value equals the `/*` entry in `dist/_headers`

A verifier that fails intermittently is suspected before the host: test it against a
known-good page.

## 6. Rollback

- `pnpm exec wrangler rollback` to the previous version, or re-run the tag workflow for the
  previous tag.
- After either, repeat §5 with the previous version number.
- Note any URL renamed since that version: its `_redirects` line is gone with the rollback.

## 7. Smoke test outline

Reads `dist/` only; no server, no network.

1. Read the page list from the same `pages` array the build uses (export it from a shared
   module).
2. For each built `.html`: collect `href`, `src`, `srcset`; skip `#…`, `mailto:`, `tel:`;
   fail on another origin in `src` / `<link href>`; map `/x/` → `dist/x/index.html` and
   `/a.css` → `dist/a.css`; fail if missing.
3. Fail on `<script>` without `src`, on any `on[a-z]+=` attribute.
4. Parse the `/*` CSP in `dist/_headers`: each directive required by the rule present with
   its required value; fail on `'unsafe-inline'` or `'unsafe-eval'`.
5. Fail on any `*.map`, `.env*` or `*.md` file in `dist/`.
6. Check `dist/version.txt`.
7. Count files; fail above 20,000 or on any file above 25 MiB.
