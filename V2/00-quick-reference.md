# Claude Code 速查卡（Quick Reference）

> 随时查阅，无需翻阅完整文档

---

## Slash Commands 速查

```bash
# 会话管理
/clear          清除当前会话
/branch [name]  创建会话分支（原 /fork）
/rewind         回滚到历史检查点（也可 Esc+Esc）
/export [file]  导出对话到文件
/resume         恢复上次会话

# 模型与性能
/model [model]  切换模型 (opus/sonnet/haiku)
/effort [level] 推理强度 (low/medium/high/max/auto)

# 计划与执行
/plan [desc]    进入 Planning Mode（只读规划）
/ultraplan      浏览器端规划，终端保持响应

# 技能与插件
/skills         查看可用技能列表
/agents         查看/管理子智能体
/plugin list    列出已安装插件
/plugin install X  安装插件

# 记忆
/init           初始化项目记忆（创建 CLAUDE.md）
/memory         编辑记忆文件
/context        查看上下文使用情况

# 调试
/help           帮助
/debug          调试当前会话
/version        查看版本
```

---

## CLI 速查

```bash
# 启动
claude                          # 交互模式
claude -p "query"               # Print Mode（单次查询）
claude -c                       # 继续上次会话
claude -r "name"                # 恢复命名会话

# 管道
cat file.ts | claude -p "分析"
git diff | claude -p "写 commit message"

# 模型
claude --model claude-opus-4-6
claude --model claude-sonnet-4-6
claude --model claude-haiku-4-5

# 权限模式
claude --permission-mode acceptEdits  # 自动接受编辑
claude --permission-mode plan         # 只规划不执行
claude --permission-mode dontAsk      # 减少确认

# 工具控制
claude --allowedTools "Read,Grep,Glob"
claude --disallowedTools "Bash,Write"

# 插件
claude --plugin-dir ./my-plugin

# 子智能体（session 级别）
claude --agents '{"name": {"description":"...", "prompt":"...", "tools":["Read"]}}'

# 输出格式
claude -p "query" --output-format json
claude -p "query" --output-format stream-json
```

---

## Memory 文件层级

```
优先级（高→低）
1. /Library/.../ClaudeCode/CLAUDE.md     # 企业托管策略（macOS）
   C:\Program Files\ClaudeCode\CLAUDE.md # 企业托管策略（Windows）
2. managed-settings.d/*.md               # 模块化策略（v2.1.83+）
3. ./CLAUDE.md 或 ./.claude/CLAUDE.md   # 项目记忆（提交 git）
4. ./.claude/rules/*.md                  # 路径特定规则
5. ~/.claude/CLAUDE.md                   # 个人偏好（所有项目）
6. ~/.claude/rules/*.md                  # 个人规则
7. ./CLAUDE.local.md                     # 本地个人偏好（不提交）
8. ~/.claude/projects/<proj>/memory/     # Claude 自动写入笔记
```

---

## Skills SKILL.md 全字段

```yaml
---
name: skill-name                    # 必须，小写+连字符，≤64字符
description: "做什么+何时触发"       # 必须，≤1024字符，含触发关键词
argument-hint: "[arg1] [arg2]"      # 可选，命令补全提示
disable-model-invocation: true      # 只有用户能触发（部署/危险操作用）
user-invocable: false               # 只有 Claude 自动触发（背景知识用）
allowed-tools: Read, Grep, Glob     # 限制可用工具
model: opus                         # 指定模型
effort: high                        # 推理强度：low/medium/high/max
context: fork                       # 在独立子智能体中运行
agent: Explore                      # 子智能体类型（配合 context:fork）
shell: bash                         # bash（默认）或 powershell
paths: "src/api/**/*.ts"            # 仅在此路径下自动激活
hooks:                              # 组件级钩子
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./validate.sh"
---
```

