# Superpowers 中文版插件迁移设计

## 背景

迁移前，本机通过本地 marketplace `superpowers-dev` 启用了插件
`superpowers@superpowers-dev`。其源码位于
`~/.codex/plugins/local/superpowers-dev`，包含一个没有远端备份的本地提交。

目标是将 `jnMetaCode/superpowers-zh` 作为独立 Codex 插件安装，插件身份使用
`superpowers-zh`，并卸载原插件。旧插件源码作为回滚副本保留，不直接删除。

## 选定方案

采用独立本地 Codex 插件方案，而不是上游文档中的 Skill-only 符号链接方案。
中文版仓库克隆至 `~/.codex/plugins/local/superpowers-zh`，并以本地
marketplace `superpowers-zh-dev` 暴露为
`superpowers-zh@superpowers-zh-dev`。

选择该方案的原因：

- 保留明确、独立的 `superpowers-zh` 插件身份。
- 保留插件目录中的 SessionStart Hook，而不仅是发现 Skills。
- 旧版和中文版的安装状态、缓存及回滚边界清晰。
- 后续可从中文版上游拉取更新，再重新生成 Codex cachebuster。

## 插件适配

上游 `.codex-plugin/plugin.json` 的名称保持 `superpowers-zh`。移除当前 Codex
插件校验器不支持的顶层 `hooks` 字段；`hooks/hooks.json` 与 Hook 脚本保留，依赖
Codex 对插件约定目录的发现机制。

版本保留上游基础版本，并由 `plugin-creator` 提供的更新脚本生成单个 Codex
cachebuster，形如 `1.7.1+codex.<timestamp>`。本地 marketplace 清单使用名称
`superpowers-zh-dev`，插件项指向同一仓库根目录，并声明安装策略、认证策略及
Developer Tools 分类。

不把旧插件中的 `writing-contextual-code-comments` 合并进中文版插件。该 Skill 已独立
安装在 `~/.agents/skills/writing-contextual-code-comments`，卸载旧插件后仍可发现。

## 迁移流程

1. 在临时目录校验中文版仓库的目标提交、目录结构和 Skill 清单。
2. 将中文版克隆到新的本地插件目录并完成 Codex 适配。
3. 校验插件 manifest、marketplace 清单、Skills frontmatter 和 Hook 脚本。
4. 使用 Codex CLI 卸载 `superpowers@superpowers-dev`，清除其安装配置和缓存。
5. 使用 Codex CLI 移除 `superpowers-dev` marketplace 注册，但保留其源码目录。
6. 注册 `superpowers-zh-dev` marketplace，并安装
   `superpowers-zh@superpowers-zh-dev`。
7. 核对配置、缓存、Skill 列表和 Hook 输出；在新任务中验证实际发现与注入。

不会修改或删除 `my_harness/global/root`，不会执行 Git 推送，也不会把项目中已有的
无关工作区修改混入本任务提交。

## 异常与回滚

迁移验证时，`codex plugin list` 曾被两个损坏的 OpenAI marketplace 显式注册阻断。迁移首先尝试
正常的插件 CLI 操作；如果相同问题阻断卸载或安装，则停止迁移并发布新的 HITL
checkpoint，不自行删除或修改这两个无关 marketplace 配置。

迁移完成后另经 `[CP-6]` 授权，仅使用 Codex CLI 移除两个损坏的显式注册。CLI 拒绝从
本地目录添加保留名称 `openai-curated`，但会自动提供内置 `openai-api-curated`；因此无需
回滚损坏配置。最终 marketplace 与 plugin 清单均可正常加载，且用户配置中不保留上述
两个显式 marketplace 配置块。

官方插件校验脚本当前因全局 Python 缺少 `PyYAML` 无法运行。不会擅自安装依赖；优先
使用现有运行时或结构化解析完成等价校验。若完整验证确实需要安装依赖，则另行确认。

迁移失败时，不删除旧插件源码。若旧插件已经卸载但新版尚未成功安装，使用保留的
`superpowers-dev` marketplace 源重新注册并安装旧插件。任何恢复操作均只覆盖本设计
说明的插件与 marketplace。

## 验收标准

- Codex 配置中启用 `superpowers-zh@superpowers-zh-dev`，不再启用
  `superpowers@superpowers-dev`。
- 新版 manifest 和 marketplace 清单可被结构化解析，插件校验无已知错误。
- 中文版全部 Skills 可被枚举，关键 `SKILL.md` frontmatter 合法。
- SessionStart Hook 可执行，在 Codex CLI `/hooks` 中为 `Trusted`，并输出
  `using-superpowers` 的中文注入内容。
- 独立的 `writing-contextual-code-comments` Skill 仍存在。
- 新任务能够发现 `superpowers-zh:*` Skills 并收到中文 SessionStart 注入；旧插件源码仍可用于回滚。
