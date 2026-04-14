# 模块 10：CLI Basics（命令行基础）完全指南

> Level 1 | 预计学习时间：2小时 | Claude Code v2.1.101+

---

## 两种核心模式

| 模式 | 启动方式 | 特点 | 适用场景 |
|------|---------|------|---------|
| **Interactive REPL** | `claude` | 多轮对话、Tab 补全、历史记录 | 日常开发 |
| **Print Mode** | `claude -p "..."` | 单次执行后退出、可管道、可脚本化 | CI/CD、自动化 |

---

## 完整 CLI 参数参考

### 基础启动

```bash
claude                          # 启动交互式 session
claude -p "query"               # Print Mode：单次查询
claude --print "query"          # 同 -p 的全称
claude -c                       # 继续上次 session
claude --continue               # 同 -c 的全称
claude -r "session-name"        # 恢复命名 session
claude --resume "session-name"  # 同 -r 的全称
```

### 模型配置

```bash
# 选择模型
claude --model claude-opus-4-6         # 最强推理，复杂架构/分析
claude --model claude-sonnet-4-6       # 均衡，日常编码（默认）
claude --model claude-haiku-4-5        # 最快，简单任务

# 推理强度（Print Mode 中）
claude -p "query" --effort high
claude -p "query" --effort max
```

### 权限与安全

```bash
# 权限模式
claude --permission-mode default        # 询问危险操作（默认）
claude --permission-mode acceptEdits    # 自动接受文件编辑
claude --permission-mode plan           # 只读规划模式
claude --permission-mode dontAsk        # 减少确认提示
claude --permission-mode auto           # AI 分类器自动决策
claude --permission-mode bypassPermissions  # ⚠️ 完全绕过

# 工具白名单
claude --allowedTools "Read,Grep,Glob"
claude --allowedTools "Read,Bash(git:*),Bash(npm:*)"

# 工具黑名单
claude --disallowedTools "Bash,Write,Edit"

# 花费限制（防止意外超额）
claude --max-spending 5.00              # 最多花 $5
claude --max-turns 10                   # 最多 10 轮智能体循环
```

### 会话与上下文

```bash
# 系统提示词
claude --append-system-prompt "你是 Flink SQL 专家，专注于流处理优化"
claude --system-prompt-file ./prompts/expert.md

# 额外目录（加载其他目录的 CLAUDE.md）
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 \
  claude --add-dir /path/to/shared-context
```

### 插件与智能体

```bash
# 加载本地插件（开发测试用）
claude --plugin-dir ./my-plugin
claude --plugin-dir ./plugin-a --plugin-dir ./plugin-b

# 定义临时子智能体（当前 session）
claude --agents '{
  "reviewer": {
    "description": "代码审查专家，代码变更后主动使用",
    "prompt": "你是资深代码审查员，专注安全性和性能",
    "tools": ["Read", "Grep", "Glob"],
    "model": "sonnet"
  }
}'

# 指定主智能体
claude --agent code-reviewer
```

### 输出格式

```bash
# 文本输出（默认）
claude -p "解释这段代码"

# JSON 输出（适合脚本解析）
claude -p "分析文件" --output-format json

# 流式 JSON（适合实时处理）
claude -p "生成代码" --output-format stream-json
```

### 调试

```bash
claude --debug                          # 启用详细调试日志
```

---

## 管道与重定向

```bash
# 基础管道
cat src/auth.ts | claude -p "找出所有安全漏洞"
git diff | claude -p "生成 conventional commit message"
ls -la | claude -p "找出最大的10个文件"

# 文件输入
claude -p "重构这个函数，添加错误处理" < src/utils.ts

# 多文件分析
cat src/*.ts | claude -p "这些文件中有哪些重复逻辑？"

# 输出到文件
claude -p "生成 README" > README.md
claude -p "代码审查报告" --output-format json > review.json

# 组合使用
git log --oneline -20 | \
  claude -p "总结最近20次提交的主要改动，按功能分类" | \
  tee changelog-draft.md
```

---

## CI/CD 集成模板

### GitHub Actions：代码审查

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: AI Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD | \
          claude -p "
            审查这个 PR 的代码变更。
            重点关注：
            1. 安全漏洞（OWASP Top 10）
            2. 性能问题（N+1查询、内存泄漏）
            3. 错误处理缺失
            4. 代码规范违反
            以 Markdown 格式输出，每类问题单独章节。
          " --permission-mode plan | \
          tee pr-review.md

      - name: Post Review Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('pr-review.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## AI Code Review\n\n${review}`
            });
