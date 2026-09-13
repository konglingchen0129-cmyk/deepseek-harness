# Agent Note: Package commands preselect pnpm's non-interactive answers

Status: implemented

English | [中文](2026-09-13-desktop-noninteractive-pnpm-purge.zh.md)

## Problem

Every documented Desktop package command starts with pnpm, and pnpm refuses to run a package script when the installed `node_modules` was materialized by a different package manager version:

```
[ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY] Aborted removal of modules directory due to no TTY
```

The purge is not a rare corruption path. `package.json#packageManager` declares `pnpm@11.7.0`, so a host whose global pnpm is any other version leaves `node_modules/.modules.yaml` recording that other version, and the next `pnpm --dir apps/desktop run package:win:x64:dir` sees the mismatch and asks to delete and reinstall the dependency tree. With a TTY the prompt is harmless; in an editor task, a piped shell, a service, or the harness that recorded this failure, pnpm aborts instead and the run dies before `package-target.ts` builds anything. The same abort reached reviewers reproducing the Windows packaging work in [#6501](https://github.com/deepseek-ai/deepseek-harness/discussions/6501), where two packaging defects were fixed and this third failure appeared only when the commands were run from a non-interactive terminal.

`package-target.ts` already owns the environment every package subprocess inherits: `withoutWindowsSigningEnvironment` and `withoutDesktopUploadCredentials` exist so a packaging run cannot accidentally sign or upload. The same construction point had no answer for pnpm's own confirmation behavior.

## Decision

`withNonInteractivePackageManager` selects pnpm's non-interactive answers for the packaging command's children:

```ts
export function withNonInteractivePackageManager(environment: NodeJS.ProcessEnv): NodeJS.ProcessEnv {
  if (environment.CI !== undefined) return environment
  return { ...environment, CI: 'true' }
}
```

`main` applies it where the shared build environment is built, so `buildEnv` and every environment derived from it — `targetEnv` for runtime preparation and `electronBuilderEnv` for electron-builder — carry `CI=true`. An explicit `CI` in the caller's environment is returned untouched, so a caller that sets `CI=false` keeps pnpm's refusal in place. The package scripts in `package.json` are unchanged; the guarantee rides on the subprocess environment rather than on every caller spelling an extra variable.

`CI=true` is the switch pnpm documents for exactly this class of prompt: it also selects non-interactive defaults for its other confirmations, and the only confirmation reachable on this path is the `node_modules` purge. Preparation subprocesses already run with a scrubbed environment (`NODE_OPTIONS`, `NODE_PATH`, and `npm`/`pnpm`/`corepack` variables removed), so `CI` does not leak into anything the Desktop runtime later ships.

## Alternatives considered

**Add `CI=true` to each `package:*` script in `package.json`.** The scripts are thin forwarders and the environment is already assembled in one function; duplicating the variable across script names leaves any new entry point — `--prepare-only`, `--unsigned`, a future target — free to reintroduce the abort.

**Tell operators to export `CI=true` or `confirmModulesPurge=false` themselves.** The command that fails is a documented one, and the failure message names a pnpm variable rather than this repository's requirement, so a contributor cannot tell whether purging is safe here. Documenting the workaround leaves the first run broken.

**Pass `--config.confirmModulesPurge=true` to the pnpm child.** It changes one config key instead of the process environment, but the abort is raised by pnpm's own dependency-status check before the requested command runs, and nested pnpm invocations spawned by package scripts would need the argument repeated. `CI` is the supported, inherited switch.

**Make the guard unconditional, overriding an inherited `CI`.** An operator running from a real CI job with `CI=false` deliberately asked pnpm to keep its refusal; a packaging script is the wrong place to overrule that.

## Consequences

Package commands run to completion in non-interactive shells on hosts whose pnpm version differs from `package.json#packageManager`. Where pnpm would have purged and reinstalled the dependency tree, it now does so unattended, which is the behavior the command needed all along — the install is the same one a TTY user would have approved. A caller that relies on pnpm's abort to protect local `node_modules` from a version-driven purge gets it only by setting `CI=false`; the guard does not silently constrain that choice.

The guard is scoped to the packaging command's children and changes no runtime behavior, no artifact, and no signing or upload path. macOS and Linux packaging inherit the same environment construction and are unaffected either way: their package commands fail on a version mismatch with the same TTY error, so the guard removes an incidental platform difference rather than adding one.

## Testing

`pnpm --dir apps/desktop run package:win:x64:dir` on a Windows x64 host returns to the packaging steps it reached before the abort: `prepare:runtime`, `prepare:packages`, `prepare:dsh` with its payload smoke, and electron-builder. The guard was exercised with `CI` absent from the launcher environment, so the packaging module is the only source of the non-interactive choice.

`package-target.spec.ts` pins the environment rule directly: an environment without `CI` gains `CI=true`, an explicit `CI=false` is preserved, and the input object is not mutated.
