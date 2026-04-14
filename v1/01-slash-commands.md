# 模块 1：Slash Commands（斜杠命令）

> Level 1 基础 | 预计学习时间：30分钟

---

## 核心概念

Slash Commands 是在交互式会话中控制 Claude 行为的快捷方式。有四种类型：

| 类型 | 说明 |
|------|------|
| **内置命令** | 原生命令，如 `/help`、`/clear`、`/model` |
| **Skills** | 用户自定义命令（推荐，见模块3） |
| **Plugin 命令** | 已安装插件提供的命令 |
| **MCP prompts** | MCP 服务器暴露的命令 |

---

## 常用内置命令

### 会话管理
```bash
/clear              # 清除当前会话
/branch [name]      # 创建会话分支（原 /fork）
/rewind             # 打开检查点浏览器（回滚用）
/export [filename]  # 导出对话到文件
/resume             # 恢复上次会话
```

### 模型与性能
```bash
/model [model]      # 切换模型（opus/sonnet/haiku）
/effort [level]     # 设置推理强度（low/medium/high/max/auto）
```

### 计划与执行
```bash
/plan [description] # 进入 Planning Mode（先规划后执行）
/ultraplan          # 云端规划，本地终端保持响应
```

### 技能与插件
```bash
/skills             # 查看所有可用技能
/agents             # 查看/管理子智能体
/plugin list        # 列出已安装插件
/plugin install X   # 安装插件
```

### 记忆与上下文
```bash
/init               # 初始化项目记忆（创建 CLAUDE.md）
/memory             # 编辑记忆文件
/context            # 查看当前上下文使用情况
```

### 调试与帮助
```bash
/help               # 查看帮助
/debug [desc]       # 调试当前会话
/version            # 查看版本信息
/sandbox            # 切换沙箱模式
```

---

## 从命令到 Skills 的迁移

> 旧版 `.claude/commands/` 目录仍可用，但推荐迁移到 Skills

**旧方式（仍可用）：**
```
.claude/commands/review.md
```

**推荐方式（Skills）：**
```
.claude/skills/review/SKILL.md
```

Skills 提供更多能力：目录打包、自动触发、执行隔离、动态上下文注入。

---

## v2.1.101 新变化

- `/fork` 改名为 `/branch`（旧别名保留）
- `/team-onboarding` 新增（为新队友生成入职指南）
- `/ultraplan` 新增（浏览器端规划）
- `/sandbox` 新增（切换沙箱）
- `/model` 选择器现在显示可读标签

---

## 实践建议

1. 先运行 `/help` 熟悉所有命令
2. 用 `/skills` 看有哪些可自动触发的技能
3. 用 `/plan` 处理复杂任务，先看规划再确认执行
4. 用 `/branch` 在尝试危险操作前创建会话分支

---

## 相关模块

- [02-memory.md](02-memory.md) — `/init` 和 `/memory` 命令详解
- [03-skills.md](03-skills.md) — 创建自定义 `/skill-name` 命令
- [08-checkpoints.md](08-checkpoints.md) — `/rewind` 深度使用
