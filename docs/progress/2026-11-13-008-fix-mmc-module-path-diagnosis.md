# 2026-11-13：定位 MMC 模块路径检查错误与异常膨胀的构建产物

## 检查范围

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- Actions Run：[`43412725023`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43412725023)
- Job：[`131842889846`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43412725023/job/131842889846)
- 本次运行对应提交：`1542b797b8e516455400db2a368c7dd2ef6b8828`
- Workflow：`.github/workflows/build-rpi2-diretta-6.2.15.yml`
- Workflow 当前文件 Blob：`2d4e8347d7444ce8ced32ba2ca0019648ee704e9`

## 结果摘要

配置解析与验证通过；Linux 6.2.15 的 zImage、所选模块及 RPi2B DTB 的编译步骤已经执行到模块安装和打包前检查阶段。构建步骤最终因脚本使用了错误的 MMC block 模块路径而失败。Diretta undefined-symbol 审计被跳过，initramfs 生成与最终启动包验证没有完成。

## 已确认的直接失败原因

日志中可见：

- 配置合并阶段：`CONFIG_MMC_BLOCK` 从基础配置的 `y` 被 fragment 改为 `m`；最终配置验证通过。
- 编译阶段：`LD [M] drivers/mmc/core/mmc_block.ko`（约 11:13:01）。
- 模块安装阶段：`INSTALL .../kernel/drivers/mmc/core/mmc_block.ko`（约 11:13:04）。
- 随后的模块白名单检查报错（11:13:05）：
  `Required RPi2 boot/audio modules missing: kernel/drivers/mmc/card/mmc_block.ko`

Linux 6.2.15 本次实际生成并安装的路径是：

`kernel/drivers/mmc/core/mmc_block.ko`

Workflow 的模块检查和 initramfs 复制逻辑却都硬编码成：

`kernel/drivers/mmc/card/mmc_block.ko`

因此脚本把已经成功编译并安装的模块误判为缺失。这是路径检查错误，不是 `CONFIG_MMC_BLOCK` 未启用，也不是 MMC 驱动编译失败。

涉及 workflow 中至少两处路径引用：
1. Python 模块白名单 `required` 集合。
2. Python initramfs 模块复制步骤的 `target` 路径。
此外，后续 initramfs 内容校验的 grep 路径也使用了旧目录，修正时必须同步检查。

## 产物体积异常（次要问题）

尽管构建步骤失败，`if: always()` 的上传步骤仍成功上传诊断 artifact：

- 名称：`rpi2-diretta-linux-6.2.15-build`
- Artifact ID：`14032160507`
- 大小：`847,862,670 bytes`
- SHA256：`5915729d1ceaf7719cdf4e308edd7a90f73ff913c39cf8eb19f8ecd1c823a7e5`
- [Artifact 下载页](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43412725023/artifacts/14032160507)

上传日志报告 `198910 files uploaded`，并出现 `.../build/include/...` 与 `.../source/include/...` 等路径重复警告。Workflow 的 `rm -f "$MODROOT/build" "$MODROOT/source"` 清理操作位于失败的模块路径检查之后，因此本次在清理执行前退出，上传步骤把模块目录中的 build/source 符号链接目标也纳入归档，导致 artifact 异常膨胀。这不是本次构建失败的首要原因，但应在后续修复中把清理时机提前到 artifact 上传之前，并验证链接没有被打包跟随。

## 阶段判定

| 阶段 | 本次结果 |
|---|---|
| 配置解析与验证 | 通过 |
| zImage / 模块 / RPi2B DTB 编译 | 已完成；日志显示 MMC block 模块也已链接 |
| 模块安装 | 已完成；MMC 模块实际位于 `drivers/mmc/core/` |
| 模块白名单与依赖闭包检查 | 因 MMC 模块路径写错而失败 |
| initramfs 生成及内容校验 | 未执行完成 |
| Diretta 静态 undefined-symbol 审计 | 跳过 |
| 最终产物打包 | 诊断 artifact 上传成功，但内容异常膨胀且不是完整可用发布包 |
| RPi2B 实机启动 / 两个 Diretta 模块加载 / 功能测试 | 尚未进行 |

## 下一步建议（尚未执行）

1. 将 workflow 中所有 `mmc_block.ko` 路径统一改为 Linux 6.2.15 实际路径 `kernel/drivers/mmc/core/mmc_block.ko`，包括 required 集合、initramfs 复制目标及 initramfs 清单检查。
2. 将 `build` / `source` 符号链接清理放到任何可能失败的检查之前，并在上传前检查模块树内不存在这两个链接。
3. 重新运行后，依次核对 initramfs 内容、Diretta 静态符号审计和 artifact 大小。
4. 只有构建与静态审计完成后，才进入真实 RPi2B 启动、`alsa_bridge.ko` 与 `diretta_direct.ko` 加载及 Diretta 功能验证。

本记录只做故障分析，没有修改 workflow、内核配置或触发新的 Actions 运行。
