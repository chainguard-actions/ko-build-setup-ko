<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.9

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **ko-build--setup-ko/v0.9** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

(a) The composite step in action.yml puts GitHub expressions straight into the run: script: `if [[ "${{ inputs.use-sudo }}" == "true" ]]`, `case ${{ inputs.version }} in`, `tag="${{ inputs.version }}"`, `curl ... -u "username:${{ github.token }}"`, `os=${{ runner.os }}`, `arch=$(echo "${{ runner.arch }}" ...)`, `echo "${{ github.token }}" | ko login ...` and `repo=$(echo "${{ github.repository }}" ...)`. (b) Workflow-controlled values are also expanded without quotes: `${tag}` in `curl -fsL https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz`, `[[ ! -z ${tag} ]]`, and the inherited `${KO_DOCKER_REPO}` in `[[ ! -z ${KO_DOCKER_REPO} ]]`.

Locations:

- `action.yml:23`
- `action.yml:31`
- `action.yml:37`
- `action.yml:40`
- `action.yml:43`
- `action.yml:44`
- `action.yml:53`
- `action.yml:56`
- `action.yml:62`
- `action.yml:66`

### github-env-injection (severity: high)

The step writes workflow-controlled values to $GITHUB_ENV without first running `printf '%s' ... | tr -d '\n\r'`. The calling workflow sets KO_DOCKER_REPO, and the step forwards it unchanged with `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV`. Separately, `repo` comes from `${{ github.repository }}` and is written with `echo "KO_DOCKER_REPO=ghcr.io/${repo}" >> $GITHUB_ENV`, also without stripping newlines.

Locations:

- `action.yml:59`
- `action.yml:68`

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

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 2f0ca74..df90be7 100644
--- a/action.yml
+++ b/action.yml
@@ -15,55 +15,81 @@ inputs:
 runs:
   using: "composite"
   steps:
-  - shell: bash
-    run: |
-      set -ex
+    - shell: bash
+      env:
+        INPUT_USE_SUDO: ${{ inputs.use-sudo }}
+        INPUT_VERSION: ${{ inputs.version }}
+        GITHUB_TOKEN: ${{ github.token }}
+        RUNNER_OS_INPUT: ${{ runner.os }}
+        RUNNER_ARCH_INPUT: ${{ runner.arch }}
+        GITHUB_REPOSITORY_INPUT: ${{ github.repository }}
+      run: |
+        set -ex
 
-      SUDO=
-      if [[ "${{ inputs.use-sudo }}" == "true" ]] && command -v sudo >/dev/null; then
-        SUDO=sudo
-      fi
+        SUDO=
+        if [[ "$INPUT_USE_SUDO" == "true" ]] && command -v sudo >/dev/null; then
+          SUDO=sudo
+        fi
 
-      # Install ko:
-      # - if version is "tip", install from tip of main.
-      # - if version is "latest-release", look up latest release.
-      # - otherwise, install the specified version.
-      case ${{ inputs.version }} in
-      tip)
-        echo "Installing ko using go install"
-        go install github.com/google/ko@main
-        ;;
-      latest-release)
-        tag=$(curl -L -s -u "username:${{ github.token }}" https://api.github.com/repos/ko-build/ko/releases/latest | jq -r '.tag_name')
-        ;;
-      *)
-        tag="${{ inputs.version }}"
-      esac
+        # Install ko:
+        # - if version is "tip", install from tip of main.
+        # - if version is "latest-release", look up latest release.
+        # - otherwise, install the specified version.
+        case "$INPUT_VERSION" in
+        tip)
+          echo "Installing ko using go install"
+          go install github.com/google/ko@main
+          ;;
+        latest-release)
+          tag=$(curl -L -s -u "username:$GITHUB_TOKEN" https://api.github.com/repos/ko-build/ko/releases/latest | jq -r '.tag_name')
+          ;;
+        *)
+          tag="$INPUT_VERSION"
+        esac
 
-      os=${{ runner.os }}
-      arch=$(echo "${{ runner.arch }}" | tr '[:upper:]' '[:lower:]')
-      if [[ $os == "macOS" ]]; then
-        os="Darwin"
-      fi
-      if [[ $arch == "x64" ]]; then
-        arch="x86_64"
-      fi
-      if [[ ! -z ${tag} ]]; then
-        echo "Installing ko @ ${tag} for ${os} ${arch}"
-        curl -fsL https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz | $SUDO tar xzf - -C /usr/local/bin ko
-      fi
+        os="$RUNNER_OS_INPUT"
+        arch=$(echo "$RUNNER_ARCH_INPUT" | tr '[:upper:]' '[:lower:]')
+        if [[ "$os" == "macOS" ]]; then
+          os="Darwin"
+        fi
+        if [[ "$arch" == "x64" ]]; then
+          arch="x86_64"
+        fi
+        if [[ ! -z "${tag}" ]]; then
+          echo "Installing ko @ ${tag} for ${os} ${arch}"
+          curl -fsL "https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz" | $SUDO tar xzf - -C /usr/local/bin ko
+        fi
 
-      if [[ ! -z ${KO_DOCKER_REPO} ]]; then
-        echo "KO_DOCKER_REPO is already set"
-        echo "Skipping login to ghcr.io and passing KO_DOCKER_REPO=${KO_DOCKER_REPO} on to future steps"
-        echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV
-      else
-        # NB: username doesn't seem to matter.
-        echo "${{ github.token }}" | ko login ghcr.io --username "dummy" --password-stdin
+        if [[ ! -z ${KO_DOCKER_REPO} ]]; then
+          echo "KO_DOCKER_REPO is already set"
+          echo "Skipping login to ghcr.io and passing KO_DOCKER_REPO=${KO_DOCKER_REPO} on to future steps"
+          safe_ko_docker_repo=$(printf '%s' "$KO_DOCKER_REPO" | tr -d '\n\r')
+          echo "KO_DOCKER_REPO=${safe_ko_docker_repo}" >> "$GITHUB_ENV"
+        else
+          # NB: username doesn't seem to matter.
+          echo "$GITHUB_TOKEN" | ko login ghcr.io --username "dummy" --password-stdin
 
-        # Set KO_DOCKER_REPO for future steps.
-        # We need to get the repository name in lowercase, otherwise it could fail
-        repo=$(echo "${{ github.re
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, static-inline-injection, prior-fix

**Notes:**

I rewrote the one composite step in action.yml. Every ${{ }} expression is now set in the step's env: block: inputs.use-sudo, inputs.version, github.token, runner.os, runner.arch and github.repository. The script reads them as double-quoted shell variables. ${tag}, the download URL and ${KO_DOCKER_REPO} are now quoted where they expand. Both writes to $GITHUB_ENV strip newlines and carriage returns first. For KO_DOCKER_REPO that uses printf | tr -d '\n\r'. For the repository name the newline strip is added to the existing lowercase pipeline, which is safe because the step runs under bash with pipefail. The token is still piped to ko login on stdin, as before. This re-applies the security changes from the previous harden. Nothing was run or tested in this pass.

