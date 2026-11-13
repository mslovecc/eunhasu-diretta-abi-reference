# 2026-11-13-010 — 修复 ARMHF BusyBox 在 x86_64 Runner 上被直接执行

## 检查范围

- 仓库：`mslovecc/eunhasu-diretta-abi-reference`
- 分支：`main`
- 失败运行：[Actions Run 43416505146](https://github.com/mslovecc/eunhasu-diretta-abi-reference/actions/runs/43416505146)
- Job：`131855647606`
- 运行结论：build failed；ABI audit skipped；artifact upload skipped
- 本轮修改提交：`ae766ca5d6e629b110aa54e24f13d57c6f230265`

## 已确认事实

1. 配置验证通过。
2. Linux 6.2.15 内核、模块和 RPi2 DTB 构建完成；模块安装完成。
3. 模块裁剪成功：796 个模块中保留 25 个，移除 771 个。
4. MMC 模块路径已正确使用 `kernel/drivers/mmc/core/mmc_block.ko`。
5. initramfs 制作阶段失败，原始错误：
   ```
   /.../busybox-armhf/extracted/usr/bin/busybox: cannot execute binary file: Exec format error
   Required BusyBox applet unavailable: sh
   ```
6. 日志显示下载的是 Ubuntu `busybox-static:armhf`（版本 `1:1.36.1-6ubuntu3.1`），但构建脚本在 x86_64 Ubuntu Runner 上直接执行该 ARMHF 程序的 `--list`。因此失败发生在宿主机执行架构不匹配，不是 ARMHF BusyBox 下载失败，也不是内核构建失败。
7. ABI 符号审计未执行；失败运行没有上传 artifact，符合“失败不上传产物”的要求。

## 本轮修改

Workflow：`.github/workflows/build-rpi2-diretta-6.2.15.yml`

- 增加安装宿主机原生 `busybox-static:amd64`，只用它检查所需 applet 名称。
- 仍将下载的 ARMHF 静态 BusyBox 放入目标 initramfs。
- 使用 `readelf -h` 检查目标 BusyBox 的 ELF Machine 为 ARM，不在 x86_64 Runner 上执行目标 ARM 程序。
- 保持内核配置 fragment、内核源码 commit、模块白名单和 Diretta 审计规则不变。
- 失败构建仍不会上传产物（上传步骤为 `if: success()`）。

## 验证状态

- GitHub 文件更新已提交：`ae766ca5d6e629b110aa54e24f13d57c6f230265`。
- 该提交只证明 workflow 文件已写入仓库；尚未由新的 Actions 运行验证。
- 需要下一次 Actions 确认：BusyBox 检查通过、initramfs 打包与内容检查通过、Diretta undefined-symbol 审计执行完成。
- 即使静态符号审计完成，也不代表模块已在真实 Raspberry Pi 2 Model B 上成功加载或 Diretta 功能已验证。
