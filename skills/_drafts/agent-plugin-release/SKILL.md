---
name: agent-plugin-release
description: Set up, check, test and release an agent-plugin repository. Use when creating an agent-plugin project or its walking skeleton; when creating or changing .claude-plugin/marketplace.json, plugin/.claude-plugin/plugin.json, plugin/skills/, plugin/agents/, plugin/hooks/, plugin/.mcp.json, scenarios/, scripts/sync_version.py, scripts/check_bundle.py, scripts/registry_provenance.py, server.json or .github/workflows/; when claude plugin validate fails; when running scenario records; when setting up npm or PyPI trusted publishing or the MCP registry; or when cutting or rolling back a release.
---

# agent-plugin-release

Procedures for `rules/agent-plugin.md` §2, §3, §5 and §6. The rules stay there; this file
says how. Steps marked **(B)** apply only when the project has an executable. Pin every
version below exactly in the project; this file names none.

## 1. New repository (rule §3, §5.2)

1. Create the layout of rule §3. A project with no executable stops at `plugin/`,
   `scenarios/`, `scripts/` and `tests/hooks/`.
2. `plugin/.claude-plugin/plugin.json`:

   ```json
   { "name": "<plugin>", "version": "0.1.0", "description": "<one sentence>",
     "author": { "name": "<owner>" } }
   ```

   `claude plugin validate --strict` fails without `author`.
3. `.claude-plugin/marketplace.json` at the repository root, before the first release:

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

   The version lives only in `plugin.json`; the entry carries no `version`.
4. Local trial and walking skeleton: run the skill's scenario record (§3) with
   `--plugin-dir ./plugin`.

## 2. Skill, agent and hook files (rule §4.1)

- Skill, `plugin/skills/<name>/SKILL.md`:

  ```markdown
  ---
  name: <name>
  description: <what it does>. Use when <the user's situation, in their words>.
  ---
  <instructions>
  ```

  Claude Code lists it as `<plugin>:<name>`.
- Agent, `plugin/agents/<name>.md`: front matter `name`, `description`, `tools` only.
  It is listed as `<plugin>:<name>`.
- Hook entry in `plugin/hooks/hooks.json`:

  ```json
  { "hooks": { "PreToolUse": [ { "matcher": "Write|Edit",
      "hooks": [ { "type": "command", "timeout": 10,
        "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/scripts/<name>.sh\"" } ] } ] } }
  ```

  A blocking hook exits 2 with the reason on stderr. Any other exit, a crash or a timeout
  lets the event pass. So a blocking script wraps its whole body: on any exception it
  prints the reason to stderr and exits 2.
- If the README lists Windows, decide once which hook command and script language run
  there, and record it in `DECISIONS.md`. Hook tests then run on a Windows runner too.

## 3. Scenario records (rule §6)

`scenarios/<name>.md`, one per skill or agent:

```markdown
# <skill>

## <case name>
- Request: <what a user types; does not name the skill>
- Starting files: <none | list>
- Expected trigger: <plugin>:<skill or agent>; no other
- Expected output: <what it produces>
- Tools (agent only): <tools it may use>
- Must not: <what would be wrong>

## unrelated request
- Request: <something nearby the skill must not handle>
- Expected: <plugin>:<skill> does not run
```

Running a record, one case at a time:

1. Make an empty temp directory and put the starting files in it. There must be no
   `CLAUDE.md` in it or above it.
2. From that directory:

   ```bash
   claude -p "<request>" --plugin-dir <repo>/plugin --setting-sources project \
     --output-format stream-json --verbose --no-session-persistence > run.jsonl
   ```

   `--setting-sources project` leaves out the user's settings, and with them the plugins
   the user installed.
3. Check the session first. In the first line with `"type":"system","subtype":"init"`, the
   `plugins` entries whose `path` is not `builtin` must be exactly this plugin, and
   `skills` or `agents` must list `<plugin>:<name>`. If not, the run proves nothing:
   fix the setup and run it again.
