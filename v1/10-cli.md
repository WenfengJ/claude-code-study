# 模块 10：CLI Basics（命令行基础）

> Level 1 基础 | 预计学习时间：30分钟

---

## 两种主要模式

| 模式 | 命令 | 特点 |
|------|------|------|
| **交互式 REPL** | `claude` | 多轮对话，Tab 补全，历史记录 |
| **Print Mode** | `claude -p "..."` | 单次查询后退出，适合脚本/CI |

---

## 核心命令

### 基础启动
```bash
claude                          # 启动交互式会话
claude -p "查询内容"            # 单次查询（Print Mode）
claude -c                       # 继续上次会话
claude -r "session-name"        # 恢复指定名称的会话
```

### 管道和重定向
```bash
# 处理管道输入
cat file.ts | claude -p "分析这段代码的安全问题"
git diff | claude -p "生成 commit message"
ls -la | claude -p "找出最大的文件"

# 处理文件
claude -p "重构这个函数" < src/utils.ts
```

### 模型选择
```bash
claude --model claude-opus-4-6         # 使用 Opus（最强）
claude --model claude-sonnet-4-6       # 使用 Sonnet（均衡）
claude --model claude-haiku-4-5        # 使用 Haiku（最快）
```

---

## 会话管理

```bash
# 继续会话
claude -c                       # 继续上次会话
claude -r my-project-session    # 恢复特定会话

# 分叉会话（从某个节点创建新分支）
/branch feature-experiment      # 在会话中执行
```

---

## 输出格式

```bash
# 文本输出（默认）
claude -p "解释这个错误"

# JSON 输出（适合脚本解析）
claude -p "分析文件" --output-format json

# 流式 JSON（适合实时处理）
claude -p "生成代码" --output-format stream-json
```

---

## 权限与安全控制

```bash
# 设置权限模式
claude --permission-mode acceptEdits    # 自动接受文件编辑
claude --permission-mode plan           # 只规划不执行
claude --permission-mode dontAsk        # 减少确认提示

# 工具白名单（只允许特定工具）
claude --allowedTools "Read,Grep,Glob"

# 工具黑名单（禁止特定工具）
claude --disallowedTools "Bash,Write"

# 最小执行模式（只读，适合安全场景）
claude --allowedTools "Read,Grep"
```

---

## CI/CD 集成

### 基础 CI 脚本
```bash
#!/bin/bash
# 代码审查自动化
git diff HEAD~1 | claude -p "
  审查这些代码变更：
  1. 安全漏洞
  2. 性能问题
  3. 代码质量
  以 JSON 格式输出发现的问题列表
" --output-format json
```

### GitHub Actions 示例
```yaml
name: Claude Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: AI Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/main | claude -p "
            审查这个 PR 的代码变更，
            重点关注安全性和性能问题。
          " --permission-mode plan
```

---

## 自定义系统提示词

```bash
# 追加到默认系统提示词
claude --append-system-prompt "你是一个专注于 Flink SQL 的专家"

# 自定义系统提示词文件
claude --system-prompt-file ./prompts/flink-expert.md
```

---

## 支出限制（防止意外消费）

```bash
# 设置最大支出（美元）
claude --max-spending 5.00

# 设置最大智能体轮数
claude --max-turns 10
```

---

## 加载额外目录的 CLAUDE.md

```bash
# 从其他目录加载上下文
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 \
claude --add-dir /path/to/shared/context
```

---

## 加载插件

```bash
# 本地测试插件
claude --plugin-dir ./my-plugin
claude --plugin-dir ./plugin-a --plugin-dir ./plugin-b
```

---

## 加载自定义子智能体（仅当前 session）

```bash
claude --agents '{
  "code-reviewer": {
    "description": "代码审查专家。代码变更后主动使用。",
    "prompt": "你是高级代码审查员，专注于安全性和性能。",
    "tools": ["Read", "Grep", "Glob"],
    "model": "sonnet"
  }
}'
```

---

## 环境变量配置

```bash
# 必须
export ANTHROPIC_API_KEY=your-api-key

# 可选
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1    # 禁用自动记忆
export CLAUDE_CODE_REMOTE=true              # 标记为远程环境
export SLASH_COMMAND_TOOL_CHAR_BUDGET=5000  # 技能描述字符预算
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=90   # 自动压缩阈值
```

---

## Print Mode vs Interactive Mode 选择指南

| 场景 | 推荐模式 |
|------|---------|
| 日常开发协作 | Interactive（`claude`） |
| 脚本/自动化 | Print（`claude -p`） |
| CI/CD 流水线 | Print（`claude -p`） |
| 管道处理文件 | Print（`claude -p`） |
| 多轮对话任务 | Interactive |
| 单次代码生成 | Print |

---

## 常用别名设置（.bashrc/.zshrc）

```bash
# 快捷别名
alias cc='claude'
alias ccp='claude -p'
alias ccr='claude -c'

# 常用场景
alias cc-review='git diff | claude -p "审查这些变更，重点关注安全性"'
alias cc-commit='git diff --staged | claude -p "生成规范的 commit message"'
alias cc-explain='claude -p "解释这段代码：$(cat"'
```

---

## 相关模块

- [01-slash-commands.md](01-slash-commands.md) — 交互模式中的命令
- [09-advanced-features.md](09-advanced-features.md) — 权限模式详解
- [07-plugins.md](07-plugins.md) — `--plugin-dir` 参数使用
- [04-subagents.md](04-subagents.md) — `--agents` 参数使用
