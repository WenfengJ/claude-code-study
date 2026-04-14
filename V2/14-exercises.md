# 模块 14：实践练习手册

> Level 1-3 | 预计完成时间：40小时 | 配套整个 v2 课程

---

## 使用说明

- 每个练习旁边标注了 **难度** 和 **预计时间**
- 按模块顺序完成，后面的练习依赖前面的知识
- 综合项目贯穿整个学习周期，每天添砖加瓦
- **动手原则**：先自己尝试，再看提示，不要直接复制答案

---

## 模块 1：Slash Commands 练习

### 练习 1.1：命令探索 ⭐ (30分钟)

**任务：**
1. 启动 `claude`，输入 `/help` 查看所有命令
2. 找出你最不熟悉的 5 个命令
3. 逐一测试这 5 个命令，在笔记中记录：这个命令做什么？什么场景最有用？
4. 使用 `/context` 查看当前上下文使用量

**验证：** 你应该能不查文档说出每个命令的用途

---

### 练习 1.2：/init 实战 ⭐ (45分钟)

**任务：**
1. 在你的一个真实项目中运行 `/init`
2. 查看生成的 CLAUDE.md，评估：
   - 是否识别出了正确的技术栈？
   - 是否遗漏了重要约定？
3. 手动补充至少 3 条项目特定规范
4. 再次与 Claude 对话，验证它是否遵守了你添加的规范

**验证：** Claude 在对话中主动遵守了你添加的约定

---

### 练习 1.3：/loop 定时任务 ⭐⭐ (30分钟)

**任务：**
1. 使用 `/loop 5m 检查时间并输出一条问候语` 创建一个简单循环
2. 观察 Claude 如何自动管理循环节奏
3. 中途修改循环指令，观察它如何适应
4. 用 Esc 或 `/stop` 结束循环

**挑战：** 创建一个每隔一定时间统计当前目录文件数量的循环

---

## 模块 2：Memory 练习

### 练习 2.1：CLAUDE.md 层级验证 ⭐ (45分钟)

**任务：**
1. 在 `~/.claude/CLAUDE.md`（全局）中添加：
   ```markdown
   ## 全局偏好
   - 所有代码注释使用中文
   - 函数必须有返回类型注解
   ```
2. 在某个项目的 `CLAUDE.md` 中添加：
   ```markdown
   ## 项目覆盖
   - 代码注释使用英文（覆盖全局设置）
   ```
3. 在该项目中与 Claude 对话，验证项目规范覆盖了全局规范
4. 在另一个没有项目 CLAUDE.md 的目录中验证全局规范生效

**验证：** 理解层级覆盖机制

---

### 练习 2.2：@import 模块化配置 ⭐⭐ (1小时)

**任务：**
1. 创建以下文件结构：
   ```
   .claude/
   ├── CLAUDE.md          # 主文件，使用 @import
   ├── rules/
   │   ├── code-style.md  # 代码风格规范
   │   ├── testing.md     # 测试约定
   │   └── security.md    # 安全规范
   ```
2. 在主 CLAUDE.md 中用 `@import` 组合：
   ```markdown
   @import rules/code-style.md
   @import rules/testing.md
   @import rules/security.md
   ```
3. 填充每个规则文件（至少 5 条规范）
4. 验证 Claude 在对话中遵守了所有规范

**挑战：** 添加 `@import .github/CONTRIBUTING.md`，验证跨目录导入

---

### 练习 2.3：自动记忆观察 ⭐ (30分钟)

**任务：**
1. 开启一个新对话，告诉 Claude 一些你的偏好：
   - "我是数据工程师，主要使用 Python 和 Spark"
   - "我不喜欢过度注释，只注释非显而易见的逻辑"
2. 结束对话
3. 开启新对话，观察 Claude 是否记住了你的偏好
4. 如果不满意自动记忆内容，用自然语言告诉 Claude 如何更新

**反思：** 自动记忆和 CLAUDE.md 各适合存储什么类型的信息？

---

## 模块 3：Skills 练习

### 练习 3.1：第一个 Skill ⭐ (1小时)

**任务：**
在你的项目中创建一个代码审查技能：

```yaml
# .claude/skills/review/SKILL.md
---
name: review
description: 代码审查。当用户说"审查"、"review"时使用。
disable-model-invocation: false
---

# 代码审查

请审查最近的代码变更：
!`git diff HEAD~1`

检查以下维度：
1. **逻辑正确性**：代码逻辑是否正确
2. **安全性**：有无明显安全问题
3. **可读性**：变量命名、函数长度是否合适

以 Markdown 格式输出审查报告。
```

