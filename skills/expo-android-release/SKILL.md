---
name: expo-android-release
description: Set up, build, sign, test and release a mobile (Expo, Android) app. Use when creating a mobile project or its walking skeleton; when creating or changing app.config.ts, plugins/with-unsigned-release-in-ci.ts, pnpm-workspace.yaml, jest.config.ts, tests/adapters/node-sqlite-connection.ts, maestro/flows/, maestro/upgrade/, the build script or .github/workflows/; when adding an Expo module (its permissions); when pnpm install fails with ERR_PNPM_IGNORED_BUILDS; when creating or backing up the upload keystore; when the APK permission or signature check fails; or when cutting a release.
---

# expo-android-release

Procedures for `rules/mobile.md` §2, §4 and §5. The rules stay there; this file says how.
Pin every version below exactly in the project; this file names none, except the SDK
major from rule §1.

## 1. Toolchain setup (rule §2)

1. Install pnpm standalone (the exact version in `packageManager`). `pnpm install` then
   installs the Node version in `devEngines.runtime`.
2. Install JDK 17 and Android Studio. In Android Studio's SDK Manager install SDK Platform
   36, Build-Tools, Platform-Tools and the Emulator, and create one emulator.
3. Set `JAVA_HOME` to the JDK 17 directory and `ANDROID_HOME` (Windows:
   `%LOCALAPPDATA%\Android\Sdk`), and add `platform-tools` to `PATH`. Confirm with
   `java -version` and `adb --version`.
4. Maestro: download `maestro.zip` from the release tagged `cli-<version>` on Maestro's
   GitHub releases, extract it to a stable path, add its `bin` to `PATH`, and confirm with
   `maestro --version`.
5. New project, from the parent directory:

   ```
   pnpm create expo-app <name> --template default@sdk-58 --no-agents-md
   ```

   - `--no-agents-md` stops the template from writing `AGENTS.md` (which tells agents to
     use `npx`) and `.claude/settings.json` (which turns on a marketplace plugin). If
     either exists, delete it.
   - Inside an existing Git repository no nested repository is created. Outside one,
     the result is a new repository with one commit.
   - The generated `.gitignore` already ignores `/android`, `/ios` and `dist/`.
6. In the project:

   ```
   echo n | node scripts/reset-project.js
   ```

   This deletes the example screens and creates an empty `src/app/`. Then remove the
   `reset-project` script from `package.json`. The example files fail `tsc --noEmit`.
7. `pnpm-workspace.yaml`:

   ```yaml
   allowBuilds:
     "@parcel/watcher": false
   ```

   - `@parcel/watcher` comes with Jest and has a prebuilt binary, so its script is not
     needed.
   - When `pnpm install` fails with `ERR_PNPM_IGNORED_BUILDS` for another package, add it
     as `false`. Set it to `true` only when something is shown to fail without it, and
     record why in `DECISIONS.md`.
   - The default `isolated` linker works: `pnpm expo export --platform android` bundles
     the template.
8. Add packages with `pnpm expo install <package>` (dev: `pnpm expo install -- -D <package>`).
   - It picks the SDK's version for packages the SDK knows, such as `jest-expo`. For any
     other package, such as `jest` or `@testing-library/react-native`, it falls back to a
     plain `pnpm add`.
   - Start with `expo-dev-client`, `expo-sqlite`, `expo-secure-store`,
     `expo-build-properties`, then `-D jest-expo jest @testing-library/react-native @types/jest`.
   - Without network access to Expo's API it fails. `EXPO_OFFLINE=1` uses the SDK's bundled
     version list instead.
9. Delete `app.json`; `app.config.ts` (§2) replaces it. In `tsconfig.json`, set
   `"allowImportingTsExtensions": true` (the plugin import in §2 needs it).
10. First run: start the emulator, then `pnpm expo run:android`. The first prebuild rewrites
    the `android` and `ios` scripts in `package.json` to `expo run:*`; commit that.

