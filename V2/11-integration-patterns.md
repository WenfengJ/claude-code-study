# 模块 11：集成模式（Integration Patterns）

> Level 3 | 预计学习时间：3小时

---

## 模式总览

| 模式 | 组合 | 核心价值 |
|------|------|---------|
| 1. 自动代码质量门控 | Skills + Hooks | 写完代码自动审查、格式化、检测 |
| 2. 并行数据处理 | Subagents + MCP | 多智能体并行拉取和处理外部数据 |
| 3. 大型任务分解 | Planning + Subagents | 先规划再并行执行，避免上下文耗尽 |
| 4. 上下文感知技能 | Memory + Skills | 技能自动读取项目约定 |
| 5. 安全变更管道 | Hooks + Checkpoints | 自动验证 + 自动回滚 |

---

## 模式1：自动代码质量门控

### 场景
每次 Claude 写入或编辑文件后，自动运行 lint、类型检查，并将结果注入上下文让 Claude 自动修复。

### 实现

**.claude/settings.json：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && node scripts/quality-check.js '$TOOL_INPUT'"
          }
        ]
      }
    ]
  }
}
```

**scripts/quality-check.js：**
```javascript
#!/usr/bin/env node
const { execSync } = require('child_process');

let input;
try {
  input = JSON.parse(process.argv[2] || '{}');
} catch (e) {
  process.exit(0);
}

const filePath = input.file_path || '';
if (!filePath || !filePath.match(/\.(ts|js|tsx|jsx)$/)) {
  process.exit(0);
}

const results = [];

// TypeScript 类型检查
try {
  execSync(`npx tsc --noEmit --skipLibCheck 2>&1`, { cwd: process.cwd() });
} catch (e) {
  results.push('TypeScript 错误:\n' + e.stdout?.toString().substring(0, 500));
}

// ESLint
try {
  execSync(`npx eslint "${filePath}" --format compact 2>&1`, { cwd: process.cwd() });
} catch (e) {
  results.push('ESLint 问题:\n' + e.stdout?.toString().substring(0, 500));
}

if (results.length > 0) {
  // 注入上下文让 Claude 自动修复
  console.log(JSON.stringify({
    systemMessage: `质量检查发现问题，请修复：\n${results.join('\n\n')}`
  }));
}
```

**效果：**
```
Claude 写入 src/auth.ts
 → PostToolUse Hook 触发
 → 运行 tsc + eslint
 → 发现 2 个类型错误
 → 注入 systemMessage
 → Claude 自动修复类型错误
 → 再次写入
 → Hook 再次触发
 → 检查通过，继续
```

### 与 Skill 结合

```yaml
---
name: quality-dev
description: 开启质量门控开发模式，所有文件变更自动检查
hooks:
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: "cd $CLAUDE_PROJECT_DIR && npm run lint:fix --silent 2>&1 | tail -5"
---

已开启质量门控模式。
每次文件变更后将自动运行 lint 并修复可自动修复的问题。
```

---

## 模式2：Subagents + MCP 并行数据处理

### 场景
需要从多个外部数据源（GitHub、数据库、Slack）并行收集数据，然后汇总分析。

### 实现

**子智能体配置（.claude/agents/data-collector.md）：**
```yaml
---
name: github-collector
description: 从 GitHub 收集 PR 和 Issue 数据
tools: Read, Bash
mcpServers: github
---

你的任务是从 GitHub 收集指定仓库的数据。
收集完成后，以 JSON 格式返回结果。
```

```yaml
---
name: db-analyst
description: 从数据库查询业务指标数据
tools: Bash
mcpServers: postgres
---

你是数据分析师，从 PostgreSQL 数据库查询业务指标。
所有结果以 JSON 格式返回。
```

**触发并行分析的提示词：**
```
并行执行以下数据收集任务：
1. 用 github-collector 获取本周新增的 PR 和 Issue
2. 用 db-analyst 查询本周的用户注册数和活跃度
3. 完成后汇总成每周报告
```

**效果：**
```
Claude
  ├── → github-collector（独立上下文）
  │     → MCP: github 工具
  │     → 返回 PR/Issue 数据
  │
  └── → db-analyst（独立上下文）
        → MCP: postgres 工具
        → 返回业务指标
  
  ← 汇总两个结果
  ← 生成周报
```

---

## 模式3：Planning + Subagents 大型任务分解

### 场景
需要对大型 Codebase 做全面重构，既要保证方向正确，又不想耗尽主上下文。

### 实现

**步骤1：规划阶段**
```bash
/plan 将整个项目的认证系统从 session-based 迁移到 JWT + Redis
```

**步骤2：分解为独立子任务**

（批准计划后，指示 Claude 分解执行）
```
按照计划，将工作分配给3个并行子智能体：
1. implementer（isolation: worktree）负责实现 JWT 核心逻辑
2. test-engineer 负责编写迁移测试
3. documentation-writer 负责更新 API 文档

各自在独立上下文中完成后汇报结果
```

**工作流结构：**
```
Planning Mode（Plan子智能体研究代码库）
    ↓ 生成计划
用户批准
    ↓
主 Claude 协调
  ├── implementer（worktree: auth-jwt）
  │     - 实现 JWT util
  │     - 修改 auth.service.ts
  │     - 修改 auth.middleware.ts
  │     → 完成，返回变更列表
  │
  ├── test-engineer（worktree: auth-tests）
  │     - 读取新实现
  │     - 编写集成测试
  │     → 完成，返回测试文件
  │
  └── documentation-writer
        - 读取新 API
        - 更新 API 文档
        → 完成，返回文档

