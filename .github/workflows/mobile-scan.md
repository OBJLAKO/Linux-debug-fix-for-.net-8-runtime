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

Scan two pipelines on `main` for Apple mobile and Android failures, triage, and propose fixes:

- **`runtime`** (AzDO definition **129**, `dnceng-public/public`) -- main rolling CI. Public mirror of the internal pipeline `dnceng/internal` def 1104.
- **`runtime-extra-platforms`** (AzDO definition **154**, `dnceng-public/public`) -- daily extra-platforms coverage.

Both definitions live in the same public project and are anonymously accessible. Scan both in every run.

Sanitize log excerpts (user paths, tokens, auth headers) before posting anything.

## Conventions

- Every shell call is a fresh subshell. Persist state to files under `/tmp/gh-aw/agent/`.
- `$(...)`, `${var@P}`, `-o` and `>` are blocked by the shell guard. Use `| tee file` and write complex commands to a script, then `bash script.sh`.
- URL-encode OData `$` params (`%24top`).

## Step 1: Load skills

Read `.github/skills/mobile-platforms/SKILL.md`. Then fetch the helix-investigation skill for console-log drill-down:

```bash
mkdir -p /tmp/gh-aw/agent
curl -sL "https://raw.githubusercontent.com/dotnet/arcade-skills/f866c30a5b58e76492c90fd089082eb5f7e81a87/plugins/dotnet-dnceng/skills/helix-investigation/SKILL.md" | tee /tmp/gh-aw/agent/helix-investigation-skill.md > /dev/null
```

## Step 2: Resolve the latest completed build for each pipeline

Fetch the latest completed build for both definitions:

```bash
for DEF in 129 154; do
  curl -sL "https://dev.azure.com/dnceng-public/public/_apis/build/builds?definitions=${DEF}&branchName=refs/heads/main&statusFilter=completed&%24top=1&api-version=7.1" \
    | tee "/tmp/gh-aw/agent/build-${DEF}.json" \
    | jq -r ".value[0] | \"def=${DEF} id=\(.id) result=\(.result)\""
done
```

If both results are `succeeded`, stop.

Record each build ID -- it MUST appear in every output (PR body, issue body, comment) that references that pipeline.

Run ci-analysis for each failed pipeline (skip pipelines whose latest build succeeded):

```bash
cat > /tmp/gh-aw/agent/run-ci-analysis.sh <<'SH'
#!/bin/bash
set -e
for DEF in 129 154; do
  RESULT=$(jq -r '.value[0].result' "/tmp/gh-aw/agent/build-${DEF}.json")
  if [ "$RESULT" = "succeeded" ]; then
    echo "def=${DEF}: succeeded, skipping" > "/tmp/gh-aw/agent/ci-analysis-${DEF}.txt"
    continue
  fi
  BUILD_ID=$(jq -r '.value[0].id' "/tmp/gh-aw/agent/build-${DEF}.json")
  echo "def=${DEF} build=${BUILD_ID}"
  pwsh .github/skills/ci-analysis/scripts/Get-CIStatus.ps1 -BuildId "$BUILD_ID" -ShowLogs \
    > "/tmp/gh-aw/agent/ci-analysis-${DEF}.txt" 2>&1
  sed -n '/\[CI_ANALYSIS_SUMMARY\]/,/^$/p' "/tmp/gh-aw/agent/ci-analysis-${DEF}.txt" \
    > "/tmp/gh-aw/agent/ci-summary-${DEF}.json"
done
SH
bash /tmp/gh-aw/agent/run-ci-analysis.sh
```

## Step 3: Filter to mobile jobs

For each pipeline's `ci-summary-<def>.json`, keep only failures whose job names match `ios`, `iossimulator`, `ioslike`, `tvos`, `maccatalyst`, or `android`. Treat each pipeline's results as a separate input set going into Step 4. If neither pipeline has mobile failures, stop.

Note: the `runtime` pipeline (def 129) runs a different job shape than `runtime-extra-platforms` (def 154). Typical mobile jobs in def 129 include `Build ios-arm64 Release AllSubsets_NativeAOT_Smoke`, `Build android-arm64 Release AllSubsets_Mono`, etc. Jobs in def 154 are the full-coverage matrix documented in `.github/skills/mobile-platforms/SKILL.md`.

## Step 4: Drill into Helix console logs

For each failed mobile work item, follow the helix-investigation skill: download the `/console` log (pass `-L`; redirects to `*.blob.core.windows.net`), extract the failing test FQN, the assertion/exception, the Helix machine name, and whether the same failure repeats across jobs or prior builds.

Capture the earliest build where the failure first appeared. Query the last ~20 builds of the **originating definition** (129 or 154) to find it:

```bash
cat > /tmp/gh-aw/agent/recent-builds.sh <<'SH'
#!/bin/bash
set -e
DEF="${1:?definition id required}"
curl -sL "https://dev.azure.com/dnceng-public/public/_apis/build/builds?definitions=${DEF}&branchName=refs/heads/main&statusFilter=completed&%24top=20&api-version=7.1" \
  | jq -r '.value[] | "\(.id)|\(.result)|\(.finishTime)"'
SH
bash /tmp/gh-aw/agent/recent-builds.sh 129 | tee /tmp/gh-aw/agent/recent-builds-129.txt
bash /tmp/gh-aw/agent/recent-builds.sh 154 | tee /tmp/gh-aw/agent/recent-builds-154.txt
```

