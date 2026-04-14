# 模块 6：Hooks（事件钩子）

> Level 2 自动化 | 预计学习时间：1小时

---

## 核心概念

Hooks 是在 Claude Code session 中特定事件发生时自动执行的脚本。

**支持的 Hook 类型：**
- **Command Hooks**：执行 shell 命令（默认）
- **HTTP Hooks**：调用远程 webhook 端点（v2.1.63+）
- **Prompt Hooks**：LLM 评估的提示词 Hook
- **Agent Hooks**：基于子智能体的验证 Hook

---

## 配置位置

| 位置 | 范围 | 是否提交 |
|------|------|---------|
| `~/.claude/settings.json` | 用户级（所有项目） | 否 |
| `.claude/settings.json` | 项目级（团队共享） | ✅ 是 |
| `.claude/settings.local.json` | 本地项目级 | 否 |
| 插件 `hooks/hooks.json` | 插件范围 | - |
| Skill/Agent frontmatter | 组件生命周期 | - |

---

## 基础配置结构

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint -- $CLAUDE_TOOL_OUTPUT_FILE 2>&1 | head -20"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Session started' >> ~/.claude/session.log"
          }
        ]
      }
    ]
  }
}
```

---

## 26个支持的 Hook 事件

### 会话生命周期
| 事件 | 触发时机 |
|------|---------|
| `SessionStart` | 会话启动 |
| `InstructionsLoaded` | 指令加载完成 |
| `SessionEnd` | 会话结束 |
| `PreCompact` | 上下文压缩前 |
| `PostCompact` | 上下文压缩后 |

### 用户交互
| 事件 | 触发时机 |
|------|---------|
| `UserPromptSubmit` | 用户提交提示词前 |
| `Notification` | 通知事件 |
| `Elicitation` | Claude 请求用户输入 |
| `ElicitationResult` | 用户响应 elicitation |

### 工具执行
| 事件 | 触发时机 |
|------|---------|
| `PreToolUse` | 工具执行前 |
| `PermissionRequest` | 权限请求 |
| `PermissionDenied` | 权限被拒绝 |
| `PostToolUse` | 工具执行成功后 |
| `PostToolUseFailure` | 工具执行失败后 |

### 子智能体
| 事件 | 触发时机 |
|------|---------|
| `SubagentStart` | 子智能体启动 |
| `SubagentStop` | 子智能体停止 |
| `TeammateIdle` | 队友完成任务且没有待办工作 |
| `TaskCompleted` | 共享任务列表中的任务完成 |
| `TaskCreated` | 新任务创建 |

### 文件系统 & 配置
| 事件 | 触发时机 |
|------|---------|
| `FileChanged` | 文件变更 |
| `CwdChanged` | 工作目录变更 |
| `ConfigChange` | 配置变更 |

### Stop & Worktree
| 事件 | 触发时机 |
|------|---------|
| `Stop` | Claude 完成响应 |
| `StopFailure` | Claude 响应失败 |
| `WorktreeCreate` | Worktree 创建 |
| `WorktreeRemove` | Worktree 移除 |

---

## 关键 Hook 事件详解

### PreToolUse — 在工具执行前验证或修改

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"permissionDecision\": \"allow\"}'"
          }
        ]
      }
    ]
  }
}
```

输出控制字段：
- `permissionDecision`：`allow` / `deny` / `ask`
- `reason`：拒绝或询问的原因
- `updatedInput`：修改后的工具输入

### PostToolUse — 工具执行后的验证和日志

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "git ls-files --others --exclude-standard | grep -E '\\.env|\\.local' || true"
          }
        ]
      }
    ]
  }
}
```

### UserPromptSubmit — 提示词提交前的验证

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"additionalContext\": \"当前分支: '$(git branch --show-current)'\"}'"
          }
        ]
      }
    ]
  }
}
```

---

## JSON 输入/输出

