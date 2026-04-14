# 模块 2：Memory（记忆系统）

> Level 1 基础 | 预计学习时间：45分钟

---

## 核心概念

Memory 让 Claude 在多个会话间保持上下文。Claude Code 使用基于文件系统的 **CLAUDE.md** 实现持久化记忆。

---

## 记忆层级（优先级从高到低）

| 层级 | 位置 | 范围 | 用途 |
|------|------|------|------|
| **1. 管理策略** | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS）<br>`C:\Program Files\ClaudeCode\CLAUDE.md`（Windows） | 组织级 | 公司统一策略 |
| **2. 管理 Drop-ins** | `managed-settings.d/*.md`（v2.1.83+） | 组织级 | 模块化策略文件 |
| **3. 项目记忆** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享 | 团队规范（提交到 git） |
| **4. 项目规则** | `./.claude/rules/*.md` | 团队共享 | 路径特定的模块化规则 |
| **5. 用户记忆** | `~/.claude/CLAUDE.md` | 个人（所有项目） | 个人偏好 |
| **6. 用户规则** | `~/.claude/rules/*.md` | 个人（所有项目） | 个人规则 |
| **7. 本地项目记忆** | `./CLAUDE.local.md` | 个人（当前项目） | 不提交的个人偏好 |
| **8. 自动记忆** | `~/.claude/projects/<project>/memory/` | 个人 | Claude 自动写入的笔记 |

> **优先级规则**：数字越小，优先级越高。

---

## 快速上手

### 初始化项目记忆
```bash
/init
# Claude 在项目根目录创建 CLAUDE.md 模板
```

### 编辑记忆文件
```bash
/memory
# 在系统编辑器中打开记忆文件
```

### 让 Claude 记住某件事
```
记住：我们项目始终使用 TypeScript strict 模式
请把这个加入记忆：优先使用 async/await 而非 Promise 链
```

---

## CLAUDE.md 写法示例

```markdown
# 项目配置

## 技术栈
- 语言：TypeScript（strict 模式）
- 后端：Node.js + Express
- 数据库：PostgreSQL
- 测试：Jest + Cypress

## 代码规范
- 缩进：2个空格
- 命名：文件 kebab-case，类 PascalCase，函数 camelCase
- 最大行长：100字符

## Git 规范
- 分支：feature/xxx 或 fix/xxx
- 提交：遵循 Conventional Commits
- PR 合并前需至少1人审批

## 常用命令
| 命令 | 用途 |
|------|------|
| `npm run dev` | 启动开发服务器 |
| `npm test` | 运行测试 |
| `npm run build` | 构建生产版本 |

## 已知问题
- PostgreSQL 连接池高峰期限制20个连接
```

---

## `@` 导入语法

在 CLAUDE.md 中引用其他文件，避免重复内容：

```markdown
# 项目文档
@README.md
@docs/architecture.md
@package.json
```

- 支持相对路径和绝对路径
- 最多支持5层递归嵌套
- 首次从外部位置导入会触发安全确认

---

## 路径特定规则（.claude/rules/）

```
project/
├── .claude/
│   ├── CLAUDE.md
│   └── rules/
│       ├── code-style.md       # 全局风格规则
│       ├── testing.md          # 测试规则
│       └── api/
│           └── conventions.md  # 仅对 API 代码生效
```

在规则文件中用 frontmatter 限定路径：
```markdown
---
paths: src/api/**/*.ts
---

# API 开发规则
- 所有端点必须包含输入校验
- 使用 Zod 做 schema 验证
```

---

## 自动记忆（Auto Memory）

Claude 在工作过程中自动记录发现的模式和规律：

- **入口文件**：`~/.claude/projects/<project>/memory/MEMORY.md`
- **加载规则**：session 启动时自动加载前200行（或25KB）
- **主题文件**：`debugging.md`、`api-conventions.md` 等，按需加载
- **版本要求**：Claude Code v2.1.59+

```bash
# 禁用自动记忆
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 claude

# 强制启用
CLAUDE_CODE_DISABLE_AUTO_MEMORY=0 claude
```

---

## 排除不需要的 CLAUDE.md（大型 monorepo）

```json
// ~/.claude/settings.json 或 .claude/settings.json
{
  "claudeMdExcludes": [
    "packages/legacy-app/CLAUDE.md",
    "vendors/**/CLAUDE.md"
  ]
}
```

---

## 最佳实践

**要做的：**
- 用具体指令，不用模糊表述（✅ "2空格缩进" ❌ "遵循最佳实践"）
- 项目记忆提交到 git，团队共享
- 用 `@path` 引用已有文档，避免重复
- 定期回顾更新，保持记忆文件新鲜

**不要做的：**
- 不要存储 API 密钥、密码等敏感信息
- 不要让单个文件超过500行
- 不要在 CLAUDE.md 中复制可以用 `@` 导入的内容

---

## 相关模块

- [01-slash-commands.md](01-slash-commands.md) — `/init` 和 `/memory` 命令
- [03-skills.md](03-skills.md) — 结合记忆使用技能
- [07-plugins.md](07-plugins.md) — 管理级配置分发
