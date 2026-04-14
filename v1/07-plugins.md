# 模块 7：Plugins（插件）

> Level 3 高阶 | 预计学习时间：2小时

---

## 核心概念

Plugins 是多个 Claude Code 功能的打包集合，一条命令安装即可使用。

**可打包的内容：**
- Slash Commands（斜杠命令）
- Subagents（子智能体）
- Skills（技能）
- MCP Servers（外部服务连接）
- Hooks（事件钩子）
- LSP Servers（代码智能）
- 默认配置

---

## 与独立功能的对比

| 功能 | 安装方式 | 分发 | 最适合 |
|------|---------|------|--------|
| 独立 Slash Command | 手动复制文件 | 仓库 | 个人/项目专用 |
| 独立 Skill | 手动复制文件 | 仓库 | 个人/项目专用 |
| 独立 Subagent | 手动配置 | 仓库 | 个人/项目专用 |
| **Plugin** | **一条命令** | **Marketplace** | **团队/企业分发** |

---

## 插件目录结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # 插件清单（必须）
├── commands/                # 斜杠命令（Markdown 文件）
│   ├── task-1.md
│   └── workflows/
├── agents/                  # 子智能体定义
│   └── specialist-1.md
├── skills/                  # 技能（包含 SKILL.md）
│   └── my-skill/
│       └── SKILL.md
├── hooks/
│   └── hooks.json           # 事件钩子配置
├── .mcp.json                # MCP 服务器配置
├── .lsp.json                # LSP 服务器配置
├── bin/                     # 可执行文件（加入 Bash PATH）
├── settings.json            # 插件默认设置（仅支持 agent 键）
├── templates/               # 模板文件
├── scripts/                 # 辅助脚本
└── docs/
    └── README.md
```

---

## plugin.json 清单格式

```json
{
  "name": "my-plugin",
  "description": "插件功能说明",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  },
  "homepage": "https://example.com",
  "repository": "https://github.com/user/repo",
  "license": "MIT",
  "userConfig": {
    "apiKey": {
      "description": "服务的 API Key",
      "sensitive": true         // 存储在系统密钥链，不写入明文文件
    },
    "region": {
      "description": "部署区域",
      "default": "us-east-1"
    }
  }
}
```

---

## 安装与管理

```bash
# 从 Marketplace 安装
/plugin install plugin-name
claude plugin install plugin-name@marketplace-name

# 本地开发测试
claude --plugin-dir ./my-plugin
claude --plugin-dir ./plugin-a --plugin-dir ./plugin-b

# 从 GitHub 安装
/plugin install github:username/repo

# 管理
/plugin list                   # 列出所有插件
/plugin enable plugin-name     # 启用
/plugin disable plugin-name    # 禁用
/plugin uninstall plugin-name  # 卸载
/plugin update plugin-name     # 更新
/reload-plugins                # 热重载（开发时用）
claude plugin validate         # 验证插件结构
```

---

## 插件来源类型

| 来源 | 语法 | 示例 |
|------|------|------|
| 相对路径 | 字符串路径 | `"./plugins/my-plugin"` |
| GitHub | `{ "source": "github", "repo": "owner/repo" }` | 支持 `ref` 和 `sha` 版本锁定 |
| Git URL | `{ "source": "url", "url": "..." }` | 任意 Git 仓库 |
| Git 子目录 | `{ "source": "git-subdir", ... }` | Monorepo 中的某个包 |
| npm | `{ "source": "npm", "package": "..." }` | npm 包 |
| pip | `{ "source": "pip", "package": "..." }` | Python 包 |

---

## LSP 服务器配置（代码智能）

```json
// .lsp.json
{
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript"
    }
  },
  "python": {
    "command": "pyright-langserver",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".py": "python"
    }
  },
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

LSP 提供：即时诊断（错误/警告）、代码导航（跳转定义）、悬停信息、符号浏览。

---

## 持久数据目录

```json
// hooks/hooks.json
{
  "PostToolUse": [
    {
      "command": "node ${CLAUDE_PLUGIN_DATA}/track-usage.js"
    }
  ]
}
```

`${CLAUDE_PLUGIN_DATA}` 是每个插件独有的持久目录，安装后创建，卸载后删除。

---

## 内联插件（settings 中定义，v2.1.80+）

```json
// settings.json
{
  "pluginMarketplaces": [
    {
      "name": "inline-tools",
      "source": "settings",
      "plugins": [
        {
          "name": "quick-lint",
          "source": "./local-plugins/quick-lint"
        }
      ]
    }
  ]
}
```

---

## 实战示例：PR Review 插件

**plugin.json：**
```json
{
  "name": "pr-review",
  "version": "1.0.0",
  "description": "完整的 PR 审查流程，含安全、测试和文档检查"
}
```

**安装后自动获得：**
```
✅ 3 个斜杠命令（/review-pr, /check-security, /check-tests）
✅ 3 个专用子智能体（security-reviewer, test-checker, perf-analyzer）
✅ 2 个 MCP 服务器（GitHub, CodeQL）
✅ 4 个 Hook（pre/post deploy, security scan）
```

**使用流程：**
```
1. 用户：/review-pr
2. pre-review.js Hook 验证 git 仓库
3. GitHub MCP 获取 PR 数据
4. security-reviewer 子智能体分析安全性
5. test-checker 子智能体验证覆盖率
6. 综合输出审查报告
```

---

## 何时创建插件

| 场景 | 建议 | 原因 |
|------|------|------|
| 团队入职自动化 | ✅ 用插件 | 一键设置所有配置 |
| 框架集成 | ✅ 用插件 | 打包框架专用命令 |
| 企业标准化 | ✅ 用插件 | 统一分发，版本控制 |
| 快速单任务 | ❌ 用 Skill | 插件太重 |
| 单领域专长 | ❌ 用 Skill | Skill 更轻量 |
| 实时数据访问 | ❌ 用 MCP | 单独配置即可 |

---

## 企业级管理

```json
// 管理员 managed settings
{
  "enabledPlugins": ["code-standards", "security-scan"],
  "deniedPlugins": ["third-party-risky-plugin"],
  "extraKnownMarketplaces": ["https://plugins.mycompany.com"],
  "strictKnownMarketplaces": ["my-org/*"],
  "allowedChannelPlugins": { "stable": ["core-tools"] }
}
```

---

## Plugin Marketplace

| 来源 | 说明 |
|------|------|
| **官方** | `anthropics/claude-plugins-official`（Anthropic 维护） |
| **社区** | 开源社区贡献 |
| **企业私有** | 组织内部私有 Marketplace |

---

## 安全限制

插件内的子智能体有以下限制（防止权限提升）：
- 不允许定义 `hooks`
- 不允许配置 `mcpServers`
- 不允许覆盖 `permissionMode`

---

## 相关模块

- [03-skills.md](03-skills.md) — 技能作为插件组件
- [04-subagents.md](04-subagents.md) — 子智能体作为插件组件
- [05-mcp.md](05-mcp.md) — MCP 服务器打包进插件
- [06-hooks.md](06-hooks.md) — 事件 Hook 打包进插件
