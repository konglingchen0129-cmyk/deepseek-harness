# Agent Note: Desktop payload smoke verifies the platform session lock

Status: implemented

English | [中文](2026-09-13-desktop-session-lock-payload-smoke.zh.md)

## Problem

The bundled-Node payload smoke required `fs-ext` unconditionally:

```js
function checkFsExt() {
  const fsExt = requireRuntime('fs-ext')
```

No manifest in the repository declares `fs-ext`, and `pnpm-lock.yaml` resolves no such package. The session write lock had already moved to the POSIX `tryLockExclusive` export of `@deepseek-ai/node-addon-system/flock`, whose platform packages cover `darwin-arm64`, `darwin-x64`, `linux-arm64`, and `linux-x64` only. Windows keeps its own semaphore lock and needs no addon, because [native/system/AGENTS.md](../../../../native/system/AGENTS.md) states that no Windows platform package exists for the flock capability.

`prepare-dsh.ts` runs that smoke against the filtered payload before signing and sealing, so every Windows Desktop packaging command — `package:desktop:win:x64`, `package:desktop:win:x64:unsigned`, and `package:desktop:win:x64:dir` — failed with `Cannot find module 'fs-ext'` and deleted `DSH_OUTPUT_ROOT` on the way out. The failure was a stale assertion, not a missing dependency: the check outlived the package it named.

## Decision

The smoke now verifies the session lock through the entry point consumers actually use, per platform:

- On POSIX it loads `@deepseek-ai/node-addon-system/flock` and acquires a real exclusive lock on a descriptor it owns.
- On Windows it asserts the documented rejection with `code === 'ERR_FLOCK_UNSUPPORTED_PLATFORM'`, because the capability has no Windows prebuild and the Harness locks through its own implementation there.

The Windows outcome is a contract, not an accident: `tryLockExclusive`'s JSDoc now names `ERR_FLOCK_UNSUPPORTED_PLATFORM` as the Windows result, so the assertion pins declared behavior rather than an incidental error path.

The check keeps its position and its purpose. The payload smoke still proves that the filtered `resources/dsh/node_modules` tree loads the native modules the filtering policy is supposed to retain — now including the session-lock entry, and no longer claiming a package the lock migration removed.

## Alternatives considered

**Delete the check and let Windows packaging proceed.** The gate exists to prove the filtered payload loads its native dependencies, and deleting one assertion to unblock one platform gives up that coverage for all of them. It also leaves the fixture describing a package the payload must not contain.

**Re-add `fs-ext` as a production dependency of the packaged closure.** `fs-ext` implements seek through `SetFilePointerEx` on Windows, so this would reintroduce a native compile step, add a package to `allowBuilds`, and expand the reviewed Windows package set for a capability the lock migration deliberately moved to `@deepseek-ai/node-addon-system`. It also reverses the migration instead of recording it.

**Skip the smoke for Windows targets.** A platform-conditional skip would drop the entire native check — PTY, FFI, image conversion, HTML parsing, and the session lock — from exactly the platform whose release target is Windows. The platform difference here is one assertion's expected outcome, not the smoke's applicability.

## Consequences

Windows Desktop packaging reaches electron-builder again, and the payload smoke's Windows run asserts one fewer native acquisition than a POSIX run: it proves absence of the addon by its declared error rather than proving a lock. POSIX runs keep the real acquisition. A future Windows flock addon would need this assertion revisited together with the platform matrix — the check would then reject the platform package's arrival, which is the intended prompt to record that decision.

`fs-ext` remains in two places that describe history rather than dependency: `runtime-file-policy.ts` excludes `fs-ext` compiler output when filtering an npm tree, and the [bundled-runtime Agent Note](../../implemented/architecture/2026-09-08-desktop-bundled-runtime-and-external-plugins.md) records that exclusion. Neither declares a dependency, and both stay correct for any profile that installs the package as a plugin.

## Testing

`apps/desktop/tests/fixtures/runtime-payload-smoke.mjs` is the assertion itself; it runs inside `prepare:dsh` for every Desktop packaging target, which is why the defect reached no release. The Windows path is exercised only on a Windows build host, and this change was verified by a complete `package:desktop:win:x64:unsigned` run: the smoke reported `{"node":"24.17.0","platform":"win32","arch":"x64","flock":true,"koffi":true,"sharp":true,"html":true,"pty":true`, NSIS produced `deepseek-harness-0.1.5-rc.2-win-x64.exe`, and that installer completed a silent install and launched the application.
