# 模块 12：真实工作流模板

> Level 3 | 预计学习时间：3小时

---

## 工作流1：完整 PR 审查流程

### 场景
每次有新 PR 时，自动执行安全扫描、测试验证、代码质量检查，汇总报告。

### 技能配置（.claude/skills/pr-review/SKILL.md）

```yaml
---
name: pr-review
description: 完整的 PR 审查：安全、测试、代码质量三维检查。当用户说"审查PR"或"review PR"时使用。
context: fork
agent: general-purpose
disable-model-invocation: true
argument-hint: "[PR号 或 留空=当前PR]"
---

# PR 审查工作流

## 当前 PR 信息
- **分支**：!`git branch --show-current`
- **变更文件**：!`git diff --name-only origin/main`
- **提交数**：!`git rev-list --count HEAD ^origin/main`
- **PR 差异**：!`gh pr diff 2>/dev/null || git diff origin/main`

## 审查任务（按顺序执行）

### 1. 安全审查
检查以下安全问题：
- SQL 注入、命令注入、XSS
- 认证/授权绕过
- 硬编码密钥或敏感信息
- 不安全的依赖

### 2. 测试覆盖验证
- 新功能是否有对应测试
- 现有测试是否通过：运行 `npm test --passWithNoTests 2>&1 | tail -20`
- 边界条件是否覆盖

### 3. 代码质量
- SOLID 原则遵守
- 函数大小（建议<50行）
- 错误处理完整性
- 代码风格一致性

## 输出格式

### PR 审查报告：#$ARGUMENTS

**总体评分：X/10**

| 维度 | 评分 | 发现问题数 |
|------|------|---------|
| 安全性 | X/10 | N |
| 测试覆盖 | X/10 | N |
| 代码质量 | X/10 | N |

### 🔴 严重问题（必须修复）
...

### 🟡 建议改进
...

### ✅ 亮点
...
```

### 使用方式
```bash
/pr-review         # 审查当前 PR
/pr-review 123     # 审查 PR #123
```

---

## 工作流2：CI/CD 代码质量门控

### 场景
在 CI 流水线中使用 Claude Code 做额外的智能质量检查，阻止低质量代码合并。

### GitHub Actions 配置

```yaml
name: Intelligent Quality Gate

on:
  pull_request:
    branches: [main, develop]

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup
        run: npm ci

      - name: Standard Checks
        run: |
          npm run lint
          npm run typecheck
          npm test -- --coverage

      - name: AI Quality Analysis
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          DIFF=$(git diff origin/${{ github.base_ref }}...HEAD)
          
          RESULT=$(echo "$DIFF" | claude -p "
            分析这个 PR 的代码变更，按以下标准评估：
            
            评分标准（每项0-10分）：
            1. 安全性：有无安全漏洞
            2. 错误处理：异常路径是否覆盖
            3. 可维护性：代码是否清晰、有无重复
            4. 测试性：新代码是否易于测试
            
            输出严格的 JSON，不要有其他文字：
            {
              \"scores\": {\"security\":0,\"errorHandling\":0,\"maintainability\":0,\"testability\":0},
              \"overall\": 0,
              \"blockReason\": null,
              \"issues\": []
            }
            
            如果 overall < 6 或 security < 7，设置 blockReason 为阻止原因。
          " --output-format json)
          
          echo "Quality Report:"
          echo "$RESULT" | python3 -m json.tool
          
          BLOCK=$(echo "$RESULT" | python3 -c "
          import sys, json
          d = json.load(sys.stdin)
          print(d.get('blockReason') or '')
          ")
          
          if [ -n "$BLOCK" ]; then
            echo "❌ Quality Gate Failed: $BLOCK"
            exit 1
          else
            echo "✅ Quality Gate Passed"
          fi

      - name: Post AI Review Comment
        if: always()
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GH_TOKEN: ${{ github.token }}
        run: |
          DIFF=$(git diff origin/${{ github.base_ref }}...HEAD | head -500)
          
          REVIEW=$(echo "$DIFF" | claude -p "
            生成一份简洁的代码审查评论（Markdown格式，中文）。
            包含：
            1. 主要变更摘要（2-3句话）
            2. 值得表扬的地方（如果有）
            3. 建议改进（如果有，最多3条）
            不要超过300字。
          ")
          
          gh pr comment ${{ github.event.pull_request.number }} \
            --body "## 🤖 AI 代码审查

          $REVIEW

          ---
          *由 Claude AI 自动生成*"
```

