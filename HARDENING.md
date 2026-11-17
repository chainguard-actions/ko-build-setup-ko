<!-- markdownlint-disable -->

# Hardening Report: ko-build--setup-ko/v0.10

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **ko-build--setup-ko/v0.10** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

The composite step copies the inherited, workflow-controlled env var KO_DOCKER_REPO into $GITHUB_ENV without stripping newlines first: `echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV`. If the value contains a newline, it can add any other environment variable to later steps. Fix: before the write, run safe=$(printf '%s' "$KO_DOCKER_REPO" | tr -d '\n\r'), then write "KO_DOCKER_REPO=$safe" >> "$GITHUB_ENV".

Locations:

- `action.yml:66`

### script-injection (severity: high)

Rule (b): workflow-controlled env vars are expanded without double quotes in the run block. Examples: `case ${KO_VERSION} in` (line 38; KO_VERSION comes from inputs.version), `if [[ ! -z ${tag} ]]; then` (line 58; tag comes from inputs.version), and `if [[ ! -z ${KO_DOCKER_REPO} ]]; then` (line 63; KO_DOCKER_REPO is inherited from the calling workflow). Quote each one: "${KO_VERSION}", "${tag}", "${KO_DOCKER_REPO}".

Locations:

- `action.yml:38`
- `action.yml:58`
- `action.yml:63`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 347e265..a93a2c6 100644
--- a/action.yml
+++ b/action.yml
@@ -15,63 +15,81 @@ inputs:
 runs:
   using: "composite"
   steps:
-  - shell: bash
-    env:
-      KO_VERSION: ${{ inputs.version }}
-      USE_SUDO: ${{ inputs.use-sudo }}
-      GH_TOKEN: ${{ github.token }}
-      RUNNER_OS: ${{ runner.os }}
-      RUNNER_ARCH: ${{ runner.arch }}
-      GITHUB_REPOSITORY: ${{ github.repository }}
-    run: |
-      set -ex
+    - shell: bash
+      env:
+        KO_VERSION: ${{ inputs.version }}
+        USE_SUDO: ${{ inputs.use-sudo }}
+        GH_TOKEN: ${{ github.token }}
+        RUNNER_OS: ${{ runner.os }}
+        RUNNER_ARCH: ${{ runner.arch }}
+        GITHUB_REPOSITORY: ${{ github.repository }}
+      run: |
+        set -ex
 
-      SUDO=
-      if [[ "${USE_SUDO}" == "true" ]] && command -v sudo >/dev/null; then
-        SUDO=sudo
-      fi
+        SUDO=
+        if [[ "${USE_SUDO}" == "true" ]] && command -v sudo >/dev/null; then
+          SUDO=sudo
+        fi
 
-      # Install ko:
-      # - if version is "tip", install from tip of main.
-      # - if version is "latest-release", look up latest release.
-      # - otherwise, install the specified version.
-      case ${KO_VERSION} in
-      tip)
-        echo "Installing ko using go install"
-        go install github.com/google/ko@main
-        ;;
-      latest-release)
-        tag=$(curl -L -s -u "username:${GH_TOKEN}" https://api.github.com/repos/ko-build/ko/releases/latest | jq -r '.tag_name')
-        ;;
-      *)
-        tag="${KO_VERSION}"
-      esac
+        # Install ko:
+        # - if version is "tip", install from tip of main.
+        # - if version is "latest-release", look up latest release.
+        # - otherwise, install the specified version.
+        case "${KO_VERSION}" in
+        tip)
+          echo "Installing ko using go install"
+          go install github.com/google/ko@main
+          ;;
+        latest-release)
+          tag=$(curl -L -s -u "username:${GH_TOKEN}" https://api.github.com/repos/ko-build/ko/releases/latest | jq -r '.tag_name')
+          ;;
+        *)
+          tag="${KO_VERSION}"
+        esac
 
-      os="${RUNNER_OS}"
-      arch=$(echo "${RUNNER_ARCH}" | tr '[:upper:]' '[:lower:]')
-      if [[ $os == "macOS" ]]; then
-        os="Darwin"
-      fi
-      if [[ $arch == "x64" ]]; then
-        arch="x86_64"
-      fi
-      if [[ ! -z ${tag} ]]; then
-        echo "Installing ko @ ${tag} for ${os} ${arch}"
-        curl -fsL "https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz" | $SUDO tar xzf - -C /usr/local/bin ko
-      fi
+        os="${RUNNER_OS}"
+        arch=$(echo "${RUNNER_ARCH}" | tr '[:upper:]' '[:lower:]')
+        if [[ "$os" == "macOS" ]]; then
+          os="Darwin"
+        fi
+        if [[ "$arch" == "x64" ]]; then
+          arch="x86_64"
+        fi
+        if [[ -n "${tag}" ]]; then
+          echo "Installing ko @ ${tag} for ${os} ${arch}"
+          curl -fsL "https://github.com/ko-build/ko/releases/download/${tag}/ko_${tag:1}_${os}_${arch}.tar.gz" | $SUDO tar xzf - -C /usr/local/bin ko
+        fi
 
-      if [[ ! -z ${KO_DOCKER_REPO} ]]; then
-        echo "KO_DOCKER_REPO is already set"
-        echo "Skipping login to ghcr.io and passing KO_DOCKER_REPO=${KO_DOCKER_REPO} on to future steps"
-        echo "KO_DOCKER_REPO=${KO_DOCKER_REPO}" >> $GITHUB_ENV
-      else
-        # NB: username doesn't seem to matter.
-        echo "${GH_TOKEN}" | ko login ghcr.io --username "dummy" --password-stdin
+        if [[ -n "${KO_DOCKER_REPO}" ]]; then
+          echo "KO_DOCKER_REPO is already set"
+          echo "Skipping login to ghcr.io and passing KO_DOCKER_REPO=${KO_DOCKER_REPO} on to future steps"
+          safe_ko_docker_repo=$(printf '%s' "${KO_DOCKER_REPO}" | tr -d '\n\r')
+          echo "KO_DOCKER_REPO=${safe_ko_docker_repo}" >> "${GITHUB_ENV}"
+        else
+          # NB: username doesn't seem to matter.
+          echo "${GH_TOKEN}" | ko login ghcr.io --username "dummy" --password-stdin
 
-       
... (truncated)
```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, prior-fix

**Notes:**

I fixed all three findings in action.yml. Nothing was run or tested.

- **github-env-injection:** when `KO_DOCKER_REPO` is already set, the step now removes newline and carriage-return characters from it (`printf '%s' | tr -d '\n\r'`) before writing it to `$GITHUB_ENV`. The `$GITHUB_ENV` path is now quoted too.
- **script-injection:** I added double quotes to the variables the step left unquoted: `KO_VERSION` in the `case` statement, `os` and `arch` in their comparisons, and `tag` and `KO_DOCKER_REPO` in their emptiness checks. I also changed `! -z` to the equivalent `-n` in those two checks.
- **prior-fix:** the earlier patch made the same changes as the two fixes above, plus a re-indent of the step, which I didn't redo. Its diff was cut off partway through. The part I couldn't see covers the other branch, which builds `KO_DOCKER_REPO` from the repository name. That branch already strips newlines and quotes its variables in the current file, so I left it alone.

