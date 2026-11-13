# 2026-11-13-012：修复 initramfs CPIO 清单路径格式校验

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- 修复前 Actions Run：[`43422922296`](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43422922296)
- Job：`131877092903`
- 修复提交：`dc762a89659b3c6ad238410dd9ffd95461ebf84a`
- 检查日期：2026-11-13

## 本次运行的实际失败点

新增的 ERR trap 已准确报告：

```
BUILD_STEP_FAILED line=312 exit=1 command=grep -Fxq "./init" "$INITRD_LIST"
```

日志同时确认以下步骤已完成：

- `INITRAMFS_INIT_SCRIPT_CREATED`
- `INITRAMFS_ARCHIVE_CREATED`
- `INITRAMFS_GZIP_VALIDATED`

因此本次失败发生在 CPIO 文件清单校验，不是内核、模块、DTB、BusyBox 架构检查、MMC 依赖收集或 gzip 完整性检查。ABI audit 与 artifact upload 均 skipped。

## 根因

归档由 GNU cpio 从 `find .` 的输出创建；校验逻辑却把清单中的路径形式固定假设为带前缀的 `./init`。本次日志证明清单中没有该精确字符串。GNU cpio 的列表输出可能省略前缀 `./`，因此校验器不应依赖此前缀。

## 修复

在生成清单时统一执行：

```bash
sed 's#^\./##'
```

将清单路径规范化为无前缀形式，再精确检查：

- `init`
- `load-mmc-modules.sh`
- `bin/busybox`
- `lib/modules/$KREL/kernel/drivers/mmc/core/mmc_block.ko`

本次只修改 workflow 的归档清单校验；没有修改内核配置、kernel commit、模块白名单、initramfs 启动逻辑或 ABI 审计规则。

## 当前状态与下一步

- 修复已提交到 `main`。
- 尚未通过新的 Actions run 验证。
- 下一次应确认 initramfs 清单校验通过，继续执行 Diretta undefined-symbol 审计。
- 即使静态符号审计通过，也仍需在真实 RPi2 上验证启动、模块加载和 Diretta 功能。
