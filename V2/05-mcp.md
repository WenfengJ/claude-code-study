# 模块 5：MCP Protocol（模型上下文协议）完全指南

> Level 2 | 预计学习时间：3小时 | Claude Code v2.1.101+

---

## 核心概念

MCP（Model Context Protocol）让 Claude 实时访问外部工具和数据源。

**工作模型：**
```
Claude Code → MCP Server → External Service
                ↑               ↓
           （请求工具）     （返回数据）
```

**关键特性：**
- 实时数据（不是训练时的静态知识）
- 双向交互（读写外部系统）
- 安全认证（OAuth、Token、密钥链存储）
- 动态更新（无需重连即可添加/删除工具）

---

## 3种传输协议

| 协议 | 连接方式 | 适用场景 | 推荐度 |
|------|---------|---------|--------|
| **HTTP** | Web 端点 + 可选认证头 | 远程服务、云 API | ✅ 推荐 |
| **Stdio** | 本地进程 stdin/stdout | 本地 Node.js 服务器 | 本地开发 |
| **SSE** | Server-Sent Events | 已废弃但仍支持 | ⚠️ 不推荐 |

---

## 配置范围与位置

| 范围 | 文件 | 共享方式 | 需要团队批准 |
|------|------|---------|------------|
| **本地** | `~/.claude.json` | 仅当前用户/项目 | 否 |
| **项目** | `.mcp.json` | 提交 git，团队共享 | ✅ 是 |
| **用户** | `~/.claude.json` | 个人所有项目 | 否 |

---

## .mcp.json 配置完整示例

### 基础结构

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-name"],
      "env": {
        "TOKEN": "${ENV_VAR}"
      }
    }
  }
}
```

### 10+ 常用服务器配置

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/project", "/tmp"]
    },

    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },

    "postgres": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-postgres",
        "postgresql://user:pass@localhost:5432/mydb"
      ]
    },

    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "./data.db"]
    },

    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${SLACK_TEAM_ID}"
      }
    },

    "google-drive": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-gdrive"],
      "env": {
        "GDRIVE_CREDENTIALS": "${GDRIVE_CREDENTIALS_JSON}"
      }
    },

    "redis": {
      "command": "npx",
      "args": ["-y", "mcp-server-redis"],
      "env": {
        "REDIS_URL": "${REDIS_URL:-redis://localhost:6379}"
      }
    },

    "jira": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-atlassian"],
      "env": {
        "ATLASSIAN_URL": "${JIRA_URL}",
        "ATLASSIAN_TOKEN": "${JIRA_TOKEN}"
      }
    },

    "my-http-api": {
      "transport": "http",
      "url": "https://api.myservice.com/mcp/v1",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}",
        "X-Client": "claude-code"
      }
    },

    "local-tools": {
      "command": "node",
      "args": ["./scripts/mcp-server.js"],
      "env": {
        "NODE_ENV": "development"
      }
    }
  }
}
```

---

## 认证方式详解

### OAuth 2.0（Claude 自动处理）

```bash
# 对支持 OAuth 的服务器（如 Notion、Stripe），Claude Code 自动：
# 1. 发现 OAuth 端点
# 2. 打开浏览器认证
# 3. 将 Token 存入系统密钥链
# 4. 缓存发现信息，加速后续连接
```

### 环境变量（推荐方式）

```json
{
  "env": {
    "API_KEY": "${API_KEY}",
    "DB_URL": "${DATABASE_URL:-postgresql://localhost/mydb}",
    "TIMEOUT": "${REQUEST_TIMEOUT:-30}"
  }
}
```

支持语法：
- `${VAR}` — 直接引用环境变量
- `${VAR:-default}` — 环境变量不存在时使用默认值

### 系统密钥链存储

对于敏感凭据，推荐用系统密钥链而非环境变量：
```bash
# macOS Keychain
security add-generic-password -a claude-code -s github-token -w "your-token"

# 在 .mcp.json 中引用
# Claude Code 自动从密钥链读取 OAuth Token
```

---

## CLI 命令完整参考

```bash
# 添加服务器
claude mcp add --transport http my-api https://api.example.com/mcp
claude mcp add --transport stdio local-server node ./server.js
claude mcp add --transport stdio npm-server npx @org/mcp-server

# 从 Claude Desktop 导入（共用配置）
claude mcp add-from-claude-desktop

# 查看服务器
claude mcp list                    # 列出所有已配置服务器
claude mcp get github              # 查看 github 服务器详情

# 管理
claude mcp remove github           # 移除服务器
claude mcp reset-project-choices   # 重置项目级选择

# 将 Claude 作为 MCP 服务器（供其他工具调用）
claude mcp serve
```

