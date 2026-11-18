---
name: lockfile-conflict
description: Resolve a conflict in uv.lock or pnpm-lock.yaml. Use when git merge, rebase or cherry-pick reports "CONFLICT" in uv.lock or pnpm-lock.yaml, when a lockfile contains <<<<<<< markers, or when `uv lock --locked` or `pnpm install --frozen-lockfile` fails right after a merge or rebase.
---

# lockfile-conflict

The lockfile is never edited by hand and never deleted. One side's lockfile is taken
whole, and the tool rebuilds it with the other side's change.

## 1. Which side is "incoming"

The incoming side is the branch being integrated into — usually `main`.

| Operation | Incoming side |
|---|---|
| `git merge main` on your branch | `--theirs` |
| `git rebase main` | `--ours` (during a rebase the sides are swapped) |
| `git cherry-pick <commit>` | `--ours` (the branch you are on) |

Below, `<side>` is the flag from this table.

## 2. Resolve

1. List the conflicted files: `git diff --name-only --diff-filter=U`.
2. **Manifest also conflicted** (`pyproject.toml` or `package.json` in the list):
   1. Take both files from the incoming side:
      `git checkout <side> pyproject.toml uv.lock` or
      `git checkout <side> package.json pnpm-lock.yaml`.
   2. Reapply this branch's dependency change with the tool, not by editing the file:
      `uv add <pkg>==<version>` / `uv remove <pkg>`, or
      `pnpm add -E <pkg>@<version>` / `pnpm remove <pkg>`. Take the package and version
      from this branch's own commit (`git show <commit> -- pyproject.toml` or
      `-- package.json`).
3. **Only the lockfile conflicted** (the manifest merged cleanly):
   1. `git checkout <side> uv.lock` or `git checkout <side> pnpm-lock.yaml`.
   2. Regenerate: `uv lock` or `pnpm install`.
4. Confirm: `uv lock --locked` or `pnpm install --frozen-lockfile` exits 0, and
   `grep -c '<<<<<<<' <lockfile>` prints 0.
5. Run the project's tests.
6. `git add` the manifest and lockfile, then `git commit` (merge) or
   `git rebase --continue` / `git cherry-pick --continue`.

## 3. When it does not resolve

- The tool cannot resolve the versions both sides require: stop and report the two
  requirements and the tool's error. Choosing between them is the human's decision.
- Never fix it by loosening a version pin that was not part of this branch's change.
