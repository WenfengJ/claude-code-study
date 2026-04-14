# 模块 7：Plugins（插件）完全指南

> Level 3 | 预计学习时间：2小时 | Claude Code v2.1.101+

---

## 核心概念

Plugin 是多个 Claude Code 功能的打包集合，一条命令安装，自动配置所有组件。

**可打包的组件：**
- Slash Commands（命令）
- Subagents（子智能体）
- Skills（技能）
- MCP Servers（外部服务）
- Hooks（事件钩子）
- LSP Servers（代码智能）
- 默认设置

---

## 插件目录结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # 清单文件（必须）
├── commands/                # 斜杠命令（Markdown）
│   ├── review.md
│   └── deploy.md
├── agents/                  # 子智能体定义
│   ├── reviewer.md
│   └── tester.md
├── skills/                  # 技能（含 SKILL.md）
│   ├── security-scan/
│   │   └── SKILL.md
│   └── code-style/
│       └── SKILL.md
├── hooks/
│   └── hooks.json           # 事件钩子
├── .mcp.json                # MCP 服务器配置
├── .lsp.json                # LSP 服务器配置
├── bin/                     # 可执行文件（加入 PATH）
│   └── my-tool
├── settings.json            # 插件默认设置（当前仅支持 agent 键）
├── templates/               # 模板文件
├── scripts/                 # 辅助脚本
└── docs/
    └── README.md
```

---

## plugin.json 完整格式

```json
{
  "name": "pr-review",
  "description": "完整的 PR 审查工作流：安全、测试、文档一体化检查",
  "version": "1.2.0",
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  },
  "homepage": "https://github.com/you/pr-review-plugin",
  "repository": "https://github.com/you/pr-review-plugin",
  "license": "MIT",

  "userConfig": {
    "githubToken": {
      "description": "GitHub Personal Access Token（用于 MCP 连接）",
      "sensitive": true
    },
    "slackWebhook": {
      "description": "Slack Webhook URL（可选，用于通知）",
      "sensitive": true
    },
    "defaultReviewDepth": {
      "description": "默认审查深度：quick / standard / thorough",
      "default": "standard"
    }
  }
}
```

**注意：** `sensitive: true` 的配置项存储在系统密钥链，不写入明文文件。

---

## LSP 服务器配置（代码智能）

LSP 提供：即时诊断、跳转定义、悬停文档、符号搜索。

```json
// .lsp.json
{
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript",
      ".jsx": "javascriptreact"
    },
    "initializationOptions": {
      "preferences": {
        "includeInlayParameterNameHints": "all"
      }
    }
  },

  "python": {
    "command": "pyright-langserver",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".py": "python",
      ".pyi": "python"
    }
  },

  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    },
    "transport": "stdio"
  },

  "rust": {
    "command": "rust-analyzer",
    "extensionToLanguage": {
      ".rs": "rust"
    },
    "restartOnCrash": true,
    "maxRestarts": 3
  },

  "java": {
    "command": "jdtls",
    "args": ["-data", "${workspaceFolder}/.jdtls"],
    "extensionToLanguage": {
      ".java": "java"
    },
    "startupTimeout": 30000
  }
}
```

**LSP 字段参考：**

| 字段 | 必须 | 说明 |
|------|------|------|
| `command` | ✅ | LSP 服务器可执行文件 |
| `extensionToLanguage` | ✅ | 文件扩展名 → 语言 ID 映射 |
| `args` | 否 | 命令行参数 |
| `transport` | 否 | `stdio`（默认）或 `socket` |
| `env` | 否 | 环境变量 |
| `initializationOptions` | 否 | LSP 初始化选项 |
| `settings` | 否 | 工作区配置 |
| `workspaceFolder` | 否 | 覆盖工作区路径 |
| `startupTimeout` | 否 | 启动超时（毫秒） |
| `shutdownTimeout` | 否 | 优雅关闭超时 |
| `restartOnCrash` | 否 | 崩溃后自动重启 |
| `maxRestarts` | 否 | 最大重启次数 |

---

## 安装与管理

```bash
# 从 Marketplace 安装
/plugin install pr-review
claude plugin install pr-review@official

# 从 GitHub 安装
/plugin install github:username/repo

# 本地开发测试（不安装，临时加载）
claude --plugin-dir ./my-plugin
claude --plugin-dir ./plugin-a --plugin-dir ./plugin-b

# 管理
/plugin list                        # 列出所有插件
/plugin enable <name>               # 启用
/plugin disable <name>              # 禁用
/plugin uninstall <name>            # 卸载
/plugin update <name>               # 更新
/reload-plugins                     # 热重载（开发时）
claude plugin validate              # 验证插件结构
```

---

## 插件来源类型

| 来源类型 | 语法示例 |
|---------|---------|
| 相对路径 | `"./plugins/my-plugin"` |
| GitHub | `{ "source": "github", "repo": "owner/repo", "ref": "v1.0" }` |
| Git URL | `{ "source": "url", "url": "https://git.internal/plugin.git" }` |
| Git 子目录 | `{ "source": "git-subdir", "url": "...", "path": "packages/plugin" }` |
| npm | `{ "source": "npm", "package": "@company/claude-plugin", "version": "^2.0" }` |
| pip | `{ "source": "pip", "package": "claude-data-plugin" }` |

---

## Marketplace 配置（marketplace.json）

```json
{
  "name": "my-team-plugins",
  "owner": "my-org",
  "plugins": [
    {
      "name": "code-standards",
      "source": "./plugins/code-standards",
      "description": "强制执行团队编码规范",
      "version": "1.2.0",
      "author": "platform-team"
    },
    {
      "name": "deploy-helper",
      "source": {
        "source": "github",
        "repo": "my-org/deploy-helper",
        "ref": "v2.0.0"
      },
      "description": "部署自动化工作流"
    }
  ]
}
```

---

## 从零创建完整插件

### 步骤1：初始化结构

```bash
mkdir my-plugin && cd my-plugin
mkdir -p .claude-plugin commands agents skills/security-scan hooks scripts

