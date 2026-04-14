# 模块 2：Memory（记忆系统）完全指南

> Level 1 | 预计学习时间：3小时 | Claude Code v2.1.101+

---

## 记忆系统全景

Claude Code 记忆体系分两类：
- **你写的记忆**：CLAUDE.md、rules/、CLAUDE.local.md
- **Claude 自动写的记忆**：auto memory（`~/.claude/projects/.../memory/`）

---

## 8层记忆层级（完整优先级表）

| 优先级 | 类型 | 路径 | 范围 | 是否提交 git |
|--------|------|------|------|-------------|
| **1**（最高） | 企业托管策略 | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS）<br>`/etc/claude-code/CLAUDE.md`（Linux）<br>`C:\Program Files\ClaudeCode\CLAUDE.md`（Win） | 组织全体 | 系统级 |
| **2** | 企业 Drop-ins | `managed-settings.d/*.md`（v2.1.83+） | 组织全体 | 系统级 |
| **3** | 项目记忆 | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队 | ✅ 是 |
| **4** | 项目规则 | `./.claude/rules/*.md` | 团队 | ✅ 是 |
| **5** | 用户记忆 | `~/.claude/CLAUDE.md` | 个人（所有项目） | 否 |
| **6** | 用户规则 | `~/.claude/rules/*.md` | 个人（所有项目） | 否 |
| **7** | 本地项目记忆 | `./CLAUDE.local.md` | 个人（当前项目） | ❌ 否（.gitignore） |
| **8**（最低） | 自动记忆 | `~/.claude/projects/<hash>/memory/` | 个人 | 否 |

---

## 快速初始化

```bash
# 方式1：/init（推荐，自动生成模板）
/init

# 方式2：增强版交互式初始化
CLAUDE_CODE_NEW_INIT=1 claude
/init

# 方式3：手动创建
touch CLAUDE.md
# 然后编辑内容
```

---

## CLAUDE.md 写法指南与模板

### 模板1：通用后端项目

```markdown
# 项目：[项目名]

## 技术栈
- 语言：TypeScript 5.x（strict 模式）
- 运行时：Node.js 20 LTS
- 框架：Express 4 / Fastify
- 数据库：PostgreSQL 16 + Prisma ORM
- 缓存：Redis 7
- 测试：Jest + Supertest
- 容器：Docker + docker-compose

## 开发命令
```bash
npm run dev          # 启动开发服务器（热重载）
npm test             # 运行测试套件
npm run test:watch   # 监听模式测试
npm run lint         # ESLint 检查
npm run build        # 生产构建
npx prisma migrate dev  # 执行数据库迁移
```

## 代码规范
- 缩进：2个空格
- 命名：文件 kebab-case | 类 PascalCase | 函数/变量 camelCase | 常量 UPPER_SNAKE_CASE
- 最大行长：100字符
- 优先使用 async/await，不用 Promise 链
- 所有公共函数必须有 JSDoc 注释
- 禁止使用 `any` 类型

## Git 规范
- 分支：`feature/xxx`、`fix/xxx`、`chore/xxx`
- Commit：遵循 Conventional Commits（feat/fix/docs/refactor/test/chore）
- PR 前必须通过 CI

## API 规范
- RESTful 风格
- 所有路由前缀：`/api/v1/`
- 统一响应格式：`{ success, data, error, timestamp }`
- 使用 Zod 做输入校验
- 错误码：遵循 HTTP 标准状态码

## 数据库约定
- 表名：snake_case 复数（`user_accounts`）
- 所有变更通过 migration，禁止手动改数据库
- 敏感操作必须在事务中执行

## 已知问题
- 数据库连接池高峰期限制20个，高并发时需开启查询队列
- Safari 14 的 async generator 兼容性问题，已用 Babel 处理
```

### 模板2：前端 React 项目

```markdown
# 项目：[前端项目名]

## 技术栈
- React 18 + TypeScript（strict）
- Vite 5 构建
- Tailwind CSS 3
- React Query（服务端状态）+ Zustand（客户端状态）
- React Hook Form + Zod
- Vitest + Testing Library

## 开发命令
```bash
npm run dev          # 启动开发服务器
npm test             # Vitest 测试
npm run build        # 生产构建
npm run preview      # 预览构建结果
npm run typecheck    # 类型检查
```

## 组件规范
- 所有组件使用函数组件 + Hooks，禁止 Class Components
- 组件文件名：PascalCase（`UserCard.tsx`）
- Props 必须定义 interface，后缀 `Props`（`UserCardProps`）
- 复杂逻辑抽取为自定义 Hook（`useUserCard`）

## 目录结构
```
src/
├── components/     # 可复用 UI 组件
├── pages/          # 路由级页面组件
├── hooks/          # 自定义 Hooks
├── stores/         # Zustand store
├── services/       # API 调用层
├── types/          # TypeScript 类型
└── utils/          # 工具函数
```
```

