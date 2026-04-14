# 模块 9：Advanced Features（高阶功能）完全指南

> Level 3 | 预计学习时间：4小时 | Claude Code v2.1.101+

---

## 功能总览

| 功能 | 核心价值 | 难度 |
|------|---------|------|
| Planning Mode | 先规划后执行，避免大改错误 | ⭐⭐ |
| Extended Thinking | 深度推理复杂问题 | ⭐ |
| Auto Mode | 高度自主执行 | ⭐⭐⭐ |
| 权限模式 | 精细化控制执行权限 | ⭐⭐ |
| Background Tasks | 不阻塞主对话的长任务 | ⭐⭐ |
| Monitor Tool | 事件驱动流处理 | ⭐⭐⭐ |
| Git Worktrees | 并行分支工作 | ⭐⭐ |
| Scheduled Tasks | 定时/循环任务 | ⭐⭐ |
| Session Management | 会话恢复与分支 | ⭐ |
| Voice Dictation | 语音输入 | ⭐ |
| Remote Control | 跨设备访问 | ⭐ |
| Chrome Integration | 浏览器自动化 | ⭐⭐⭐ |
| Ultraplan | 云端并发规划 | ⭐ |
| Task Lists | 跨压缩的任务追踪 | ⭐ |

---

## 1. Planning Mode（规划模式）

### 工作原理

```
用户：/plan 重构认证模块
           ↓
  进入 Plan Mode（只读）
           ↓
  Plan 子智能体研究代码库
  - 读取现有认证代码
  - 识别依赖关系
  - 分析影响范围
           ↓
  生成详细实施计划（带步骤、文件列表、风险点）
           ↓
  等待用户审批
           ↓
  用户批准 → 退出 Plan Mode → 开始执行
  用户修改 → 调整计划 → 再次审批
  用户拒绝 → 结束
```

### 使用方式

```bash
# 方式1：斜杠命令
/plan 重构用户认证模块，迁移到 JWT + Redis Session

# 方式2：自然语言
"先制定计划，我确认后再执行"
"帮我规划一下这个功能的实现方案，先不要动代码"

# 方式3：CLI 参数
claude --permission-mode plan
```

### 何时用 Planning Mode

✅ 适合场景：
- 影响多个文件的大型重构
- 架构级别的变更
- 不确定实现路径的复杂功能
- 有风险的操作（数据库 schema 变更）

❌ 不适合场景：
- 修复单个简单 bug
- 添加小功能（单个文件）
- 格式化/重命名等机械操作

### 规划文件写法

```markdown
<!-- .claude/plans/auth-refactor.md -->

## Context
为什么要重构：当前 session-based 认证不支持水平扩展，
需要迁移到 JWT + Redis，支持微服务架构。

## 推荐方案

### 阶段1：基础设施（2天）
- 安装 jsonwebtoken、redis 依赖
- 创建 JWT 工具类（src/utils/jwt.ts）
- 配置 Redis 连接（src/config/redis.ts）

### 阶段2：认证逻辑（3天）
- 修改 auth.service.ts：login 返回 JWT
- 修改 auth.middleware.ts：验证 JWT 而非 session
- 更新 refresh token 逻辑

### 阶段3：迁移（1天）
- 双模式支持（session 和 JWT 同时支持）
- 灰度切换

## 关键文件
- src/services/auth.service.ts（主要修改）
- src/middleware/auth.middleware.ts（主要修改）
- src/config/redis.ts（新建）
- src/utils/jwt.ts（新建）

## 风险点
- 现有登录状态会失效（需要用户重新登录）
- 需要 Redis 基础设施

## 验证
- 运行 npm test（现有测试）
- 手动测试登录/登出流程
- 测试 token 刷新
```

---

## 2. Extended Thinking（扩展思考）

```bash
# 设置高推理强度
/effort max
/effort high

# 在提示词中引导深度思考
"请深入分析这个架构设计的权衡，不要急于给出答案"
"从多个角度评估这个方案，包括性能、可扩展性、运维成本"
```

