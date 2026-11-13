# 2026-11-13 — 首次完整构建阶段核查

## 1. 本次目标

接续 `mslovecc/eunhasu-diretta-abi-reference` 项目，核实 ARMHF 软件源修复后的最新仓库状态、Actions 运行进度和当前构建阶段。目标平台为 Raspberry Pi 2 Model B，内核基线 Linux 6.2.15，目标是使原 A20 Eunhasu 的 `alsa_bridge.ko` 与 `diretta_direct.ko` 在 RPi 2 上加载并正常工作。

## 2. 仓库基线

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- 本次检查的最新提交：`0b7e30c158e5e37b633de6b8575e68b2deec1a1d`
- 提交说明：`Fix ARMHF BusyBox package source on Ubuntu 24.04 runners`
- 目标源码：Torvalds Linux `3c6c5ba197a9daf02c77ae39d65677c5e63e0845`，构建/验证工作流检查到的内核版本为 `6.2.15`。

## 3. 文件核查

已核查以下仓库文件：

- `.github/workflows/validate-rpi2-diretta-6.2.15.yml`
- `config/rpi2-diretta-6.2.15.fragment`
- 构建工作流：`.github/workflows/build-rpi2-diretta-6.2.15.yml`（通过最新 Actions 运行记录确认其路径）

配置 fragment 的 Blob SHA：
`7a5858a379e207ec07466a0ecb316f0f927779a2`

本次读取到的 fragment 仍包含 ARMv7 / AEABI / SMP / PREEMPT、NEON、关闭 MODVERSIONS、原模块 release 字符串、ALSA USB Audio、BCM2835、RPi USB 网络与存储等配置。没有提出新的配置修改；本轮遵循冻结已验证配置的原则。

## 4. Actions 运行记录

### 最新运行

- Workflow：Build RPi2 Diretta Linux 6.2.15 and audit module ABI
- Run ID：`43399919217`
- Job ID：`131799937111`
- Run：https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217
- 触发提交：`0b7e30c158e5e37b633de6b8575e68b2deec1a1d`
- 本次检查时状态：`in_progress`

本次检查时步骤状态：

| 步骤 | 状态 |
|---|---|
| Set up job | 成功 |
| Install ARM cross-build dependencies | 成功 |
| Checkout ABI reference and config | 成功 |
| Fetch pinned upstream Linux 6.2.15 | 成功 |
| Resolve and verify kernel configuration | 成功 |
| Build zImage, modules and device trees | 进行中 |
| Audit all original Diretta undefined symbols | 等待 |
| Upload kernel build and ABI audit artifacts | 等待 |

本次检查时尚无构建 artifacts。内核编译、符号审计和产物上传均未完成。

### 先前运行概况

- `43399655146`：失败
- `43399512886`：失败
- `43396467310`：失败
- `43396214043`：失败
- 配置验证运行 `43395830642`：成功

上述历史运行的具体失败原因应以各自完整日志为准；不能仅凭运行结果推断全部根因。

## 5. 四阶段验收状态

| 阶段 | 当前结论 | 证据/限制 |
|---|---|---|
| 配置验证 | **通过** | 最新构建运行中的 Resolve and verify kernel configuration 步骤成功；另有独立验证运行 `43395830642` 成功 |
| 内核完整构建 | **进行中，尚未通过** | zImage、modules、DTB 构建步骤尚未结束 |
| ABI 兼容 | **未验证** | 原模块 83 个唯一未定义符号的审计步骤尚未运行完成；符号存在也不等于完整 ABI 兼容 |
| RPi 2 实机测试 | **未测试** | 尚无目标设备启动、模块加载和音频传输结果 |

## 6. 当前判断

ARMHF 依赖安装步骤在最新运行中成功，源码获取和配置验证也成功，说明工作流已越过此前阻断构建的依赖安装/配置准备阶段，并已进入内核编译步骤。

这不代表内核已经构建成功，也不代表 Diretta 模块 ABI 兼容或已经在 Raspberry Pi 2 上运行。

## 7. 下一步

1. 等待 Run `43399919217` 完成，不修改已验证的 fragment。
2. 若构建失败，先定位失败步骤和首个有效错误，再决定是否只修复工作流。
3. 若构建成功，检查 zImage、RPi 2 DTB、模块目录、kernelrelease 和产物清单。
4. 再检查原始两个模块的架构、vermagic，以及 83 个唯一未定义符号在新内核中的导出情况。
5. ABI 静态审计通过后，才进入 Raspberry Pi 2 实机加载和功能测试。

## 8. 记录边界

本记录是截至本次 GitHub 检查时的状态快照。Actions 运行可能在记录提交后继续变化；后续结果应新增记录，不应回写或覆盖本记录。