---

## 工作流3：大型 Codebase 重构

### 场景
将整个项目从 CommonJS 迁移到 ES Modules，涉及数十个文件。

### 分阶段迁移策略

```bash
# 阶段1：评估和规划
/plan 将这个 Node.js 项目从 CommonJS 迁移到 ES Modules

# 批准计划后，阶段2：自动化迁移
/batch 将所有 .js 文件的 require() 改为 import，module.exports 改为 export
# Claude 自动：
# 1. 分析所有文件
# 2. 使用 worktrees 并行处理
# 3. 对每个文件做精确替换
# 4. 汇总所有变更

# 阶段3：验证
npm test
# 如果失败，用检查点回滚到修复点
```

### 批量处理技能

```yaml
---
name: batch-migrate
description: 大规模批量代码迁移任务。当需要对多个文件做统一修改时使用。
disable-model-invocation: true
argument-hint: "[迁移描述]"
---

# 批量迁移任务：$ARGUMENTS

## 执行流程

### 1. 分析阶段（只读）
- 统计受影响文件数量
- 识别迁移模式
- 评估风险点

### 2. 制定计划
列出所有需要修改的文件，以及每个文件的具体变更内容。
等待确认后再执行。

### 3. 执行迁移
- 每次处理一个文件
- 修改后验证语法正确
- 记录每个文件的处理结果

### 4. 验证
- 运行测试套件
- 检查类型错误
- 报告成功/失败数量

## 当前仓库状态
- 总文件数：!`git ls-files --count`
- 分支：!`git branch --show-current`
```

---

## 工作流4：文档自动同步流水线

### 场景
代码变更后，自动检测 API 变更并更新对应文档，保持文档与代码同步。

