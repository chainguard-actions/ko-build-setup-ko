<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.9

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **ko-build--setup-ko/v0.9** was hardened automatically. 7 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple ${{ }} expressions are directly interpolated inside the run: shell block in action.yml, violating rule (a). This allows an attacker-controlled value to be injected into the shell before quoting can protect it. Offending lines include:
- Line 22: `if [[ "${{ inputs.use-sudo }}" == "true" ]]` — inputs.use-sudo interpolated directly
- Line 30: `case ${{ inputs.version }} in` — inputs.version interpolated unquoted into a case statement (also rule b)
- Line 35: `curl -L -s -u "username:${{ github.token }}"` — github.token interpolated directly
- Line 38: `tag="${{ inputs.version }}"` — inputs.version interpolated directly
- Line 41: `os=${{ runner.os }}` — runner.os interpolated unquoted (also rule b)
- Line 42: `arch=$(echo "${{ runner.arch }}")` — runner.arch interpolated directly
- Line 57: `echo "${{ github.token }}"` — github.token interpolated directly
- Line 61: `repo=$(echo "${{ github.repository }}")` — github.repository interpolated directly
All of these must be moved to env: variables and the shell expansions must be double-quoted.

Locations:

- `action.yml:22`
- `action.yml:30`
- `action.yml:35`
- `action.yml:38`
- `action.yml:41`
- `action.yml:42`
- `action.yml:57`
- `action.yml:61`

### github-env-injection (severity: high)

Two writes to $GITHUB_ENV in action.yml are missing the required sanitization step (`printf '%s' ... | tr -d '\n\r'`):
1. Line 58: `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV` — KO_DOCKER_REPO is an inherited process env var set by the calling workflow (untrusted). Writing it unsanitized to GITHUB_ENV allows newline injection to set arbitrary environment variables.
2. Line 63: `echo "KO_DOCKER_REPO=ghcr.io/${repo}" >> $GITHUB_ENV` — `repo` is derived from `${{ github.repository }}` (attacker-influenced via fork names). Writing it unsanitized to GITHUB_ENV allows newline injection. Both writes must be preceded by `safe=$(printf '%s' "$VAR" | tr -d '\n\r')` before the echo.

Locations:

- `action.yml:58`
- `action.yml:63`

### unpinned-uses (severity: high)

use-action.yaml references `ko-build/setup-ko@main` in four steps using a mutable branch name instead of a full 40-character commit SHA. This means the action can be silently updated (or compromised) without the workflow noticing, enabling supply-chain attacks. All four occurrences must be pinned to a specific commit SHA (e.g., `ko-build/setup-ko@<40-char-sha> # vX.Y.Z`).

Locations:

- `.github/workflows/use-action.yaml:27`
- `.github/workflows/use-action.yaml:33`
- `.github/workflows/use-action.yaml:39`
- `.github/workflows/use-action.yaml:46`

### missing-permissions (severity: medium)

Neither ci.yaml nor use-action.yaml declares a top-level `permissions:` key, and no job in either file has a job-level `permissions:` block. Without explicit permissions, GitHub Actions grants the default token permissions (which may include write access to contents, packages, etc. depending on repository settings). Both workflow files must declare minimal explicit permissions at the top level or per job.

Locations:

- `.github/workflows/ci.yaml:1`
- `.github/workflows/use-action.yaml:1`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.use-sudo }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:23`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.version }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:31`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.version }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:40`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, static-inline-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed action.yml: moved all ${{ }} expressions (inputs.use-sudo, inputs.version, github.token, runner.os, runner.arch, github.repository) into the step's env: block and referenced them as plain shell variables with proper double-quoting. Sanitized both GITHUB_ENV writes using printf '%s' ... | tr -d '\n\r'. Fixed use-action.yaml: pinned all four ko-build/setup-ko@main references to SHA b8c95b8733ce397fa7482de3d56ab6b45ccb207f and added permissions: {} at top level with packages: write at job level. Fixed ci.yaml: added permissions: {} at top level with packages: write at job level.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable expansions in action.yml. Added double quotes around `${tag}` in the `if [[ ! -z ... ]]` conditional, and wrapped the entire curl URL in double quotes to properly quote `${tag}`, `${tag:1}`, `${os}`, and `${arch}`. This prevents shell metacharacter injection from caller-controlled inputs (version) and runner context values (runner.os, runner.arch).

