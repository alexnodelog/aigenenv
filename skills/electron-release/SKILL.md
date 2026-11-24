---
name: electron-release
description: Set up, build, test, package and release a pc-app-electron app. Use when creating a pc-app-electron project or its walking skeleton; when creating or changing package.json scripts, pnpm-workspace.yaml, electron.vite.config.ts, electron-builder.yml, src/main/index.ts (protocol handler, --self-check, E2E data dir), src/preload/index.ts, playwright.config.ts, tests/app/, scripts/check-fuses.sh or .github/workflows/; when upgrading electron, electron-vite, electron-builder or better-sqlite3; when pnpm install fails with ERR_PNPM_IGNORED_BUILDS; when electron-builder fails rebuilding better-sqlite3 with node-gyp; when --self-check or a Playwright Electron test hangs or fails; or when cutting a release.
---

# electron-release

Procedures for `rules/pc-app-electron.md` §2, §4 and §5. The rules stay there; this file
says how. Pin every version below exactly in the project; this file names none.

## 1. Toolchain setup (rule §2)

1. Install pnpm standalone (the exact version in `packageManager`). `pnpm install` then
   installs the Node version in `devEngines.runtime`.
2. `package.json` has `"type": "module"` and `"main": "out/main/index.js"`.
3. `pnpm-workspace.yaml`:

   ```yaml
   nodeLinker: hoisted
   allowBuilds:
     electron-winstaller: false
   ```

   - Neither `electron` nor `better-sqlite3` runs an install script. Electron downloads
     its binary on first use; better-sqlite3 ships prebuilt Node-API binaries for
     linux/darwin/win32 × x64/arm64 (and musl) that load in Electron unchanged.
   - `electron-winstaller` comes with electron-builder and is only used for Squirrel,
     which this archetype does not build.
   - When `pnpm install` fails with `ERR_PNPM_IGNORED_BUILDS` for another package, add it
     as `false`. Set it to `true` only when the app is shown to fail without its script,
     and record why in `DECISIONS.md`.
4. `pnpm exec playwright install chromium` once, for renderer component tests. CI on Linux:
   `pnpm exec playwright install --with-deps chromium`. Whole-app tests drive the Electron
   binary and need no browser download.
5. Every declared target OS must have a better-sqlite3 prebuild in
   `node_modules/better-sqlite3/prebuilds/`. If one does not, stop and report: a source
   build needs the C toolchain of rule §2 and a decision in `DECISIONS.md`.

## 2. Custom scheme (rule §4.2)

In `src/main/index.ts`, before `app.whenReady()`:

```ts
protocol.registerSchemesAsPrivileged([
  { scheme: "app", privileges: { standard: true, secure: true, supportFetchAPI: true } },
]);
```

After `ready`, `protocol.handle("app", handler)`, where `rendererDir` is
`join(import.meta.dirname, "../renderer")`. The handler:

