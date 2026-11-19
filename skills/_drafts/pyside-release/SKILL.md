---
name: pyside-release
description: Set up the build toolchain, lock it, build with pyside6-deploy, run --self-check on the built app, package it (Inno Setup, macOS zip, AppImage) and configure CI for a pc-app-python app. Use when creating or changing pysidedeploy.spec, the build dependency group or [tool.uv] required-environments in pyproject.toml, installer/windows.iss, scripts/package.py, CI workflows, when a build or packaging step fails, or when cutting a release.
---

# pyside-release

Procedures for `rules/pc-app-python.md` §2 and §5. `<app>` is the package name under
`src/`, `<title>` the `title` in `pysidedeploy.spec`.

## 1. Check the prerequisites (rule §2)

Run the check for the OS before any build. A failing check stops the build; install the
item, then check again.

| OS | Install | Check |
|---|---|---|
| Windows | Visual Studio Build Tools 2022 or later, workload "Desktop development with C++". Nuitka uses it when installed; no `PATH` change is needed | `"%ProgramFiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe" -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath` prints a path |
| Windows | Inno Setup | `where iscc` prints a path. If not, add the Inno Setup install directory to `PATH` |
| macOS | `xcode-select --install` | `clang --version` |
| Linux | gcc (or clang), ccache, and the Qt libraries: `sudo apt-get install libxcb-cursor0 libxkbcommon-x11-0 libegl1 libgl1` (Debian/Ubuntu names) | `gcc --version` |
| Linux | `appimagetool` and the AppImage runtime, §6.3 | `sha256sum` matches `DECISIONS.md` |

MinGW is not used: Nuitka downloads it itself and it does not support Python 3.13 or later.
If Nuitka, during a build, asks to download a compiler or any other tool, answer no, stop,
and report which tool it asked for. MUST NOT add `--assume-yes-for-downloads`.

## 2. Lock the toolchain (rule §5.1)

1. Build group, exact versions; `patchelf` only on Linux (it has no Windows wheel):

   ```
   uv add --group build "nuitka==<v>" "ordered-set==<v>" "zstandard==<v>" "patchelf==<v>; sys_platform == 'linux'"
   ```

2. `pyproject.toml`, one line per OS that `README.md` lists as a target:

   ```toml
   [tool.uv]
   required-environments = [
       "sys_platform == 'win32' and platform_machine == 'AMD64'",
       "sys_platform == 'darwin' and platform_machine == 'arm64'",
       "sys_platform == 'linux' and platform_machine == 'x86_64'",
   ]
   ```

   Then `uv lock`. A failure names the package with no build for an OS. Report it; do not
   drop the OS from the list.

## 3. pysidedeploy.spec (rule §5.1, §3)

1. `uv run pyside6-deploy --init src/<app>/main.py` creates `src/<app>/pysidedeploy.spec`.
2. Edit the generated file:

   | Key | Value | Why |
   |---|---|---|
   | `[app] title` | the app name | names the output directory |
   | `[app] exec_directory` | `../../dist` | output goes outside `src/<app>/` (rule §3); the default `.` puts it inside |
   | `[app] icon` | path to the icon under `src/<app>/resources/`, or empty | the generated value is an absolute path into `.venv` |
   | `[python] packages` | `Nuitka==<version in uv.lock>` | the generated value is an older version |
   | `[nuitka] mode` | `standalone` | rule §1 |

3. Add `src/<app>/deployment/` to `.gitignore`. Nuitka writes its intermediate files there
   and deletes them after a successful build, but a failed build leaves them.
4. Add this test. `--no-install` (§4) checks only that Nuitka is installed, not its
   version, so the test is what keeps the spec and `uv.lock` equal:

   ```python
   # tests/test_build_spec.py
   import configparser
   import tomllib
   from pathlib import Path

   ROOT = Path(__file__).resolve().parents[1]
   SPEC = ROOT / "src" / "<app>" / "pysidedeploy.spec"


   def test_spec_nuitka_version_matches_lock() -> None:
       spec = configparser.ConfigParser()
       spec.read(SPEC, encoding="utf-8")
       pinned = dict(p.strip().split("==") for p in spec["python"]["packages"].split(","))
       lock = tomllib.loads((ROOT / "uv.lock").read_text(encoding="utf-8"))
       locked = {p["name"]: p["version"] for p in lock["package"]}
       assert pinned["Nuitka"] == locked["nuitka"]
   ```

