# Project Progress Records

本目录保存 Raspberry Pi 2 Model B / Linux 6.2.15 / A20 Eunhasu Diretta ABI 兼容项目的阶段性分析记录。

## 记录规则

- 每次完成一轮实质性分析、故障排查、代码修改或 Actions 验证后，新增一份 Markdown 记录。
- 文件名使用 `YYYY-MM-DD-NNN-short-topic.md`，不覆盖历史记录。
- 记录必须注明检查时的仓库分支、Commit SHA、Actions Run/Job ID 和证据链接（适用时）。
- 严格区分：配置验证、内核构建、ABI 兼容、树莓派实机测试。
- 只记录已观察到的事实；推断、未验证事项和下一步计划必须明确标注。
- 如果 Actions 仍在运行，记录为“进行中”，不能提前写成成功。
- 记录内容以中文为主，保留必要的英文文件名、符号名和日志原文。

## 项目目标

使原 A20 Eunhasu 系统的 `alsa_bridge.ko` 和 `diretta_direct.ko` 能在 Raspberry Pi 2 Model B 上加载并正常工作。

## 记录索引

| 日期 | 记录 | 阶段 |
|---|---|---|
| 2026-11-13 | [首次完整构建阶段核查](2026-11-13-001-initial-build-check.md) | 配置验证通过；内核构建进行中 |
| 2026-11-13 | [项目目标、资料基线与当前进度复述](2026-11-13-002-goal-and-current-status.md) | 配置验证通过；内核构建进行中（当时状态） |
| 2026-11-13 | [首次完整内核构建成功与 Diretta 符号审计](2026-11-13-003-build-success-symbol-audit.md) | 内核构建通过；57/58 符号已导出；1 项待核实 |
| 2026-11-13 | [RPi 2B 专用产物与 initramfs 启动修复（下一版）](2026-11-13-004-rpi2-initramfs-payload-fix.md) | workflow 已修改；等待 Actions 构建与实机验证 |
| 2026-11-13 | [进一步收紧 RPi 2B 模块白名单](2026-11-13-005-rpi2-module-whitelist-tightening.md) | 模块白名单已收紧；等待 Actions 构建验证 |
| 2026-11-13 | [修复新版 workflow 的 YAML 缩进错误](2026-11-13-006-fix-workflow-yaml-indentation.md) | 已修复并静态复核；等待 GitHub Actions 加载验证 |
| 2026-11-13 | [修复 RPi 2B DTB 单目标构建路径重复](2026-11-13-007-fix-dtb-make-target.md) | workflow 已修复；等待 Actions 构建验证 |
| 2026-11-13 | [定位 MMC 模块路径检查错误与异常膨胀的构建产物](2026-11-13-008-fix-mmc-module-path-diagnosis.md) | 内核与模块已编译；模块路径检查失败；initramfs 与 Diretta 审计未完成 |
| 2026-11-13 | [修正 MMC 模块路径并禁止失败构建上传产物](2026-11-13-009-mmc-path-and-success-only-artifacts.md) | Workflow 已提交；等待新的 Actions 构建验证 |
