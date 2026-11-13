# 2026-11-13：RPi 2B 专用产物与 initramfs 启动修复（下一版）

## 修改对象

- Repository: `mslovecc/eunhasu-diretta-abi-reference`
- Branch: `main`
- 修改后的 workflow commit: `07a9ee879bab07d1519afa94c013352df99d55e3`
- Workflow: [build-rpi2-diretta-6.2.15.yml](https://github.com/mslovecc/eunhasu-diretta-abi-reference/blob/main/.github/workflows/build-rpi2-diretta-6.2.15.yml)
- 上一轮基线构建：Run [43399919217](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217)，Job [131799937111](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43399919217/job/131799937111)

## 本轮修改

### 1. 限定 Raspberry Pi 2 Model B DTB

- 编译目标从整个 `dtbs` 改为单独的 `arch/arm/boot/dts/bcm2836-rpi-2-b.dtb`。
- 产物只放置两个同一 DTB 内容的兼容文件名：
  - `bcm2836-rpi-2-b.dtb`
  - `bcm2709-rpi-2-b.dtb`
- 不再复制其他 ARM 板卡 DTB，也不再复制整套 overlays。

### 2. 精简模块产物

- 保留 Raspberry Pi 平台相关模块、MMC、USB 网络、ALSA 核心、USB Audio、BCM 音频及相关驱动。
- 对保留模块递归读取 `modules.dep`，保留其完整依赖闭包后才删除其他模块。
- 强制检查以下目标模块存在：
  - `mmc_block.ko`
  - `snd-usb-audio.ko`
  - `smsc95xx.ko`
- 删除 `modules_install` 产生的 `build` 和 `source` 符号链接，避免 artifact 跟随链接打包整个 Linux 源码树。
- 删除后重新执行 `depmod`。

注意：模块筛选结果及数量必须以本轮 Actions 日志为准；本次尚未运行，不能预先声称已成功精简或确认最终模块数量。

### 3. 修复 initramfs 根分区启动路径

此前上传的 `initramfs7` 只有 BusyBox 和基础目录，没有模块目录；其 `/init` 只接受 `root=/dev/...`，而实机 cmdline 使用 `root=PARTUUID=fba41a53-02`。

本轮改为：

1. 将 `mmc_block.ko` 及 `modules.dep` 中的递归依赖复制进 initramfs。
2. 通过 BusyBox `insmod` 按依赖顺序加载 MMC 模块。
3. 在 `/sys/class/block/*/uevent` 中读取 `PARTUUID`，不依赖额外的 blkid applet。
4. 等待对应块设备出现，再挂载根分区。
5. 成功后将 `/dev`、`/proc`、`/sys` 移交给新根文件系统，并执行 `switch_root /newroot /sbin/init`。
6. 在 initramfs 启动阶段输出串口诊断信息；失败时停留在 shell 并显示明确错误。

### 4. 增加产物自检

- 检查 initramfs gzip 格式。
- 解出 CPIO 文件清单并检查 `init`、`load-mmc-modules.sh`、BusyBox 和 `mmc_block.ko` 确实存在。
- 检查 `/init` 包含 PARTUUID 处理逻辑。
- 检查最终 boot 目录 DTB 数量必须等于 2。

## 内核配置基线

本轮没有修改 `config/rpi2-diretta-6.2.15.fragment`。重新读取到的 blob SHA 仍为：

`7a5858a379e207ec07466a0ecb316f0f927779a2`

因此本轮变更集中在 workflow、initramfs 和产物整理，不改变已验证的内核配置片段。

## 当前验证状态

| 阶段 | 状态 |
|---|---|
| 已有配置验证 | 沿用上一轮通过结果；配置片段未修改 |
| 新 workflow 内容静态核对 | 已完成关键锚点检查 |
| 新版完整内核构建 | 尚未运行 |
| 新版 initramfs 内容自检 | 已加入 workflow，等待 Actions 执行 |
| 新版 DTB / 模块产物数量 | 等待 Actions 执行 |
| RPi 2B 实机启动 | 尚未验证 |
| Diretta 模块 ABI / 实际加载 | 尚未验证 |

本次 workflow 使用 `workflow_dispatch` 手动触发。当前可用 GitHub 操作未提供 workflow dispatch 调用，因此本轮已提交 workflow 修改，但没有伪称已触发 Actions。需要在 GitHub Actions 页面手动运行该 workflow，之后再根据实际日志修正构建问题。

## 下一步

1. 运行本 workflow，确认编译是否成功。
2. 核查 DTB 数量恰好为 2。
3. 核查保留模块清单、数量及依赖闭包，特别是 MMC、USB Audio、SMSC95xx。
4. 核查 initramfs 中确实包含 MMC 模块和依赖，并通过新增的 gzip/CPIO 内容检查。
5. 将新产物写入 RPi 2B，检查 TTL 是否依次出现 MMC 模块加载、PARTUUID 解析、根分区挂载和 switch_root 信息。
6. 实机启动成功后，再进行原始 Diretta 模块加载和功能验证。

## 重要限制

workflow 的修改和静态内容检查不等于构建成功。即使新 artifact 上传成功，也不能直接等同于 RPi 2B 启动成功或 Diretta ABI 兼容。
