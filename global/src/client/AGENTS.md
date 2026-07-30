# Client Agent（客户端）

## 职责与范围

- 负责 Unity 2022.3 客户端逻辑、UI、战斗、场景、特效与 HybridCLR 热更。
- 默认仅可写：`Src/Client/Assets/Scripts/Core/`、`Src/Client/Assets/Scripts/HotUpdate/`。其他 Client 路径必须由子任务 Prompt 以绝对路径逐项授权。
- 只读：`Src/Lib/`、`Src/Server/`、`Src/Tests/`；跨端接口先交由当前 `coordinator` 固化到 `docs/contracts/`。
- `Assets/AssetBundle/` 和任意 `Resources/` 默认不在工作范围且不得自行下载；任何场景或资源修改、提交前必须获得用户明确同意。

## Unity 约束

- 挂载 `GameObject` 的单例继承 `MonoSingleton<T>`，非`GameObject`类可以使用共享库的 `Singleton<T>` ；
- 事件使用 `EVENT`，记得在 `OnDestroy` / `Dispose` 取消订阅。
- 新增 Manager/Service 必须加入 `GameEntry._initializables` 并确认初始化顺序；不得改动 `GameEntry.cs` 的反射加载结构。
- 禁止未经授权重构 `*.asmdef`。
- HotUpdate 不得直接引用编辑器 API；共享 DLL 位于 `Assets/References/`，不得绕过 Lib 授权门禁修改其来源。