测试：`/review` 或告诉 Claude "帮我审查代码"

**验证：** 技能正确触发，动态注入了 git diff 内容

---

### 练习 3.2：参数化技能 ⭐⭐ (1小时)

**任务：**
创建一个接受参数的文档生成技能：

```yaml
---
name: gen-docs
description: 生成函数或模块文档。使用方式：/gen-docs [文件路径]
argument-hint: "[文件路径]"
---

# 文档生成：$ARGUMENTS

请为以下文件生成完整的中文 API 文档：
- 文件路径：$ARGUMENTS
- 文档包含：函数签名、参数说明、返回值、使用示例

如果 $ARGUMENTS 为空，分析当前目录中最近修改的文件：
!`git diff --name-only HEAD~1 | head -5`
```

测试：
- `/gen-docs src/auth.ts` — 指定文件
- `/gen-docs` — 使用动态注入的最近修改文件

---

### 练习 3.3：限制工具的 Skill ⭐⭐ (45分钟)

**任务：**
创建一个只读分析技能（不允许修改文件）：

```yaml
---
name: analyze
description: 只读代码分析，不做任何修改。
allowed-tools: Read, Grep, Glob, Bash
disable-model-invocation: false
---

# 代码库分析

仅使用 Read、Grep、Glob 和只读 Bash 命令分析代码库。
**不允许修改任何文件**。

分析以下方面：
1. 代码结构（目录树）
2. 最大的文件（潜在过长）
3. TODO/FIXME 计数
4. 测试覆盖率（如果有）
```

验证 Claude 在这个技能中确实不尝试修改文件。

---

## 模块 4：Subagents 练习

### 练习 4.1：第一个子智能体 ⭐ (1小时)

**任务：**
创建一个代码审查员智能体：

```yaml
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: 专业代码审查，代码变更时主动使用。
tools: Read, Grep, Glob, Bash
---

你是资深代码审查员，有10年经验。

**审查标准：**
1. 安全性（OWASP Top 10）
2. 性能（N+1查询、内存泄漏）
3. 可维护性（函数长度、命名规范）
4. 测试覆盖

**输出格式：**
- 总体评级：🔴/🟡/🟢
- 严重问题列表（必须修复）
- 建议改进列表
- 亮点（好的代码值得表扬）
```

在对话中用"让 code-reviewer 审查这段代码"触发，或通过 Task 工具。

---

### 练习 4.2：并行子智能体 ⭐⭐ (2小时)

**任务：**
设计并实现一个并行分析系统：

1. 创建 3 个子智能体：
   - `security-auditor`：检查安全问题
   - `performance-analyst`：检查性能问题  
   - `test-coverage-checker`：检查测试覆盖

2. 在主对话中同时触发所有三个：
   ```
   并行执行：
   1. 让 security-auditor 扫描 src/ 目录
   2. 让 performance-analyst 分析数据库查询
   3. 让 test-coverage-checker 评估测试覆盖率
   
   完成后汇总三个报告
   ```

3. 观察并行执行和最终汇总

**反思：** 与串行执行相比，速度和质量有什么差异？

---

### 练习 4.3：Worktree 隔离智能体 ⭐⭐⭐ (3小时)

**任务：**
创建一个在 worktree 中独立工作的实现智能体：

```yaml
---
name: feature-implementer
description: 在独立分支实现新功能
isolation: worktree
tools: Read, Write, Edit, Bash, Grep, Glob
---

你是高级开发工程师，在独立的 git worktree 中实现功能。

工作流程：
1. 阅读需求和相关代码
2. 制定实现计划（列出要创建/修改的文件）
3. 实现功能
4. 编写基础测试
5. 汇报完成的工作和分支名称
```

触发：要求 Claude 用 `feature-implementer` 实现一个小功能（如"添加用户头像上传接口"），观察它如何在独立 worktree 中工作，完成后返回分支路径供你审查。

---

## 模块 5：MCP 练习

### 练习 5.1：配置 Filesystem MCP ⭐ (30分钟)

**任务：**
1. 在项目根目录创建 `.mcp.json`：
   ```json
   {
     "mcpServers": {
       "filesystem": {
         "command": "npx",
         "args": [
           "-y",
           "@modelcontextprotocol/server-filesystem",
           "./docs"
         ]
       }
     }
   }
   ```
2. 重启 Claude Code，验证 MCP 服务器连接
3. 让 Claude 使用 filesystem MCP 工具列出 docs/ 目录内容（而非 Bash 工具）

