# SourceGit 2026.21-r1 的 Gentoo 源码打包说明

[English](README.md)

本版以 SourceGit 的 `v2026.21` 标签为源码基线，对应提交为
`a4ae633c51c922150ea113913c682a8213308560`。构建使用 Native AOT，安装后的
SourceGit 可执行文件不需要 .NET 运行时。

修改范围包括 SourceGit 自身的 NuGet 依赖清单、可选的原生 Wayland 支持，
以及可选的 KWin 图标兼容规则。SDK 10.0.111 的官方归档作为构建输入，
只在 `${WORKDIR}/dotnet-sdk` 中临时解包使用，不依赖系统 SDK 包。

## 验证范围与 libc 限制

**本版只验证了 amd64/glibc。** ebuild 保留 `KEYWORDS="~amd64"` 和
`REQUIRED_USE` 中的 `elibc_glibc` 限制。其他架构未验证，musl 的适配、构建
及运行验证有待使用该环境的贡献者继续完成。

当前使用的 AOT 运行时包及随应用安装的本地库也需要针对 musl 检查；只移除
`elibc_glibc` 限制不能视为完成适配。完整的 pkgcheck 扫描仍会针对 amd64 的
musl profiles 给出 `RequiredUseDefaults` 和 `RequiredUseUnsatisfiableInDev`
两类提示，原因是这些 profiles 无法满足 glibc 要求。本版保留这些提示，并未
宣称通过 musl 验证。

## USE 配置与安装

| USE 配置 | 框架与运行行为 |
| --- | --- |
| `-wayland -kwin-iconfix` | Avalonia 11.3.22、AvaloniaEdit `ddf00a91`，使用 X11 后端 |
| `wayland -kwin-iconfix` | Avalonia 12.1.3、AvaloniaEdit `358f2c9e`；设置了 `WAYLAND_DISPLAY` 时使用原生 Wayland，否则使用 X11 |
| `wayland kwin-iconfix` | 使用上述 Wayland 构建，并通过启动器添加 KWin 图标兼容规则 |

`kwin-iconfix` 要求同时启用 `wayland`。两项功能都没有在 ebuild 中设置为
默认启用，但所选桌面 profile 仍可能启用 `wayland`。`debug` USE
用于保留本地调试信息。

启用包含本版本的 gentoo-zh overlay 后，可在
`/etc/portage/package.use/sourcegit` 中配置所需组合，例如：

```text
dev-vcs/sourcegit wayland kwin-iconfix
```

本版使用测试关键字 `~amd64`，如需接受该关键字，可在
`/etc/portage/package.accept_keywords/sourcegit` 中添加：

```text
=dev-vcs/sourcegit-2026.21-r1 ~amd64
```

然后安装指定版本：

```sh
emerge --ask =dev-vcs/sourcegit-2026.21-r1
```

安装后通过 `sourcegit` 命令或桌面应用菜单启动。