1. Parses `request.url`; refuses (404) any host other than `app`.
2. Decodes the pathname once with `decodeURIComponent`; a decode error → 400.
3. Refuses (404) a decoded path containing `\`, a NUL byte, or a `..` segment. A
   double-encoded `%252e%252e` decodes to `%2e%2e` and is then a literal, nonexistent name.
4. Maps `/` to `/index.html`, then `resolved = path.resolve(rendererDir, "." + decoded)`.
   Refuses (404) when `rel = path.relative(rendererDir, resolved)` is empty, absolute,
   equal to `..`, or starts with `..` + `path.sep`.
5. Returns `net.fetch(pathToFileURL(resolved).toString())` with the CSP and
   `X-Content-Type-Options` of rule §4.4 set on a copy of its headers:

   ```ts
   const res = await net.fetch(pathToFileURL(resolved).toString());
   const headers = new Headers(res.headers);
   headers.set("Content-Security-Policy", CSP);
   headers.set("X-Content-Type-Options", "nosniff");
   return new Response(res.body, { status: res.status, headers });
   ```

Window load:

```ts
const devUrl = process.env.ELECTRON_RENDERER_URL;
if (!app.isPackaged && devUrl) await win.loadURL(devUrl);
else await win.loadURL("app://app/");
```

Sender check (rule §4.1). `new URL("app://app/").origin` is the string `"null"` — `app:`
is not a special scheme — so never compare `.origin` for the app's own pages:

```ts
function isAppSender(e: IpcMainInvokeEvent): boolean {
  const frame = e.senderFrame;
  if (!frame || frame !== e.sender.mainFrame) return false;
  if (!BrowserWindow.fromWebContents(e.sender)) return false;
  const u = new URL(frame.url);
  if (u.protocol === "app:" && u.host === "app") return true;
  return !app.isPackaged && !!devUrl && u.origin === new URL(devUrl).origin;
}
```

The `style-src 'unsafe-inline'` in rule §4.4 is needed: Radix dialogs (and shadcn/ui
built on them) insert a `<style>` element for scroll locking, which `style-src 'self'`
blocks.

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

`electron.vite.config.ts`:

```ts
import { defineConfig } from "electron-vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  main: {},
  preload: {
    build: {
      externalizeDeps: false,
      rolldownOptions: { output: { format: "cjs" } },
    },
  },
  renderer: { plugins: [react(), tailwindcss()] },
});
```

- With `"type": "module"`, electron-vite builds the preload as an ES module unless
  `format: "cjs"` is set. The output is `out/preload/index.cjs`; the window's `preload`
  option names that file: `join(import.meta.dirname, "../preload/index.cjs")`.
- `externalizeDeps: false` bundles every import into the preload, including packages in
  `dependencies`.
- Check after each build: `out/preload/index.cjs` exists and has no `require(` other than
  `require("electron")`. If it does, stop and report.

## 4. electron-builder.yml (rule §5.2, §5.4)

```yaml
appId: <reverse-dns id>
productName: <App Name>
directories: { output: dist }
files: [out/**, package.json]
asar: true
npmRebuild: false
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
linux: { target: AppImage, category: Utility, executableName: <app> }
publish: null
```

- `npmRebuild: false` is required. Without it electron-builder runs `@electron/rebuild`,
  which compiles better-sqlite3 from source with node-gyp: it needs a C toolchain and
  network access to Electron's headers, and replaces a prebuild that already works.
- Keep only the targets of declared OSes. electron-builder unpacks `.node` files to
  `app.asar.unpacked` by itself.
- Packaged executables: Linux `dist/linux-unpacked/<app>`; Windows
  `dist/win-unpacked/<App Name>.exe`; macOS
  `dist/mac-arm64/<App Name>.app/Contents/MacOS/<App Name>`.

`package.json` scripts:

```json
"build": "tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.web.json && tsc --noEmit -p tsconfig.core.json && electron-vite build",
"package": "electron-builder --publish never"
```

`scripts/check-fuses.sh`:

```bash
#!/usr/bin/env bash
# Usage: scripts/check-fuses.sh <packaged executable or .app>
set -euo pipefail
out=$(pnpm exec electron-fuses read --app "$1")
echo "$out"
for line in \
  "RunAsNode is Disabled" \
  "EnableCookieEncryption is Enabled" \
  "EnableNodeOptionsEnvironmentVariable is Disabled" \
  "EnableNodeCliInspectArguments is Disabled" \
  "EnableEmbeddedAsarIntegrityValidation is Enabled" \
  "OnlyLoadAppFromAsar is Enabled" \
  "GrantFileProtocolExtraPrivileges is Disabled"; do
  grep -Fxq "  $line" <<<"$out" || { echo "fuse mismatch: expected '$line'" >&2; exit 1; }
done
```

Run it once against `node_modules/electron/dist/electron`: it must fail there. A script
that passes on an unfused binary checks nothing.

## 5. Self-check (rule §5.3)

At the top of `src/main/index.ts`, before `ready`:

```ts
const selfCheck = process.argv.includes("--self-check");
if (selfCheck) {
  // Remove earlier runs' dirs; the current run's cannot be removed while Chromium runs.
  for (const name of readdirSync(tmpdir())) {
    if (name.startsWith("<app>-selfcheck-")) {
      try { rmSync(join(tmpdir(), name), { recursive: true, force: true }); } catch { /* best effort */ }
    }
  }
  const root = mkdtempSync(join(tmpdir(), "<app>-selfcheck-"));
  app.setPath("userData", join(root, "data"));
  app.setPath("logs", join(root, "logs"));
}
```

Derive every `Paths` entry from these. Start normally with credential features disabled,
create the window with `show: false`, and have the renderer send `selfCheck:ready` after
its first render. Start a 60 s timer at launch and a 30 s ready timer; either firing calls
`finish(1, "timeout: …")`. `finish(code, reason)` runs once:

```ts
function finish(code: number, reason: string): void {
  setTimeout(() => app.exit(1), 10_000).unref(); // shutdown limit
  try {
    console.log(reason); // the version on success
    for (const w of BrowserWindow.getAllWindows()) w.destroy();
    worker?.kill();
    db.close();
  } finally {
    app.exit(code);
  }
}
```

- Do not delete the temp dir inside `finish`. Chromium keeps writing into `userData`
  until the process ends: a recursive `rmSync` there throws `ENOTEMPTY` or the dir comes
  back. A throw before `app.exit` leaves the process idle forever, so `app.exit` sits in
  `finally`. The next self-check removes the leftover dir.
- Local run: `pnpm build && pnpm package`, then the packaged executable of §4 with
  `--self-check`. Linux without a display: `xvfb-run -a <executable> --self-check`.
  AppImage: `<file>.AppImage --appimage-extract-and-run --self-check`.
- Running as root (containers) needs `--no-sandbox`. Never add it anywhere else.

## 6. Whole-app tests (rule §6)

`main/index.ts`, before `ready`, and only when `app.isPackaged` is false and the
variable is set:

```ts
if (!app.isPackaged && process.env.E2E_USER_DATA_DIR) {
  app.setPath("userData", process.env.E2E_USER_DATA_DIR);
}
```

Launch with the project directory, not `out/main/index.js`. Otherwise Electron does not
read `package.json` and `app.getVersion()` returns Electron's version:

```ts
const tmpDir = mkdtempSync(join(tmpdir(), "e2e-"));
const app = await _electron.launch({
  args: ["."],
  env: { PATH: process.env.PATH ?? "", DISPLAY: process.env.DISPLAY ?? "", E2E_USER_DATA_DIR: tmpDir },
});
const win = await app.firstWindow();
const cspErrors: string[] = [];
win.on("console", (m) => {
  if (m.type() === "error" && m.text().includes("Content Security Policy")) cspErrors.push(m.text());
});
// … test …
await app.close();
expect(cspErrors).toEqual([]);
```

- Chromium reports CSP violations as console errors containing
  "Content Security Policy". The listener above is the CSP check of rule §6.
- Check `win.url()` is `app://app/` once in the walking skeleton.

