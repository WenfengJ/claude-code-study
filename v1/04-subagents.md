# 模块 4：Subagents（子智能体）

> Level 2 自动化 | 预计学习时间：1.5小时

---

## 核心概念

Subagents 是 Claude Code 可以委托任务的专用 AI 助手。每个子智能体：
- 有独立的上下文窗口（不污染主对话）
- 可配置专属工具权限
- 可自定义系统提示词
- 支持并行执行

---

## 内置子智能体

| 智能体 | 模型 | 用途 |
|--------|------|------|
| **general-purpose** | 继承父级 | 复杂多步骤任务 |
| **Plan** | 继承父级 | Planning Mode 下的代码库研究 |
| **Explore** | Haiku（快） | 只读代码库探索（快/中/深度） |
| **Bash** | 继承父级 | 在独立上下文中执行终端命令 |
| **statusline-setup** | Sonnet | 配置状态栏 |
| **Claude Code Guide** | Haiku（快） | 回答 Claude Code 使用问题 |

---

## 配置文件格式

```yaml
---
name: your-agent-name                    # 必须：小写+连字符
description: |                           # 必须：何时调用此智能体
  代码安全审查专家。写完代码后主动使用。  # 加 "PROACTIVELY" 鼓励自动触发
tools: Read, Grep, Glob, Bash            # 省略则继承所有工具
disallowedTools: Write, Edit             # 明确禁止的工具
model: sonnet                            # sonnet/opus/haiku/inherit
permissionMode: default                  # default/acceptEdits/dontAsk/bypassPermissions/plan
maxTurns: 20                             # 最大执行轮数
skills: code-review, lint                # 预加载的技能
mcpServers: github                       # 可访问的 MCP 服务器
memory: project                          # 持久记忆范围：user/project/local
background: false                        # true=始终在后台运行
effort: high                             # 推理强度：low/medium/high/max
isolation: worktree                      # 给子智能体独立 git worktree
initialPrompt: "先分析代码库结构"        # 自动提交的第一条消息
hooks:                                   # 生命周期钩子
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---

子智能体的系统提示词写在这里。
可以是多段落，清晰定义角色、能力和工作方式。
```

---

## 文件存储位置与优先级

| 优先级 | 类型 | 位置 | 范围 |
|--------|------|------|------|
| 1（最高） | CLI 定义 | `--agents` 参数（JSON） | 仅当前 session |
| 2 | 项目级 | `.claude/agents/` | 当前项目 |
| 3 | 用户级 | `~/.claude/agents/` | 所有项目 |
| 4（最低） | 插件级 | 插件 `agents/` 目录 | 通过插件安装 |

---

## 管理子智能体

```bash
/agents           # 交互式菜单：查看/创建/编辑/删除子智能体
claude agents     # CLI 命令：列出所有智能体（含来源）
```

---

## 调用方式

### 自动委托
Claude 根据任务描述和 `description` 字段自动决定是否委托：
```
# 在 description 中加关键词触发
description: "安全审查专家。代码变更后主动使用(PROACTIVELY)。"
```

### 显式调用
```
用 test-runner 子智能体修复失败的测试
让 code-reviewer 智能体检查我最近的修改
请 debugger 智能体排查这个错误
```

### @ 提及（保证触发）
```
@"code-reviewer (agent)" 审查 auth 模块
```

### 整个 session 使用指定智能体
```bash
claude --agent code-reviewer    # CLI 参数
# 或在 settings.json 中
{ "agent": "code-reviewer" }
```

---

## 工具访问配置

```yaml
# 方式1：继承所有工具（省略 tools 字段）
---
name: full-access-agent
description: 拥有所有工具的智能体
---

# 方式2：指定具体工具
---
name: read-only-agent
tools: Read, Grep, Glob
---

# 方式3：条件性工具访问
---
name: npm-only-agent
tools: Read, Bash(npm:*), Bash(test:*)
---
```

---

## 持久记忆

```yaml
---
name: researcher
memory: user    # user=个人跨项目 | project=项目共享 | local=本地不提交
---
```

- 首次执行时自动创建记忆目录
- `MEMORY.md` 的前200行自动加入系统提示
- 子智能体可自由读写记忆目录中的文件

记忆目录位置：
- `user` → `~/.claude/agent-memory/<name>/`
- `project` → `.claude/agent-memory/<name>/`
- `local` → `.claude/agent-memory-local/<name>/`

---

## 后台运行

```yaml
---
name: long-runner
background: true
description: 在后台执行长时间分析任务
---
```

| 快捷键 | 操作 |
|--------|------|
| `Ctrl+B` | 将正在运行的任务放到后台 |
| `Ctrl+F` | 终止所有后台智能体（按两次确认） |

---

## Worktree 隔离

```yaml
---
name: feature-builder
isolation: worktree
description: 在独立 git worktree 中实现功能
tools: Read, Write, Edit, Bash, Grep, Glob
---
```

- 子智能体在独立 git 分支上工作
- 无变更则自动清理 worktree
- 有变更则返回 worktree 路径和分支名供主智能体审查合并

---

## 限制子智能体可生成的子智能体

```yaml
---
name: coordinator
tools: Agent(worker, researcher), Read, Bash
description: 协调多个专用智能体的工作
---
```

`coordinator` 只能派生 `worker` 和 `researcher`，无法派生其他智能体。

---

## Agent Teams（实验性功能）

> 需要 Claude Code v2.1.32+，默认关闭

```bash
# 启用 Agent Teams
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
# 或在 settings.json 中设置
```

启用后可让 Claude 组建团队：
```
构建认证模块。组建一个团队：
一个负责 API 端点，一个负责数据库 schema，一个负责测试套件。
```

**Subagents vs Agent Teams 对比：**

| 维度 | Subagents | Agent Teams |
|------|-----------|-------------|
| 委托模式 | 父级委托子任务，等待结果 | 团队长协调，队员独立执行 |
| 上下文 | 每次新鲜上下文 | 每个队员有自己的持久上下文窗口 |
| 通信 | 只返回结果给父级 | 队员间可通过邮箱直接通信 |
| 适合场景 | 明确聚焦的子任务 | 需要智能体间通信的复杂工作 |

---

## 预设子智能体示例

| 智能体 | 工具权限 | 用途 |
|--------|---------|------|
| code-reviewer | Read, Grep, Glob, Bash | 代码质量+安全审查 |
| test-engineer | Read, Write, Bash, Grep | 测试策略+覆盖率分析 |
| documentation-writer | Read, Write, Grep | 技术文档生成 |
| secure-reviewer | Read, Grep（只读） | 安全漏洞扫描 |
| debugger | Read, Edit, Bash, Grep, Glob | 错误根因分析 |
| data-scientist | Bash, Read, Write | SQL 查询+数据分析 |

---

## 何时使用子智能体

| 场景 | 用子智能体 | 原因 |
|------|-----------|------|
| 复杂功能开发（多步骤） | ✅ | 分离关注点，防止上下文污染 |
| 并行任务执行 | ✅ | 各自有独立上下文 |
| 专业领域任务 | ✅ | 自定义系统提示词 |
| 长时间分析 | ✅ | 防止主上下文耗尽 |
| 简单单步任务 | ❌ | 增加不必要延迟 |
| 快速代码审查 | ❌ | 开销不值得 |

---

## 相关模块

- [03-skills.md](03-skills.md) — `context: fork` 技能运行在子智能体中
- [06-hooks.md](06-hooks.md) — 子智能体生命周期钩子（SubagentStart/SubagentStop）
- [09-advanced-features.md](09-advanced-features.md) — Planning Mode 使用 Plan 子智能体
