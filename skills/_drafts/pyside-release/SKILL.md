---
name: pyside-release
description: DRAFT. Setting up the build toolchain, building, packaging, and the CI jobs for a pc-app-python app.
---

<!-- Extracted from rules/pc-app-python.md §2 and §5 in step 4. Rewrite in step 7. -->

## Prerequisites

| Item | Detail |
|---|---|
| C compiler | Windows: VS Build Tools (C++ workload), MSVC on `PATH` via `vcvarsall.bat`. macOS: Xcode CLT. Linux: gcc/clang + ccache |
| `build` group | nuitka, ordered-set, zstandard, patchelf (Linux only) |
| Inno Setup | `iscc` on `PATH`, Windows only |
| `appimagetool` | Linux only. Pin a tagged release; record version and SHA-256 in `DECISIONS.md` |
| Qt libraries | Linux only: `libxcb-cursor0`, `libxkbcommon-x11-0`, `libegl1`, `libgl1`. Qt 6.5+ will not start without `libxcb-cursor0` |

Verify the compiler (and `iscc` on Windows) before building.

## Locking the toolchain

- Without a locked `build` group, `pyside6-deploy` pip-installs Nuitka at build time, which
  fails under uv or escapes `uv.lock`.
- Add `pip` to the `build` group if `pyside6-deploy` still needs it importable.
- Set the Nuitka version in `pysidedeploy.spec` to the one in `uv.lock`.

## Build

```
uv sync --locked --group build
uv run pyside6-deploy -c src/<app>/pysidedeploy.spec -f
```

`--dry-run` prints the Nuitka command without building.

## Package

`scripts/package.py`, run after the build on each OS:

| OS | Action | Output |
|---|---|---|
| Windows | `iscc installer/windows.iss` | `<app>-<version>-setup.exe` |
| macOS | zip the `.app` | `<app>-<version>-macos.zip` |
| Linux | standalone dir → `appimagetool` | `<app>-<version>-x86_64.AppImage`, or `<app>-<version>-linux-x86_64.tar.gz` if the ADR chose a tarball |

## CI

- Push / PR (Linux): install the Qt libraries above, then `uv sync --locked`, Ruff, mypy,
  pytest offscreen with the boundary check.
- Version tag, one job per declared target OS: verify compiler (and `iscc`) → test → build
  → `--self-check` → `scripts/package.py` → attach to the Release. Publish only if every
  job succeeds.
