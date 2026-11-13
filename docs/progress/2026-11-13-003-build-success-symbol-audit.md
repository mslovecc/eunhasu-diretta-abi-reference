# 2026-11-13：首次完整内核构建成功与 Diretta 符号审计

## 检查对象

- Repository: `mslovecc/eunhasu-diretta-abi-reference`
- Branch: `main`
- Workflow Run: [43399919217](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217)
- Job: [131799937111](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217/job/131799937111)
- Build commit: `0b7e30c158e5e37b633de6b8575e68b2deec1a1d`
- Upstream Linux source commit: `3c6c5ba197a9daf02c77ae39d65677c5e63e0845`
- Config fragment blob: `7a5858a379e207ec07466a0ecb316f0f927779a2`

## Actions 结果

Run 和唯一 build Job 均为 `completed / success`。以下步骤全部成功：

1. 安装 ARM 交叉编译依赖
2. Checkout ABI reference 和配置
3. 获取固定版本 Linux 6.2.15 源码
4. 合并并验证内核配置
5. 编译 zImage、modules 和 DTB
6. 执行原始 Diretta undefined-symbol 审计
7. 上传构建产物

内核构建步骤成功，说明本次配置能够在 GitHub Ubuntu 24.04 runner 上完成 Linux 6.2.15 的 ARM 交叉编译；这不是 Raspberry Pi 2 实机启动测试。

## Diretta undefined-symbol 静态审计

审计输入文件统计：

- `alsa_bridge.undefined-symbols.txt`：46 个 undefined-symbol 条目
- `diretta_direct.undefined-symbols.txt`：37 个 undefined-symbol 条目
- 两个文件合并去重后：58 个唯一符号（不是 83 个唯一符号；83 是两文件条目数相加）

针对本次构建生成的 `Module.symvers`：

- 已导出：57 / 58
- 未导出：1 / 58
- 唯一未导出符号：`__gnu_mcount_nc`

注意：工作流将缺失符号作为报告项，而非失败条件，因此符号审计步骤仍然显示 success。不能把该步骤 success 理解为 58 个符号全部匹配，也不能据此宣布模块 ABI 兼容。

`__gnu_mcount_nc` 的性质和模块加载器对该 ARM 符号的处理仍需进一步核实。当前不能直接断言它一定导致加载失败，也不能忽略它。

## 构建产物

Artifact：

- 名称：`rpi2-diretta-linux-6.2.15-build`
- Artifact ID：`14026082639`
- 下载页：[GitHub Actions artifact](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217/artifacts/14026082639)
- 大小：896,093,234 bytes
- SHA-256：`c55899512c2db01688c3fba7d2b7cab4a3370acb897786bc7faf644fade13e42`
- 过期时间：2027-02-11

本次尝试通过 GitHub artifact 下载接口读取 ZIP 时，接口拒绝了大于 536,870,912 bytes 的文件。因此，本轮已确认 artifact 成功上传，但尚未在本地解包检查其中的具体文件。上传日志还显示模块目录中的 `build` / `source` 路径导致约 205,035 个文件被纳入上传，且出现同路径覆盖提示；后续可考虑修正打包方式，避免将内核源码树经符号链接重复打包。

## 阶段状态

| 阶段 | 状态 | 说明 |
|---|---|---|
| 配置验证 | 已通过 | workflow 的 config 验证步骤成功 |
| 完整内核构建 | 已通过 | zImage、modules、DTB 构建步骤成功 |
| 符号静态审计 | 部分通过 / 有 1 项待核实 | 57 个已导出，`__gnu_mcount_nc` 未出现在目标 Module.symvers |
| 模块 ABI 兼容 | 未确认 | 尚未对原始 .ko 执行目标内核上的实际装载验证 |
| RPi 2 实机启动 | 未测试 | 尚无实机结果 |
| Diretta 功能 | 未测试 | 尚无实际音频/网络功能结果 |

## 下一步

1. 获取并审阅 `diretta-symbol-audit.txt` 完整内容（本轮日志已确认统计和唯一缺失符号）。
2. 针对 `__gnu_mcount_nc`，检查 ARM 目标内核的 mcount/ftrace 实现、导出策略和模块重定位处理，确定是否为可接受的特殊符号或真实缺失依赖。
3. 以更精简的 artifact 打包配置重新组织产物，避免通过 `build` / `source` 符号链接重复上传源码树；不因此改变已验证的 kernel config。
4. 检查 kernel release、zImage、Pi 2 DTB、initramfs、模块目录和 Module.symvers。
5. 只有静态证据充分后，才进入 Raspberry Pi 2 实机模块加载测试。

## 重要限制

本次成功证明的是：指定 Linux 6.2.15 源码、当前 ARM 配置和构建脚本可以完成交叉编译。它没有证明原始 `alsa_bridge.ko` 与 `diretta_direct.ko` 已能加载，更没有证明 Diretta 已正常工作。
