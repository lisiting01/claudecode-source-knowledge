---
title: BypassPermissions 与 DontAsk 模式
tags:
  - 权限系统
  - BypassPermissions
  - DontAsk
  - 企业管控
  - Claude-Code
domain: B-安全与信任
description: 权限光谱的两个极端——BypassPermissions 全放行与 DontAsk 全拒绝，以及企业级权限治理机制
date: 2026-03-31
---

> [!abstract]
> BypassPermissions 和 DontAsk 是权限光谱的两个极端：一个几乎全部放行，一个全部拒绝。它们的设计形成了优雅的对称——一个在判定管线的头部短路，一个在尾部兜底。这篇笔记分析它们的实现细节、使用场景、以及企业级权限治理。

## 一、BypassPermissions：全放行

### 核心行为

BypassPermissions 在判定管线的**最前端**短路——大多数操作直接返回 `allow`，不经过任何中间步骤。

```typescript
// permissions.ts（简化）
// Step 1g: bypassPermissions 模式提前返回
if (mode === 'bypassPermissions') {
  // 仅有 safetyCheck 且非 classifierApprovable 的检查不被绕过
  if (result.decisionReason?.type === 'safetyCheck'
      && !result.decisionReason.classifierApprovable) {
    return result  // 安全检查仍然生效
  }
  return { behavior: 'allow' }
}
```

### 不能绕过什么

即使在 BypassPermissions 模式下，有一类操作**仍然会被拦截**：标记为 `classifierApprovable: false` 的安全检查。这些是硬编码的底线，比如修改受保护的系统文件。

这意味着 BypassPermissions 不是真正的"全部放行"——它是"**跳过用户交互，但保留安全底线**"。

### 进入确认

用户切换到 BypassPermissions 模式时，必须看到一个**强制确认对话框**（`BypassPermissionsModeDialog.tsx`）：

```
⚠️ WARNING
Claude Code will not ask for your approval before running 
potentially dangerous commands.

[No, exit]  [Yes, I accept]
```

确认后，用户可以设置 `skipDangerousModePermissionPrompt` 跳过后续的确认。

### 企业级禁用

组织管理员可以通过远程门控**完全禁用**这个模式：

| 控制方式 | 配置 |
|---------|------|
| 远程门控 | `tengu_disable_bypass_permissions_mode` → Statsig |
| 本地设置 | `permissions.disableBypassPermissionsMode: 'disable'` → settings.json |
| 运行时检查 | `isBypassPermissionsModeAvailable` 字段在 `ToolPermissionContext` 中 |

当 bypass 被禁用时，用户即使在 CLI 中指定 `--permission-mode bypassPermissions` 也无法使用，系统会静默降级到 `default`。

> [!tip] 设计启示
> "可被企业禁用"是面向企业的 AI 产品的关键特性。个人用户可以选择"我信任 AI"，但企业不能让员工单方面做这个决定。**组织策略必须能覆盖个人偏好**。

## 二、DontAsk：全拒绝

### 核心行为

DontAsk 在判定管线的**最末端**兜底——把所有 `ask` 决策转换为 `deny`：

```typescript
// permissions.ts:505-517
// Apply dontAsk mode transformation: convert 'ask' to 'deny'
// This is done at the end so it can't be bypassed by early returns
if (result.behavior === 'ask') {
  if (appState.toolPermissionContext.mode === 'dontAsk') {
    return {
      behavior: 'deny',
      decisionReason: { type: 'mode', mode: 'dontAsk' },
      message: DONT_ASK_REJECT_MESSAGE(tool.name),
    }
  }
}
```

注意代码注释："在最后做，这样不能被提前返回绕过"。这是一个有意识的安全决策——确保 dontAsk 的"全拒绝"语义不会被任何中间逻辑破坏。

### 使用场景

DontAsk 不是给普通用户用的——它服务于**无人值守**的场景：

| 场景 | 为什么用 DontAsk |
|------|-----------------|
| **Headless 模式** | 没有终端，无法弹出对话框 |
| **SDK 集成** | 嵌入到其他应用中，不能打断宿主程序 |
| **后台 Agent** | 自动运行的代理，不应该等待人工输入 |
| **CI/CD 流水线** | 自动化管线中不能有交互式确认 |

### 与 shouldAvoidPermissionPrompts 的关系

DontAsk 模式和 `shouldAvoidPermissionPrompts` 标记有类似但不同的用途：

| 机制 | 作用域 | 效果 |
|------|--------|------|
| `dontAsk` 模式 | 全局模式切换 | 所有 ask → deny |
| `shouldAvoidPermissionPrompts` | 上下文标记 | 影响对话框是否弹出 |

两者可以叠加——在协调者模式下的异步子代理，既设了 `shouldAvoidPermissionPrompts`，也可能处于 `dontAsk` 模式。

### 拒绝消息

DontAsk 的拒绝消息是专门设计的，告诉 AI "这个工具当前不可用"：

