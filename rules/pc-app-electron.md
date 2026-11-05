# rules/pc-app-electron.md

Desktop app, TypeScript end to end, React UI in Electron, single user, local data.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if the UI does not need web technology or the app needs Python libraries
(`pc-app-python`), if two people share the data (a web archetype), or if there is no
window (a script).

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Node.js (tooling) | An LTS line, installed by pnpm, pinned in `package.json` `devEngines.runtime` | — | Odd-numbered (non-LTS) majors; `.nvmrc`, nvm, fnm, Volta as the version record |
| Packaging | pnpm, standalone install, exact version in `packageManager`; `pnpm-lock.yaml`; `nodeLinker: hoisted` | — | npm, yarn, bun; Corepack; `npx` in scripts or docs |
| Language | TypeScript, `strict: true`, ES modules | — | Plain JavaScript in new code |
| Shell | Electron, one of the three supported majors, exact version | Tauri 2, when installer size or memory is a stated product requirement | A major outside the support window; `@electron/remote`; `<webview>` |
| Build | electron-vite 6: main, preload and renderer from one `electron.vite.config.ts` | — | `vite-plugin-electron` (a different project with a similar name); Electron Forge's Vite plugin; webpack |
| UI | React | — | — |
| Styling | Tailwind CSS v4 via `@tailwindcss/vite`; shadcn/ui, Radix base, copied into `src/renderer/src/components/ui/` | CSS Modules | `tailwind.config.js`, `@tailwind` directives; CSS-in-JS; full component kits installed as packages (MUI, Chakra, Mantine, Ant Design) |
| Renderer state | None — `useState`, `useReducer` | Zustand, for state shared by distant components | Redux, Redux Toolkit |
| Renderer data | Loaded through the bridge (§4.1) by the view that shows it, reloaded after the action that changes it | TanStack Query, for polling or optimistic updates | SWR |
| Validation | Zod 4 | — | — |
| IPC | `ipcRenderer.invoke` ↔ `ipcMain.handle`, behind one typed object exposed by `contextBridge` | — | `nodeIntegration: true`, `contextIsolation: false`, `sandbox: false`; exposing `ipcRenderer` or a generic `invoke(channel, …)`; `sendSync` |
| Persistence | SQLite via better-sqlite3 13+ in the main process | — | Any database access from the renderer; IndexedDB or `localStorage` as the store of record; `node:sqlite` while it is a release candidate; `sqlite3` (node-sqlite3) |
| Migrations | Ordered SQL on `PRAGMA user_version` (§4.3) | — | Schema created outside a migration; editing a shipped migration |
| Paths | `app.getPath()` | — | Paths relative to cwd or `__dirname` for user data; writing into the install dir or `app.asar` |
| Settings | JSON in `userData`, parsed with Zod, read at startup | — | Renderer `localStorage` for settings |
| User credentials | `safeStorage` async API; ciphertext in the DB | — | keytar; `safeStorage.setUsePlainTextEncryption`; a credential in settings, logs, or renderer storage |
| Background work | `utilityProcess` | — | Long synchronous work on the main process; `child_process.fork` of app code (disabled by the `runAsNode` fuse) |
| Lint / format | Biome | ESLint + Prettier | — |
| Type check | `tsc --noEmit` on every config (§3) | — | Treating a passing `electron-vite build` as a type check |
| Tests | Vitest — `node` for logic and adapters; browser mode (`@vitest/browser-playwright`, `vitest-browser-react`) for renderer components; Playwright `_electron` for the app | — | jsdom, happy-dom; Jest; Spectron |
| Package | electron-builder: Windows NSIS (per-user), macOS `.dmg`, Linux AppImage | Electron Forge | Building another OS's installer on one machine (Wine or equivalents); distributing a dev-machine build |
| Auto-update | None; `README.md` links to GitHub Releases | electron-updater from GitHub Releases | — |
| Third-party runtime resources | None: scripts, styles, fonts and images ship in the app | — | `<script>` or `<link>` to a CDN; remote content in a window that has the preload |

---

## 2. Prerequisites

`pnpm install` neither installs nor reveals these. Procedure: skill `electron-release`.