**验证：** `claude mcp list` 显示 filesystem 服务器状态为 connected

---

### 练习 5.2：MCP 与子智能体结合 ⭐⭐ (1.5小时)

**任务：**
1. 配置 GitHub MCP（需要 GITHUB_PERSONAL_ACCESS_TOKEN）
2. 创建一个使用 GitHub MCP 的智能体：
   ```yaml
   ---
   name: issue-tracker
   description: GitHub Issue 分析和管理
   tools: Read
   mcpServers: github
   ---
   
   你专门分析 GitHub Issues，提供优先级建议和处理方案。
   ```
3. 触发：让 `issue-tracker` 分析某个仓库最近10个 Issues
4. 验证智能体通过 MCP 而非 Bash 访问 GitHub

---

## 模块 6：Hooks 练习

### 练习 6.1：简单 PostToolUse Hook ⭐ (45分钟)

**任务：**
配置一个在每次文件修改后运行 lint 的 Hook：

1. 在 `.claude/settings.json` 中添加：
   ```json
   {
     "hooks": {
       "PostToolUse": [
         {
           "matcher": "Write|Edit",
           "hooks": [
             {
               "type": "command",
               "command": "npm run lint 2>&1 | tail -10"
             }
           ]
         }
       ]
     }
   }
   ```
2. 让 Claude 修改一个有 lint 错误的文件
3. 观察 Hook 如何捕获错误并注入到上下文
4. 验证 Claude 自动修复了 lint 错误

---

### 练习 6.2：PreToolUse 安全 Hook ⭐⭐ (1.5小时)

**任务：**
创建一个阻止危险命令的 Hook：

1. 创建 `scripts/safety-guard.js`（参考模块 11 的示例）
2. 添加你自己的危险模式（至少 5 条）
3. 配置为 PreToolUse Hook
4. 测试：让 Claude 尝试执行一个被阻止的命令
5. 验证阻止消息清晰地说明了原因，并建议了安全替代方案

**挑战：** 添加白名单功能——某些命令只有在特定文件存在时才被阻止

---

### 练习 6.3：SessionStart 上下文注入 Hook ⭐⭐ (1小时)

**任务：**
创建一个在每次 session 开始时注入 git 状态的 Hook：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Branch: $(git branch --show-current)\" > $CLAUDE_ENV_FILE && echo \"Status: $(git status --short | head -5)\" >> $CLAUDE_ENV_FILE && echo \"Last commit: $(git log --oneline -1)\" >> $CLAUDE_ENV_FILE"
          }
        ]
      }
    ]
  }
}
```

测试：重启 Claude Code，验证 Claude 在对话开始时就知道当前 git 状态。

---

## 模块 7：Plugins 练习

### 练习 7.1：分析一个现有插件 ⭐ (1小时)

**任务：**
1. 安装 `claude plugin install gh:luongnv89/claude-howto/plugins/portability-check`（如果该插件存在）或从 Marketplace 安装任意插件
2. 查看插件结构：
   - `plugin.json` 中定义了什么？
   - 包含哪些技能、智能体、Hook？
   - LSP 配置是否存在？
3. 测试插件的主要功能
4. 在笔记中记录：这个插件如何将多个组件整合为一个功能单元？

---

### 练习 7.2：创建简单插件 ⭐⭐⭐ (4小时)

**任务：**
为你的项目创建一个完整的插件，包含：
- 至少 2 个相关技能
- 1 个子智能体
- 1 个 Hook
- 基础 `plugin.json`

**示例：Git 工作流插件**

```
my-git-plugin/
├── plugin.json
├── skills/
│   ├── git-commit/SKILL.md   # 生成 conventional commit message
│   └── git-pr/SKILL.md       # 生成 PR 描述
├── agents/
│   └── git-reviewer.md       # 代码审查智能体
└── hooks/
    └── pre-push-check.js     # push 前的安全检查
