# Agent Note: Package commands preselect pnpm's non-interactive answers

Status: implemented

[English](2026-09-13-desktop-noninteractive-pnpm-purge.md) | 中文

## Problem

每一条有文档的桌面打包命令都以 pnpm 起步，而只要已安装的 `node_modules` 是由另一个包管理器版本生成的，pnpm 就会拒绝运行 package 脚本：

```
[ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY] Aborted removal of modules directory due to no TTY
```

清理并不是罕见的损坏路径。`package.json#packageManager` 声明了 `pnpm@11.7.0`，因此全局 pnpm 是任何其他版本的机器都会在 `node_modules/.modules.yaml` 里记下那个版本，下一次 `pnpm --dir apps/desktop run package:win:x64:dir` 就会看到不一致，要求删除并重装整棵依赖树。有 TTY 时这个提示无害；而在编辑器任务、管道 shell、服务进程，或记录下这次失败的 harness 里，pnpm 会直接中止，运行在 `package-target.ts` 构建任何东西之前就死掉。同样的问题也出现在复现 Windows 打包工作的评审者那里（[#6501](https://github.com/deepseek-ai/deepseek-harness/discussions/6501)）：那儿修掉了两处打包缺陷，而第三个失败只在非交互式终端里运行命令时才出现。

`package-target.ts` 本来就掌管每个打包子进程继承的环境：`withoutWindowsSigningEnvironment` 与 `withoutDesktopUploadCredentials` 的存在，是为了让一次打包运行不会意外签名或上传。同一个构造点此前对 pnpm 自身的确认行为没有任何交代。

## Decision

`withNonInteractivePackageManager` 为打包命令的子进程选定 pnpm 的非交互式答复：

```ts
export function withNonInteractivePackageManager(environment: NodeJS.ProcessEnv): NodeJS.ProcessEnv {
  if (environment.CI !== undefined) return environment
  return { ...environment, CI: 'true' }
}
```

`main` 在构建共享打包环境的位置应用它，因此 `buildEnv` 以及由它派生的一切环境 —— 用于运行时准备的 `targetEnv`、用于 electron-builder 的 `electronBuilderEnv` —— 都带上 `CI=true`。调用方环境里显式存在的 `CI` 原样返回，因此把 `CI=false` 写进环境的调用方仍保留 pnpm 的拒绝行为。`package.json` 里的 package 脚本保持不变；这层保证依托子进程环境，而不要求每个调用方多写一个变量。

`CI=true` 正是 pnpm 为这一类提示提供的开关：它同时为 pnpm 的其他确认选定非交互式默认值，而这条路径上唯一可达的确认就是 `node_modules` 清理。准备类子进程本来就以净化过的环境运行（移除了 `NODE_OPTIONS`、`NODE_PATH` 以及 `npm`/`pnpm`/`corepack` 变量），因此 `CI` 不会泄漏进桌面运行时日后随包分发的东西。

## Alternatives considered

**在 `package.json` 的每个 `package:*` 脚本里加 `CI=true`。** 这些脚本只是转发器，而环境已经在一个函数里组装完成；把变量复制到各个脚本名上，会让任何新的入口 —— `--prepare-only`、`--unsigned`、将来的某个 target —— 依然可以把这个中止带回来。

**让运维自己导出 `CI=true` 或 `confirmModulesPurge=false`。** 失败的那条命令是有文档的命令，而报错信息指出的是 pnpm 的变量、而不是本仓库的要求，贡献者无从判断在这里清理是否安全。把绕行方案写进文档，等于让第一次运行保持损坏。

**给 pnpm 子进程传 `--config.confirmModulesPurge=true`。** 它改的是一个配置键而不是进程环境，但中止是由 pnpm 自己的依赖状态检查在目标命令运行之前抛出的，而 package 脚本再生成的嵌套 pnpm 调用还得把该参数重复一遍。`CI` 才是受支持、且会被继承的开关。

**让守卫无条件生效，覆盖继承来的 `CI`。** 在真实 CI 任务里以 `CI=false` 运行的运维是明确要求 pnpm 保留拒绝行为的；打包脚本不是推翻这个选择的合适位置。

## Consequences

在 pnpm 版本与 `package.json#packageManager` 不一致的机器上，打包命令现在能在非交互式 shell 里跑完。凡 pnpm 本会清理并重装依赖树之处，如今都在无人值守下完成 —— 这正是该命令一直需要的行为：这次安装与 TTY 用户会批准的那次完全相同。依赖 pnpm 的中止来保护本地 `node_modules` 免受版本驱动清理的调用方，只有设置 `CI=false` 才能得到它；守卫不会悄悄限制这一选择。

守卫的作用域仅限于打包命令的子进程，不改变任何运行时行为、产物，以及签名或上传路径。macOS 与 Linux 打包沿用同一套环境构造，两种情况都不受影响：它们的打包命令遇到版本不一致同样会以这个 TTY 错误失败，因此守卫消除的是一个偶然的平台差异，而不是新增差异。

## Testing

在 Windows x64 主机上，`pnpm --dir apps/desktop run package:win:x64:dir` 能回到中止之前本应到达的打包步骤：`prepare:runtime`、`prepare:packages`、带 payload smoke 的 `prepare:dsh`，以及 electron-builder。验证时启动器环境里完全没有 `CI`，因此非交互式选择只可能来自打包模块。

`package-target.spec.ts` 直接钉住这条环境规则：不含 `CI` 的环境会得到 `CI=true`，显式的 `CI=false` 被保留，且传入的对象不被修改。
