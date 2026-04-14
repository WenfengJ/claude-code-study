# Claude Code 高阶用法学习手册

> 来源：[claude-howto](https://github.com/luongnv89/claude-howto)（Stars 26,100+）  
> 整理日期：2026-04-14 | 适配版本：Claude Code v2.1.101+

---

## 学习路线

### Level 1 — 基础（约3小时）

| 序号 | 模块 | 文件 | 核心收益 |
|------|------|------|---------|
| 1 | Slash Commands | [01-slash-commands.md](01-slash-commands.md) | 60+ 内置命令，自定义快捷指令 |
| 2 | Memory | [02-memory.md](02-memory.md) | 跨 session 持久上下文，8层记忆体系 |
| 3 | Checkpoints | [08-checkpoints.md](08-checkpoints.md) | 会话快照与回滚，安全实验 |
| 4 | CLI Basics | [10-cli.md](10-cli.md) | 交互/脚本/CI-CD 模式 |

### Level 2 — 自动化（约5小时）

| 序号 | 模块 | 文件 | 核心收益 |
|------|------|------|---------|
| 5 | Skills | [03-skills.md](03-skills.md) | 封装可复用技能，自动/手动触发 |
| 6 | Hooks | [06-hooks.md](06-hooks.md) | 26个事件驱动钩子，全自动化 |
| 7 | MCP Protocol | [05-mcp.md](05-mcp.md) | 接入外部工具（DB、GitHub、Slack等） |
| 8 | Subagents | [04-subagents.md](04-subagents.md) | 任务委托给专用子智能体，并行执行 |

### Level 3 — 高阶（约5小时）

| 序号 | 模块 | 文件 | 核心收益 |
|------|------|------|---------|
| 9 | Advanced Features | [09-advanced-features.md](09-advanced-features.md) | Planning Mode / Auto Mode / Worktrees / Voice 等 |
| 10 | Plugins | [07-plugins.md](07-plugins.md) | 功能集合打包，团队/企业分发 |

---

## 按场景快速定位

| 我想要… | 看哪个文件 |
|---------|-----------|
| 快速执行重复任务 | [01-slash-commands.md](01-slash-commands.md) |
| 让 Claude 记住项目规范 | [02-memory.md](02-memory.md) |
| 实验新方案不怕失败 | [08-checkpoints.md](08-checkpoints.md) |
| 把重复指令变成一个命令 | [03-skills.md](03-skills.md) |
| 自动化代码检查/格式化 | [06-hooks.md](06-hooks.md) |
| 让 Claude 读写数据库/API | [05-mcp.md](05-mcp.md) |
| 多任务并行，不撞上下文限制 | [04-subagents.md](04-subagents.md) |
| CI/CD 流水线集成 | [10-cli.md](10-cli.md) |
| 规划再执行，避免大错误 | [09-advanced-features.md](09-advanced-features.md) |
| 给团队分发统一工具集 | [07-plugins.md](07-plugins.md) |

---

## 关键概念一览

```
Claude Code 能力图谱
│
├── 用户发起
│   ├── Slash Commands（/命令）
│   └── Skills（/技能名）
│
├── 自动触发
│   ├── Skills（auto-invocation）
│   ├── Hooks（事件驱动）
│   └── Subagents（任务委托）
│
├── 持久化
│   ├── Memory（CLAUDE.md）
│   ├── Checkpoints（会话快照）
│   └── MCP（外部数据源）
│
└── 打包分发
    └── Plugins（技能+智能体+钩子+MCP 的集合）
```
