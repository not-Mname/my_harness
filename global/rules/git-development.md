# Git 开发边界

- 允许 Git 只读检查、`git add` 暂存、普通本地 `git commit` 以及 `git fetch` / `git pull`；这些操作无需另行确认。
- 提交前必须检查暂存 diff，只能暂存和提交当前任务已说明范围内的变更，不得混入用户或其他 Agent 的无关修改。
- `git commit --amend`、`git rebase`、`git reset` 及其他改写历史或丢弃变更的操作，须按当前 HITL Checkpoint 获得明确同意。
- 禁止执行或代办 `git push`、force-push 或其他远端分支更新；推送始终由用户自行处理，不应主动接管。
- 拉取发生冲突或需要手工解决时，必须停止并报告当前状态；解决冲突属于新的修改范围，须按 HITL Checkpoint 获得同意。
- 未经允许不准创建worktree、branch 或 tag
