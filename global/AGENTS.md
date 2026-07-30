## 编辑前阅读

1. 阅读根目录的 `AGENTS.md`。
2. 确定你预计要触碰的每个文件或文件夹。
3. 从仓库根目录开始，遍历到每个目标路径。
4. 读取沿途找到的每一份 `AGENTS.md`。
5. 如果某份父级 `AGENTS.md` 中列出了一个作用域包含该路径的子级 `AGENTS.md`，则读取该子级文档，并从此处继续向下遍历。
6. 如果文档之间存在冲突，则距离更近的文档对本地工作细节有更高优先级。

## 回应风格

- 除代码标识符、API、协议、命令、路径、日志原文、固定技术术语及用户要求保留的原文外，用户可见内容默认使用简体中文；新建或修改的文档、代码注释、计划、handoff、review 结论和提交说明同样适用。不得为翻译而改动外部契约或生成代码。
- 所有可见消息须以独立首行 `你好，今甫；` 开头。

## 流程模式

- 默认使用 `$workflow-auto`。
- 出现任一流程模式 Skill 或模式标签，或任务需要开发流程时，须先读取 `~/.codex/rules/workflow-policy.md`。

## 多Agent协助规则

- 如果要使用多个子Agent协作，必须读取 `~/.codex/rules/multi-agent-assistance.md`；

## CodeGraph

- repo root 存在 `.codegraph/` 时，理解或定位 code 应在 grep/find 或读取 files 前优先使用 CodeGraph。可用时使用 `codegraph_explore`；tool 为 deferred 时先 discovery 并加载。Shell 降级为 `codegraph explore "<symbol names or question>"`；无索引则跳过，不自行建立。

## Editor MCP

- 涉及 Cocos、Unity、scene、节点、GameObject、Prefab、material、asset、资源或项目设置时，须先读取 `~/.codex/rules/editor-mcp-checkpoint.md`。

## Git

涉及 Git 状态、暂存、拉取、提交或远端操作时，须先读取 `~/.codex/rules/git-development.md`。

## 人工确认（HITL）检查点

需确认的操作/假设或目标须标记任务内唯一的 `[CP-n]`，说明当前事实、具体拟修改内容或拟采用值、影响范围和验证方式，再暂停等待明确回复。可回复 `c`（continue）、`a`（adjust）或 `s`（stop）；单字母仅绑定最近一个未决 checkpoint，同时存在多个时必须附编号，如 `c CP-2`。`a` 只表示提出调整，主 Agent 须关闭原编号、发布新编号并再次等待；`c` 或 `s` 处理后立即关闭对应编号。授权仅对对应编号和已说明范围有效，不得沿用或扩大。

## 假设协议

启动阶段可合并的需求澄清最多一轮；后续强制 HITL Checkpoint 不受此限。低影响行业惯例、既有模式或无业务影响细节可标记 `[ASSUMED]` 并附依据。需求目标、数据结构、API 契约、交互流程、性能指标及场景、资源、项目设置、数据、安全或外部副作用不得假设，须进入 checkpoint。无法继续时输出 `[BLOCKED: 缺少 X/Y/Z 信息]`。
