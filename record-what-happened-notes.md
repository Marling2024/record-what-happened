# record-what-happened 项目笔记

> 把 200 行 Agent 噪声变成 50 行可追查的任务日志

---

## 1. 项目概览

| 属性       | 值 |
|------------|-----|
| **项目名称** | record-what-happened |
| **类型**     | Agent Skill（遵循 [Agent Skills 规范](https://agentskills.io/specification)） |
| **作者**     | 1Zero2four |
| **版本**     | 1.1 |
| **许可证**   | MIT |
| **语言**     | 中文（不影响 Agent 执行） |
| **兼容平台** | opencode、Claude Code 及其他支持 skill/plugin 的工具 |

---

## 2. 解决的问题

AI 编码 Agent（Claude Code、opencode、Cursor 等）的输出**高噪声、低信噪比**。一个任务跑完后：

- 遗留大量工具调用、零散输出、碎片化推理，事后无法快速回顾
- Agent 对上次试过什么、什么失败了、学到了什么**毫无记忆**，下次重复踩坑

---

## 3. 解决方案

在每次任务结束后，自动生成一份**结构化、低噪声的任务日志**，涵盖：

- 做了什么
- 出了什么问题
- 怎么修的（包括失败的尝试）
- 沿途发现了什么

### 3.1 日志文件结构

```
task-logs/
├── TASK-0001-setup-database.md
├── TASK-0002-fix-login-bug.md
└── TASK-0003-add-export-feature.md
```

### 3.2 日志 6 章节模板

| 章节 | 内容 |
|------|------|
| **1. 任务上下文** | 原始指令、子任务拆解、完成标准 |
| **2. 执行轨迹** | 步骤概要 + 关键决策展开（不逐条记录工具调用） |
| **3. 遇到的问题与解决** | 现象、根因、尝试方案（含失败）、最终修复、遗留 TODO |
| **4. 信息发现** | 与预期不符之处、项目事实、踩坑记录 |
| **5. 验证与结果** | 验证手段、产出物清单、完成标准核对 |
| **6. 回顾笔记**（可选） | 做得好的、可改进的、重来的最优路径 |

### 3.3 核心洞见

> **失败尝试比成功尝试更有价值。** 日志记录 Agent 试过但没用 的方案，使下次运行不会重复走死路。

---

## 4. 项目结构

```
record-what-happened/
├── .gitignore              # IDE / OS 文件、task-logs/ 目录
├── LICENSE                 # MIT License
├── README.md               # 英文说明
├── README.zh.md            # 中文说明
└── skill/
    ├── SKILL.md            # Skill 主文件（~165 行），激活时加载
    ├── assets/
    │   └── TEMPLATE.md     # 日志模板（~148 行），生成日志时加载
    └── references/
        └── RECORDING_GUIDE.md  # 逐章节填写指南（~182 行），按需加载
```

### 4.1 渐进式加载策略

Agent 不会一次性加载所有文件，而是按优先级逐步加载：

| 优先级 | 文件 | 时机 | 行数 |
|--------|------|------|------|
| 必读 | `SKILL.md` | 激活时 | ~165 |
| 必读 | `assets/TEMPLATE.md` | 生成日志时 | ~148 |
| 按需 | `references/RECORDING_GUIDE.md` | 不确定如何填写时 | ~182 |

---

## 5. 安装方式

### 方式 A：项目级（推荐）

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

- **opencode**: `~/.config/opencode/skills/task-log/`
- **Claude Code**: `~/.claude/skills/task-log/`

> `SKILL.md` 的 `name` 字段为 `task-log`，目录名必须一致。

---

## 6. 使用方式

### 6.1 手动触发

任务完成后，对 Agent 说：

```
记录一下刚才的任务过程
```

Agent 会在 `task-logs/` 下生成结构化日志，并返回文件路径。

### 6.2 自动触发

在 opencode 中配置本 skill 为 post-task hook，即可在每次任务后自动执行。

### 6.3 触发时机

- 用户显式要求「记录任务」「整理过程」「写日志」
- 任务完成后自动触发（需配置 post-task hook）
- 任务失败或中途取消时，记录已执行部分

---

## 7. 六步工作流程

| 步骤 | 内容 |
|------|------|
| **第 1 步** | 确定日志存放位置：`task-logs/TASK-{编号}-{slug}.md`，编号自动递增 |
| **第 2 步** | 读取 `assets/TEMPLATE.md` 作为骨架 |
| **第 3 步** | 按模板结构逐节填写，遵循 5 条核心原则 |
| **第 4 步** | 补充「信息发现」章节 |
| **第 5 步** | 如实记录验证手段和结果 |
| **第 6 步** | 写入文件并告知用户（路径、编号、状态摘要） |

### 7.1 文件命名规范

- 格式：`TASK-{编号}-{简短slug}.md`
- 编号：四位补零，从 `0001` 开始，自动递增
- slug：仅小写字母、数字、连字符 `-`、下划线 `_`，长度不超过 50 字符
- 续篇：超过 500 行时，拆分为 `TASK-{编号}-{slug}-part2.md` 等

---

## 8. 核心原则

| # | 原则 | 说明 |
|---|------|------|
| 1 | **安全优先** | 密钥、Token、PII 在记录前用 `[REDACTED]` 脱敏，与源代码同等保密 |
| 2 | **事实与判断分离** | 报错原文（脱敏后）与分析推理分开记录 |
| 3 | **失败比成功重要** | 失败方案连同失败原因一起记录，这是可复用知识 |
| 4 | **控制信噪比** | 关键决策/异常展开，顺利步骤一行带过；单条日志不超过 500 行 |
| 5 | **可检索性** | YAML front-matter 完整填写，tags 从受控词表选取 |

### 8.1 受控标签词表

```
bugfix | feature | refactor | config | research | test | doc | chore | hotfix
```

必要时可新增，但应在日志中注明理由。

### 8.2 status 枚举

```
completed | partial | failed | cancelled
```

---

## 9. YAML Front-Matter 规范

```yaml
---
task_id: "TASK-0001"      # 项目内唯一，四位补零
session_id: ""            # 如环境未暴露则留空
title: ""                 # 任务标题
date: "YYYY-MM-DD"
start_time: "HH:MM"
end_time: "HH:MM"
duration: ""              # 耗时
status: "completed"       # 枚举值
tags: []                  # 受控词表
related_tasks: []         # 关联任务 ID
key_files: []             # 相对路径
prev_task: ""             # 延续任务时填写上一个 task_id
---
```

---

## 10. 边缘情况处理

| 场景 | 处理方式 |
|------|----------|
| **任务失败/取消** | 仍生成日志，`status` 设为 `failed`/`cancelled`，记录到中断点 |
| **极简任务** | 日志可精简，但 front-matter 和完成标准不可省略 |
| **多会话接力** | 同一任务：`task_id` 不变，末尾追加新会话段落；新任务：`task_id` 递增，`prev_task` 填旧 ID |
| **日志过大** | 超过 500 行时拆分为续篇 `-part2.md`、`-part3.md` 等 |
| **敏感信息** | 脱敏优先于一切记录原则 |
| **日志保留** | 与源代码同等保密；长期项目可定期归档 90 天前的日志 |

---

## 11. 记录优先级

当简洁与完整产生冲突时：

> **关键决策与异常 > 失败尝试 > 验证结果 > 顺利步骤**

---

## 12. 示例日志片段

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
