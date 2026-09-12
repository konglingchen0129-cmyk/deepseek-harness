# Agent Note: Directory builds take the local-only signing path

Status: implemented

English | [中文](2026-09-13-desktop-directory-build-signing.zh.md)

## Problem

`package:win:x64:dir` stops at an unpacked application directory, yet it selected the release signing path and demanded Windows EV certificate inputs. `package-target.ts` passed only `invocation.unsigned` into `desktopElectronBuilderEnvironment`, which unconditionally selects the flag:

```ts
const selected: NodeJS.ProcessEnv = { ...environment, DSH_DESKTOP_UNSIGNED: unsigned ? '1' : '0' }
```

A directory build therefore forced `DSH_DESKTOP_UNSIGNED=0` before electron-builder started, so `createElectronBuilderConfig` resolved a Windows token signer and threw `DSH_DESKTOP_WINDOWS_CER_FILE must identify the public X.509 leaf certificate file` at config load. The failure happened after every preparation step, including the payload smoke, so a machine without a SafeNet token lost the whole run.

The flag cannot come from the caller either: the overwrite above discards an inherited `DSH_DESKTOP_UNSIGNED=1`, and `--dir` may not be combined with `--unsigned`. The config module itself is correct — evaluated directly under a forced unsigned, win32, x64 environment it resolves and writes to `unsigned-artifacts` — which places the defect entirely in the environment construction.

## Decision

`--dir` joins the local-only condition: `desktopElectronBuilderEnvironment(targetEnv, invocation.unsigned || invocation.directory)`.

A directory build now selects the same environment as `--unsigned` — the Windows signer omitted, signing credential names stripped, `CSC_IDENTITY_AUTO_DISCOVERY` disabled — while `package:win:x64` keeps the strict requirement. The `directory` flag continues to control electron-builder's `--dir` argument and remains the reason no release record is written.

## Alternatives considered

**Permit `--dir` together with `--unsigned`.** Two flags describing one state ask every caller to know which combination is legal, and the scripts in `package.json` would still need a third spelling for the unpacked case.

**Let a directory build resolve an empty signer when the four inputs are absent.** This hides a missing credential on the release path too, so a mistyped variable in a release run would produce an unsigned installer instead of failing.

**Leave it and document copying `win-unpacked` out of `unsigned-artifacts`.** The documented `package:desktop:*:dir` commands exist for exactly this inspection, so an unusable command plus a manual workaround is worse than fixing the condition.

## Consequences

Directory builds succeed without signing credentials, which is the point of the target. They keep producing an unpacked tree rather than a publishable artifact, they still write no release record, and they still select no auto-update channel. Release packaging (`package:desktop:win:x64`) is unchanged: it still demands the certificate, SignTool, container, and token PIN, and any missing input still fails the run.

POSIX directory builds are unaffected: the unsigned branch is reachable only for `--unsigned`, which the invocation parser restricts to `win-x64`.

## Testing

`package:desktop:win:x64:dir` on a Windows x64 host without the four signing variables reaches electron-builder and writes the unpacked application. The payload smoke and the runtime verification are unchanged by this fix and continue to run before it, so a passing directory build still proves the filtered runtime and its native modules load.