### 模板3：个人偏好（~/.claude/CLAUDE.md）

```markdown
# 我的开发偏好

## 关于我
- 经验：8年全栈开发
- 主攻：TypeScript / Python 后端
- 当前学习：大数据 / Flink SQL
- 习惯：TDD，写测试再写实现

## 沟通风格
- 解释请给具体例子，不要只说理论
- 复杂问题先给结论，再展开细节
- 代码变更请给 diff 风格的 before/after

## 代码偏好
- 错误处理：显式 try/catch，有意义的错误信息
- 注释：解释 WHY，不解释 WHAT（代码本身说明）
- 测试：优先测试行为，不测实现细节

## 工具偏好
- IDE：VS Code + vim 键位
- 格式化：Prettier（100字符行宽）
- Shell：Zsh + Oh-My-Zsh
```

### 模板4：数据工程项目（Flink/大数据）

```markdown
# 项目：数据总线平台

## 技术栈
- 流处理：Apache Flink 1.18
- SQL 层：Flink SQL
- 消息队列：Apache Kafka
- 存储：Apache Iceberg + HDFS
- 元数据：Apache Hive Metastore
- 调度：Apache Airflow

## 关键约定
- Flink Job 类名：`XxxJob`，入口 `main` 方法
- SQL 文件命名：`<source>_to_<sink>_<purpose>.sql`
- 所有 DDL 必须有 COMMENT 字段说明含义
- Checkpoint 间隔：生产环境 60s，测试 10s

## 常用命令
```bash
flink run -c com.company.XxxJob target/app.jar  # 提交作业
flink list                                       # 查看运行中的作业
flink cancel <jobId>                             # 取消作业
```
```

---

## `@` 导入语法完整参考

```markdown
# 在 CLAUDE.md 中使用 @ 引入其他文件

# 相对路径（推荐）
@README.md
@docs/architecture.md
@package.json
@src/types/index.ts    # 让 Claude 了解类型定义

# 绝对路径
@~/.claude/shared-conventions.md

# 嵌套：导入的文件也可以 @ 其他文件（最多5层）
# file-a.md 可以 @file-b.md，file-b.md 可以 @file-c.md
```

**注意：**
- 第一次从外部位置导入会触发安全确认
- markdown 代码块内的 `@xxx` 不会被执行（安全）
- 同步文档变更：只需更新源文件，CLAUDE.md 的引用自动生效

---

## 路径特定规则（.claude/rules/）深度

### 目录结构
```
.claude/
└── rules/
    ├── general.md          # 通用规则（所有文件生效）
    ├── testing.md          # 测试规则
    ├── security.md         # 安全规则
    ├── api/
    │   ├── rest.md         # REST API 规则
    │   └── graphql.md      # GraphQL 规则（如有）
    └── database/
        └── migrations.md  # 数据库迁移规则
```

### 规则文件格式（含路径限定）

```markdown
---
paths: src/api/**/*.ts
---

# API 开发规则

## 输入校验
- 所有端点必须用 Zod 做 schema 校验
- 校验失败返回 400，包含字段级错误详情
- 禁止直接使用 `req.body` 未校验的数据

## 认证
- 所有需要认证的端点使用 `requireAuth` 中间件
- JWT 有效期：access token 15分钟，refresh token 7天
```

```markdown
---
paths: "**/*.test.ts, **/*.spec.ts"
---

# 测试规则

- 测试文件必须和被测文件同目录
- 测试描述使用中文（便于阅读报告）
- 不要 mock 数据库——使用测试数据库
- 每个测试用例结束后清理测试数据
```

### Glob 模式参考

| 模式 | 匹配 |
|------|------|
| `**/*.ts` | 所有 TypeScript 文件 |
| `src/**/*` | src 目录下所有文件 |
| `src/**/*.{ts,tsx}` | src 下的 ts 和 tsx |
| `{src,lib}/**/*.ts` | src 或 lib 下的 ts |
| `!node_modules/**` | 排除 node_modules |

---

## 自动记忆（Auto Memory）深度配置

### 工作机制

```
session 启动时：
  └→ 加载 MEMORY.md 前200行（或25KB）注入系统提示

session 运行时：
  └→ Claude 发现有价值的模式/规律
  └→ 写入 MEMORY.md 或新的主题文件（debugging.md 等）
  └→ 主题文件：按需加载，不在启动时全部加载
```

### 目录结构

