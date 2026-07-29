# Superpowers 中文版插件迁移实施计划

> **面向 Agent 执行者：** REQUIRED SUB-SKILL：使用 `superpowers-zh:subagent-driven-development` 或 `superpowers-zh:executing-plans` 逐项实施。每个步骤使用 checkbox 跟踪；本计划操作用户级全局插件状态，不创建 Git worktree。

**目标：** 将 `jnMetaCode/superpowers-zh` 安装为独立的本地 Codex 插件 `superpowers-zh@superpowers-zh-dev`，卸载原有 `superpowers@superpowers-dev`，并保留旧源码以供回滚。

**架构：** 新版源码作为独立 Git 仓库克隆到用户级插件目录，通过仓库内的本地 marketplace 清单注册。先完成 manifest、Skills 和 Hook 的离线验证，再使用 Codex CLI 切换安装状态；任何切换失败都停止扩大修改，并利用保留的旧源码恢复。

**技术栈：** Git、Codex Plugin CLI、JSON、Ruby 标准库 `json`/`yaml`、Bash。

**实施结果：** 迁移、Hook 信任审核和新任务运行时验证均已完成。最终启用
`superpowers-zh@superpowers-zh-dev` 版本 `1.7.1+codex.20260729073527`；新任务
`019facfb-9963-7062-88fc-5e5242c5f36c` 已发现 `superpowers-zh:*` Skills，并收到中文
SessionStart 注入。

## 全局约束

- 新插件身份固定为 `superpowers-zh@superpowers-zh-dev`。
- 新插件源码固定为 `/Users/edy/.codex/plugins/local/superpowers-zh`。
- 旧插件源码 `/Users/edy/.codex/plugins/local/superpowers-dev` 必须保留，不得删除或改写历史。
- 不修改或删除 `/Users/edy/Desktop/my_harness/global/root`。
- 迁移阶段不修改损坏的 `openai-curated` 和 `openai-api-curated` marketplace；其后经
  `[CP-6]` 单独授权，仅使用 Codex CLI 移除两个损坏的显式注册并恢复清单命令。
- 不安装 `PyYAML` 或其他依赖；使用 Ruby 标准库完成结构化校验。
- 不执行 Git push、force-push、rebase、reset、amend、branch、tag 或 worktree 操作。
- 项目仓库中的现有无关修改不得暂存、提交、覆盖或删除。

---

### Task 1：准备并验证中文版插件源

**文件：**

- 创建：`/Users/edy/.codex/plugins/local/superpowers-zh/`
- 修改：`/Users/edy/.codex/plugins/local/superpowers-zh/.codex-plugin/plugin.json`
- 创建：`/Users/edy/.codex/plugins/local/superpowers-zh/.agents/plugins/marketplace.json`
- 验证：`/Users/edy/.codex/plugins/local/superpowers-zh/skills/*/SKILL.md`
- 验证：`/Users/edy/.codex/plugins/local/superpowers-zh/hooks/session-start`

**接口：**

- 消费：上游仓库 `https://github.com/jnMetaCode/superpowers-zh.git` 的 `main` 分支。
- 产出：可由 marketplace `superpowers-zh-dev` 安装的插件源，插件名为 `superpowers-zh`，包含 20 个可解析的 Skills 和可执行的 SessionStart Hook。

- [x] **Step 1：确认目标目录尚未占用并记录上游状态**

运行：

```bash
test ! -e /Users/edy/.codex/plugins/local/superpowers-zh
git -C /tmp/superpowers-zh-review rev-parse HEAD
git -C /tmp/superpowers-zh-review remote get-url origin
```

预期：第一条命令退出码为 `0`；后两条分别输出已审查提交和
`https://github.com/jnMetaCode/superpowers-zh.git`。若目标目录已存在，停止并发布新的
HITL checkpoint，不覆盖目录。

- [x] **Step 2：克隆中文版仓库**

运行：

```bash
git clone --branch main --single-branch https://github.com/jnMetaCode/superpowers-zh.git /Users/edy/.codex/plugins/local/superpowers-zh
git -C /Users/edy/.codex/plugins/local/superpowers-zh rev-parse --abbrev-ref HEAD
git -C /Users/edy/.codex/plugins/local/superpowers-zh remote get-url origin
```

预期：分支为 `main`，远端为用户指定的 GitHub 仓库。

- [x] **Step 3：移除 Codex manifest 中不受支持的字段**

使用 `apply_patch` 删除以下内容，其他上游元数据保持不变：

