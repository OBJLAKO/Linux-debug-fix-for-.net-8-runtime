---
name: "Mobile Platform Failure Scanner"
description: "Daily scan of the runtime-extra-platforms pipeline for Apple mobile and Android failures. Investigates and proposes fixes."

permissions:
  contents: read
  issues: read
  pull-requests: read

on:
  schedule: daily
  workflow_dispatch:
  roles: [admin, maintainer, write]

# ###############################################################
# Override the COPILOT_GITHUB_TOKEN secret usage for the workflow
# with a randomly-selected token from a pool of secrets.
#
# As soon as organization-level billing is offered for Agentic
# Workflows, this stop-gap approach will be removed.
#
# See: /.github/actions/select-copilot-pat/README.md
# ###############################################################

  # Add the pre-activation step of selecting a random PAT from the supplied secrets
  steps:
    - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      name: Checkout the select-copilot-pat action folder
      with:
        persist-credentials: false
        sparse-checkout: .github/actions/select-copilot-pat
        sparse-checkout-cone-mode: true
        fetch-depth: 1

    - id: select-copilot-pat
      name: Select Copilot token from pool
      uses: ./.github/actions/select-copilot-pat
      env:
        SECRET_0: ${{ secrets.COPILOT_PAT_0 }}
        SECRET_1: ${{ secrets.COPILOT_PAT_1 }}
        SECRET_2: ${{ secrets.COPILOT_PAT_2 }}
        SECRET_3: ${{ secrets.COPILOT_PAT_3 }}
        SECRET_4: ${{ secrets.COPILOT_PAT_4 }}
        SECRET_5: ${{ secrets.COPILOT_PAT_5 }}
        SECRET_6: ${{ secrets.COPILOT_PAT_6 }}
        SECRET_7: ${{ secrets.COPILOT_PAT_7 }}
        SECRET_8: ${{ secrets.COPILOT_PAT_8 }}
        SECRET_9: ${{ secrets.COPILOT_PAT_9 }}

# Add the pre-activation output of the randomly selected PAT
jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}

# Override the COPILOT_GITHUB_TOKEN expression used in the activation job
# Consume the PAT number from the pre-activation step and select the corresponding secret
engine:
  id: copilot
  model: claude-sonnet-4.5
  env:
    # We cannot use line breaks in this expression as it leads to a syntax error in the compiled workflow
    # If none of the `COPILOT_PAT_#` secrets were selected, then the default COPILOT_GITHUB_TOKEN is used
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_PAT_0, needs.pre_activation.outputs.copilot_pat_number == '1', secrets.COPILOT_PAT_1, needs.pre_activation.outputs.copilot_pat_number == '2', secrets.COPILOT_PAT_2, needs.pre_activation.outputs.copilot_pat_number == '3', secrets.COPILOT_PAT_3, needs.pre_activation.outputs.copilot_pat_number == '4', secrets.COPILOT_PAT_4, needs.pre_activation.outputs.copilot_pat_number == '5', secrets.COPILOT_PAT_5, needs.pre_activation.outputs.copilot_pat_number == '6', secrets.COPILOT_PAT_6, needs.pre_activation.outputs.copilot_pat_number == '7', secrets.COPILOT_PAT_7, needs.pre_activation.outputs.copilot_pat_number == '8', secrets.COPILOT_PAT_8, needs.pre_activation.outputs.copilot_pat_number == '9', secrets.COPILOT_PAT_9, secrets.COPILOT_GITHUB_TOKEN) }}

concurrency:
  group: "mobile-scan"
  cancel-in-progress: true

tools:
  github:
    toolsets: [pull_requests, repos, issues, search]
  edit:
  bash: ["dotnet", "git", "find", "ls", "cat", "grep", "head", "tail", "wc", "curl", "jq", "pwsh", "tee", "sed", "awk", "tr", "cut", "sort", "uniq", "xargs", "echo", "date", "mkdir", "test", "env", "basename", "dirname", "bash", "sh", "chmod"]

checkout:
  fetch-depth: 50

safe-outputs:
  create-pull-request:
    title-prefix: "[mobile] "
    draft: true
    max: 2
    protected-files: fallback-to-issue
    labels: [agentic-workflows]
  create-issue:
    max: 2
    labels: [agentic-workflows, untriaged]
  add-comment:
    max: 5
    target: "*"

