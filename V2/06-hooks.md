# 模块 6：Hooks（事件钩子）完全指南

> Level 2 | 预计学习时间：3小时 | Claude Code v2.1.101+

---

## 核心概念

Hooks 在 Claude Code session 特定事件发生时自动执行脚本，实现：
- **验证**：阻止有问题的操作
- **自动化**：执行后续操作（lint、测试、通知）
- **日志**：记录操作历史
- **上下文注入**：为 Claude 提供额外信息

---

## 4种 Hook 类型

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| **Command**（默认） | 执行 shell 命令，通过 stdin/stdout 和退出码交互 | 大多数自动化场景 |
| **HTTP**（v2.1.63+） | 调用远程 webhook | 通知系统、外部审计 |
| **Prompt** | LLM 评估的提示词 | 复杂语义判断 |
| **Agent** | 子智能体验证 | 需要 AI 推理的复杂检查 |

---

## 配置位置与范围

| 位置 | 范围 | 是否提交 |
|------|------|---------|
| `~/.claude/settings.json` | 用户（所有项目） | 否 |
| `.claude/settings.json` | 项目（团队共享） | ✅ |
| `.claude/settings.local.json` | 本地（个人） | 否 |
| `<plugin>/hooks/hooks.json` | 插件范围 | - |
| Skill/Agent frontmatter | 组件生命周期 | - |

---

## 26个 Hook 事件完整参考

### 会话生命周期
```
SessionStart        → 会话启动（可写入 CLAUDE_ENV_FILE 设置环境变量）
InstructionsLoaded  → 指令加载完成
PreCompact          → 上下文压缩前
PostCompact         → 上下文压缩后
SessionEnd          → 会话结束（有专属超时控制）
```

### 用户交互
```
UserPromptSubmit    → 用户提交提示词前（可阻止或注入上下文）
Notification        → Claude 发出通知
Elicitation         → Claude 请求用户额外输入
ElicitationResult   → 用户响应了 elicitation
```

### 工具执行
```
PreToolUse          → 工具执行前（可 allow/deny/ask，可修改输入）
PermissionRequest   → Claude 请求额外权限
PermissionDenied    → 权限被拒绝
PostToolUse         → 工具成功执行后（可注入上下文，可阻止）
PostToolUseFailure  → 工具执行失败后
```

### 子智能体
```
SubagentStart       → 子智能体启动
SubagentStop        → 子智能体停止（Prompt Hook 在此评估任务完成度）
Stop                → Claude 主响应完成
StopFailure         → Claude 响应失败
TeammateIdle        → Agent Teams：队友空闲（Agent Teams 实验功能）
TaskCompleted       → Agent Teams：任务完成
TaskCreated         → Agent Teams：新任务创建
```

### 文件系统 & 配置
```
FileChanged         → 文件变更（可写入 CLAUDE_ENV_FILE）
CwdChanged          → 工作目录变更（可写入 CLAUDE_ENV_FILE）
ConfigChange        → 配置变更
```

### Worktree
```
WorktreeCreate      → Worktree 创建
WorktreeRemove      → Worktree 移除
```

---

## Command Hook 完整配置

### 基础结构
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint 2>&1 | head -30",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### JSON 输入格式（Hook 从 stdin 接收）
```json
{
  "session_id": "abc-123",
  "transcript_path": "~/.claude/projects/.../conversation.jsonl",
  "cwd": "/path/to/project",
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "src/auth.ts",
    "content": "..."
  },
  "tool_response": {
    "success": true
  }
}
```

### JSON 输出格式（Hook 写到 stdout）
```json
{
  "continue": true,
  "stopReason": "安全检查失败",
  "suppressOutput": false,
  "systemMessage": "注意：检测到敏感文件变更",
  "hookSpecificOutput": {
    "permissionDecision": "allow",
    "reason": "操作安全",
    "updatedInput": {}
  }
}
```

