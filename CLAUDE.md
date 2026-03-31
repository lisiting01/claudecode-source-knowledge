# Claude Code 源码知识库 — 工作指南

## 项目目的

这是一个 **Obsidian 知识库**，用于持续分析 Claude Code 源码，提炼对构建 AI Agent 类产品有价值的设计思想。

- 源码位置：`C:\docs\claude-code-sourcemap`（v2.1.88，已还原，共 1,902 个文件）
- 知识库位置：`C:\CodingProject\ObsidianRep\claudecode_source_knowledge\`

## 核心原则

- **关注设计，不是实现**：分析背后的设计逻辑、架构模式，而不是照搬代码
- **中文表达，通俗易懂**：技术术语需附简短说明，对非开发人员也要友好
- **产品视角优先**：每个发现都要关联"对构建 AI Agent 产品有什么启示"
- **持续积累**：每次学习都添加到笔记，不要推倒重来

## 知识域架构

知识库按 **5 个知识域** 组织，每个域代表 AI Agent 运行时的一个架构层。域索引使用 Dataview 动态查询，笔记只需写对 frontmatter 就能自动归位。

| 域 | 索引笔记 | 核心问题 |
|---|---------|---------|
| A 核心运行时 | `核心运行时.md` | 引擎怎么转 |
| B 安全与信任 | `安全与信任.md` | 什么不能做 |
| C 配置与提示词 | `配置与提示词.md` | 怎么调控行为 |
| D 协作与扩展 | `协作与扩展.md` | 怎么长出新能力 |
| E 交互与体验 | `交互与体验.md` | 用户怎么感知 |

### 贯穿笔记（跨域）

| 笔记 | 定位 |
|------|------|
| `Claude Code 架构总览.md` | 顶层 MOC，导航到各域 |
| `设计哲学与核心理念.md` | 所有域共享的设计原则基底 |
| `构建 AI Agent 的设计启示.md` | 从所有域萃取的产品设计经验 |

### 全局视图

`知识库全局视图.base` — Bases 数据库视图，支持按域分组、全部笔记、卡片三种视图。

## 后续学习方向（按域分组）

### A 域 — 核心运行时
- [ ] **推测执行（Speculation）**：源码中有 `speculation` 状态，AI 是否会预测用户下一步？
- [ ] **Token 估算**：在发出请求前如何估算 token 数？

### B 域 — 安全与信任
- [x] **权限模式深挖**：完整判定流水线 + 各模式独立分析 → [[权限判定的完整流水线]]、[[Auto 模式与 AI 分类器]]、[[Default 与 AcceptEdits 模式]]、[[BypassPermissions 与 DontAsk 模式]]

### D 域 — 协作与扩展
- [ ] **Bridge 系统**：CLI 与 VS Code/JetBrains IDE 扩展如何通信？
- [ ] **远程代理（CCR）**：企业版的云端代理如何工作？

### E 域 — 交互与体验
- [x] **Plan 模式的实现细节**：EnterPlanMode / ExitPlanMode 工具如何控制 AI 行为？→ [[Plan 模式的实现细节]]
- [ ] **LSP 集成**：如何把语言服务器的能力（跳转定义、查引用）提供给 AI？
- [ ] **Voice 特性**：语音输入输出的架构设计

## 新笔记归属规则

创建新笔记时，只需两步：

1. **写对 frontmatter**：
```yaml
---
title: 笔记标题
tags:
  - 主题标签
  - Claude-Code
domain: X-域名称          # A-核心运行时 / B-安全与信任 / C-配置与提示词 / D-协作与扩展 / E-交互与体验
parent: 父笔记标题        # 可选，子笔记才需要
description: 一句话描述    # 用于 Dataview 表格和 Bases 视图
date: YYYY-MM-DD
---
```

2. **笔记自动出现在域索引和全局视图中**——因为域索引用 Dataview 查询 `domain` 属性，Bases 视图也基于属性聚合。不需要手动更新任何索引。

### 跨域笔记

domain 字段用列表格式：
```yaml
domain:
  - C-配置与提示词
  - B-安全与信任
```

## 笔记写作规范

### 文件命名
- 纯语义命名：`笔记主题.md`（不加编号前缀）
- 靠 frontmatter 的 `domain` 属性分组，不靠文件名排序

### 内容结构
1. `> [!abstract]` callout 说明这篇解决什么核心问题
2. 用 `##` 分节，每节聚焦一个子主题
3. 复杂流程用 Mermaid 图或代码块表示
4. 每个重要观点附 `> [!tip] 设计启示` callout
5. 结尾用 `---` 分隔后跟所属域和相关笔记链接

### Callout 使用约定
- `[!abstract]` — 开篇，说明核心问题
- `[!tip]` — 设计启示，对构建产品的建议
- `[!important]` — 重要但容易忽视的点
- `[!warning]` — 陷阱或常见错误
- `[!example]` — 具体例子或类比
- `[!info]` — 背景知识解释

## 源码探索方法

**快速查找文件：**
```
Glob: C:\docs\claude-code-sourcemap\restored-src\src\**\*.ts
```

**关键目录：**
- `src/tools/` — 所有工具实现
- `src/services/` — 后端服务
- `src/utils/permissions/` — 权限系统
- `src/services/compact/` — 上下文压缩
- `src/tasks/` — 任务/代理系统
- `src/utils/hooks.ts` — Hook 系统
- `src/skills/` — Skill 系统
- `src/services/mcp/` — MCP 集成
- `src/constants/prompts.ts` — 系统提示词组装
- `src/utils/claudemd.ts` — CLAUDE.md 发现与加载
- `src/memdir/` — 记忆系统提示词