## 2. app.config.ts and the CI plugin (rule §4.1, §4.2, §5.2)

```ts
import type { ExpoConfig } from "expo/config";
import pkg from "./package.json";
import withUnsignedReleaseInCi from "./plugins/with-unsigned-release-in-ci.ts";

const USED_PERMISSIONS: string[] = [];          // the single list rule §4.1 refers to
const BLOCKED_PERMISSIONS: string[] = [];       // added by the template or plugins, not used
const DEV_ONLY_PERMISSIONS: string[] = ["android.permission.INTERNET"]; // the development build reaches Metro
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
    blockedPermissions: process.env.CI ? [...BLOCKED_PERMISSIONS, ...DEV_ONLY_PERMISSIONS] : BLOCKED_PERMISSIONS,
    allowBackup: ALLOW_BACKUP,
  },
  plugins: ["expo-router", "expo-sqlite", "expo-secure-store"],
};

export default withUnsignedReleaseInCi(config);
```

Permissions:

- `android.permissions` only adds permissions. It does not remove those the template's
  manifest already has: `INTERNET`, `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`,
  `SYSTEM_ALERT_WINDOW` and `VIBRATE`. Every one of them the app does not use goes into
  `BLOCKED_PERMISSIONS`, which prebuild writes as `tools:node="remove"`.
- `INTERNET` stays allowed on developer machines so the development build can reach Metro.
  CI builds (`CI` set) remove it. If the app has a network feature, move `INTERNET` to
  `USED_PERMISSIONS` and leave `DEV_ONLY_PERMISSIONS` empty.
- Libraries add more permissions when Gradle merges manifests. The APK check (§5) prints
  the final list. Every permission there that is not in `USED_PERMISSIONS` goes into
  `BLOCKED_PERMISSIONS`. Do this after the first CI build and after adding any module.
- `CI` must be `true`, `false`, `1` or `0`; Expo refuses an empty `CI`.

`plugins/with-unsigned-release-in-ci.ts`:

```ts
import { type ConfigPlugin, withAppBuildGradle } from "expo/config-plugins";

const DEBUG_SIGNING = /(release\s*\{[^}]*?)\n\s*signingConfig signingConfigs\.debug/;

/** In CI (`CI` set), the release build type gets no signing config: Gradle builds unsigned files. */
const withUnsignedReleaseInCi: ConfigPlugin = (config) =>
  withAppBuildGradle(config, (cfg) => {
    if (!process.env.CI) return cfg;
    const src = cfg.modResults.contents;
    if (!DEBUG_SIGNING.test(src)) {
      throw new Error("with-unsigned-release-in-ci: release signingConfig line not found in app/build.gradle");
    }
    cfg.modResults.contents = src.replace(DEBUG_SIGNING, "$1");
    return cfg;
  });

export default withUnsignedReleaseInCi;
```

- It is imported in `app.config.ts`, with the `.ts` extension. Expo's config loader does
  not find `./plugins/<name>` for a `.ts` file in the `plugins` array.
- Check after `CI=true pnpm expo prebuild --clean --platform android --no-install`: the
  `release { … }` block in `android/app/build.gradle` has no `signingConfig`, and
  `debug { … }` still has one. If the plugin throws, the generated file changed shape in
  a new SDK: stop and report.

## 3. Upload keystore (rule §2, §5.2)

Once, on the developer's machine, outside the repository:

```
keytool -genkeypair -v -storetype PKCS12 -keystore upload.keystore -alias upload -keyalg RSA -keysize 2048 -validity 10000
```

It is one line, so it runs unchanged in CMD, PowerShell and Bash.

- Back up the file and both passwords to a place off this machine before the first
  release. Record where in `DECISIONS.md`: the location, never the passwords.
