# Editor MCP 与资源安全

Cocos Creator 使用 `funplay_cocos`，Unity 使用 `unityMCP`。操作引擎时优先使用对应 MCP；项目已打开且 MCP 可用时，优先查询项目、scene、node/GameObject、component、Prefab、material、asset 和运行状态。

修改项目、scene、节点、GameObject、component、Prefab、material、asset、资源或项目设置前，必须确认 MCP 连接的是目标项目和 scene，并读取当前状态。随后按照全局 Human-in-the-Loop Checkpoint 说明目标、当前事实、具体拟修改内容、影响范围和验证方式；获得明确同意后才能在已说明范围内修改，修改后再次查询验证。

未经同意时仅可读取、查询和诊断。一次授权不得沿用到后续 checkpoint，也不得扩大到未说明的对象、属性或项目设置。任何场景或资源修改都必须单独获得明确同意。

## Cocos 安全约束

- 修改节点前先查询 position、scale、anchor、widget 等当前状态，禁止盲写绝对值。
- 涉及坐标或尺寸计算时，默认采用 Anchor `(0.5,0.5)` 居中锚点体系；若目标节点 anchor 不同，必须在修改前显式说明。
- 不得仅为采用默认计算体系而要求把 anchor 改为 `(0.5,0.5)`；用户明确授权修改 anchor 时，才可按授权范围执行。

MCP 不可用时遵循全局 Capability Recovery。不得擅自启停编辑器、运行 Cocos 项目或让 Unity 进入 Play Mode；恢复失败后回退到文件只读分析，并报告已尝试操作、失败原因、影响和降级方案。