**适合场景：**
- 复杂系统架构决策（微服务 vs 单体）
- 性能瓶颈根因分析
- 算法选型（权衡多种方案）
- 安全漏洞的深层分析
- 数据建模的一致性设计

---

## 3. Auto Mode（实验性）

```bash
# 启用 Auto Mode
claude --permission-mode auto

# 在 settings.json 中设置
{ "permissionMode": "auto" }
```

Auto Mode 使用后台安全分类器审查操作：
- 安全操作：自动执行
- 可疑操作：要求确认
- 危险操作：拒绝并说明原因

> 使用建议：先在非关键项目上验证行为，再用于重要项目。

---

## 4. 权限模式完整指南

### 6种权限模式

```bash
# 1. default（默认）
# 危险操作前询问，适合日常使用
claude --permission-mode default

# 2. acceptEdits
# 自动接受文件编辑，减少"是否允许编辑"确认
# 适合：信任 Claude 编辑代码的场景
claude --permission-mode acceptEdits

# 3. plan
# 只读模式，只能规划不能执行
# 适合：让 Claude 分析问题而不修改任何文件
claude --permission-mode plan

# 4. dontAsk
# 大幅减少确认提示
# 适合：熟悉项目后的高效场景
claude --permission-mode dontAsk

# 5. auto
# 后台安全分类器自动决策
# 适合：高度自主的自动化场景
claude --permission-mode auto

# 6. bypassPermissions
# 完全绕过所有权限检查
# ⚠️ 警告：只在完全可控的环境中使用
claude --permission-mode bypassPermissions
```

### 工具级别权限控制

```bash
# 只允许特定工具
claude --allowedTools "Read,Grep,Glob,Bash(git:*)"

# 禁止特定工具
claude --disallowedTools "Write,Edit"

# 在 settings.json 中配置
{
  "allowedTools": ["Read", "Grep", "Glob"],
  "disallowedTools": ["Bash", "Write"]
}
```

---

## 5. Background Tasks（后台任务）

```bash
# 触发后台任务
# 在子智能体中配置 background: true
---
name: long-analyzer
background: true
---

# 或在 session 中手动放到后台
Ctrl+B              # 将当前任务放到后台

# 管理后台任务
Ctrl+F              # 终止所有后台智能体（×2确认）

# 完全禁用后台任务
export CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1
```

**后台智能体的权限限制：**
- 所有未预批准的权限请求自动拒绝
- 需要在智能体配置中预先指定 `allowedTools`

---

## 6. Monitor Tool（监控工具）

用于监控命令输出并对事件做出反应：

```bash
# 在 Hook 中使用 Monitor
# 每行 stdout 输出触发一次通知
# 适合：监控构建、测试、部署进度

# 示例：监控测试输出
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "npm test 2>&1 | tee /tmp/test-output.log"
      }]
    }]
  }
}
```

---

## 7. Git Worktrees（并行工作树）

### 手动使用

```bash
# 创建 worktree（不切换当前分支）
git worktree add ../feature-auth feature/user-auth
git worktree add ../bugfix-login fix/login-redirect

# 在 worktree 中工作
cd ../feature-auth
claude  # 在这个 worktree 中启动 Claude Code

# 查看所有 worktree
git worktree list

# 删除 worktree
git worktree remove ../feature-auth
```

### 子智能体使用（自动 worktree）

```yaml
---
name: feature-builder
isolation: worktree
description: 在独立分支实现功能，不影响主工作区
tools: Read, Write, Edit, Bash, Grep, Glob
---
```

- 子智能体在新 worktree（新分支）工作
- 无变更时自动清理
- 有变更时返回路径和分支名供审查

### /batch 技能使用多 worktree

```bash
/batch 将所有 REST API 端点从 Express 4 迁移到 Express 5 语法
# Claude 自动：
# 1. 分析所有需要修改的文件
# 2. 创建多个 worktree 并行处理
# 3. 汇总所有变更
```

---

## 8. Scheduled Tasks（定时任务）

### 本地定时（/loop）

```bash
/loop 5m 检查 CI 状态并汇报
/loop 1h 运行安全扫描并生成报告
/loop 30s 等待测试通过（最多等10分钟）

# 自动节奏（Claude 自行决定频率）
/loop 监控部署进度直到完成
```

