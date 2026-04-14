# 模块 3：Skills（技能）完全指南

> Level 2 | 预计学习时间：3小时 | Claude Code v2.1.101+

---

## 核心架构：渐进式加载

```
Level 1：元数据（启动时加载）  ~100 tokens/技能
   ↓ 触发时
Level 2：SKILL.md 正文        < 5k tokens
   ↓ 需要时
Level 3：附带文件              实际无限制（脚本/模板/文档）
```

可安装上百个技能，不消耗上下文。只有被触发的技能才加载完整内容。

---

## 目录结构

```
.claude/skills/<技能名>/
├── SKILL.md           # 主指令（必须）
├── templates/         # 模板文件
│   └── output.md
├── examples/          # 示例输出
│   └── sample.md
├── references/        # 领域知识文档
│   └── api-spec.md
└── scripts/           # 可执行脚本
    └── validate.sh
```

**存储位置（优先级从高到低）：**

| 类型 | 位置 | 范围 | 覆盖关系 |
|------|------|------|---------|
| 企业级 | 管理员配置 | 全组织 | 最高优先级 |
| 个人 | `~/.claude/skills/<name>/SKILL.md` | 所有项目 | 覆盖项目级 |
| 项目 | `.claude/skills/<name>/SKILL.md` | 当前项目（git） | 最低优先级 |
| 插件 | `<plugin>/skills/<name>/SKILL.md` | 插件范围 | 独立命名空间 |

插件技能使用 `plugin-name:skill-name` 命名空间，不会与其他技能冲突。

---

## SKILL.md 完整字段参考

```yaml
---
# === 必须字段 ===
name: skill-name
# 规则：小写字母、数字、连字符，最多64字符
# 禁止包含 "anthropic" 或 "claude"

description: |
  这个技能做什么，以及何时使用它。
  包含触发关键词很重要，例如："当用户提到代码审查、PR 分析或安全检查时使用"
# 规则：最多1024字符，Claude 根据此字段决定是否自动调用

# === 触发控制 ===
disable-model-invocation: true
# true = 只有用户能触发（防止 Claude 自动调用有副作用的技能）
# 适合：/deploy、/commit、/send-notification

user-invocable: false
# false = 只有 Claude 能自动调用（隐藏出 / 菜单）
# 适合：背景知识类技能，不适合用户手动调用

# === 工具权限 ===
allowed-tools: Read, Grep, Glob, Bash(npm:*)
# 限制这个技能可以使用哪些工具
# 支持工具名、Bash(pattern:*) 条件限制

# === 模型配置 ===
model: opus
# 覆盖模型：opus / sonnet / haiku / 完整模型 ID
effort: high
# 推理强度：low / medium / high / max

# === 执行隔离 ===
context: fork
# fork = 在独立子智能体中运行（有独立上下文窗口）
agent: Explore
# 子智能体类型：Explore / Plan / general-purpose / 自定义

# === Shell ===
shell: bash
# bash（默认）或 powershell

# === 路径限制 ===
paths: "src/api/**/*.ts"
# 仅在匹配这些路径时自动激活
# 可用逗号分隔多个模式

# === 命令补全 ===
argument-hint: "[filename] [format]"
# 在 / 菜单中显示的参数提示

# === 组件级钩子 ===
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
          timeout: 30
  Stop:
    - hooks:
        - type: command
          command: "./scripts/notify.sh"
---

# 技能正文从这里开始
```

---

## 变量与动态注入

### 变量替换

| 变量 | 说明 | 示例 |
|------|------|------|
| `$ARGUMENTS` | 所有参数 | `/fix-issue 123` → `$ARGUMENTS` = `123` |
| `$0`、`$1`... | 按索引的参数 | `/deploy prod v2.0` → `$1` = `prod` |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID | - |
| `${CLAUDE_SKILL_DIR}` | SKILL.md 所在目录 | 引用相对路径文件时用 |

### 动态上下文注入

`` !`command` `` 语法在技能内容发送给 Claude **之前**执行 shell 命令：

```yaml
---
name: pr-review
description: 审查当前 PR 的代码变更
context: fork
agent: Explore
---

## 当前 PR 信息

- **分支**：!`git branch --show-current`
- **差异文件**：!`gh pr diff --name-only`
- **提交数**：!`git rev-list --count HEAD ^main`
- **PR 描述**：
  !`gh pr view --json title,body --jq '"标题：" + .title + "\n描述：" + .body'`

## 审查任务

基于以上信息，进行全面代码审查：
1. 安全漏洞检查
2. 性能影响分析
3. 代码质量评估
4. 测试覆盖率验证
```

