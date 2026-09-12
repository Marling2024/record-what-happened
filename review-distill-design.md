# 详细设计方案：回顾（Review）与经验固化（Distill）

> 目标版本：v1.2（当前 v1.1）
> 状态：待评审
> 已确认前提：扩展现有 `task-log` 单 skill；`experience/` 纳入 `.gitignore`；本文件评审通过后再实施

---

## 1. 需求与设计约束

| # | 需求 | 对应模式 | 产出物 |
|---|------|----------|--------|
| 1 | 指导 LLM 回顾全部日志或特定主题日志 | **review** | 聊天内总结报告（只读，无副作用） |
| 2 | 回顾日志并总结生成固化经验 | **distill** | `experience/EXP-{编号}-{slug}.md` |
| 3 | 维持易检索 | 设计约束 | 复用 front-matter + 受控 tags + 命名规范，双目录统一检索协议 |

沿用现有架构的三条不变项：
- **渐进式加载**：新功能细节放按需加载的 reference，不膨胀日常记录模式的上下文
- **纯 markdown**：无脚本、无索引文件、无新依赖
- **五条记录原则**（脱敏、事实/判断分离、记录失败、控制信噪比、可检索）对三种模式同等生效

---

## 2. 总体设计：单 skill 三模式

### 2.1 模式判定

| 用户意图 | 模式 | 关键词示例 |
|----------|------|-----------|
| 记录本次任务 | record | 记录任务、写日志、整理过程 |
| 看历史、找规律、复盘 | review | 回顾、复盘、以前怎么处理的、之前踩过什么坑 |
| 把经验写成文件 | distill | 固化经验、沉淀经验、总结经验文件、把教训存下来 |

规则：
- 「总结一下」等模糊表达 → 先走 **review**，报告末尾列出「可固化经验候选」并询问是否执行 distill（审核门）
- **distill 只在显式要求或用户确认后执行**；review 本身纯只读
- 允许链式：record → review → distill；一次会话可完成多步

### 2.2 模式关系

```
task-logs/*.md ──检索──> [review] ──候选──> 用户确认 ──> [distill] ──> experience/*.md
       ▲                                                                      │
       └────────────────── 后续任务读取经验，避免重复踩坑 ◄───────────────────┘
```

---

## 3. 文件级改动清单

| 文件 | 操作 | 说明 | 预计规模 |
|------|------|------|----------|
| `skill/SKILL.md` | 修改 | description 扩展触发词；新增「四种模式」章节；版本 1.1 → 1.2；allowed-tools 收窄 | +25~35 行 |
| `skill/references/REVIEW_GUIDE.md` | **新增** | 回顾流程 + 固化流程 + 检索协议 + 经验库规范（按需加载） | 200~240 行 |
| `skill/references/CONSOLIDATE_GUIDE.md` | **新增** | 经验整合原则（仅在整合时加载） | 60~90 行 |
| `skill/assets/TEMPLATE-EXP.md` | **新增** | temp 经验条目骨架（固化时按需加载） | ~32 行 |
| `skill/assets/TEMPLATE-EXPERIENCE.md` | **新增** | 主经验文件骨架（新建/重写主文件时加载） | ~21 行 |
| `.gitignore` | 修改 | 增加 `experience/` | +2 行 |
| `README.md` / `README.zh.md` | 修改 | 新功能说明、`experience/` 说明、加载矩阵更新 | +15~25 行/个 |
| `skill/assets/TEMPLATE.md` | **不改** | 日志模板与回顾/固化无关 | - |
| `skill/references/RECORDING_GUIDE.md` | **不改** | 记录流程不变 | - |

不新增 skill、不改目录名（`name: task-log` 保留，改名会破坏所有已安装副本的目录约定）。

---

## 4. SKILL.md 改动详情

### 4.1 description 替换草案（前置对照）

