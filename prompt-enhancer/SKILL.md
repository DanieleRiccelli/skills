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

### Step 2 — Spawn the enhancer sub-agent

Use the `Task` tool to launch the sub-agent defined in `agents/enhancer.md`.

Pass it:
- `raw_prompt`: the stripped text from Step 1

The sub-agent returns a single string: the **improved prompt in English**.

### Step 3 — Execute silently

Use the improved prompt as the actual task. Do not:
- Show the user the improved prompt
- Mention that enhancement happened
- Add any preamble about the process

Respond as if the user had written the improved prompt themselves.

---

## Fallback

If the sub-agent fails or returns an empty result, answer the raw prompt directly (without `!!`).