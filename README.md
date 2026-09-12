# record-what-happened

[中文 README](./README.zh.md)

record-what-happened is an Agent Skill. It has your coding agent (opencode, Claude Code, Cursor and the like) write a structured log when a task ends: the steps it took, what broke, how it fixed it, and the approaches that didn't work. Logs go in a `task-logs/` directory at the project root.

It also reviews past logs on demand, distills the reusable lessons into an experience library, and consolidates that library when it grows stale.

You can skim afterward what the agent did without digging through its raw output, and on the next task the agent can read its own history to avoid dead ends it already hit. The failure records matter most: an approach the agent tried and gave up on is usually the most useful thing to keep.

Every log has the same six sections:

1. Task context: the original instruction, subtask breakdown, success criteria
2. Execution trace: step summaries, expanding only key decisions, not every tool call
3. Problems and fixes: symptom, root cause, attempts including failed ones, the final fix, leftovers
4. Discoveries: project facts learned during execution that differed from expectations
5. Verification: how completion was checked, which files changed
6. Retrospective (optional): what went well, what to improve

```
task-logs/
├── TASK-0001-setup-database.md
├── TASK-0002-fix-login-bug.md
└── TASK-0003-add-export-feature.md
```

## Modes

| Mode | What it does | Output |
|------|--------------|--------|
| Record (default) | Writes a structured log when a task ends | `task-logs/TASK-*.md` |
| Review | Retrieves and summarizes past logs, by topic or in full | Summary in chat, no files |
| Distill | Turns reusable lessons into experience entries | `experience/EXPERIENCE.md` + `experience/temp/EXP-*.md` |
| Consolidate | Prunes stale entries, merges duplicates, compacts the master file | Updated experience library |

The experience library has two layers:

```
experience/
├── EXPERIENCE.md          # master file: concise, actionable, meant to be read at runtime (AGENTS.md style)
└── temp/                  # per-entry files with evidence; can be deleted after consolidation
```

To have agents consult it during normal work, reference `experience/EXPERIENCE.md` from your project's `AGENTS.md` (or `CLAUDE.md`). Both `task-logs/` and `experience/` hold user-specific data; add them to your project's `.gitignore`.

## Install

### Project-level (recommended)

Copy `skill/` into your project's skills directory:

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

### Global (all projects)

Copy `skill/` into your global skills directory:

- opencode: `~/.config/opencode/skills/task-log/`
- Claude Code: `~/.claude/skills/task-log/`

The `name` field in `SKILL.md` is `task-log`; the directory must match it.

## Usage

### Manual trigger

After a task ends, tell the agent:

```
记录一下刚才的任务过程
```

or

```
Log this task
```

The agent writes a log to `task-logs/` and reports the file path.

Other modes work the same way: ask it to review past work ("回顾一下登录相关的日志"), distill lessons ("把经验固化下来"), or consolidate the library ("整合一下经验库").

### Automatic trigger

In opencode, configure this skill as a post-task hook to run it after every task. Hook setup is in the [opencode docs](https://opencode.ai).

## How it works

The skill loads files on demand instead of all at once, to keep context usage down:

```
skill/
├── SKILL.md                    # loaded on activation, ~190 lines: modes, record workflow, recording rules
├── assets/
│   ├── TEMPLATE.md             # task log skeleton, ~148 lines: six sections plus front-matter
│   ├── TEMPLATE-EXP.md         # experience entry skeleton, ~32 lines
│   └── TEMPLATE-EXPERIENCE.md  # master experience file skeleton, ~21 lines
└── references/
    ├── RECORDING_GUIDE.md      # record mode, on demand, ~180 lines: per-section guide
    ├── REVIEW_GUIDE.md         # review/distill modes, ~250 lines: retrieval protocol, report structure, experience spec
    └── CONSOLIDATE_GUIDE.md    # consolidate mode only, ~60 lines: pruning and merging rules
```

The agent reads `SKILL.md` first, then loads only the reference and template files its mode needs.

Five rules apply while recording:

| Rule | Meaning |
|------|---------|
| Safety | redact secrets, tokens, and PII with `[REDACTED]` before logging, error output included; treat logs like source code |
| Facts vs. judgment | raw (redacted) output stays separate from analysis and reasoning |
| Record failures | write down abandoned approaches and why they failed; that's the reusable part |
| Control length | one line for routine steps, expand only decisions and exceptions; keep a log under 500 lines |
| Searchability | fill in task_id, tags, and key_files in front-matter; tags come from a fixed vocabulary |

## Compatibility

Any environment that can read and write files and follows the [Agent Skills spec](https://agentskills.io/specification):

- opencode
- Claude Code
- similar tools

The skill instructions are written in Chinese. This does not affect execution: models handle Chinese instructions as well as English, and generated logs follow the language of your session.

## Example log

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
## 3. Problems and fixes

### Problem 1: token expires immediately after login
- root cause: expiration hardcoded to 0 in login.ts
- attempt 1: edit the config file, failed, overwritten at runtime
- attempt 2: read from an env var at login.ts:42, works
- leftover: the config override mechanism still needs investigation
```

## License

MIT
