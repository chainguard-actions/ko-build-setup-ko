<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.8

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **ko-build--setup-ko/v0.8** was hardened automatically. 6 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

action.yml contains multiple direct ${{ }} expression interpolations inside a run: shell block, violating sub-rule (a). Attacker-controlled and template-substituted values are injected directly into shell commands before the shell ever sees them:
- Line 22: `case ${{ inputs.version }} in` — unquoted, attacker-controlled input used as a case pattern
- Line 29: `tag="${{ inputs.version }}"` — attacker-controlled input assigned to shell variable
- Line 32: `os=${{ runner.os }}` — expression interpolated unquoted into shell
- Line 33: `arch=$(echo "${{ runner.arch }}" | tr ...)` — expression inside command substitution
- Line 43: `echo "${{ github.token }}" | ko login ...` — token expression in run block
- Line 47: `repo=$(echo "${{ github.repository }}" | tr ...)` — expression inside command substitution
All of these should be moved to env: variables and referenced as shell variables (e.g. $INPUTS_VERSION, $RUNNER_OS) instead.

Locations:

- `action.yml:22`
- `action.yml:29`
- `action.yml:32`
- `action.yml:33`
- `action.yml:43`
- `action.yml:47`

### github-env-injection (severity: high)

action.yml writes untrusted values to $GITHUB_ENV without sanitization (no `printf '%s' ... | tr -d '\n\r'` step):
1. The inherited process env var `KO_DOCKER_REPO` (set by the calling workflow and therefore workflow-controlled/untrusted) is written directly: `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV` — a newline in KO_DOCKER_REPO could inject arbitrary environment variables.
2. The value derived from `${{ github.repository }}` (stored in shell variable `repo`) is written directly: `echo "KO_DOCKER_REPO=ghcr.io/${repo}" >> $GITHUB_ENV` — without sanitization, a crafted repository name containing a newline could inject additional entries into GITHUB_ENV.
Both writes require the sanitization pattern: `safe=$(printf '%s' "$VAR" | tr -d '\n\r')` before the write.

Locations:

- `action.yml:40`
- `action.yml:48`

### unpinned-uses (severity: high)

The workflow file use-action.yaml references `ko-build/setup-ko@main` four times. `@main` is a mutable branch ref, not a pinned 40-character commit SHA. This means the action can be silently updated (or compromised) without the workflow noticing, enabling supply-chain attacks. All four occurrences should be pinned to a full SHA digest (e.g. `ko-build/setup-ko@<40-hex-sha> # vX.Y.Z`).

Locations:

- `.github/workflows/use-action.yaml:21`
- `.github/workflows/use-action.yaml:27`
- `.github/workflows/use-action.yaml:34`
- `.github/workflows/use-action.yaml:41`

### missing-permissions (severity: medium)

Neither workflow file defines a `permissions:` block at the top level or at the job level. Without explicit permissions, GitHub Actions grants the default token permissions (which may include write access to contents, packages, etc. depending on repository settings). Both ci.yaml and use-action.yaml should declare minimal required permissions (e.g. `permissions: read-all` or specific scopes) to follow the principle of least privilege.

Locations:

- `.github/workflows/ci.yaml:1`
- `.github/workflows/use-action.yaml:1`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.version }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:22`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.version }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:31`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions, static-inline-injection

**Notes:**

Fixed all findings in action.yml, .github/workflows/use-action.yaml, and .github/workflows/ci.yaml:

1. action.yml - script-injection/static-inline-injection: Moved all ${{ }} expressions (${{ inputs.version }}, ${{ github.token }}, ${{ runner.os }}, ${{ runner.arch }}, ${{ github.repository }}) out of the run: shell block into an env: map. Shell script now uses $INPUTS_VERSION, $GITHUB_TOKEN, $RUNNER_OS_INPUT, $RUNNER_ARCH_INPUT, $GITHUB_REPOSITORY_INPUT.

2. action.yml - github-env-injection: Sanitized both GITHUB_ENV writes with tr -d '\n\r': KO_DOCKER_REPO from environment is sanitized via safe_ko_docker_repo variable; repo derived from github.repository is sanitized inline in the pipeline.

3. use-action.yaml - unpinned-uses: All four ko-build/setup-ko@main references pinned to full SHA b8c95b8733ce397fa7482de3d56ab6b45ccb207f # main.

4. ci.yaml and use-action.yaml - missing-permissions: Added permissions: { contents: read, packages: write } to both workflow files (packages: write needed for ko to push images to ghcr.io).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in action.yml:
1. Quoted `$INPUTS_VERSION` in the `case` statement (line 29): changed `case $INPUTS_VERSION in` to `case "$INPUTS_VERSION" in` to prevent word splitting and glob expansion on the user-controlled version input.
2. Quoted the curl URL (line 51): wrapped the URL containing `${tag}`, `${tag:1}`, `${os}`, and `${arch}` in double quotes to prevent word splitting and glob expansion on those variables.

