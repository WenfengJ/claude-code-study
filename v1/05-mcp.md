# 模块 5：MCP Protocol（模型上下文协议）

> Level 2 自动化 | 预计学习时间：1小时

---

## 核心概念

MCP（Model Context Protocol）让 Claude 访问外部工具、API 和实时数据源。

**核心能力：**
- 实时访问外部服务（数据库、GitHub、Slack 等）
- 双向数据同步
- 安全认证管理
- 可扩展架构

**三层架构：** Claude → MCP Server → External Service → 返回数据

---

## 传输协议

| 协议 | 说明 | 推荐程度 |
|------|------|---------|
| **HTTP Transport** | Web 连接，支持认证头 | ✅ 推荐 |
| **Stdio Transport** | 本地 Node.js 服务器 | 本地开发用 |
| **SSE Transport** | Server-Sent Events | ⚠️ 已废弃但仍支持 |

> Windows 用户在原生环境下调用 npx 命令时使用 `cmd /c`

---

## 配置范围

| 范围 | 位置 | 共享 | 适合场景 |
|------|------|------|---------|
| **本地** | `~/.claude.json` | 当前用户/项目 | 个人临时配置 |
| **项目** | `.mcp.json` | 团队共享 | 团队协作 |
| **用户** | `~/.claude.json` | 所有项目 | 个人固定配置 |

---

## CLI 命令

```bash
# 添加 MCP 服务器
claude mcp add --transport http my-server https://api.example.com
claude mcp add --transport stdio my-local-server node server.js

# 管理
claude mcp list                    # 列出所有服务器
claude mcp get <name>              # 查看服务器详情
claude mcp remove <name>           # 移除服务器
claude mcp reset-project-choices   # 重置项目选择
claude mcp add-from-claude-desktop # 从 Claude Desktop 导入

# 将 Claude 作为 MCP 服务器运行
claude mcp serve
```

---

## 认证方式

### OAuth 2.0（自动处理）
```bash
# Claude Code 自动处理整个 OAuth 流程
# - 浏览器交互式认证
# - Token 存储在系统密钥链
# - 自动缓存发现加速重连
```

### 环境变量凭据
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}",
        "API_KEY": "${API_KEY:-default-value}"
      }
    }
  }
}
```

支持 `${VAR}` 和 `${VAR:-default}` 语法。

---

## .mcp.json 配置示例

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/project"]
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
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://user:pass@localhost/mydb"]
    },
    "my-api": {
      "transport": "http",
      "url": "https://api.myservice.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```

---

## 常用 MCP 服务器

| 服务器 | 功能 |
|--------|------|
| **Filesystem** | 文件读写操作 |
| **GitHub** | 仓库管理、PR、Issues |
| **Slack** | 团队通讯 |
| **Database/PostgreSQL** | SQL 查询 |
| **Google Docs** | 文档访问 |
| **Asana** | 项目管理 |
| **Stripe** | 支付数据 |

---

## 在资源引用中使用 @

```bash
# 引用 MCP 资源
@server-name:protocol://resource/path

# 示例
@database:postgres://mydb/users
@github:github://repos/myorg/myrepo/contents/README.md
```

---

## 上下文膨胀问题的解决方案

当 MCP 工具描述超过上下文的10%时，Claude Code 自动启用高效工具选择（需要 Sonnet 4+ 或 Opus 4+）。

**Anthropic 推荐的代码执行模式（可减少98.7% token 使用）：**
```
不要直接调用工具（每次都消耗大量 token）
而是用代码执行模式：
1. 渐进式加载工具定义
2. 在执行环境中过滤和转换数据
3. 使用循环和条件语句，减少模型往返
4. 中间数据不进入上下文，保护隐私
```

---

## 子智能体中的 MCP

```yaml
---
name: data-analyst
mcpServers: postgres, bigquery
description: 数据分析专家，可访问生产数据库
---
```

在子智能体 frontmatter 中指定 `mcpServers`，限制该智能体只能访问特定服务器。

---

## 输出限制

| 参数 | 默认值 |
|------|--------|
| 警告阈值 | 10,000 tokens |
| 最大输出 | 25,000 tokens |
| 磁盘持久化 | 50,000+ 字符 |

---

## 企业级管理

管理员可通过 `managed-mcp.json` 控制组织的 MCP 访问策略：

```json
{
  "allowlist": ["github", "filesystem"],
  "blocklist": ["*external-api*"]
}
```

---

## 安全最佳实践

**要做的：**
- 用环境变量存储凭据，不要硬编码
- 每月轮换 Token
- 尽可能使用只读访问
- 监控使用日志

**不要做的：**
- 永远不要在代码中硬编码凭据
- 不要提交含有密钥的文件到 git
- 不要在聊天中分享 Token
- 不要授予不必要的权限

---

## 相关模块

- [04-subagents.md](04-subagents.md) — 在子智能体中限定 MCP 访问范围
- [02-memory.md](02-memory.md) — MCP 与记忆系统结合使用
- [07-plugins.md](07-plugins.md) — 将 MCP 配置打包到插件