```
~/.claude/projects/<project-hash>/memory/
├── MEMORY.md              # 入口文件（启动时加载前200行）
├── debugging.md           # 调试模式笔记（按需加载）
├── api-conventions.md     # API 约定（按需加载）
├── performance-notes.md   # 性能观察（按需加载）
└── architecture.md        # 架构理解（按需加载）
```

### 配置选项

```json
// ~/.claude/settings.json（只能在用户级或本地设置中配置）
{
  "autoMemoryDirectory": "/path/to/custom/memory"
}
```

```bash
# 控制自动记忆
CLAUDE_CODE_DISABLE_AUTO_MEMORY=0    # 强制启用
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1    # 强制禁用
# （不设置）                          # 默认：启用
```

### 子智能体中的记忆

```yaml
---
name: researcher
memory: user      # 只加载用户级记忆
# memory: project # 只加载项目级记忆
# memory: local   # 只加载本地记忆
---
```

---

## 排除不需要的 CLAUDE.md

```json
// ~/.claude/settings.json 或 .claude/settings.json
{
  "claudeMdExcludes": [
    "packages/legacy-app/CLAUDE.md",
    "vendors/**/CLAUDE.md",
    "docs/examples/**"
  ]
}
```

---

## 加载额外目录

```bash
# 启用 --add-dir 功能
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 claude --add-dir /path/to/shared-context

# 使用场景：monorepo 中工作在某个包，同时需要根目录的 CLAUDE.md
```

---

## 企业托管策略（Managed Settings）

### 部署方式
- **macOS**：plist 文件
- **Windows**：注册表
- **Linux**：JSON 文件（`/etc/claude-code/CLAUDE.md`）

### Managed Drop-ins（v2.1.83+）
```
/Library/Application Support/ClaudeCode/
├── CLAUDE.md              # 主策略文件
└── managed-settings.d/
    ├── 01-security.md     # 安全策略（按字母顺序合并）
    ├── 02-coding-style.md # 编码规范
    └── 03-compliance.md   # 合规要求
```

Drop-ins 按字母顺序合并，支持团队不同部门维护不同策略文件。

---

## 设置文件层级

```json
// ~/.claude/settings.json（用户级）
{
  "autoMemoryDirectory": "~/my-claude-memory",
  "claudeMdExcludes": ["vendors/**"],
  "permissionMode": "default"
}
```

```json
// .claude/settings.json（项目级，提交 git）
{
  "permissionMode": "acceptEdits",
  "allowedTools": ["Read", "Write", "Edit", "Bash", "Grep", "Glob"]
}
```

```json
// .claude/settings.local.json（本地，不提交）
{
  "permissionMode": "dontAsk"
}
```

**优先级：** 企业托管 > 用户级 > 项目级 > 本地

---

## 记忆更新工作流

### 方式1：对话中请求
```
记住：我们项目的 Kafka topic 命名规范是 <业务域>.<实体>.<操作>
请加入到项目记忆：所有 Flink job 的 checkpoint 目录是 hdfs:///flink/checkpoints/
```

### 方式2：/memory 命令
```bash
/memory
# 选择要编辑的记忆文件
# → 在系统编辑器中打开
# → 修改保存
# → Claude 自动重新加载
```

### 方式3：直接编辑文件
```bash
# 项目记忆
code ./CLAUDE.md

# 个人记忆
code ~/.claude/CLAUDE.md

# 自动记忆
code ~/.claude/projects/<hash>/memory/MEMORY.md
```

---

## 最佳实践

### 写什么进记忆

✅ **适合放入记忆的内容：**
- 项目技术栈和版本（团队必须统一）
- 代码规范（具体的，如"2空格缩进"而非"遵循最佳实践"）
- 常用命令（频繁使用的）
- 已知问题和绕过方案
- 团队约定（分支命名、PR 流程）
- 架构决策和设计原则

❌ **不适合放入记忆的内容：**
- API 密钥、密码、Token（安全风险）
- 个人信息（PII）
- 临时调试代码
- 单次任务的上下文（用 Checkpoints 代替）
- 完整文档（用 `@import` 引用代替）

### 组织原则

- 项目记忆提交 git，团队共享
- 个人偏好放用户记忆，不影响他人
- 路径特定规则用 rules/ 目录，而非在主 CLAUDE.md 里堆砌所有规则
- 保持文件精简（<300行），过长会影响加载效率

---

## 相关模块

- [01-slash-commands.md](01-slash-commands.md) — `/init` 和 `/memory` 命令
- [03-skills.md](03-skills.md) — 技能如何利用记忆上下文
- [07-plugins.md](07-plugins.md) — 企业级记忆策略分发
- [11-integration-patterns.md](11-integration-patterns.md) — Memory + Skills 集成模式