现在：
```
在 Agent 每次执行完任务后，自动生成结构化的任务执行时序记录……当用户希望任务过程
清晰可追查、便于回溯、或抱怨 Agent 输出信噪比低难以阅读时，都应使用本 skill。
```

改为：
```
任务日志三模式 skill。①记录：任务结束后生成结构化执行日志（上下文、轨迹、问题与解决、
信息发现、验证）；②回顾：按主题或全量检索历史日志并总结（复现问题、失败方案、候选经验）；
③固化：把日志中的可复用结论提炼为经验文件，供后续任务读取。当用户要求记录任务、
回顾/复盘历史工作、总结经验沉淀知识，或抱怨 Agent 输出信噪比低难以回溯时使用。
```

要点：模式关键词前置、总行数与原 description 持平（避免影响激活判定）。

### 4.2 新增章节草案（主体后部，约 30 行）

```markdown
## 四种模式

| 模式 | 触发 | 加载文件 | 产出 |
|------|------|----------|------|
| 记录（默认） | 记录/写日志 | TEMPLATE.md（+RECORDING_GUIDE 按需） | task-logs/TASK-*.md |
| 回顾 | 回顾/复盘/查历史 | references/REVIEW_GUIDE.md | 聊天内总结报告 |
| 固化 | 总结经验文件 | references/REVIEW_GUIDE.md + assets/TEMPLATE-EXP.md（新建主文件时加 assets/TEMPLATE-EXPERIENCE.md） | experience/EXPERIENCE.md + temp/EXP-*.md |
| 整合 | 整合/清理经验 | references/CONSOLIDATE_GUIDE.md（重写主文件时加 assets/TEMPLATE-EXPERIENCE.md） | 更新主文件、清理 temp |

- 「总结一下」类模糊请求 → 先回顾，报告末尾列候选并询问是否固化
- 固化产物分两层：`experience/EXPERIENCE.md` 主经验文件（运行时参考入口，类似 AGENTS.md 的作用）+ `experience/temp/` 明细暂存
- 回顾/固化/整合不改变日志记录流程；记录模式的六步流程与五条原则不变
- 检索协议（tags、front-matter、命名规范）各模式共用，详见 REVIEW_GUIDE
```

### 4.3 引用文件加载策略（SKILL.md 现有章节）改写

- 记录模式：必读 `assets/TEMPLATE.md`；`references/RECORDING_GUIDE.md` 按需
- 回顾模式：必读 `references/REVIEW_GUIDE.md`，不加载任何模板
- 固化模式：必读 `references/REVIEW_GUIDE.md`；写条目时读 `assets/TEMPLATE-EXP.md`；新建主文件时读 `assets/TEMPLATE-EXPERIENCE.md`
- 整合模式：必读 `references/CONSOLIDATE_GUIDE.md`；重写主文件时读 `assets/TEMPLATE-EXPERIENCE.md`

---

## 5. references/REVIEW_GUIDE.md 设计（新文件）

### 5.1 章节结构

1. 检索协议（回顾/固化共用）
2. 回顾流程（review）
3. 回顾报告结构
4. 经验固化流程（distill）
5. 经验库结构与文件规范（主文件 + temp 明细）
6. 新建 vs 更新判定
7. 冲突与失效处理
8. 质量门槛（反垃圾经验）
9. 边界情况
10. 整合（consolidate）概述（细节在 CONSOLIDATE_GUIDE.md）

### 5.2 检索协议（两阶段，防止上下文爆炸）

**Stage 1 — 快筛**（只读元数据，消耗极小）：
- `Glob task-logs/TASK-*.md` 取全量清单
- `Grep` 模式 `^(task_id|exp_id|title|date|status|tags):` 于 `task-logs/` 与 `experience/temp/` → 每个文件约 5 行元数据
- 按主题追加关键词 Grep（不区分大小写，标题+正文），或用 `tags:` 过滤受控词表

**Stage 2 — 精读**：读取命中文件全文，仅展开与回顾主题相关的章节（问题、失败尝试、发现）。

