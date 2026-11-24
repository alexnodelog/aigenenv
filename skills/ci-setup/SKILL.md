---
name: ci-setup
description: Set up and change GitHub Actions for any archetype, and prepare a release. Use when adding the first file under .github/workflows/; when creating or changing ci.yml, release.yml, .github/dependabot.yml or .gitleaks.toml; when adding or updating a uses: line or an action SHA; when zizmor or Gitleaks fails; when checking repository settings (SHA pinning, workflow permissions, immutable releases, the release environment); when a job fails with "Resource not accessible by integration"; or when proposing a version, tagging, or resuming a failed release.
---

# ci-setup

Procedures for `rules/module-deployment.md` §2–§6. The rules stay there; this file says
how. The archetype's own release skill adds its build, check and publish steps to the
skeletons below. Pin every version below exactly in the project; this file names none.

## 1. Check the repository settings (rule §2)

Before the first workflow lands, read each setting with `gh api` (read-only), from inside
the repository. Report every one that is not as expected. The human changes settings; the
agent never does.

```bash
gh api repos/{owner}/{repo}/actions/permissions --jq '.sha_pinning_required'
# expected: true

gh api repos/{owner}/{repo}/actions/permissions/workflow \
  --jq '[.default_workflow_permissions, .can_approve_pull_request_reviews]'
# expected: ["read", false]

gh api repos/{owner}/{repo}/immutable-releases --jq '.enabled'
# archetypes that publish GitHub Release files; expected: true (404 means not enabled)

gh api repos/{owner}/{repo} --jq '[.visibility, .permissions.push]'
# visibility decides the next two checks; .permissions.push is the agent credential's
```

- **Public repository:** check the `release` environment.

  ```bash
  gh api repos/{owner}/{repo}/environments/release \
    --jq '[.deployment_branch_policy, [.protection_rules[] | .type]]'
  # expected: {"protected_branches":false,"custom_branch_policies":true} and "required_reviewers" listed
  gh api repos/{owner}/{repo}/environments/release/deployment-branch-policies \
    --jq '[.branch_policies[] | [.name, .type]]'
  # expected: [["v*","tag"]] and nothing else
  ```

  Also confirm in the web UI that the required reviewer is the owner, and that the
  repository has no repository-level secrets (`gh secret list` must print nothing).
- **Private repository on GitHub Free:** `.permissions.push` MUST be `false` for the
  agent's credential. Record the arrangement once in `DECISIONS.md`.
- A setting the plan does not offer: record it in `DECISIONS.md` as unavailable.
- If `gh` is not signed in, or a call returns 403 or 404 for lack of admin access, stop
  and ask the human to run the commands. Do not guess the setting from the UI.

## 2. Pin an action (rule §4.4)

1. Find the action's newest release tag on its own repository page, and use that
   repository's URL, never a fork's.
2. Resolve the tag to a commit:

   ```bash
   git ls-remote https://github.com/<owner>/<action> "refs/tags/<tag>" "refs/tags/<tag>^{}" | awk '{print $1}' | tail -1
   ```

   - An annotated tag prints two lines; the `^{}` line is the commit, and it comes last.
     The first line of an annotated tag is the tag object, which is not a commit.
   - An empty result means the tag does not exist there. Stop and report.
3. Write `uses: <owner>/<action>@<40-char sha> # <tag>`.

Never type a SHA from memory or copy one from another project.

## 3. `ci.yml` (rule §3, §4, §5.3)

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

permissions: {}

jobs:
  scan:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<sha> # <tag>
        with:
          persist-credentials: false
          fetch-depth: 0
      - name: zizmor
        run: |
          pipx install "uv==<exact>"
          uvx "zizmor==<exact>" --offline .github/
      - name: Gitleaks
        env:
          GITLEAKS_VERSION: "<exact>"
          GITLEAKS_SHA256: "<sha256 of gitleaks_<version>_linux_x64.tar.gz, as in DECISIONS.md>"
        run: |
          cd "$RUNNER_TEMP"
          curl -fsSLo gitleaks.tgz "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz"
          echo "${GITLEAKS_SHA256}  gitleaks.tgz" | sha256sum -c -
          tar xzf gitleaks.tgz gitleaks
          cd "$GITHUB_WORKSPACE"
          "$RUNNER_TEMP/gitleaks" git . --redact --no-banner

  checks:
    needs: scan
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<sha> # <tag>
        with:
          persist-credentials: false
          fetch-depth: 0
      # the archetype's "Push / PR" list
```

- `pipx` is preinstalled on GitHub's Ubuntu images, so no setup action is needed for uv.
- `gitleaks git .` scans every branch and tag in the checkout. It exits 1 on a finding.
- Gitleaks' SHA-256 comes from the release's `gitleaks_<version>_checksums.txt`. Copy the
  `linux_x64` line into `DECISIONS.md` and into the workflow.
- An event field (PR title, branch name) goes through `env:` and is used quoted:

  ```yaml
  - env:
      TITLE: ${{ github.event.pull_request.title }}
    run: echo "$TITLE"
  ```

- `ci.yml` MAY cache; `release.yml` never does.

## 4. `release.yml` (rule §3, §4, §5)

```yaml
name: release
on:
  push:
    tags: ["v*"]

