---
name: expo-android-release
description: Set up the Android toolchain, configure app.config.ts (version, permissions, backup), create the upload keystore, wire the SQLite test connection, write the Maestro smoke and upgrade flows, check the built APK, set up CI, and cut a release of a mobile (Expo) app. Use when creating or changing app.config.ts, pnpm-workspace.yaml, jest.config.ts, maestro/, CI workflows, the keystore, or when cutting a release.
---

# expo-android-release (draft)

Raw procedures moved out of `rules/mobile.md`. The rules they serve stay in that file
(§2, §4, §5). This draft leaves `_drafts/` only when the Unverified list at the end is
empty.

## 1. Toolchain setup (rule §2)

- Install pnpm standalone first; `pnpm install` then installs the Node version in
  `devEngines.runtime`.
- Install JDK 17 and Android Studio. In Android Studio's SDK Manager, install SDK
  Platform 36, Build-Tools, Platform-Tools and the Emulator; create one emulator.
- Set `JAVA_HOME` (the JDK 17 directory) and `ANDROID_HOME`
  (Windows: `%LOCALAPPDATA%\Android\Sdk`). Add `%ANDROID_HOME%\platform-tools` to `PATH`.
  Confirm with `java -version` and `adb --version`.
- Maestro on Windows: download `maestro.zip` from the Maestro GitHub releases, extract to
  a stable path, add its `bin` to `PATH`, confirm with `maestro --help`.
- New project: `pnpm create expo-app <name> --template default@sdk-58`, then `cd <name>`,
  then `pnpm expo install expo-dev-client`, then add `android/` and `ios/` to `.gitignore`.
  The SDK number follows rule §1.
- `pnpm-workspace.yaml`:

  ```yaml
  allowBuilds: {}
  ```

  Add a package to `allowBuilds` only when `pnpm install` reports its install script was
  skipped and the app fails without it. `nodeLinker: hoisted` goes here only after the
  ADR of rule §1.
- First run: start the emulator, then `pnpm expo run:android`. It generates `android/`,
  builds with Gradle, installs and starts Metro.

## 2. app.config.ts (rule §4.1, §4.2, §5.2)

```ts
import type { ExpoConfig } from "expo/config";
import pkg from "./package.json";

const USED_PERMISSIONS: string[] = [];          // the single list rule §4.1 refers to
const BLOCKED_PERMISSIONS: string[] = [];       // added by plugins, not used
const ALLOW_BACKUP: boolean = ;                 // deliberately unset: ask the human, record the choice in DECISIONS.md, then set it (rule §4.2)

function versionCode(version: string): number {
  const [major, minor, patch] = version.split(".").map(Number);
  if (minor > 99 || patch > 99) throw new Error(`version ${version}: minor and patch must be <= 99`);
  return major * 10000 + minor * 100 + patch;
}

const config: ExpoConfig = {
  name: "<App name>",
  slug: "<slug>",
  scheme: "<scheme>",
  version: pkg.version,
  android: {
    package: "<reverse.domain.app>",
    versionCode: versionCode(pkg.version),
    permissions: USED_PERMISSIONS,
    blockedPermissions: BLOCKED_PERMISSIONS,
    allowBackup: ALLOW_BACKUP,
  },
  plugins: ["expo-router", "expo-sqlite", "expo-secure-store", "./plugins/with-release-signing"],
};

export default config;
```

After adding a module, run `pnpm expo prebuild --clean` and read the merged manifest's
permissions (§5); every one not in `USED_PERMISSIONS` goes to `BLOCKED_PERMISSIONS`.

## 3. Upload keystore (rule §2, §5.2)

Once, on the developer's machine, outside the repository:

```
keytool -genkeypair -v -storetype PKCS12 -keystore upload.keystore -alias upload -keyalg RSA -keysize 2048 -validity 10000
```

One line, so it runs unchanged in CMD, PowerShell and Bash.

- Back up the file and both passwords to a place off this machine before the first
  release. Record where in `DECISIONS.md` (the location, never the passwords).
- CI secrets: `ANDROID_KEYSTORE_BASE64` (the file, base64), `ANDROID_KEYSTORE_PASSWORD`,
  `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`.
- Signing is applied after `prebuild` by `plugins/with-release-signing.ts` (rule §1, Default),
  which reads those variables into the release `signingConfig`; when they are absent it
  MUST fail the release build, not fall back to the debug key.

## 4. SQLite test connection (rule §4.3, §6)

`src/adapters/sqlite/connection.ts` declares the subset in use:

```ts
export type SqlParam = string | number | null | Uint8Array;

/** What a repository runs statements on: the connection, or a transaction. */
export interface SqlExecutor {
  execAsync(sql: string): Promise<void>;
  runAsync(sql: string, params: SqlParam[]): Promise<{ changes: number; lastInsertRowId: number }>;
  getAllAsync<T>(sql: string, params: SqlParam[]): Promise<T[]>;
  getFirstAsync<T>(sql: string, params: SqlParam[]): Promise<T | null>;
}

export interface SqlConnection extends SqlExecutor {
  withExclusiveTransactionAsync(task: (txn: SqlExecutor) => Promise<void>): Promise<void>;
}
```

A transaction receives only `SqlExecutor`: nesting transactions is not part of the
contract. Values are always passed in `params`, never built into `sql`.

`tests/adapters/node-sqlite-connection.ts` implements it over
`new DatabaseSync(":memory:")` from `node:sqlite`, wrapping each call in a resolved promise
and the transaction in `BEGIN IMMEDIATE` / `COMMIT` / `ROLLBACK`. Every adapter test opens
a fresh connection and runs `migrations.ts` on it first.

