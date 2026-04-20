# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of **Claude Code skills** — reusable, invocable workflows defined in Markdown. Each skill lives in its own directory and may include sub-agents in an `agents/` subdirectory. There are no build, test, or lint commands; skills are Markdown specifications executed by Claude's agent system.

## Skill Structure

Each skill follows this layout:

```
<skill-name>/
├── SKILL.md          # Entry point: front-matter + orchestration instructions
└── agents/           # Optional sub-agent definitions
    └── <agent>.md    # Role, tools, and step-by-step behavior for each sub-agent
```

**SKILL.md front-matter fields** (YAML):
- `name`: Display name
- `description`: When to trigger the skill (used by Claude to match invocations)
- `tools`: Allowed tool list (restricts what Claude may call)
- `user-invocable`: Whether the user can invoke it via `/skill-name`

## Current Skills

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `commit-description` | Implicit on commit | Generates conventional commit messages with structured body |
| `frontend-docs` | `/frontend-docs` | Produces Italian functional documentation for frontend apps via 3-phase orchestration with user checkpoints |
| `security-audit` | `/security-audit` | Runs a full codebase security audit: maps sections, spawns parallel auditor agents (batches of max 5), consolidates findings into `analysis/SECURITY_AUDIT_REPORT.md` |
| `prompt-enhancer` | Any message ending with `!!` | Rewrites vague/multilingual prompts into structured English before execution |

## Adding or Editing Skills

- **New skill**: create `<skill-name>/SKILL.md` with proper front-matter; add `agents/` only if the skill delegates to sub-agents. After creating it, always suggest the user run:
  ```
  ln -s $(pwd)/<skill-name> ~/.claude/skills/<skill-name>
  ```
  to link the skill into Claude Code's skills folder so it becomes available.
- **Sub-agents**: each agent file specifies its own role, allowed tools, inputs, and output format. Keep agent files focused on a single responsibility.
- **Output files**: skills that write intermediate files (e.g., `_pages_map.json`, `_doc_*.md`) must clean them up before finishing.
- **Language**: skill instructions are in English; output language is defined per-skill (e.g., frontend-docs outputs Italian).

## Conventions

- Commit messages follow conventional commits: `type(scope): description` with an optional structured body (`## What` / `## Changes`).
- The `prompt-enhancer` skill must be checked **before any other action** on every user message — its trigger check is mandatory.
- User checkpoints (pausing for review before continuing) are a first-class pattern used by multi-phase skills.