permissions: {}

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  gate:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<sha> # <tag>
        with:
          persist-credentials: false
          fetch-depth: 0
      - name: Tag commit is on main
        run: git merge-base --is-ancestor "$GITHUB_SHA" origin/main
      - name: zizmor
        run: |
          pipx install "uv==<exact>"
          uvx "zizmor==<exact>" --offline .github/
      - name: Gitleaks over history reachable from the tag
        env:
          GITLEAKS_VERSION: "<exact>"
          GITLEAKS_SHA256: "<sha256>"
        run: |
          cd "$RUNNER_TEMP"
          curl -fsSLo gitleaks.tgz "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz"
          echo "${GITLEAKS_SHA256}  gitleaks.tgz" | sha256sum -c -
          tar xzf gitleaks.tgz gitleaks
          cd "$GITHUB_WORKSPACE"
          "$RUNNER_TEMP/gitleaks" git . --redact --no-banner --log-opts="$GITHUB_SHA"

  # build jobs: needs: gate, permissions: contents: read, no secrets.
  # Each ends with actions/upload-artifact of its files.

  publish:
    needs: [gate]          # plus every build and check job
    runs-on: ubuntu-24.04
    # environment: release  # public repository with a release secret
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@<sha> # <tag>
        with:
          path: out
          merge-multiple: true
      # the publish steps of §5, or the archetype's deploy step
```

- `--log-opts="$GITHUB_SHA"` limits Gitleaks to the history reachable from the tag's commit.
- The publish job runs no project code and installs nothing. It does not check out the
  repository unless a step needs a file from it. `gh` then needs
  `GH_REPO: ${{ github.repository }}`.
- A secret goes only in the `env:` of the one step that uses it:

  ```yaml
  - name: Deploy
    env:
      HOOK_URL: ${{ secrets.DEPLOY_HOOK_URL }}
    run: curl -fsS -X POST "$HOOK_URL"
  ```

- `runs-on` uses an OS version label, never `*-latest`.

## 5. Publishing downloaded files (rule §5.4)

In the publish job, after the artifacts are downloaded into `out/`:

```yaml
- name: Draft, attach, publish, verify
  env:
    GH_TOKEN: ${{ github.token }}
    GH_REPO: ${{ github.repository }}
    TAG: ${{ github.ref_name }}
  run: |
    cd out
    sha256sum -- * > ../SHA256SUMS
    mv ../SHA256SUMS .
    gh release create "$TAG" --draft --verify-tag --title "$TAG" --notes-file ../notes.md
    gh release upload "$TAG" -- *
    gh release edit "$TAG" --draft=false
    mkdir ../check && cd ../check
    gh release download "$TAG"
    sha256sum -c SHA256SUMS
```

- `notes.md` comes from the approved version proposal (§8). Write it into an artifact of
  the gate job, or check out the tag to read it.
- If a release for the tag already exists as a draft (a resumed run, §9), skip
  `gh release create` and upload only the files it lacks. If it is already published,
  stop: nothing is attached or replaced after publishing.

## 6. `.github/dependabot.yml` (rule §1, §4.4)

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    cooldown:
      default-days: 7
  # one entry per project ecosystem, same schedule and cooldown:
  # - package-ecosystem: uv      (uv.lock)
  # - package-ecosystem: npm     (package.json with pnpm-lock.yaml)
```

- `cooldown` delays version updates only; security updates are not delayed.
- Dependabot PRs are reviewed and merged by the human.

## 7. `.gitleaks.toml` (rule §5.3)

Only when a test fixture holds a made-up value the default rules flag:

```toml
[extend]
useDefault = true

[[allowlists]]
description = "<why: made-up value for <test name>>"
paths = ['''^tests/fixtures/<one file>$''']
```

One file per entry, anchored with `^` and `$`. A directory or pattern entry is PROHIBITED.

## 8. Before committing a workflow change (rule §6)

1. `uvx "zizmor==<exact>" --offline .github/` exits 0. An ignore is
   `# zizmor: ignore[<audit>] <reason>` on the offending line.
2. Every new `uses:` was resolved by §2.
3. Every new permission is job-level, and is named in the report.

## 9. Version proposal, tag and resume (rule §4.1, §5.4)

Propose:

```bash
prev=$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null || echo "")
range=${prev:+$prev..}HEAD
git log "$range" --format='%H %s'
git log "$range" --format='%B' | grep -E '^BREAKING CHANGE:' || true
```

- No previous tag: `v0.1.0`.
- A subject with `!` before the colon (`feat!:`, `refactor(x)!:`) or a `BREAKING CHANGE:`
  footer raises major. Before 1.0.0 it raises minor instead.
- Otherwise any `feat` raises minor, and anything else raises patch.
- Give the human the version, `git rev-parse HEAD`, the notes grouped as Breaking,
  Features, Fixes, Other, and every change to `release.yml` since `prev`
  (`git diff "$prev" -- .github/workflows/release.yml`).

After approval of that version and that SHA:

```bash
git tag -a "v<version>" "<approved sha>" -m "v<version>"
test "$(git rev-parse "v<version>^{commit}")" = "<approved sha>"
git push origin "v<version>"
```

In a private repository on GitHub Free the human runs the push.

One release at a time: do not propose the next tag until this run has succeeded, or the
human has abandoned it (draft deleted, reason in `known-issues.md`).

Resume a failed run, with the human's approval:

```bash
gh run list --workflow release.yml --limit 5
gh release view "v<version>" --json isDraft --jq .isDraft   # must print true, or fail with "release not found"
gh run rerun <run-id> --failed
```

Never delete or move the tag, and never push a new tag for the same version.