| Item | Scope |
|---|---|
| pnpm | System, standalone, installed first. The exact version in `packageManager` |
| Node.js | Installed by pnpm from `devEngines.runtime`. No separate Node installer |
| Chromium for renderer tests | Project, downloaded once by `pnpm exec playwright install chromium`. Whole-app tests drive the Electron binary |
| C compiler | System, only when better-sqlite3 has no prebuilt binary for the platform. Windows: VS Build Tools (C++) + Python. macOS: Xcode CLT. Linux: build-essential |
| A machine or CI runner per declared target OS | Builds run on the OS they target (§5.2) |
| Linux display for whole-app tests in CI | Xvfb on the runner |

---

## 3. Layout

```
<repo>/
├── package.json            version, packageManager, devEngines.runtime, main, scripts
├── pnpm-lock.yaml
├── pnpm-workspace.yaml     allowBuilds, nodeLinker: hoisted
├── electron.vite.config.ts main, preload, renderer
├── electron-builder.yml    targets, asar, electronFuses
├── tsconfig.json           references the three below
├── tsconfig.node.json      main/, preload/, shared/ — Node and Electron types
├── tsconfig.web.json       renderer/, shared/ — DOM
├── tsconfig.core.json      domain/, application/ — no DOM, no Node types
├── biome.json
├── components.json         shadcn/ui
├── vitest.config.ts        projects: logic and adapters in Node, renderer in Chromium
├── playwright.config.ts
├── build/                  icons, installer resources
├── src/
│   ├── shared/             ipc.ts: channel names, Zod schemas, result and error types
│   ├── domain/             pure TypeScript
│   ├── application/        use cases + ports.ts
│   ├── main/
│   │   ├── index.ts        entry + composition root
│   │   ├── ipc/            one handler module per channel group
│   │   └── adapters/       sqlite, migrations.ts, settings, credentials, logger, clock
│   ├── preload/index.ts    the bridge; exports the type of what it exposes
│   └── renderer/           index.html; src/: main.tsx, views/, components/, components/ui/
├── tests/                  domain/, application/, adapters/, ipc/, renderer/, app/, boundaries.test.ts
└── out/, dist/             gitignored
```

- `main/index.ts` is the only module that calls `app.getPath`, opens the database,
  constructs adapters and use cases, registers handlers, and creates windows.
- An app with no business rules MAY omit `domain/` and `application/`, and MUST NOT add
  them to match this layout. A new top-level dir needs an ADR only if it changes a row of
  the table below.

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | `domain/` | `electron`, Node built-ins, better-sqlite3, React, Zod, every other `src/` dir |
| `application/` | `domain/`, own `ports.ts` | `electron`, Node built-ins, better-sqlite3, React, `main/`, `preload/`, `renderer/` |
| `shared/` | `shared/`, `domain/` types, Zod | `electron`, Node built-ins, React, `application/`, `main/`, `preload/`, `renderer/` |
| `main/` | `application/`, `domain/`, `shared/`, `electron`, Node built-ins, better-sqlite3 | `preload/`, `renderer/`, React |
| `preload/` | `electron` (`contextBridge`, `ipcRenderer`), `shared/` | Everything else |
| `renderer/` | `shared/` and `domain/` types, the preload's exported type, React, renderer packages | `electron`, Node built-ins, better-sqlite3, `application/`, `main/` |

- No Electron type (`IpcMainInvokeEvent`, `BrowserWindow`, `WebContents`) outside `main/`
  and `preload/`.
- A handler in `main/ipc/` checks the sender, parses the payload with its schema from
  `shared/`, calls one use case, and returns the result. A decision about *what the
  application does*, taken in a handler, the preload, or a component, is a defect.
- `tsconfig.core.json` includes only `domain/` and `application/`, with no `DOM` in
  `lib` and `types: []`, so DOM or Node use there fails the type check.
  `tests/boundaries.test.ts` fails on any import that breaks the table. These two are the
  boundary check CI and §8 refer to.

---

## 4. Runtime

### 4.1 Processes and IPC