Backup before a migration (rule §4.3) — `backupDatabaseAsync` copies into an *open*
database, not a path:

```ts
const dest = await openDatabaseAsync(`${DB_NAME}.backup-v${fromVersion}`);
await backupDatabaseAsync({ sourceDatabase: db, destDatabase: dest });
const { user_version } = (await dest.getFirstAsync<{ user_version: number }>("PRAGMA user_version", []))!;
await dest.closeAsync();
if (user_version !== fromVersion) throw new Error("backup does not report the old version");
```

This double is not `expo-sqlite`. Type conversion, error codes and the SQLite version can
differ; the upgrade flow (rule §6) is what proves migrations on the real engine.

`jest.config.ts`: two projects — `core` (`tests/core`, `tests/adapters`,
`testEnvironment: "node"`) and `ui` (`tests/ui`, `preset: "jest-expo"`).

## 5. Built-APK checks (rule §5.3)

With Build-Tools on `PATH` (`$ANDROID_HOME/build-tools/<version>`):

- Signer: `apksigner verify --print-certs app-release.apk` — SHA-256 equals the upload
  key's (`keytool -list -v -keystore upload.keystore`).
- Manifest: `aapt2 dump badging app-release.apk` — `versionCode`, `versionName`,
  `uses-permission` lines (compare with `USED_PERMISSIONS`), and no `application-debuggable`.
- Contents: unzip to a temp dir; fail on `.env*` or `*.map`; run the project's secret check
  over the directory.

## 6. Maestro smoke and upgrade flows (rule §5.3, §6)

`maestro/flows/smoke.yaml`:

```yaml
appId: <reverse.domain.app>
---
- clearState
- launchApp
- extendedWaitUntil:
    visible:
      id: "app-ready"
    timeout: 30000
- tapOn: "About"
- assertVisible: "Version ${APP_VERSION}"
- stopApp
- launchApp
- extendedWaitUntil:
    visible:
      id: "app-ready"
    timeout: 30000
```

Run: `maestro test -e APP_VERSION=<version> maestro/flows/smoke.yaml`, with the emulator's network
off (`adb shell svc wifi disable`, `adb shell svc data disable`).

Upgrade flow (rule §6), from the second release on:

1. `gh release download v<previous> -p '*.apk'` and `adb install` it on a fresh emulator.
2. `maestro test maestro/upgrade/before.yaml` — `clearState`, `launchApp`, create one
   record through the UI with a fixed, recognisable value.
3. `adb install -r` the new APK (replace, data kept).
4. `maestro test maestro/upgrade/after.yaml` — `launchApp` without `clearState`, wait for
   `app-ready`, assert the record from step 2 is shown.

## 7. CI (rule §5.5)

- PR workflow: ubuntu job, steps as rule §5.5.
- Tag workflow, ubuntu:
  1. Checkout; install pnpm; `pnpm install --frozen-lockfile`.
  2. Set up JDK 17 (Temurin) and the Android SDK Platform 36.
  3. Checks of the PR workflow.
  4. Fail if the tag is not `v` + `version` in `package.json`.
  5. Signing step only: decode the keystore secret to a temp file; `pnpm build`; delete the
     file.
  6. Outputs: `android/app/build/outputs/apk/release/app-release.apk` and
     `android/app/build/outputs/bundle/release/app-release.aab`. Copy both to `dist/`
     as `<slug>-<version>.apk` and `.aab`; fail if either is missing or empty.
  7. APK checks (§5).
  8. Start an emulator (KVM), install the APK with `adb install`, run `maestro test maestro/flows/`,
     then the upgrade flow (this skill §6) on a fresh emulator.
  9. `sha256sum` the APK into `<slug>-<version>.apk.sha256`; `gh release create` with the
     APK and its checksum only; upload the AAB as a workflow artifact, never to the
     release.

## 8. Release

1. Bump `version` in `package.json`, commit.
2. Human approves the release (`GLOBAL_RULES.md` §12), then tag `v<version>` and push.
3. Download the APK from the release page, check its SHA-256, install it on a device or
   emulator, and confirm the about screen shows the tag's version.
4. Store release: the human uploads the AAB to Google Play. A new personal Play account
   first runs the closed test the account requires.

## Unverified

- [verify] `pnpm create expo-app` with `--template default@sdk-58` (the form Expo's SDK 58
  notes give for `npx create-expo-app`), and whether the result works under pnpm's
  `isolated` linker without changes.
- [verify] The `allowBuilds` entries a fresh SDK 58 project actually needs.
- [verify] Whether `android.blockedPermissions` and `android.permissions` behave as
  described on SDK 58.
- [verify] The cleanest way to inject the release `signingConfig` after `prebuild`: a local
  config plugin, or Gradle properties read by the generated `build.gradle`.
- [verify] `node:sqlite` loaded inside Jest (`testEnvironment: "node"`) on the Node LTS in
  `devEngines.runtime`, and SQL behaviour differences from the SQLite bundled by
  `expo-sqlite`.
- [verify] `backupDatabaseAsync` unchanged in SDK 58 (confirmed in `expo-sqlite` 57.0.1).
- [verify] Android emulator with KVM on standard GitHub-hosted ubuntu runners, and its
  run time for the tag workflow.
- [verify] `maestro test <directory>` runs every flow in it; if not, CI lists each flow.
- [verify] Maestro `extendedWaitUntil` with `id`, and passing `-e` variables, on the
  current Maestro release.
- [verify] Gradle output paths and file names after `prebuild` on SDK 58.
- [verify] `adb install -r` keeps app data for an APK signed with the same key, on the
  emulator image used in CI.
- [verify] Whether `jest-expo` and `@testing-library/react-native` versions are picked by
  `pnpm expo install`.
