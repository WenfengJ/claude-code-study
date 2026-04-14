# Claude Code 高阶用法完全手册 v2

> 来源：[claude-howto](https://github.com/luongnv89/claude-howto)（Stars 26,100+）  
> 版本：Claude Code v2.1.101+ | 整理日期：2026-04-14  
> 适合：有一周（40h+）系统学习时间的开发者

---

## 40小时学习计划

### 第1天（8h）：基础三件套
| 时间 | 内容 | 文件 |
|------|------|------|
| 2h | Slash Commands 全面掌握 | [01-slash-commands.md](01-slash-commands.md) |
| 3h | Memory 记忆系统深度配置 | [02-memory.md](02-memory.md) |
| 2h | CLI 模式与 CI/CD 集成 | [10-cli.md](10-cli.md) |
| 1h | Checkpoints 工作流实践 | [08-checkpoints.md](08-checkpoints.md) |

### 第2天（8h）：自动化核心
| 时间 | 内容 | 文件 |
|------|------|------|
| 3h | Skills 技能封装与触发控制 | [03-skills.md](03-skills.md) |
| 3h | Hooks 26个事件全面掌握 | [06-hooks.md](06-hooks.md) |
| 2h | 练习：配置项目自动化钩子 | [14-exercises.md](14-exercises.md) §1-2 |

### 第3天（8h）：外部集成
| 时间 | 内容 | 文件 |
|------|------|------|
| 3h | MCP Protocol 外部服务接入 | [05-mcp.md](05-mcp.md) |
| 3h | Subagents 子智能体并行架构 | [04-subagents.md](04-subagents.md) |
| 2h | 练习：配置 GitHub MCP + 代码审查 Agent | [14-exercises.md](14-exercises.md) §3-4 |

### 第4天（8h）：高阶功能
| 时间 | 内容 | 文件 |
|------|------|------|
| 4h | Advanced Features 全功能详解 | [09-advanced-features.md](09-advanced-features.md) |
| 2h | Plugins 插件设计与分发 | [07-plugins.md](07-plugins.md) |
| 2h | 练习：Planning Mode + Subagents 大任务拆解 | [14-exercises.md](14-exercises.md) §5-6 |

### 第5天（8h）：集成与工作流
| 时间 | 内容 | 文件 |
|------|------|------|
| 3h | 跨模块集成模式（5大模式） | [11-integration-patterns.md](11-integration-patterns.md) |
| 3h | 真实工作流模板（6个场景） | [12-real-world-workflows.md](12-real-world-workflows.md) |
| 2h | 故障排查手册 | [13-troubleshooting.md](13-troubleshooting.md) |

### 第6-7天（8h）：实战项目
| 时间 | 内容 | 文件 |
|------|------|------|
| 4h | 综合项目：为自己的项目配置完整 Claude Code 环境 | [14-exercises.md](14-exercises.md) §综合 |
| 4h | 进阶挑战：构建一个完整的 PR Review Plugin | [14-exercises.md](14-exercises.md) §挑战 |

---

## 全部文件导航

| 文件 | 内容 | 适合阶段 |
|------|------|---------|
| [00-quick-reference.md](00-quick-reference.md) | 单页速查卡，随时查阅 | 全程 |
| [01-slash-commands.md](01-slash-commands.md) | 60+ 命令完整参考 | 第1天 |
| [02-memory.md](02-memory.md) | 8层记忆体系 + 完整模板 | 第1天 |
| [03-skills.md](03-skills.md) | 技能全字段参考 + 6个示例 | 第2天 |
| [04-subagents.md](04-subagents.md) | 子智能体完整指南 + Agent Teams | 第3天 |
| [05-mcp.md](05-mcp.md) | MCP 配置 + OAuth + 企业管理 | 第3天 |
| [06-hooks.md](06-hooks.md) | 26事件全示例 + 4种 Hook 类型 | 第2天 |
| [07-plugins.md](07-plugins.md) | 插件创建到发布完整流程 | 第4天 |
| [08-checkpoints.md](08-checkpoints.md) | 检查点工作流 + 上下文监控 | 第1天 |
| [09-advanced-features.md](09-advanced-features.md) | 14个高阶功能详解 | 第4天 |
| [10-cli.md](10-cli.md) | CLI 完整参考 + CI/CD 模板 | 第1天 |
| [11-integration-patterns.md](11-integration-patterns.md) | 5大跨模块集成模式 | 第5天 |
| [12-real-world-workflows.md](12-real-world-workflows.md) | 6个真实工作流模板 | 第5天 |
| [13-troubleshooting.md](13-troubleshooting.md) | 常见问题故障排查 | 第5天 |
| [14-exercises.md](14-exercises.md) | 分模块练习 + 综合项目 | 第2-7天 |

---

## 能力地图

```
Claude Code 完整能力体系
│
├── 用户发起（主动触发）
│   ├── Slash Commands  → /help /plan /model /branch ...
│   ├── Skills          → /skill-name（用户手动）
│   └── CLI             → claude -p "..." | pipe | CI/CD
│
├── 自动触发（智能感知）
│   ├── Skills          → 关键词匹配自动调用
│   ├── Hooks           → 26个事件（Pre/Post/Session/File...）
│   └── Subagents       → description 关键词匹配委托
│
├── 持久化（跨会话记忆）
│   ├── Memory          → CLAUDE.md（8层层级）
│   ├── Checkpoints     → 会话快照（自动 + 手动回滚）
│   └── Auto Memory     → Claude 自主写入的笔记
│
├── 外部集成（实时数据）
│   └── MCP             → GitHub / DB / Slack / API ...
│
├── 并行执行（大任务分解）
│   └── Subagents       → 独立上下文 + 并行 + Agent Teams
│
└── 打包分发（团队/企业）
    └── Plugins         → Skills + Agents + MCP + Hooks 集合
```

---

## v1 vs v2 对比

| 维度 | v1 | v2 |
|------|----|----|
| 总文件数 | 11个 | 16个 |
| 内容深度 | 概念 + 基础示例 | 完整参考 + 多示例 + 练习 |
| 配置示例 | 简略 | 可直接复制使用 |
| 学习时长 | ~11h | 40h+ |
| 新增内容 | - | 集成模式、真实工作流、故障排查、练习题 |
