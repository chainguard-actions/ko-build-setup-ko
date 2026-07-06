<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.10

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **ko-build--setup-ko/v0.10** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In action.yml, the inherited process environment variable KO_DOCKER_REPO (set by the calling workflow and therefore untrusted) is written directly to $GITHUB_ENV without the required sanitization step (printf '%s' ... | tr -d '\n\r'). An attacker-controlled calling workflow could inject newlines into KO_DOCKER_REPO to poison the environment file. The offending line is: `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV`

Locations:

- `action.yml:56`

### script-injection (severity: high)

Sub-rule (b): In action.yml, the env var KO_VERSION (sourced from inputs.version via the env: block) is expanded unquoted in a case statement: `case ${KO_VERSION} in`. An attacker-controlled version input containing shell metacharacters (e.g. glob patterns, word-splitting characters) could cause unexpected shell behaviour. Additionally, $os and $arch (derived from runner.os and runner.arch via env vars) are used unquoted in if-conditions (`if [[ $os == "macOS" ]]`, `if [[ $arch == "x64" ]]`).

Locations:

- `action.yml:33`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two security issues in action.yml: (1) script-injection: quoted '${KO_VERSION}' in the case statement (was unquoted, allowing glob/word-splitting attacks), and quoted '$os' and '$arch' in if-conditions. (2) github-env-injection: added sanitization of the untrusted KO_DOCKER_REPO environment variable using 'printf "%s" ... | tr -d "\n\r"' before writing it to $GITHUB_ENV, preventing newline injection attacks that could poison the environment file.