```diff
   "skills": "./skills/",
-  "hooks": {},
   "interface": {
```

然后运行：

```bash
ruby -EUTF-8:UTF-8 -rjson -e 'p=ARGV.fetch(0); j=JSON.parse(File.read(p)); abort "wrong name" unless j["name"]=="superpowers-zh"; abort "unsupported hooks field" if j.key?("hooks")' /Users/edy/.codex/plugins/local/superpowers-zh/.codex-plugin/plugin.json
```

预期：命令退出码为 `0`，无输出。

- [x] **Step 4：创建独立 marketplace 清单**

使用 `apply_patch` 创建
`/Users/edy/.codex/plugins/local/superpowers-zh/.agents/plugins/marketplace.json`：

创建父目录：

```bash
mkdir -p /Users/edy/.codex/plugins/local/superpowers-zh/.agents/plugins
```

```json
{
  "name": "superpowers-zh-dev",
  "interface": {
    "displayName": "Superpowers 中文版 Dev"
  },
  "plugins": [
    {
      "name": "superpowers-zh",
      "source": {
        "source": "url",
        "url": "./"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Developer Tools"
    }
  ]
}
```

运行：

```bash
ruby -EUTF-8:UTF-8 -rjson -e 'j=JSON.parse(File.read(ARGV.fetch(0))); p=j.fetch("plugins").fetch(0); abort "wrong marketplace" unless j["name"]=="superpowers-zh-dev"; abort "wrong plugin" unless p["name"]=="superpowers-zh"; abort "wrong source" unless p.dig("source","url")=="./"' /Users/edy/.codex/plugins/local/superpowers-zh/.agents/plugins/marketplace.json
```

预期：命令退出码为 `0`，无输出。

- [x] **Step 5：生成 Codex cachebuster**

运行：

```bash
python3 /Users/edy/.codex/skills/.system/plugin-creator/scripts/update_plugin_cachebuster.py /Users/edy/.codex/plugins/local/superpowers-zh
ruby -EUTF-8:UTF-8 -rjson -e 'v=JSON.parse(File.read(ARGV.fetch(0))).fetch("version"); abort "missing cachebuster: #{v}" unless v.match?(/\A1\.7\.1\+codex\.[0-9A-Za-z.-]+\z/); puts v' /Users/edy/.codex/plugins/local/superpowers-zh/.codex-plugin/plugin.json
```

预期：输出一个且仅一个形如 `1.7.1+codex.<cachebuster>` 的版本号。

- [x] **Step 6：验证全部 Skills 的 frontmatter**

运行：

```bash
ruby -EUTF-8:UTF-8 -ryaml -e '
files=Dir["/Users/edy/.codex/plugins/local/superpowers-zh/skills/*/SKILL.md"].sort
abort "expected 20 skills, got #{files.length}" unless files.length==20
files.each do |f|
  text=File.read(f)
  match=text.match(/\A---\n(.*?)\n---\n/m) or abort "missing frontmatter: #{f}"
  data=YAML.safe_load(match[1], permitted_classes: [], aliases: false)
  abort "missing name: #{f}" unless data.is_a?(Hash) && data["name"].is_a?(String) && !data["name"].empty?
  abort "missing description: #{f}" unless data["description"].is_a?(String) && !data["description"].empty?
end
puts "validated #{files.length} skills"
'
```

预期：输出 `validated 20 skills`。

- [x] **Step 7：验证 SessionStart Hook**

运行：

```bash
bash -n /Users/edy/.codex/plugins/local/superpowers-zh/hooks/session-start
/Users/edy/.codex/plugins/local/superpowers-zh/hooks/session-start | ruby -EUTF-8:UTF-8 -rjson -e 'j=JSON.parse(STDIN.read); c=j["additionalContext"] || j.dig("hookSpecificOutput","additionalContext"); abort "missing context" unless c&.include?("using-superpowers"); abort "missing Chinese content" unless c.include?("\u8C03\u7528\u76F8\u5173\u6216\u88AB\u8BF7\u6C42\u7684\u6280\u80FD")'
```

预期：所有命令退出码为 `0`，无输出。

- [x] **Step 8：提交本地适配，保留可追溯更新点**

运行：

```bash
git -C /Users/edy/.codex/plugins/local/superpowers-zh status --short
git -C /Users/edy/.codex/plugins/local/superpowers-zh diff --check
git -C /Users/edy/.codex/plugins/local/superpowers-zh add -- .codex-plugin/plugin.json .agents/plugins/marketplace.json
git -C /Users/edy/.codex/plugins/local/superpowers-zh diff --cached --check
git -C /Users/edy/.codex/plugins/local/superpowers-zh commit -m "适配 Codex 本地插件安装"
```