---

## 在子智能体中限定 MCP 访问

```yaml
---
name: db-analyst
mcpServers: postgres, bigquery
description: 数据分析专家，只能访问数据库服务器
tools: Read, Bash
---

你是数据分析专家，只能通过 PostgreSQL 和 BigQuery MCP 访问数据。
禁止直接读写文件系统。
```

---

## @ 资源引用语法

```bash
# 在提示词中直接引用 MCP 资源
@github:github://repos/myorg/myrepo/contents/README.md
@database:postgres://mydb/users
@slack:slack://channels/C1234567890

# 格式：@<server-name>:<protocol>://<resource-path>
```

---

## 上下文膨胀解决方案

当 MCP 工具数量过多（描述超上下文10%）时：

### 方案1：自动工具选择
Claude Code 对 Sonnet 4+ 和 Opus 4+ 自动启用高效工具选择，只在需要时加载工具描述。

### 方案2：代码执行模式（推荐，可减少98.7% token 使用）

```
不要让 Claude 逐个调用 MCP 工具（每次往返都消耗 token）
而是让 Claude 编写代码在执行环境中处理：

示例：不要
  → 调用 list_repos 工具
  → 对每个 repo 调用 get_contents 工具
  → ...（100次往返）

而是：
  → 编写一个脚本，用 GitHub API 批量获取所有需要的数据
  → 在执行环境中过滤/转换
  → 只把最终结果带回上下文
```

---

## 工具输出限制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 警告阈值 | 10,000 tokens | 超过时警告 |
| 最大输出 | 25,000 tokens | 超过时截断 |
| 磁盘持久化 | 50,000+ 字符 | 存储到磁盘不占上下文 |

---

## 企业级管理

### 白名单/黑名单控制

```json
// managed-settings 或组织级配置
{
  "allowedMcpServers": ["github", "filesystem", "internal-*"],
  "deniedMcpServers": ["*external*", "unknown-vendor"]
}
```

### 自定义 npm Registry（用于私有 MCP 服务器）

```json
{
  "mcpServers": {
    "company-tools": {
      "command": "npx",
      "args": ["--registry", "https://npm.mycompany.com", "@company/mcp-tools"]
    }
  }
}
```

---

## 构建自定义 MCP 服务器

### 简单示例（Node.js）

```javascript
// my-mcp-server.js
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "my-company-tools",
  version: "1.0.0"
}, {
  capabilities: { tools: {} }
});

server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "get_deployment_status",
    description: "获取指定服务的部署状态",
    inputSchema: {
      type: "object",
      properties: {
        service: { type: "string", description: "服务名称" }
      },
      required: ["service"]
    }
  }]
}));

server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "get_deployment_status") {
    const service = request.params.arguments.service;
    // 调用内部 API
    const status = await fetchDeploymentStatus(service);
    return {
      content: [{ type: "text", text: JSON.stringify(status) }]
    };
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

```json
// .mcp.json 中引用
{
  "mcpServers": {
    "company-tools": {
      "command": "node",
      "args": ["./scripts/my-mcp-server.js"]
    }
  }
}
```

---

## 安全最佳实践

### 凭据管理

```bash
# ✅ 正确：环境变量
export GITHUB_TOKEN=ghp_xxx

# ✅ 正确：系统密钥链（OAuth 自动）

# ❌ 错误：硬编码在 .mcp.json
"env": { "TOKEN": "ghp_hardcoded_xxx" }

# ❌ 错误：提交含 token 的配置
git add .mcp.json  # 先检查有没有真实 token！
```

### 最小权限原则

```json
// ✅ 只给必要权限
{
  "mcpServers": {
    "github": {
      // 只使用 read-only scope 的 token
      "env": { "GITHUB_TOKEN": "${GITHUB_READONLY_TOKEN}" }
    }
  }
}
```

---

## 调试 MCP 连接

```bash
# 查看连接状态
claude mcp list

# 测试连接
claude mcp get <server-name>

# Debug 模式查看详细日志
claude --debug

# 常见问题
# 1. 服务器启动失败：检查 command 和 args 是否正确
# 2. 认证失败：检查环境变量是否已设置
# 3. 工具不出现：重启 Claude Code 或检查服务器实现
```

---

## 相关模块

- [04-subagents.md](04-subagents.md) — 在子智能体中限定 MCP 范围
- [07-plugins.md](07-plugins.md) — 将 MCP 配置打包到插件
- [11-integration-patterns.md](11-integration-patterns.md) — Subagents + MCP 数据处理模式
- [13-troubleshooting.md](13-troubleshooting.md) — MCP 连接失败排查
