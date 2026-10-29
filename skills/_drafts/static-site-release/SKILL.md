# static-site-release (draft)

Procedure extracted from `rules/web-static.md` §2 and §5. Draft for step 7 — items marked
**[verify]** were not confirmed against current tool documentation when this was written.

## 1. Machine setup

1. Node: install the LTS major named in `.nvmrc`. Check: `node --version`.
2. pnpm: install the exact version in `package.json` `packageManager`, standalone.
   Node 25+ does not bundle Corepack, so `corepack enable` is not a setup step.
   - `npm install -g pnpm@<version>` works anywhere npm exists. **[verify]** pnpm's own
     standalone installer as the preferred path.
   - Check: `pnpm --version` equals the `packageManager` version.
3. `pnpm install`. If it stops on an install-script error, add only the named package to
   `allowBuilds` in `pnpm-workspace.yaml`, after confirming why it needs a script.

## 2. Project skeleton (walking skeleton, first week)

- `vite.config.ts`:
  ```ts
  import { defineConfig } from "vite";
  import tailwindcss from "@tailwindcss/vite";
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
  });
  ```
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
- First deploy of the skeleton follows §4, then the human checks the `*.workers.dev` URL.

## 3. Cloudflare account and token

1. Create a free Cloudflare account; note the account ID.
2. Create an API token with only the permission needed to deploy Workers on that account
   **[verify]** exact permission name (Account → Workers Scripts → Edit).
3. Add `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` as GitHub Actions secrets. List
   both in `.env.example` with placeholders. Never in `wrangler.jsonc`, never local files
   committed.
4. Do not connect the repository to Cloudflare's Git builds.

## 4. CI workflow

- On push / PR:
  1. checkout
  2. setup Node from `.nvmrc` (`actions/setup-node`, `node-version-file`)
  3. setup pnpm from `packageManager` **[verify]** current action: `pnpm/action-setup` or
     `pnpm/setup` (v2 requires pnpm 11+)
  4. `pnpm install --frozen-lockfile`
  5. `pnpm exec biome ci .`
  6. `pnpm exec tsc --noEmit -p tsconfig.json && pnpm exec tsc --noEmit -p tsconfig.lib.json`
  7. `pnpm exec vitest run`
  8. `pnpm build`
  9. smoke test (`pnpm exec vitest run tests/smoke` or a dedicated script)
- On tag `v*`: all of the above, then
  1. fail if the tag ≠ `v` + `package.json` `version`
  2. `pnpm exec wrangler deploy` with the two secrets in the environment
  3. post-deploy check (§5)
  4. on any failure: stop, report, no retry, no redeploy

## 5. Post-deploy check

Against the deployed URL:

- `GET /version.txt` → body equals the tag's version
- `GET /` → 200
- `GET /<random-string>/` → 404 and a non-empty HTML body
- `GET /` → response has `Content-Security-Policy`

A verifier that fails intermittently is suspected before the host: test it against a
known-good page.

## 6. Rollback

- `pnpm exec wrangler rollback` to the previous version (Workers keeps the last 100), or
  re-run the tag workflow for the previous tag.
- After either, repeat §5 with the previous version number.
- Note any URL renamed since that version: its `_redirects` line is gone with the rollback.

## 7. Smoke test outline

Reads `dist/` only; no server, no network.

1. Parse the build input list from `vite.config.ts` (or export `pages` from a shared module).
2. For each built `.html`: collect `href`, `src`, `srcset`; skip `#…`, `mailto:`, `tel:`;
   fail on another origin in `src` / `<link href>`; map `/x/` → `dist/x/index.html` and
   `/a.css` → `dist/a.css`; fail if missing.
3. Fail on `<script>` without `src`, on any `on[a-z]+=` attribute.
4. Check `dist/_headers` contains `Content-Security-Policy`.
5. Check `dist/version.txt`.
6. Count files; fail above 20,000 or on any file above 25 MiB.