**Systemic-failure short-circuit.** If >10 mobile jobs fail in the current build with the same signature, OR the last 5+ consecutive builds all failed, treat this as systemic. Skip per-work-item drill-down (one representative console log is enough) and jump to Step 5 targeting the central mobile tracking issue.

**Group failures by signature.** Before Step 5, bucket work-item failures by test FQN or distinct error signature. Each bucket is handled independently in Steps 5-8: a PR covering bucket A does not excuse silence on bucket B. Ignore buckets with <2 occurrences only if the error is clearly a per-machine infra blip.

## Step 5: Deduplicate before acting

**Hard rule: never open a new issue or PR when one already covers the failure.**

Search first (do all three):

```bash
gh search issues "<test FQN or error key>" --repo dotnet/runtime --state open --limit 20
gh search prs    "<test FQN or error key>" --repo dotnet/runtime --state open --limit 20
gh search prs    "[mobile]"                 --repo dotnet/runtime --state open --limit 20
```

Decide:

- **Matching open PR exists** → `noop`. Add a comment to the related tracking issue linking the PR only if that link is not already present.
- **Matching open tracking issue exists** → comment **only if** you bring new information: a build number the issue does not already cite, a new Helix machine, a platform/arch not yet listed, a new failure pattern (e.g., N consecutive failures), or a distinct error signature. Use the template in Step 7. If no new info, `noop`.
- **No match** → proceed to Step 6.

Before posting a comment, read the issue's latest ~10 comments (`gh issue view <n> --repo dotnet/runtime --comments`). If the most recent comment already cites the current build ID, `noop`.

## Step 6: Classify and act

Classify using `.github/skills/mobile-platforms/SKILL.md`:

1. **Infrastructure** (provisioning/timeout/device-lost/network/Helix agent): open a tracking issue (labels `area-Infrastructure` + `os-*`). No code fix.
2. **Platform-unsupported test**: auto-fix with `[SkipOnPlatform(...)]` or a narrowed `[ConditionalFact]` predicate on the specific test.
3. **AOT/reflection-dependent test**: auto-fix by guarding with `PlatformDetection.IsReflectionEmitSupported` / `IsNotBuiltWithAggressiveTrimming`.
4. **Test project excludes a mobile TFM incorrectly**: auto-fix via `TargetFrameworks` / `<Compile Condition>` in the `.csproj`.
5. **Code regression on `main`**: open a tracking issue linking the suspect commit (`git log --oneline --since='3 days ago' -- <path>`). Do not revert.
6. **Native crash (SIGSEGV/SIGBUS/SIGABRT) or >3 unrelated test assemblies failing**: tracking issue only, no auto-fix.

Auto-fix mechanics:

```bash
git fetch origin main
git switch -c mobile-fix-<slug> origin/main
# edit only src/** test files or their .csproj
git diff --name-only --cached   # abort if anything is under .github/, eng/, docs/, or repo root
git add <specific file>
```

Required labels on the PR/issue (pass via safeoutputs):
- OS labels matching affected platforms: `os-ios`, `os-tvos`, `os-maccatalyst`, `os-android`.
- One `area-*` label matching the test's library.
- `arch-arm64` / `arch-x64` only if the failure is architecture-specific.

## Step 7: Output format

**Every PR body, issue body, and comment uses the same three-paragraph template. Nothing else. No preambles, no step-by-step narration, no full console dumps.**

Always include the originating pipeline's build number in the Impact paragraph. Use the following format to disambiguate:
- For def 129: `runtime build #<id>`
- For def 154: `runtime-extra-platforms build #<id>`

If the same signature fails on both pipelines in the current scan, cite both build numbers.

```
**Why.** <1-3 sentences: failure class + the fix (for PRs) or suspected cause (for issues/comments).>

**Impact.** <1-2 sentences: affected platforms (os-*, arch), affected test FQN(s) or assembly, originating pipeline and build (e.g. "runtime build #<id>" and/or "runtime-extra-platforms build #<id>"). For systemic failures, cite the consecutive-build pattern (e.g. "last 20 builds all failed").>

**Trace.** First seen in build #<earliest-id>; most recent #<id>. Helix machine(s): <names>. Sanitized excerpt:
```
<<=15 lines from console log or test output>
```
```

Hard caps:
- Total body ≤ 40 lines.
- Error excerpt ≤ 15 lines.
- ≤ 5 test/assembly names in Impact; if more, use "... and N more".
- No @mentions. No markdown tables unless reporting >1 build recurrence.

The same template applies whether opening a PR, opening an issue, or commenting on an existing issue. The only difference is what goes in Why: for a comment, Why states why *this* comment adds value (what new info).

## Step 8: Submit

- Emit at most one artifact per distinct failure signature.
- For systemic failures (Step 4 short-circuit), the single comment on the central tracking issue is the final output. Do not also open sibling issues.
- Do not emit both `add_comment` and `noop` for the same failure.
- If no classification fits and no issue exists, open a short tracking issue using the Step 7 template -- do not emit `noop` with "manual investigation required".