**预算阈值**（写入 guide，硬性约束）：
- 单批精读 ≤ 10 份日志
- 命中超过 15 份或元数据总量 > 100KB → 按「月份」或「tag」分批做 map-reduce：每批产出要点，再合并总结
- 严禁一次性全量读取所有日志
- **进度分期同步**：Stage 1 完成后先向用户报一句「已扫描 N 份，命中 M 份，开始精读」，避免长任务全程无输出

### 5.3 回顾报告结构（聊天输出，不写文件）

```markdown
# 回顾：{主题 | 全量}（{时间范围}）
## 1. 范围与命中
检索方式、命中数、时间跨度、使用的关键词
## 2. 主题脉络
按时间线梳理：TASK-XXXX 做了什么、结果如何
## 3. 反复出现的问题
问题 → 出现次数 → 来源任务 → 当前状态（已解决 / 悬而未决）
## 4. 失败方案汇总
| 方案 | 失败原因 | 来源 | 是否值得再试 |
## 5. 可固化经验候选
候选：{一句话结论} ← 来源 TASK-XXXX（建议固化）
## 6. 数据缺口
日志没回答的问题；建议后续记录时补充什么
## 7. 与已固化经验的差异（仅主文件存在时）
日志新事实 vs 主文件条目；需更新/失效的条目建议
```

约束：报告基于日志事实，不编造；引用的日志必须存在；发现的敏感信息同样脱敏。

### 5.4 经验固化流程（distill，7 步）

1. **划定范围**：来源 = 全量 / tag / 时间范围 / 指定 task_id 列表
2. **先查经验库**：读 `experience/EXPERIENCE.md`（不存在则视为空库）+ `Glob experience/temp/EXP-*.md` + 同义词 Grep → 判定新建还是更新（见 5.6）
3. **检索日志**：复用 5.2 两阶段协议
4. **抽取候选**：每条候选必须通过质量门槛（见 5.8）
5. **冲突检查**：新结论与已有经验矛盾时按 5.7 处理
6. **写入/更新**：先写 temp 明细文件（新建或更新 + changelog，front-matter 同步 `updated`、`source_tasks`），再把结论即时并入/更新主文件 `experience/EXPERIENCE.md`
7. **回报**：文件路径、新建/更新、条数、被推翻的旧结论清单；只报入口与变更点，不复述文件全文

### 5.5 经验库结构与文件规范（主文件 + temp 明细）

目录结构（已决策）：

```
experience/
├── EXPERIENCE.md          # 主经验文件：精炼、可执行，运行时参考入口（类似 AGENTS.md 的作用）
└── temp/                  # 明细暂存：EXP-*.md 逐条经验文件，整合吸收后可删除
```

**主经验文件 `experience/EXPERIENCE.md`**：
- 定位：运行时参考入口，只放精炼结论；目标 ≤200 行，超出触发整合建议（见 5.10）
- 骨架：`assets/TEMPLATE-EXPERIENCE.md`，创建/重写前读取
- 结构：H1 标题 + 「最后更新」行 + 按主题分节；条目格式：
  - `- [稳定] {一句话结论}（来源 TASK-XXXX / 详情 EXP-XXXX）`
  - 时变型条目追加子行 `复核触发：{条件}`
- 由 distill 即时并入/更新，保证运行时参考始终最新

**明细条目 `experience/temp/EXP-{编号}-{slug}.md`**：
- 编号：Glob `experience/temp/EXP-*.md` 取最大值 +1；slug 约束同日志（小写字母、数字、连字符、下划线，≤50 字符）
- 单文件 ≤500 行，超出按子主题拆分（原文件标 `superseded` + `superseded_by`）
- 骨架：`assets/TEMPLATE-EXP.md`，新建/更新前读取

front-matter：
```yaml
---
exp_id: "EXP-0001"
title: ""
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
status: active            # active | superseded
tags: []                  # 与日志同一受控词表
source_tasks: []          # [TASK-0003, TASK-0007]
related_exps: []
superseded_by: ""
---
```

