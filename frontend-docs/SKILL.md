---
name: frontend-docs
description: Generates functional documentation for a frontend application. Invoke with /frontend-docs.
user-invocable: true
---

# Frontend Docs Skill

Generates a complete **functional documentation** of a frontend application in **three sequential phases**, each gated by a user checkpoint.

---

## Workflow Overview

```
FASE 1: Mappatura pagine  →  ⏸️ CHECKPOINT 1 (conferma)
FASE 2: Analisi per pagina (sotto-agenti paralleli)  →  ⏸️ CHECKPOINT 2 (conferma)
FASE 3: Assemblaggio documento finale  →  pulizia file temporanei
```

---

## FASE 1 — Mappatura delle Pagine

Spawn a sub-agent using the definition in `agents/page-mapper.md`.

Pass it:
- The current working directory path

The agent will:
1. Detect the framework (Angular, React, Vue, Next.js, Nuxt, Svelte, etc.)
2. Analyze routing, navigation configs, guards/middleware
3. Write `_pages_map.json` to the project root

After the agent completes, **display the full contents of `_pages_map.json`** to the user and halt.

> ⏸️ **CHECKPOINT 1** — Show the generated map and wait for explicit confirmation before proceeding.
> The user may: add/remove pages, rename labels to match business language, group or split pages.
> Do NOT start Phase 2 until the user says to continue.

---

## FASE 2 — Analisi per Pagina

> Start this phase only after CHECKPOINT 1 confirmation.

Read `_pages_map.json` and, for each page entry, spawn a **separate sub-agent** using the definition in `agents/page-analyzer.md`.

Pass each agent:
- `page`: the full page object from the JSON (id, route, label, entry_point, children, notes)
- `framework`: the detected framework from the JSON

Agents can run in parallel. Each agent writes a `_doc_[page-id].md` file to the project root.

After all agents complete:
- Collect the one-line summary and the distinctive UI labels list returned by each agent
- Build a `labels_by_page` map: `{ [page-id]: [labels list] }`
- List all generated `_doc_*.md` files with the one-line summary for each
- Halt and wait for confirmation

> ⏸️ **CHECKPOINT 2** — Show the file list and wait for explicit confirmation before proceeding.
> The user may: exclude certain files, remove sections globally (`Azioni disponibili`, `Comportamenti rilevanti`, `Accesso`), or add specific notes/integrations.
> Do NOT start Phase 3 until the user says to continue.

---

## FASE 3 — Assemblaggio del Documento Finale

> Start this phase only after CHECKPOINT 2 confirmation.

Spawn a sub-agent using the definition in `agents/doc-assembler.md`.

Pass it:
- The list of approved `_doc_*.md` files (per user instructions at Checkpoint 2)
- Any sections to include/exclude globally
- Any additional notes from the user
- The page hierarchy from `_pages_map.json`
- The `labels_by_page` map collected at the end of Phase 2

The agent writes `DOCUMENTAZIONE_FUNZIONALE.md` to the project root.

After the agent completes:
1. Delete all `_doc_*.md` files
2. Delete `_pages_map.json`
3. Confirm to the user that `DOCUMENTAZIONE_FUNZIONALE.md` has been created

---

## Global Rules

- **Detect framework first** — exploration strategy adapts accordingly
- **No technical details** in any phase: no class names, methods, decorators, patterns, libraries
- **Document user actions and outcomes**, not what is visible on screen
- **Use real UI labels** (from templates, JSX, i18n files)
- **Italian language** throughout the final document
- **Ignore** test files, build configs, CI/CD scripts
- **Two mandatory checkpoints** — always wait for explicit user confirmation
