---
name: agent-plugin-release
description: Set up an agent-plugin repository (marketplace, plugin/ bundle, scenario records, bundle check, version sync), test hook scripts, set up trusted publishing and the MCP registry for an executable, configure CI, and cut or roll back a release. Use when creating or changing marketplace.json, plugin.json, hooks, scenarios/, scripts/, server.json, CI workflows, or when releasing.
---

# agent-plugin-release (draft)

Raw procedures moved out of `rules/agent-plugin.md`. The rules they serve stay in that file
(§2, §3, §5, §6). This draft leaves `_drafts/` only when the Unverified list at the end is
empty. Steps marked (B) apply only when the project has an executable.

## 1. New repository (rule §3, §5.2)

- Create the layout of rule §3. A project with no executable stops at `plugin/`,
  `scenarios/`, `scripts/`, `tests/hooks/`.
- `plugin/.claude-plugin/plugin.json`:

  ```json
  { "name": "<plugin>", "version": "0.1.0", "description": "<one sentence>" }
  ```

- `.claude-plugin/marketplace.json` at the repository root, before the first release:

  ```json
  {
    "name": "<marketplace>",
    "owner": { "name": "<owner>" },
    "description": "<one sentence>",
    "plugins": [
      { "name": "<plugin>",
        "source": { "source": "git-subdir", "url": "<owner>/<repo>", "path": "plugin",
                    "ref": "v0.1.0", "sha": "<40-char SHA of v0.1.0>" } }
    ]
  }
  ```

  Version lives only in `plugin.json`; the entry carries no `version`.
- Local trial and walking skeleton: `claude --plugin-dir ./plugin`, then run the skill's
  scenario record.

## 2. Skill, agent, hook files (rule §4.1)

- Skill:

  ```markdown
  ---
  description: <what it does>. Use when <the user's situation, in their words>.
  ---
  <instructions>
  ```

- Agent: front matter `name`, `description`, `tools` only.
- Hook entry in `plugin/hooks/hooks.json`:

  ```json
  { "hooks": { "PreToolUse": [ { "matcher": "Write|Edit",
      "hooks": [ { "type": "command", "timeout": 10,
        "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/scripts/<name>.sh\"" } ] } ] } }
  ```

  A blocking hook exits 2 with the reason on stderr. Any other exit, a crash or a timeout
  lets the event pass, so a blocking script wraps its whole body: on any exception it
  prints the reason to stderr and exits 2.
- Windows listed in the README: the hook command and script language that run there are
  decided once and recorded in `DECISIONS.md`; hook tests run on `windows-latest` too.

## 3. Scenario record (rule §6)

`scenarios/<name>.md`, one per skill or agent:

```markdown
# <skill>

## <case name>
- Request: <what a user types; does not name the skill>
- Starting files: <none | list>
- Expected: <skill or agent that runs; what it produces>
- Tools (agent only): <tools it may use>
- Must not: <what would be wrong>

## unrelated request
- Request: <something nearby the skill must not handle>
- Expected: <skill> does not run
```

Running a record: new session in an empty directory holding the starting files, only this
plugin enabled, no `CLAUDE.md`. Type the request, compare with Expected and Must not, write
pass or fail.

Tag annotation at release:

```
v1.2.0
claude-code <version>: <skill>/<case> pass, ...
<host> <version>: ...
```

## 4. Repository scripts (rule §5.2, §5.3)

- Both scripts start with an inline dependency block (PEP 723) with exact versions and run
  as `uv run scripts/<name>.py`.
- `sync_version.py`: read `plugin/.claude-plugin/plugin.json` `version`; write it into
  `package.json` or `pyproject.toml`, `server.json` (`version` and each `packages[].version`),
  the package version in `plugin/.mcp.json`, and each generated host manifest. With `--check`, write nothing and exit 1 on any
  difference.
- `check_bundle.py`, in this order, collecting every failure before exiting:
  1. `claude plugin validate --strict plugin` and `claude plugin validate --strict .`
  2. Markdown links and `${CLAUDE_PLUGIN_ROOT}/...` paths under `plugin/` resolve.
  3. Skill directories plus agent files and `scenarios/*.md` match one to one.
  4. `sync_version.py --check`.
  5. No `CLAUDE.md`, `.claude/`, `commands/`, top-level `bin/`, test or scenario file under
     `plugin/`, and no `SKILL.md` deeper than `plugin/skills/<name>/`.
  6. Every `npx`/`uvx` package argument under `plugin/` carries the source version.
- Generated host manifests: written by `sync_version.py` from the canonical manifest when a
  second host is adopted; the mapping goes in the ADR adopting that host.

## 5. Hook script tests (rule §6)

- Save one real event payload per case under `tests/hooks/events/` (capture it once with a
  hook that writes stdin to a file).
- pytest: run the script with `subprocess.run([...], input=payload, capture_output=True,
  timeout=10)`, assert `returncode` (2 for block, 0 for pass) and the stderr reason.

## 6. Executable (B) (rule §3, §5.2, §5.3)