预期：只提交两个适配文件；上游其他文件没有未提交修改。

### Task 2：切换插件与 marketplace

**文件：**

- 修改：`/Users/edy/.codex/config.toml`（仅由 Codex CLI 管理对应插件和 marketplace 项）
- 删除：旧插件的安装缓存（仅由 `codex plugin remove` 管理）
- 创建：新插件的安装缓存（仅由 `codex plugin add` 管理）
- 保留：`/Users/edy/.codex/plugins/local/superpowers-dev/`

**接口：**

- 消费：Task 1 已验证并提交的 `superpowers-zh` 插件源。
- 产出：启用 `superpowers-zh@superpowers-zh-dev`，并移除旧插件的启用项和旧 marketplace 注册。

- [x] **Step 1：记录切换前状态**

运行：

```bash
rg -n 'superpowers|superpowers-dev' /Users/edy/.codex/config.toml
git -C /Users/edy/.codex/plugins/local/superpowers-dev status --short
git -C /Users/edy/.codex/plugins/local/superpowers-dev log -1 --oneline
```

预期：配置包含 `superpowers@superpowers-dev` 和 `marketplaces.superpowers-dev`；旧源码仍位于本地提交 `2bc55b6`。记录实际输出，后续不得修改旧源码。

- [x] **Step 2：使用 Codex CLI 卸载旧插件**

运行：

```bash
codex plugin remove superpowers@superpowers-dev --json
```

预期：JSON 表示卸载成功，旧插件启用项和对应安装缓存被移除。如果命令因
`openai-curated` 或 `openai-api-curated` marketplace 快照错误失败，立即停止并发布新的
HITL checkpoint；不要手工编辑 `config.toml`。

- [x] **Step 3：移除旧 marketplace 注册**

运行：

```bash
codex plugin marketplace remove superpowers-dev --json
test -d /Users/edy/.codex/plugins/local/superpowers-dev
```

预期：CLI 报告移除成功，且第二条命令确认旧源码目录仍存在。

- [x] **Step 4：注册中文版 marketplace**

运行：

```bash
codex plugin marketplace add /Users/edy/.codex/plugins/local/superpowers-zh --json
```

预期：JSON 中 marketplace 名称为 `superpowers-zh-dev`，来源为指定本地路径。

- [x] **Step 5：安装中文版插件**

运行：

```bash
codex plugin add superpowers-zh@superpowers-zh-dev --json
```

预期：JSON 表示 `superpowers-zh@superpowers-zh-dev` 安装并启用成功。

- [x] **Step 6：定义并核对两种切换中断回滚分支（本次均未触发）**

**分支 A：Step 2 成功，但 Step 3 失败，且旧 marketplace 仍存在。**

先只读确认 `[marketplaces.superpowers-dev]` 尚未被移除，再直接恢复旧插件；不得重复添加
仍存在的 marketplace：

```bash
ruby -EUTF-8:UTF-8 -e 't=File.read("/Users/edy/.codex/config.toml"); abort "old marketplace missing" unless t.include?("[marketplaces.superpowers-dev]")'
codex plugin add superpowers@superpowers-dev --json
codex plugin list
ruby -EUTF-8:UTF-8 -e 't=File.read("/Users/edy/.codex/config.toml"); abort "old marketplace missing" unless t.include?("[marketplaces.superpowers-dev]"); abort "old plugin missing" unless t.include?(%q{[plugins."superpowers@superpowers-dev"]}); abort "new plugin unexpectedly enabled" if t.include?(%q{[plugins."superpowers-zh@superpowers-zh-dev"]})'
test -d /Users/edy/.codex/plugins/local/superpowers-dev
```

预期：旧 marketplace 保持注册，`superpowers@superpowers-dev` 恢复为已启用；
`codex plugin list`、配置断言和旧源码目录检查均退出 `0`。如果恢复命令也失败，记录 Step 3
及恢复命令的原始错误并停止，不继续重试。

**分支 B：Step 3 已成功，但 Step 4 或 Step 5 失败。**

旧 marketplace 已不存在，因此先重新注册保留的旧源码，再恢复旧插件：

