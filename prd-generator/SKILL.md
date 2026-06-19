---
name: prd-generator
description: Reconstructs a Product Requirements Document (PRD) in Markdown from an existing codebase. Detects the project, maps modules to functional requirements, spawns parallel sub-agents to write each PRD block, then assembles a single PRD-<project>.md. Invoke with /prd-generator.
user-invocable: true
---

# PRD Generator Skill

Reconstructs a **Product Requirements Document (PRD)** from an **existing codebase**, in **three sequential phases**, using parallel sub-agents to write each block of the document. The final deliverable is a single `PRD-<project>.md` written to the project root, formatted in the Nextadv PRD/SRS style (no header image), in **Italian**.

---

## Foundational Principle — Derivable vs Not Derivable

The skill reconstructs the PRD from code that already exists. Every block must respect three content tiers:

| Tier | Examples | Behavior |
| :---- | :---- | :---- |
| **Derivable from code** | Stack, architecture, roles, features, integrations, non-functional requirements | Extract and describe with confidence |
| **Partially inferable** | Goals, context, product purpose | Infer from README/naming/structure → **mark as hypothesis** |
| **NOT derivable** | Client, project code, version, budget, man-days, timeline, recurring costs | Insert `[DA COMPILARE]` placeholder or ask the user |

**Hard rule: never invent** monetary figures, client names, or dates. An explicit placeholder always beats a plausible-but-false value.

---

## Workflow Overview

```
FASE 1: Scan & mappatura  →  _prd_scan.json  →  ⏸️ CHECKPOINT (conferma nome, moduli, sezioni condizionali)
FASE 2: Scrittura blocchi in parallelo (batch di max 5 agenti) → un file _prd_NN_<block-id>.md per blocco
FASE 3: Assemblaggio PRD-<progetto>.md + appendice note di generazione + pulizia file temporanei
```

Optional flag: **`--with-economics`** — before Phase 3, interactively collect the economic data (scope, man-days, costs) from the user and pass it to the assembler instead of leaving the empty skeleton.

---

## FASE 0 — Direttive Utente

Whatever the user passes after `/prd-generator` is a set of **directives** that override or steer the default behavior. **Always parse them first** and thread the relevant ones through every phase. Capture them into a `user_directives` object and honor them. Typical directives:

- **Non-derivable data** provided upfront — client name, project code, version, date, status. Use them to fill the corresponding `[DA COMPILARE]` placeholders instead of leaving them empty (pass to the assembler; also reflect in the scan/checkpoint).
- **Scope focus** — e.g. "focus on the backend", "only the billing and auth modules", "ignore the admin panel". Restrict the scan, the functional-requirements modules, and the blocks accordingly.
- **Block inclusion/exclusion** — e.g. "skip the AI section", "no roadmap", "add a deployment section". Adjust `active_blocks`.
- **Formatting / language tweaks** — e.g. "shorter intro", "English output", "more detail on integrations". Pass as instructions to the relevant writer(s); default output stays Italian unless the user asks otherwise.
- **Flags** — `--with-economics` (and any future flag).

Rules for directives:
- A directive that supplies non-derivable data **takes precedence** over a placeholder — never overwrite a value the user gave you with `[DA COMPILARE]`.
- A directive **never** licenses inventing data the user did *not* provide — anything still unknown stays `[DA COMPILARE]`.
- If a directive is ambiguous or conflicts with what the scan finds, surface it at the **CHECKPOINT** rather than guessing.
- When relevant, pass the captured directives to each agent: `user_directives` to the scanner (scope/data), to each section writer (scope/formatting for its block), and to the assembler (non-derivable data, economics, language).

---

## FASE 1 — Scan & Mappatura

Spawn a sub-agent using the definition in `agents/prd-scanner.md`.

Pass it:
- `project_root`: the current working directory path

The agent will:
1. Detect project type, languages, frameworks, build system
2. Extract the inputs every PRD block needs (stack, folder architecture, roles, route/module → functional-requirement map, external integrations, env vars, AI presence, data model hints, TODO/roadmap signals)
3. Decide which **conditional blocks** apply (e.g. `ai-architecture` only if AI is detected)
4. Write `_prd_scan.json` to the project root

After the agent completes, **display the full contents of `_prd_scan.json`** to the user and halt.

> ⏸️ **CHECKPOINT** — Mostra la mappa generata e attendi conferma esplicita prima di procedere.
> L'utente può: correggere il nome/payoff del progetto, aggiungere/rimuovere/rinominare i moduli funzionali, unire o separare requisiti, includere/escludere blocchi condizionali (es. AI), correggere ruoli o integrazioni rilevate, fornire dati non derivabili (cliente, codice, versione).
> Non avviare la Fase 2 finché l'utente non conferma.

---

## FASE 2 — Scrittura dei Blocchi in Parallelo

> Start this phase only after the CHECKPOINT confirmation.

1. Read `_prd_scan.json` and build the **list of blocks to write** from the block catalog below, including conditional blocks only when flagged active in the scan.
2. For each block, spawn a sub-agent using the definition in `agents/prd-section-writer.md`.

**Block catalog** (the worker agent holds the full per-block spec; the orchestrator just assigns `block_id` and `order`):

| order | block_id | PRD section | Conditional |
| :---- | :---- | :---- | :---- |
| 01 | `introduction` | 1. Introduzione (scopo, contesto, glossario) | always |
| 02 | `stack` | 2. Stack Tecnologico | always |
| 03 | `architecture` | 3. Architettura di Sistema (data model, sicurezza/compliance, infra/devops) | always |
| 04 | `users-roles` | 4. Utenti e Ruoli | always |
| 05 | `functional-requirements` | 5. Requisiti Funzionali | always |
| 06 | `integrations` | 6. Integrazioni Esterne | always |
| 07 | `ai-architecture` | 7. Architettura AI | only if AI detected |
| 08 | `non-functional` | 8. Requisiti Non Funzionali | always |
| 09 | `current-state` | 9. Stato Attuale / Roadmap | always |
| 10 | `assumptions-constraints` | 11. Assunzioni e Vincoli | always |