# 创建清单
cat > .claude-plugin/plugin.json << 'EOF'
{
  "name": "my-plugin",
  "description": "我的自定义工作流插件",
  "version": "1.0.0",
  "author": { "name": "Your Name" },
  "license": "MIT"
}
EOF
```

### 步骤2：添加命令

```bash
cat > commands/review.md << 'EOF'
---
name: Review Code
description: 启动完整代码审查流程
---

# 代码审查

执行安全、性能和质量三维审查。
EOF
```

### 步骤3：添加子智能体

```bash
cat > agents/security-reviewer.md << 'EOF'
---
name: security-reviewer
description: 安全漏洞扫描专家
tools: Read, Grep
---

你是安全专家，专注于发现漏洞...
EOF
```

### 步骤4：配置 MCP

```bash
cat > .mcp.json << 'EOF'
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
EOF
```

### 步骤5：添加 Hooks

```bash
cat > hooks/hooks.json << 'EOF'
{
  "PostToolUse": [
    {
      "matcher": "Write|Edit",
      "hooks": [
        {
          "type": "command",
          "command": "cd $CLAUDE_PROJECT_DIR && npm run lint 2>&1 | tail -5"
        }
      ]
    }
  ]
}
EOF
```

### 步骤6：本地测试

```bash
claude --plugin-dir ./my-plugin
# 验证命令是否出现：/review
# 验证智能体是否出现：/agents
```

### 步骤7：热重载开发

```bash
# 在另一个终端修改插件文件
# 然后在 Claude Code 中：
/reload-plugins
```

### 步骤8：发布

```bash
# 选项1：发布到 GitHub（团队用）
git init && git add . && git commit -m "Initial plugin release"
git remote add origin https://github.com/you/my-plugin.git
git push -u origin main
# 用户安装：/plugin install github:you/my-plugin

# 选项2：提交到官方 Marketplace
# https://claude.ai/settings/plugins/submit
# 或
# https://platform.claude.com/plugins/submit
```

---

## 持久数据目录（v2.1.78+）

```bash
# ${CLAUDE_PLUGIN_DATA} 是插件专属的持久目录
# 安装时自动创建，卸载时删除

# 在 hooks.json 中使用
{
  "PostToolUse": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "node ${CLAUDE_PLUGIN_DATA}/track-usage.js >> ${CLAUDE_PLUGIN_DATA}/usage.log"
        }
      ]
    }
  ]
}

# 适合存储：缓存、数据库、使用统计、配置文件
```

---

## 内联插件（settings.json 中定义，v2.1.80+）

```json
{
  "pluginMarketplaces": [
    {
      "name": "local-tools",
      "source": "settings",
      "plugins": [
        {
          "name": "quick-lint",
          "source": "./local-plugins/quick-lint"
        },
        {
          "name": "deploy-check",
          "source": "./local-plugins/deploy-check"
        }
      ]
    }
  ]
}
```

---

## 企业级管理

```json
// 管理员 managed settings
{
  "enabledPlugins": ["code-standards", "security-scan"],
  "deniedPlugins": ["untrusted-plugin"],
  "extraKnownMarketplaces": [
    {
      "name": "company-internal",
      "url": "https://plugins.mycompany.com/marketplace.json"
    }
  ],
  "strictKnownMarketplaces": ["my-org/*", "anthropics/*"],
  "allowedChannelPlugins": {
    "stable": ["core-tools"],
    "beta": ["core-tools", "experimental-tools"]
  }
}
```

**严格模式（strictKnownMarketplaces）：**
- 空数组 `[]`：完全锁定，不允许任何 Marketplace
- 指定 glob 模式：只允许匹配的 Marketplace

---

## 插件安全限制

插件内的子智能体不允许（防止权限提升）：
- `hooks` — 不能定义生命周期钩子
- `mcpServers` — 不能配置 MCP 服务器
- `permissionMode` — 不能覆盖权限模型

---

## 何时用插件 vs 独立功能

| 场景 | 推荐 | 原因 |
|------|------|------|
| 团队入职自动化 | ✅ Plugin | 一键设置全部配置 |
| 框架集成（React/Vue/Spring） | ✅ Plugin | 框架专用命令 + LSP + 钩子 |
| 企业规范强制执行 | ✅ Plugin | 统一分发，版本管理 |
| 个人单次任务 | ❌ Skill/Command | Plugin 太重 |
| 单领域知识 | ❌ Skill | 更轻量 |
| 实时数据访问 | ❌ MCP | 直接配置即可 |

---

## 相关模块

- [03-skills.md](03-skills.md) — 技能作为插件组件
- [04-subagents.md](04-subagents.md) — 子智能体作为插件组件
- [05-mcp.md](05-mcp.md) — MCP 配置打包进插件
- [06-hooks.md](06-hooks.md) — 事件 Hook 打包进插件