```

**验证：** 用 `claude --plugin-dir ./my-git-plugin` 加载并测试所有功能

---

## 模块 8：Checkpoints 练习

### 练习 8.1：安全实验工作流 ⭐ (1小时)

**任务：**
使用检查点对比两种实现方案：

1. 在任意任务开始前记录检查点位置
2. 实现方案 A（例如：用数组存储数据）
3. 测试方案 A 的效果
4. Esc+Esc → 恢复到初始检查点
5. 实现方案 B（例如：用 Map 存储数据）
6. 对比两种方案
7. 选择更好的方案继续

**反思：** 这个工作流相比"先删除再重写"有什么优势？

---

### 练习 8.2：上下文管理策略 ⭐⭐ (45分钟)

**任务：**
1. 进行一个较长的对话（30+ 轮）
2. 使用 `/context` 查看使用量
3. 选择合适的时机，用 Esc+Esc → "从这里摘要" 压缩
4. 验证摘要后：
   - 文件是否完整保留？
   - Claude 是否还记得关键上下文？
   - 对话质量是否有提升？

---

## 模块 9：Advanced Features 练习

### 练习 9.1：Planning Mode 实战 ⭐⭐ (1.5小时)

**任务：**
1. 选择一个复杂任务（如：重构你项目中的某个模块）
2. 使用 `/plan` 请求 Claude 先规划
3. 仔细阅读生成的计划：
   - 步骤是否合理？
   - 是否遗漏了关键依赖？
   - 风险点是否正确识别？
4. 修改计划中你不同意的部分
5. 批准计划后让 Claude 执行
6. 对比：有计划和没计划的执行质量差异

---

### 练习 9.2：Extended Thinking 对比 ⭐ (45分钟)

**任务：**
对同一个复杂问题，分别在默认模式和 Extended Thinking 模式下提问：

问题示例：
```
分析这个架构决策的权衡：
我们是否应该将单体 Node.js 应用拆分为微服务？
项目规模：10万行代码，5人团队，每月10万用户
```

1. 默认模式下提问，记录回答的深度
2. `/effort max` 后提同样的问题
3. 对比两个答案的质量、深度、考虑维度

---

### 练习 9.3：/batch 批量处理 ⭐⭐⭐ (2小时)

**任务：**
使用 `/batch` 对项目做批量改动（选一个安全的改动）：

示例任务：
- 将所有 `console.log` 替换为 `logger.log`
- 统一所有 TypeScript 函数的返回类型注解
- 为所有缺少注释的 public 函数添加 JSDoc

**注意：** 在 git 仓库中操作，随时可以用 `git diff` 检查变更，`git checkout .` 回滚

---

## 模块 10：CLI 练习

### 练习 10.1：Print Mode 管道链 ⭐ (1小时)

**任务：**
构建一个代码审查管道：

```bash
# 任务：自动审查 PR 并生成报告
git diff origin/main...HEAD | \
  claude -p "
    审查这个 PR 的代码变更。
    重点：安全漏洞、性能问题、错误处理。
    输出 JSON：{score: 0-10, issues: [], highlights: []}
  " --output-format json | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'评分: {d[\"score\"]}/10')
print(f'问题数: {len(d[\"issues\"])}')
for i in d['issues']:
    print(f'  - {i}')
  "
```

修改这个脚本，使其：
1. 当分数低于 7 时以非零退出码退出
2. 将报告保存到 `review-report.json`

---

### 练习 10.2：GitHub Actions 集成 ⭐⭐⭐ (2小时)

**任务：**
在你的项目中配置 Claude Code CI/CD 集成：

1. 创建 `.github/workflows/ai-review.yml`（参考模块 10 的模板）
2. 在 GitHub Secrets 中添加 `ANTHROPIC_API_KEY`
3. 提交一个测试 PR，验证：
   - AI 审查评论是否出现在 PR 上
   - 质量门控是否正常工作

如果没有 GitHub 项目，在本地模拟：
```bash
# 模拟 CI 环境
export ANTHROPIC_API_KEY="your-key"
git diff origin/main | \
  claude -p "生成 PR 审查报告（Markdown）" \
  --permission-mode plan
```

---

## 综合项目：为自己的项目配置完整 Claude Code 环境

**目标：** 为你的一个真实项目（工作或个人项目）配置完整的 Claude Code 开发环境

**要求：**

### 1. 记忆层（3小时）

创建分层 CLAUDE.md：
```
.claude/
├── CLAUDE.md          # 主配置（导入其他文件）
└── rules/
    ├── tech-stack.md  # 技术栈和框架约定
    ├── code-style.md  # 代码风格
    ├── testing.md     # 测试策略
    └── workflow.md    # 开发工作流
