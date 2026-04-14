# 模块 13：故障排查与调试指南

> Level 2 | 预计学习时间：2小时 | Claude Code v2.1.101+

---

## 快速诊断索引

| 症状 | 跳转 |
|------|------|
| Skills 命令不触发 | [Skills 排查](#1-skills-不触发排查) |
| Hooks 没有执行 | [Hooks 排查](#2-hooks-不执行排查) |
| MCP 连接失败 | [MCP 排查](#3-mcp-连接失败排查) |
| 子智能体权限错误 | [Subagents 排查](#4-子智能体权限问题) |
| 速率限制被触发 | [速率限制应对](#5-速率限制应对策略) |
| 上下文耗尽 | [上下文管理](#6-上下文耗尽处理) |
| 输出质量下降 | [质量恢复](#7-响应质量下降处理) |
| 文件修改不符合预期 | [文件操作排查](#8-文件操作问题排查) |

---

## 1. Skills 不触发排查

### 症状
- 输入 `/skill-name` 后没有响应
- Claude 不理解技能指令
- 技能描述似乎没被读取

### 诊断步骤

```bash
# 检查技能文件是否存在
ls .claude/skills/

# 检查文件结构是否正确
ls .claude/skills/your-skill/
# 必须有 SKILL.md 文件

# 检查 YAML frontmatter 格式
head -10 .claude/skills/your-skill/SKILL.md
```

### 常见原因和修复

**原因1：YAML frontmatter 格式错误**

```yaml
# ❌ 错误：name 字段有大写或特殊字符
---
name: MySkill
---

# ✅ 正确：小写字母和连字符
---
name: my-skill
---
```

**原因2：文件放在错误目录**

```
❌ 错误路径：
.claude/my-skill/SKILL.md       # 缺少 skills/ 层级
.claude/skills/SKILL.md         # 技能文件不能直接放在 skills/

✅ 正确路径：
.claude/skills/my-skill/SKILL.md
```

**原因3：description 为空或太模糊**

```yaml
# ❌ 没有 description，Claude 不知道何时调用
---
name: security-check
---

# ✅ 清晰的触发描述
---
name: security-check
description: 安全审查工具。当用户说"安全检查"、"security check"、"审查安全"时使用。
---
```

**原因4：`disable-model-invocation: true` 但没有配置触发词**

```yaml
# disable-model-invocation: true 时，只有斜杠命令触发
# 确保用 /skill-name 而非自然语言调用
---
name: deploy
disable-model-invocation: true  # 必须用 /deploy 触发
---
```

**原因5：字符预算不足**

```bash
# 检查是否超过字符预算（默认 8000 字符）
wc -c .claude/skills/*/SKILL.md

# 如果总量太大，调整预算
export SLASH_COMMAND_TOOL_CHAR_BUDGET=15000
```

### 调试方法

```bash
# 查看 Claude 加载了哪些技能（在交互模式中）
/help     # 列出所有可用命令，包括自定义技能

# 强制重新加载技能
# 退出并重新启动 Claude Code
claude
```

---

## 2. Hooks 不执行排查

### 症状
- 文件保存后 lint 没有运行
- PreToolUse 没有阻止危险命令
- PostToolUse 的 systemMessage 没有注入

### 诊断步骤

**步骤1：确认 settings.json 位置和格式**

```bash
# 检查项目级配置
cat .claude/settings.json

# 检查全局配置
cat ~/.claude/settings.json

# 验证 JSON 格式
python3 -m json.tool .claude/settings.json
```

**步骤2：检查 Hook 配置结构**

```json
{
  "hooks": {
    "PostToolUse": [          // ← 正确的事件名
      {
        "matcher": "Write",   // ← 工具名
        "hooks": [            // ← 注意是数组
          {
            "type": "command",
            "command": "echo test"
          }
        ]
      }
    ]
  }
}
```

### 常见原因和修复

**原因1：事件名大小写错误**

```json
// ❌ 错误：小写
{ "hooks": { "posttooluse": [...] } }

// ✅ 正确：PascalCase
{ "hooks": { "PostToolUse": [...] } }
```

**完整的正确事件名：**
```
PreToolUse    PostToolUse
Stop          SubagentStop
Notification  UserPromptSubmit
SessionStart  SessionEnd
```

**原因2：matcher 与工具名不匹配**

```json
// ❌ 错误：工具名有空格或大小写错误
{ "matcher": "write" }
{ "matcher": "edit " }

// ✅ 正确：精确工具名
{ "matcher": "Write" }
{ "matcher": "Write|Edit" }    // 多个用 | 分隔
{ "matcher": "Bash(npm:*)" }  // 限定子命令
```

**原因3：命令脚本有语法错误**

```bash
# 测试 Hook 命令是否能独立运行
echo '{"tool_input":{"file_path":"test.ts"}}' | node scripts/quality-check.js

# 检查脚本权限（仅 macOS/Linux）
ls -la scripts/quality-check.js
chmod +x scripts/quality-check.js
```

**原因4：Hook 返回格式错误**

```javascript
// ❌ 错误：返回非 JSON
console.log("发现错误");

// ✅ 正确：systemMessage 格式
console.log(JSON.stringify({
  systemMessage: "发现错误：请修复"
}));

// ✅ 阻止执行格式
console.log(JSON.stringify({
  continue: false,
  stopReason: "危险操作被阻止"
}));
process.exit(2);  // exit code 2 = 阻止
```

**原因5：stdout/stderr 混淆**

```javascript
// ❌ 错误：systemMessage 写到 stderr
process.stderr.write(JSON.stringify({systemMessage: "..."}));

// ✅ 正确：必须写到 stdout
process.stdout.write(JSON.stringify({systemMessage: "..."}));
// 或
console.log(JSON.stringify({systemMessage: "..."}));
```

### Hook 调试模式

```bash
# 添加调试输出到 Hook 脚本
process.stdin.on('end', () => {
  const input = JSON.parse(data || '{}');
  
  // 调试：写到文件（不影响 stdout）
  require('fs').appendFileSync('/tmp/hook-debug.log', 
    JSON.stringify({input, time: new Date()}, null, 2) + '\n'
  );
  
  // 正常处理...
});

# 实时查看调试日志
tail -f /tmp/hook-debug.log
```

---

## 3. MCP 连接失败排查

### 症状
- `claude mcp list` 显示服务器状态为 error
- 工具调用返回 "MCP server not available"
- OAuth 授权后仍然报错

### 诊断步骤

```bash
# 查看所有 MCP 服务器状态
claude mcp list

# 查看特定服务器详情
claude mcp get server-name

# 测试连接
claude mcp test server-name
```

### 常见原因和修复

**原因1：.mcp.json 格式错误**

```json
// ❌ 常见错误：args 不是数组
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": "-y @modelcontextprotocol/server-filesystem /tmp"  // 字符串！
    }
  }
}

// ✅ 正确：args 必须是数组
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    }
  }
}
```

**原因2：依赖包未安装**

```bash
# 检查 MCP 服务器包是否可用
npx @modelcontextprotocol/server-filesystem --version

# 如果报错，手动安装
npm install -g @modelcontextprotocol/server-filesystem
```

**原因3：stdio MCP 服务器进程崩溃**

```bash
# 直接运行 MCP 服务器查看错误
npx @modelcontextprotocol/server-github 2>&1

# 检查环境变量是否设置
echo $GITHUB_PERSONAL_ACCESS_TOKEN
```

**原因4：HTTP MCP 服务器 URL 错误**

```json
{
  "mcpServers": {
    "my-api": {
      "type": "http",
      "url": "http://localhost:3000/mcp"  // 确认端口和路径正确
    }
  }
}

// 测试 HTTP 端点
curl http://localhost:3000/mcp/health
```

**原因5：OAuth 未完成**

```bash
# 重新触发 OAuth 流程
claude mcp auth server-name

# 清除已保存的令牌重新授权
claude mcp auth --reset server-name
```

**原因6：配置文件作用域问题**

```bash
# 本地 .mcp.json（仅当前项目）
cat .mcp.json

# 全局配置（所有项目）
cat ~/.claude/mcp.json

# 工作区配置（当前目录）
cat .claude/mcp.json

# 检查优先级：本地 > 工作区 > 全局
```

---

## 4. 子智能体权限问题

### 症状
- 子智能体报告 "Permission denied"
- 子智能体无法使用某些工具
- 后台任务被自动拒绝

### 诊断步骤

```bash
# 检查子智能体配置
cat .claude/agents/agent-name.md

# 查看头部 YAML
head -15 .claude/agents/agent-name.md
```

### 常见原因和修复

**原因1：后台子智能体没有预批准工具**

```yaml
# ❌ 错误：background 任务需要预批准所有工具
---
name: background-worker
background: true
# 没有指定 tools！
---

# ✅ 正确：明确列出所有需要的工具
---
name: background-worker
background: true
tools: Read, Grep, Glob, Write, Bash
---
```

**原因2：工具名拼写错误**

```yaml
# ❌ 错误：工具名大小写错误
tools: read, grep, bash

# ✅ 正确：工具名区分大小写
tools: Read, Grep, Bash
```

**完整工具名列表：**
```
Read, Write, Edit, MultiEdit
Bash, Glob, Grep
Agent, Task
WebFetch, WebSearch
NotebookEdit
```

**原因3：尝试使用未列出的 MCP 工具**

```yaml
# ❌ 只列出了 tools，但需要 MCP 工具
---
name: github-agent
tools: Read, Bash
# 缺少 MCP 配置！
---

# ✅ 同时配置 tools 和 mcpServers
---
name: github-agent
tools: Read, Bash
mcpServers: github
---
```

**原因4：worktree 隔离的权限问题**

```yaml
# worktree 隔离的子智能体默认没有网络工具
---
name: isolated-builder
isolation: worktree
tools: Read, Write, Edit, Bash  # 明确列出需要的工具
---
```

**原因5：父进程的工具黑名单影响子智能体**

```bash
# 如果主 Claude 启动时用了 --disallowedTools
claude --disallowedTools "Bash"
# 子智能体继承这个限制！

# 解决：在 Claude 启动时不添加不必要的限制
claude  # 正常启动，子智能体配置控制自身权限
```

---

## 5. 速率限制应对策略

### 触发原因
- 单次请求 token 过多（大文件、大 diff）
- 短时间内请求过于频繁
- 同时运行过多子智能体

### 立即应对

```bash
# 等待后重试（速率限制通常持续 60-300 秒）
# Claude Code 会自动重试，你也可以手动等待后继续

# 查看当前请求大小
# Esc+Esc → 检查对话长度
/context   # 查看上下文使用量
```

### 预防策略

**策略1：分批次处理大型任务**

```bash
# ❌ 一次提交所有工作
"重构认证系统、添加缓存层、更新所有测试、同步文档"

# ✅ 分多个 session 处理
Session 1: "重构认证系统"
Session 2: "添加缓存层"  
Session 3: "更新所有测试"
Session 4: "同步文档"
```

**策略2：截断大型输入**

```bash
# ❌ 直接发送大文件
cat huge-file.ts | claude -p "分析这个文件"

# ✅ 截断到合理大小
head -200 huge-file.ts | claude -p "分析前200行，关注关键逻辑"

# ✅ 大型 diff 截断
git diff | head -300 | claude -p "审查这些变更的关键部分"
```

**策略3：使用 Print Mode 处理机械性任务**

```bash
# Print Mode 每次独立请求，不累积上下文
for file in src/*.ts; do
  echo "处理: $file"
  cat "$file" | claude -p "生成 JSDoc 注释，仅输出修改后的内容" > "/tmp/$(basename $file)"
  sleep 2  # 请求间隔
done
```

**策略4：使用 --max-spending 限制**

```bash
# 防止意外超额消费
claude --max-spending 2.00

# 限制最大轮数
claude --max-turns 20
```

**策略5：合理使用子智能体**

```
✅ 子智能体适合并行的独立任务
❌ 不要为了绕过速率限制而滥用子智能体
（过多并行子智能体会加速触发速率限制）

建议：同时运行 2-4 个子智能体，不要超过 8 个
```

---

## 6. 上下文耗尽处理

### 信号识别

```
⚠️ 警告信号：
- Claude 开始重复已经说过的内容
- 忽略之前确认过的约束
- 回答质量明显下降
- /context 显示使用率 > 80%
- 对话超过 80-100 轮
```

### 处理方法

**方法1：摘要压缩（推荐）**

```bash
# Esc+Esc → 选择合适的检查点 → "从这里摘要"
# 将旧消息压缩为摘要，释放上下文空间
# 文件不受影响，可以继续工作
```

**方法2：新 Session 继续**

```bash
# 保存当前工作状态
git add -A && git commit -m "wip: checkpoint before context reset"

# 导出关键信息
/export session-notes.md

# 开启新 session，注入关键上下文
claude -c   # 继续上次 session（有摘要）
# 或
claude      # 全新 session，手动提供背景
```

**方法3：主动规划阶段**

```
长期任务建议：
1. 每 20-30 轮对话，做一次 git commit
2. 大阶段切换时，开新 session
3. 使用 CLAUDE.md 保存项目上下文（不占对话空间）
4. 使用 /branch 标记重要节点
```

**方法4：CLAUDE.md 承载长期上下文**

```markdown
<!-- CLAUDE.md：将项目上下文写入文件，不占对话空间 -->

## 当前重构进度（2026-04-13）
- ✅ 完成：auth 模块 JWT 迁移
- ✅ 完成：Redis 配置
- 🚧 进行中：中间件更新（src/middleware/auth.middleware.ts）
- ⏳ 待做：集成测试
- ⏳ 待做：文档更新
```

---

## 7. 响应质量下降处理

### 常见质量问题

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 忽略代码风格规范 | CLAUDE.md 未加载 | 检查文件位置 |
| 使用错误框架 | 技术栈未在 CLAUDE.md 声明 | 更新项目规范 |
| 生成不必要的代码 | 指令太模糊 | 更具体的提示词 |
| 重复已做的修改 | 上下文耗尽 | 见第6节 |
| 测试用错工具（Jest vs Vitest）| 项目规范未指定 | 在 CLAUDE.md 中明确 |

### 提升质量的 CLAUDE.md 配置

```markdown
## 严格约束（优先级最高）

### 禁止事项
- 禁止使用 console.log（使用 logger 模块）
- 禁止 any 类型（TypeScript strict 模式）
- 禁止不带 await 的 async 函数
- 禁止硬编码配置值（使用环境变量）

### 必须使用
- 测试框架：Vitest（不是 Jest）
- API 框架：Fastify（不是 Express）
- 校验库：Zod
- 错误处理：Result<T, E> 类型（不用 try/catch）

### 代码风格
- 函数长度：< 50 行
- 文件长度：< 300 行
- 注释语言：中文
```

---

## 8. 文件操作问题排查

### 症状
- Edit 工具报 "Pattern not found"
- Write 覆盖了不期望的文件
- 检查点恢复后文件状态不对

### Edit 工具 "Pattern not found" 排查

```bash
# 确认要修改的文本确实存在
grep -n "exact-text-to-find" src/file.ts

# 检查是否有隐藏字符（Windows 换行符问题）
cat -A src/file.ts | head -5
# ^M 表示 Windows 换行符

# 如果有换行符问题
dos2unix src/file.ts
```

**正确的 Edit 使用：**
```
# old_string 必须完全精确匹配文件中的内容
# 包括缩进、空格、换行

# 如果 old_string 在文件中出现多次，需要提供更多上下文
# 使用 replace_all: true 替换所有匹配
```

### 检查点恢复后文件不同步

```bash
# 检查 git 状态
git status

# 如果有未跟踪的变更（检查点外部操作）
git diff

# 选择处理方式：
git stash       # 暂存外部变更
# 或
git add -A && git commit -m "外部变更"
# 然后再恢复检查点
```

### 文件被意外修改

```bash
# 查看最近的文件变更
git diff HEAD~1

# 恢复单个文件
git checkout HEAD~1 -- src/affected-file.ts

# 或通过检查点完全回滚
# Esc+Esc → 选择正确的节点 → "恢复代码"
```

---

## 9. 调试工具和技巧

### Claude Code 内置调试

```bash
# 开启详细调试日志
claude --debug

# 查看当前配置
claude config list
claude config get hooks

# 查看 MCP 服务器状态
claude mcp list
claude mcp get server-name
```

### 环境检查脚本

```bash
#!/bin/bash
# debug-claude-env.sh - 检查 Claude Code 环境状态

echo "=== Claude Code 版本 ==="
claude --version

echo ""
echo "=== 配置文件 ==="
for f in ".claude/settings.json" "~/.claude/settings.json" "CLAUDE.md" ".claude/CLAUDE.md"; do
  if [ -f "$f" ]; then
    echo "✅ $f ($(wc -c < "$f") bytes)"
  else
    echo "❌ $f (不存在)"
  fi
done

echo ""
echo "=== Skills ==="
ls .claude/skills/ 2>/dev/null || echo "无自定义 Skills"

echo ""
echo "=== Agents ==="
ls .claude/agents/ 2>/dev/null || echo "无自定义 Agents"

echo ""
echo "=== MCP 服务器 ==="
claude mcp list 2>/dev/null || echo "无 MCP 配置"

echo ""
echo "=== Git 状态 ==="
git status --short 2>/dev/null || echo "不是 git 仓库"

echo ""
echo "=== 环境变量 ==="
echo "ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:+已设置}"
echo "ANTHROPIC_BASE_URL: ${ANTHROPIC_BASE_URL:-未设置}"
echo "CLAUDE_CODE_DISABLE_AUTO_MEMORY: ${CLAUDE_CODE_DISABLE_AUTO_MEMORY:-未设置}"
```

### Hook 调试模板

```javascript
// scripts/debug-hook.js
// 将此脚本设为任意 Hook 的命令来捕获输入

const fs = require('fs');

process.stdin.resume();
let data = '';
process.stdin.on('data', chunk => data += chunk);
process.stdin.on('end', () => {
  const input = JSON.parse(data || '{}');
  
  // 写入调试文件
  fs.appendFileSync('/tmp/claude-hook-debug.log', 
    `\n[${new Date().toISOString()}]\n` +
    JSON.stringify(input, null, 2) + '\n'
  );
  
  // 不阻止执行，继续
  process.exit(0);
});
```

```bash
# 配置为 PostToolUse Hook
{
  "hooks": {
    "PostToolUse": [{
      "matcher": ".*",
      "hooks": [{"type": "command", "command": "node scripts/debug-hook.js"}]
    }]
  }
}

# 实时监控
tail -f /tmp/claude-hook-debug.log
```

---

## 10. 错误信息速查

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `Rate limit exceeded` | 请求过于频繁 | 等待 60s，减小批次大小 |
| `Context length exceeded` | 上下文太长 | 使用 Esc+Esc 摘要压缩 |
| `Tool not allowed` | 工具未在白名单 | 检查 allowedTools 配置 |
| `MCP server not found` | 服务器名拼写错误 | `claude mcp list` 确认名称 |
| `SKILL.md not found` | 技能目录结构错误 | 确认是 `.claude/skills/name/SKILL.md` |
| `Pattern not found in Edit` | old_string 不精确 | 复制文件中的精确文本 |
| `Permission denied in subagent` | 后台智能体未预批准工具 | 在 agent.md 中添加 tools 字段 |
| `Hook exit code 2` | Hook 主动阻止操作 | 检查 stopReason，手动确认操作 |
| `Insufficient balance` | API 额度不足 | 充值或等待额度刷新 |
| `Invalid JSON in settings.json` | 配置文件格式错误 | `python3 -m json.tool settings.json` 验证 |

---

## 相关模块

- [03-skills.md](03-skills.md) — Skills 配置详解
- [06-hooks.md](06-hooks.md) — Hooks 详细文档
- [05-mcp.md](05-mcp.md) — MCP 服务器配置
- [04-subagents.md](04-subagents.md) — 子智能体配置
- [08-checkpoints.md](08-checkpoints.md) — 检查点和上下文管理
- [14-exercises.md](14-exercises.md) — 实践练习