> Block `10. Preventivo Economico` is **not** written by an agent — it is not derivable from code. The assembler inserts the empty skeleton (or the user-supplied data when `--with-economics` is used).

### Sharding del blocco `functional-requirements`

The functional-requirements block is the heaviest: with many modules a single agent becomes slow and shallow. Shard it across agents based on the `modules` count in `_prd_scan.json`:

- **≤ 6 modules** → a single `functional-requirements` agent (no sharding). Output: `_prd_05_functional-requirements.md`.
- **> 6 modules** → split the **ordered** `modules` list into shards of **5 modules each** (last shard may be smaller). Spawn one agent per shard.

For each shard, in module order, pass:
- `modules_subset`: the ordered slice of `modules` this shard owns
- `sub_start`: the sub-section index of this shard's **first** module = `1 + (number of modules in all preceding shards)` (shard 1 → 1, shard 2 → 6, shard 3 → 11, …)
- `is_first_shard`: `true` only for the first shard — only it emits the block's `# **N. Requisiti Funzionali**` H1 heading; the others emit just their `## **N.<m>**` sub-sections
- `shard_index`: two-digit index (`01`, `02`, …) for the output filename

Each shard writes `_prd_05_functional-requirements_<shard_index>.md`. Because `sub_start` is precomputed and contiguous, sub-section numbers never collide across shards (shard 1 → `N.1..N.5`, shard 2 → `N.6..N.10`, …). All other blocks are never sharded.

**Parallelism constraint — max 5 agents at a time**:
- Build the full agent list = all active blocks, with `functional-requirements` expanded into its shards (when sharded). Each shard counts as one agent.
- If the list has ≤ 5 agents, launch them all in parallel in a single tool-call block.
- If it has > 5 agents, run sequential **batches of 5**: launch 5 in parallel, wait for ALL to complete, then launch the next 5. Do NOT launch a new batch before the previous one finishes.

Pass each agent:
- `block_id`: the block identifier from the catalog
- `order`: the two-digit order prefix (for the output filename)
- `project_root`: absolute path to the project
- `scan_path`: path to `_prd_scan.json`
- For `functional-requirements` shards also pass: `modules_subset`, `sub_start`, `is_first_shard`, `shard_index` (see the sharding rules above)

Each agent reads its block spec from its own catalog, reads only the relevant source files, and writes `_prd_<order>_<block-id>.md` to the project root (e.g. `_prd_05_functional-requirements.md`), or `_prd_05_functional-requirements_<shard_index>.md` for a shard.

After all batches complete, collect from each agent the one-line summary and the output file path. Proceed to Phase 3.

---

## FASE 3 — Assemblaggio del PRD

Spawn a sub-agent using the definition in `agents/prd-assembler.md`.

Pass it:
- `block_files`: list of the `_prd_<order>_<block-id>.md` files produced in Phase 2 (in order)
- `scan_path`: path to `_prd_scan.json`
- `project_root`: absolute path to the project
- `economics`: the user-supplied economic data when `--with-economics` was used; otherwise `null`

The agent:
1. Reads `_prd_scan.json` for project metadata and the active block list
2. Builds the header block (no image) with `[DA COMPILARE]` placeholders for non-derivable fields
3. Concatenates the block files in skeleton order, **renumbering sections** to account for conditional blocks (e.g. when AI is absent, what would be §7 shifts up)
4. Inserts the `Preventivo Economico` section (empty skeleton or user data)
5. Appends an appendix **"Note di generazione"** listing inferred (hypothesis) sections and every `[DA COMPILARE]` item to validate manually
6. Writes `PRD-<project-name>.md` to the project root

After the agent completes:
1. Delete all `_prd_*.md` block files — they are intermediate
2. Delete `_prd_scan.json` — it is intermediate
3. **Keep** `PRD-<project-name>.md` — the only deliverable
4. Confirm to the user the file written and echo the "Note di generazione" list (sections inferred + fields to fill manually)

---

## Global Rules

- **Output**: a single `PRD-<project-name>.md` at the project root; all `_prd_*.md` and `_prd_scan.json` are intermediate and removed at the end
- **Never invent** client names, project codes, versions, dates, or any economic/timeline figure — use `[DA COMPILARE]`
- **Mark inferences**: goals/context/purpose inferred from README/naming must be flagged as hypotheses, and listed in the final "Note di generazione"
- **Formatting conventions (constant)**:
  - Numbered, bold headings: `# **1. Introduzione**`, `## **2.1 ...**`, `### **2.1.1 ...**`
  - Standard Markdown tables with alignment row (`| :---- | :---- |`)
  - Bulleted lists with a **blank line between items**
  - Glossary, Stack, Roles → always rendered as **tables**
  - Amounts (if any) as `€ 80.000`, totals in **bold**
  - **No image/logo block** at the top
  - Tone: descriptive and technical — prose for introductions, bullets for features
- **Conditional sections**: the `ai-architecture` block is included only when AI usage is detected; the assembler renumbers accordingly
- **Read-only**: agents must never modify source code
- **One mandatory checkpoint** — always wait for explicit user confirmation before Phase 2
- **Parallel batching**: maximum 5 agents in parallel; never exceed this limit
- **Prose language**: Italian for all descriptive text; keep code, paths, env-var names, library names, and technical keywords in original form