- Fingerprint to pin in `release.yml`, as lowercase hex with no colons:

  ```bash
  keytool -list -v -keystore upload.keystore -alias upload | grep 'SHA256:' | sed 's/.*SHA256: //' | tr -d ':' | tr 'A-F' 'a-f'
  ```

- CI secrets: `ANDROID_KEYSTORE_BASE64` (the file, base64), `ANDROID_KEYSTORE_PASSWORD`,
  `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`.

## 4. SQLite test connection (rule §4.3, §6)

`src/adapters/sqlite/connection.ts` declares the subset of `expo-sqlite` in use:

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

A transaction receives only `SqlExecutor`: nested transactions are not part of the
contract. Values always go in `params`, never into `sql`.

`tests/adapters/node-sqlite-connection.ts`:

```ts
import { DatabaseSync } from "node:sqlite";
import type { SqlConnection, SqlExecutor, SqlParam } from "../../src/adapters/sqlite/connection";

export function openTestConnection(): SqlConnection {
  const db = new DatabaseSync(":memory:");
  db.exec("PRAGMA foreign_keys=ON");
  const exec: SqlExecutor = {
    async execAsync(sql) { db.exec(sql); },
    async runAsync(sql, params: SqlParam[]) {
      const r = db.prepare(sql).run(...params);
      return { changes: Number(r.changes), lastInsertRowId: Number(r.lastInsertRowid) };
    },
    async getAllAsync<T>(sql: string, params: SqlParam[]) { return db.prepare(sql).all(...params) as T[]; },
    async getFirstAsync<T>(sql: string, params: SqlParam[]) {
      return (db.prepare(sql).get(...params) as T | undefined) ?? null;
    },
  };
  return {
    ...exec,
    async withExclusiveTransactionAsync(task) {
      db.exec("BEGIN IMMEDIATE");
      try { await task(exec); db.exec("COMMIT"); } catch (e) { db.exec("ROLLBACK"); throw e; }
    },
  };
}
```

- Every adapter test opens a fresh connection and runs `migrations.ts` on it first.
- `node:sqlite` prints an `ExperimentalWarning`; that is expected.
- This double is not `expo-sqlite`. The SQLite version differs (Node bundles its own), and
  so can type conversion and error codes. The upgrade flow (§6) is what proves migrations
  on the real engine.

Backup before a migration (rule §4.3). `backupDatabaseAsync` copies into an *open*
database, not a path:

```ts
const dest = await openDatabaseAsync(`${DB_NAME}.backup-v${fromVersion}`);
await backupDatabaseAsync({ sourceDatabase: db, destDatabase: dest });
const { user_version } = (await dest.getFirstAsync<{ user_version: number }>("PRAGMA user_version", []))!;
await dest.closeAsync();
if (user_version !== fromVersion) throw new Error("backup does not report the old version");
```

`jest.config.ts`:

```ts
import type { Config } from "jest";

const config: Config = {
  projects: [
    {
      displayName: "core",
      testEnvironment: "node",
      roots: ["<rootDir>/tests/core", "<rootDir>/tests/adapters"],
      transform: { "^.+\\.tsx?$": ["babel-jest", { presets: ["babel-preset-expo"] }] },
    },
    { displayName: "ui", preset: "jest-expo", roots: ["<rootDir>/tests/ui"] },
  ],
};

export default config;
```

`pnpm exec jest` must report both projects ("Ran all test suites in 2 projects").

## 5. Build, sign and check (rule §5.2, §5.3)

`package.json` script, run in CI (Linux):

```json
"build": "tsc --noEmit && expo prebuild --clean --platform android --no-install && cd android && ./gradlew :app:assembleRelease :app:bundleRelease"
```

- On a developer machine use `pnpm expo run:android --variant release` instead (debug-signed,
  rule §5.2).
