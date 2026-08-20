<!-- markdownlint-disable -->

# Hardening Report: bufbuild--buf-setup-action/v1.50.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **bufbuild--buf-setup-action/v1.50.0** was hardened automatically. 5 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags instead of pinned 40-character SHA digests, making them vulnerable to supply-chain attacks:
- ci.yaml: actions/checkout@v4, actions/setup-go@v4, actions/setup-node@v4
- create-release-pr.yaml: actions/create-github-app-token@v1, actions/checkout@v3, actions/setup-go@v3
- draft-release.yaml: actions/create-github-app-token@v1, actions/checkout@v3
- add-to-project.yaml: bufbuild/base-workflows/.github/workflows/add-to-project.yaml@main
- emergency-review-bypass.yaml: bufbuild/base-workflows/.github/workflows/emergency-review-bypass.yaml@main
- notify-approval-bypass.yaml: bufbuild/base-workflows/.github/workflows/notify-approval-bypass.yaml@main
- pr-title.yaml: bufbuild/base-workflows/.github/workflows/pr-title.yaml@main

Locations:

- `.github/workflows/ci.yaml:14`
- `.github/workflows/ci.yaml:17`
- `.github/workflows/ci.yaml:20`
- `.github/workflows/create-release-pr.yaml:22`
- `.github/workflows/create-release-pr.yaml:27`
- `.github/workflows/create-release-pr.yaml:30`
- `.github/workflows/draft-release.yaml:20`
- `.github/workflows/draft-release.yaml:27`
- `.github/workflows/add-to-project.yaml:16`
- `.github/workflows/emergency-review-bypass.yaml:10`
- `.github/workflows/notify-approval-bypass.yaml:9`
- `.github/workflows/pr-title.yaml:18`

### script-injection (severity: high)

GitHub Actions expression values are interpolated directly inside run: shell command strings, enabling script injection.

(a) draft-release.yaml: `VERSION="${{ github.event.inputs.version || github.head_ref}}"` — both github.event.inputs.version (user-supplied via workflow_dispatch) and github.head_ref (attacker-controlled via PR) are interpolated directly into the shell command. An attacker could craft a value containing shell metacharacters.

(a) create-release-pr.yaml: `run: make updateversion VERSION=${{env.VERSION}}` — env.VERSION is derived from github.event.inputs.version and is interpolated unquoted directly into the run: command.

(a) create-release-pr.yaml: `git config user.name "${{ github.actor }}"` and `git config user.email "${{ github.actor }}@users.noreply.github.com"` — github.actor is interpolated directly into shell commands.

Locations:

- `.github/workflows/draft-release.yaml:22`
- `.github/workflows/create-release-pr.yaml:35`
- `.github/workflows/create-release-pr.yaml:38`
- `.github/workflows/create-release-pr.yaml:39`

### github-env-injection (severity: high)

draft-release.yaml writes a value derived from attacker-controlled input to $GITHUB_ENV without sanitization. The step 'Set VERSION variable' sets VERSION from `${{ github.event.inputs.version || github.head_ref }}` (both are untrusted: one is user-supplied via workflow_dispatch, the other is the PR head branch name which an attacker controls). It then writes `echo "VERSION=${VERSION##*/v}" >> $GITHUB_ENV` without applying `printf '%s' ... | tr -d '\n\r'` before the write. A newline-containing value could inject arbitrary environment variables into subsequent steps.

Locations:

- `.github/workflows/draft-release.yaml:22`

### broad-permissions (severity: medium)

ci.yaml sets `permissions: read-all` at the top level. The `read-all` value grants overly broad read access across all scopes and must be replaced with specific minimal permissions (e.g., `contents: read`).

Locations:

- `.github/workflows/ci.yaml:10`

### missing-permissions (severity: medium)

add-to-project.yaml has no top-level `permissions:` key and its only job (call-workflow-add-to-project) also has no job-level `permissions:` key. This means the workflow runs with the default token permissions, which may be broader than necessary. The workflow uses `pull_request_target` which is a high-privilege trigger, making this especially risky.

Locations:

- `.github/workflows/add-to-project.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, broad-permissions, missing-permissions

**Notes:**

Fixed all 5 findings across 7 workflow files:

1. unpinned-uses: Pinned all action references to full SHA digests:
   - actions/checkout@v4 → @11d5960a326750d5838078e36cf38b85af677262
   - actions/setup-go@v4 → @7b8cf10d4e4a01d4992d18a89f4d7dc5a3e6d6f4
   - actions/setup-node@v4 → @49933ea5288caeca8642d1e84afbd3f7d6820020
   - actions/create-github-app-token@v1 → @d72941d797fd3113feb6b93fd0dec494b13a2547
   - actions/checkout@v3 → @a37ce9120846195fa4ece8f58b268e6043cb2f26
   - actions/setup-go@v3 → @be3c94b385c4f180051c996d336f57a34c397495
   - bufbuild/base-workflows@main → @7c0b54b718244c78f00037343ff7cb33ef7caca9 (used in add-to-project.yaml, emergency-review-bypass.yaml, notify-approval-bypass.yaml, pr-title.yaml)

2. script-injection: Moved all ${{ }} expressions out of run: shell strings into env: blocks:
   - draft-release.yaml: github.event.inputs.version → INPUT_VERSION, github.head_ref → HEAD_REF
   - create-release-pr.yaml: github.actor → ACTOR; VERSION env var already set at workflow level so used $VERSION directly; fixed unquoted VERSION in make command

3. github-env-injection: draft-release.yaml's Set VERSION step now sanitizes with `printf '%s' "$TRIMMED" | tr -d '\n\r'` before writing to $GITHUB_ENV, and quotes the $GITHUB_ENV path

4. broad-permissions: ci.yaml changed from `permissions: read-all` to `permissions:\n  contents: read`

5. missing-permissions: add-to-project.yaml now has `permissions: {}` at top level

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two unquoted shell variable expansions of `${BRANCH}` in `.github/workflows/create-release-pr.yaml`. The `BRANCH` variable is derived from `VERSION` which comes from the workflow_dispatch input `github.event.inputs.version`. Both `git switch -C ${BRANCH}` (line 48) and `git push --set-upstream origin --force ${BRANCH}` (line 51) were updated to use double-quoted expansions: `"${BRANCH}"`. This prevents shell metacharacters in the version input from causing command injection.

