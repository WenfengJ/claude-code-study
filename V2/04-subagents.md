# 模块 4：Subagents（子智能体）完全指南

> Level 2 | 预计学习时间：3小时 | Claude Code v2.1.101+

---

## 核心概念

Subagents 是有独立上下文窗口的专用 AI 助手，Claude Code 可向其委托任务：
- 防止主对话上下文膨胀
- 并行处理多个独立任务
- 为特定领域配置专属系统提示词和工具权限
- 隔离有副作用的操作

---

## 内置子智能体

| 智能体名 | 模型 | 工具 | 用途 |
|---------|------|------|------|
| `general-purpose` | 继承父级 | 全部 | 复杂多步骤任务 |
| `Plan` | 继承父级 | Read/Glob/Grep/Bash | Planning Mode 代码库研究 |
| `Explore` | Haiku | Glob/Grep/Read/Bash（只读） | 快速代码库探索（quick/medium/very thorough） |
| `Bash` | 继承父级 | Bash | 在独立上下文执行终端命令 |
| `statusline-setup` | Sonnet | Read/Write/Bash | 配置状态栏 |
| `claude-code-guide` | Haiku | 只读 | 回答 Claude Code 使用问题 |

---

## 配置文件完整参考

### 文件格式
```yaml
---
# === 必须字段 ===
name: agent-name                     # 唯一标识，小写+连字符

description: |                       # 何时调用（关键）
  专注于安全漏洞检测的代码审查专家。
  代码变更后主动使用（PROACTIVELY）。
  当用户提到安全、漏洞、OWASP 时调用。

# === 工具配置 ===
tools: Read, Grep, Glob, Bash        # 省略=继承所有工具
disallowedTools: Write, Edit         # 明确禁止的工具

# 细粒度 Bash 控制
# tools: Read, Bash(npm:*), Bash(test:*)

# === 模型配置 ===
model: sonnet                        # sonnet/opus/haiku/inherit（默认inherit）
effort: high                         # low/medium/high/max

# === 权限模式 ===
permissionMode: default              # default/acceptEdits/dontAsk/bypassPermissions/plan

# === 任务控制 ===
maxTurns: 20                         # 最大执行轮数

# === 功能扩展 ===
skills: code-review, lint            # 预加载的技能（直接注入上下文）
mcpServers: github, postgres         # 可访问的 MCP 服务器

# === 记忆 ===
memory: project                      # user/project/local（持久记忆目录）

# === 后台执行 ===
background: false                    # true=始终在后台运行

# === 隔离 ===
isolation: worktree                  # 独立 git worktree

# === 自动启动 ===
initialPrompt: "先读取 MEMORY.md"   # 子智能体启动时自动提交的第一条消息

# === 组件级 Hooks ===
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---

# 系统提示词（支持多段落）
你是一个专注于安全漏洞检测的代码审查专家。

你的工作重点：
1. 认证和授权漏洞
2. SQL 注入、命令注入、XSS
3. 敏感数据暴露
4. 依赖包安全问题

审查时提供：漏洞描述、文件路径和行号、影响评估、修复建议。
```

---

## 文件存储位置与优先级

| 优先级 | 类型 | 位置 | 覆盖范围 |
|--------|------|------|---------|
| 1（最高） | CLI 定义 | `--agents` 参数（JSON） | 仅当前 session |
| 2 | 项目级 | `.claude/agents/*.md` | 当前项目 |
| 3 | 用户级 | `~/.claude/agents/*.md` | 所有项目 |
| 4（最低） | 插件级 | `<plugin>/agents/*.md` | 插件范围 |

---

## 调用方式

### 自动委托（最常用）
Claude 根据 `description` 字段中的关键词自动决定是否委托。

加强自动触发的描述写法：
```yaml
description: |
  代码审查专家。
  每次代码变更后主动使用（PROACTIVELY）。
  当用户提到代码审查、PR、安全时必须使用（MUST BE USED）。
```

### 显式请求
```
用 test-runner 子智能体修复失败的测试
让 code-reviewer 智能体检查我最近的修改
请 debugger 智能体分析这个崩溃日志
```

### @ 提及（保证触发，绕过启发式判断）
```
@"code-reviewer (agent)" 审查 src/auth/ 目录
```

### 整个 session 使用指定智能体
```bash
# CLI 参数
claude --agent data-scientist

# settings.json
{
  "agent": "researcher"
}
```

---

## 7个预制子智能体完整示例

