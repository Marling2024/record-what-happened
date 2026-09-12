# record-what-happened

[English README](./README.md)

record-what-happened 是一个 Agent Skill。它让编码 Agent（opencode、Claude Code、Cursor 这类）在每个任务结束时写一份结构化日志：做了哪些步骤，哪里出了问题，怎么改的，以及试过但没用的方案。日志统一放在项目根目录的 `task-logs/` 下。

它还支持按需回顾历史日志、把可复用的教训固化成经验库，以及在经验过期时整合清理。

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

## 四种模式

| 模式 | 作用 | 产出 |
|------|------|------|
| 记录（默认） | 任务结束后生成结构化日志 | `task-logs/TASK-*.md` |
| 回顾 | 按主题或全量检索历史日志并总结 | 聊天内报告，不写文件 |
| 固化 | 把可复用结论提炼为经验条目 | `experience/EXPERIENCE.md` + `experience/temp/EXP-*.md` |
| 整合 | 剔除过期、合并重复、压缩主经验文件 | 更新后的经验库 |

经验库分两层：

```
experience/
├── EXPERIENCE.md          # 主经验文件：精炼、可执行，运行时参考（类似 AGENTS.md 的作用）
└── temp/                  # 逐条明细与依据，整合吸收后可删除
```

想让 Agent 在日常任务里自动参考，可在项目的 `AGENTS.md`（或 `CLAUDE.md`）里引用 `experience/EXPERIENCE.md`。`task-logs/` 与 `experience/` 都是用户数据，建议加入你项目自己的 `.gitignore`。

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
            │   ├── TEMPLATE.md
            │   ├── TEMPLATE-EXP.md
            │   └── TEMPLATE-EXPERIENCE.md
            └── references/
                ├── RECORDING_GUIDE.md
                ├── REVIEW_GUIDE.md
                └── CONSOLIDATE_GUIDE.md
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

其他模式同理：回顾（「回顾一下登录相关的日志」）、固化（「把经验固化下来」）、整合（「整合一下经验库」）。

### 自动触发

在 opencode 里把本 skill 配成任务结束后的 hook，即可每次任务自动生成。配置方式见 [opencode 文档](https://opencode.ai)。

## 工作原理

Skill 按需加载文件，而不是一次全读，以控制上下文占用：

```
skill/
├── SKILL.md                    # 激活时加载，约 190 行：四种模式、记录流程、记录原则
├── assets/
│   ├── TEMPLATE.md             # 日志骨架，约 148 行：6 章节加 front-matter
│   ├── TEMPLATE-EXP.md         # 经验条目骨架，约 32 行
│   └── TEMPLATE-EXPERIENCE.md  # 主经验文件骨架，约 21 行
└── references/
    ├── RECORDING_GUIDE.md      # 记录模式按需加载，约 180 行：逐章节填写指南
    ├── REVIEW_GUIDE.md         # 回顾/固化模式加载，约 250 行：检索协议、报告结构、经验库规范
    └── CONSOLIDATE_GUIDE.md    # 仅整合模式加载，约 60 行：整合清理原则
```

Agent 先读 `SKILL.md`，然后只加载当前模式需要的引用与模板文件。

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