正文 6 章节：
1. **结论**：一句话，可执行、可判定（不是心得，是「下次遇到 X 就这么做/不要这么做」）
2. **适用场景与前提**：什么条件下成立
3. **时效性**：标注 `稳定型`（项目约定、架构事实，不随时间/版本漂移）或 `时变型`（依赖版本行为、外部工具、环境）；时变型必须写明**复核触发条件**（如「升级 Node 大版本后复核」），防止经验沉睡腐化
4. **依据**：来源 task_id + 现象/证据摘要（**自包含**，见 5.9 边界）
5. **反例与失效条件**：什么情况下结论不成立
6. **更新记录**：`YYYY-MM-DD 依据 TASK-XXXX 创建/修订/推翻 …`

正文示例：
```markdown
## 结论
路径别名直接在 vite.config.ts 配 alias，不要只改 tsconfig paths。

## 适用场景与前提
涉及构建期路径解析时；项目构建工具为 Vite（以 package.json 为准）。

## 时效性
时变型。复核触发条件：构建工具变更，或 Node/Vite 大版本升级后重新验证。

## 依据
- TASK-0003：报错 "Cannot find module '@/utils'"，改 tsconfig paths 无效；
  vite.config 加 alias 后构建通过（0 errors）。
- TASK-0007：新入口再次验证 alias 方案生效。

## 反例与失效条件
- 构建工具换为 webpack/tsup 时失效。
- `tsc --noEmit` 类型检查仍需 tsconfig paths，两者要同时配（TASK-0009 观察）。

## 更新记录
- 2026-09-12 创建（依据 TASK-0003、TASK-0007）
- 2026-10-01 TASK-0009 补充类型检查例外
```

### 5.6 新建 vs 更新判定

| 条件 | 动作 |
|------|------|
| 主文件中无同主题条目 | 新建主文件条目 + temp 新文件 |
| 主文件有同主题条目且结论可合并 | 就地更新主文件条目 + temp 新文件 + changelog |
| temp 同主题文件内容将超 500 行 | 拆分为子主题新文件；原文件 `status: superseded`、填 `superseded_by` |
| 只是新增独立结论、与旧文件主题相关但不同 | 新建，`related_exps` 互链 |

去重原则：**先 Grep 同义词再动笔**，同一知识点只应存在于一个 EXP 文件中。

### 5.7 冲突与失效处理

**指令优先级**（仲裁前提）：当次用户指令 > 经验文件 > 一般最佳实践。经验与最新日志事实冲突时，以日志事实为准并按下方流程标记失效，不得盲从经验。

- 新证据推翻旧结论：旧条目就地标记 `~~原结论~~（失效于 YYYY-MM-DD，原因，来源 TASK-XXXX）`，保留痕迹不删除
- 部分失效：保留成立部分，失效部分单独标注
- 证据不足以判定谁对：并排保留两个结论，各自标注来源与「未验证」，并提示后续任务需要什么证据才能裁决

### 5.8 质量门槛（反垃圾经验）

每条经验必须同时满足：
1. 结论可判定：具体到「做什么/不做什么/什么条件下」，拒绝「要注意代码质量」类空话
2. 至少一个具体来源：`task_id` + 当时的现象或结果摘要
3. 时变型结论必须带复核触发条件（见 5.5 正文第 3 节）
4. 推测必须显式标注「（未验证）」
5. 无法满足 → 不写入，在回报中说明缺什么证据、建议后续日志记录哪些信息

### 5.9 边界情况

