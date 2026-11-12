# rules/agent-plugin.md

A bundle installed into an AI coding tool (the host): skills, agents and hooks the host's
model reads, and — only when the host cannot do the work — an executable, a CLI or an MCP
server, published to a package registry.

`GLOBAL_RULES.md` → this file → `CLAUDE.md` → current task. This file holds only what
differs from, narrows, or adds to `GLOBAL_RULES.md`. **Default** = use it; **Alternative** =
allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited column covers technology
choices only; every other prohibition sits with the rule it belongs to.

Wrong archetype if it needs its own window (`pc-app-python`, `pc-app-electron`) or its own
site or API (a web archetype). Instructions for one repository only are that repository's
`CLAUDE.md` and `.claude/`, not a plugin.

**Components.** Every project has the bundle. An executable is added only when the
capability does not exist in the host: a binary file format, OCR, an authenticated API,
anything needing a library. Rules marked **(B)** apply only to a project that has an
executable.

---

## 1. Stack

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| Bundle format | Markdown with YAML front matter; every skill and agent file has a `description` | — | Instructions inside code strings |
| Components | `skills/<name>/SKILL.md`, `agents/<name>.md`, `hooks/hooks.json` | — | New files in `commands/` (superseded by skills); a `CLAUDE.md` in the bundle (not loaded); a top-level `bin/` (host-specific; claude.ai and Cowork refuse the plugin) |
| Hosts | Claude Code; its manifest is canonical | Further hosts, each with a manifest generated from the canonical one | Hand-maintained manifest copies |
| Hooks | Check, block, log | — | A hook that approves a tool call |
| Bootstrap scripts | None | A script that must run before any runtime exists: PowerShell and POSIX `sh` as a matched pair | A script where a skill is enough |
| Validation | `claude plugin validate --strict`, plus the bundle check (§5.3) | — | — |
| Behaviour tests | Scenario records in Markdown, run by hand in a live host (§6) | — | A host-specific eval tool as the only record |
| Repository tooling | Python scripts with inline dependencies, run by `uv run`; pytest for hook scripts | — | Adding Node to a project with no TypeScript executable |
| Distribution | GitHub repository with `.claude-plugin/marketplace.json`; the entry pinned to a release tag and its SHA | A third-party marketplace listing, in addition | A marketplace entry that follows a branch; files distributed by hand |
| Language **(B)** | TypeScript, `strict: true`, on a Node.js LTS line installed by pnpm (`devEngines.runtime`) | Python 3.12+ with uv, when the work needs Python libraries | A language the users cannot run without a toolchain |
| Packaging **(B)** | pnpm, standalone, exact version in `packageManager`; `pnpm-lock.yaml`. Python: `uv.lock` | — | npm, yarn, bun, Corepack; `npx` in `package.json` scripts, CI or developer docs |
| MCP SDK **(B)** | `@modelcontextprotocol/server` (v2); Python `mcp` 2.x (`MCPServer`) | `@modelcontextprotocol/sdk` (v1) for an existing server only | `mcp<2` in a new server; the separate `fastmcp` package; framework wrappers |
| Transport **(B)** | stdio | Streamable HTTP, for a server that must be remote | HTTP+SSE |
| Lint / type check **(B)** | Biome, `tsc --noEmit`. Python: Ruff, mypy strict on `core/` | ESLint + Prettier; pyright | — |
| Tests **(B)** | Vitest. Python: pytest | — | Jest; testing only through the host's model |
| Registries **(B)** | npm or PyPI by trusted publishing (OIDC); the official MCP registry for a server | — | A long-lived publish token |

---

## 2. Prerequisites

Nothing in the repository installs or reveals these. Procedure: skill `agent-plugin-release`.

| Item | Scope |
|---|---|
| Claude Code | System. Runs `plugin validate` and the scenario records |
| Each further supported host | System, with the account its model needs |
| pnpm **(B)** | System, standalone. Also needed for the MCP Inspector when the executable is Python |
| Node.js **(B)** | Installed by pnpm from `devEngines.runtime` |
| npm or PyPI account **(B)** | Account, with the repository's tag workflow registered as trusted publisher |
| MCP registry namespace **(B)** | `io.github.<owner>/`, proven through GitHub |

