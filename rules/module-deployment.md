# rules/module-deployment.md

How GitHub Actions gets the code, permissions and secrets it needs to check, build, release
and deploy — and nothing more.

`GLOBAL_RULES.md` → `rules/<archetype>.md` and this file → `CLAUDE.md` → current task. The
archetype rule says what is released, where it goes, and how to roll it back; this file says
how the workflows that do it are written. Where both speak, the stricter applies. This file
holds only what `GLOBAL_RULES.md` and the archetype rules do not already say. **Default** =
use it; **Alternative** = allowed with an ADR; **Prohibited** = MUST NOT. The §1 Prohibited
column covers technology choices only; every other prohibition sits with the rule it
belongs to.

---

## 1. Stack

**Applies to** every project from the commit that adds its first file under
`.github/workflows/`, release or not: a workflow that only runs tests still executes
third-party code with a token. Sections §2–§4 and §5.1–§5.3 apply to every archetype. §5.4
applies where a release publishes files people download.

| Archetype | A release publishes | Release secret | Downloaded files (§5.4) |
|---|---|---|---|
| `pc-app-python`, `pc-app-electron` | Installers, one job per declared OS → GitHub Release | None | Yes |
| `web-static` | `dist/` → host, by the archetype's deploy step | Host deploy token | No |
| `web-python`, `web-ts`, `web-api` | Host deploy of the tagged commit | Deploy hook URL | No |
| `mobile` | Signed APK → GitHub Release; AAB → workflow artifact | Upload keystore and its passwords | APK only |
| `agent-plugin` | Package to npm or PyPI, entry in the MCP registry | None: OIDC (first npm version excepted, per the archetype) | No: registry provenance |

| Category | Default | Alternative | Prohibited |
|---|---|---|---|
| CI | GitHub Actions on GitHub-hosted runners | — | Self-hosted runners |
| Action references | Full 40-character commit SHA in `uses:`, its release tag as a trailing comment | Workflow lockfile (`.github/workflows/actions.lock`, `gh actions-lock`) | Tags, branches, short SHAs |
| Action updates | Dependabot, `github-actions` ecosystem, with a cooldown of at least 7 days | — | — |
| Workflow check | zizmor, offline audits, version pinned in the workflow | — | — |
| Secret scan | Gitleaks CLI, tagged release binary | Betterleaks | gitleaks-action |
| Release record | GitHub Release, immutable; drafted, assets and `SHA256SUMS` attached, then published | — | Third-party release actions |
| Versioning | SemVer from Conventional Commits; the agent proposes, the human approves | — | semantic-release, and any tool that tags or publishes on merge |

---

## 2. Prerequisites

Procedure: skill `ci-setup`.

Settings live on GitHub, not in the repository, so no commit records them. The agent checks
them read-only with `gh api` before the first workflow lands; the human changes them.

| Item | Scope |
|---|---|
| Actions policy "Require actions to be pinned to a full-length commit SHA" | Repository setting, every project |
| Workflow permissions: read repository contents; "Allow GitHub Actions to create and approve pull requests" off | Repository setting, every project |
| Immutable releases | Repository setting, archetypes that publish GitHub Release files. Not verified on private Free repositories; if unavailable, `DECISIONS.md` records it |
| `release` environment: deployment tags `v*` only, required reviewer the owner, holding every release secret | Public repository only |
| Agent's Git credential without push access | Private repository on GitHub Free (§4.3) |
| zizmor | CI only, `uvx zizmor==<version>` |
| Gitleaks binary | CI only. A tagged release, never `latest`; version and SHA-256 recorded in `DECISIONS.md` |

---

## 3. Layout

```
<repo>/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml        push to main, pull_request: checks only
│   │   └── release.yml   push of a v* tag: checks, build, publish or deploy
│   └── dependabot.yml    github-actions plus the project's package ecosystems
└── .gitleaks.toml        only when the default rules need an allowlist
```

