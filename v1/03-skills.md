# 模块 3：Skills（技能）

> Level 2 自动化 | 预计学习时间：1小时

---

## 核心概念

Skills 是文件系统上可复用的能力包，让 Claude 在相关任务时自动使用专业知识，无需每次重复说明。

**与普通提示词的区别：**
- 提示词：一次性，每次对话都要重新输入
- Skills：定义一次，Claude 根据场景自动触发

---

## 渐进式加载（核心架构）

Skills 采用三级加载，避免浪费上下文：

| 级别 | 何时加载 | Token 消耗 | 内容 |
|------|---------|------------|------|
| **Level 1：元数据** | 始终（启动时） | ~每个技能 100 tokens | name + description |
| **Level 2：指令** | 技能触发时 | <5k tokens | SKILL.md 正文 |
| **Level 3：资源** | 按需加载 | 实际无限制 | 脚本、模板、参考文档 |

> 安装100个技能也不会浪费上下文，只有被触发的技能才会加载完整内容。

---

## 技能目录结构

```
.claude/skills/<技能名>/
├── SKILL.md           # 主指令文件（必须）
├── templates/         # 模板文件
├── examples/          # 示例输出
├── references/        # 领域知识文档
└── scripts/           # 可执行脚本
```

**存储位置与优先级：**

| 类型 | 位置 | 范围 |
|------|------|------|
| 企业级 | 管理员设置 | 组织所有用户 |
| 个人 | `~/.claude/skills/<name>/SKILL.md` | 个人（所有项目） |
| 项目 | `.claude/skills/<name>/SKILL.md` | 团队（via git） |
| 插件 | `<plugin>/skills/<name>/SKILL.md` | 插件范围 |

---

## SKILL.md 格式

```yaml
---
name: your-skill-name
description: 这个技能做什么，以及何时使用它（关键词触发）
---

# 技能名称

## 使用步骤
...

## 示例
...
```

### 关键 Frontmatter 字段

```yaml
---
name: code-review              # 必须：小写+连字符，最多64字符
description: |                 # 必须：说明功能+触发时机，最多1024字符
  对代码进行安全、性能和质量审查。
  在用户请求代码审查、PR 分析或提到安全检查时使用。
argument-hint: "[filename]"    # 命令补全提示
disable-model-invocation: true # 只有用户能触发（防止 Claude 自动执行危险操作）
user-invocable: false          # 只有 Claude 能自动触发（隐藏出 / 菜单）
allowed-tools: Read, Grep      # 限制工具访问
model: opus                    # 指定使用的模型
effort: high                   # 推理强度
context: fork                  # 在独立子智能体中运行（隔离上下文）
agent: Explore                 # 子智能体类型（配合 context: fork）
paths: "src/api/**/*.ts"       # 仅在这些路径下自动激活
---
```

---

## 触发控制

| Frontmatter 配置 | 用户可触发 | Claude 自动触发 |
|-----------------|-----------|----------------|
| （默认）         | ✅ | ✅ |
| `disable-model-invocation: true` | ✅ | ❌ |
| `user-invocable: false` | ❌ | ✅ |

**使用 `disable-model-invocation: true` 的场景：** `/commit`、`/deploy`、`/send-notification` 等有副作用的操作，不希望 Claude 自动触发。

**使用 `user-invocable: false` 的场景：** 背景知识类技能，如 `legacy-system-context`，给 Claude 提供领域知识，但用户不需要手动调用。

---

## 动态上下文注入

用 `` !`command` `` 在技能内容发送给 Claude 前，先执行 shell 命令并内联输出：

```yaml
---
name: pr-summary
description: 总结 Pull Request 的变更
context: fork
agent: Explore
---

## Pull Request 上下文
- PR diff: !`gh pr diff`
- PR 评论: !`gh pr view --comments`
- 变更文件: !`gh pr diff --name-only`

## 任务
总结这个 Pull Request 的主要变更...
```

---

## 变量替换

| 变量 | 说明 |
|------|------|
| `$ARGUMENTS` | 调用时传入的所有参数 |
| `$0`, `$1`, ... | 按索引访问参数 |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID |
| `${CLAUDE_SKILL_DIR}` | SKILL.md 所在目录 |

```yaml
---
name: fix-issue
description: 修复 GitHub Issue
---

修复 GitHub Issue $ARGUMENTS，遵循项目编码规范：
1. 阅读 issue 描述
2. 实现修复
3. 编写测试
4. 创建提交
```

运行 `/fix-issue 123` 时，`$ARGUMENTS` 会被替换为 `123`。

---

## 实战示例：代码审查技能

```yaml
---
name: code-review-specialist
description: |
  全面的代码审查，包含安全、性能和质量分析。
  当用户请求审查代码、评估 PR 或提到安全分析时使用。
---

# 代码审查技能

## 审查维度
1. **安全性** - 认证/授权、数据暴露、注入漏洞
2. **性能** - 算法复杂度、内存优化、数据库查询
3. **代码质量** - SOLID原则、命名规范、测试覆盖率
4. **可维护性** - 可读性、函数大小（<50行）、圈复杂度

## 输出格式
### 摘要
- 总体质量评分（1-5）
- 关键问题数量
- 优先改进建议

### 严重问题（如有）
- **问题**：描述
- **位置**：文件和行号
- **影响**：为何重要
- **严重性**：严重/高/中
- **修复**：代码示例
```

---

## 在子智能体中运行（context: fork）

```yaml
---
name: deep-research
description: 深入研究某个主题
context: fork
agent: Explore
---

研究 $ARGUMENTS：
1. 用 Glob 和 Grep 查找相关文件
2. 阅读并分析代码
3. 输出含具体文件引用的总结报告
```

子智能体类型选择：

| 类型 | 适用场景 |
|------|---------|
| `Explore` | 只读研究、代码库分析 |
| `Plan` | 创建实施计划 |
| `general-purpose` | 需要所有工具的综合任务 |

---

## 内置技能

| 技能 | 说明 |
|------|------|
| `/simplify` | 审查修改过的代码，生成3个并行审查智能体 |
| `/batch <指令>` | 使用 git worktrees 跨代码库大规模并行修改 |
| `/debug [描述]` | 读取 debug 日志排查当前 session 问题 |
| `/loop [间隔] <提示>` | 按间隔重复执行（如 `/loop 5m 检查部署状态`） |
| `/claude-api` | 加载 Claude API/SDK 参考文档 |

---

## 最佳实践

1. **描述要具体**：包含触发关键词，如"当用户提到 Excel 文件或 .xlsx 时"
2. **一个技能一个职责**：不要把太多功能塞入一个技能
3. **SKILL.md 控制在500行以内**：超出部分放到独立文件，用相对路径引用
4. **部署副作用操作时加 `disable-model-invocation: true`**
5. **背景知识类技能用 `user-invocable: false`**

---

## 相关模块

- [01-slash-commands.md](01-slash-commands.md) — `/skill-name` 调用方式
- [04-subagents.md](04-subagents.md) — `context: fork` 的子智能体机制
- [07-plugins.md](07-plugins.md) — 将技能打包到插件分发
