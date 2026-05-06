---
name: prompt-enhancer
description: >
  MANDATORY: Before acting on ANY user message, load this skill and run its
  Step 0 gate check. The skill intercepts every user message, runs a lightweight
  classifier that decides SKIP (pass through unchanged) or ENHANCE (full rewrite),
  and applies enhancement only when a real task is detected. When ENHANCE is
  selected, the skill rewrites the prompt into a high-quality structured English
  prompt using Anthropic prompt engineering best practices (role, XML tags,
  chain-of-thought, output format, positive framing, acceptance criteria), then
  executes it silently — answering the user in their original language. This
  pre-check is mandatory and runs on every single message, without exception.
---

# Prompt Enhancer Skill

## ⚠️ FIRST THING TO DO ON EVERY MESSAGE

**Before reading any file, running any tool, or writing any response:** run the Step 0 gate. The gate decides whether the message goes through enhancement or passes through untouched. This check has priority over everything else.

---

## Workflow Overview

```
user message → [0] gate check → SKIP: answer directly
                              ↘ ENHANCE: [1] capture → [2] detect language →
                                         [3] gather context → [4] spawn enhancer →
                                         [5] handle response → [6] execute silently
```

---

## Step 0 — Gate Check

Classify the incoming message as **SKIP** or **ENHANCE**. This must be the first thing you do.

### SKIP — pass the message through with zero enhancement

The message matches any of these:

- **Pure confirmation or acknowledgment**: "ok", "sì", "perfetto", "grazie", "capito", "esatto", "va bene", "certo", "no", "yes", "sure", "thanks", "got it"
- **Single word or very short** (under 6 words) with **no task intent**
- **A direct answer** to a question Claude just asked (e.g., "il file è in src/components", "use Postgres", "yes, on Linux")
- **Emotional or social**: "ottimo lavoro", "sei forte", "non importa", "nice", "great work"
- **Continuing a Claude-led flow**: "continua", "vai avanti", "procedi", "go on", "next" — when the previous turn was Claude offering to continue. (If the previous turn did not set up a continuation, treat it as ambiguous → ENHANCE.)

### ENHANCE — run the full workflow

The message matches any of these:

- Contains **task verbs**: *crea, scrivi, implementa, aggiungi, rimuovi, fix, correggi, refactora, migra, ottimizza, analizza, spiega, confronta, build, create, write, implement, add, remove, fix, refactor, migrate, optimize, analyze, explain, compare* — or equivalents in any language
- References a **file, component, function, endpoint, database, API, or technical artifact** by name
- Is a **question requiring multi-step reasoning or technical depth** (more than a quick factual lookup)
- Contains a **description of a problem or unexpected behavior**
- Is **longer than 15 words** and has clear task intent
- Asks for **output to be created** (code, document, email, prompt, plan, etc.)

### AMBIGUOUS

Short messages that *might* be a real task → default to **ENHANCE** with minimal scaffolding. It is better to enhance a simple message than to skip a real task.

### Action

- **SKIP** → execute the message directly with zero enhancement, zero overhead. Do not spawn the sub-agent. Do not add scaffolding. Do not change languages.
- **ENHANCE** → proceed with Steps 1 through 6 below.

---

## Step 1 — Capture the raw prompt

Keep the message verbatim in memory — you'll need it if the enhancer fails.

## Step 2 — Detect the user's language

Identify the language of the raw prompt (Italian, English, Spanish, etc.). The **enhanced prompt will be in English**, but the **final answer to the user must be in their detected language**. This is non-negotiable: the user does not see the rewriting step, so a language switch in your reply would feel broken.

## Step 3 — Gather context

Collect inputs that ground the rewrite:

- **project_context**: contents of project `CLAUDE.md` if present (stack, conventions). Empty string if none.
- **conversation_summary**: 2–3 sentences of what's been discussed/done so far. Critical for follow-ups like *"continua con la prossima fase"*. Empty string if first message.
- **working_directory**: the current cwd path.
- **detected_language**: from Step 2.

## Step 4 — Spawn the enhancer sub-agent

Launch the agent at `agents/enhancer.md` with these inputs:

- `raw_prompt`
- `project_context`
- `conversation_summary`
- `working_directory`
- `detected_language`

The sub-agent classifies the prompt type, applies the appropriate rewriting strategy, and returns one of:

1. **An improved prompt** (the normal case) — a self-contained English prompt, possibly with XML tags, ending with a directive to answer in `detected_language`.
2. **A `BLOCKED:` response** — when the prompt has contradictions, critical missing information only the user can supply, or fundamentally ambiguous targets. Format: `BLOCKED: <one-line reason> | ASK: <one short clarifying question to the user, in detected_language>`.
3. **A `META:` response** — when the prompt is itself a meta-prompt (e.g., *"write a system prompt for a chatbot that…"*). Format: `META: <the rewritten prompt-as-artifact>`. The user wants a *prompt as output*, not the prompt executed.

## Step 5 — Handle the enhancer's response

- **`BLOCKED:`** → Do not execute. Reply to the user in `detected_language` with only the clarifying question. Do not mention the enhancer.
- **`META:`** → Treat the enhancer's output as the *content of the answer*. Present the rewritten prompt-artifact to the user in `detected_language` (with the artifact itself in whatever language is appropriate for its target system, usually English). Do not execute it.
- **Normal** → Use the improved prompt as the actual task. Plan mode only if the task genuinely benefits from a structured plan (multi-file refactors, architectural changes). One-shot edits, questions, and explanations skip plan mode.

## Step 6 — Execute silently in the user's language

Respond as if the user had written the improved prompt themselves. Do **not**:
- Show the user the improved prompt
- Mention that enhancement happened
- Add preamble about the process

Do:
- Answer in `detected_language`
- If the enhancer flagged an interpretation call in its output (a `# INTERPRETATION:` line), surface that single line at the end of your reply, translated into `detected_language`, as an italic note. Example: *"Interpretato come: …"*. This is the only allowed visibility into the enhancement.

---

## Best practices the enhancer applies

The sub-agent owns the actual rewriting. The orchestration above is what *this* skill enforces. The enhancer's checklist is documented in `agents/enhancer.md` and includes:

- Prompt-type classification (technical-code, debug, creative, agentic/Claude-Code, meta, conversational, analytical)
- Role/persona scaffolding when useful
- XML tags (`<task>`, `<context>`, `<constraints>`, `<output-format>`) for multi-concern prompts
- Chain-of-thought trigger for analytical/debug prompts
- Acceptance criteria for code/agentic prompts
- Positive framing (do X, not don't-do-Y)
- Few-shot examples when the desired output shape is unusual
- Native-quality translation (idiomatic, not literal)
- The Cardinal Rule: **clarify, don't invent**

---

## Fallback

- If the **gate check itself** fails or you can't confidently classify the message, default to **ENHANCE**. It is safer to over-enhance a chatty message than to skip a real task.
- If the **sub-agent** fails, returns empty, or errors:
  1. Answer the raw prompt directly.
  2. Reply in `detected_language`.
  3. Do not mention the failure.