- TypeScript `package.json`:

  ```json
  { "name": "<pkg>", "version": "0.1.0", "type": "module",
    "bin": { "<cmd>": "dist/cli.js" }, "files": ["dist", "LICENSE"],
    "packageManager": "pnpm@<exact>",
    "devEngines": { "runtime": { "name": "node", "version": "<lts>", "onFail": "download" } },
    "mcpName": "io.github.<owner>/<server>",
    "repository": { "type": "git", "url": "git+https://github.com/<owner>/<repo>.git" } }
  ```

  `dist/cli.js` starts with `#!/usr/bin/env node`.
- Python: `[project.scripts] <cmd> = "<pkg>.cli:main"`; the README carries a line
  `mcp-name: io.github.<owner>/<server>` for registry verification.
- `plugin/.mcp.json`:

  ```json
  { "mcpServers": { "<server>": { "command": "npx", "args": ["-y", "<pkg>@1.2.0", "mcp"] } } }
  ```

  Python: `"command": "uvx", "args": ["<pkg>@1.2.0", "mcp"]` (the command name equals the package name; otherwise `["--from", "<pkg>==1.2.0", "<cmd>", "mcp"]`).
  `sync_version.py` writes the version into both forms.
- Packed-artifact check: `pnpm pack` (or `uv build`), install the tarball or wheel into a temp
  directory, run `<cmd> --version`, then a Vitest/pytest test that starts `<cmd> mcp` over
  stdio with the SDK client, calls list-tools, closes, and asserts the process exited.
- Manual tool check: `pnpm dlx @modelcontextprotocol/inspector@<exact> <cmd> mcp`.

## 7. Registries, one-time setup (B)

1. PyPI: add a pending trusted publisher (repository, workflow file, environment) before the
   first release.
2. npm, first version only: create a granular token limited to publishing, store it as a CI
   secret, release v0.1.0 through §9, then configure the trusted publisher on the package
   page, delete the secret and revoke the token.
3. MCP registry: `mcp-publisher init` to create `server.json`; namespace
   `io.github.<owner>/<server>`; `mcp-publisher validate`.

## 8. CI (rule §5.5)

- PR workflow: install uv; bundle check; hook tests. (B) pnpm/uv frozen install, lint, type
  check, build, tests, packed-artifact check.
- Tag workflow:
  1. Checks of the PR workflow.
  2. Fail if the tag is not `v` + the source version.
  3. (B) One publish job, the only one with `permissions: id-token: write`:
     a. If the version is not on npm or PyPI: `pnpm publish` or `uv publish`.
     b. (MCP server) If the version is not in the MCP registry: download `mcp-publisher` at
        the version and SHA-256 recorded in `DECISIONS.md`, `mcp-publisher login
        github-oidc`, `mcp-publisher publish`.
- Claude Code in CI, for `plugin validate`: installed at the exact version pinned in the
  workflow (method under Unverified).

## 9. Release

1. Bump `version` in `plugin.json`; `uv run scripts/sync_version.py`; commit.
2. Run every scenario record on every supported host at that commit.
3. Ask the human to approve the release, naming the version and what changed.
4. Create the annotated tag with the results of step 2; push it.
5. Wait for the tag workflow. If any job fails: stop, report, leave the pin. After the
   cause is fixed and the human approves, re-run the same workflow; steps already done
   skip themselves (§8).
6. Update `ref` and `sha` in `.claude-plugin/marketplace.json`; commit to `main`; push.
7. Confirm: `claude plugin marketplace update <marketplace>`, then install or update the
   plugin in a clean configuration and check its version; (B) run the `.mcp.json` launch
   command with `--version`.

Rollback: set `ref` and `sha` back to the previous tag, with the human's approval, and
confirm as in step 7.

## Unverified

- [verify] `claude plugin validate --strict` on the marketplace root with a `git-subdir`
  entry, and whether it runs in CI without signing in.
- [verify] How to install a pinned Claude Code version on a GitHub-hosted runner: the
  native installer with a version argument, or the npm package (which would put Node in
  CI for a project that has none).
- [verify] Whether a crash (non-zero exit other than 2) and a timeout of a plugin hook let
  the event pass, for each event the plugin uses.
- [verify] Hook command execution on native Windows: which shell runs `command`.
- [verify] A `git-subdir` source whose `url` is the marketplace's own repository: install,
  update to a new `sha`, and `claude plugin marketplace update` behaviour.
- [verify] Whether Codex, reading `.claude-plugin/marketplace.json`, honours `git-subdir`
  with `sha`.
- [verify] `timeout` unit and default for plugin hook entries, and exit-code 2 semantics for
  every event the plugin uses.
- [verify] `pnpm publish` with trusted publishing on pnpm 11 from GitHub Actions, without a
  token and without the npm CLI.
- [verify] `uv publish` trusted publishing flags on the current uv release.
- [verify] `mcp-publisher` release asset names and checksum file for the Linux runner.
- [verify] `mcp-publisher init` output for a PyPI package, and the README `mcp-name` line.
- [verify] Where to capture a real hook event payload (stdin dump) and whether the payload
  shape differs between Claude Code versions.
