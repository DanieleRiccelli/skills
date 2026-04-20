---
name: prompt-enhancer
description: >
  MANDATORY: Before doing ANYTHING else, check if the user's message ends with '!!'
  (double exclamation mark). This check must happen for every single message, without
  exception. If '!!' is detected at the end — even after whitespace or newlines — you
  MUST load this skill and follow its workflow. Do NOT proceed with any task, answer,
  or tool call until you have checked for '!!'. This skill intercepts the raw prompt,
  runs an enhancer sub-agent that rewrites it into clear structured English, and then
  executes the improved version silently. The user sees only the final answer.
  Examples that MUST trigger this skill: "fix this bug !!", "crea una funzione !!",
  "Ho bisogno di fare una modifica ... !!". No exceptions, no skipping.
---

# Prompt Enhancer Skill

## ⚠️ FIRST THING TO DO ON EVERY MESSAGE

**Before reading any file, running any tool, or writing any response:**
Check if the message ends with `!!`.

```
message.trimEnd().endsWith('!!') → LOAD THIS SKILL
```

This check has priority over everything else.

## Trigger Detection

This skill activates when the user's message **ends with `!!`** (double exclamation mark),
with or without trailing whitespace or newlines.

| Input | Trigger? |
|---|---|
| `"crea una funzione python !!"` | ✅ Yes |
| `"explain docker compose !!"` | ✅ Yes |
| `"Ho bisogno di fare una modifica ... !!"` | ✅ Yes |
| `"fix this bug !!"` | ✅ Yes |
| `"crea una funzione python"` | ❌ No |

---

## Workflow

### Step 1 — Extract the raw prompt

Strip `!!` and any surrounding whitespace from the user's message to get the **raw prompt**.

### Step 2 — Gather context

Before spawning the enhancer, collect context that will help it produce a grounded rewrite:

- **project_context**: Read the project's `CLAUDE.md` (if any) to understand the codebase, stack, and conventions.
- **conversation_summary**: If this is not the first message in the conversation, write a 2-3 sentence summary of what has been discussed/done so far. This is critical for follow-up prompts like `"continua con la prossima fase !!"` that make no sense in isolation.
- **working_directory**: The current working directory path.

### Step 3 — Spawn the enhancer sub-agent

Use the `Agent` tool to launch the sub-agent defined in `agents/enhancer.md`.

Pass it:
- `raw_prompt`: the stripped text from Step 1
- `project_context`: stack, conventions, and relevant project info (from Step 2)
- `conversation_summary`: what happened so far in this conversation (empty string if first message)
- `working_directory`: the current working directory path

The sub-agent returns a single string: the **improved prompt in English**.

### Step 4 — Decide execution mode

Assess the complexity of the improved prompt:

- **Simple** (question, one-liner, single file edit, explanation): execute directly, do NOT enter plan mode.
- **Complex** (multi-file changes, architecture decisions, multi-step workflows): call `EnterPlanMode` first to plan a structured approach.

Use your judgment — the threshold is whether the task benefits from an explicit plan.

### Step 5 — Execute silently

Use the improved prompt as the actual task. Do not:
- Show the user the improved prompt
- Mention that enhancement happened
- Add any preamble about the process

Respond as if the user had written the improved prompt themselves.

---

## Enforcement

The `!!` detection relies on Claude reading the skill description before acting on any message.
To maximize reliability:

1. The project `CLAUDE.md` must include a reminder about the mandatory check (already present).
2. The skill description is written as a **MANDATORY** pre-check instruction.
3. If you are configuring a new project that uses this skill, always add the following line
   to the project's `CLAUDE.md`:
   ```
   The `prompt-enhancer` skill must be checked **before any other action** on every user message — its trigger check is mandatory.
   ```

> **Limitation**: Claude Code hooks do not currently support a `PromptSubmit` event,
> so there is no way to enforce detection externally. If a future hook event becomes
> available, add a hook that checks for `!!` and injects a system reminder.

---

## Fallback

If the sub-agent fails or returns an empty result, answer the raw prompt directly (without `!!`).