# Agent Note: Desktop payload smoke verifies the platform session lock

Status: implemented

[English](2026-09-13-desktop-session-lock-payload-smoke.md) | 中文

## Problem

内置 Node 的产物 smoke 无条件要求 `fs-ext`：

```js
function checkFsExt() {
  const fsExt = requireRuntime('fs-ext')
```

仓库中没有任何清单声明 `fs-ext`，`pnpm-lock.yaml` 也未解析出该包。会话写锁早已迁移到 `@deepseek-ai/node-addon-system/flock` 的 POSIX `tryLockExclusive` 导出，而它的平台包只覆盖 `darwin-arm64`、`darwin-x64`、`linux-arm64` 和 `linux-x64`。[native/system/AGENTS.md](../../../../native/system/AGENTS.md) 明确说明 flock 能力不需要 Windows 平台包，Windows 保留 Harness 自身的信号量锁实现。

`prepare-dsh.ts` 在签名和封存之前用该 smoke 校验过滤后的产物，因此每一条 Windows 桌面打包命令——`package:desktop:win:x64`、`package:desktop:win:x64:unsigned` 和 `package:desktop:win:x64:dir`——都以 `Cannot find module 'fs-ext'` 失败，并在退出前删除 `DSH_OUTPUT_ROOT`。这是过期的断言，不是缺失的依赖：这条检查活得比它所指的包更久。

## Decision

smoke 现在按平台通过消费者真正使用的入口校验会话锁：

- POSIX 上加载 `@deepseek-ai/node-addon-system/flock`，并在自己持有的描述符上取得一次真实的独占锁。
- Windows 上断言文档化的拒绝，即 `code === 'ERR_FLOCK_UNSUPPORTED_PLATFORM'`，因为该能力没有 Windows 预构建，Harness 在 Windows 上通过自身实现加锁。

Windows 的结果是契约而非偶然：`tryLockExclusive` 的 JSDoc 现在把 `ERR_FLOCK_UNSUPPORTED_PLATFORM` 写为 Windows 的行为，因此该断言锁定的是已声明的行为，而不是某个顺带的错误路径。

检查的位置和目的保持不变。产物 smoke 仍然证明过滤后的 `resources/dsh/node_modules` 树能加载过滤策略本应保留的原生模块——如今包含会话锁入口，且不再声称某个已被锁迁移移除的包。

## Alternatives considered

**删除该检查，让 Windows 打包通过。** 这道门的存在意义是证明过滤后的产物能加载其原生依赖；为放行一个平台而删掉一条断言，等于对所有平台放弃了这份覆盖。它还会让夹具继续描述一个产物中不应存在的包。

**把 `fs-ext` 重新加为打包闭包的生产依赖。** `fs-ext` 在 Windows 上用 `SetFilePointerEx` 实现 seek，这样做会重新引入原生编译步骤、把该包加入 `allowBuilds`，并为锁迁移已刻意改用 `@deepseek-ai/node-addon-system` 的能力扩大受审核的 Windows 包集合。它还会倒退这次迁移，而不是把迁移记录下来。

**对 Windows 目标跳过整个 smoke。** 按平台条件跳过会把整块原生检查（PTY、FFI、图像转换、HTML 解析和会话锁）从恰恰以 Windows 为发布目标的平台上抽掉。这里的平台差异只是一条断言的期望结果，不是 smoke 是否适用。

## Consequences

Windows 桌面打包重新能走到 electron-builder；产物 smoke 的 Windows 运行比 POSIX 运行少证明一次原生获取：它通过声明的错误证明加载器缺席，而不是证明取到了锁。POSIX 运行仍然做真实获取。将来若出现 Windows flock 加载器，需要连同平台矩阵一起重新审视这条断言——那时该检查会拒绝平台包的到来，这正是要求记录该决定的提示。

`fs-ext` 仍留在两处描述历史而非依赖的地方：`runtime-file-policy.ts` 在过滤 npm 树时排除 `fs-ext` 的编译产物，[bundled-runtime Agent Note](../../implemented/architecture/2026-09-08-desktop-bundled-runtime-and-external-plugins.zh.md) 记录了该排除。两者都没有声明依赖，并且对任何以插件方式安装该包的 profile 仍然成立。

## Testing

`apps/desktop/tests/fixtures/runtime-payload-smoke.mjs` 就是断言本身；它在每个桌面打包目标的 `prepare:dsh` 中运行，这也是该缺陷从未进入发布的原因。Windows 路径只在 Windows 构建主机上被走到，本次改动通过一次完整的 `package:desktop:win:x64:unsigned` 运行验证：smoke 报告 `{"node":"24.17.0","platform":"win32","arch":"x64","flock":true,"koffi":true,"sharp":true,"html":true,"pty":true`，NSIS 产出 `deepseek-harness-0.1.5-rc.2-win-x64.exe`，该安装包完成静默安装并启动了应用。