---

## 触发控制详解

| 配置 | 用户 `/name` | Claude 自动 | 适用场景 |
|------|------------|------------|---------|
| 默认 | ✅ | ✅ | 普通技能 |
| `disable-model-invocation: true` | ✅ | ❌ | 部署、提交等有副作用操作 |
| `user-invocable: false` | ❌ | ✅ | 背景知识、领域上下文 |

**描述字符预算：** 技能描述总量上限为上下文窗口的1%（fallback：8000字符），每条上限250字符。超过时描述被截断，但技能名始终显示。可通过环境变量调整：
```bash
export SLASH_COMMAND_TOOL_CHAR_BUDGET=5000
```

---

## 6个完整技能示例

### 示例1：代码安全审查

```yaml
---
name: security-audit
description: |
  对代码进行深度安全审查。
  当用户提到安全审查、漏洞检测、OWASP、SQL注入、XSS时自动使用。
context: fork
agent: Explore
---

# 安全审查技能

## 审查范围
1. **注入漏洞**：SQL注入、命令注入、XSS
2. **认证授权**：身份验证绕过、权限越界
3. **数据暴露**：敏感数据明文存储、日志泄露
4. **依赖安全**：过期依赖、已知漏洞
5. **配置安全**：硬编码密钥、不安全的默认配置

## 输出格式

### 安全评分：X/10

### 严重问题（立即修复）
| 漏洞 | 位置 | 影响 | 修复方案 |
|------|------|------|---------|
| ... | ... | ... | ... |

### 中等问题（计划修复）
...

### 建议改进
...
```

### 示例2：Git 可移植性检查

```yaml
---
name: portability-check
description: |
  检查项目的 git 克隆可移植性。
  当用户提到可移植性、硬编码路径、git克隆、环境迁移时使用。
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob
---

# Git 可移植性检查

## 执行步骤

1. **扫描硬编码路径**
   ```bash
   git ls-files | xargs grep -rn '/home/\|/Users/\|C:\\Users\\' 2>/dev/null
   ```

2. **检查追踪的本地配置文件**
   ```bash
   git ls-files | grep -E '\.local\.|settings\.local\.|\.env'
   ```

3. **验证 .gitignore 覆盖**
   - 检查 *.local.* 是否已在 .gitignore
   - 检查 .env* 是否已在 .gitignore
   - 检查个人路径配置是否已排除

4. **生成报告**

## 输出格式

### 可移植性评分：X/10

### 发现的问题
| 文件 | 行号 | 问题 | 修复建议 |
|------|------|------|---------|
| ... | ... | ... | ... |

### 需要加入 .gitignore 的文件
...

### 建议的修复命令
```bash
git rm --cached settings.local.json
echo "settings.local.json" >> .gitignore
```
```

### 示例3：Flink SQL 验证技能

```yaml
---
name: flink-sql-check
description: |
  验证 Flink SQL 作业的正确性和性能。
  当用户处理 Flink SQL 文件、水位线、窗口函数、Kafka connector 时自动使用。
paths: "**/*.sql, **/flink/**"
allowed-tools: Read, Bash, Grep
---

# Flink SQL 检查技能

## 检查维度

### 1. 语法完整性
- DDL 表是否有 PRIMARY KEY（如需 changelog 流）
- WATERMARK 定义是否正确
- CONNECTOR 属性是否完整

### 2. 性能检查
- GROUP BY 窗口是否有合理的 TUMBLE/HOP 窗口大小
- 是否存在无界流的非窗口聚合（危险！）
- JOIN 是否使用了时间区间条件

### 3. Kafka Connector 规范
```sql
-- 正确格式
'connector' = 'kafka',
'topic' = 'your-topic',
'properties.bootstrap.servers' = '${kafka.bootstrap.servers}',
'format' = 'json'
```

### 4. 命名规范验证
- topic 命名：`<业务域>.<实体>.<操作>`
- 临时表前缀：`tmp_`
- 结果表前缀：`sink_`

## 输出报告
...
```

### 示例4：自动部署（仅用户触发）

```yaml
---
name: deploy
description: 将应用部署到指定环境
disable-model-invocation: true
argument-hint: "[env: dev|staging|prod] [version]"
allowed-tools: Bash(docker:*), Bash(kubectl:*), Bash(git:*), Read
---

# 部署技能

将 $0 环境部署版本 $1。

## 预检查
1. 确认目标环境：$0
2. 确认版本：$1
3. 验证 Docker 镜像存在
4. 检查健康端点可访问

## 部署步骤
1. `git tag $1` 打标签
2. `docker build -t myapp:$1 .`
3. `docker push registry/myapp:$1`
4. `kubectl set image deployment/myapp myapp=registry/myapp:$1`
5. `kubectl rollout status deployment/myapp`
6. 验证部署：curl 健康端点

## 回滚方案
如果部署失败：`kubectl rollout undo deployment/myapp`
```

