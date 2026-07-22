<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.10

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **ko-build--setup-ko/v0.10** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (b): The env var KO_VERSION (sourced from `${{ inputs.version }}`, a user-controlled input) is expanded unquoted in a `case` statement: `case ${KO_VERSION} in`. An unquoted shell expansion allows an attacker to inject shell metacharacters (`;`, `|`, `&`, etc.) via the `version` input. It should be `case "${KO_VERSION}" in`.

Locations:

- `action.yml:36`

### github-env-injection (severity: high)

The inherited process env var `KO_DOCKER_REPO` (set by the calling workflow — untrusted) is written directly to `$GITHUB_ENV` without sanitization: `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV`. A newline in the value would allow an attacker to inject arbitrary environment variables into subsequent steps. The required sanitization step (`printf '%s' "$KO_DOCKER_REPO" | tr -d '\n\r'`) is missing before the write.

Locations:

- `action.yml:58`

### unpinned-uses (severity: high)

Four `uses:` references in use-action.yaml point to `ko-build/setup-ko@main`, which is a mutable branch ref rather than an immutable 40-character commit SHA. This exposes the workflow to supply-chain attacks if the `main` branch is compromised. All four occurrences (lines 30, 35, 42, 51) must be pinned to a full SHA (e.g. `ko-build/setup-ko@<40-char-sha> # vX.Y.Z`).

Locations:

- `.github/workflows/use-action.yaml:30`
- `.github/workflows/use-action.yaml:35`
- `.github/workflows/use-action.yaml:42`
- `.github/workflows/use-action.yaml:51`

### missing-permissions (severity: medium)

The workflow file use-action.yaml has no top-level `permissions:` key and the single job `use-action` also has no `permissions:` key. Without explicit permissions, the workflow inherits the repository default (typically `contents: write` for push-triggered workflows), granting broader access than necessary. A minimal `permissions:` block (e.g. `contents: read`) should be added.

Locations:

- `.github/workflows/use-action.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings: (1) Quoted ${KO_VERSION} in the case statement in action.yml to prevent shell metacharacter injection; (2) Added sanitization of KO_DOCKER_REPO via `printf '%s' "${KO_DOCKER_REPO}" | tr -d '\n\r'` before writing to $GITHUB_ENV to prevent newline injection; (3) Pinned all four `ko-build/setup-ko@main` references in use-action.yaml to the full commit SHA `b8c95b8733ce397fa7482de3d56ab6b45ccb207f # main`; (4) Added `permissions: contents: read` top-level block to use-action.yaml.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed four unquoted shell variable expansions in action.yml that could allow shell metacharacter injection: (1) `$os` in `[[ $os == "macOS" ]]` → `[[ "$os" == "macOS" ]]`; (2) `$arch` in `[[ $arch == "x64" ]]` → `[[ "$arch" == "x64" ]]`; (3) `${tag}` in `[[ ! -z ${tag} ]]` → `[[ -n "${tag}" ]]`; (4) `${KO_DOCKER_REPO}` in `[[ ! -z ${KO_DOCKER_REPO} ]]` → `[[ -n "${KO_DOCKER_REPO}" ]]`. Also replaced the deprecated `! -z` form with the idiomatic `-n` test.

