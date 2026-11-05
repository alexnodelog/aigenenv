# rules/mobile.md

Android app built with Expo (React Native) in TypeScript. One person's data on one device;
any server it talks to is a separate project.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if a responsive web page would do (`web-static` or `web-ts`), or if it
runs on a PC (`pc-app-python`, `pc-app-electron`). Data shared between users or devices
needs a server: that server is a web archetype project, and this app is its client.

iOS is deferred. Nothing here builds, tests or ships for iOS; starting iOS work is an ADR.

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Node.js | An LTS line the Expo SDK supports, installed by pnpm, pinned in `package.json` `devEngines.runtime` | — | Odd-numbered (non-LTS) majors; `.nvmrc`, nvm, fnm, Volta as the version record |
| Packaging | pnpm, standalone install, exact version in `package.json` `packageManager`; `pnpm-lock.yaml`; `nodeLinker: isolated` (pnpm's default) | `nodeLinker: hoisted`, with the ADR naming the library that failed under `isolated` | npm, yarn, bun; Corepack; `npx` in scripts or docs; `enableGlobalVirtualStore` — Metro cannot resolve packages stored outside the project |
| Language | TypeScript, `strict: true` | — | Plain JavaScript in new code |
| Framework | Expo SDK 58 (React Native 0.88), Continuous Native Generation: `android/` and `ios/` are generated and gitignored | — | Bare React Native; committed native directories |
| Native configuration | `app.config.ts` and config plugins, `expo-build-properties` for Gradle settings | A local config plugin in `plugins/` | Editing anything under `android/` |
| CLI | Expo CLI from the `expo` package, run as `pnpm expo` | EAS CLI, only with an EAS Alternative below, run as `pnpm dlx eas-cli@<exact version>` | `eas-cli` as a project dependency; a global install of either |
| Navigation | Expo Router, routes in `src/app/` | React Navigation | — |
| State | None — `useState`, `useReducer` | Zustand, for state shared by distant screens | Redux, Redux Toolkit |
| Persistence | SQLite via `expo-sqlite`, async API, plain SQL | AsyncStorage for a few key-value preferences | An ORM (Drizzle, TypeORM, WatermelonDB); records in AsyncStorage; the `*Sync` methods of `expo-sqlite` |
| Migrations | Ordered functions on `PRAGMA user_version` (§4.3) | — | Editing a shipped migration |
| Credentials | `expo-secure-store` | — | Tokens in AsyncStorage, SQLite, or a file |
| Lint / format | Biome | ESLint + Prettier | — |
| Type check | `tsc --noEmit` | — | Treating a running Metro bundle as a type check |
| Tests | Jest via `jest-expo`, `@testing-library/react-native`; Maestro on an Android emulator | — | `react-test-renderer`; Detox, Appium |
| Development build | `pnpm expo run:android` with `expo-dev-client`, on the emulator or a USB device | EAS cloud build | Expo Go as the only test path |
| Release build | GitHub Actions: `expo prebuild` + Gradle, signed with the upload key from CI secrets | EAS cloud build — personal non-commercial projects only | A release built on a developer machine; `eas build --local` on Windows |
| OTA updates | None | EAS Update — personal non-commercial projects only | Shipping a native change as an OTA update |
| Third-party runtime services | None: no analytics, crash-reporting or advertising SDK | One error-reporting service, by ADR (§4.4) | Tracking SDKs |

---

## 2. Prerequisites

`pnpm install` neither installs nor reveals these. Procedure: skill `expo-android-release`.

| Item | Scope |
|---|---|
| pnpm | System, standalone, installed first. The exact version in `packageManager` |
| Node.js | Installed by pnpm from `devEngines.runtime`. No separate Node installer |
| Android Studio, Android SDK Platform 36, an emulator image | System. `ANDROID_HOME` set, `platform-tools` on `PATH` |
| JDK 17 | System. `JAVA_HOME` set explicitly, not left to Android Studio's bundled JDK |
| Maestro CLI | System. Needs Java 17+ |
| Upload keystore | A file the project owns, outside the repository, backed up off the machine. CI receives it and its passwords as secrets |
| Google Play developer account | Account. Only at store release |

The first `pnpm expo run:android` downloads Gradle's dependencies. After that the
development loop works offline; the Gradle cache MUST NOT be cleared to fix a build.

---

## 3. Layout

```
<repo>/
├── package.json          version, packageManager, devEngines.runtime, scripts
├── pnpm-lock.yaml
├── pnpm-workspace.yaml   pnpm settings: allowBuilds
├── app.config.ts         identity, version, permissions, plugins
├── tsconfig.json         strict
├── biome.json
├── jest.config.ts        projects: core in node, ui in jest-expo
├── .env.example
├── plugins/              local config plugins, only when §1's Alternative is taken
├── src/
│   ├── app/              Expo Router routes only: _layout.tsx, one file per screen
│   ├── screens/          one component per screen; routes render these
│   ├── components/       shared components
│   ├── config.ts         the only reader of process.env
│   ├── composition.ts    composition root: opens the database, builds adapters and services
│   ├── domain/           pure TypeScript
│   ├── services/         use cases + ports.ts
│   └── adapters/         sqlite/ (connection.ts, migrations.ts, repositories), secure-store, files, clock
├── maestro/              end-to-end flows; smoke.yaml is the self-check (§5.3)
├── tests/                core/, adapters/, ui/, boundaries.test.ts
├── android/, ios/        generated by prebuild; gitignored
└── dist/                 gitignored; release artifacts
```

- Every file in `src/app/` is a route, reachable by deep link. Components, hooks and helpers
  MUST NOT live there. A route file renders one `screens/` component and passes it the
  validated route params.
- `src/app/_layout.tsx` calls `composition.ts` once at start and provides the services
  through React context. Nothing else constructs an adapter or opens the database.
- An app with no business rules MAY omit `domain/` and `services/`, and MUST NOT add them
  to match this layout. A new top-level dir needs an ADR only if it changes a row of the
  import table below.

| Directory | MAY import | MUST NOT import |
|---|---|---|
| `domain/` | `domain/` | React, `react-native`, `expo*`, `adapters/`, `app/`, `screens/`, `components/` |
| `services/` | `domain/`, own `ports.ts` | React, `react-native`, `expo*`, `adapters/`, `app/`, `screens/`, `components/` |
| `adapters/` | `services/ports.ts`, `domain/`, `expo-sqlite`, `expo-secure-store`, `expo-file-system` | React, `react-native`, `app/`, `screens/`, `components/` |
| `app/`, `screens/`, `components/` | `services/` types, `domain/` types, React, `react-native`, `expo-router`, Expo UI and device modules | `adapters/`, `composition.ts`, `expo-sqlite`, `expo-secure-store`, `expo-file-system` |

- A handler in a screen reads input, calls one use case, and renders the result. A decision
  about *what the application does*, taken in a screen or component, is a defect.
- `tests/boundaries.test.ts` walks the imports of every file under `src/` and fails on any
  that breaks the table. That test is the boundary check CI and §8 refer to.

---

## 4. Runtime

### 4.1 App lifecycle and the JS thread

- Android may kill the app in the background without warning or callback. Every change the
  user makes is written when it is made; nothing waits for the app to close or to go to
  the background. When the app returns to the foreground, the visible screen re-reads
  what it shows.
- All JavaScript runs on one thread, which also handles touches. Work that can exceed
  ~100 ms — a large query, parsing a file, a loop over many records — is paged or moved
  out of a render, and the screen shows that it is working.
- Every route param is input from outside the app: anyone can open a deep link. The route
  file validates it before the screen sees it, and an invalid one shows the not-found
  screen.
- A permission is requested when the user starts the feature that needs it, never at
  startup. Denial leaves the rest of the app working and tells the user how to grant it
  in system settings.
- The built app declares exactly the permissions its features use. Expo modules add
  permissions through their config plugins; every one the app does not use is listed in
  `android.blockedPermissions`. The list of permissions the app uses is kept in
  `app.config.ts` and checked against the built APK (§5.3).
- The core of the app works offline. A network call goes through an adapter, over HTTPS
  only, with a timeout, a visible failure state, and its response validated before use.
  Cleartext traffic stays disabled.
- Data reaches the screen as `<Text>` children and component props. Rendering stored or
  fetched HTML or Markdown as markup is PROHIBITED without an ADR naming the sanitizer.

### 4.2 Paths

- Files the app keeps live in the app's document directory (`expo-file-system`). The cache
  directory holds only what can be recreated: Android clears it when space runs low.
- Reading or writing a file outside the app's own directories goes through the system
  picker or the share sheet, chosen by the user each time. Broad storage permissions are
  PROHIBITED.
- Bundled assets are imported (`require`, `expo-asset`), never read by path.
- `android.allowBackup` is set explicitly in `app.config.ts`, and the choice is recorded in
  `DECISIONS.md`. Android's automatic backup copies the database to the user's cloud
  storage but not `expo-secure-store` values, so a restored install has data without its
  credentials and MUST start working without them.
- Uninstalling deletes every file the app kept. `README.md` tells users so, and what the
  export of §4.3 protects.

### 4.3 Persistence

- One connection, opened in `composition.ts` with `openDatabaseAsync`, used by every
  repository. `PRAGMA foreign_keys=ON` on open.
- Repositories in `adapters/sqlite/` depend on `SqlConnection`, an interface in
  `connection.ts` naming only the `expo-sqlite` methods the app uses. Production passes the
  `expo-sqlite` database; tests pass an implementation over `node:sqlite` (§6).
- A use case that writes more than once runs those writes in one
  `withExclusiveTransactionAsync`. `withTransactionAsync` is PROHIBITED: other queries
  issued meanwhile join its transaction. A failed use case leaves no partial write.
- Each migration and its `user_version` write run in the same transaction.
- Before migrating an existing database, back it up beside the original with
  `backupDatabaseAsync`, and verify the backup opens and reports the old version. A file
  copy is PROHIBITED.
- A failed migration: roll back, keep original and backup, do not advance `user_version`,
  refuse to open, and offer to send the backup file out through the share sheet.
- `user_version` higher than the app supports → refuse to open, tell the user to update.
- Work spanning a file and the database writes the new file under a fresh name first, then
  commits the transaction that references it, then deletes the file it replaced. At
  startup, files in the app's file directory that no row references are deleted.
- An app holding data its users cannot recreate offers export to a file through the share
  sheet and import through the document picker. A lost or replaced phone loses everything
  else. The format and what it covers are recorded in `DECISIONS.md` before the app holds
  real data.
- Shipped migrations are append-only. Fixes are new migrations. A test records a checksum
  of every shipped migration and fails if one changes; a new migration adds its checksum in
  the release that ships it.

### 4.4 Settings, credentials, errors

- Everything in the APK can be read by anyone who has it. The app ships no deployment
  secret: a key for a paid or private service needs a server, which is a web archetype
  project.
- `src/config.ts` reads `process.env.EXPO_PUBLIC_*`, validates presence, type and format,
  and exports a frozen object; a missing or invalid value fails the build. These values are
  compiled into the bundle and are never secret. `app.config.ts` reads only build settings
  (identity, version, plugins).
- User preferences: a typed object read once at startup. A value that fails validation
  takes its default, and the app MUST start either way.
- A credential the user gives the app goes to `expo-secure-store`. If the app signs in to a
  server, it stores only the token that server issues, never the password.
- `src/app/_layout.tsx` exports an `ErrorBoundary` that shows what failed and what to try,
  with a control to retry. Screens that load data handle their own failure state; the
  boundary is the last resort.
- Release builds have no console a user can read. No error-reporting service is used by
  default. Adding one is an ADR naming its free-tier terms and exactly what data leaves the
  device.

---

## 5. Build and package

Procedure: skill `expo-android-release`.

### 5.1 Lock the toolchain

- `packageManager` and `devEngines.runtime` (with `onFail: "download"`) are the single
  source for the pnpm and Node versions; CI installs from both. Every install outside a
  developer's first setup uses `pnpm install --frozen-lockfile`.
- Dependency install scripts run only for packages named in `allowBuilds` in
  `pnpm-workspace.yaml`. Allowing every package, or setting `minimumReleaseAge` to 0 to get
  a just-published version, is PROHIBITED.
- Packages are added with `pnpm expo install <package>`, which picks the version the SDK
  supports. `pnpm expo install --check` passes in CI.
- CI builds with JDK 17 and the Android SDK Platform of §2. The Gradle version comes from
  the native project the SDK generates; it is never set by hand.
- An SDK upgrade is its own task: one SDK major at a time, with `android/` regenerated and
  a new development build made before any other change.

### 5.2 Build

- `pnpm build` runs `tsc --noEmit`, then `expo prebuild --clean`, then the Gradle release
  tasks for an APK and an AAB. It needs the upload key, so it runs in CI; on a developer
  machine a release-variant check is `pnpm expo run:android --variant release`.
- `version` in `package.json` is the single source. `app.config.ts` reads it for `version`
  and computes `android.versionCode` as major × 10000 + minor × 100 + patch, with minor and
  patch each at most 99. A release never lowers `versionCode`.
- `android.package` is fixed at the first release that leaves the developer and never
  changes: Google Play and Android developer verification identify the app by it.
- Release builds are signed only with the upload key. The debug keystore never signs
  anything given to another person, and the keystore never enters the repository.
- `minSdkVersion` and `targetSdkVersion` stay at the SDK's values or higher. Lowering
  either is PROHIBITED.
- R8 minification stays on in release builds. A library that needs it gets keep rules for
  its own classes through `expo-build-properties`; turning minification off needs an ADR.
- Before any feature beyond it is built, a walking skeleton — one screen, the database with
  its first migration, the version shown in the app — is built by CI as a signed release,
  installed on an emulator, and passes §5.3.

### 5.3 Self-check

`maestro/smoke.yaml` runs on a freshly installed release APK on an emulator, offline, and
exits 0 — or non-zero naming the failure. It:

- launches the app and waits for an element with test ID `app-ready`, which the root layout
  shows only after the database has opened and every migration has applied;
- reads the version on the about screen and checks it equals `version` in `package.json`;
- relaunches the app and checks it reaches `app-ready` again.

Timeouts: `app-ready` within 30 s of launch, whole flow within 120 s. A timeout fails.

Checks on the built APK, before the smoke flow:

- `debuggable` is false, and its signing certificate is the upload key's;
- the permissions in its manifest equal the list in `app.config.ts`;
- it holds no `.env*` file and no source map, and the project's secret check passes over
  its unpacked contents as well as the tree;
- `versionCode` and `versionName` match §5.2.

### 5.4 Package

| Output | Where it goes |
|---|---|
| Signed release APK, built by CI at a tagged commit, with its SHA-256 | GitHub Release asset |
| Signed AAB from the same commit | Google Play, uploaded by the human |

- CI fails a tag that does not match `version`.
- On Google Play, Play App Signing is on; the upload key can then be reset through Play if
  lost. An APK distributed outside Play has no such recovery: a lost key means users must
  uninstall, losing their data, to get another update.
- Android refuses to install a lower `versionCode` over a higher one, so rollback is a new
  release: the previous code with a higher version. A release that carried a migration can
  only be rolled back by a build that still contains that migration; the human decides
  before such a release ships (`GLOBAL_RULES.md` §12).
- `README.md` tells testers how to install the APK and that installing an older one
  requires uninstalling, which deletes their data.

### 5.5 CI

- Push / PR: one Linux job — `pnpm install --frozen-lockfile`, Biome, `tsc --noEmit`, Jest
  including the boundary check and the adapter tests, `pnpm expo install --check`.
- Version tag: all of the above → `pnpm build` → the APK checks of §5.3 → the smoke flow
  and every Maestro flow on an emulator → GitHub Release with the APK and its checksum,
  and the AAB as a workflow artifact. The release is published only if every step passes.
- The keystore and its passwords are set only on the signing step; no other job or step
  can read them.

---

## 6. Testing

| Layer | Tool | React Native |
|---|---|---|
| `domain/`, `services/` | Jest, `node` environment, fake ports | MUST NOT import it |
| `adapters/sqlite/` | Jest, `node` environment, `SqlConnection` over in-memory `node:sqlite`, migrated from `migrations.ts` | no |
| `screens/`, `components/` | Jest via `jest-expo`, `@testing-library/react-native` | yes, native modules mocked |
| Built app | Maestro on an emulator (§5.3) | yes, on a device |

Most tests are in the first two rows. A screen test proves an input reaches the right use
case and the result is shown — not the rule itself.

- `jest-expo` replaces native modules with mocks, so an adapter over `expo-secure-store` or
  `expo-file-system` is not tested by Jest. Each such adapter is reached by at least one
  Maestro flow.
- Controls are found by role and label text (`getByRole`, `getByLabelText`), never by test
  ID; the only test ID in the app is `app-ready`. A control that shows only an icon has an
  `accessibilityLabel`, so one without fails its test.
- Assert what the user sees: text, `disabled`, presence of an element.
- Time is controlled with fake timers. Waiting real time is PROHIBITED in Jest and in
  Maestro: wait for an element with a timeout.
- Every Maestro flow starts with the app's state cleared, so no flow depends on another.
- A feature behind a permission has one flow where it is granted and one where it is denied.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. Removed Expo commands and APIs: the global `expo-cli` package, `expo build:android`,
   `expo eject`, and `expo-sqlite`'s `openDatabase` with `transaction(tx => tx.executeSql())`.
   Current Expo uses the CLI inside the `expo` package, `prebuild`, and
   `openDatabaseAsync`.
2. pnpm 9/10 setting names (`onlyBuiltDependencies`, `node-linker` in `.npmrc`). pnpm 11
   reads `allowBuilds` and `nodeLinker` from `pnpm-workspace.yaml`.
3. Moving to Expo Go or an EAS cloud build because the local Gradle build fails. A build
   error is not grounds for the Alternative's ADR.
4. Silencing `pnpm expo install --check` with `overrides`, `--force`, or a hand-picked
   version of a native library. The SDK's version is the supported one.
5. A release-only crash "fixed" by turning off R8 or by a keep rule covering every class
   (`-keep class ** { *; }`).

---

## 8. Definition of Done — additions

Every task here:

- [ ] Boundary check (§3) passes
- [ ] No `*Sync` SQLite call; no new work over ~100 ms inside a render
- [ ] Screen tests find controls by role and label; no real-time waits

Schema change:

- [ ] New migration appended, none edited; tested from the previous version, and tested
      failing mid-migration — original schema and rows unchanged, version unchanged, backup
      opens

Work that writes both a file and the database:

- [ ] Tested failing after the file write and after the commit; the next start shows the
      old state or the new one, with no unreferenced file left

Permission, config plugin, or `app.config.ts` change:

- [ ] Built APK's permissions equal the declared list; the denied path has a Maestro flow

Export or import change:

- [ ] Tested importing into a fresh install; the rows match the source

Dependency, SDK, or native configuration change:

- [ ] `pnpm expo install --check` passes; `pnpm expo prebuild --clean` and
      `pnpm expo run:android` succeed from a frozen install; anything handed to another
      person goes through §5.5
