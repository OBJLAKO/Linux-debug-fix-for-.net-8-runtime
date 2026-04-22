---
name: mobile-platforms
description: "Domain knowledge for triaging and fixing .NET failures on Apple mobile (iOS, tvOS, MacCatalyst) and Android. Use when runtime-extra-platforms or mobile CI is failing, when investigating iOS, tvOS, MacCatalyst, iossimulator, tvossimulator, or Android build/test failures, or when a change touches mobile pipeline YAML, AppleAppBuilder/AndroidAppBuilder, code signing, provisioning, simulator/emulator startup, platform conditionals, or NativeAOT-on-mobile behavior. Covers failure triage (infrastructure vs code), CI pipeline structure, platform-specific code paths, and NativeAOT compilation on mobile."
---

# Mobile Platforms

Triage and fix .NET failures on Apple mobile (iOS, tvOS, MacCatalyst) and Android in dotnet/runtime.

Apple forbids JIT in production (simulators allow it), requires code signing, has six OS variants (`ios`, `iossimulator`, `tvos`, `tvossimulator`, `maccatalyst` x64+arm64) that typically need identical handling. Android requires cross-compilation across four architectures. CoreCLR is the primary runtime; NativeAOT is the default publish; Mono also runs in CI.

## CI pipelines (`dnceng-public/public`, anon-accessible)

- **`runtime`** (def **129**) — main rolling CI. Public mirror of internal `dnceng/internal` def 1104. Mobile jobs are a Smoke / NativeAOT subset (e.g. `Build ios-arm64 Release AllSubsets_NativeAOT_Smoke`).
- **`runtime-extra-platforms`** (def **154**) — daily full mobile matrix. Jobs follow `{platform} {config} {subset}`:

| Platform      | Job pattern                  | Variants                                |
|---------------|------------------------------|-----------------------------------------|
| iOS simulator | `iossimulator-{x64,arm64}`   | CoreCLR, Mono, NativeAOT                |
| tvOS          | `tvos_arm64`                 | CoreCLR, Mono, NativeAOT                |
| MacCatalyst   | `maccatalyst-{x64,arm64}`    | CoreCLR, Mono, NativeAOT, AppSandbox    |
| Android       | `android-{arm,arm64,x64,x86}`| CoreCLR, Mono, NativeAOT                |

Suffixes: empty (libraries), `RuntimeTests`, `Smoke`, `AppSandbox`, `Interp`.

## Where to look by failure type

| Symptom                              | Apple                                                                                     | Android                                                |
|--------------------------------------|-------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Build / packaging (pre-test)         | `src/mono/msbuild/apple/build/`, `src/tasks/AppleAppBuilder/`                              | `src/mono/msbuild/android/build/`, `src/tasks/AndroidAppBuilder/` |
| Test runner / test targets           | `src/libraries/Common/tests/AppleTestRunner/`, `eng/testing/tests.ioslike.targets`        | `src/libraries/Common/tests/AndroidTestRunner/`, `eng/testing/tests.android.targets` |
| Native interop / P/Invoke            | `src/native/libs/System.Security.Cryptography.Native.Apple/`, `src/native/libs/System.Native/ios/` | `src/native/libs/` (shared with Linux/Bionic)    |
| Pipeline YAML                        | `eng/pipelines/extra-platforms/runtime-extra-platforms-{ioslike,ioslikesimulator,maccatalyst}.yml` | `eng/pipelines/extra-platforms/runtime-extra-platforms-{android,androidemulator}.yml` |

Apple signing uses `DevTeamProvisioning`: `-` for simulators, `adhoc` for MacCatalyst. A provisioning/signing error means the wrong value is being passed in the pipeline YAML.

## NativeAOT on mobile

`Directory.Build.targets` evaluates before NuGet `.targets`, so `_IsApplePlatform` from `Microsoft.DotNet.ILCompiler` is not visible in `eng/toolAot.targets`. For Apple mobile library tests, `ILCompilerTargetsPath` must be set to `$(CoreCLRBuildIntegrationDir)Microsoft.DotNet.ILCompiler.SingleEntry.targets` with `_IlcReferencedAsPackage=false`. Key files: `eng/targetingpacks.targets`, `eng/toolAot.targets`.

## Gotchas

- Six Apple OS variants — a one-variant fix usually needs all six; missing one shows up on the next pipeline run.
- `dsymutil` writes errors to stdout, not stderr. `2>/dev/null` does not suppress them; use `>/dev/null 2>&1`.
- Android has four arches; `android-x86` (32-bit) exposes issues the 64-bit targets hide.
- MSBuild evaluation order: package `.targets` properties are unavailable in `Directory.Build.targets` and `eng/toolAot.targets`.

## Triage: infrastructure vs code

**Infrastructure** — timeouts with no test output, "device not found", provisioning/code-signing errors, Helix payload download errors, machine-specific passes, "no space left on device". File on an existing tracking issue with a table row:

| Build | Date | Machine | Job | Error |
|---|---|---|---|---|

If no matching issue, open one with `area-Infrastructure` + `os-*` labels. If the root cause is code- or config-actionable (not raw device provisioning), open a PR instead.

**Code** — the test was passing, the error references managed code, a recent commit touched platform-sensitive code. Start with `git log --oneline --since='3 days ago' -- <path>`. Common patterns: new API missing mobile handling, `#if` missing a mobile target, P/Invoke change with no mobile native impl, test assuming desktop behavior (process spawn, JIT, fs paths).

## Self-improvement

When a mobile workflow discovers a new pattern or workaround, record it as a comment on the tracking issue (or a new one labeled `area-Infrastructure-mono`) so the team can fold it back into this file. Keep mobile-fix PRs focused — do not edit this file in PRs that are not explicitly documenting mobile work.
