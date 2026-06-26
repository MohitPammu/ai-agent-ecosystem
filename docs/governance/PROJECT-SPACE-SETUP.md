# Claude Project Setup — AI Agent Ecosystem

This document records the Project configuration, so it's reproducible if the Project ever needs to be rebuilt, and so the setup is version-controlled like everything else in this repo.

---

## 1. Name

**AI Agent Ecosystem**

Matches the GitHub repo name exactly — `MohitPammu/ai-agent-ecosystem` — for unambiguous cross-referencing between chat and repo.

---

## 2. Description

> Production-grade, modular multi-agent framework powering 3 data science portfolio projects (healthcare fraud detection, fantasy football intelligence, financial analysis). Built on LangGraph with a zero-additional-cost stack. Currently closing out Phase 0 (architecture specification) via a 7-card reference library and a formal cohesion-review remediation cycle, before Phase 1 (core infrastructure) implementation begins.

---

## 3. Project Memory

Claude Projects regenerate a memory summary nightly from chat activity within the Project, visible only to the account owner. This runs automatically — no setup required.

This memory is a convenience layer, not the system of record. The Master Plan's Progress Log is the authoritative, version-controlled, human-reviewable history — it lives in the GitHub repo, is visible to anyone reviewing the work, and is portable outside claude.ai. Project memory exists to give Claude continuity across sessions inside this Project specifically; it does not replace or substitute for the Master Plan.

State progress explicitly at the end of each working session (e.g., "Stage 1 complete, moving to Stage 2 next session") — this gives the nightly regeneration clean material to work with.

---

## 4. Custom Instructions

Kept short, since this text is processed on every message in the Project. Paste this into the Project's "Custom Instructions" field:

```
You are acting as the senior agentic systems architect for the AI Agent Ecosystem project — a production-grade multi-agent framework (LangGraph, zero-cost model stack) underpinning 3 data science portfolio projects.

CORE DISCIPLINE (non-negotiable):
- Spec before code. Nothing gets implemented without an approved spec (per Card 07's SDD discipline).
- One file/edit at a time. Do not batch unrelated changes — this project moves through deliberate, scrutinized stages, not bulk edits.
- Every fix is checked against docs/governance/architecture-traceability-matrix.md before being marked complete (once that file exists — Closure Plan Stage 2).
- Any reference card you edit gets its **Card Version** line bumped (e.g., 1.0 → 2.0) with a one-line changelog note, per Working Principle #7 in the Master Plan. Cards you don't touch keep their current version.
- After any change to 00-MASTER-EXECUTION-PLAN.md, give the user the exact local move+commit+push commands — never assume the repo is updated until they confirm.

SESSION START: Begin by checking 00-MASTER-EXECUTION-PLAN.md's status table and Progress Log, and PHASE-0-CLOSURE-PLAN.md's stage checklist, to establish exactly where things stand before proceeding. If the user says "resume" or seems unsure where things left off, this lookup is the answer — don't ask them to remind you.

TOKEN DISCIPLINE: Don't re-paste or re-summarize full reference cards unless actively editing them. Reference by Card/§ number. Pull a card's full content into the conversation only when that specific card is the one being worked on.

TONE: Direct, technical, no preamble/postamble padding. This is engineering work, not a tutorial.
```

---

## 5. File List (Project Knowledge)

**Default knowledge set — upload these:**

| File | Why it's in the default set |
|---|---|
| `00-MASTER-EXECUTION-PLAN.md` | The state of the entire project — read every session |
| `01-SESSION-PROMPT-GUIDE.md` | The resume/workflow prompts |
| `docs/governance/PHASE-0-CLOSURE-PLAN.md` | The active 10-stage plan currently being executed |
| `docs/governance/ecosystem-cohesion-review-rubric.md` | Needed for any future re-scoring or new review cycle |
| `docs/governance/cohesion-reviews/v1/review-reconciliation.md` | The reconciled findings driving the Closure Plan's content |
| `docs/reference/01-architecture-taxonomy.md` through `07-spec-driven-production.md` (all 7) | The architecture being edited during Closure Plan Stages 3-8 |

**Excluded from default knowledge — load only when a specific session needs them:**

- The 8 raw review-part files (`claude/part-1.md` through `chatgpt/part-4.md`) and `architecture-review-board-synthesis.md` — historical source material the reconciliation doc already distilled. Paste the specific file into chat if a Stage 1 audit question needs to trace back to original review language.
- `README.md`, `CONTRIBUTING.md`, `LICENSE` — repo hygiene files, not architecture content.
- `core/`, `skills/`, `mcp-servers/`, `projects/` — empty scaffold directories. Nothing to load until Phase 1 produces files there.

**Rule of thumb:** files every session needs (state-tracking, active plans) stay in Knowledge permanently. Files only an occasional, specific question needs get pasted into that one conversation instead.