4. Read the result. Every `"type":"assistant"` line's `message.content[]` entry with
   `"type":"tool_use"` is one tool call:
   - A skill ran when `name` is `Skill` and `input.skill` is `<plugin>:<name>` (or
     `<name>`).
   - An agent ran when `name` is `Agent` (older versions: `Task`) and
     `input.subagent_type` is the agent.
   - If no line in `run.jsonl` has `"type":"tool_use"`, or the tool names differ from
     these, the host's output changed: stop and report.
5. Compare the files and the final `"type":"result"` line with Expected output, then
   check Must not. Write the three verdicts (trigger, output, must not). The case passes
   only when all three pass.

If the record is run interactively instead, the transcript shows a skill call as
`Skill(<plugin>:<name>)` and an agent call under the agent's name.

Tag annotation at release:

```
v1.2.0
commit <sha>
claude-code <version>: <skill>/<case> pass, ...
<host> <version>: ...
```

## 4. Repository scripts (rule §5.2, §5.3)

- Every script under `scripts/` starts with an inline dependency block (PEP 723) with exact
  versions, and runs as `uv run scripts/<name>.py`.
- `sync_version.py`:
  - Reads `version` from `plugin/.claude-plugin/plugin.json`.
  - Writes it into `package.json` or `pyproject.toml`, `server.json` (`version` and each
    `packages[].version`), the package version in `plugin/.mcp.json`, and each generated
    host manifest.
  - With `--check`, writes nothing and exits 1 on any difference.
- `check_bundle.py` runs these checks in order, collecting every failure before exiting:
  1. `claude plugin validate --strict plugin` and `claude plugin validate --strict .`.
     Both run offline and need no sign-in. `--strict` turns warnings into errors.
  2. Markdown links and `${CLAUDE_PLUGIN_ROOT}/...` paths under `plugin/` resolve.
  3. Skill directories plus agent files match `scenarios/*.md` one to one.
  4. `sync_version.py --check` passes.
  5. `plugin/` holds no `CLAUDE.md`, `.claude/`, `commands/`, top-level `bin/`, test or
     scenario file, and no `SKILL.md` deeper than `plugin/skills/<name>/`.
  6. Every `npx`/`uvx` package argument under `plugin/` carries the source version.
- Generated host manifests: `sync_version.py` writes them from the canonical manifest once
  a second host is adopted. The mapping goes in the ADR that adopts that host.
- `claude` in these scripts is the version pinned for CI (§8). Locally, use the same
  version: `pnpm dlx @anthropic-ai/claude-code@<exact> plugin validate --strict plugin`.

## 5. Hook script tests (rule §6)

- Save one real event payload per case under `tests/hooks/events/`. Capture it once with a
  temporary hook whose command writes stdin to a file.
- In pytest, run the script with
  `subprocess.run([...], input=payload, capture_output=True, timeout=10)`. Assert
  `returncode` (2 for block, 0 for pass) and the stderr reason.
- An unreadable event (empty stdin, invalid JSON) is one more case: a blocking hook exits 2.

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

  `dist/cli.js` starts with `#!/usr/bin/env node`. `repository.url` must name this
  repository: npm checks it against the provenance.
- Python: `[project.scripts] <cmd> = "<pkg>.cli:main"`. The README carries a line
  `mcp-name: io.github.<owner>/<server>` for registry verification.
- `plugin/.mcp.json`:

  ```json
  { "mcpServers": { "<server>": { "command": "npx", "args": ["-y", "<pkg>@1.2.0", "mcp"] } } }
  ```

  - Python: `"command": "uvx", "args": ["<pkg>@1.2.0", "mcp"]`, when the command name
    equals the package name. Otherwise `["--from", "<pkg>==1.2.0", "<cmd>", "mcp"]`.
  - `sync_version.py` writes the version into both forms.
- Packed-artifact check:
  1. `pnpm pack` (or `uv build`), then install the tarball or wheel into a temp directory
     and run `<cmd> --version`.
  2. A Vitest or pytest test starts `<cmd> mcp` over stdio with the SDK client, calls
     list-tools, closes, and asserts the process exited.
  3. List the artifact's files (`tar -tzf` or `unzip -l`) and compare them with the
     `files` allow-list or the wheel's expected contents.
  4. Run the secret check on the unpacked artifact.