### 退出码
| 退出码 | 行为 |
|--------|------|
| `0` | 成功，继续执行 |
| `2` | 阻塞错误，停止当前操作 |
| 其他 | 非阻塞错误，记录日志但继续 |

---

## Matcher 匹配模式

```json
// 精确工具名
"matcher": "Write"

// 正则（多个工具）
"matcher": "Write|Edit|Bash"

// 通配符（所有工具）
"matcher": "*"

// MCP 工具（格式：mcp__server__tool）
"matcher": "mcp__github__create_issue"

// InstructionsLoaded 特殊值
"matcher": "session_start"        // 仅会话启动时
"matcher": "nested_traversal"     // 子目录遍历时
"matcher": "path_glob_match"      // 路径匹配时
```

---

## 实战示例大全

### 示例1：写文件后自动 Lint + 类型检查

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && npx tsc --noEmit 2>&1 | head -20 && npx eslint $(echo '$TOOL_INPUT' | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get(\"file_path\",\"\"))') 2>&1 | head -20"
          }
        ]
      }
    ]
  }
}
```

### 示例2：Session 启动时注入 Git 上下文

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"CURRENT_BRANCH=$(git branch --show-current)\nLAST_COMMIT=$(git log --oneline -1)\nUNSTAGED_FILES=$(git status --porcelain | wc -l)\" > $CLAUDE_ENV_FILE"
          }
        ]
      }
    ]
  }
}
```

### 示例3：PreToolUse 阻止删除生产配置

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "INPUT=$(cat); CMD=$(echo $INPUT | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get(\"tool_input\",{}).get(\"command\",\"\"))'); if echo \"$CMD\" | grep -qE 'rm.*(production|prod|config\\.yaml)'; then echo '{\"continue\": false, \"stopReason\": \"禁止删除生产配置文件\"}'; exit 2; fi"
          }
        ]
      }
    ]
  }
}
```

### 示例4：检测 git 中的硬编码路径（写文件后）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "HARDCODED=$(git ls-files | xargs grep -l '/home/\\|/Users/\\|C:\\\\\\\\Users\\\\\\\\' 2>/dev/null); if [ -n \"$HARDCODED\" ]; then echo \"{\\\"systemMessage\\\": \\\"警告：以下文件包含硬编码路径：$HARDCODED\\\"}\"; fi"
          }
        ]
      }
    ]
  }
}
```

### 示例5：UserPromptSubmit 注入环境信息

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"{\\\"additionalContext\\\": \\\"当前分支: $(git branch --show-current), Node版本: $(node -v), 时间: $(date '+%Y-%m-%d %H:%M')\\\"}\" "
          }
        ]
      }
    ]
  }
}
```

### 示例6：文件变更后触发测试

```json
{
  "hooks": {
    "FileChanged": [
      {
        "matcher": "**/*.ts",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && npm test -- --passWithNoTests 2>&1 | tail -20"
          }
        ]
      }
    ]
  }
}
```

### 示例7：Stop Hook 验证任务完成度（Prompt 类型）

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "检查 last_assistant_message，判断任务是否真正完成：\n1. 是否有遗留的 TODO 或 FIXME\n2. 是否提到还需要手动操作\n3. 是否所有测试都通过\n如果未完成，返回 {\"continue\": true, \"systemMessage\": \"任务未完成，原因：xxx\"}\n如果完成，返回 {\"continue\": false}"
          }
        ]
      }
    ]
  }
}
```

### 示例8：HTTP Hook 发送 Slack 通知

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.slack.com/services/xxx/yyy/zzz",
            "headers": {
              "Content-Type": "application/json",
              "Authorization": "Bearer ${SLACK_TOKEN}"
            },
            "allowedEnvVars": ["SLACK_TOKEN"],
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

---

## HTTP Hook 完整配置

