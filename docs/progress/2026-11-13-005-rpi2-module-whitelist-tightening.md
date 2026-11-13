# 2026-11-13：进一步收紧 RPi 2B 模块白名单

## 修改对象

- Repository: `mslovecc/eunhasu-diretta-abi-reference`
- Branch: `main`
- Workflow commit: `a7e626b1aaf7d13db6e11a05289d425dfdc7c007`
- Workflow: [build-rpi2-diretta-6.2.15.yml](https://github.com/mslovecc/eunhasu-diretta-abi-reference/blob/main/.github/workflows/build-rpi2-diretta-6.2.15.yml)
- 对应前序记录：[004 RPi 2B 专用产物与 initramfs 启动修复](2026-11-13-004-rpi2-initramfs-payload-fix.md)

## 本次收紧

复核上一版模块白名单后，发现保留整个 `drivers/mmc/`、`drivers/i2c/busses/`、`drivers/spi/`、`drivers/watchdog/` 目录会把其他平台的通用控制器模块也带进最终包，不符合“只保留 RPi 2B 需要的模块”的目标。

已将白名单收紧为：

- ALSA 核心、USB Audio 全部模块，保留 USB 声卡通用驱动及厂商设备 quirks 支持。
- BCM 音频 SoC 模块。
- Raspberry Pi firmware、VC4/V3D 图形模块。
- 文件名匹配 BCM2835/BCM2836/BCM2708/BCM2709、BCM pinctrl、VC4/V3D 的平台模块。
- 精确指定 RPi 相关 I2C、SPI、RNG、watchdog 模块（如果它们以模块形式构建）。
- 必须保留 `mmc_block.ko`、`snd-usb-audio.ko`、`smsc95xx.ko`。
- 最后递归保留以上模块在 `modules.dep` 中的所有依赖。

这样不再按整个通用 I2C、SPI、MMC host 或 watchdog 目录保留模块。依赖闭包仍然可能包含通用内核模块，这是运行目标驱动所必需的，不应误删。

## 配置与验证边界

- 内核 config fragment 未修改。
- 本次只修改最终模块打包白名单。
- 这只是 workflow 静态修改；必须等 Actions 执行后，才能确认目标模块数量、依赖闭包和产物体积。
- 尚未进行 RPi 2B 实机启动或 Diretta 模块加载测试。