- The archetype rule's "Push / PR" list runs in `ci.yml`; its "Version tag" list runs in
  `release.yml`. A third workflow needs an ADR naming why neither fits.
- Inside `release.yml`, files are built in jobs that hold no secret and no write permission,
  and handed on as artifacts of the same run. The job that publishes them runs no project
  code and installs nothing. Where the archetype rule names a step that must run project
  tooling with a secret — the host deploy step, APK signing — that step is the only one
  holding it.

---

## 4. Runtime

### 4.1 Triggers and untrusted input

- `ci.yml`: `push` to `main`, `pull_request`. `release.yml`: `push` of tags `v*`, and
  `workflow_dispatch` only to re-run a release of an existing tag with the human's
  approval.
- `pull_request_target`, `workflow_run`, and any trigger fired by a comment are
  PROHIBITED: they run with the base repository's secrets and write token.
- `release.yml` sets `concurrency` with `cancel-in-progress: false`; two releases never run
  at once and none is cut off halfway.
- Event fields — branch name, tag name, PR title and body, commit message — are external
  input. A `run:` script never contains `${{ github.event.* }}`, `${{ github.head_ref }}`
  or a step output inline; it reads them from `env:`, quoted.

### 4.2 Permissions

Every workflow starts with `permissions: {}`. Each job declares only what it uses:

| Job | `permissions` |
|---|---|
| Checks, builds, deploy by token or hook | `contents: read` |
| Creating a GitHub Release and uploading to it | `contents: write` |
| Registry publish by OIDC | `contents: read`, `id-token: write` |

- `write-all`, `read-all`, and a job with no `permissions` block are PROHIBITED.
- `actions/checkout` runs with `persist-credentials: false`. No workflow pushes to the
  repository.

### 4.3 Secrets

- Secrets appear only in `release.yml`, only in the `env:` of the step that uses them —
  never in a job's or the workflow's `env:`, never through `secrets: inherit` or
  `toJSON(secrets)`. `ci.yml` references none: a pull request from a branch of the same
  repository would receive them.
- A value derived from a secret is masked with `::add-mask::` before anything can print it.
- A registry or host that accepts OIDC gets OIDC, not a stored token.
- **Public repository:** every release secret is a secret of the `release` environment;
  repository secrets hold none. The release job waits for the owner's approval in GitHub,
  which shows the commit — a tag pushed from an unmerged branch is caught there.
- **Private repository on GitHub Free:** environments do not exist there, so any workflow
  on any pushed branch can read every secret. The agent's Git credential therefore has no
  push access; the agent prepares commits and tags, and the human pushes them.
  `DECISIONS.md` records this once.

### 4.4 Third-party code

- Every `uses:` names an action repository's commit by full SHA, resolved from that
  repository's release tag when written, with the tag as a trailing comment
  (`# v6.0.2`). `docker://` references are pinned by digest. Local actions (`./…`) are
  exempt.
- An action is added only when a `run:` step with a tool the runner already has — `gh`,
  `sha256sum`, the project's own toolchain — cannot do the same job. Releases are created
  and uploaded with `gh release`.
- Dependabot update PRs are reviewed and merged by the human like any dependency change.
- `release.yml` restores no cache: no `cache:` input on setup actions, no `actions/cache`.
  A cache written by a pull-request run could be poisoned. `ci.yml` MAY cache.
- A binary downloaded in a workflow is checked against the SHA-256 recorded in
  `DECISIONS.md` before it runs.

---

## 5. Build and package

Procedure: skill `ci-setup`.

### 5.1 Lock the toolchain

- Action SHAs (§4.4), the zizmor version and the Gitleaks version are pinned in the
  workflow files. Changing any of them is a dependency change.
- `runs-on` names an image version label, never `*-latest`: the image under a release must
  not change between two releases of the same code.

### 5.2 Build

- `release.yml` builds from the tagged commit's checkout. Artifacts from `ci.yml` runs or
  earlier releases are never reused.