---

## 3. Layout

```
<repo>/
├── .claude-plugin/marketplace.json   one entry: git-subdir, path plugin, ref and sha of the release
├── plugin/                           the only directory users receive
│   ├── .claude-plugin/plugin.json    manifest; version — the single source
│   ├── skills/<name>/SKILL.md
│   ├── agents/<name>.md
│   ├── hooks/hooks.json
│   ├── hooks/scripts/                what hooks.json runs, nothing else
│   └── .mcp.json                     (B) launches the published package
├── scenarios/<name>.md               scenario records, one file per skill or agent (§6)
├── scripts/                          sync_version.py, check_bundle.py
├── tests/                            hooks/; (B) core/, cli/, mcp/, boundaries
├── src/                              (B) core/, cli/, mcp/
├── package.json, pnpm-lock.yaml      (B, TypeScript) — or pyproject.toml, uv.lock (B, Python)
└── server.json                       (B, MCP server) registry entry
```

- Everything a user needs is in `plugin/`, and nothing else is: no tests, scenarios, notes
  or build files.
- A project with no executable has no `src/`, package files or `server.json`, and MUST NOT
  add them to match this layout.
- A directory a supported host claims for itself — at least one host reads a root `rules/`
  — MUST NOT be used for anything else in `plugin/`.

| Directory **(B)** | MAY import | MUST NOT import |
|---|---|---|
| `src/core/` | the standard library, `src/core/`, packages that need no I/O | the MCP SDK, the argument parser, `src/cli/`, `src/mcp/`; the network, filesystem or clock except through parameters |
| `src/cli/` | `src/core/`, the argument parser | the MCP SDK, `src/mcp/` |
| `src/mcp/` | `src/core/`, the MCP SDK | the argument parser, `src/cli/` |

- `tests/boundaries` fails on any import that breaks this table. That test is the boundary
  check CI and §8 refer to.
- A CLI command or MCP tool handler parses input, calls one `core/` function, and formats the
  result. A rule written in `cli/` or `mcp/` is a defect.

---

## 4. Runtime

### 4.1 Instructions and the host

- The host picks a skill by its `description`. The description states what the skill does
  and when to use it, in the words a user would use. One skill does one job.
- Where an executable exists **(B)**, the skill calls it and stays thin. If the executable
  is missing, the skill says what to install and stops; it MUST NOT improvise the
  capability in shell commands.
- A hook fires on every matching event in every session once the plugin is enabled, not
  only when its skill runs. Its matcher is no wider than its job, and every hook entry
  sets `timeout`. A hook script reads the event, decides, and exits; it does no other work.
- Only an explicit block stops an event; a hook that crashes or times out lets it pass. A
  hook that blocks harm therefore fails closed: an unreadable event, a missing tool or
  unexpected input is answered with a block and its reason. It does nothing that can
  approach its timeout.
- A shipped agent definition has no `model` field — it inherits the user's session model —
  and lists `tools` explicitly, no more than its job needs.
- **(B)** stdout belongs to the protocol. Logs and diagnostics go to stderr; a stray write
  to stdout breaks the connection.
- **(B)** Few tools, each doing one thing, described for a model choosing between them.
  Every outbound call has a timeout. `<PLUGIN>_OFFLINE=1` (the plugin name in upper case,
  `-` as `_`) disables every outbound call; a tool that needs the network then returns an
  error saying so. Unset, the executable makes only the calls the README declares.

### 4.2 Paths

- Files in the bundle are referenced through `${CLAUDE_PLUGIN_ROOT}`, quoted in shell-form
  hook commands. Absolute paths, `~` and paths relative to the working directory are
  PROHIBITED.
- The bundle's directory is read-only at runtime: it changes with every version.
- **(B)** The executable reaches only paths its caller passed in. It does not search the
  user's home or project on its own.
- Hook scripts run on every OS the README lists, and their tests run in CI on each of
  them. A POSIX-only script on a Windows-supported plugin is a defect.