### Hook 配置

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node scripts/detect-api-change.js"
          }
        ]
      }
    ]
  }
}
```

**scripts/detect-api-change.js：**
```javascript
process.stdin.resume();
let data = '';
process.stdin.on('data', c => data += c);
process.stdin.on('end', () => {
  const input = JSON.parse(data || '{}');
  const filePath = input.tool_input?.file_path || '';
  
  // 检测 API 文件变更
  if (filePath.match(/src\/(routes|api|controllers)\//)) {
    console.log(JSON.stringify({
      systemMessage: `检测到 API 文件变更：${filePath}\n请同步更新 docs/api/ 中对应的文档文件。`
    }));
  }
});
```

### 文档同步技能

```yaml
---
name: sync-docs
description: 同步 API 文档。当用户提到文档更新、API 变更同步时使用。
allowed-tools: Read, Write, Edit, Grep, Glob
---

# 文档同步任务

## 检查哪些文档需要更新
1. 找出最近修改的 API 文件：!`git diff --name-only HEAD~1 | grep 'src/routes\|src/api'`
2. 对每个变更的 API 文件，检查对应文档是否存在
3. 如果文档不存在，生成新文档
4. 如果文档存在，检查是否需要更新

## 文档格式标准
### {端点名} API

**路径**：`METHOD /api/v1/path`

**请求参数**：
| 参数 | 类型 | 必须 | 说明 |
|------|------|------|------|

**响应示例**：
```json
{
  "success": true,
  "data": {}
}
```

**错误码**：...
```

---

## 工作流5：多语言项目 Git 可移植性检查

### 场景
确保项目可以在任何开发者的机器上 clone 后直接运行，没有硬编码路径。

### 可移植性检查技能

```yaml
---
name: portability-check
description: 全面检查 git 克隆可移植性。当用户提到可移植性、迁移、环境配置时使用。
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob
---

# Git 可移植性检查

## 执行检查

### 1. 硬编码路径扫描
扫描所有追踪文件中的硬编码路径：
```bash
git ls-files | xargs grep -rn \
  '/home/\|/Users/\|C:\\Users\\\|/opt/homebrew\|/usr/local/lib' \
  2>/dev/null | grep -v '.min.js\|node_modules'
```

### 2. 追踪的本地配置文件
```bash
git ls-files | grep -E \
  '\.local\.|settings\.local\.|\.env$|\.env\.[^example]'
```

### 3. .gitignore 覆盖验证
检查以下文件是否已加入 .gitignore：
- `*.local.*`
- `.env`（非 .env.example）
- `settings.local.*`
- 操作系统文件（.DS_Store, Thumbs.db）

### 4. 相对路径验证
所有配置文件中的路径应使用相对路径：
```bash
grep -r 'file://' . --include="*.json" --include="*.yaml" 2>/dev/null
```

## 输出格式

### 可移植性报告

**评分：X/10**

| 检查项 | 状态 | 详情 |
|--------|------|------|
| 硬编码路径 | ✅/❌ | ... |
| 本地配置追踪 | ✅/❌ | ... |
| .gitignore 完整 | ✅/❌ | ... |

### 需要修复的问题
...

### 建议的修复命令
```bash
# 示例
git rm --cached settings.local.json
echo "settings.local.json" >> .gitignore
git add .gitignore
git commit -m "fix: remove local config from tracking"
```

## 自动修复选项
发现问题后询问：是否自动修复可修复的问题？
```

---

## 工作流6：每日技术债务审计定时任务

### 场景
每天早上自动扫描代码库，生成技术债务报告，追踪趋势。

### 定时任务配置

```bash
# 使用 /loop 每天执行一次
/loop 24h 执行每日技术债务审计并保存报告
```

### 审计技能

```yaml
---
name: debt-audit
description: 技术债务审计，生成可追踪的债务报告。每日审计或用户请求时使用。
disable-model-invocation: true
allowed-tools: Bash, Read, Write, Grep, Glob
---

# 每日技术债务审计

## 审计日期：!`date '+%Y-%m-%d'`
## 仓库：!`git remote get-url origin`
## 分支：!`git branch --show-current`

## 执行以下审计

### 1. TODO/FIXME/HACK 统计
```bash
git grep -n 'TODO\|FIXME\|HACK\|XXX' | wc -l
git grep -n 'TODO\|FIXME\|HACK\|XXX' | head -20
```

### 2. 过长文件检测（>300行）
```bash
git ls-files '*.ts' '*.js' '*.py' | xargs wc -l | \
  awk '$1 > 300 {print $1, $2}' | sort -rn | head -10
```

### 3. 重复代码检测
使用 grep 找出可能的重复函数名：
```bash
git ls-files '*.ts' '*.js' | \
  xargs grep -h 'function\|const.*=.*=>' | \
  sort | uniq -d | head -10
```

### 4. 依赖过期检测
```bash
npm outdated --json 2>/dev/null | python3 -c "
import sys, json
d = json.load(sys.stdin)
for k,v in d.items():
    if v.get('type') == 'major':
        print(f'[MAJOR] {k}: {v.get(\"current\")} → {v.get(\"latest\")}')
" | head -10
```

### 5. 测试覆盖率趋势
```bash
npm test -- --coverage --coverageReporters=text-summary 2>&1 | \
  grep 'Statements\|Branches\|Functions\|Lines'
```

## 保存报告
将报告追加到 `docs/tech-debt/$(date +%Y-%m).md`

## 输出格式

---
**日期**：!`date '+%Y-%m-%d'`

| 指标 | 今日值 | 趋势 |
|------|--------|------|
| TODO/FIXME 数 | N | ↑/↓/→ |
| 过长文件数 | N | ↑/↓/→ |
| 测试覆盖率 | N% | ↑/↓/→ |
| 过期依赖数 | N | ↑/↓/→ |

### 最需要关注的问题
1. ...
---
```

---

## 相关模块

- [11-integration-patterns.md](11-integration-patterns.md) — 模式级别的集成
- [03-skills.md](03-skills.md) — 技能配置
- [06-hooks.md](06-hooks.md) — Hook 自动化
- [10-cli.md](10-cli.md) — CI/CD 集成
- [14-exercises.md](14-exercises.md) — 动手实现这些工作流