```bash
codex plugin marketplace add /Users/edy/.codex/plugins/local/superpowers-dev --json
codex plugin add superpowers@superpowers-dev --json
codex plugin list
ruby -EUTF-8:UTF-8 -e 't=File.read("/Users/edy/.codex/config.toml"); abort "old marketplace missing" unless t.include?("[marketplaces.superpowers-dev]"); abort "old plugin missing" unless t.include?(%q{[plugins."superpowers@superpowers-dev"]}); abort "new plugin unexpectedly enabled" if t.include?(%q{[plugins."superpowers-zh@superpowers-zh-dev"]})'
test -d /Users/edy/.codex/plugins/local/superpowers-dev
```

预期：旧 marketplace 和旧插件恢复启用；`codex plugin list`、配置断言和旧源码目录检查均
退出 `0`。报告新版失败命令及原始错误；如果恢复失败，同时报告恢复命令原始错误并停止，
不继续重试。

实际：本次 Step 3、Step 4 和 Step 5 均成功，两条回滚分支均未触发；checkbox 表示分支已
定义并核对，不表示执行过恢复命令。旧源码继续作为回滚副本保留。

### Task 3：验证全局插件状态

**文件：**

- 验证：`/Users/edy/.codex/config.toml`
- 验证：`/Users/edy/.codex/plugins/cache/superpowers-zh-dev/`
- 验证：`/Users/edy/.agents/skills/writing-contextual-code-comments/SKILL.md`

**接口：**

- 消费：Task 2 完成后的 Codex 全局插件状态。
- 产出：满足设计文档全部验收标准的验证证据，以及需要在新任务完成的最终发现检查。

- [x] **Step 1：核对配置中的新旧身份**

运行：

```bash
ruby -EUTF-8:UTF-8 -e 't=File.read("/Users/edy/.codex/config.toml"); abort "new marketplace missing" unless t.include?("[marketplaces.superpowers-zh-dev]"); abort "new plugin missing" unless t.include?(%q{[plugins."superpowers-zh@superpowers-zh-dev"]}); abort "old marketplace remains" if t.include?("[marketplaces.superpowers-dev]"); abort "old plugin remains" if t.include?(%q{[plugins."superpowers@superpowers-dev"]})'
```

预期：命令退出码为 `0`，无输出。

- [x] **Step 2：核对安装缓存和独立 Skill**

运行：

```bash
find /Users/edy/.codex/plugins/cache/superpowers-zh-dev -path '*/.codex-plugin/plugin.json' -o -path '*/skills/using-superpowers/SKILL.md'
test -f /Users/edy/.agents/skills/writing-contextual-code-comments/SKILL.md
test -d /Users/edy/.codex/plugins/local/superpowers-dev
```

预期：找到新版缓存中的 manifest 和 `using-superpowers`，独立注释 Skill 与旧源码目录均仍存在。

- [x] **Step 3：运行插件清单检查**

运行：

```bash
codex plugin list
```

预期：列出 `superpowers-zh@superpowers-zh-dev` 为已启用，不列出旧插件为已启用。

实际：首次检查因两个损坏的 OpenAI marketplace 显式注册退出 `1`。经 `[CP-6]` 授权，
Codex CLI 成功移除这两个配置块；随后尝试从本地目录添加保留名称 `openai-curated` 时退出
`1`，因此停止后续状态写入。无需回滚已删除的损坏配置：CLI 会自动提供内置
`openai-api-curated`。最终 `codex plugin marketplace list` 和 `codex plugin list` 均退出
`0`，后者列出 `superpowers-zh@superpowers-zh-dev` 为 `installed, enabled`。

- [x] **Step 4：核对项目仓库未混入全局迁移产物**

运行：

```bash
git -C /Users/edy/Desktop/my_harness status --short
git -C /Users/edy/Desktop/my_harness diff --check
```

预期：除实施前已经存在的用户修改外，没有插件源码、缓存或 `global/root` 变更；设计和实施计划提交保持独立。

- [x] **Step 5：在新 Codex 任务验证发现、Hook 信任与注入**

新任务 `019facfb-9963-7062-88fc-5e5242c5f36c` 已确认发现
`superpowers-zh:using-superpowers`、`superpowers-zh:chinese-documentation` 和
`superpowers-zh:workflow-runner`，并收到包含 `You have superpowers`、
`<EXTREMELY_IMPORTANT>` 和 `<SUBAGENT-STOP>` 标记的中文 SessionStart 注入。

Codex CLI `/hooks` 审核显示该 Hook 为 `Trusted`，来源为
`superpowers-zh@superpowers-zh-dev`；事件为 `SessionStart`，matcher 为
`startup|clear|compact`，命令指向安装缓存版本
`1.7.1+codex.20260729073527/hooks/run-hook.cmd session-start`。