主 Claude 汇总所有变更，审查冲突，整合到 main
```

---

## 模式4：Memory + Skills 上下文感知技能

### 场景
技能不仅仅是固定指令，而是能感知项目上下文（技术栈、代码规范）并相应调整行为。

### 实现

**CLAUDE.md 写入项目规范：**
```markdown
## 技术栈
- 语言：TypeScript strict
- 框架：Fastify（不是 Express！）
- 测试：Vitest（不是 Jest！）
- 校验：Zod

## 代码规范
- 错误处理：使用 Result<T, E> 类型（不用 try/catch）
- API 响应：统一 { success, data, error } 格式
- 日志：使用项目的 logger 模块（不用 console.log）
```

**技能读取并应用上下文：**
```yaml
---
name: add-api-endpoint
description: 按照项目规范添加新的 API 端点。当用户要添加路由或 API 时使用。
---

# 添加 API 端点

在添加端点之前，先读取：
1. CLAUDE.md 了解技术栈约定
2. 现有的一个端点文件（了解代码风格）
3. src/types/ 了解可用的类型定义

然后按照以下规范创建端点：
- 使用 Fastify（从 CLAUDE.md 得知）
- 使用 Zod 校验输入
- 返回 { success, data, error } 格式
- 使用 logger 模块（不用 console.log）
- 错误处理用 Result<T, E>
- 在 src/routes/ 对应目录创建文件

最后：为这个端点创建 Vitest 测试（不用 Jest）。
```

**效果：**
```
不同项目中，同一个 add-api-endpoint 技能会：
- 项目A（Express + Jest）→ 按 Express 风格生成，Jest 测试
- 项目B（Fastify + Vitest）→ 按 Fastify 风格生成，Vitest 测试
```

---

## 模式5：Hooks + Checkpoints 安全变更管道

### 场景
对生产代码做高风险变更，需要自动安全检查、失败时自动保护。

### 实现

**.claude/settings.json：**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node scripts/pre-execution-guard.js"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node scripts/post-change-verify.js"
          }
        ]
      }
    ]
  }
}
```

**scripts/pre-execution-guard.js：**
```javascript
// 读取 stdin 的 JSON 输入
process.stdin.resume();
let data = '';
process.stdin.on('data', chunk => data += chunk);
process.stdin.on('end', () => {
  const input = JSON.parse(data || '{}');
  const command = input.tool_input?.command || '';
  
  // 阻止危险命令
  const dangerousPatterns = [
    /rm\s+-rf\s+\/(?!tmp)/,           // 删除根目录（/tmp 除外）
    /DROP\s+TABLE/i,                    // 删除数据库表
    /git\s+push\s+--force.*main/,      // 强推 main
    /kubectl\s+delete\s+namespace/     // 删除 k8s 命名空间
  ];
  
  for (const pattern of dangerousPatterns) {
    if (pattern.test(command)) {
      console.log(JSON.stringify({
        continue: false,
        stopReason: `检测到危险操作：${command}\n如果确实需要执行，请手动运行。`
      }));
      process.exit(2);
    }
  }
});
```

**工作流：**
```
Claude 准备执行 Bash 命令
 → PreToolUse Hook 检查命令安全性
 → 危险：拒绝 + 说明原因 → Claude 调整策略
 → 安全：允许执行

Claude 修改文件
 → PostToolUse Hook 验证
 → 发现问题：注入 systemMessage → Claude 修复
 → 验证通过：继续

用户可以随时：
 → Esc+Esc → 查看检查点
 → 选择任意节点恢复
 → 试不同方案
```

---

## 组合使用建议

### 日常开发（推荐配置）

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "npm run lint 2>&1 | tail -10" }]
      }
    ],
    "SessionStart": [
      {
        "hooks": [{ "type": "command", "command": "echo \"Branch: $(git branch --show-current)\" > $CLAUDE_ENV_FILE" }]
      }
    ]
  }
}
```

```yaml
# .claude/skills/dev-mode/SKILL.md
---
name: dev-mode
description: 开启高效开发模式：自动审查、类型检查、测试
disable-model-invocation: true
---

已进入开发模式：
- 所有文件变更自动运行 lint
- 所有 TypeScript 文件变更自动类型检查
- 完成后自动运行相关测试

开始工作！
```

### 大型项目（推荐配置）

```
技能：
- portability-check（定期可移植性验证）
- security-audit（关键变更安全扫描）
- add-api-endpoint（上下文感知的 API 生成）

智能体：
- code-reviewer（代码变更后自动触发）
- test-engineer（新功能后自动触发）
- debugger（遇到错误时触发）

Hooks：
- SessionStart → 注入 git 状态
- PostToolUse[Write|Edit] → lint + 类型检查
- Stop → 验证任务完成

MCP：
- github（PR/Issue 管理）
- postgres（生产数据查询）
```

---

## 相关模块

- [03-skills.md](03-skills.md) — Skills 技能配置详解
- [04-subagents.md](04-subagents.md) — Subagents 并行执行
- [05-mcp.md](05-mcp.md) — MCP 外部数据接入
- [06-hooks.md](06-hooks.md) — Hooks 事件自动化
- [08-checkpoints.md](08-checkpoints.md) — 检查点安全保护
- [12-real-world-workflows.md](12-real-world-workflows.md) — 完整工作流实例
