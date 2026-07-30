# MMORPG 多 Agent 协作规则

本文件是共享规则与编排规则的唯一权威来源；子目录 `AGENTS.md` 仅补充本目录边界。

## 编排

| Agent | 职责与写入范围 |
|---|---|
| `coordinator` | 唯一协调者：维护 `docs/contracts/`、划分角色所有权、处理 blocker、汇总 handoff 和最终验证；手动模式由用户承担，Leader 模式由当前主 Agent 承担。 |
| `client` | `Src/Client/` Unity 客户端。 |
| `server` | `Src/Server/` .NET 服务端。 |
| `lib` | `Src/Lib/` 共同契约，仅在用户明确授权后工作。 |
| `test` | `Src/Tests/` 测试与验证。 |

- `client`、`server`、`lib`、`test`、`reviewer` 不得继续派生子任务；每个子任务 Prompt 必须包含目标文件绝对路径、依赖接口/类型摘要、上游输出契约和验收条件。
- 跨端接口先交由当前 `coordinator` 稳定到 `docs/contracts/`；仅在 Client/Server 写入范围不重叠后才可并行。
- handoff 固定报告：变更文件、契约使用或变更、关键决策、验证结果、风险、剩余工作。

## 共享库与构建

- `Src/Lib` 是 Client/Server 共同契约，默认只读；修改前必须获得用户明确授权。
- 构建顺序为 `.proto` -> `genproto.bat` -> `Protocol.csproj` -> `Common.csproj`，DLL 随后复制到 `Src/Client/Assets/References/`。
- 仓库未跟踪 `Tools/genproto.bat`：协议修改发现该生成器缺失时必须报告阻塞，禁止手工修改生成代码或执行无条件生成命令。
- `Src/Lib/Protocol` 与 `Src/Lib/Common` 目标为 `net48`、C# 8.0；不得使用 .NET Core+ 专有 API。Server 不得依赖 Unity 类型。

## 代码与变更边界

- 字段用 `_camelCase`，属性和方法用 `PascalCase`。
- `message.proto` 属于 Client/Server 共同契约，禁止未经授权重构。
- 只实现获准范围内的必要改动；结构性定位优先使用 CodeGraph。