- Each job that produces downloadable files also writes their SHA-256 in `sha256sum`
  format; the publish job merges them into one `SHA256SUMS`.

### 5.3 Self-check

Run in `ci.yml` on every push and pull request, before the archetype's checks:

- zizmor over `.github/`, failing on any finding. An ignore is an inline
  `# zizmor: ignore[<audit>]` with the reason on the same line.
- Gitleaks over the full history (`fetch-depth: 0`). An allowlist entry in
  `.gitleaks.toml` covers a path under `tests/` only, with its reason.

`release.yml` runs Gitleaks over the tagged tree before anything is built — the secret
check `GLOBAL_RULES.md` §12 requires.

### 5.4 Package

For archetypes that publish downloaded files:

- Order: draft release at the tag (`gh release create --draft --verify-tag`); attach every
  file and `SHA256SUMS`; publish only after every job of the run succeeded. Nothing is
  attached or replaced after publishing; a defective release is followed by a new version.
- After publishing, CI downloads the assets back and runs `sha256sum -c SHA256SUMS`.
- `README.md` shows how to check a download: `sha256sum -c` on Linux and macOS,
  `Get-FileHash` on Windows, or `gh release verify-asset`.

For every archetype:

- The agent proposes the next version from the Conventional Commits since the previous tag
  — a breaking change raises major (minor before 1.0.0), `feat` raises minor, anything
  else patch — together with the commit SHA and release notes grouped the same way. The
  human approves that version and that commit.

### 5.5 CI

- Push / PR: §5.3, then the archetype's list.
- Version tag: Gitleaks over the tree, the archetype's list, then publishing — the jobs
  holding release secrets on `environment: release` in a public repository — then §5.4's
  download check where it applies.

---

## 6. Testing

Workflows have no unit tests; the checks of §5.3 are their tests.

- Before committing a workflow change, the agent runs `uvx zizmor==<version> .github/`
  locally.
- `release.yml` runs only on a release. Its first run is the archetype's walking-skeleton
  release. Every later change to it is named in the next version proposal, so the human
  approves that release knowing the workflow changed.

---

## 7. Where agents go wrong

Only what the sections above do not already say: idioms from older versions, and ways
around a prohibition when something does not work.

1. Old workflow commands — `::set-output`, `::save-state`, `::set-env`, `::add-path` —
   instead of `$GITHUB_OUTPUT`, `$GITHUB_STATE`, `$GITHUB_ENV`, `$GITHUB_PATH`. Action
   majors and the archived `actions/create-release` and `actions/upload-release-asset`,
   recalled from training.
2. A SHA typed from memory. It may not exist, or may belong to a fork, where GitHub still
   resolves it under the original name.
3. Answering `Resource not accessible by integration` by widening permissions for the whole
   workflow, instead of adding the one scope to the one job that needs it.
4. Moving a check to `pull_request_target`, or putting a secret into `ci.yml`, so that a
   pull request can reach a live service. A check that needs a live service is run by the
   human.
5. Turning a failing scan green: `continue-on-error`, `|| true`, an allowlist entry or an
   ignore comment written to get past it.
6. Replacing a SHA with a tag because the pinning policy blocked an action that calls
   another action by tag. Choose a different action, or do the step in `run:`.

---

## 8. Definition of Done — additions

Any change under `.github/`:

- [ ] zizmor passes with no new ignore, or each new ignore states its reason
- [ ] Every new `uses:` is a full SHA resolved from the action's repository, with its tag
- [ ] Every new permission is job-level and named in the report
- [ ] No secret referenced outside a step's `env:` in `release.yml`

First workflow in a repository:

- [ ] §2 settings confirmed with `gh api`; anything unavailable recorded in `DECISIONS.md`

Release:

- [ ] Version proposal names the version, the commit SHA, the notes, and any change to
      `release.yml` since the previous release
- [ ] Downloaded files: `SHA256SUMS` attached before publishing; the download check passed