| 场景 | 处理 |
|------|------|
| `task-logs/` 为空 | 如实告知无日志可回顾，不编造 |
| 主题零命中 | 扩大同义词重试一次；仍无则明说，并列出可用 tags |
| 日志数量超阈值 | 分批 map-reduce（5.2） |
| source_tasks 指向的日志已被清理/归档 | 经验文件必须**自包含证据摘要**，task_id 仅作溯源参考 |
| 同一 tag 下主题漂移 | 以关键词相关性为准，必要时拆成多个 EXP 文件 |
| 回顾/固化中发现敏感信息 | 与日志同规则脱敏；经验是浓缩物，泄露危害更大，宁可少写 |
| 用户只要回顾、不做固化 | 尊重，不自动写 experience/ |

### 5.10 整合（consolidate）概述

**触发**（满足任一即建议，用户确认后执行；不自动删除文件）：
- 用户显式要求「整合/清理经验」
- `experience/temp/` 中文件数 ≥50
- 主文件超过 200 行
- 回顾/固化中发现明显重复、过期或互相矛盾的条目

**原则**（细节独立成 `references/CONSOLIDATE_GUIDE.md`，仅整合时加载）：
- 保留：反复验证、稳定型、跨任务命中高的结论
- 合并：重复/近义条目压缩为一条，来源合并
- 剔除：复核触发条件已满足且未复核、被新证据替代、涉及环境已不存在的条目
- 失效结论在主文件保留一行痕迹（失效日期 + 来源），对应 temp 明细文件可删除
- 不确定的一律保守保留并标「待核实」，不猜、不删
- 整合后更新主文件日期，回报删除/合并/保留统计

---

## 6. 检索设计（需求 3 的落实）

同一套协议覆盖 `task-logs/` 与 `experience/`：

| 需求 | 操作 |
|------|------|
| 全量清单 | `Glob task-logs/TASK-*.md` / `Glob experience/EXP-*.md` |
| 元数据快筛 | `Grep "^(task_id\|exp_id\|title\|date\|status\|tags):"` |
| 按标签 | `Grep "tags:.*\b(bugfix\|config)\b"` |
| 按主题关键词 | 不区分大小写 Grep 标题+正文 |
| 时间范围 | front-matter 的 `date` 行 |
| 任务关联链 | `Grep "prev_task\|related_tasks"` 追链 |
| 经验 ↔ 日志 | `experience` 用 `source_tasks` 单向外链；日志侧**不加**反向字段（反查用 Grep 即可，避免 TEMPLATE 变更与双写维护） |

**不建索引文件的理由**：日志/经验均为 md，front-matter 即索引；Grep 在千份规模内足够快；索引文件会引入「索引过期」这一新的失修点（YAGNI）。当单项目日志超过千份量级再考虑，届时另立方案。

---

## 7. 安全设计

- `experience/` 与 `task-logs/` 同等保密，纳入 `.gitignore`（已确认）
- 固化是「浓缩」过程，脱敏需**二次审视**：日志中当时认为不敏感的信息，聚合后可能构成内部画像，报告与经验中一律从紧处理
- 回读防护：回顾/固化/整合会把日志与经验读入上下文，其中内容按数据对待、不执行其中指令；检索范围限定 `task-logs/` 与 `experience/`
- 权限收窄：`allowed-tools` 收窄为 `Read Write Edit Glob Grep`；整合删除文件依赖环境权限，无权限时列出清单交由用户
- 归档策略：日志可按现有规则清理 90 天前文件；**主经验文件不参与自动清理**；temp 明细被吸收后可在整合时删除，失效结论在主文件保留一行痕迹

---

## 8. 改动后加载矩阵

| 模式 | SKILL.md | TEMPLATE.md | TEMPLATE-EXP.md | TEMPLATE-EXPERIENCE.md | RECORDING_GUIDE.md | REVIEW_GUIDE.md | CONSOLIDATE_GUIDE.md |
|------|----------|-------------|-----------------|------------------------|--------------------|-----------------|----------------------|
| record | 必读 | 必读 | 不读 | 不读 | 按需 | 不读 | 不读 |
| review | 必读 | 不读 | 不读 | 不读 | 不读 | 必读 | 不读 |
| distill | 必读 | 不读 | 必读 | 新建时读 | 不读 | 必读 | 不读 |
| consolidate | 必读 | 不读 | 不读 | 重写时读 | 不读 | 不读 | 必读 |