### 4.3 Persistence

- The bundle keeps no state by default. State that must survive a session goes under
  `${CLAUDE_PLUGIN_DATA}`, which survives updates, as JSON carrying a `schema` number; a
  value the plugin cannot read is left untouched and the plugin runs on defaults.
- The plugin writes into the user's project only what the user asked a skill to produce.
- **(B)** Caches are deletable without loss.

### 4.4 Settings, credentials, errors

- Values the user must supply are declared in the manifest's `userConfig`. Telling the user
  to edit a host's settings file by hand is PROHIBITED.
- The plugin holds no credentials. An executable that needs a user's credential for another
  service is decided by ADR before it is built. The bundle contains no tokens.
- The project has no deployment secrets: publishing uses OIDC. The one exception is the
  first npm version (§5.4).
- A skill whose prerequisite is missing names it and says how to get it.
- **(B)** CLI: exit 0 on success; non-zero with a message on stderr. MCP: a failed tool call
  returns a tool error stating what failed and what to try.
- The plugin makes no network call the README does not declare.

---

## 5. Build and package

Procedure: skill `agent-plugin-release`.

### 5.1 Lock the toolchain

- The Claude Code version CI validates with is pinned in the workflow.
- Repository scripts declare their dependencies inline with exact versions.
- **(B, TypeScript)** `packageManager` and `devEngines.runtime` are the single source for
  the pnpm and Node versions. Every install outside first setup uses
  `pnpm install --frozen-lockfile`. Install scripts run only for packages in `allowBuilds`.
- **(B, Python)** Every install outside first setup uses `uv sync --locked`.

### 5.2 Build

- A project with no executable has no build: the bundle ships as written.
- `version` in `plugin/.claude-plugin/plugin.json` is the single source.
  `scripts/sync_version.py` writes it into `package.json` or `pyproject.toml`, `server.json`,
  the launch command in `plugin/.mcp.json`, and every generated host manifest; those files are never edited by hand.
- **(B, TypeScript)** `pnpm build` runs `tsc --noEmit`, then compiles to `dist/`.
  `package.json` `files` lists only `dist/` and the licence; `bin` points at a `dist/`
  entry. **(B, Python)** `uv build`; the command is a `[project.scripts]` entry.
- **(B)** `plugin/.mcp.json` launches the published package at the exact version released
  with it (`npx -y <pkg>@<version>`; `uvx <pkg>@<version>`), written by `sync_version.py`. A
  version range is PROHIBITED: it would hand users a package before the pin moves. These
  run on the
  user's machine, where pnpm cannot be assumed.
- Before any feature beyond it is built, a walking skeleton — one skill, its scenario record,
  and the executable if there is one — is loaded locally (`--plugin-dir`) and passes its
  record on every supported host; **(B)** the packed-artifact check (§5.3) passes. The
  marketplace is first used at the first release.

### 5.3 Self-check

The bundle check (`scripts/check_bundle.py`) runs offline and exits 0, or non-zero naming
each failure. It checks that:

- `claude plugin validate --strict` passes for `plugin/` and for the marketplace;
- every relative link and every `${CLAUDE_PLUGIN_ROOT}` path in `plugin/` resolves;
- every skill and agent has a scenario record, and every record names an existing one;
- every file `sync_version.py` writes holds the source version;
- `plugin/` contains nothing §3 excludes and nothing §1 prohibits, and the project's secret
  check passes over it;
- every launch command and skill names a package at the source version — no `@latest`, no range.

**(B)** After the build, the packed artifact is installed into a temporary directory and,
offline: `<cmd> --version` prints the source version; an MCP server is started over stdio,
a test client lists its tools and closes, and the process exits.

### 5.4 Package

| Output | Where it goes |
|---|---|
| `plugin/` at the tagged commit | Users, through the marketplace pin |
| **(B)** the package | npm or PyPI |
| **(B, MCP server)** `server.json` | The official MCP registry |

- A release is one approval for two steps. The agent pushes the tag; CI checks, then
  publishes the package and the registry entry. Only after every job succeeds does the
  agent commit the marketplace pin — `ref` the tag, `sha` its commit — to `main`. If CI
  fails, the pin does not move and users keep the previous version.