Hook 通过 stdin 接收 JSON：

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../conversation.jsonl",
  "cwd": "/path/to/project",
  "hook_event_name": "PostToolUse",
  "tool_name": "Write",
  "tool_input": { "file_path": "src/app.ts", "content": "..." }
}
```

**退出码：**
- `0`：成功
- `2`：阻塞错误（会阻止操作）
- 其他：非阻塞错误

**JSON 输出字段：**
- `continue`：是否继续
- `stopReason`：停止原因
- `suppressOutput`：是否抑制输出
- `systemMessage`：注入系统消息
- `hookSpecificOutput`：事件特定字段

---

## HTTP Hook（远程 Webhook）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "http",
            "url": "https://my-service.com/webhook",
            "headers": {
              "Authorization": "Bearer ${WEBHOOK_TOKEN}"
            },
            "allowedEnvVars": ["WEBHOOK_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

---

## Matcher 匹配模式

```json
// 精确匹配
"matcher": "Write"

// 正则模式
"matcher": "Write|Edit|Bash"

// 通配符
"matcher": "*"

// MCP 工具
"matcher": "mcp__github__create_issue"

// InstructionsLoaded 特殊值
"matcher": "session_start"        // 会话启动
"matcher": "nested_traversal"     // 嵌套遍历
"matcher": "path_glob_match"      // 路径 glob 匹配
```

---

## 实战示例

### 示例1：自动检测 git 中的硬编码路径

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "git ls-files | xargs grep -l '/home/\\|/Users/' 2>/dev/null | head -5 || echo 'No hardcoded paths found'"
          }
        ]
      }
    ]
  }
}
```

### 示例2：自动运行 Lint（写文件后）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && npm run lint 2>&1 | tail -20"
          }
        ]
      }
    ]
  }
}
```

### 示例3：Session 开始时自动设置上下文

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Current branch: $(git branch --show-current)\nLast commit: $(git log --oneline -1)\nUnstaged changes: $(git status --porcelain | wc -l)\" > $CLAUDE_ENV_FILE"
          }
        ]
      }
    ]
  }
}
```

### 示例4：阻止删除生产文件

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo '$TOOL_INPUT' | grep -q 'rm.*production'; then echo '{\"permissionDecision\": \"deny\", \"reason\": \"禁止删除生产文件\"}'; fi"
          }
        ]
      }
    ]
  }
}
```

---

## 环境变量

| 变量 | 说明 |
|------|------|
| `CLAUDE_PROJECT_DIR` | 项目根路径 |
| `CLAUDE_ENV_FILE` | 用于持久化环境变量的文件 |
| `CLAUDE_CODE_REMOTE` | 远程环境时为 "true" |
| `${CLAUDE_PLUGIN_ROOT}` | 插件根目录 |
| `${CLAUDE_PLUGIN_DATA}` | 插件持久数据目录 |

---

## 组件级 Hook

在 SKILL.md 或 agent.md 的 frontmatter 中定义，仅在组件生命周期内生效：

```yaml
---
name: my-skill
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
  Stop:
    - hooks:
        - type: command
          command: "./scripts/cleanup.sh"
---
```

支持的事件：`PreToolUse`、`PostToolUse`、`Stop`（子智能体中自动转为 `SubagentStop`）

---

## 执行细节

- 默认超时：60秒（可配置）
- 匹配的 Hook 并行运行
- 相同命令会去重
- 在当前目录下用 Claude Code 的环境执行

---

## 调试

```bash
claude --debug    # 启用详细日志
# 按 Ctrl+O 查看 verbose 模式
```

---

## 安全注意事项

> **使用风险自担**：Hook 执行任意 shell 命令，安全责任由配置者承担。

最佳实践：
- 验证所有输入
- 引号包裹 shell 变量
- 阻止路径遍历攻击
- 使用 `$CLAUDE_PROJECT_DIR` 绝对路径
- 跳过敏感文件
- 先独立测试 Hook
- HTTP Hook 使用显式 `allowedEnvVars`

---

## 相关模块

- [03-skills.md](03-skills.md) — 在技能中定义组件级 Hook
- [04-subagents.md](04-subagents.md) — SubagentStart/SubagentStop 事件
- [09-advanced-features.md](09-advanced-features.md) — 与 Planning Mode 结合使用
