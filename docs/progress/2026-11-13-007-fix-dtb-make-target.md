# 修复 RPi 2B DTB 单目标构建路径重复

- 日期：2026-11-13
- 分支：`main`
- 修复前 workflow blob：`ccf7797c781bc7f3f7b640774515dcffada15f97`
- workflow 修复提交：`88d361bcf661600834059b32d84f7d2b275eb014`
- 失败 Actions Run：[43408488499](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43408488499)
- 失败 Job：`131828669284`

## 观察到的结果

本次 Actions 中，依赖安装、仓库检出、Linux 6.2.15 源码获取、配置解析和配置验证均成功。构建步骤失败，日志的直接错误为：

```text
make[1]: *** No rule to make target 'arch/arm/boot/dts/arch/arm/boot/dts/bcm2836-rpi-2-b.dtb'. Stop.
```

workflow 原先在 Linux 源码树顶层调用：

```sh
make ... arch/arm/boot/dts/bcm2836-rpi-2-b.dtb
```

Linux Kbuild 顶层 Makefile 对单个 `%.dtb` 目标会自动将目标转交给 `arch/arm/boot/dts` 子目录，并在转交时补上该目录前缀。因此传入已经带目录前缀的目标，会形成重复路径。

## 修改

将 make 命令中的目标改为 DTB 文件名本身：

```sh
make ... bcm2836-rpi-2-b.dtb
```

Kbuild 顶层规则负责转交到设备树子目录。产物检查和复制路径仍保持：

```text
arch/arm/boot/dts/bcm2836-rpi-2-b.dtb
```

没有修改 kernel fragment、内核配置、模块白名单、initramfs 逻辑或 Diretta ABI 审计规则。

## 当前结论

- 本次失败根因已定位为 DTB 单目标路径重复。
- workflow 已提交上述最小修复。
- 本次运行未完成 zImage、模块和 DTB 构建；ABI 审计步骤被跳过。
- Upload 步骤虽成功，但 artifact 仅约 58 KB，是失败时已存在的报告文件，不是完整内核包。
- 修复后的 workflow 尚未经过下一次 GitHub Actions 实际构建验证。
- 仍未达到内核构建成功、ABI 兼容、RPi 2B 实机启动或 Diretta 模块加载验证阶段。

## 下一步

重新运行 workflow，首先确认 DTB 目标不再报错，并等待完整构建步骤结束。若下一次失败，应以新日志中第一个真实错误为依据继续处理。