- Every `BrowserWindow` writes out `contextIsolation`, `sandbox` and `nodeIntegration`
  with the values §1 allows, and the preload, even where they are defaults.
- The preload exposes one object through `contextBridge.exposeInMainWorld`. Each member
  is a named function that calls `ipcRenderer.invoke` on one fixed channel from
  `shared/ipc.ts`. A main-to-renderer event is exposed as a function that takes a
  callback and returns an unsubscribe; the callback never receives the
  `IpcRendererEvent`.
- Every handler first checks that `event.senderFrame` belongs to the app's own origin
  (§4.2), then parses the payload with Zod. Either failure is refused before any use case
  runs.
- A handler returns `{ ok: true, value }` or `{ ok: false, error: { code, message } }`.
  An expected failure is returned as a value and shown next to its field. Unexpected
  errors are caught by one wrapper around every handler, logged, and returned as
  `{ code: "internal" }`: a thrown error reaches the renderer with its message.
- Only plain data crosses IPC: values whose types are declared in `shared/`. Class
  instances, functions and Buffers that the renderer would display are converted first.
- A use case on the main process returns within ~100 ms; better-sqlite3 is synchronous
  and a slow call freezes every window. Longer work runs in a `utilityProcess`, which
  opens its own database connection and reports progress and results by message.
- Navigation is limited: `will-navigate` is prevented for any URL outside the app's
  origin, and `setWindowOpenHandler` returns `deny`. A link meant for the browser goes
  through `shell.openExternal`, only after the main process confirms the URL is
  `https:`.
- `session.setPermissionRequestHandler` denies every permission the app does not list.
- A single file write replaces the target atomically: temp file on the **same
  filesystem**, then rename, original left intact if the rename fails. A single DB write
  is one transaction. Work spanning both MUST define its own recovery order.
- `app.requestSingleInstanceLock()` runs before anything opens the database. A second
  instance focuses the first window and quits.

### 4.2 Paths and origin

- `main/index.ts` resolves `userData` and `logs` through `app.getPath`, creates them if
  missing, and passes them to the adapters as one frozen `Paths` object. An adapter that
  resolves its own path cannot be redirected, and `--self-check` (§5.3) depends on
  redirecting it.
- The renderer is served from a custom scheme (`app://`), registered with
  `protocol.registerSchemesAsPrivileged` before `ready` and answered by `protocol.handle`
  from the renderer build. The handler resolves each request inside that directory and
  refuses any path that leaves it. `loadFile` and `file://` URLs are PROHIBITED.
- The dev server URL (`ELECTRON_RENDERER_URL`) is loaded only when `app.isPackaged` is
  false and the variable is set. Every other run, including whole-app tests, uses the
  custom scheme.
- The install directory and `app.asar` are read-only at runtime. Bundled resources are
  located through `app.getAppPath()` or `process.resourcesPath`, never cwd.

### 4.3 Persistence

- One connection per process, opened in `main/index.ts`, with `journal_mode=WAL`,
  `foreign_keys=ON` and a non-zero `busy_timeout`. A write that still hits `SQLITE_BUSY`
  fails as an error; it MUST NOT be retried by re-running the use case.
- Each migration and its `user_version` write run in the **same** `db.transaction`.
- Before migrating an existing file, back it up beside the DB with better-sqlite3's
  `backup()`; a file copy is PROHIBITED. Verify the backup opens and reports the old
  version before migrating.
- A failed migration: roll back, keep original and backup, do not advance
  `user_version`, refuse to open, and offer the user a way to open the backup's folder.
- `user_version` higher than the app supports → refuse to open, tell the user to update.
- Shipped migrations are append-only. Fixes are new migrations. A test records a checksum
  of every shipped migration and fails if one changes; a new migration adds its checksum
  in the release that ships it.

### 4.4 Settings, credentials, errors

- Settings: a Zod schema with defaults and bounds, read once at startup, passed
  explicitly. Wrong types, out-of-bound values, and unknown or missing keys take defaults
  with a logged warning.
- A malformed file is renamed aside (never deleted) and replaced. If the rename or write
  fails, the app runs on in-memory defaults and tells the user settings cannot be saved.
  The app MUST start either way.
