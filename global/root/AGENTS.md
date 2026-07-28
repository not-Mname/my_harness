# MMORPG 多 Agent 协作规则

本文件是共享规则与编排规则的唯一权威来源；子目录 `AGENTS.md` 仅补充本目录边界。

## 编排

| Agent | 职责与写入范围 |
|---|---|
| `orchestrator` | 唯一 primary Agent：分类需求、创建和维护 `docs/contracts/`、拆分和调度任务、汇总 handoff、最终验证；不直接实现专业目录业务代码。 |
| `client` | `Src/Client/` Unity 客户端。 |
| `server` | `Src/Server/` .NET 服务端。 |
| `lib` | `Src/Lib/` 共同契约，仅在用户明确授权后工作。 |
| `test` | `Src/Tests/` 测试与验证。 |
| `reviewer` | 只读审查。检查跨目录越权、契约兼容性、生成代码、DB、Unity 注册、测试充分性和无关改动。 |

- `client`、`server`、`lib`、`test`、`reviewer` 不得继续派生子任务；每个子任务 Prompt 必须包含目标文件绝对路径、依赖接口/类型摘要、上游输出契约和验收条件。
- 跨端接口先稳定到 `docs/contracts/`；仅在 Client/Server 写入范围不重叠后才可并行。
- handoff 固定报告：变更文件、契约使用或变更、关键决策、验证结果、风险、剩余工作。

## 共享库与构建

- `Src/Lib` 是 Client/Server 共同契约，默认只读；修改前必须获得用户明确授权。
- 构建顺序为 `.proto` -> `genproto.bat` -> `Protocol.csproj` -> `Common.csproj`，DLL 随后复制到 `Src/Client/Assets/References/`。
- 仓库未跟踪 `Tools/genproto.bat`：协议修改发现该生成器缺失时必须报告阻塞，禁止手工修改生成代码或执行无条件生成命令。
- `Src/Lib/Protocol` 与 `Src/Lib/Common` 目标为 `net48`、C# 8.0；不得使用 .NET Core+ 专有 API。Server 不得依赖 Unity 类型。

## 项目约束

| 范围 | 规则 |
|---|---|
| 运行时 | Client 使用 Unity 2022.3；Server 的 GameServer 目标为 `net10.0`。 |
| Singleton | 共享库使用 `Singleton<T>`（纯 C#、`new()` 约束）；Unity `MonoBehaviour` 使用 `MonoSingleton<T>`。不得混用。 |
| 注册 | Server 的 Service/Manager 由反射发现 `IInitializable` 并调用 `Init()`；Client 新增 Manager/Service 必须加入 `GameEntry._initializables`，并确认顺序。 |
| 数据库 | 所有服务端 DB 访问必须包在 `using (DBService.Instance.BeginScope())` 中；不得在 scope 外访问 `DBService.Instance.Entities`。 |
| 测试 | 当前无测试基础设施、CI 或 lint；按任务风险采用最小充分验证，不默认搭建重型基础设施。 |
| 场景与资源 | `AssetBundle/` 与所有 `Resources/` 默认不在 Agent 工作范围且不得自行下载；任何场景或资源修改、提交前必须说明内容并获得用户明确同意。 |

## 代码与变更边界

- 注释使用中文；字段用 `_camelCase`，属性和方法用 `PascalCase`。Server 使用 `Log.Info()` / `Log.Error()`，不用 `Console.WriteLine()`。
- Client 跨模块通信使用 `EventManager.Publish()` / `Subscribe()`，并在 `OnDestroy` / `Dispose` 取消订阅。
- 禁止重构 `GameEntry.cs`、`message.proto` 与 `*.asmdef`；`Entities.tt` 生成的 `Entities.cs` 不得手工编辑。
- 只实现获准范围内的必要改动；结构性定位优先使用 CodeGraph。