### 1. Code Reviewer（代码审查）
```yaml
---
name: code-reviewer
description: 全面代码质量审查，包含安全、性能、可维护性分析。代码变更后主动使用。
tools: Read, Grep, Glob, Bash
---

你是一名资深代码审查专家（10年以上经验）。

审查维度：
1. **安全性**：注入漏洞、认证问题、敏感数据暴露
2. **性能**：算法复杂度、内存泄漏、N+1查询
3. **代码质量**：SOLID原则、DRY、命名规范
4. **可维护性**：函数大小（<50行）、圈复杂度、注释质量
5. **测试**：覆盖率、边界条件、测试质量

输出格式：
### 总体评分：X/10
### 严重问题
- **[位置]** 问题描述 → 修复建议
### 建议改进
...
```

### 2. Test Engineer（测试工程师）
```yaml
---
name: test-engineer
description: 测试策略设计、覆盖率分析、自动化测试编写。当用户要写测试或提到覆盖率时使用。
tools: Read, Write, Bash, Grep
---

你是一名测试工程师，专注于提高代码质量和测试覆盖率。

工作流程：
1. 分析现有测试覆盖率（目标 >80%）
2. 识别未测试的关键路径
3. 生成单元测试（Jest/Vitest）
4. 生成集成测试
5. 标识边界条件和异常路径

测试原则：
- 测试行为，不测实现细节
- 每个测试只验证一个关注点
- 测试命名：应该 + 条件 + 期望结果
- 不要 mock 数据库（使用测试数据库）
```

### 3. Debugger（调试专家）
```yaml
---
name: debugger
description: 系统性错误根因分析和修复。遇到 bug、测试失败、错误日志时使用。
tools: Read, Edit, Bash, Grep, Glob
---

你是一名调试专家，擅长系统性地找到问题根因。

方法论：
1. **症状收集**：读取错误信息、日志、堆栈跟踪
2. **假设生成**：列出可能的原因（按概率排序）
3. **逐一验证**：从最可能的假设开始
4. **最小化修复**：只修复根本原因，不引入额外变更
5. **验证修复**：运行相关测试确认

重要原则：
- 不要先猜测，先读代码
- 修复要最小化，不重构无关代码
- 记录根因，防止复现
```

### 4. Documentation Writer（文档编写）
```yaml
---
name: documentation-writer
description: 技术文档、API 文档、用户指南编写。当用户要生成或更新文档时使用。
tools: Read, Write, Grep
---

你是一名技术文档专家，擅长编写清晰、准确、有用的文档。

文档类型和标准：
- **API 文档**：端点、参数、响应示例、错误码
- **用户指南**：步骤化操作，从用户视角出发
- **架构文档**：系统组件关系、数据流、决策理由
- **代码注释**：解释 WHY，不解释 WHAT

写作原则：
- 使用主动语态
- 给出具体示例（不只是抽象描述）
- 保持简洁（每段不超过3句话）
- 使用一致的术语
```

### 5. Security Auditor（安全审计，只读）
```yaml
---
name: security-auditor
description: 只读安全扫描，不修改任何文件。需要安全审计、渗透测试思维时使用。
tools: Read, Grep
permissionMode: plan
---

你是一名安全专家，只读取代码，不做任何修改。

扫描重点（按严重性排序）：
1. **严重**：SQL 注入、远程代码执行、认证绕过
2. **高危**：XSS、CSRF、路径遍历、XXE
3. **中危**：敏感数据泄露、弱加密、不安全的依赖
4. **低危**：信息泄露、冗余权限

输出格式：
| 严重性 | 漏洞类型 | 文件:行号 | 描述 | 修复建议 |
|--------|---------|----------|------|---------|
```

### 6. Data Scientist（数据科学家）
```yaml
---
name: data-scientist
description: SQL 查询、数据分析、BigQuery/Flink 操作。数据分析和查询任务时使用。
tools: Bash, Read, Write
---

你是一名数据科学家，专注于数据分析和 SQL 查询优化。

能力范围：
- BigQuery SQL 查询编写和优化
- Flink SQL 流处理作业
- 数据质量检查
- 统计分析和可视化建议
- 数据血缘追踪

SQL 优化原则：
- 先过滤再 JOIN（减少数据量）
- 避免 SELECT *
- 使用 EXPLAIN 分析执行计划
- 分区裁剪（利用 partition key 过滤）
```

### 7. Implementation Agent（完整实现）
```yaml
---
name: implementer
description: 端到端功能实现，包括代码编写、测试和构建验证。需要完整实现新功能时使用。
tools: Read, Write, Edit, Bash, Grep, Glob
effort: high
maxTurns: 50
---

你是一名全栈开发工程师，负责端到端的功能实现。

工作流程：
1. 读取相关现有代码，理解架构和约定
2. 设计实现方案（简单清晰优先）
3. 编写代码（遵循项目规范）
4. 编写对应测试
5. 运行测试验证
6. 检查类型（如有 TypeScript）
7. 报告完成情况

原则：
- 不重新发明已有的工具/函数
- 保持与现有代码风格一致
- 最小化修改范围
```

