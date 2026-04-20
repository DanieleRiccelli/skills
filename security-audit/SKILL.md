---
name: security-audit
description: Performs a comprehensive security audit of the entire codebase by dividing it into sections and analyzing each in parallel. Invoke with /security-audit.
user-invocable: true
---

# Security Audit Skill

Performs a **full security audit** of the codebase through **three sequential phases**, using parallel sub-agents for efficient analysis. Per-section reports and a consolidated executive report are written to `analysis/` at the project root.

---

## Workflow Overview

```
FASE 1: Mappatura sezioni  →  ⏸️ CHECKPOINT (conferma/modifica divisione)
FASE 2: Audit parallelo (batch di max 5 agenti)
FASE 3: Consolidamento report finale
```

---

## FASE 1 — Mappatura delle Sezioni

Spawn a sub-agent using the definition in `agents/section-mapper.md`.

Pass it:
- The current working directory path (`project_root`)

The agent will:
1. Detect the project type (frontend / backend / fullstack / library / CLI / mobile), languages, and frameworks
2. Choose the most meaningful division criterion (**by feature/domain**, **by page/route**, **by layer**, **by module**, or **mixed**)
3. Write `_sections_map.json` to the project root

After the agent completes, **display the full contents of `_sections_map.json`** to the user and halt.

> ⏸️ **CHECKPOINT** — Mostra la divisione proposta e attendi conferma esplicita.
> L'utente può: aggiungere/rimuovere sezioni, unire o separare sezioni, modificare i path inclusi, cambiare il criterio di divisione, regolare la criticità.
> Non avviare la Fase 2 finché l'utente non conferma.

---

## FASE 2 — Audit Parallelo delle Sezioni

> Start this phase only after the CHECKPOINT confirmation.

1. Ensure the `analysis/` directory exists at the project root — create it if missing.
2. Read `_sections_map.json` and get the list of sections.
3. For each section, spawn a **sub-agent** using the definition in `agents/section-auditor.md`.

**Parallelism constraint — max 5 agents at a time**:
- If there are ≤ 5 sections, launch them all in parallel in a single tool-call block.
- If there are > 5 sections, run sequential **batches of 5**: launch 5 in parallel, wait for ALL to complete, then launch the next 5. Continue until every section is processed. Do NOT launch a new batch before the previous one finishes.

Pass each agent:
- `section`: the full section object from the JSON (id, label, paths, description, criticality)
- `project_root`: absolute path to the project
- `project_type`: detected type from the map
- `languages`: detected languages
- `frameworks`: detected frameworks

Each agent writes `analysis/<section-id>.md`.

After all batches complete, collect from each agent the one-line severity summary (e.g. `"Critical: 2, High: 4, Medium: 3, Low: 1, Info: 0"`) and the output file path. Proceed to Phase 3.

---

## FASE 3 — Consolidamento Report Finale

Spawn a sub-agent using the definition in `agents/report-consolidator.md`.

Pass it:
- `section_reports`: list of the `analysis/<section-id>.md` file paths produced in Phase 2
- `sections_map_path`: path to `_sections_map.json`
- `project_root`: absolute path to the project

The agent:
1. Reads all per-section reports
2. Deduplicates and correlates cross-cutting issues (same vulnerability in multiple sections → single consolidated finding)
3. Identifies systemic patterns (classes of issues affecting > 3 sections)
4. Writes `analysis/SECURITY_AUDIT_REPORT.md` with executive summary, systemic issues, prioritized findings, and recommendations

After the agent completes:
1. Delete `_sections_map.json` — it is an intermediate file
2. **Keep** every `analysis/<section-id>.md` and `analysis/SECURITY_AUDIT_REPORT.md` — these are the final deliverables
3. Confirm to the user the files written in `analysis/` and the total issues by severity

---

## Global Rules

- **Output directory**: every report lives in `analysis/` at the project root
- **Per-section reports are preserved** — they are final deliverables, not intermediate files; only `_sections_map.json` is cleaned up at the end
- **Parallel batching**: maximum 5 agents in parallel; never exceed this limit
- **OWASP framing**: where applicable, map each finding to OWASP Top 10 (Web) or OWASP Mobile Top 10
- **Severity levels**: Critical / High / Medium / Low / Informational
- **Confidence levels**: Confirmed / Likely / Possible
- **Every finding includes**: location (`file:line`), description, impact, remediation, severity, confidence, OWASP category when applicable
- **Low-noise policy**: when in doubt, downgrade severity or confidence — false positives erode trust
- **Read-only audit**: agents must never modify source code
- **One mandatory checkpoint** — always wait for explicit user confirmation before Phase 2
- **Prose language**: Italian for descriptive text (summary, description, impact, remediation); keep code, paths, OWASP identifiers, and technical keywords in original form