- Manual tool check: `pnpm dlx @modelcontextprotocol/inspector@<exact> <cmd> mcp`.

## 7. Registries, one-time setup (B)

1. **PyPI.** Add a pending trusted publisher (repository, workflow file, environment)
   before the first release.
2. **npm.** A trusted publisher can be set only on a package that exists.
   1. First version only: create a granular token limited to publishing and store it as a
      CI secret. Release v0.1.0 through §9.
   2. Then, on the package's settings page, add the trusted publisher: this repository,
      `release.yml`, and the environment if one is used.
   3. Delete the secret and revoke the token.
   - A trusted-publisher configuration created after 3 September 2026 allows only
     `npm stage publish` until the maintainer also allows direct publishing in it. The
     release flow here publishes directly, so allow it there.
   - A new configuration expires unless a publish succeeds within 2 days. Cut the next
     release soon after setting it up.
3. **MCP registry.**
   1. Run `mcp-publisher init`. It writes `server.json` from `package.json` and the Git
      remote, with namespace `io.github.<owner>/<server>`.
   2. Delete the placeholder `environmentVariables` entry it adds (`YOUR_API_KEY`) and
      replace the placeholder description.
   3. Run `mcp-publisher validate`. It sends `server.json` to the registry, so it needs
      network access.
   - `mcp-publisher` comes from the `modelcontextprotocol/registry` GitHub releases
     (`mcp-publisher_<os>_<arch>.tar.gz`). Record the version and SHA-256 in
     `DECISIONS.md`.

`scripts/registry_provenance.py`, used by the publish job (§8):

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["cryptography==<exact>"]
# ///
"""Usage: registry_provenance.py npm|pypi <package> <version> <owner/repo> <commit>

Exit 0: published, and every file's provenance names <owner/repo> and <commit>.
Exit 10: not published.
Exit 1: published without matching provenance.
"""
import base64, json, sys, urllib.error, urllib.parse, urllib.request
from cryptography import x509

REPO_OID = x509.ObjectIdentifier("1.3.6.1.4.1.57264.1.12")    # Source Repository URI
COMMIT_OID = x509.ObjectIdentifier("1.3.6.1.4.1.57264.1.13")  # Source Repository Digest

def get(url: str, accept: str = "application/json") -> dict | None:
    req = urllib.request.Request(url, headers={"Accept": accept})
    try:
        with urllib.request.urlopen(req, timeout=30) as r:
            return json.load(r)
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise

def der_utf8(raw: bytes) -> str:
    if raw[0] != 0x0C:
        raise ValueError("unexpected DER tag")
    n = raw[1]
    return raw[(2 if n < 0x80 else 2 + (n & 0x7F)):].decode()

def npm(pkg: str, version: str) -> list[tuple[str, str]] | None:
    if get(f"https://registry.npmjs.org/{pkg.replace('/', '%2f')}/{version}") is None:
        return None
    att = get(f"https://registry.npmjs.org/-/npm/v1/attestations/{urllib.parse.quote(pkg, safe='')}@{version}")
    found = []
    for a in (att or {}).get("attestations", []):
        if a["predicateType"].startswith("https://slsa.dev/provenance/"):
            st = json.loads(base64.b64decode(a["bundle"]["dsseEnvelope"]["payload"]))
            bd = st["predicate"]["buildDefinition"]
            found.append((bd["externalParameters"]["workflow"]["repository"],
                          bd["resolvedDependencies"][0]["digest"]["gitCommit"]))
    return found

def pypi(project: str, version: str) -> list[tuple[str, str]] | None:
    meta = get(f"https://pypi.org/pypi/{project}/{version}/json")
    if meta is None:
        return None
    found = []
    for f in meta["urls"]:
        prov = get(f"https://pypi.org/integrity/{project}/{version}/{f['filename']}/provenance",
                   "application/vnd.pypi.integrity.v1+json")
        if prov is None:
            return []  # a file without provenance
        for b in prov["attestation_bundles"]:
            for a in b["attestations"]:
                cert = x509.load_der_x509_certificate(base64.b64decode(a["verification_material"]["certificate"]))
                found.append((der_utf8(cert.extensions.get_extension_for_oid(REPO_OID).value.value),
                              der_utf8(cert.extensions.get_extension_for_oid(COMMIT_OID).value.value)))
    return found

