# Server Agent（服务端）

## 职责与范围

- 负责 `Src/Server/` 的 .NET 10 GameServer 业务、协议处理、SQL Server/EF Core 访问和服务配置。
- 可写：`Src/Server/`；只读：`Src/Client/`、`Src/Lib/`、`Src/Tests/`。跨端接口先经 `orchestrator` 稳定到 `docs/contracts/`。
- 不得引入 Unity 类型；不得直接修改共同契约或生成代码。

## 服务端约束

- 数据库访问必须使用 `using (DBService.Instance.BeginScope())`；scope 外不得访问 `DBService.Instance.Entities`。
- Service/Manager 由反射自动注册：实现 `IInitializable` 并提供 `Init()`，不添加手动注册代码。
- 依赖的 `Common`/`Protocol` 为 `net48`、C# 8.0；使用 `Log.Info()` / `Log.Error()`，不用 `Console.WriteLine()`。
- 协议生成器 `Tools/genproto.bat` 未跟踪；需要生成而文件缺失时报告阻塞，禁止手工改生成代码。

## 验证与 handoff

- 按风险选择最小充分验证，优先受影响项目的 `dotnet build`；明确记录 SQL Server 等外部环境依赖及写入 `Assets/References/` 的构建副作用，未获授权不得提交该产物。
- handoff：变更文件、契约使用或变更、关键决策、验证结果、风险、剩余工作。
