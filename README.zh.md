# record-what-happened

[English README](./README.md)

> 把 200 行 Agent 噪声变成 50 行可追查的任务日志。

## 问题

AI 编码 Agent（Claude Code、opencode、Cursor 等）很强，但它们的输出**高噪声、低信噪比**。一个任务跑完，你面对的是一大堆工具调用、零散输出和碎片化推理——事后几乎没法快速回顾。

更糟的是：下次你再处理同一个项目时，Agent 对上次试过什么、什么失败了、学到了什么，**毫无记忆**。

## 解决方案

**record-what-happened** 是一个 [Agent Skill](https://agentskills.io)，指导你的 AI 编码 Agent 在每次任务结束后生成一份**结构化、低噪声的任务日志**——记录它做了什么、出了什么问题、怎么修的、沿途发现了什么。

```
task-logs/
├── TASK-0001-setup-database.md
├── TASK-0002-fix-login-bug.md
└── TASK-0003-add-export-feature.md
```

每份日志遵循统一的 6 章节结构：

1. **任务上下文** — 原始指令、子任务拆解、完成标准
2. **执行轨迹** — 步骤概要 + 关键决策展开（不是每个工具调用都记）
3. **遇到的问题与解决** — 什么坏了、根因是什么、试过哪些方案（包括失败的）、最终怎么修的、遗留 TODO
4. **信息发现** — 执行中发现的项目事实，与预期不符之处
5. **验证与结果** — 怎么确认任务完成的、改了哪些文件
6. **回顾笔记**（可选）— 做得好的、可改进的

核心洞见：**失败尝试比成功尝试更有价值。** 日志记录 Agent 试过但*没用*的方案——这样下次就不会重复走死路。

## 安装

### 方式 A：项目级（推荐）

将 `skill/` 目录复制到项目的 skill 文件夹下：

```
your-project/
└── .opencode/
    └── skills/
        └── task-log/
            ├── SKILL.md
            ├── assets/
            │   └── TEMPLATE.md
            └── references/
                └── RECORDING_GUIDE.md
```

### 方式 B：全局（所有项目共享）

将 `skill/` 复制到全局 skills 目录：

- **opencode**：`~/.config/opencode/skills/task-log/`
- **Claude Code**：`~/.claude/skills/task-log/`

> `SKILL.md` 中的 `name` 字段为 `task-log`，目录名必须与此一致。

## 使用方式

### 手动触发

任务完成后，告诉 Agent：

```
记录一下刚才的任务过程
```

Agent 会在 `task-logs/` 下生成结构化日志，并告知文件路径。

### 自动触发（post-task hook）

在 opencode 中，可配置此 skill 在每次任务后自动执行。参见 [opencode 文档](https://opencode.ai)。

## 工作原理

本 skill 采用**渐进式加载**，最大限度减少上下文占用：

```
skill/
├── SKILL.md              # 激活时加载（约 165 行）
│                         # 包含：触发条件、6 步工作流、5 条核心原则
├── assets/
│   └── TEMPLATE.md       # 生成日志时加载（约 148 行）
│                         # 6 章节骨架 + YAML front-matter
└── references/
    └── RECORDING_GUIDE.md  # 仅按需加载（约 170 行）
                            # 逐章节填写指南 + 好坏示例对照
```

Agent 不会一次性加载全部文件——先读 `SKILL.md`，写日志时拉取模板，只在某章节不确定如何填写时才查阅参考指南。

### 核心原则

| 原则 | 含义 |
|------|------|
| **安全优先** | 密钥、Token、个人隐私信息在记录前脱敏——即使在报错原文中也不例外 |
| **事实与判断分离** | 原始输出（脱敏后）与分析推理分开记录 |
| **失败比成功重要** | 失败方案连同原因一起记录——这才是可复用的知识 |
| **信噪比优先** | 顺利步骤一行带过，只有决策和异常才展开 |
| **可检索性** | YAML front-matter 含任务编号、标签、关键文件，便于日后检索 |

## 兼容性

任何支持文件读写并遵循 [Agent Skills 规范](https://agentskills.io/specification) 的 Agent 环境：

- opencode
- Claude Code
- 其他支持 skill/plugin 的类似工具

## 示例日志

```yaml
---
task_id: "TASK-0002"
title: "Fix login token expiration"
date: "2026-08-26"
status: "completed"
tags: [bugfix, config]
key_files: [src/auth/login.ts, src/config/index.ts]
---
```

```markdown
## 3. 遇到的问题与解决

### 问题 1 — 登录后 Token 立即过期
- **根因**：login.ts 中过期时间硬编码为 0
- **尝试 1**：修改配置文件 → 失败（运行时被覆盖）
- **尝试 2**：在 login.ts:42 从环境变量读取 → 成功
- **遗留**：配置覆盖机制需进一步排查
```

## License

MIT