原生 Wayland 需要升级框架，因此 `wayland` USE 会应用一份基于发布标签的
Avalonia 12 迁移补丁，并选择对应的 AvaloniaEdit 源码归档。此迁移属于下游
修改；[SourceGit PR #2768](https://github.com/sourcegit-scm/sourcegit/pull/2768)
尚未被上游接受。补丁不包含后续 `develop` 分支的改动，也不包含单独开发的
标签拖放增强。两个 AvaloniaEdit 归档都固定到公开上游 fork 中的完整提交。

## NuGet 依赖与离线构建

两个框架版本共用 .NET 10.0.11 的 AOT 编译器和运行时包。NuGet 清单包含
26 个共用包、29 个 Avalonia 11 专用包和 34 个 Avalonia 12 专用包。
`SRC_URI` 根据 USE 选择需要的框架包及 AvaloniaEdit 归档，一次构建只产出
所选变体，不会同时安装两套 SourceGit。

恢复依赖时使用 ebuild 声明的 distfiles 和新的 NuGet 缓存，构建阶段无需
联网获取额外 NuGet。SDK 归档也通过 `SRC_URI` 和 `Manifest` 固定，构建
始终调用工作目录中的 `dotnet`。因此 SourceGit 不会通过系统 SDK 的
`PDEPEND` 拉入 .NET 6–10 的运行时 NuGet 包。

SDK 仍需要下载，归档大小为 239,732,427 字节，约 228.63 MiB。构建依赖
声明其需要的 ICU、OpenSSL、Kerberos 和 LTTng 库；SDK 不会安装到系统，
也不随 SourceGit 产物安装。此方案只作用于本包，不修改系统 SDK ebuild。
SDK 版本、归档校验值和应用 NuGet 清单都需要在后续升级时核验。

应用安装到 `/usr/lib64/sourcegit/`，包含一个 Native AOT 可执行文件和三个
私有本地库，同时安装命令入口、桌面条目和主题图标。应用内的更新检测已
关闭，后续版本由包管理器维护。

因为 Native AOT 产物是原生 ELF，而 `dotnet-pkg-base` 的 strip 限制用于
保护托管产物，所以这里直接继承 `nuget`，不设置 `RESTRICT=strip`。发布
阶段将 strip 留给 Portage，由用户的 strip 和调试配置决定最终处理方式。

## KWin 图标兼容规则

`kwin-iconfix` 是针对 KWin 的规则 workaround，用于让原生 Wayland 窗口
关联到 `sourcegit.desktop`，从而正确显示应用图标。

启用后，启动器以当前用户身份调用 helper。helper 只在 Wayland 会话且
会话总线上已有 `org.kde.KWin` 时生效；它为 `sourcegit` 创建一条固定 ID 的
窗口规则，保留其他规则，重复启动不会重复添加。root 和非 KWin 会话会跳过
此操作。

Portage 安装阶段不修改用户配置。该规则由启动应用的用户拥有，关闭 USE
或卸载软件后仍会保留。如需清理，应在移除 helper 前，以该用户身份运行：

```sh
/usr/libexec/sourcegit/sourcegit-kwin-iconfix --remove
```

此命令只清理 helper 创建的规则。此前手工创建且 ID 不同的 SourceGit 规则
会保留；本包不包含 1Password 的规则。

## 已完成的验证与后续检查

在 amd64/glibc 上，X11、Wayland、`wayland kwin-iconfix` 三种组合均使用
新的 NuGet 缓存完成了隔离 Portage 构建，检查到了安装镜像阶段。启动检查
覆盖了 X11 和 Wayland 打开仓库，以及 Wayland 打开 Blame。这些是短时间
启动检查，未覆盖长期性能和全部交互行为。

KWin helper 的回归检查使用真实的 KConfig 工具和隔离配置目录，覆盖
添加规则、重复运行、保留其他规则及移除自身规则。在 overlay 根目录运行：

```sh
python3 scripts/sourcegit-kwin-iconfix-test.py
```

完整依赖验证从 SDK 和 distfiles 的缓存核验开始。两个变体共有 93 个不同
输入，均通过 Manifest 的大小、BLAKE2B 和 SHA512 检查。X11 构建选择
58 个输入，Wayland 构建选择 63 个输入，每种组合都只包含一个 SDK 归档。

两种 USE 状态都完成了空安装根目录中的依赖预演，以及正常
`emerge --onlydeps --with-bdeps=y`。依赖计划均不包含系统 SDK 或
`dotnet-runtime-nugets` 包。USE 设置只作用于 SourceGit。

两种状态随后均使用正常源码 `emerge` 完成了构建及实际 merge，没有使用
`--nodeps` 或用户 phase hook。构建进程的可执行文件和 CLR 库均来自
`${WORKDIR}/dotnet-sdk`。验证时设置了 `SANDBOX_DENY`，禁止读取系统
SDK 目录和 `/opt/dotnet-nugets`，并启用了 `network-sandbox`。输入全部
命中 distfiles 缓存，每次恢复使用新的 NuGet 缓存，安装时由 Portage
执行 strip。依赖检查及两次 merge 均没有产生 `qa/warn/error` elog；
安装文件清单不包含 SDK。

| 已验证 USE 配置 | 原生可执行文件 | 启动后端 |
| --- | --- | --- |
| `-wayland -kwin-iconfix -debug` | 51.16 MiB，已 strip | X11 |
| `wayland kwin-iconfix -debug` | 56.56 MiB，已 strip | 原生 Wayland |

安装后的本地库依赖均能解析。两种安装版本都完成了 13 秒启动检查，没有
CLR 库加载或崩溃日志。这些检查未覆盖长期性能和全部交互行为；架构和
libc 验证范围仍限于 amd64/glibc。

Avalonia 11 和 12 的 DataGrid 均保留现有 `IL3053` AOT 分析警告，没有
通过全局关闭警告来隐藏。SourceGit 使用显式模板列，关闭了自动生成列及
用户排序。

版本跟踪已加入 `.github/workflows/overlay.toml`，未启用自动 bump。因为
框架迁移补丁、AvaloniaEdit 提交、SDK 和 NuGet 清单需要逐版核验，所以后续更新
由维护者人工完成。