timeout-minutes: 60

network:
  allowed:
    - defaults
    - github
    - dev.azure.com
    - helix.dot.net
    - "*.blob.core.windows.net"
---

# Mobile Platform Failure Scanner

Scan two AzDO pipelines in `dnceng-public/public` on `main` for Apple mobile and Android failures. Triage, dedup, and act.

- **`runtime`** (def **129**) — main rolling CI. Public mirror of internal `dnceng/internal` def 1104.
- **`runtime-extra-platforms`** (def **154**) — full mobile matrix, daily.

Both are anon-accessible. Sanitize log excerpts (paths/tokens/auth) before posting.

## Conventions

- Every shell call is a fresh subshell. Persist state under `/tmp/gh-aw/agent/`.
- `$(...)`, `${var@P}`, `-o`, `>` are blocked. Use `| tee` and script files (`bash script.sh`).
- URL-encode OData `$` params (`%24top`).

## Step 1: Load skills

```bash
mkdir -p /tmp/gh-aw/agent
curl -sL "https://raw.githubusercontent.com/dotnet/arcade-skills/f866c30a5b58e76492c90fd089082eb5f7e81a87/plugins/dotnet-dnceng/skills/helix-investigation/SKILL.md" \
  | tee /tmp/gh-aw/agent/helix-investigation-skill.md > /dev/null
```

Also read `.github/skills/mobile-platforms/SKILL.md`.

## Step 2: Fetch latest completed builds

```bash
for DEF in 129 154; do
  curl -sL "https://dev.azure.com/dnceng-public/public/_apis/build/builds?definitions=${DEF}&branchName=refs/heads/main&statusFilter=completed&%24top=1&api-version=7.1" \
    | tee "/tmp/gh-aw/agent/build-${DEF}.json" \
    | jq -r ".value[0] | \"def=${DEF} id=\(.id) result=\(.result)\""
done
```

If both succeeded, stop. Otherwise run ci-analysis per failed pipeline:

```bash
cat > /tmp/gh-aw/agent/run-ci-analysis.sh <<'SH'
#!/bin/bash
set -e
for DEF in 129 154; do
  R=$(jq -r '.value[0].result' "/tmp/gh-aw/agent/build-${DEF}.json")
  [ "$R" = "succeeded" ] && { echo "def=${DEF}: ok" > "/tmp/gh-aw/agent/ci-analysis-${DEF}.txt"; continue; }
  B=$(jq -r '.value[0].id' "/tmp/gh-aw/agent/build-${DEF}.json")
  pwsh .github/skills/ci-analysis/scripts/Get-CIStatus.ps1 -BuildId "$B" -ShowLogs \
    > "/tmp/gh-aw/agent/ci-analysis-${DEF}.txt" 2>&1
  sed -n '/\[CI_ANALYSIS_SUMMARY\]/,/^$/p' "/tmp/gh-aw/agent/ci-analysis-${DEF}.txt" \
    > "/tmp/gh-aw/agent/ci-summary-${DEF}.json"
done
SH
bash /tmp/gh-aw/agent/run-ci-analysis.sh
```

Record each build ID — required in every emitted artifact that references that pipeline.

## Step 3: Filter mobile jobs

Keep failures whose job name matches `ios|iossimulator|ioslike|tvos|maccatalyst|android`. Treat each pipeline as a separate input. If neither has mobile failures, stop.

## Step 4: Drill and bucket

Per failed mobile work item (helix-investigation skill): download `/console` with `curl -L` (redirects to `*.blob.core.windows.net`), extract failing test FQN, exception/assertion, Helix machine, and whether the signature repeats across jobs or recent builds.

Find earliest occurrence by scanning the last ~20 builds of the originating definition:

```bash
cat > /tmp/gh-aw/agent/recent-builds.sh <<'SH'
#!/bin/bash
DEF="${1:?def required}"
curl -sL "https://dev.azure.com/dnceng-public/public/_apis/build/builds?definitions=${DEF}&branchName=refs/heads/main&statusFilter=completed&%24top=20&api-version=7.1" \
  | jq -r '.value[] | "\(.id)|\(.result)|\(.finishTime)"'
SH
bash /tmp/gh-aw/agent/recent-builds.sh 129 | tee /tmp/gh-aw/agent/recent-builds-129.txt
bash /tmp/gh-aw/agent/recent-builds.sh 154 | tee /tmp/gh-aw/agent/recent-builds-154.txt
```

