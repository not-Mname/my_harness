# Superpowers-zh 技能版本库

本仓库用于保存 superpowers-zh 插件技能的历史版本，配合 my_harness 对插件的直接修改。

## 版本记录

- `baseline`：v1.7.1+codex.20260731143213 原版快照（对应插件仓库提交 `08c4d2c`，2026-08-01 归档）
- 1.7.1-myh：按 my_harness 分级适配 6 个技能（仅替换冲突处为分级门禁，原版内容保留），2026-08-01 归档

## 使用说明

- 插件运行时目录：`C:\Users\wjf\.codex\plugins\local\superpowers-zh\skills`
- 修改插件技能后，将 `skills/*` 同步回本仓库并提交，作为新版本快照。
- 原始安装方式的 junction 已改名备份为 `C:\Users\wjf\.agents\skills\superpowers-native-link`（指向插件 skills）。