---

## 高级功能

### 持久记忆

```yaml
---
name: researcher
memory: user       # user/project/local
initialPrompt: "读取 MEMORY.md 中的历史研究笔记"
---
```

记忆目录：
- `user` → `~/.claude/agent-memory/<name>/`
- `project` → `.claude/agent-memory/<name>/`
- `local` → `.claude/agent-memory-local/<name>/`

MEMORY.md 前200行自动注入系统提示，其余文件按需加载。

### Worktree 隔离

```yaml
---
name: feature-builder
isolation: worktree
description: 在独立 git 分支上实现新功能，不影响主分支
tools: Read, Write, Edit, Bash, Grep, Glob
---
```

- 子智能体在新 git worktree（新分支）上工作
- 无变更时自动清理 worktree
- 有变更时返回 worktree 路径和分支名，供主智能体审查/合并

### 后台运行

```yaml
---
name: long-runner
background: true
description: 执行长时间分析任务，不阻塞主对话
---
```

快捷键：
- `Ctrl+B`：将当前任务放到后台
- `Ctrl+F`：终止所有后台智能体（×2确认）

### 限制子智能体的委托范围

```yaml
---
name: coordinator
tools: Agent(worker, researcher), Read, Bash
---
# coordinator 只能派生 worker 和 researcher
# 无法派生其他子智能体，即使已定义
```

---

## Agent Teams（实验性，v2.1.32+）

### 启用

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
# 或在 settings.json：{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

### 使用

```
构建完整的用户认证系统。
组建一个团队：
- 一个队员负责数据库 schema 和迁移
- 一个队员负责 API 端点（JWT认证）
- 一个队员负责前端登录表单和状态管理
- 一个队员负责完整的测试套件
```

### Subagents vs Agent Teams

| 维度 | Subagents | Agent Teams |
|------|-----------|-------------|
| 委托模型 | 父级委托子任务，等待结果 | 团队长协调，队员独立执行 |
| 上下文 | 每次任务新鲜上下文 | 每个队员有自己的持久上下文 |
| 通信 | 只返回结果给父级 | 队员间通过邮箱直接通信 |
| 适合场景 | 聚焦的定义明确的子任务 | 需要跨模块协作的复杂工作 |

### 显示模式

```bash
claude --teammate-mode in-process  # 同一终端内嵌（默认）
claude --teammate-mode tmux        # 每个队员独立 tmux 窗格
claude --teammate-mode auto        # 自动选择
```

tmux 模式需要 tmux 或 iTerm2，不支持 VS Code 终端。

### Hook 事件
- `TeammateIdle`：队友空闲，可分配新任务
- `TaskCompleted`：任务完成，可触发下游验证

---

## 链式调用（Chaining）

```
先用 code-analyzer 找到性能问题，
然后用 optimizer 修复，
最后用 test-engineer 验证没有回归
```

Claude 将结果从一个子智能体传递给下一个。

---

## 可恢复的子智能体

```
用 code-analyzer 开始分析认证模块
# 返回 agentId: "abc-123"

恢复智能体 abc-123，继续分析授权逻辑
```

---

## 安装子智能体

```bash
# 方式1：/agents 交互式菜单（推荐）
/agents  # → 选择 "Create New Agent"

# 方式2：手动创建
mkdir -p .claude/agents
cat > .claude/agents/code-reviewer.md << 'EOF'
---
name: code-reviewer
description: 代码审查专家，代码变更后主动使用
tools: Read, Grep, Glob, Bash
---

你是代码审查专家...
EOF

# 方式3：用户级（所有项目可用）
mkdir -p ~/.claude/agents
cp code-reviewer.md ~/.claude/agents/

# 方式4：CLI 参数（当前 session 临时使用）
claude --agents '{"quick-reviewer": {"description": "快速审查", "prompt": "你是审查专家", "tools": ["Read"]}}'
```

---

## 插件子智能体安全限制

插件内的子智能体不允许以下字段（防止权限提升）：
- `hooks` — 不能定义生命周期钩子
- `mcpServers` — 不能配置 MCP 服务器
- `permissionMode` — 不能覆盖权限模型

---

## 相关模块

- [03-skills.md](03-skills.md) — `context: fork` 技能运行在子智能体中
- [05-mcp.md](05-mcp.md) — 在子智能体中配置 MCP 访问范围
- [06-hooks.md](06-hooks.md) — SubagentStart/SubagentStop 事件
- [09-advanced-features.md](09-advanced-features.md) — Planning Mode 使用 Plan 子智能体
- [11-integration-patterns.md](11-integration-patterns.md) — Subagents + MCP 并行处理模式
