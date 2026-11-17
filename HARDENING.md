<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.8

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **ko-build--setup-ko/v0.8** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): the composite run step puts GitHub expressions straight into the shell script: `case ${{ inputs.version }} in` (line 22), `tag=$(curl -L -s -u "username:${{ github.token }}" ...)` (line 28), `tag="${{ inputs.version }}"` (line 31), `os=${{ runner.os }}` (line 34), `arch=$(echo "${{ runner.arch }}" ...)` (line 35), `echo "${{ github.token }}" | ko login ...` (line 53) and `repo=$(echo "${{ github.repository }}" ...)` (line 57). Rule (b): variables holding workflow-controlled data are expanded without quotes, e.g. `if [[ ! -z ${KO_DOCKER_REPO} ]]` (line 47) and `curl -fsL https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz` (line 44), where tag comes from inputs.version. Fix: pass each value in through an env: block and wrap every expansion in double quotes.

Locations:

- `action.yml:22`
- `action.yml:28`
- `action.yml:31`
- `action.yml:34`
- `action.yml:35`
- `action.yml:44`
- `action.yml:47`
- `action.yml:53`
- `action.yml:57`

### github-env-injection (severity: high)

The composite step writes untrusted values to $GITHUB_ENV without first running them through `printf '%s' ... | tr -d '\n\r'`. Line 50 copies the inherited env var KO_DOCKER_REPO, which the calling workflow controls: `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV`. Line 59 writes a value built from github.repository: `echo "KO_DOCKER_REPO=ghcr.io/${repo}" >> $GITHUB_ENV`. A newline in either value would let it inject extra environment variables.

Locations:

- `action.yml:50`
- `action.yml:59`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.version }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:22`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.version }}" appears directly in run: block of step ""; move to env: map

Locations:

- `action.yml:31`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index df238bc..e1adaf3 100644
--- a/action.yml
+++ b/action.yml
@@ -11,50 +11,74 @@ inputs:
 runs:
   using: "composite"
   steps:
-  - shell: bash
-    run: |
-      set -ex
+    - shell: bash
+      env:
+        INPUTS_VERSION: ${{ inputs.version }}
+        GITHUB_TOKEN: ${{ github.token }}
+        RUNNER_OS_INPUT: ${{ runner.os }}
+        RUNNER_ARCH_INPUT: ${{ runner.arch }}
+        GITHUB_REPOSITORY_INPUT: ${{ github.repository }}
+      run: |
+        set -ex
 
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
+        case "$INPUTS_VERSION" in
+        tip)
+          echo "Installing ko using go install"
+          go install github.com/google/ko@main
+          ;;
+        latest-release)
+          tag=$(curl -L -s -u "username:${GITHUB_TOKEN}" https://api.github.com/repos/ko-build/ko/releases/latest | jq -r '.tag_name')
+          ;;
+        *)
+          tag="$INPUTS_VERSION"
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
-        curl -fsL https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz | tar xzf - -C /usr/local/bin ko
-      fi
+        os=$RUNNER_OS_INPUT
+        arch=$(echo "$RUNNER_ARCH_INPUT" | tr '[:upper:]' '[:lower:]')
+        if [[ $os == "macOS" ]]; then
+          os="Darwin"
+        fi
+        if [[ $arch == "x64" ]]; then
+          arch="x86_64"
+        fi
+        if [[ ! -z ${tag} ]]; then
+          echo "Installing ko @ ${tag} for ${os} ${arch}"
+          curl -fsL "https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz" | tar xzf - -C /usr/local/bin ko
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
+          safe_ko_docker_repo=$(printf '%s' "${KO_DOCKER_REPO}" | tr -d '\n\r')
+          echo "KO_DOCKER_REPO=${safe_ko_docker_repo}" >> "$GITHUB_ENV"
+        else
+          # NB: username doesn't seem to matter.
+          echo "${GITHUB_TOKEN}" | ko login ghcr.io --username "dummy" --password-stdin
 
-        # Set KO_DOCKER_REPO for future steps.
-        # We need to get the repository name in lowercase, otherwise it could fail
-        repo=$(echo "${{ github.repository }}" | tr '[:upper:]' '[:lower:]')
-        echo "KO_DOCKER_REPO=ghcr.io/${repo}"
-        echo "KO_DOCKER_REPO=ghcr.io/${repo}" >> $GITHUB_ENV
-      fi
+          # Set KO_DOCKER_REPO for future steps.
+          # We need to get the repository name in lowercase, otherwise it could fail
+          repo=$(prin
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, static-inline-injection, prior-fix

**Notes:**

Moved every ${{ }} expression in action.yml (inputs.version, github.token, runner.os, runner.arch, github.repository) into the step's env: block. The token goes in as SETUP_KO_TOKEN rather than GITHUB_TOKEN, so tools the step runs don't pick up a variable they didn't get before. Variable expansions are now double-quoted, including the download URL and the KO_DOCKER_REPO check. Newlines are stripped from KO_DOCKER_REPO and from the lowercased repository name before either is written to $GITHUB_ENV, and $GITHUB_ENV is quoted. The install and login logic is otherwise unchanged. This also covers the prior-fix finding, since it re-applies what the previous harden did.