- Outputs: exactly one `.apk` in `android/app/build/outputs/apk/release/` and exactly one
  `.aab` in `android/app/build/outputs/bundle/release/`. Copy them to
  `dist/<slug>-<version>-unsigned.apk` and `.aab`. Unsigned APKs are usually named
  `app-release-unsigned.apk`; find the file by extension and fail if the count is not 1.

Signing job (rule §5.2). The tools come from the runner:

```bash
BT="$ANDROID_HOME/build-tools/$(ls -1 "$ANDROID_HOME/build-tools" | sort -V | tail -1)"
test -x "$BT/apksigner" && test -x "$BT/zipalign" || { echo "build-tools missing" >&2; exit 1; }
```

Steps, with the keystore written to `$RUNNER_TEMP/upload.keystore` by `base64 -d`:

1. Check alignment of the unsigned APK: `"$BT/zipalign" -c -P 16 4 <unsigned.apk>`.
2. Sign the APK:

   ```bash
   "$BT/apksigner" sign --ks "$RUNNER_TEMP/upload.keystore" --ks-key-alias "$ANDROID_KEY_ALIAS" \
     --ks-pass env:ANDROID_KEYSTORE_PASSWORD --key-pass env:ANDROID_KEY_PASSWORD \
     --out <signed.apk> <unsigned.apk>
   ```

3. Sign the AAB:

   ```bash
   jarsigner -keystore "$RUNNER_TEMP/upload.keystore" -storepass:env ANDROID_KEYSTORE_PASSWORD \
     -keypass:env ANDROID_KEY_PASSWORD -sigalg SHA256withRSA -digestalg SHA-256 \
     -signedjar <signed.aab> <unsigned.aab> "$ANDROID_KEY_ALIAS"
   ```

4. Delete the keystore in a step with `if: always()`.

Checks on the signed files (rule §5.3). `PINNED` is the fingerprint from §3:

```bash
# APK: apksigner exits non-zero on an unsigned or broken APK
apk_fp=$("$BT/apksigner" verify --print-certs <signed.apk> | sed -n 's/^Signer #1 certificate SHA-256 digest: //p')
[ "$apk_fp" = "$PINNED" ] || { echo "APK signer $apk_fp != pinned" >&2; exit 1; }

# AAB: jarsigner -verify exits 0 even on an unsigned file, so check its text
jarsigner -verify <signed.aab> | grep -qx 'jar verified.' || { echo "AAB not verified" >&2; exit 1; }
aab_fp=$(keytool -printcert -jarfile <signed.aab> | sed -n 's/^[[:space:]]*SHA256: //p' | head -1 | tr -d ':' | tr 'A-F' 'a-f')
[ "$aab_fp" = "$PINNED" ] || { echo "AAB signer $aab_fp != pinned" >&2; exit 1; }
```

- Do not add `-strict` to `jarsigner -verify`: it exits 4 on every self-signed upload key.
- Manifest, from `aapt2 dump badging <signed.apk>` (aapt2 is in the same build-tools
  directory):
  - The output has a `package:` line, otherwise stop: the dump failed.
  - `versionCode='…'` and `versionName='…'` on that line equal §2's values.
  - There is no `application-debuggable` line.
  - Declared permissions are the `name='…'` values on lines that start with
    `uses-permission:` or `uses-permission-sdk-23:`. Ignore `uses-implied-permission:`
    lines. Print the sorted set, and fail unless it equals `USED_PERMISSIONS`.
- Contents: unzip the APK to a temp dir, fail on any `.env*` or `*.map` file, and run the
  project's secret check over the directory.

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

- Check every flow with `maestro check-syntax <file>` before committing it.
- Run with the emulator's network off (`adb shell svc wifi disable`,
  `adb shell svc data disable`):

  ```
  timeout 120 maestro test -e APP_VERSION=<version> maestro/flows/
  ```

  `maestro test` accepts a folder and runs the flow files in it; it does not descend into
  subfolders. `timeout` exits non-zero when rule §5.3's 120 s run out.

Upgrade flow (rule §6), from the second release on:

