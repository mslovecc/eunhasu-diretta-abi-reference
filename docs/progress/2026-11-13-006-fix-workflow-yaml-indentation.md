# 2026-11-13：修复新版 workflow 的 YAML 缩进错误

## 问题

用户反馈提交的 `.github/workflows/build-rpi2-diretta-6.2.15.yml` 无法解析。重新读取 GitHub 当前文件后，确认 Build 步骤结束后的下一步骤缩进错误：

错误：
```yaml
                - name: Audit all original Diretta undefined symbols
```

正确：
```yaml
      - name: Audit all original Diretta undefined symbols
```

该步骤应与同一 `jobs.build.steps` 下的其他步骤保持 6 个空格缩进。

## 修复提交

- Repository: `mslovecc/eunhasu-diretta-abi-reference`
- Branch: `main`
- 修复 commit: `a165f5b1271e2424233330132bca3b93af4dcc18`
- Workflow 文件：[build-rpi2-diretta-6.2.15.yml](https://github.com/mslovecc/eunhasu-diretta-abi-reference/blob/main/.github/workflows/build-rpi2-diretta-6.2.15.yml)
- 修复后文件 blob SHA: `ccf7797c781bc7f3f7b640774515dcffada15f97`

## 修复后静态复核

重新读取 GitHub 文件后，确认：

- 七个 workflow step 均位于相同的 `jobs.build.steps` 缩进层级。
- 没有发现其他缩进异常的 `- name:` 步骤行。
- Build 步骤结束后，Audit 和 Upload 步骤均已恢复为正常步骤层级。
- 未改动内核配置 fragment。

本次是针对已确认错误的定点修复。以上属于文件结构静态复核，不应冒称已通过独立 YAML parser 或 GitHub Actions 实际运行验证。下一步应在 GitHub Actions 页面确认 workflow 能正常加载，再触发构建。