```

### 2. Skills 库（3小时）

至少创建以下技能：
- `/review` — 代码审查
- `/docs` — 文档生成
- `/test` — 测试生成
- 一个项目特定技能（你最常重复的任务）

### 3. 智能体团队（2小时）

至少创建：
- `code-reviewer` — 代码审查员
- `test-engineer` — 测试工程师

### 4. Hooks 自动化（2小时）

配置：
- PostToolUse：文件修改后自动 lint
- SessionStart：注入 git 状态和项目信息
- Stop：验证任务完成质量

### 5. 可选 MCP（1小时）

根据项目类型添加合适的 MCP 服务器：
- GitHub 项目 → github MCP
- 有数据库 → postgres/sqlite MCP
- 有文档 → filesystem MCP

### 6. 验证（1小时）

完成配置后，执行以下验证：
- [ ] 技能正确触发
- [ ] 智能体能访问需要的工具
- [ ] Hooks 在正确的事件上执行
- [ ] MCP 服务器连接正常
- [ ] Claude 在所有对话中遵守代码规范
- [ ] `/portability-check` 验证配置的可移植性

---

## 高阶挑战：构建完整的 PR Review Plugin

**目标：** 将你学到的所有功能整合为一个完整的 PR 审查插件，可以发布给团队使用

**规格要求：**

### 插件结构

```
pr-review-plugin/
├── plugin.json                 # 插件元数据
├── skills/
│   ├── pr-review/SKILL.md      # 完整 PR 审查
│   ├── security-scan/SKILL.md  # 安全扫描
│   └── perf-check/SKILL.md     # 性能检查
├── agents/
│   ├── security-auditor.md     # 安全审查智能体
│   ├── perf-analyzer.md        # 性能分析智能体
│   └── review-summarizer.md    # 汇总审查结果
├── hooks/
│   ├── pre-commit-guard.js     # 提交前安全检查
│   └── post-push-notify.js     # 推送后通知
└── README.md                   # 使用说明
```

### 功能要求

**1. /pr-review 技能**
- 动态获取当前 PR 的 diff
- 并行触发安全审查、性能检查、代码质量审查三个智能体
- 汇总所有结果为统一的审查报告
- 评分：0-10，低于 7 输出警告

**2. 安全审查智能体**
- 检查 OWASP Top 10
- 检查硬编码密钥
- 检查不安全的依赖

**3. PreToolUse Hook**
- 阻止直接 push 到 main
- 阻止删除数据库迁移文件
- 记录审计日志

**4. Plugin.json**
```json
{
  "name": "pr-review-suite",
  "version": "1.0.0",
  "description": "完整的 PR 审查套件",
  "userConfig": {
    "strictMode": {
      "type": "boolean",
      "description": "严格模式：低于7分自动阻止",
      "default": false
    },
    "notifySlack": {
      "type": "boolean",
      "description": "审查完成后通知 Slack",
      "default": false
    }
  },
  "sensitive": {
    "slackWebhookUrl": {
      "description": "Slack Webhook URL（notifySlack 为 true 时必填）"
    }
  }
}
```

### 验收标准

- [ ] 插件可以通过 `claude --plugin-dir ./pr-review-plugin` 加载
- [ ] `/pr-review` 正确生成审查报告
- [ ] 三个智能体并行执行，主 Claude 汇总
- [ ] PreToolUse Hook 正确阻止危险操作
- [ ] strictMode 配置项生效
- [ ] README 包含安装和使用说明

**时间估计：** 8-12小时（分2-3天完成）

---

## 学习进度追踪

| 模块 | 练习 1 | 练习 2 | 练习 3 | 完成日期 |
|------|--------|--------|--------|---------|
| M1 Slash Commands | ⬜ | ⬜ | ⬜ | |
| M2 Memory | ⬜ | ⬜ | ⬜ | |
| M3 Skills | ⬜ | ⬜ | ⬜ | |
| M4 Subagents | ⬜ | ⬜ | ⬜ | |
| M5 MCP | ⬜ | ⬜ | — | |
| M6 Hooks | ⬜ | ⬜ | ⬜ | |
| M7 Plugins | ⬜ | ⬜ | — | |
| M8 Checkpoints | ⬜ | ⬜ | — | |
| M9 Advanced | ⬜ | ⬜ | ⬜ | |
| M10 CLI | ⬜ | ⬜ | — | |
| 综合项目 | ⬜ | ⬜ | ⬜ | |
| 高阶挑战 | ⬜ | | | |

---

## 相关模块

- [00-index.md](00-index.md) — 40小时学习计划总览
- [13-troubleshooting.md](13-troubleshooting.md) — 遇到问题时的排查指南
- [11-integration-patterns.md](11-integration-patterns.md) — 综合项目参考
- [12-real-world-workflows.md](12-real-world-workflows.md) — 高阶挑战参考
