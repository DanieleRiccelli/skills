# Claude Code Skills

A collection of reusable skills for [Claude Code](https://claude.ai/code) — invocable workflows defined in Markdown that extend Claude's default behavior.

## What is a skill?

A skill is a Markdown file that tells Claude Code how to behave in a specific scenario. Skills can be triggered automatically (by pattern matching), manually via slash commands, or implicitly during normal interactions.

## Skills

### `commit-description`

Generates structured commit messages following the [Conventional Commits](https://www.conventionalcommits.org/) spec.

**Triggers:** automatically when creating a commit or when asked to summarize changes.

**Output format:**
```
type(scope): short imperative description [vX.Y.Z]

## What
One sentence explaining what this commit does.

## Changes
- Bullet points of specific changes
```

---

### `frontend-docs`

Generates complete functional documentation for a frontend application through a 3-phase orchestration with user checkpoints.

**Trigger:** `/frontend-docs`

**Phases:**
1. **Page mapping** — detects framework, analyzes routing, produces a page map for review
2. **Page analysis** — spawns parallel sub-agents to analyze each page
3. **Assembly** — assembles the final `DOCUMENTAZIONE_FUNZIONALE.md` and cleans up temp files

Supports Angular, React, Vue, Next.js, Nuxt, Svelte, and more. Output is in Italian.

---

### `prompt-enhancer`

Intercepts vague or multilingual prompts and rewrites them into clear, structured English before execution.

**Trigger:** any message ending with `!!`

**Examples:**
```
fix this bug !!
crea una funzione python !!
Ho bisogno di fare una modifica ... !!
```

The enhancement happens silently — the user sees only the final answer.

---

## Structure

```
<skill-name>/
├── SKILL.md          # Entry point: front-matter + orchestration instructions
└── agents/           # Optional sub-agents
    └── <agent>.md    # Role, tools, and step-by-step behavior
```

**SKILL.md front-matter fields:**

| Field | Description |
|-------|-------------|
| `name` | Display name |
| `description` | When to trigger the skill |
| `tools` | Allowed tools (restricts what Claude may call) |
| `user-invocable` | Whether the user can invoke it via `/skill-name` |

## Installation

Clone this repo and symlink the skills you want into Claude Code's skills folder:

```bash
git clone https://github.com/DanieleRiccelli/skills.git ~/claude-skills
cd ~/claude-skills

# Link individual skills
ln -s $(pwd)/commit-description ~/.claude/skills/commit-description
ln -s $(pwd)/frontend-docs ~/.claude/skills/frontend-docs
ln -s $(pwd)/prompt-enhancer ~/.claude/skills/prompt-enhancer
```

## Adding a new skill

1. Create `<skill-name>/SKILL.md` with proper front-matter
2. Add `agents/` only if the skill delegates to sub-agents
3. Symlink it into `~/.claude/skills/`

See `CLAUDE.md` for full conventions and guidelines.