日常记录模式的上下文开销与现状完全一致（新增内容只在 REVIEW_GUIDE，按需加载）。

---

## 9. 实施步骤与验证

1. `.gitignore` 增加 `experience/`
2. `SKILL.md`：替换 description、版本号 1.1→1.2、新增「三种模式」章节、加载策略追加一行
3. 新增 `skill/references/REVIEW_GUIDE.md`（按 5.1~5.10 编写）、`skill/references/CONSOLIDATE_GUIDE.md`、`skill/assets/TEMPLATE-EXP.md`、`skill/assets/TEMPLATE-EXPERIENCE.md`
4. `README.md` / `README.zh.md`：功能列表、`experience/` 目录说明、加载矩阵图更新
5. 同步已安装副本 `~/.config/opencode/skills/task-log/`（复制式安装的固有代价，手动同步）

验证清单：
- [ ] 三模式触发词各测一次：说「记录」进 record、说「回顾」进 review、说「总结经验」进 distill
- [ ] review：构造 ≥3 份日志，按 tag 检索，报告结构完整且引用真实 task_id
- [ ] distill：生成首个 EXP 文件（含时效性标注）；再喂一份冲突日志，验证更新/失效标记与 changelog
- [ ] distill：时变型结论缺少复核触发条件时不得写入（质量门槛生效）
- [ ] 检索：Grep 快筛模式在 `experience/temp/` 同样命中
- [ ] consolidate：构造触发条件（模拟 ≥50 条目或主文件超行），执行整合后主文件压缩、明细删除、失效痕迹保留
- [ ] 固化：生成的 EXP 条目与主文件结构与 `assets/TEMPLATE-EXP.md` / `TEMPLATE-EXPERIENCE.md` 一致
- [ ] 回归：record 模式六步流程与日志格式无变化

---

## 10. 非目标（明确不做）

- 不建索引文件 / 数据库 / 缓存
- 不写脚本、不加自动化依赖
- 不改 `TEMPLATE.md` 与现有记录流程
- 不拆新 skill
- 不做自动定时总结（仍需显式触发或用户配置 hook）
- 不做经验文件的项目间共享（跨项目知识库超出本 skill 范围）

---

## 11. 风险与缓解

| 风险 | 级别 | 缓解 |
|------|------|------|
| description 膨胀导致激活判定变模糊 | 中 | 控制行数与原版持平，模式关键词前置 |
| 回顾时上下文爆炸 | 中 | 两阶段检索 + 分批阈值写死进 guide |
| 经验腐化、自相矛盾 | 中 | changelog + 失效标记 + 质量门槛 |
| 经验文件膨胀 | 低 | 500 行上限 + 子主题拆分机制 |
| 回顾报告流于流水账 | 中 | 固定报告结构（脉络/反复问题/失败方案/候选） |
| 已安装副本与仓库不同步 | 低 | 实施后手动同步；属复制式安装固有代价 |

---

## 12. 决策结论（已定）

1. **回顾报告留档**：不留档，纯聊天输出；需要时用户可让 Agent 另存。
2. **经验库架构**：采用「主经验文件 + temp 明细暂存」双层结构（本文件 5.5~5.6、5.10 已更新）：
   - `experience/EXPERIENCE.md` 为主经验文件，精炼、可执行，运行时参考入口，类似 AGENTS.md 的作用
   - 逐条明细文件放 `experience/temp/EXP-*.md`，整合吸收后可删除
   - 到达阈值（temp ≥50 个或主文件 >200 行）时**建议**用户整合，不自动执行；整合原则独立成 `CONSOLIDATE_GUIDE.md`，仅在需要时加载，指导剔除过期经验、留下可用结论
3. **日志反向字段**：不加 `related_experience`，维持单向外链（experience → task-logs），避免双写维护成本。
