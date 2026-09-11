# record-what-happened

[English README](./README.md)

record-what-happened 是一个 Agent Skill。它让编码 Agent（opencode、Claude Code、Cursor 这类）在每个任务结束时写一份结构化日志：做了哪些步骤，哪里出了问题，怎么改的，以及试过但没用的方案。日志统一放在项目根目录的 `task-logs/` 下。

这样你事后能快速看清 Agent 做了什么，不用翻它的原始输出；Agent 下次开工先读历史日志，也能知道哪些路走不通，不必重复。失败记录尤其有用：Agent 试过又放弃的方案，往往正是最该留下的信息。

每份日志都是固定的 6 个章节：

1. 任务上下文：原始指令、子任务拆解、完成标准
2. 执行轨迹：步骤概要，只展开关键决策，不逐条记工具调用
3. 遇到的问题与解决：现象、根因、试过的方案（含失败的）、最终修复、遗留项
4. 信息发现：执行中了解到的、和预期不符的项目事实
5. 验证与结果：怎么确认做完了、改了哪些文件
6. 回顾笔记（可选）：做得好的、可改进的

```
task-logs/
├── TASK-0001-setup-database.md
├── TASK-0002-fix-login-bug.md
└── TASK-0003-add-export-feature.md
```

## 安装

### 项目级（推荐）

把 `skill/` 复制到项目的 skills 目录：

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

### 全局（所有项目共享）

把 `skill/` 复制到全局 skills 目录：

- opencode：`~/.config/opencode/skills/task-log/`
- Claude Code：`~/.claude/skills/task-log/`

`SKILL.md` 里的 `name` 是 `task-log`，目录名必须一致。

## 使用

### 手动触发

任务结束后对 Agent 说：

```
记录一下刚才的任务过程
```

Agent 会在 `task-logs/` 下生成日志，并告诉你文件路径。

### 自动触发

在 opencode 里把本 skill 配成任务结束后的 hook，即可每次任务自动生成。配置方式见 [opencode 文档](https://opencode.ai)。

## 工作原理

Skill 按需加载文件，而不是一次全读，以控制上下文占用：

```
skill/
├── SKILL.md                # 激活时加载，约 165 行：触发条件、6 步流程、记录原则
├── assets/
│   └── TEMPLATE.md         # 写日志时加载，约 148 行：6 章节骨架加 front-matter
└── references/
    └── RECORDING_GUIDE.md  # 用到才加载，约 170 行：逐章节填写指南
```

Agent 先读 `SKILL.md`，写日志时取模板，只有某章节不确定怎么填时才查参考指南。

记录时遵循五条原则：

| 原则 | 含义 |
|------|------|
| 安全 | 密钥、token、PII 在记录前用 `[REDACTED]` 替换，报错原文也不例外；日志与源码同等保密 |
| 事实与判断分开 | 脱敏后的原始输出归事实，分析推理归判断，互不混淆 |
| 记录失败 | 试过的方案连同失败原因一起写，这是下次用得上的知识 |
| 控制篇幅 | 顺利步骤一行带过，只展开决策和异常；单条日志不超过 500 行 |
| 方便检索 | front-matter 写全 task_id、tags、key_files，tags 从固定词表里选 |

## 兼容性

支持文件读写、遵循 [Agent Skills 规范](https://agentskills.io/specification) 的环境都可以用：

- opencode
- Claude Code
- 其他同类工具

Skill 指令用中文写成，这不影响执行：模型处理中文指令没有问题，生成的日志跟随你会话的语言。

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

### 问题 1：登录后 token 立即过期
- 根因：login.ts 里过期时间硬编码为 0
- 尝试 1：改配置文件，失败，运行时被覆盖
- 尝试 2：在 login.ts:42 改为读环境变量，成功
- 遗留：配置覆盖机制还要再查
```

## License

MIT