**变量：** `$ARGUMENTS` `$0` `$1` `${CLAUDE_SESSION_ID}` `${CLAUDE_SKILL_DIR}`  
**动态注入：** `` !`shell-command` ``

---

## Subagents agent.md 全字段

```yaml
---
name: agent-name                     # 必须
description: "何时调用（加PROACTIVELY鼓励自动触发）"
tools: Read, Grep, Glob, Bash        # 省略=继承全部
disallowedTools: Write, Edit         # 明确禁止
model: sonnet                        # sonnet/opus/haiku/inherit
permissionMode: default              # default/acceptEdits/dontAsk/plan
maxTurns: 20                         # 最大执行轮数
skills: skill1, skill2               # 预加载技能
mcpServers: github, postgres         # 可访问的 MCP 服务器
memory: project                      # user/project/local
background: false                    # true=始终后台运行
effort: high                         # low/medium/high/max
isolation: worktree                  # 独立 git worktree
initialPrompt: "先分析代码库结构"    # 自动提交的第一条消息
hooks:                               # 组件级钩子
  PreToolUse: [...]
---
系统提示词在这里
```

---

## Hooks 26个事件速查

```
会话:   SessionStart  InstructionsLoaded  SessionEnd  PreCompact  PostCompact
用户:   UserPromptSubmit  Notification  Elicitation  ElicitationResult
工具:   PreToolUse  PermissionRequest  PermissionDenied  PostToolUse  PostToolUseFailure
Agent:  SubagentStart  SubagentStop  TeammateIdle  TaskCompleted  TaskCreated
文件:   FileChanged  CwdChanged  ConfigChange
停止:   Stop  StopFailure
Tree:   WorktreeCreate  WorktreeRemove
```

**Hook 退出码：** 0=成功 | 2=阻塞错误 | 其他=非阻塞

**Hook 类型：** command（默认）| http（v2.1.63+）| prompt | agent

---

## MCP CLI 速查

```bash
claude mcp add --transport http name https://url
claude mcp add --transport stdio name npx server
claude mcp list
claude mcp get <name>
claude mcp remove <name>
claude mcp reset-project-choices
claude mcp add-from-claude-desktop
claude mcp serve                    # 将 Claude 作为 MCP 服务器
```

---

## Plugin CLI 速查

```bash
/plugin install <name>              # 安装
/plugin list                        # 列出已安装
/plugin enable/disable <name>       # 启用/禁用
/plugin uninstall <name>            # 卸载
/plugin update <name>               # 更新
/reload-plugins                     # 热重载（开发时）
claude --plugin-dir ./path          # 本地测试插件
claude plugin validate              # 验证插件结构
```

---

## 权限模式速查

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `default` | 危险操作前询问 | 日常 |
| `acceptEdits` | 自动接受文件编辑 | 信任编辑场景 |
| `plan` | 只读，生成计划不执行 | 规划阶段 |
| `dontAsk` | 减少确认提示 | 熟悉项目 |
| `auto` | 后台安全分类器 | 高自主场景 |
| `bypassPermissions` | 完全绕过 | ⚠️ 慎用 |

---

## 环境变量速查

```bash
ANTHROPIC_API_KEY                        # API 密钥（必须）
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1        # 禁用自动记忆
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1   # 启用 Agent Teams
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1   # 禁用后台任务
SLASH_COMMAND_TOOL_CHAR_BUDGET=8000      # 技能描述字符预算
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=90       # 自动压缩阈值
CLAUDE_CODE_NEW_INIT=1                   # 增强 /init 交互流程
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1  # 启用 --add-dir
```

---

## 快捷键速查

| 快捷键 | 操作 |
|--------|------|
| `Esc + Esc` | 打开检查点浏览器 |
| `Ctrl+B` | 将当前任务放到后台 |
| `Ctrl+F` | 终止所有后台智能体（×2确认） |
| `Ctrl+O` | 切换 verbose 模式 |
| `Shift+Down` | Agent Teams：切换队员（split-pane模式） |