- A desktop app has no deployment secrets and ships no `.env.example`. Every secret it
  holds is an end-user credential. This archetype's OS credential store is `safeStorage`:
  the key stays in the OS store, the ciphertext in the database, and only the main
  process decrypts. A credential crosses IPC once, from renderer to main, to be saved; it
  is never sent back.
- Credential features are disabled when `safeStorage.isAsyncEncryptionAvailable()`
  resolves false, or on Linux when `getSelectedStorageBackend()` returns `basic_text`. A
  decrypt result with `shouldReEncrypt` is re-encrypted and saved.
- Released builds have no console: the main process logs one JSON line per event to a
  file in `logs`, rotated by size. The renderer reports errors through a bridge function;
  it writes no log itself.
- `main/index.ts` installs `uncaughtException` and `unhandledRejection` handlers before
  the first window opens. They log and show what failed and what to do, with a control
  that opens the log directory.
- The renderer root has an error boundary, and window `error` and `unhandledrejection`
  listeners report through the bridge. The window shows what failed and what to try.
- A failed action MUST leave no partial write in the DB or on disk. The window MAY show
  stale data as long as the user is told the action failed.
- The `protocol.handle` response sets this Content-Security-Policy: `default-src 'none';
  script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:;
  font-src 'self' data:; connect-src 'self'; object-src 'none'; base-uri 'none';
  form-action 'none'; frame-ancestors 'none'`, and `X-Content-Type-Options: nosniff`.
  Loosening a directive needs an ADR. Network calls to other services run in the main
  process, behind an adapter.

---

## 5. Build and package

Procedure: skill `electron-release`.

### 5.1 Lock the toolchain

- `packageManager` and `devEngines.runtime` (with `onFail: "download"`) are the single
  source for the pnpm and Node versions; CI installs from both. Every install outside a
  developer's first setup uses `pnpm install --frozen-lockfile`.
- Dependency install scripts run only for packages named in `allowBuilds` in
  `pnpm-workspace.yaml`. Allowing every package, or setting `minimumReleaseAge` to 0 to
  get a just-published version, is PROHIBITED.
- The Node in `devEngines.runtime` runs the tools; the app runs on the Node inside
  Electron. `@types/node` follows Electron's Node major.
- `electron`, `electron-vite` and `electron-builder` are locked devDependencies with exact
  versions. One Electron version per release.

### 5.2 Build

- `pnpm build` runs `tsc --noEmit` on every config, then `electron-vite build` into
  `out/`.
- `dependencies` holds only what the main process loads at runtime; electron-builder
  ships every package in it. Everything the renderer uses is bundled by Vite and is a
  devDependency.
- The preload is one CommonJS file with its imports bundled in: a sandboxed preload
  cannot load ES modules or `node_modules`.
- Tailwind v4 is configured in the renderer's CSS entry (`@import "tailwindcss";`). There
  is no `tailwind.config.js`.
- `electron-builder.yml` sets `electronFuses`: `runAsNode`,
  `enableNodeOptionsEnvironmentVariable`, `enableNodeCliInspectArguments` and
  `grantFileProtocolExtraPrivileges` false; `enableCookieEncryption`,
  `enableEmbeddedAsarIntegrityValidation` and `onlyLoadAppFromAsar` true.
- **Declared target OSes** are the ones `README.md` lists under its install section. That
  list is the single source for the electron-builder targets, the first-week builds, and
  the CI matrix; adding or dropping an OS means editing it there first.
- Build and package once on every declared target OS in the first week, as a walking
  skeleton — on CI where the developer has no such machine. Leaving the first
  cross-platform build until release is PROHIBITED.

### 5.3 Self-check

The packaged app run with `--self-check` starts, applies all migrations, opens the window
hidden, waits until the renderer reports ready over IPC, quits, and exits 0 — or non-zero
with the reason.

- It MUST NOT touch the user's environment: before `ready` it points `userData`, `logs`
  and the `Paths` of §4.2 at subdirectories of one temp dir, and it disables
  credential-store access. A test asserts that a `--self-check` run creates nothing under
  the real directories.
