# CodeGraph 代码导航

repo root 存在 `.codegraph/` 时，理解或定位 code 应在 grep/find 或读取 files 前优先使用 CodeGraph。可用时使用 `codegraph_explore`；tool 为 deferred 时先 discovery 并加载。Shell 降级为 `codegraph explore "<symbol names or question>"`；无索引则跳过，不自行建立。
