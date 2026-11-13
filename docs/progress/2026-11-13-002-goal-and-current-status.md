# 2026-11-13-002：项目目标、资料基线与当前进度复述

## 1. 本次项目目标

继续项目 `mslovecc/eunhasu-diretta-abi-reference`，目标是在 Raspberry Pi 2 Model B（BCM2836，32-bit ARMv7/EABI）上，使原 A20 Eunhasu 系统的两个专有 Diretta 内核模块：

- `alsa_bridge.ko`
- `diretta_direct.ko`

能够成功加载并正常工作。

目标内核基线为 Linux 6.2.15。原系统内核 release 标识为 `6.2.15-215.fc36.armv7hl`。本项目的目标不是只编译一个可启动内核，而是尽可能满足原始模块所依赖的 ABI 条件，并最终在 Raspberry Pi 2 实机验证模块加载和功能。

## 2. 已掌握资料与基线

### 2.1 目标平台和内核

- 目标板：Raspberry Pi 2 Model B / BCM2836。
- 架构：32 位 ARM、ARMv7、EABI。
- 内核版本：Linux 6.2.15。
- 目标内核 release 字符串：`6.2.15-215.fc36.armv7hl`。
- 工作流固定的上游 Linux 源码 commit：`3c6c5ba197a9daf02c77ae39d65677c5e63e0845`。

### 2.2 配置基线

配置方案为上游 `multi_v7_defconfig` 加项目配置片段：

`config/rpi2-diretta-6.2.15.fragment`

已知配置要点包括 ARMv7、AEABI、SMP、PREEMPT、NEON、内核模块支持，以及关闭 `CONFIG_MODVERSIONS`。配置保留 ALSA/USB Audio、树莓派相关硬件和启动存储所需功能，并关闭一批非目标平台或高成本驱动。

此前配置验证结果为 98 项 fragment 设置匹配。本次重新读取 GitHub 文件，配置片段 SHA 为：

`7a5858a379e207ec07466a0ecb316f0f927779a2`

本次未修改配置片段。不得在没有构建日志或 ABI 证据的情况下重新设计、随意增删已验证配置。

### 2.3 原始模块与 ABI 资料

- 原始模块文件名：`alsa_bridge.ko`、`diretta_direct.ko`。
- 之前的静态分析记录指出，两模块合计涉及 83 个唯一未定义符号。
- 原始内核配置已知 `CONFIG_MODVERSIONS` 未启用。
- 后续需要使用目标构建生成的 `Module.symvers`、`System.map`、最终配置及模块文件，核查符号导出和 ABI 条件。

注意：符号名称存在不代表模块必然可加载；vermagic 相同也不代表二进制 ABI 兼容；静态符号审计通过不代表模块功能正常。

## 3. 本次重新核查的 GitHub 状态

仓库：`mslovecc/eunhasu-diretta-abi-reference`

- 默认分支：`main`。
- 本次读取到的 main 分支 HEAD：`97e18f2c82c4213fe5455ffcdc8025bfe0e1b46d`。
- 最新已知工作流修复提交：`0b7e30c158e5e37b633de6b8575e68b2deec1a1d`，主题为修复 Ubuntu 24.04 Runner 的 ARMHF BusyBox 软件包来源。
- 构建工作流：`.github/workflows/build-rpi2-diretta-6.2.15.yml`。
- 配置验证工作流：`.github/workflows/validate-rpi2-diretta-6.2.15.yml`。
- 配置片段：`config/rpi2-diretta-6.2.15.fragment`。

当前构建工作流将 Ubuntu 原生软件源限制为 amd64，并为 armhf 添加 Ubuntu Ports 源 `ports.ubuntu.com/ubuntu-ports`，随后下载并解包 `busybox-static:armhf`。工作流还固定 GCC 12 ARMHF 交叉编译器，并检查目标 kernelrelease、zImage、RPi2 DTB、Module.symvers、System.map、安装后的模块树和 initramfs。

## 4. 最新 Actions 运行状态

- Actions Run：[#43399919217](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217)
- Job ID：`131799937111`
- 本次核查时 Job 状态：`in_progress`。
- 依赖安装：成功。
- 仓库与配置 checkout：成功。
- 固定版本 Linux 6.2.15 源码获取：成功。
- 配置解析与验证：成功。
- `Build zImage, modules and device trees`：仍在运行。
- `Audit all original Diretta undefined symbols`：尚未开始。
- `Upload kernel build and ABI audit artifacts`：尚未开始。
- 本次核查时该 Run 尚无可下载 artifacts。

因此，本次不能记录为完整内核构建成功。当前状态只是依赖、源码获取和配置验证步骤成功，实际编译尚未确认完成。

## 5. 阶段状态（严格区分）

| 阶段 | 状态 | 说明 |
|---|---|---|
| 配置验证 | 已通过 | 此前 98 项设置匹配；本次 Actions 的配置验证步骤也成功 |
| 内核完整构建 | 进行中 / 未确认 | 编译步骤仍在运行，尚无产物可核验 |
| Diretta ABI 兼容 | 未验证 | 尚未取得本次构建的完整符号审计结果，也未验证模块加载 |
| Raspberry Pi 2 实机测试 | 未开始 | 尚无实机启动、模块加载及功能测试证据 |

## 6. 下一步

1. 继续检查 Run `43399919217` 的状态和编译步骤结果，定位第一条真实失败信息（如果失败）。
2. 如果构建成功，下载并检查 artifact，确认 kernel image、RPi2 DTB、initramfs、完整模块树、kernelrelease、`Module.symvers`、`System.map` 和最终配置。
3. 检查原始 83 个唯一未定义符号的静态审计结果；逐项区分已导出和未导出符号。
4. 在静态分析基础上再制定实机测试步骤，分别验证启动、模块加载和 Diretta 功能。
5. 除非日志或 ABI 证据明确要求，否则不修改已经验证的配置片段。

## 7. 本次修改

本次只新增本进度记录并更新进度索引；没有修改内核配置、构建工作流，也没有触发新的 Actions 运行。
