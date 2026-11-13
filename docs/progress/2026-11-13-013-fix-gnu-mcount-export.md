# 013：修复 `__gnu_mcount_nc` 未导出

## 本次核查对象

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- Actions Run：[`43425338998`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43425338998)
- Job：[`131885274147`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43425338998/job/131885274147)
- Artifact：[`rpi2-diretta-linux-6.2.15-build`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43425338998/artifacts/14037584175)
- Artifact ID：`14037584175`
- Artifact 大小：13,409,691 bytes
- 本次构建所用上游 Linux commit：`3c6c5ba197a9daf02c77ae39d65677c5e63e0845`

## Run 43425338998 结果

本次完整通过以下步骤：

1. 配置合并与校验。
2. RPi2 内核、模块、DTB 构建。
3. 模块裁剪：796 个模块中保留 25 个。
4. initramfs 生成、gzip 校验及内容清单校验。
5. Diretta undefined-symbol 审计。
6. Artifact 上传成功。

产物检查结果：

- `kernel7.img`：10,412,544 bytes
- `initramfs7`：906,786 bytes
- 两个 RPi2 DTB：各 14,500 bytes
- 25 个选定模块
- Artifact ZIP：约 13 MB，未再出现此前数百 MB 的 source/build symlink 膨胀。

## ABI 审计结论

原始模块 undefined symbols：

- `alsa_bridge.ko`：46
- `diretta_direct.ko`：37
- 合并去重：58

本次目标内核：

- 已导出：57/58
- 未导出：1
- 唯一缺失：`__gnu_mcount_nc`

## 根因已由源码确认

目标配置中原本有 `CONFIG_FTRACE=y`，但没有启用 `CONFIG_FUNCTION_TRACER`。

在本项目固定的 Linux 源码 commit `3c6c5ba197a9daf02c77ae39d65677c5e63e0845` 中，`arch/arm/kernel/armksyms.c` 明确写有：

```c
#ifdef CONFIG_FUNCTION_TRACER
EXPORT_SYMBOL(__gnu_mcount_nc);
#endif
```

因此，仅有 `CONFIG_FTRACE=y` 不足以导出该符号；必须启用 `CONFIG_FUNCTION_TRACER=y`。

交叉比对原始 A20 Eunhasu 内核配置 `a20-original/kernel/config-6.2.15-215.fc36.armv7hl`，其中本来就启用了：

- `CONFIG_FTRACE=y`
- `CONFIG_FUNCTION_TRACER=y`
- `CONFIG_DYNAMIC_FTRACE=y)

这与原始 Diretta 模块对 `__gnu_mcount_nc` 的引用相吻合。此次缺失不是普通模块白名单问题，也不是 initramfs 问题，而是目标内核配置没有打开 ARM function tracer 导出路径。

## 已提交的修复

### Config fragment

文件：`config/rpi2-diretta-6.2.15.fragment`

在模块 ABI 配置区增加：

```text
# The original A20 Diretta modules reference __gnu_mcount_nc.
# ARM exports this symbol only when CONFIG_FUNCTION_TRACER is enabled.
CONFIG_FUNCTION_TRACER=y
```

### Workflow validation

文件：`.github/workflows/build-rpi2-diretta-6.2.15.yml`

将 `FUNCTION_TRACER=y` 加入必需配置校验，避免后续 Kconfig 合并时静默丢失。

本轮提交：

- Config fragment commit：`8cea1cb87b67b1fb394acb006f1a2fcae93121e8`
- Workflow commit：`8f318b0a0e19de5e6fa38587fb0dff47b08a612f`

## 尚未验证

本次修改尚未经过新的 GitHub Actions 构建。预期下一次审计应显示：

- `CONFIG_FUNCTION_TRACER=y`
- `__gnu_mcount_nc` 出现在目标内核 `Module.symvers`
- 原始 58 个 undefined symbols 全部有对应导出（58/58）

即使静态审计达到 58/58，也仍不等于两个专有模块已经在真实 Raspberry Pi 2 上成功加载。后续仍需实机验证启动、模块加载及 Diretta 功能。
