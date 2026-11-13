# 2026-11-13：修正 MMC 模块路径并禁止失败构建上传产物

## 修改范围

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- 修改文件：`.github/workflows/build-rpi2-diretta-6.2.15.yml`
- Workflow 修改提交：`a20c426e601c8b9b0efd6ddeb78d99c788dbaf48`
- 新 Workflow Blob：`59d8ecfdfbf7c002afe5d0b9449acfa6c15111fa`
- 前置故障记录：[008](2026-11-13-008-fix-mmc-module-path-diagnosis.md)

## 修改内容

### 1. 修正 MMC block 模块路径

将 Workflow 中全部三处：

`kernel/drivers/mmc/card/mmc_block.ko`

改为 Linux 6.2.15 本次编译实际生成的：

`kernel/drivers/mmc/core/mmc_block.ko`

修改覆盖：
- RPi2 模块白名单必需模块检查。
- initramfs 的 MMC 模块及依赖复制入口。
- initramfs CPIO 内容清单检查。

### 2. 提前清理 modules_install 创建的符号链接

将：

`rm -f "$MODROOT/build" "$MODROOT/source"`

移动到 `modules_install` 后、后续模块检查和 `depmod` 之前。这样即使后续白名单检查或 initramfs 生成失败，模块树也不会保留指向完整内核源码树的 `build` / `source` 链接，避免失败产物被意外归档时膨胀。

模块裁剪后的第二次 `depmod` 保留，重复的延迟清理命令已移除。

### 3. 失败时不上传任何构建产物

将上传步骤条件从：

`if: always()`

改为：

`if: success()`

因此只要此前任意构建或 ABI 审计步骤失败，上传步骤就会被跳过；不会上传部分内核、诊断文件或失败构建产物。只有构建和静态符号审计步骤全部成功后才上传 artifact。

## 修改后静态复核

重新读取 GitHub 上提交后的 Workflow，确认：
- `mmc_block.ko` 的三处引用全部指向 `drivers/mmc/core/`，没有残留 `drivers/mmc/card/`。
- `build` / `source` 清理命令位于首次 `depmod` 之前。
- Artifact 上传步骤明确使用 `if: success()`。

## 当前验证状态与限制

本次完成的是 Workflow 文件修改和提交后的内容复核，尚未有新的 Actions 构建结果。当前可用的 GitHub 操作没有提供手动触发 `workflow_dispatch` 的接口；不能用旧 Run 的 rerun 代替，因为旧 Run 会使用旧提交，无法验证本次新 Workflow。因此本记录不声称新构建已触发或已通过。

## 下一步

在 GitHub 仓库 Actions 页面手动运行 `Build RPi2 Diretta Linux 6.2.15 and audit module ABI`。检查：
1. MMC 模块白名单能否通过。
2. initramfs 是否包含 `drivers/mmc/core/mmc_block.ko` 及依赖，PARTUUID 启动逻辑校验是否通过。
3. Diretta 静态 undefined-symbol 审计是否执行并生成报告。
4. 成功时 artifact 体积是否恢复正常；失败时 Upload 步骤必须显示 skipped，且 Run 不应产生 artifact。
5. 即使静态符号审计完成，也仍需在真实 RPi2B 上验证启动、两个 Diretta 模块加载及实际功能。

本次没有修改内核配置、模块白名单策略以外的驱动选择逻辑，也没有触发新的 Actions 运行。