```json
{
  "type": "http",
  "url": "https://api.example.com/webhook",
  "headers": {
    "Authorization": "Bearer ${API_TOKEN}",
    "X-Source": "claude-code"
  },
  "allowedEnvVars": ["API_TOKEN"],   // 必须显式声明允许的环境变量
  "timeout": 30                       // 超时秒数
}
```

---

## 组件级 Hook（Skill/Agent 内部）

```yaml
---
name: safe-deploy
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/pre-deploy-check.sh"
          timeout: 60
  PostToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/post-deploy-verify.sh"
  Stop:
    - hooks:
        - type: command
          command: "./scripts/notify-deploy-done.sh"
---

部署技能：执行生产部署流程
```

组件级 Hook 仅在该 Skill/Agent 活跃期间生效，不影响其他 session。

---

## 执行机制

| 特性 | 说明 |
|------|------|
| 并行执行 | 匹配同一事件的 Hook 并行运行 |
| 命令去重 | 完全相同的命令自动去重（不重复执行） |
| 默认超时 | 60秒（可在 Hook 配置中覆盖） |
| 执行环境 | 当前工作目录 + Claude Code 的环境变量 |
| SessionEnd 超时 | 可用 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 覆盖 |

---

## 环境变量

| 变量 | 说明 | 可写事件 |
|------|------|---------|
| `CLAUDE_PROJECT_DIR` | 项目根目录绝对路径 | 所有 |
| `CLAUDE_ENV_FILE` | 持久化环境变量的文件路径 | SessionStart, CwdChanged, FileChanged |
| `CLAUDE_CODE_REMOTE` | 远程环境时为 "true" | 所有 |
| `${CLAUDE_PLUGIN_ROOT}` | 插件根目录 | 插件 Hook |
| `${CLAUDE_PLUGIN_DATA}` | 插件持久数据目录 | 插件 Hook |

**使用 CLAUDE_ENV_FILE 持久化变量：**
```bash
# SessionStart Hook 中
echo "DEPLOY_ENV=production" >> $CLAUDE_ENV_FILE
echo "DB_REPLICA=replica.example.com" >> $CLAUDE_ENV_FILE
# 这些变量在整个 session 中对 Claude 可见
```

---

## 调试 Hooks

```bash
# 方式1：Claude Code debug 模式
claude --debug

# 方式2：Verbose 模式
# 在 session 中按 Ctrl+O

# 方式3：手动测试 Hook（模拟输入）
echo '{"session_id":"test","hook_event_name":"PostToolUse","tool_name":"Write","tool_input":{"file_path":"test.ts"}}' | bash your-hook.sh
```

---

## 安全注意事项

> **重要**：Hooks 执行任意 shell 命令，安全责任由配置者承担。

**必须做的：**
```bash
# ✅ 验证所有输入
INPUT=$(cat)
FILE=$(echo "$INPUT" | python3 -c 'import sys,json; print(json.load(sys.stdin).get("tool_input",{}).get("file_path",""))')

# ✅ 引号包裹变量（防止注入）
if [ -n "$FILE" ]; then
  eslint "$FILE"
fi

# ✅ 使用绝对路径
cd "$CLAUDE_PROJECT_DIR" && npm test

# ✅ 阻止路径遍历
if echo "$FILE" | grep -q '\.\.'; then
  echo "路径遍历攻击，拒绝" >&2
  exit 2
fi
```

**不要做的：**
```bash
# ❌ 直接使用未验证的输入
eval "$USER_INPUT"

# ❌ 硬编码凭据
curl -H "Authorization: Bearer hardcoded-secret" ...

# ❌ 在 HTTP Hook 中不声明 allowedEnvVars
```

---

## 相关模块

- [03-skills.md](03-skills.md) — Skill 内定义组件级 Hook
- [04-subagents.md](04-subagents.md) — SubagentStart/SubagentStop 事件
- [11-integration-patterns.md](11-integration-patterns.md) — Skills + Hooks 自动化集成模式
- [13-troubleshooting.md](13-troubleshooting.md) — Hook 不执行的排查方法