### 云端定时（/schedule）

```bash
/schedule    # 打开定时任务管理界面
# 创建在云端执行的定时 Claude Code 任务
# 支持 cron 表达式
# 任务独立于本地终端运行
```

---

## 9. Session Management（会话管理）

```bash
# 恢复会话
claude -c                        # 继续上次会话
claude -r "session-name"         # 恢复命名会话

# 在会话中命名
# （通过提示词告知 Claude 记住会话名称）
"记住这个会话叫做 auth-refactor-2026"

# 导出对话
/export auth-refactor-session.md

# 创建分支
/branch before-big-change

# 从 transcript 恢复
# ~/.claude/projects/<hash>/<session-id>/conversation.jsonl
```

---

## 10. Voice Dictation（语音输入）

- 支持20种语言
- 用法：按住语音键说话，松开后自动转文字
- 适合：免提操作、长提示词输入

---

## 11. Remote Control（远程控制）

```bash
# 在 Claude Code 中开启远程访问
# Settings → Remote Control → 生成访问链接

# 从任意浏览器访问本地 Claude Code 会话
# 适合：手机/平板继续工作、跨设备协作
```

---

## 12. Chrome Integration（Chrome 集成）

连接 Chrome 后 Claude 可以：
```
- 截图当前页面并分析 UI 问题
- 点击元素、填写表单（测试自动化）
- 检查网络请求
- 读取控制台错误
- 运行 JavaScript
```

配置：在 Claude Code 设置中连接 Chrome DevTools Protocol。

---

## 13. Ultraplan

```bash
/ultraplan    # 在浏览器中生成详细规划，本地终端不被占用

# 适合：
# - 需要长时间规划的复杂任务
# - 不想占用终端的场景
# - 规划结果保存在浏览器中
```

---

## 14. Task Lists（跨压缩的任务追踪）

任务列表在上下文压缩时不会丢失：

```bash
# Claude 自动创建和维护任务列表
# 在长时间复杂任务中追踪进度

# 查看当前任务列表
TaskList

# 典型用途：
# 1. 长期重构任务的阶段追踪
# 2. 多文件修改的进度管理
# 3. Bug 修复中的子任务管理
```

---

## 高阶组合工作流

### 组合1：Planning + Subagents + Hooks（大型功能完整流程）

```
1. /plan 新功能实现计划（Planning Mode）
   → Plan 子智能体研究代码库
   → 生成分阶段实施计划

2. 批准计划后，用 Subagents 并行执行各阶段：
   - implementer agent：实现核心逻辑
   - test-engineer agent：同步编写测试
   - doc-writer agent：同步生成文档

3. Hooks 自动验证：
   - PostToolUse → lint + 类型检查
   - Stop → 验证测试是否通过

4. Checkpoints 保存每个里程碑
5. Git commit 固化成果
```

### 组合2：速率限制应对策略

```
问题：大型任务容易触发速率限制

解决方案：
1. 分阶段执行（不要一次性提交超大任务）
   /plan 先生成计划
   批准计划
   分多个 session 执行各阶段

2. 使用 Subagents 并行（比串行更快，单 session 消耗更少）
   让 3 个子智能体分别处理：
   - 后端 API 修改
   - 前端组件修改
   - 测试套件更新

3. 机械性任务用 Headless Mode（非交互）
   cat filelist.txt | claude -p "对每个文件做X处理" --output-format json

4. 用 Checkpoints 保存进度
   每个阶段完成后，检查点 + git commit
   下次 session 从 checkpoint 继续
```

---

## 相关模块

- [04-subagents.md](04-subagents.md) — Planning Mode 使用 Plan 子智能体
- [08-checkpoints.md](08-checkpoints.md) — 高阶工作流中的检查点策略
- [06-hooks.md](06-hooks.md) — 自动化验证与监控
- [10-cli.md](10-cli.md) — Headless Mode 用于 CI/CD 和批处理
- [11-integration-patterns.md](11-integration-patterns.md) — 多功能组合模式
