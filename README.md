# record-what-happened

[中文 README](./README.zh.md)

> Turn 200 lines of Agent noise into a 50-line traceable task log.

## The Problem

AI coding agents (Claude Code, opencode, Cursor, etc.) are powerful, but their output is **high-noise, low-signal**. When an agent completes a task, you're left with a wall of tool calls, partial outputs, and scattered reasoning — nearly impossible to skim after the fact.

Worse: next time you work on the same project, the agent has **zero memory** of what it tried, what failed, and what it learned last time.

## The Solution

**record-what-happened** is an [Agent Skill](https://agentskills.io) that instructs your AI coding agent to generate a **structured, low-noise task log** after each task — covering what it did, what went wrong, how it fixed it, and what it discovered along the way.

```
task-logs/
├── TASK-0001-setup-database.md
├── TASK-0002-fix-login-bug.md
└── TASK-0003-add-export-feature.md
```

Each log follows a consistent 6-section structure:

1. **Task Context** — original instruction, subtask breakdown, success criteria
2. **Execution Trace** — step summary + key decisions expanded (not every tool call)
3. **Problems & Solutions** — what broke, root cause, failed attempts, final fix, leftover TODOs
4. **Information Discoveries** — facts about the project that differed from expectations
5. **Verification & Results** — how completion was validated, artifacts changed
6. **Retrospective** (optional) — what went well, what to improve

The key insight: **failed attempts are more valuable than successful ones.** The log records what the agent tried that *didn't* work — so the next run doesn't repeat the same dead ends.

## Install

### Option A: Project-level (recommended)

Copy the `skill/` directory into your project's skill folder:

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

### Option B: Global (all projects)

Copy `skill/` to your global skills directory:

- **opencode**: `~/.config/opencode/skills/task-log/`
- **Claude Code**: `~/.claude/skills/task-log/`

> The skill's `name` field in `SKILL.md` is `task-log` — the directory must match this name.

## Usage

### Manual trigger

After a task completes, tell your agent:

```
记录一下刚才的任务过程
```

or

```
Log this task
```

The agent will generate a structured log in `task-logs/` and report the file path.

### Automatic trigger (post-task hook)

In opencode, you can configure this skill to run automatically after every task. See the [opencode documentation](https://opencode.ai) for hook configuration.

## How It Works

The skill uses **progressive disclosure** to minimize context usage:

```
skill/
├── SKILL.md              # Loaded on activation (~165 lines)
│                         # Contains: trigger conditions, 6-step workflow, 5 core principles
├── assets/
│   └── TEMPLATE.md       # Loaded when generating a log (~148 lines)
│                         # The skeleton with 6 sections + YAML front-matter
└── references/
    └── RECORDING_GUIDE.md  # Loaded on-demand only (~170 lines)
                            # Detailed per-section guide with good/bad examples
```

The agent doesn't load everything at once — it reads `SKILL.md` first, pulls in the template when writing, and only consults the reference guide when uncertain about a specific section.

### Core Principles

| Principle | What it means |
|-----------|---------------|
| **Safety first** | Secrets, tokens, PII are redacted before logging — even in error messages |
| **Fact vs. judgment** | Raw outputs (redacted) are separated from analysis and reasoning |
| **Failures > successes** | Failed approaches are recorded with reasons — that's the reusable knowledge |
| **Signal over noise** | Routine steps get one line; only decisions and exceptions are expanded |
| **Searchability** | YAML front-matter with task ID, tags, key files for later retrieval |

## Compatibility

Any agent environment that supports file read/write and the [Agent Skills spec](https://agentskills.io/specification):

- opencode
- Claude Code
- Similar tools with skill/plugin support

> **Note on language**: The skill instructions are written in Chinese. This does not affect agent execution — modern LLMs handle Chinese instructions as well as English. The generated logs will follow whatever language your session uses.

## Example Log

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
## 3. Problems & Solutions

### Problem 1 — Token expires immediately after login
- **Root cause**: Expiration hardcoded to 0 in login.ts
- **Attempt 1**: Modify config file → Failed (overwritten at runtime)
- **Attempt 2**: Read from env var in login.ts:42 → Success
- **Leftover**: Config override mechanism needs investigation
```

## License

MIT