```

### GitHub Actions：安全扫描

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # 每周一凌晨2点

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Security Scan
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          find src -name "*.ts" -o -name "*.js" | \
          xargs cat | \
          claude -p "
            对这些源文件进行安全审计：
            - 认证和授权漏洞
            - SQL 注入点
            - 硬编码凭据
            - 不安全的数据处理
            输出 JSON 格式：{issues: [{severity, type, location, description}]}
          " --output-format json > security-report.json

      - name: Check Critical Issues
        run: |
          CRITICAL=$(cat security-report.json | jq '[.issues[] | select(.severity=="critical")] | length')
          if [ "$CRITICAL" -gt 0 ]; then
            echo "发现 $CRITICAL 个严重安全问题！"
            cat security-report.json | jq '.issues[] | select(.severity=="critical")'
            exit 1
          fi
```

### 文档自动生成

```bash
#!/bin/bash
# generate-docs.sh

# 为每个主要模块生成 API 文档
for module in src/services/*.ts; do
  basename="${module%.ts}"
  name=$(basename "$basename")
  
  echo "生成 $name 的文档..."
  
  cat "$module" | claude -p "
    生成这个模块的 API 文档，包含：
    1. 模块概述
    2. 每个导出函数的：参数、返回值、示例
    3. 常见使用模式
    以 Markdown 格式输出。
  " > "docs/api/$name.md"
  
  echo "✅ $name.md 生成完成"
done

echo "所有文档生成完毕"
```

---

## 环境变量完整参考

```bash
# === 必须配置 ===
ANTHROPIC_API_KEY="your-api-key"

# === 记忆控制 ===
CLAUDE_CODE_DISABLE_AUTO_MEMORY=0          # 强制启用自动记忆
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1          # 强制禁用自动记忆

# === 功能开关 ===
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1     # 启用 Agent Teams（实验性）
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1     # 禁用后台任务
CLAUDE_CODE_NEW_INIT=1                     # 增强版 /init 交互流程
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1  # 启用 --add-dir

# === 调优 ===
SLASH_COMMAND_TOOL_CHAR_BUDGET=8000        # 技能描述字符预算（默认8000）
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=90         # 自动压缩阈值（默认95%）
CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS=5000  # SessionEnd Hook 超时

# === 代理 ===
ANTHROPIC_BASE_URL="https://api.example.com"  # 自定义 API 端点（企业代理）

# === 遥测 ===
CLAUDE_CODE_DISABLE_TELEMETRY=1            # 禁用使用数据上报
```

---

## 常用别名和脚本

### Shell 别名（.bashrc/.zshrc）

```bash
# 基础别名
alias cc='claude'
alias ccp='claude -p'
alias ccr='claude -c'

# 常用工作流
alias cc-review='git diff | claude -p "审查这些代码变更，重点关注安全性和性能"'
alias cc-commit='git diff --staged | claude -p "生成规范的 Conventional Commit message，直接输出，不要解释"'
alias cc-explain='claude -p "解释这段代码的功能和潜在问题："'
alias cc-test='claude -p "为这段代码生成全面的测试用例（包括边界条件和错误路径）："'
alias cc-doc='claude -p "生成这个模块的 API 文档，Markdown 格式："'

# 项目特定
alias cc-sql='claude -p "你是 Flink SQL 专家。回答这个问题："'
alias cc-port='claude -p "检查这些文件的 git 可移植性问题（硬编码路径、本地配置等）："'
```

### 实用脚本

```bash
#!/bin/bash
# cc-batch-review.sh - 批量审查所有最近修改的文件

DAYS=${1:-7}
echo "审查过去 $DAYS 天内修改的文件..."

git log --since="$DAYS days ago" --name-only --pretty=format: | \
  sort -u | \
  grep -E '\.(ts|js|py|go|java)$' | \
  while read file; do
    if [ -f "$file" ]; then
      echo "=== 审查: $file ==="
      cat "$file" | claude -p "简要审查这个文件，列出最重要的3个改进点" 2>/dev/null
      echo ""
    fi
  done
```

---

## Print Mode 最佳实践

```bash
# ✅ 好的提示词：具体、有明确输出格式
claude -p "
分析 package.json 的依赖，找出：
1. 过期超过6个月的依赖（对比当前日期 $(date +%Y-%m-%d)）
2. 有已知安全漏洞的依赖
输出 JSON 格式：{outdated: [], vulnerable: []}
" < package.json --output-format json

# ❌ 避免：模糊的提示词
claude -p "分析这个文件" < package.json
```

---

## 相关模块

- [01-slash-commands.md](01-slash-commands.md) — 交互模式中的命令
- [09-advanced-features.md](09-advanced-features.md) — 权限模式详解
- [07-plugins.md](07-plugins.md) — `--plugin-dir` 参数
- [04-subagents.md](04-subagents.md) — `--agents` 参数
- [12-real-world-workflows.md](12-real-world-workflows.md) — 完整 CI/CD 工作流