```typescript
const DONT_ASK_REJECT_MESSAGE = (toolName: string) =>
  `The ${toolName} tool requires user approval, but don't-ask mode is active. ` +
  `The tool was rejected. Try a different approach.`
```

> [!tip] 设计启示
> DontAsk 模式的关键洞见是：**不是所有 AI Agent 都需要人工确认**。在自动化场景中，"宁可失败也不等待"比"等待人工确认"更合理。设计 AI Agent 产品时，需要考虑无人值守场景的权限策略。

## 三、对称设计

BypassPermissions 和 DontAsk 在管线中形成了优雅的对称：

```
判定管线：

  ┌────────────────────────────────────────────────────┐
  │                                                    │
  │  bypass → 头部短路（大多数直接 allow）              │
  │    ↓                                               │
  │  tool.checkPermissions() → allow/ask/deny          │
  │    ↓                                               │
  │  规则匹配 → allow/deny 规则覆盖                    │
  │    ↓                                               │
  │  模式分流 → auto/plan/default 各自处理             │
  │    ↓                                               │
  │  dontAsk → 尾部兜底（所有 ask 变 deny）            │
  │                                                    │
  └────────────────────────────────────────────────────┘
```

| 维度 | BypassPermissions | DontAsk |
|------|-------------------|---------|
| 管线位置 | 头部短路 | 尾部兜底 |
| 效果 | ask → allow | ask → deny |
| 进入门槛 | 需要确认对话框 | 静默生效 |
| 企业管控 | 可被禁用 | 不受限 |
| 安全底线 | 保留 safetyCheck | 无需（已是最安全） |
| 典型用户 | 信任环境中的高级用户 | 自动化系统 |

> [!tip] 设计启示
> 两个极端模式在管线中的不同位置体现了不同的安全思维：
> - BypassPermissions 在**头部**，因为它需要尽快短路以提升效率
> - DontAsk 在**尾部**，因为它需要确保没有任何"漏网之鱼"——即使某些中间逻辑返回了 ask，最后都会变成 deny
>
> 这是一个通用模式：**宽松策略应尽早生效（减少不必要的计算），严格策略应尽晚生效（确保不被绕过）**。

## 四、企业级权限治理

### policySettings 来源

企业管理员通过 `policySettings` 源施加控制，这是优先级最高的用户可控来源（仅次于远程 `flagSettings`）：

```
flagSettings (远程平台策略)
  ↓
policySettings (企业组织策略) ← 管理员在这里控制
  ↓
userSettings (个人偏好)
  ↓
projectSettings / localSettings / ...
```

### 关键企业控制

| 控制 | 设置 | 效果 |
|------|------|------|
| 禁用 bypass 模式 | `disableBypassPermissionsMode` | 用户无法进入 bypass |
| 只允许托管规则 | `allowManagedPermissionRulesOnly` | 隐藏"Always Allow"选项 |
| 禁用 auto 模式 | `disableAutoMode` | 用户无法使用 AI 分类器 |
| 工具限制 | allow/deny 规则 | 限制可用工具和命令 |

### allowManagedPermissionRulesOnly

这是企业管控中最重要的设置之一。当启用时：

- 权限对话框中**隐藏**"Always Allow"和持久化选项
- 只显示 "Accept Once"（session）和 "Reject"
- 用户不能自己创建永久规则
- 只有管理员预设的规则生效

这意味着企业可以精确控制"AI 被允许做什么"，员工不能自行扩大 AI 的权限范围。

> [!tip] 设计启示
> 对企业级 AI Agent 产品来说，**"谁能授予 AI 权限"本身就是一个权限问题**。普通员工能允许 AI 运行 `sudo` 吗？能允许 AI 访问生产数据库吗？`allowManagedPermissionRulesOnly` 解决的就是这个问题——把"授权 AI"的权力收归管理员。

## 五、设计模式总结

| 模式 | 怎么做 | 为什么 |
|------|--------|--------|
| 头部短路 vs 尾部兜底 | bypass 在前，dontAsk 在后 | 宽松策略尽早生效，严格策略确保不被绕过 |
| 保留安全底线 | bypass 不绕过 safetyCheck | 即使"信任一切"也有不可逾越的红线 |
| 强制确认进入 | bypass 需要明确同意 | 防止误切换到危险模式 |
| 企业可禁用 | bypass 可被远程门控关闭 | 组织策略覆盖个人偏好 |
| 授权权限分离 | allowManagedPermissionRulesOnly | 区分"使用 AI"和"授权 AI" |
| 友好的拒绝消息 | dontAsk 告知 AI 尝试其他方法 | 拒绝不是终点，是引导 AI 换策略 |

---

> **所属域**：[[安全与信任]]
> **相关笔记**：[[权限判定的完整流水线]]、[[Auto 模式与 AI 分类器]]、[[Default 与 AcceptEdits 模式]]、[[权限与安全模型]]
