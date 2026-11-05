---
name: electron-release
description: Set up the toolchain, serve the renderer from the custom scheme, configure electron-vite and electron-builder (targets, fuses), implement --self-check, run whole-app tests, and cut a release of a pc-app-electron app. Use when creating or changing electron.vite.config.ts, electron-builder.yml, the protocol handler, the preload build, CI workflows, or when cutting a release.
---

# electron-release (draft)

Raw procedures moved out of `rules/pc-app-electron.md`. The rules they serve stay in that
file (§2, §4.2, §5). This draft leaves `_drafts/` only when the Unverified list at the end
is empty.

## 1. Toolchain setup (rule §2)

- Install pnpm standalone first; `pnpm install` then installs the Node version in
  `devEngines.runtime`.
- `pnpm-workspace.yaml`:

  ```yaml
  nodeLinker: hoisted
  allowBuilds:
    electron: true
    better-sqlite3: true
  ```

  Add a package to `allowBuilds` only when `pnpm install` reports its install script was
  skipped and the app fails without it.
- `pnpm exec playwright install chromium`, once, for renderer component tests.
- If `pnpm install` compiles better-sqlite3 from source, the platform has no prebuilt
  binary: install the C toolchain of rule §2 and record the platform in `README.md`.

## 2. Custom scheme (rule §4.2)

In `main/index.ts`, before `app.whenReady()`:

```ts
protocol.registerSchemesAsPrivileged([
  { scheme: "app", privileges: { standard: true, secure: true, supportFetchAPI: true } },
]);
```

After `ready`, `protocol.handle("app", handler)`. The handler:

1. Parses `request.url`; refuses any host other than the one the app uses (`app://app/`).
2. Decodes the path, joins it to the renderer build directory, normalizes, and refuses
   (404) if the result is outside that directory.
3. Serves `index.html` for `/`.
4. Returns `net.fetch(pathToFileURL(file).toString())` with the CSP and
   `X-Content-Type-Options` headers of rule §4.4 added.

Window load:

```ts
const devUrl = process.env.ELECTRON_RENDERER_URL;
if (!app.isPackaged && devUrl) await win.loadURL(devUrl);
else await win.loadURL("app://app/");
```

The sender check of rule §4.1 compares `new URL(event.senderFrame.url).origin` with
`app://app` or, under the same condition as above, the dev server's origin.

## 3. Preload (rule §4.1, §5.2)

```ts
// src/preload/index.ts
const api = {
  listItems: (input: ListItemsInput) => ipcRenderer.invoke(CH.listItems, input),
  onProgress: (cb: (p: Progress) => void) => {
    const listener = (_e: unknown, p: Progress) => cb(p);
    ipcRenderer.on(CH.progress, listener);
    return () => ipcRenderer.removeListener(CH.progress, listener);
  },
} satisfies Bridge;
contextBridge.exposeInMainWorld("api", api);
export type Api = typeof api;
```

`renderer/src/env.d.ts` declares `window.api: Api` with a type-only import.

electron-vite: preload output format `cjs`, dependencies bundled (not externalized).

## 4. electron-builder.yml (rule §5.2, §5.4)

```yaml
appId: <reverse-dns id>
directories: { output: dist }
files: [out/**, package.json]
asar: true
electronFuses:
  runAsNode: false
  enableCookieEncryption: true
  enableNodeOptionsEnvironmentVariable: false
  enableNodeCliInspectArguments: false
  enableEmbeddedAsarIntegrityValidation: true
  onlyLoadAppFromAsar: true
  grantFileProtocolExtraPrivileges: false
win: { target: nsis }
nsis: { oneClick: false, perMachine: false, deleteAppDataOnUninstall: false }
mac: { target: dmg }
linux: { target: AppImage }
publish: null
```

Only targets for declared OSes. `package.json` scripts:

```json
"build": "tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.web.json && tsc --noEmit -p tsconfig.core.json && electron-vite build",
"package": "electron-builder --publish never"
```

## 5. Self-check (rule §5.3)

At the top of `main/index.ts`, before `ready`:

```ts
const selfCheck = process.argv.includes("--self-check");
if (selfCheck) {
  const root = mkdtempSync(join(tmpdir(), "<app>-selfcheck-"));
  app.setPath("userData", join(root, "data"));
  app.setPath("logs", join(root, "logs"));
}
```

Then the normal start with credentials disabled, the window created with `show: false`,
and a `selfCheck:ready` message from the renderer after first render. On ready: print the
version, `app.quit()`, exit 0. A timer per timeout of rule §5.3 exits non-zero with the
reason.

## 6. Whole-app tests (rule §6)

```ts
const app = await _electron.launch({
  args: ["out/main/index.js"],
  env: { ...process.env, <APP>_TEST_USER_DATA: tmpDir },
});
```

`main/index.ts` honours `<APP>_TEST_USER_DATA` only when `app.isPackaged` is false.
Dialog stub:

```ts
await app.evaluate(({ dialog }, file) => {
  dialog.showOpenDialog = async () => ({ canceled: false, filePaths: [file] });
}, fixturePath);
```

CI on Linux: `xvfb-run -a pnpm exec playwright test`.

## 7. CI (rule §5.5)

- PR workflow: ubuntu job, steps as rule §5.5.
- Tag workflow: matrix over the declared OSes (`windows-latest`, `macos-latest`,
  `ubuntu-latest`); each runs tests, `pnpm build`, `pnpm package`, then:
  - fuses: `pnpm exec electron-fuses read --app <packaged binary>` and compare with §4;
  - self-check: run the packaged binary with `--self-check` (Linux: under `xvfb-run`).
- A final job, needing every matrix job, creates the GitHub Release for the tag with
  `gh release create` and uploads the artifacts from each job. The token is available to
  that job only.

## 8. Release

1. Bump `version` in `package.json`, commit.
2. Human approves the release (`GLOBAL_RULES.md` §12), then tag `v<version>` and push.
3. Confirm the release page lists one artifact per declared OS, and that the version the
   downloaded app reports is the tag.

## Unverified

- [verify] electron-vite 6 option names for preload `cjs` output and for bundling the
  preload's dependencies.
- [verify] Whether electron-builder 26 runs `@electron/rebuild` on better-sqlite3 13 by
  default, and whether `npmRebuild: false` is needed to keep the prebuilt binary.
- [verify] The `allowBuilds` entries a fresh project actually needs with Electron 45.
- [verify] How `isAsyncEncryptionAvailable()` behaves on Linux with no secret service, and
  whether `getSelectedStorageBackend()` reflects the async provider.
- [verify] Whether `style-src 'unsafe-inline'` is needed with shadcn/ui and Radix, or
  `style-src 'self'` suffices.
- [verify] The `@electron/fuses` CLI command name and output format.
- [verify] Playwright `_electron.launch` with a sandboxed window and the custom scheme on
  Electron 45.