- Timeouts: ready within 30 s, shutdown within 10 s, whole run within 60 s. A timeout
  exits non-zero.
- It prints the app version.

### 5.4 Package

- Packaging is done only by electron-builder with the committed `electron-builder.yml`,
  through `pnpm package`, after the build on each OS. CI's package step calls that script;
  inline packaging commands are PROHIBITED.
- `version` in `package.json` is the single source; the installer, the About dialog and
  artifact names read it. CI fails a tag that does not match.
- Per-user install by default. Uninstall MUST NOT delete the data directory without
  asking.
- Builds are unsigned. `README.md` tells users to expect SmartScreen and Gatekeeper
  warnings.

### 5.5 CI

- Push / PR: one Linux job — `pnpm install --frozen-lockfile`, install Chromium, Biome,
  the type checks, Vitest in Node and Chromium including the boundary check, `pnpm build`,
  Playwright against `out/` under Xvfb.
- Version tag: one job per declared target OS — tests, build, package, read the packaged
  binary's fuses and fail if any differs from §5.2, `--self-check` on the packaged app.
  The release is published to GitHub Releases only if every OS job succeeds.

---

## 6. Testing

| Layer | Tool | Electron |
|---|---|---|
| `domain/`, `application/` | Vitest, `node`, fake ports | MUST NOT import it |
| `main/adapters/` | Vitest, `node`, a real SQLite file in a temp dir | no |
| `main/ipc/` | Vitest, `node`, handlers called with a fake event and a fake use case | no |
| Renderer components | Vitest browser mode, Chromium, `vitest-browser-react`; the bridge replaced by a fake | no |
| Whole app | Playwright `_electron.launch` against `out/` | yes |
| Packaged app | `--self-check` | yes |

Most tests are in the first three rows. A component or whole-app test proves an input
reaches the right use case and the result is shown — not the rule itself.

- The fake bridge in renderer tests is typed by the preload's exported type, so a fake
  that drifts from the real bridge fails the type check.
- Every handler is tested with a payload its schema refuses and with a sender from
  another origin; both are refused before the use case is called.
- Each Playwright run starts the app with its own empty data directory in a temp dir. A
  test that passes only because the developer's data exists is a defect.
- Every native dialog a whole-app test reaches is stubbed in the main process through
  `electronApp.evaluate`.
- Controls are found by role and label text, never by CSS selector or id.
- Assert what the user sees: text, `disabled`, `aria-invalid`, row count.
- One test submits input containing markup and asserts it comes back as text.
- A Content-Security-Policy violation reported in the window fails the Playwright run.
- Waiting real time is PROHIBITED; wait for an element, an IPC result, or a window.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. Deprecated Electron APIs from old tutorials: `protocol.registerFileProtocol` and the
   other `register*Protocol` methods, the `new-window` event. Current Electron uses
   `protocol.handle` and `setWindowOpenHandler`.
2. A `postinstall` running `electron-builder install-app-deps` or `@electron/rebuild` for
   better-sqlite3. From 13.0 its prebuilt binaries load in Electron without a rebuild.
3. Turning `sandbox` off so the preload can `require` a package or run as an ES module.
4. Re-enabling `enableNodeCliInspectArguments` so Playwright can drive the packaged app.
   Playwright runs against `out/`; the packaged app is proven by `--self-check`.
5. Setting `webSecurity: false` to get past a CORS or CSP error.
6. Switching to Electron Forge or Tauri to escape an electron-builder error. A build
   error is not grounds for the Alternative's ADR.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes
- [ ] Every new or changed channel is declared in `shared/ipc.ts`, has a schema and the
      sender check, and has the handler tests of §6
- [ ] No new long synchronous work on the main process

Schema change:

- [ ] New migration appended, none edited; tested from the previous version, and tested
      failing mid-migration — original schema and rows unchanged, version unchanged,
      backup opens

Packaging, dependencies, `main/index.ts`, the preload, `electron-builder.yml`, or the
protocol handler change:

- [ ] `pnpm build` and Playwright pass from a frozen install; `--self-check` passes on
      the packaged app on the developer's OS during the task; anything handed to another
      person goes through §5.5