- Each publish step first looks up its version on its registry and skips if it is already
  there. After a partial failure the tag workflow is re-run, with the human's approval, and
  resumes where it stopped; a partial failure is never a reason for a new version. A failed
  pin commit is simply retried.
- After release, confirm: the marketplace file on `main` names the tag; a fresh install from
  the marketplace shows the new version; **(B)** the launch command from `.mcp.json` prints
  it.
- A published package version cannot be replaced; a fix is a new version. Rolling users back
  is moving the pin to the previous tag, which is itself a release.
- npm accepts a trusted publisher only for a package that exists. The first version is
  published by CI with a short-lived granular token, revoked once trusted publishing is set.
- The README states the prerequisite, the supported hosts with each host's install command,
  the manual fallback, and every network call the plugin makes.

### 5.5 CI

- Push / PR: the bundle check; hook script tests. **(B)** frozen install, lint, type check,
  build, tests including the boundary check, the packed-artifact check.
- Version tag: all of the above, a check that the tag equals the source version, then
  one publish job — **(B)** the package, then **(B, MCP server)** the registry entry — the
  only job with `id-token: write`.
- Scenario records are not run in CI: they need a live host and a paid model.

---

## 6. Testing

| Layer | Tool | Live host |
|---|---|---|
| Bundle files | bundle check (§5.3) | no |
| Hook scripts | pytest: a recorded event on stdin, assert exit code and output | no |
| Skills and agents | scenario records, by hand | yes |
| **(B)** `src/core/` | Vitest or pytest with fakes | no |
| **(B)** `src/cli/`, `src/mcp/` | CLI by arguments, stdout and exit code; MCP through the SDK's client | no |
| **(B)** packed artifact | §5.3 | no |

- A scenario record holds: the request, phrased as a user would without naming the skill;
  the starting files; the expected observable result — which skill ran, what it produced —
  and what MUST NOT happen. An agent's record also names the agent expected to run and the
  tools it may use. Each skill and agent has at least one record whose request is unrelated
  and MUST NOT trigger it.
- A record is written before its skill is added or its behaviour changed, and run then.
  Before a release, every record is run on every supported host at the tagged commit, in a
  session with no other plugin and no project instructions that could produce the result.
  The results — host, host version, pass or fail per record — go in the tag's annotation.
- Every hook that blocks has a test where it blocks and one where it lets the event pass.
- **(B)** One test asserts the tool list — names, descriptions, input schemas — so that a
  change to what the model sees is deliberate.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. MCP SDK v1 imports in new code: `from mcp.server.fastmcp import FastMCP` and
   `@modelcontextprotocol/sdk/server/mcp.js`. v2 is `from mcp.server import MCPServer` and
   `@modelcontextprotocol/server`.
2. pnpm 9/10 setting names (`onlyBuiltDependencies`). pnpm 11 reads `allowBuilds`.
3. A skill that does not trigger "fixed" by a hook that injects its instructions into every
   prompt. The fix is the description.
4. A credential the plugin may not hold obtained by telling the user to export an
   environment variable, or by a `userConfig` field the README calls optional.
5. The marketplace entry switched to a branch or a relative path to release faster.

---

## 8. Definition of Done — additions

Every task here:

- [ ] Bundle check passes
- [ ] Nothing added to `plugin/` that users do not need

New skill or agent, or a change to what one does:

- [ ] Its scenario records were written first and run on Claude Code; results reported
      per record

Hook change:

- [ ] Script tests pass for blocking and passing events, and for an unreadable event,
      which a blocking hook blocks; matcher no wider than the job; `timeout` set

**(B)** any change:

- [ ] Boundary check (§3) and the packed-artifact check pass

**(B)** change to the tool list:

- [ ] Tool-list test updated in the same commit; the scenarios that use the tool re-run

Release:

- [ ] Every scenario record run on every supported host at the tagged commit, results in the
      tag annotation; pin moved only after CI succeeded; fresh install shows the new version