Dialog stub:

```ts
await app.evaluate(({ dialog }, file) => {
  dialog.showOpenDialog = async () => ({ canceled: false, filePaths: [file] });
}, fixturePath);
```

Linux: `xvfb-run -a pnpm exec playwright test`.

## 7. CI (rule §5.5)

Linux runners (Ubuntu 24.04+) restrict unprivileged user namespaces, so Electron's
sandbox cannot start. On Linux jobs, before any Electron run (Playwright or self-check):

```yaml
- name: Allow user namespaces for the Chromium sandbox (runner only)
  if: runner.os == 'Linux'
  shell: bash
  run: |
    if [ "$(sysctl -n kernel.apparmor_restrict_unprivileged_userns 2>/dev/null || echo 0)" = "1" ]; then
      sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
    fi
```

This changes the disposable runner only. Adding `--no-sandbox` or turning `sandbox`
off instead is PROHIBITED.

- **`ci.yml`, push / PR:** one `ubuntu-24.04` job with the steps of rule §5.5.
- **`release.yml`, tag:**
  - Start with the opening checks of rule module-deployment §5.3.
  - Run a matrix over the declared OSes, OS version labels only (for example
    `ubuntu-24.04`, `windows-2025`, `macos-26`).
  - Each matrix job: tests, `pnpm build`, `pnpm package`, `scripts/check-fuses.sh` on the
    packaged executable (`shell: bash`, macOS on the `.app`), the self-check, then
    `actions/upload-artifact` of its installer.
  - Linux and macOS self-check: run the executable with `--self-check` (Linux: the
    AppImage with `--appimage-extract-and-run`, under `xvfb-run -a`).
  - Windows self-check: the executable is a GUI program, so PowerShell does not wait for
    it and shows no output by default:

    ```yaml
    - name: Self-check (Windows)
      if: runner.os == 'Windows'
      shell: pwsh
      run: |
        $p = Start-Process -FilePath "dist/win-unpacked/<App Name>.exe" -ArgumentList '--self-check' `
          -Wait -PassThru -NoNewWindow -RedirectStandardOutput sc-out.txt -RedirectStandardError sc-err.txt
        Get-Content sc-out.txt, sc-err.txt
        if ($p.ExitCode -ne 0) { exit 1 }
        $v = (Get-Content package.json -Raw | ConvertFrom-Json).version
        if (-not (Select-String -Path sc-out.txt -SimpleMatch $v -Quiet)) { exit 1 }
    ```

    If the exit code is 0 but the version is missing, the output did not reach the file:
    stop and report. Do not drop the version check.
  - Publish job on Linux, needing every matrix job:
    1. Download the artifacts and write `SHA256SUMS` with `sha256sum`.
    2. `gh release create "$TAG" --draft --verify-tag --title "$TAG" --notes-file notes.md`.
    3. `gh release upload "$TAG" <files> SHA256SUMS`.
    4. `gh release edit "$TAG" --draft=false`.
    5. `gh release download "$TAG" -D check && (cd check && sha256sum -c SHA256SUMS)`.

    Only this job gets `contents: write`.

## 8. Release

1. Follow the version proposal of rule module-deployment §5.4.
2. Bump `version` in `package.json` and commit.
3. Tag locally and confirm `git rev-parse v<version>^{commit}` is the approved SHA before
   pushing. The tag workflow fails unless the tag is `v` + `package.json`'s `version`:

   ```bash
   test "v$(node -p "require('./package.json').version")" = "$GITHUB_REF_NAME"
   ```

4. Confirm the release page lists one installer per declared OS plus `SHA256SUMS`, and that
   a downloaded app shows the tag's version.
5. The Linux install section of `README.md` tells Ubuntu 24.04+ users that the AppImage
   needs unprivileged user namespaces allowed for it (an AppArmor profile for that file, or
   the sysctl above). It never suggests `--no-sandbox`.
