---
name: using-git-worktrees
description: 当需要隔离工作区进行功能开发或执行实施计划时使用——默认禁止创建 worktree/branch/tag，须先获得 HITL 授权
---

# 使用 Git 工作树（Harness 裁剪版）

本技能是插件 `superpowers-zh:using-git-worktrees` 的 harness 门禁版。与 `~/.codex/rules/git-development.md` 冲突时以 git 规则为准。

## 门禁（默认禁止）

本 harness 的 `git-development.md` 明确：**未经允许不准创建 worktree、branch 或 tag**。

1. 使用本技能前必须通过 HITL Checkpoint（`[CP-n]`）逐项说明并获授权：worktree 创建、`.gitignore` 或项目设置修改及其提交、依赖安装与生命周期脚本、lockfile 或构建生成物变化、服务启动和基线测试。
2. 未获授权：**在当前目录原地工作**，不创建任何隔离区，不修改 `.gitignore` 或项目设置。
3. 平台已有原生隔离（如已处于功能分支）时不创建额外 worktree；优先使用当前分支与现有隔离。
4. 必要步骤未获授权时不得启动本技能；可退化为当前目录串行工作。

## 流程（获得授权后）

1. **检测现有隔离** — 已在 linked worktree 或已切换的功能分支内则跳过创建，直接进入项目设置。
2. **优先原生 worktree 工具** — 平台有原生工具时使用；仅在没有原生工具时才回退到 `git worktree add`。
3. **目录选择与忽略验证** — 按优先级：明确声明的工作树目录偏好 → 现有项目本地目录（`.worktrees/` 优先于 `worktrees/`）→ 默认 `.worktrees/`；创建前用 `git check-ignore` 验证目录已忽略，未忽略须经 HITL 添加 `.gitignore`。
4. **项目设置** — 自动检测并运行设置命令（npm install / cargo build / pip install 等，按项目实际）。
5. **验证基线干净** — 运行测试确认工作区初始状态；失败则报告并询问，不带着失败基线继续。
6. **报告就绪** — 输出工作树路径、测试结果，然后开始实现。

## 红线

- 未获 HITL 授权绝不创建 worktree、branch 或 tag。
- 已有原生隔离时不嵌套创建 worktree。
- 不跳过忽略验证；不跳过基线测试验证。
- 工作完成后按 `git-development.md` 收尾：不 push、不改写历史、提交前检查暂存 diff。