registry, pkg, version, repo, commit = sys.argv[1:6]
found = {"npm": npm, "pypi": pypi}[registry](pkg, version)
if found is None:
    print(f"{pkg}@{version}: not published"); sys.exit(10)
want = (f"https://github.com/{repo}", commit)
print(f"{pkg}@{version}: provenance {found or 'none'}")
sys.exit(0 if found and all(f == want for f in found) else 1)
```

- npm keeps the repository and commit in the SLSA provenance statement. PyPI keeps them
  in the signing certificate of each file's attestation.
- Before relying on it, run it once against a package known to have provenance (for
  example the MCP SDK on npm and on PyPI). If it fails there, the registry's format
  changed: stop and report.

## 8. CI (rule §5.5)

- **Claude Code in CI**, for `plugin validate`: install pnpm standalone, then run
  `pnpm dlx @anthropic-ai/claude-code@<exact> plugin validate --strict ...`.
  - The exact version is pinned in the workflow.
  - It needs no sign-in.
- **`ci.yml`, push / PR:** install uv, run the bundle check and the hook tests.
  **(B)** Also: pnpm or uv frozen install, lint, type check, build, tests, and the
  packed-artifact check.
- **`release.yml`, tag:**
  1. The opening checks of rule module-deployment §5.3, then the checks of `ci.yml`.
  2. Fail unless the tag is `v` + the source version.
  3. **(B)** One publish job, the only one with `permissions: id-token: write`:
     1. Check the package:
        `uv run scripts/registry_provenance.py <npm|pypi> <pkg> <version> "$GITHUB_REPOSITORY" "$GITHUB_SHA"`.
        - Exit 0: already published by this commit, so skip to step 3.
        - Exit 10: publish it (step 2).
        - Any other exit: fail.
     2. Publish:
        - npm: `pnpm publish --no-git-checks`. pnpm does the OIDC exchange itself, and
          provenance is added automatically for a public package from a public
          repository.
        - PyPI: `uvx pypi-attestations@<exact> sign dist/*`, then `uv publish`. uv uploads
          the `.publish.attestation` files but does not create them, and without them
          step 1 can never match on a re-run.
        - Then run step 1 again; it must exit 0.
     3. **(MCP server)** Check the registry:

        ```bash
        name=$(python3 -c 'import json,urllib.parse;print(urllib.parse.quote(json.load(open("server.json"))["name"],safe=""))')
        code=$(curl -s -o /dev/null -w '%{http_code}' "https://registry.modelcontextprotocol.io/v0.1/servers/$name/versions/$VERSION")
        ```

        - `200`: already there, so skip. Only this repository's GitHub OIDC identity can
          publish in its namespace.
        - `404`: download `mcp-publisher` and check its SHA-256 against `DECISIONS.md`.
          Then `mcp-publisher login github-oidc` and `mcp-publisher publish`.
        - Anything else: fail.

## 9. Release

1. Bump `version` in `plugin.json`, run `uv run scripts/sync_version.py`, and commit.
2. Run every scenario record on every supported host at that commit (§3).
3. Ask the human to approve the release, naming the version, the commit SHA and what
   changed (rule module-deployment §5.4).
4. Create the annotated tag on that SHA (`git tag -a v<version> <sha>`) with the results of
   step 2. Confirm `git rev-parse v<version>^{commit}` is the approved SHA, then push it.
5. Wait for the tag workflow. If any job fails: stop, report, and leave the pin. After the
   cause is fixed and the human approves, re-run the same workflow; steps already done
   skip themselves (§8).
6. Update `ref` and `sha` in `.claude-plugin/marketplace.json`, commit to `main`, and push.
7. Confirm:
   1. `claude plugin marketplace update <marketplace>`.
   2. Install or update the plugin in a clean configuration and check its version.
   3. **(B)** Run the `.mcp.json` launch command with `--version`.

Rollback: with the human's approval, set `ref` and `sha` back to the previous tag and
confirm as in step 7.
