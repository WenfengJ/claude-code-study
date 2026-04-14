# 模块 9：Advanced Features（高阶功能）

> Level 3 高阶 | 预计学习时间：2-3小时

---

## 1. Planning Mode（计划模式）

### 核心概念
Planning Mode 让 Claude 先生成详细实施计划供你审查，确认后再执行。避免对大型任务直接修改代码。

### 进入方式
```bash
/plan [任务描述]           # 斜杠命令进入
# 或者在提示词中说：
"先制定计划，我确认后再执行"
```

### 工作流程
```
1. Claude 进入 Plan Mode（只读，不修改文件）
2. Plan 子智能体研究代码库结构
3. 生成带步骤的实施计划
4. 你审查并决定是否批准
5. 批准后 Claude 开始实际执行
```

### 在计划文件中定义
```markdown
<!-- .claude/plans/my-plan.md -->
## Context
为什么要做这件事，问题背景

## 方案
推荐实施路径

## 关键文件
- src/auth/middleware.ts（需要修改）
- src/models/user.ts（需要修改）

## 验证
如何测试变更是否正确
```

---

## 2. Extended Thinking（扩展思考）

### 核心概念
让 Claude 在回答前进行更深度的推理，适合复杂架构决策、算法分析等场景。

### 使用方式
```bash
/effort max              # 设置最高推理强度
# 或在提示词中说：
"请深入思考这个架构问题，不要急于给出答案"
"分析所有可能的实现方案，权衡利弊"
```

### 适合场景
- 复杂系统架构设计
- 算法选型与性能分析
- 安全漏洞的根因分析
- 复杂 bug 的根本原因排查

---

## 3. Auto Mode（自动模式，Research Preview）

### 核心概念
Auto Mode 使用后台安全分类器在执行前审查操作，实现更高度的自主执行。

### 启用方式
```bash
# 在 settings.json 中配置
{
  "permissionMode": "auto"
}

# 或 CLI 参数
claude --permission-mode auto
```

> **注意**：Auto Mode 是研究预览版，在非关键性工作上测试后再正式使用。

---

## 4. 权限模式（Permission Modes）

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `default` | 危险操作前询问 | 日常使用 |
| `acceptEdits` | 自动接受文件编辑 | 信任代码编辑场景 |
| `plan` | 只读，生成计划不执行 | 复杂任务规划阶段 |
| `dontAsk` | 减少确认提示 | 熟悉项目时提效 |
| `auto` | 后台安全分类器审查 | 高度自主执行 |
| `bypassPermissions` | 完全绕过权限 | ⚠️ 非必要不用 |

```bash
# CLI 设置
claude --permission-mode acceptEdits

# settings.json
{
  "permissionMode": "dontAsk"
}
```

---

## 5. Background Tasks（后台任务）

### 核心概念
后台任务让长时间运行的操作不阻塞主对话。

### 启动方式
```bash
# 在子智能体配置中设置
---
name: long-runner
background: true
---

# 或在会话中按快捷键
Ctrl+B    # 将当前运行的任务放到后台
Ctrl+F    # 终止所有后台智能体（按两次确认）
```

---

## 6. Monitor Tool（监控工具）

### 核心概念
Watch 命令输出并对匹配事件作出反应，适合监控构建、测试、部署过程。

```bash
# 示例：监控测试输出
# 在 Hook 或 Skill 中使用 Monitor 工具
# 每行 stdout 触发一次通知
```

---

## 7. Git Worktrees（并行工作树）

### 核心概念
Worktrees 让你在不同分支上并行工作，无需 stash 切换。

### 使用方式
```bash
# 创建 worktree
git worktree add ../feature-branch feature/new-auth

# 子智能体使用 worktree 隔离
---
name: feature-builder
isolation: worktree
---
```

### 与 Checkpoints 的区别
- Checkpoints：会话级别的回滚，同一分支
- Worktrees：文件系统级别，并行多分支

---

## 8. Scheduled Tasks（定时任务）

### 本地定时（/loop）
```bash
/loop 5m 检查部署状态并汇报    # 每5分钟执行一次
/loop 1h 审查代码变更           # 每小时执行一次
```

### 云端定时（/schedule）
```bash
/schedule                        # 打开定时任务管理界面
# 创建在云端执行的定时 Claude Code 任务
```

---

## 9. Session Management（会话管理）

### 会话恢复
```bash
claude -c                        # 继续上次对话
claude -r "session-name"         # 恢复特定名称的会话
```

### 会话导出
```bash
/export [filename]               # 导出当前对话到文件
```

### 任务列表（跨上下文压缩存活）
```
任务列表在上下文压缩时不会丢失，
适合在长时间工作中追踪进度。
```

---

## 10. Voice Dictation（语音输入）

- 支持20种语言
- 按住快捷键语音输入，松开后转为文字
- 适合免提操作场景

---

## 11. Remote Control（远程控制）

```bash
# 在任意浏览器继续本地 Claude Code 会话
# 设置 → Remote Control → 生成访问链接
```

适合：在手机/平板上继续工作，多设备协作。

---

## 12. Chrome Integration（Chrome 集成）

连接 Chrome 后 Claude 可以：
- 实时浏览器自动化
- 调试 Web 应用
- 截图并分析 UI
- 测试前端功能

---

## 13. Desktop App 特性

- 可视化 diff 审查
- 并行会话管理
- 原生系统集成

---

## 14. Ultraplan

```bash
/ultraplan    # 在浏览器中创建详细规划，本地终端保持响应
```

适合：需要长时间规划且不想占用终端的复杂任务。

---

## 高阶工作流推荐

### 复杂任务的标准流程

```
1. /plan [任务]          # 进入规划模式，生成计划
2. 审查计划 → 批准        # 确认方向正确
3. 执行阶段用 Subagents   # 并行处理，避免上下文耗尽
4. Hooks 自动验证结果      # PostToolUse 后自动运行测试/lint
5. Checkpoints 保存里程碑 # 随时可回滚
6. 完成后 git commit      # 固化变更
```

### 速率限制应对策略

- 将大型任务分拆为多个独立 session
- 用 Subagents 并行处理（减少单 session 消耗）
- 机械性任务用 Headless/Print Mode（非交互）
- 关键阶段之间用 Checkpoints 保存进度

---

## 相关模块

- [04-subagents.md](04-subagents.md) — Planning Mode 使用 Plan 子智能体
- [08-checkpoints.md](08-checkpoints.md) — 高阶工作流中的检查点策略
- [06-hooks.md](06-hooks.md) — 自动化验证与监控
- [10-cli.md](10-cli.md) — Headless/Print Mode 用于 CI/CD
