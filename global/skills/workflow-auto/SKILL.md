---
name: workflow-auto
description: Use by default when a new Codex conversation starts, or when 用户显式调用 $workflow-auto，要求恢复逐任务自动判断流程模式
---

# 持续自动判断流程

开始任务前必须读取 `~/.codex/rules/workflow-policy.md`。

1. 新对话默认进入自动模式；每个任务独立判断轻量级、中量级或重量级。
2. 输出 `[自动模式 -> 档位]` 和一句核心理由后立即执行，不请求用户批准。
3. 本次判定不改变持续模式；下一个任务重新判断。
4. 任务中途发现新风险时可以升级并告知，不得自动降级。
5. 显式调用不含任务正文时，只确认已恢复持续自动判断流程。
6. 最新显式模式切换优先，任务中途切换只影响未完成步骤。