5. Commit the spec.

## 4. Build (rule §5.2)

```
uv sync --locked --group build
uv run pyside6-deploy -c src/<app>/pysidedeploy.spec -f --no-install
```

- `--no-install` is always given. Without it `pyside6-deploy` runs pip to install or
  replace Nuitka at build time. A missing package then fails with its name: run
  `uv sync --locked --group build`, never install it by hand.
- `--dry-run` prints the Nuitka command without building.
- Output: `dist/<title>.dist/`, executable named after the entry file (`main.bin` on
  Linux); `dist/<title>.app` on macOS. Expect about a minute on Linux, longer on a first
  Windows build.

## 5. Self-check of the build (rule §5.3)

Run the built executable, not `uv run`:

- Linux, without a display: `QT_QPA_PLATFORM=offscreen dist/<title>.dist/main.bin --self-check`
- Windows, macOS: the executable in `dist/` with `--self-check`

Exit code 0 passes. Anything else is a failed build; report the printed reason.

## 6. Package (rule §5.4)

`scripts/package.py` does all of it. It reads `version` from `pyproject.toml` with
`tomllib`, finds the build output in `dist/` and fails unless exactly one exists, runs the
step for the current OS, and fails if the expected output file is missing.

### 6.1 Windows

`iscc /DAppVersion=<version> installer/windows.iss`. The `.iss` file takes `AppVersion`
from that define, installs per user, and sets `OutputBaseFilename=<app>-<version>-setup`.
Its uninstall section MUST NOT delete the data directory without asking.

### 6.2 macOS

`ditto -c -k --keepParent dist/<title>.app dist/<app>-<version>-macos.zip`. `ditto` keeps
the symlinks and attributes inside the bundle; a plain `zip` can break them.

### 6.3 Linux AppImage

`appimagetool` downloads the AppImage runtime from a changing `continuous` release unless
given one. Both files are pinned:

1. Once per project: download `appimagetool-x86_64.AppImage` from a tagged release of
   `AppImage/appimagetool` and `runtime-x86_64` from a dated release of
   `AppImage/type2-runtime`. Record both release names and SHA-256 values in
   `DECISIONS.md` (skill `adr`). Store them outside the repository, or fetch them in CI by
   that release name and check the SHA-256 before use.
2. AppDir: copy `dist/<title>.dist/` to `build/AppDir/`, then in it create the symlink
   `AppRun → main.bin`, `<app>.desktop` (`Type=Application`, `Name`, `Exec=main.bin`,
   `Icon=<app>`, `Categories=`) and `<app>.png`.
3. `ARCH=x86_64 appimagetool --runtime-file runtime-x86_64 build/AppDir dist/<app>-<version>-x86_64.AppImage`.
   The output MUST NOT contain "Downloading runtime".
4. Without FUSE (containers, some CI runners), run either tool with
   `--appimage-extract-and-run` as its first argument.
5. Check: `dist/<app>-<version>-x86_64.AppImage --self-check`, with the §5 environment.

If the ADR chose a tarball: `tar -czf dist/<app>-<version>-linux-x86_64.tar.gz -C dist <title>.dist`.

## 7. CI (rule §5.5)

- Push / PR, one Linux job: install the Qt libraries of §1 → `uv sync --locked` → Ruff →
  mypy → pytest with `QT_QPA_PLATFORM=offscreen` (includes the boundary check and
  `test_build_spec.py`).
- Version tag, one job per target OS in `README.md`: §1 check → `uv sync --locked --group
  build` → tests → §4 build → §5 self-check → `uv run python scripts/package.py` → upload
  the artifact. A last job, which needs all OS jobs, creates the GitHub Release with every
  artifact. No OS job publishes on its own.
- A job whose §1 check fails stops there and reports the missing item. It MUST NOT
  install a different compiler to get past it.
