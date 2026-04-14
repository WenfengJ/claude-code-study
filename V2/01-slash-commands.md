# 模块 1：Slash Commands（斜杠命令）完全指南

> Level 1 | 预计学习时间：2小时 | Claude Code v2.1.101+

---

## 四种命令类型

| 类型 | 来源 | 命名空间 | 示例 |
|------|------|---------|------|
| **内置命令** | Claude Code 原生 | 无前缀 | `/help`、`/model` |
| **Skills** | `SKILL.md` 文件 | 无前缀 | `/code-review`、`/deploy` |
| **Plugin 命令** | 已安装插件 | `plugin:name` | `/pr-review:check` |
| **MCP prompts** | MCP 服务器 | `mcp:name` | - |

---

## 完整内置命令参考

### 会话控制
| 命令 | 说明 | 示例 |
|------|------|------|
| `/clear` | 清除当前会话历史 | `/clear` |
| `/branch [name]` | 创建会话分支（原 `/fork`，别名保留） | `/branch try-redis` |
| `/rewind` | 打开检查点浏览器 | `/rewind` |
| `/checkpoint` | `/rewind` 的别名 | `/checkpoint` |
| `/export [filename]` | 导出对话到文件 | `/export session.md` |
| `/resume` | 恢复上次会话 | `/resume` |
| `/compact` | 手动压缩对话历史（释放上下文） | `/compact` |

### 模型与性能
| 命令 | 说明 | 示例 |
|------|------|------|
| `/model [model]` | 切换模型，显示可读标签 | `/model opus` |
| `/effort [level]` | 设置推理强度 | `/effort max` |
| `/fast` | 切换 Fast Mode（更快输出，同等模型） | `/fast` |
| `/sandbox` | 切换沙箱模式 | `/sandbox` |

**effort 级别说明：**
- `low` - 快速、简单任务
- `medium` - 默认，均衡
- `high` - 复杂推理
- `max` - 最深度推理，最慢
- `auto` - Claude 自动选择

### 计划与任务
| 命令 | 说明 | 示例 |
|------|------|------|
| `/plan [description]` | 进入 Planning Mode | `/plan 重构认证模块` |
| `/ultraplan` | 浏览器端规划，本地终端保持响应 | `/ultraplan` |

### 技能与智能体
| 命令 | 说明 | 示例 |
|------|------|------|
| `/skills` | 列出所有可用技能（内置+自定义） | `/skills` |
| `/agents` | 交互式管理子智能体 | `/agents` |

### 记忆管理
| 命令 | 说明 | 示例 |
|------|------|------|
| `/init` | 初始化项目记忆（创建 CLAUDE.md） | `/init` |
| `/memory` | 在编辑器中打开记忆文件 | `/memory` |
| `/context` | 查看当前上下文使用情况 | `/context` |

### 插件管理
| 命令 | 说明 | 示例 |
|------|------|------|
| `/plugin list` | 列出已安装插件 | `/plugin list` |
| `/plugin install X` | 安装插件 | `/plugin install pr-review` |
| `/plugin enable X` | 启用插件 | `/plugin enable code-standards` |
| `/plugin disable X` | 禁用插件 | `/plugin disable legacy-tools` |
| `/plugin uninstall X` | 卸载插件 | `/plugin uninstall old-plugin` |
| `/reload-plugins` | 热重载插件（开发时用） | `/reload-plugins` |

### 调试与信息
| 命令 | 说明 | 示例 |
|------|------|------|
| `/help` | 查看帮助（含所有命令列表） | `/help` |
| `/debug [description]` | 读取 debug 日志排查问题 | `/debug hooks not running` |
| `/version` | 查看 Claude Code 版本 | `/version` |

### 团队协作（v2.1.101 新增）
| 命令 | 说明 | 示例 |
|------|------|------|
| `/team-onboarding` | 为新队友生成入职指南 | `/team-onboarding` |

---

## 内置 Skills（开箱即用）

以下技能无需安装，随时可用：

