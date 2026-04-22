# .NET Runtime — Linux Debug Fix (release/8.0)

[![Build Status](https://dev.azure.com/dnceng-public/public/_apis/build/status/dotnet/runtime/runtime?branchName=main)](https://dev.azure.com/dnceng-public/public/_build/latest?definitionId=129&branchName=main)
[![Help Wanted](https://img.shields.io/github/issues/dotnet/runtime/help%20wanted?style=flat-square&color=%232EA043&label=help%20wanted)](https://github.com/dotnet/runtime/labels/help%20wanted)
[![Discord](https://img.shields.io/discord/732297728826277939?style=flat-square&label=Discord&logo=discord&logoColor=white&color=7289DA)](https://aka.ms/dotnet-discord)

> **This is a fork of [dotnet/runtime](https://github.com/dotnet/runtime) with a fix for custom debugger notifications on Linux (.NET 8).**

* [What is this fork?](#what-is-this-fork)
* [The fix](#the-fix)
* [How to build](#how-to-build)
* [What is .NET?](#what-is-net)
* [How can I contribute?](#how-can-i-contribute)
* [Reporting security issues and security bugs](#reporting-security-issues-and-security-bugs)
* [Filing issues](#filing-issues)
* [Useful Links](#useful-links)
* [.NET Foundation](#net-foundation)
* [License](#license)

This repo contains the code to build the .NET runtime, libraries and shared host (`dotnet`) installers for
all supported platforms, as well as the sources to .NET runtime and libraries.

## What is this fork?

This fork fixes a bug in the .NET 8 runtime (`release/8.0`) where **custom debugger notifications
(`ICorDebugProcess3::SetEnableCustomNotification`) did not work correctly on Linux**.

In the original implementation, the enabled/disabled state for custom notifications was stored
client-side in `CordbClass` (right side). This caused the state to be lost across debug sessions
and made it impossible for the runtime (left side) to filter notifications independently.

This fix moves the state to the **left side** (the runtime itself) using a per-module/type hash table
(`CustomNotificationTable`), and makes `SetEnableCustomNotification` send an IPC event to the
runtime instead of just flipping a local flag.

## The fix

Changed files:

| File | What changed |
|------|-------------|
| `src/coreclr/debug/di/process.cpp` | `SetEnableCustomNotification` now sends `DB_IPCE_SET_ENABLE_CUSTOM_NOTIFICATION` IPC event to runtime instead of setting a local flag |
| `src/coreclr/debug/di/rsclass.cpp` | Removed `m_fCustomNotificationsEnabled` field initializer |
| `src/coreclr/debug/di/rspriv.h` | Removed `m_fCustomNotificationsEnabled` field and `SetCustomNotifications`/`CustomNotificationsEnabled` methods from `CordbClass` |
| `src/coreclr/debug/ee/debugger.h` | Added `TypeInModule`, `CustomNotificationSHashTraits`, `CustomNotificationTable`; declarations for `ShouldSendCustomNotification`, `UpdateCustomNotificationTable`, `m_pCustomNotificationTable` |
| `src/coreclr/debug/ee/debugger.cpp` | Initialization of `CustomNotificationTable` in constructor; implementation of `ShouldSendCustomNotification` and `UpdateCustomNotificationTable`; new IPC event handler; filter in `SendCustomDebuggerNotification` |
| `src/coreclr/debug/inc/dbgipcevents.h` | Added `CustomNotificationData` struct to `DebuggerIPCEvent` union |
| `src/coreclr/debug/inc/dbgipceventtypes.h` | Added `DB_IPCE_SET_ENABLE_CUSTOM_NOTIFICATION` and `DB_IPCE_SET_ENABLE_CUSTOM_NOTIFICATION_RESULT` event types |
| `src/coreclr/debug/shared/dbgtransportsession.cpp` | Added `DB_IPCE_SET_ENABLE_CUSTOM_NOTIFICATION` case in `GetEventSize` |

## How to build

### Prerequisites (Arch Linux)

```bash
sudo pacman -S clang llvm lttng-ust cmake ninja python
```

### Prerequisites (Ubuntu/Debian)

```bash
sudo apt-get install clang llvm liblttng-ust-dev cmake ninja-build python3
```

### Build CoreCLR only

```bash
./build.sh -subset clr -c Release
```

Binaries will be in `artifacts/bin/coreclr/linux.x64.Release/`.

### Use your custom build with an existing .NET app

```bash
# Point your app to the custom coreclr
export DOTNET_EnableDiagnostics=1
export CORECLR_PATH=/path/to/this/repo/artifacts/bin/coreclr/linux.x64.Release
dotnet run
```

## What is .NET?

Official Starting Page: <https://dotnet.microsoft.com>

* [How to use .NET](https://docs.microsoft.com/dotnet/core/get-started) (with VS, VS Code, command-line CLI)
  * [Install official releases](https://dotnet.microsoft.com/download)
  * [Install daily builds](docs/project/dogfooding.md)
  * [Documentation](https://docs.microsoft.com/dotnet/core) (Get Started, Tutorials, Porting from .NET Framework, API reference, ...)
    * [Deploying apps](https://docs.microsoft.com/dotnet/core/deploying)
  * [Supported OS versions](https://github.com/dotnet/core/blob/main/os-lifecycle-policy.md)
* [Roadmap](https://github.com/dotnet/core/blob/main/roadmap.md)
* [Releases](https://github.com/dotnet/core/tree/main/release-notes)

## How can I contribute?

We welcome contributions! Many people all over the world have helped make this project better.

* [Contributing](CONTRIBUTING.md) explains what kinds of contributions we welcome
* [Workflow Instructions](docs/workflow/README.md) explains how to build and test
* [Get Up and Running on .NET Core](docs/project/dogfooding.md) explains how to get nightly builds of the runtime and its libraries to test them in your own projects.

## Reporting security issues and security bugs

Security issues and bugs should be reported privately, via email, to the Microsoft Security Response Center (MSRC) <secure@microsoft.com>. You should receive a response within 24 hours. If for some reason you do not, please follow up via email to ensure we received your original message. Further information, including the MSRC PGP key, can be found in the [Security TechCenter](https://www.microsoft.com/msrc/faqs-report-an-issue). You can also find these instructions in this repo's [Security doc](SECURITY.md).

Also see info about related [Microsoft .NET Core and ASP.NET Core Bug Bounty Program](https://www.microsoft.com/msrc/bounty-dot-net-core).

## Filing issues

This repo should contain issues that are tied to the runtime, the class libraries and frameworks, the installation of the `dotnet` binary (sometimes known as the `muxer`) and installation of the .NET runtime and libraries.

For other issues, please file them to their appropriate sibling repos. We have links to many of them on [our new issue page](https://github.com/dotnet/runtime/issues/new/choose).

## Useful Links

* [.NET Core source index](https://source.dot.net) / [.NET Framework source index](https://referencesource.microsoft.com)
* [API Reference docs](https://docs.microsoft.com/dotnet/api)
* [.NET API Catalog](https://apisof.net) (incl. APIs from daily builds and API usage info)
* [API docs writing guidelines](https://github.com/dotnet/dotnet-api-docs/wiki) - useful when writing /// comments
* [.NET Discord Server](https://aka.ms/dotnet-discord) - a place to discuss the development of .NET and its ecosystem

## .NET Foundation

.NET Runtime is a [.NET Foundation](https://www.dotnetfoundation.org/projects) project.

There are many .NET related projects on GitHub.

* [.NET home repo](https://github.com/Microsoft/dotnet) - links to 100s of .NET projects, from Microsoft and the community.
* [ASP.NET Core home](https://docs.microsoft.com/aspnet/core) - the best place to start learning about ASP.NET Core.

This project has adopted the code of conduct defined by the [Contributor Covenant](https://contributor-covenant.org) to clarify expected behavior in our community. For more information, see the [.NET Foundation Code of Conduct](https://www.dotnetfoundation.org/code-of-conduct).

General .NET OSS discussions: [.NET Foundation Discussions](https://github.com/dotnet-foundation/Home/discussions)

## License

.NET (including the runtime repo) is licensed under the [MIT](LICENSE.TXT) license.
