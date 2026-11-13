# 2026-11-13-011：定位 initramfs 阶段的未标记退出

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- Actions Run：[`43419373184`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43419373184)
- Job：`131865225335`
- 检查日期：2026-11-13

## 已观察事实

本次运行中，配置验证通过，Linux 6.2.15 内核、RPi2 DTB 和模块构建完成。模块裁剪从 796 个模块中保留 25 个；必需的 `mmc_block.ko` 已位于 `kernel/drivers/mmc/core/mmc_block.ko`。ARMHF BusyBox 已通过前置架构检查并进入 initramfs 构建流程。MMC 模块依赖收集脚本正常输出：

```
Initramfs MMC module load order:
  kernel/drivers/mmc/core/mmc_block.ko
```

随后 Build step 以 exit code 1 结束。ABI audit 和 artifact upload 均 skipped；没有产物发布。

## 当前诊断结论

现有日志没有暴露导致退出的具体 shell 命令：该 step 启用了 `set -Eeuo pipefail`，但没有 ERR trap；若后续 `cat`、`cpio`、`gzip`、归档清单检查或 `grep` 验证失败，日志只显示 step exit code 1。因此，不能仅凭当前日志把原因断定为 cpio、gzip 或某一条清单检查。

## 本次 workflow 诊断改动

在 Build step 增加 ERR trap，失败时输出实际行号、退出码和触发失败的命令；并在 init 脚本创建、cpio 归档创建、gzip 校验通过后分别输出阶段标记。未修改 kernel fragment、Kconfig、内核源码 commit、模块白名单、initramfs 启动逻辑或 Diretta ABI 审计规则。

## 验证状态

- 本次 Run：内核/模块/DTB 构建通过；initramfs 阶段失败。
- 新增诊断 trap：已提交，尚待新的 Actions run 验证。
- Diretta undefined-symbol 审计：本次未运行。
- RPi2 实机启动、两个 Diretta 模块加载及功能测试：尚未验证。
