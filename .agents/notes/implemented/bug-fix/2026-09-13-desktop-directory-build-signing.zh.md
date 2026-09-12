# Agent Note: Directory builds take the local-only signing path

Status: implemented

[English](2026-09-13-desktop-directory-build-signing.md) | 中文

## Problem

`package:win:x64:dir` 停在解包后的应用目录，却选择了正式发布签名路径，并要求 Windows EV 证书输入。`package-target.ts` 只把 `invocation.unsigned` 传进 `desktopElectronBuilderEnvironment`，而该函数无条件选择该标志：

```ts
const selected: NodeJS.ProcessEnv = { ...environment, DSH_DESKTOP_UNSIGNED: unsigned ? '1' : '0' }
```

因此目录构建在 electron-builder 启动前就把 `DSH_DESKTOP_UNSIGNED` 强制为 `0`，使 `createElectronBuilderConfig` 解析出 Windows Token 签名器，并在加载配置时抛出 `DSH_DESKTOP_WINDOWS_CER_FILE must identify the public X.509 leaf certificate file`。失败发生在所有准备步骤（含产物 smoke）之后，所以没有 SafeNet Token 的机器会白跑整轮。

该标志也无法由调用方提供：上面的覆写会丢弃继承来的 `DSH_DESKTOP_UNSIGNED=1`，而 `--dir` 不允许与 `--unsigned` 组合。配置模块本身是正确的——在强制的 unsigned、win32、x64 环境下直接求值，它能正常解析并写入 `unsigned-artifacts`——这说明缺陷完全位于环境构造处。

## Decision

`--dir` 并入本地构建条件：`desktopElectronBuilderEnvironment(targetEnv, invocation.unsigned || invocation.directory)`。

目录构建现在与 `--unsigned` 选择同一套环境——省略 Windows 签名器、剔除签名凭据名、关闭 `CSC_IDENTITY_AUTO_DISCOVERY`——而 `package:win:x64` 保持严格校验。`directory` 标志继续控制 electron-builder 的 `--dir` 参数，并继续作为不写发布记录的原因。

## Alternatives considered

**允许 `--dir` 与 `--unsigned` 同时使用。** 两个标志描述同一种状态，等于要求每个调用方记住哪种组合合法，而 `package.json` 里的脚本仍然需要第三种写法来表达解包场景。

**让目录构建在缺少四个输入时解析出空签名器。** 这同时会掩盖发布路径上的凭据缺失，发布运行中一个拼错的变量将从"失败"变成"产出未签名安装包"。

**保持现状，改为文档化"从 `unsigned-artifacts` 里复制 `win-unpacked`"。** 官方文档列出的 `package:desktop:*:dir` 命令正是为了这种检查场景而存在，留下一条不可用的命令加一个手工绕法，比修正判定条件更糟。

## Consequences

目录构建在没有签名凭据时也能成功，这正是该目标的意义。它仍然产出解包树而非可发布产物，仍然不写发布记录，也仍然不选择自动更新通道。正式发布打包（`package:desktop:win:x64`）不变：仍然要求证书、SignTool、容器和 Token PIN，任何输入缺失仍然使整轮失败。

POSIX 目录构建不受影响：unsigned 分支只对 `--unsigned` 可达，而调用解析器将其限制为 `win-x64`。

## Testing

在没有四个签名变量的 Windows x64 主机上运行 `package:desktop:win:x64:dir`，可走到 electron-builder 并写出解包应用。产物 smoke 与运行时校验不受本次修复影响，仍在它之前运行，因此目录构建通过依旧证明过滤后的运行时及其原生模块可加载。