**Systemic short-circuit:** if >10 mobile jobs fail with the same signature or 5+ consecutive builds failed, treat as systemic. Skip per-work-item drill-down (one representative log suffices) and aim at the central mobile tracking issue.

**Bucket by signature:** group failures by test FQN or distinct error. Each bucket is handled independently — a PR covering bucket A does not excuse silence on bucket B. Drop buckets with <2 occurrences unless clearly a per-machine infra blip.

## Step 5: Deduplicate

**Hard rule: never open a new issue or PR when one already covers the failure.**

```bash
gh search issues "<test FQN or error key>" --repo dotnet/runtime --state open --limit 20
gh search prs    "<test FQN or error key>" --repo dotnet/runtime --state open --limit 20
gh search prs    "[mobile]"                 --repo dotnet/runtime --state open --limit 20
```

Decide:
- **Matching open PR** → `noop`. Link it from the tracking issue only if the link is not already there.
- **Matching open issue** → comment **only if** you add new info: build number not already cited, new Helix machine, new platform/arch, new consecutive-failure count, or new error signature. Read the last ~10 comments (`gh issue view <n> --comments`) first; if the latest already cites the current build, `noop`.
- **No match** → Step 6.

## Step 6: Classify and act

Using `.github/skills/mobile-platforms/SKILL.md`:

1. **Infrastructure** (provisioning/timeout/device-lost/network/Helix): tracking issue only, labels `area-Infrastructure` + `os-*`.
2. **Platform-unsupported test**: fix with `[SkipOnPlatform(...)]` or narrowed `[ConditionalFact]`.
3. **AOT/reflection**: guard with `PlatformDetection.IsReflectionEmitSupported` / `IsNotBuiltWithAggressiveTrimming`.
4. **Wrong `TargetFrameworks` / `<Compile Condition>`**: fix in the `.csproj`.
5. **Code regression on `main`**: tracking issue linking the suspect commit. No revert.
6. **Native crash (SIGSEGV/SIGBUS/SIGABRT)** or **>3 unrelated assemblies failing**: tracking issue only.

Auto-fix mechanics:

```bash
git fetch origin main
git switch -c mobile-fix-<slug> origin/main
# edit only src/** test files or their .csproj
git diff --name-only --cached  # abort if anything is under .github/, eng/, docs/, or repo root
```

Labels on every PR/issue:
- `os-ios` / `os-tvos` / `os-maccatalyst` / `os-android` for affected platforms.
- One `area-*` matching the test's library.
- `arch-arm64` / `arch-x64` only if arch-specific.

## Step 7: Output format

Every PR, issue, and comment uses exactly this template — verbatim headers, in this order:

````
## Background

2–4 sentences. What is failing (test FQN or job name), when it started (first observed build id), suspected root cause. For a PR, describe the fix. For a comment on an existing issue, state explicitly what new info is being added.

## Impact

3–6 bullets:
- **Pipeline and build:** `runtime build #<id>` and/or `runtime-extra-platforms build #<id>`.
- **Platforms:** os-*/arch tuples (`android-arm64`, `ios-arm64`, ...).
- **Jobs:** failing job names (≤5, then "... and N more").
- **Tests:** failing FQNs/assembly (≤5, "... and N more"). Omit for build/infra failures.
- **Severity:** systemic (cite consecutive-build count) vs. isolated.
- **Helix machine(s):** only if machine-specific.

## Trace

- First seen: build #<earliest-id>.
- Most recent: build #<id>.
- Failing work item(s)/job(s): <names, ≤5>.
- Sanitized excerpt (≤20 lines, fenced):

```
<compiler error, exception stack, or XHarness exit + message>
```
````

Caps: body ≤60 lines, excerpt ≤20 lines, ≤5 names per list. No @mentions. No tables unless reporting the same failure across multiple builds/pipelines.

## Step 8: Submit

- One artifact per distinct signature.
- Systemic short-circuit → one comment on the central tracking issue; no sibling issues.
- Never emit both `add_comment` and `noop` for the same failure.
- If no classification fits and no issue exists, open a tracking issue using Step 7. Do not emit `noop` with "manual investigation required".