| 命令 | 说明 | 自动触发？ |
|------|------|-----------|
| `/simplify` | 审查变更代码的质量和效率，生成3个并行审查智能体 | 否（手动） |
| `/batch <指令>` | 使用 git worktrees 跨代码库大规模并行修改 | 否（手动） |
| `/debug [描述]` | 读取 debug 日志排查 session 问题 | 否（手动） |
| `/loop [间隔] <提示>` | 按间隔重复执行任务 | 否（手动） |
| `/claude-api` | 加载 Claude API/SDK 参考，在 import anthropic 时自动激活 | ✅ 是 |

---

## 自定义命令（旧版）vs Skills（推荐）

### 旧版自定义命令
```
.claude/commands/review.md    → /review
.claude/commands/deploy.md    → /deploy
```
- 仍然可用
- 不支持自动触发、子智能体执行、动态上下文注入

### 推荐：迁移到 Skills
```
.claude/skills/review/SKILL.md    → /review（也可自动触发）
.claude/skills/deploy/SKILL.md    → /deploy（可设为只有用户能触发）
```

**迁移步骤：**
```bash
# 1. 为技能创建目录
mkdir -p .claude/skills/my-command

# 2. 移动内容（添加 YAML frontmatter）
cat > .claude/skills/my-command/SKILL.md << 'EOF'
---
name: my-command
description: 原来命令的功能描述，加上触发关键词
---

# 原命令内容
...
EOF

# 3. 删除旧文件（可选，两者可共存，Skills 优先）
rm .claude/commands/my-command.md
```

---

## /init 深度使用

```bash
/init
# 基础初始化，在项目根目录创建 CLAUDE.md

CLAUDE_CODE_NEW_INIT=1 claude
/init
# 增强版：多阶段交互式流程，逐步引导配置
```

`/init` 生成的 CLAUDE.md 模板结构：
```markdown
# 项目配置

## 项目概述
- 名称：
- 技术栈：
- 团队规模：

## 开发规范
## 常用命令
## 已知问题
```

---

## /loop 定时重复执行

```bash
/loop 5m 检查部署状态并汇报
/loop 1h 审查今天的代码变更并生成摘要
/loop 30s 运行测试套件直到全部通过
```

格式：`/loop [间隔] [任务描述]`
- 间隔支持：`s`（秒）、`m`（分钟）、`h`（小时）
- 省略间隔：Claude 自动决定执行节奏

---

## /plan 规划模式详解

```bash
/plan 重构用户认证模块，迁移到 JWT

# Claude 进入只读模式：
# 1. Plan 子智能体研究代码库
# 2. 生成带步骤的实施计划
# 3. 等待你确认
# 4. 批准后开始执行
```

与直接要求 Claude 执行的区别：
- Planning Mode：先看计划，确认方向后执行，降低大改错误的风险
- 直接执行：快速但可能走偏方向

---

## v2.1.101 版本变化汇总

| 变化 | 说明 |
|------|------|
| `/fork` → `/branch` | 原别名保留，新名称更清晰 |
| `/team-onboarding` 新增 | 自动为新队友生成项目入职文档 |
| `/ultraplan` 新增 | 浏览器端规划，终端不被占用 |
| `/sandbox` 新增 | 切换沙箱隔离模式 |
| `/model` 改进 | 选择器现在显示人类可读的标签 |

---

## 实践场景示例

### 场景1：开始一个新功能
```bash
/plan 实现用户头像上传功能（支持 PNG/JPEG，最大5MB，存 S3）
# → 审查计划 → 批准 → 执行
```

### 场景2：实验后悔了
```bash
/branch before-redis    # 创建分支（记录当前状态）
# 开始修改...
# 发现方向不对
Esc + Esc              # 打开检查点
# 选择 before-redis 节点 → 恢复代码和对话
```

### 场景3：定期代码审查
```bash
/loop 1h 检查 git diff --stat，如果有变更就做代码审查并汇报问题
```

### 场景4：跨模块大修改
```bash
/batch 将所有 .js 文件中的 var 替换为 const 或 let（根据是否重新赋值判断）
```

---

## 相关模块

- [03-skills.md](03-skills.md) — 创建自定义 `/skill-name` 命令的完整指南
- [08-checkpoints.md](08-checkpoints.md) — `/rewind` 和检查点工作流深度使用
- [02-memory.md](02-memory.md) — `/init` 和 `/memory` 命令详解
- [09-advanced-features.md](09-advanced-features.md) — `/plan` 和 Planning Mode 深度使用