### 示例5：品牌声音（背景知识，仅自动触发）

```yaml
---
name: brand-voice
description: |
  确保所有对外沟通符合品牌声音和风格指南。
  当创建营销文案、客户邮件、产品文档、博客文章时自动使用。
user-invocable: false
---

# 品牌声音指南

## 语调
- **友好而专业**：亲切但不随意，专业但不冷漠
- **清晰简洁**：避免术语，一句话说清一个意思
- **自信有力**：我们清楚自己在做什么
- **以用户为中心**：关注他们的需求，不是我们的功能

## 写作规范
- 使用"你"而非"用户"
- 使用主动语态
- 句子不超过20个单词
- 先说价值，再说方法

## 禁止用词
- 禁止：很简单、非常容易（显得傲慢）
- 禁止：独一无二、前所未有（过度夸张）
- 禁止：其实、说实话（暗示之前不诚实）
```

### 示例6：CLAUDE.md 生成器

```yaml
---
name: claude-md
description: |
  为项目创建或更新优质的 CLAUDE.md 文件。
  当用户提到 CLAUDE.md、项目记忆、AI 入职、为 Claude 写说明时使用。
---

# CLAUDE.md 生成器

## 核心原则

1. **少即是多**：300行以内（理想100行以内）
2. **普遍适用**：只放每次 session 都用到的信息
3. **不用 Claude 做 Linter**：用确定性工具（ESLint/tsc）
4. **禁止自动生成**：手工精心撰写，不要一键生成大段内容

## 必须包含的章节

- **项目名称**：一句话描述
- **技术栈**：语言、框架、数据库
- **开发命令**：安装、测试、构建
- **关键约定**：非显而易见的、影响大的约定
- **已知问题/注意事项**：容易踩坑的点

## 不要放的内容
- 完整 README（用 @README.md 引入）
- 详细架构文档（用 @docs/architecture.md 引入）
- 秘钥和凭据（绝对禁止）
- 显而易见的通用约定（如"使用有意义的变量名"）

## 生成步骤
1. 读取 package.json / pyproject.toml 了解技术栈
2. 读取现有 README.md
3. 分析主要目录结构
4. 生成简洁的 CLAUDE.md
```

---

## 技能管理

### 查看可用技能
```bash
/skills              # 在会话中列出所有技能
ls ~/.claude/skills/ # 列出个人技能
ls .claude/skills/   # 列出项目技能
```

### 创建技能
```bash
# 快速创建
/agents              # → 选择 "Create New Skill"

# 手动创建
mkdir -p .claude/skills/my-skill
cat > .claude/skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: 技能说明（包含触发关键词）
---

# 技能内容
EOF
```

### 测试技能
```bash
# 方式1：自动触发测试
# 说一句匹配 description 关键词的话

# 方式2：直接调用
/my-skill [参数]

# 方式3：让 Claude 检查可用技能
"有哪些可用的技能？"
```

### 权限控制
```bash
# 在 /permissions 中禁用所有技能
Skill

# 只允许特定技能
Skill(commit)
Skill(review-pr *)

# 禁止特定技能
# 在 disallowedTools 中添加 Skill(dangerous-skill)
```

---

## 调试技巧

| 问题 | 解决方案 |
|------|---------|
| Claude 不自动触发技能 | 在 description 中加更多匹配关键词 |
| 技能触发太频繁 | 缩小 description，或加 `disable-model-invocation: true` |
| Claude 看不到所有技能 | 运行 `/context` 查看是否有描述被截断的警告 |
| 脚本不执行 | 检查权限：`chmod +x scripts/*.sh` |
| YAML 解析错误 | 检查 `---` 标记，确保缩进正确，不用 tab |

---

## 相关模块

- [01-slash-commands.md](01-slash-commands.md) — `/skill-name` 调用、内置技能
- [04-subagents.md](04-subagents.md) — `context: fork` 子智能体执行
- [06-hooks.md](06-hooks.md) — 技能内的生命周期 Hook
- [07-plugins.md](07-plugins.md) — 将技能打包进插件
- [11-integration-patterns.md](11-integration-patterns.md) — Memory + Skills 集成