1. On a fresh emulator, `gh release download v<previous> -p '*.apk'` and `adb install` it.
2. `maestro test maestro/upgrade/before.yaml`: `clearState`, `launchApp`, then create one
   record through the UI with a fixed, recognisable value.
3. `adb install -r <new.apk>`. This replaces the app and keeps its data, which works only
   because both APKs are signed with the same key.
4. `maestro test maestro/upgrade/after.yaml`: `launchApp` without `clearState`, wait for
   `app-ready`, and assert the record from step 2 is shown.

## 7. CI (rule §5.5)

- **`ci.yml`, push / PR:** one `ubuntu-24.04` job with the steps of rule §5.5.
- **`release.yml`, tag.** Start with the opening checks of rule module-deployment §5.3.
  Every job uses OS version labels only.
  1. **Build job:**
     - Checkout, install pnpm and Node from `package.json`, then
       `pnpm install --frozen-lockfile`.
     - JDK 17 from the runner: set `JAVA_HOME` to `$JAVA_HOME_17_X64` and check that
       `java -version` reports 17.
     - Run the checks of `ci.yml`.
     - Fail unless the tag is `v` + `version` in `package.json`:
       `test "v$(node -p "require('./package.json').version")" = "$GITHUB_REF_NAME"`.
     - `pnpm build` (the runner sets `CI=true`). Then upload the unsigned files as
       workflow artifacts.
  2. **Signing job** (rule §5.2, this skill §5). It checks out no code. The base64
     keystore is set only on the decode step, and the passwords only on the signing step.
     It runs the checks of §5 and uploads the signed files.
  3. **Device job** (`ubuntu-24.04`, x86_64; ARM runners have no `/dev/kvm`):
     1. Enable KVM for the runner user:

        ```bash
        echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
        sudo udevadm control --reload-rules
        sudo udevadm trigger --name-match=kvm
        test -w /dev/kvm || { echo "no KVM" >&2; exit 1; }
        ```

     2. Create the emulator with the runner's SDK:
        `sdkmanager "system-images;android-36;google_apis;x86_64" "emulator"`, then
        `avdmanager create avd -n ci -k "system-images;android-36;google_apis;x86_64" --force`.
     3. Start it: `emulator -avd ci -no-window -no-audio -no-boot-anim &`.
     4. Wait for boot with a limit:

        ```bash
        timeout 300 bash -c 'adb wait-for-device; until [ "$(adb shell getprop sys.boot_completed | tr -d "\r")" = 1 ]; do sleep 2; done'
        ```

     5. Install Maestro from `cli-<version>/maestro.zip`. Check its SHA-256 against
        `DECISIONS.md` before running it.
     6. `adb install` the signed APK, then run the smoke and other flows (§6).
     7. From the second release on, run the upgrade flow on a new emulator (`-wipe-data`).
  4. **Publish job** (Linux), needing every other job:
     1. `sha256sum` the APK into `SHA256SUMS`.
     2. `gh release create "$TAG" --draft --verify-tag --title "$TAG" --notes-file notes.md`.
     3. `gh release upload "$TAG" <apk> SHA256SUMS`.
     4. `gh release edit "$TAG" --draft=false`.
     5. `gh release download "$TAG" -D check && (cd check && sha256sum -c SHA256SUMS)`.

     The AAB stays a workflow artifact and never goes on the release. Only this job gets
     `contents: write`.

## 8. Release

1. Follow the version proposal of rule module-deployment §5.4.
2. Bump `version` in `package.json` and commit.
3. Tag locally and confirm `git rev-parse v<version>^{commit}` is the approved SHA before
   pushing.
4. Download the APK from the release page, check it with `SHA256SUMS`, install it on a
   device or emulator, and confirm the about screen shows the tag's version.
5. Store release: the human uploads the AAB to Google Play. A new personal Play account
   first runs the closed test the account requires.
