---
title: Default 与 AcceptEdits 模式
tags:
  - 权限系统
  - Default模式
  - AcceptEdits模式
  - 交互设计
  - Claude-Code
domain: B-安全与信任
description: 两种基础权限模式——Default 每次都问的交互设计与 AcceptEdits 工作目录内自动编辑的信任机制
date: 2026-03-31
---

> [!abstract]
> Default 和 AcceptEdits 是最常用的两种权限模式。Default 是"安全基线"——每次都问用户；AcceptEdits 是"日常开发快捷模式"——工作目录内的文件编辑自动放行。这篇笔记分析它们的交互设计、规则匹配机制、以及被 Auto 模式复用的精巧之处。

## 一、Default 模式：每次都问

Default 模式是最直觉的权限策略：AI 想做任何需要权限的事，都弹出一个对话框问用户。

### 工作流程

```
AI 调用工具
  → tool.checkPermissions() 返回 'ask'
  → 检查是否有匹配的 allow/deny 规则
    → 有 allow → 直接放行
    → 有 deny → 直接拒绝
    → 无匹配 → 弹出权限对话框
```

### 权限对话框路由

`PermissionRequest.tsx` 是一个路由组件，根据工具类型分发到不同的专属对话框：

| 工具类型 | 对话框组件 | 展示什么 |
|---------|-----------|---------|
| Bash / Shell | `BashPermissionRequest` | 完整命令、命令说明 |
| FileEdit | `FileEditPermissionRequest` | 文件路径、修改 diff |
| FileWrite | `FileWritePermissionRequest` | 文件路径、写入内容 |
| Glob / Grep / Read | `FilesystemPermissionRequest` | 搜索路径/模式 |
| WebFetch | `WebFetchPermissionRequest` | 目标 URL |
| NotebookEdit | `NotebookEditPermissionRequest` | notebook 路径、单元格内容 |
| EnterPlanMode | `EnterPlanModePermissionRequest` | Plan 模式说明 |
| ExitPlanMode | `ExitPlanModePermissionRequest` | Plan 内容预览 |
| Skill | `SkillPermissionRequest` | Skill 名称和描述 |
| MCP 工具 | 动态对话框 | 服务器名、工具名、参数 |

> [!tip] 设计启示
> **不同操作用不同的审批界面**是好的 UX 设计。用户审批 `rm -rf` 和审批 `npm test` 需要看到完全不同的信息。通用的"是否允许？"对话框不够——用户需要足够的上下文来做出明智决定。

### 三级审批选项

每个权限对话框提供三种审批粒度：

| 选项 | 内部名称 | 效果 | 持久化 |
|------|---------|------|--------|
| "Yes" | `accept-once` | 仅允许这一次 | 无 |
| "Yes, allow all ... this session" | `accept-session` | 会话内同类操作自动放行 | 会话内存 |
| "No" | reject | 拒绝，可附带反馈 | 无 |

Accept Session 的关键行为：
- 根据操作类型生成规则（如 `Bash(npm *)` 或 `FileEdit(src/**)`）
- 规则存入 `session` destination（会话结束清除）
- 特殊作用域：`.claude` 文件夹有独立的 `claude-folder` 和 `global-claude-folder` 选项

### 用户反馈通道

当用户拒绝操作时，可以附带文字说明原因。这个反馈会被注入回对话，让 AI 理解为什么被拒绝并调整行为：

```typescript
// 拒绝消息格式
"The user doesn't want to proceed with this tool use. 
 The tool use was rejected. To tell you how to proceed, the user said:\n"
 + userFeedback
```

> [!tip] 设计启示
> 权限拒绝不只是一个布尔值——**附带理由的拒绝比单纯的"No"有价值得多**。它让 AI 能理解用户的意图并调整策略，而不是盲目重试或放弃。

## 二、AcceptEdits 模式：工作目录信任

AcceptEdits 是 Claude Code 推荐的日常开发模式。它的核心假设是：**在工作目录内编辑文件是安全的**——这正是用户请 AI 来做的事。

### 行为差异

| 操作 | Default | AcceptEdits |
|------|---------|-------------|
| 在 CWD 内编辑文件 | 询问 | **自动允许** |
| 在 CWD 内创建文件 | 询问 | **自动允许** |
| 在 CWD 外编辑文件 | 询问 | 询问 |
| 运行 Bash 命令 | 询问 | 询问 |
| 读取文件 | 允许 | 允许 |

关键：AcceptEdits **只放行文件编辑**，不放行 Bash 命令。这是一个精准的信任边界——编辑文件的破坏性有限（可以 git revert），但 Bash 命令可能做任何事。

### 实现方式

每个文件操作工具（FileEdit、FileWrite、NotebookEdit）在自己的 `checkPermissions()` 中检查模式：

```typescript
// 伪代码
async checkPermissions(input, context) {
  const mode = context.getAppState().toolPermissionContext.mode
  const targetPath = resolve(input.file_path)
  
  if (mode === 'acceptEdits' && isWithinWorkingDirectory(targetPath)) {
    return { behavior: 'allow' }
  }
  
  // 默认行为：ask
  return { behavior: 'ask', message: `Edit ${targetPath}?` }
}
```

### 被 Auto 模式复用

AcceptEdits 的逻辑被 Auto 模式的**快速路径 Level 1** 直接复用——Auto 模式假装自己是 AcceptEdits，调用工具的 `checkPermissions()`：

```typescript
// permissions.ts:607
const acceptEditsResult = await tool.checkPermissions(parsedInput, {
  ...context,
  getAppState: () => ({
    ...state,
    toolPermissionContext: { ...state.toolPermissionContext, mode: 'acceptEdits' },
  }),
})
```

这意味着 AcceptEdits 的信任边界被"嵌入"到了 Auto 模式中——在 Auto 模式下，工作目录内的文件编辑也会直接放行，无需分类器评估。

> [!tip] 设计启示
> **把一种模式的逻辑作为另一种模式的快速路径**是一个巧妙的复用。AcceptEdits 定义了"什么样的文件操作是安全的"，Auto 模式直接借用这个判断，而不是重新发明。这保证了行为一致性，也减少了代码重复。

## 三、规则匹配详解

规则系统是 Default 和 AcceptEdits 模式共用的"记忆机制"——用户审批过的操作，下次不再问。

### Bash 规则匹配

Bash 是规则匹配最复杂的工具，因为命令千变万化：

| 规则写法 | 含义 | 匹配示例 |
|---------|------|---------|
| `Bash(npm install)` | 精确匹配 | `npm install`（只匹配这一条） |
| `Bash(npm *)` | 通配符 | `npm install`、`npm run build`、`npm test` |
| `Bash(npm:*)` | 前缀匹配（旧语法） | 同上 |
| `Bash(python *)` | 通配符 | `python script.py`、`python -m pytest` |
| `Bash` | 整个工具 | 所有 Bash 命令（**危险！**） |

### 文件规则匹配

文件工具支持路径模式：

| 规则写法 | 含义 |
|---------|------|
| `FileEdit(src/**)` | src 目录下所有文件的编辑 |
| `FileWrite(*.log)` | 所有 .log 文件的写入 |
| `Read(/Users/name/project/**)` | 该项目目录的读取 |

### MCP 工具规则

MCP（Model Context Protocol）工具使用命名空间匹配：

| 规则写法 | 含义 |
|---------|------|
| `mcp__github` | github 服务器的所有工具 |
| `mcp__github__create_issue` | github 服务器的 create_issue 工具 |
| `mcp__github__*` | 同上（通配符写法） |

## 四、交互细节

### Shift+Tab 快捷键

在权限对话框中，用户可以用 `Shift+Tab` 快速切换到"Accept Session"选项，加速审批流程。这对频繁出现的低风险操作特别有用。

### `.claude` 文件夹特殊处理

当操作涉及 `.claude/` 目录时，审批选项会出现额外的作用域选择：

- **claude-folder**：允许编辑当前项目的 `.claude/` 目录
- **global-claude-folder**：允许编辑用户全局的 `~/.claude/` 目录

这些操作通常是 Claude Code 自身的配置更新（如 `.claude/settings.json`），需要比普通文件编辑更谨慎的审批。

## 五、设计模式总结

| 模式 | 怎么做 | 为什么 |
|------|--------|--------|
| 工具专属对话框 | 不同工具不同 UI | 用户需要不同的上下文做决策 |
| 三级审批粒度 | once / session / reject | 平衡安全性和便捷性 |
| 附带理由的拒绝 | 用户反馈注入回对话 | AI 理解意图而非盲目重试 |
| 工作目录信任 | CWD 内编辑自动放行 | 编辑文件是 AI 的本职工作 |
| 模式复用为快速路径 | Auto 借用 AcceptEdits 逻辑 | 行为一致 + 避免重复 |
| Shift+Tab 快捷键 | 快速跳到 session 允许 | 减少高频操作的审批负担 |

---

> **所属域**：[[安全与信任]]
> **相关笔记**：[[权限判定的完整流水线]]、[[Auto 模式与 AI 分类器]]、[[BypassPermissions 与 DontAsk 模式]]、[[权限与安全模型]]
