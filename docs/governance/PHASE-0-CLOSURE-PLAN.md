# Phase 0 Closure Plan
## A Sequential, Self-Reinforcing Remediation Plan for Cards 01-07 — Category A + B Items

**Purpose:** This is not a list of independent patches. Each stage below is ordered so that it produces an artifact the *next* stage depends on — fixing things in the wrong order risks "fixing" one card while leaving its mirror image stale elsewhere, which is exactly the failure mode this review process exists to catch. No stage begins until the prior stage's exit criteria are met.

**Scope:** Category A (Must Do) + Category B (Should Do) items from the Review Board Synthesis, reconciled against both independent cohesion reviews. Category C (post-implementation verification execution) and Category D (rejected recommendations) are explicitly out of scope here — see `REVIEW-RECONCILIATION.md` for why each was accepted, rejected, or downgraded.

**Governing principle for every stage:** no fix is applied in isolation. Every fix is checked against the full set of cards before being marked complete, using the Traceability Matrix (produced in Stage 2) as the verification tool — not memory, not assumption.

---

## Stage 0 — Freeze the Input Set

Before touching anything, lock down exactly what's being fixed against.

**Actions:**
1. Confirm Cards 01-07 as currently uploaded are the canonical version under remediation (matching what both independent reviews assessed).
2. Confirm the rubric (`ecosystem-cohesion-review-rubric.md`) remains frozen at v1.0 — this plan fixes the architecture, not the rubric.

**Exit criteria:** No ambiguity about which file versions this plan operates on. (This stage is procedural — confirm and move on.)

---

## Stage 1 — Full Forward-Reference Audit (discharges Synthesis A3)

**Why this goes first:** every subsequent fix in this plan depends on knowing the *complete* set of places a concept is referenced. Fixing Card 07's Tier 3/4 error without first knowing every other place "Tier" is referenced risks fixing one occurrence while missing a sibling error elsewhere that neither review happened to catch.

**Actions:**
1. For every "See Card X §Y" / "per Card X" / forward-reference phrase across all 7 cards, verify three things: (a) the cited section still exists at that number, (b) the cited section still says what the citing card claims it says, (c) ownership still aligns with what's claimed.
2. Build a raw list: `[Citing Card/Section] → [Cited Card/Section] → [Claim being made] → [Verified: Yes/No/Stale]`.
3. Flag every "No" or "Stale" result — this list becomes the authoritative defect list for Stage 3, superseding the informal findings from both independent reviews (which were sampling-based, not exhaustive).

**Known findings to verify as part of this pass (not an exhaustive list — the audit may surface more):**
- Card 07 §6 citing Card 06 §3 (Tier 3 vs Tier 4 — confirmed stale by both reviews' source material)
- Card 01 §8 citing Card 05's observability schema (evaluation score — confirmed stale)
- Card 06 §26 citing Card 03 §2-3 for "source boundary preservation" (confirmed the cited mechanism doesn't exist in Card 03)

**Exit criteria:** A complete, verified table of every forward reference in the ecosystem, with every "Stale" entry flagged. This table is the direct input to Stage 2.

---

## Stage 2 — Build the Architecture Traceability Matrix (discharges Synthesis B4)

**Why this goes second, before any fix is applied:** this is the single highest-leverage artifact identified across both reviews and the reconciliation — it converts "did I remember to check everywhere this is mentioned" from a manual, error-prone process into a lookup table. Building it *before* fixing anything means every fix in Stage 3 can be checked against a complete map, not partial memory.

**Actions:**
1. Create `docs/governance/architecture-traceability-matrix.md`.
2. For every concept that has exactly one canonical owner (per Dimension 1's findings — Risk Tiers, HITL, the Contract pattern, Memory types, the Failure Taxonomy, Observability fields, etc.), populate a row: `Concept | Canonical Owner (Card §) | Referencing Cards/Sections | Last Verified`.
3. Seed this directly from Stage 1's audit table — every reference relationship already discovered there becomes a matrix row.

**Exit criteria:** A populated matrix covering every cross-card concept reference identified in Stage 1. This is now the tool used for every remaining stage — no further fix is applied without checking it against this matrix first and updating the matrix's "Last Verified" column after.

---

## Stage 3 — Resolve Factual Contradictions and Stale Definitions (discharges Synthesis A1 + A2)

**Why this goes third, only after Stages 1-2:** fixing a contradiction without the matrix risks fixing the citing card but not the cited card's own internal consistency, or fixing one of two stale copies while missing a third nobody had flagged yet.

**Actions, each checked against the Stage 2 matrix before being marked done:**

1. **Card 07 §6 Tier contradiction:** Correct to read *"Tier 4+ per Card 06 §3 (or Tier 3 specifically where Card 06 §14's regulated-project override applies)"* — matching Card 06's actual table and its own Project 1/3 exception. Update the matrix's Risk Tier row to confirm Card 07 now correctly references Card 06 §3.

2. **Card 01 §8 stale observability field:** Replace the restated field list with a pure pointer: *"See Card 05 §5 for the current observability field schema — not restated here to avoid drift."* This directly implements the Synthesis's A1 principle (replace duplicated summaries with "See Card X §Y," maintain one canonical definition). Update the matrix's Observability Schema row accordingly.

3. **Sweep for any other stale/duplicated definitions surfaced by Stage 1's exhaustive audit** (not just the two known findings) — apply the same "replace restatement with pointer" pattern to each.

**Exit criteria:** Every entry flagged "Stale" or "No" in Stage 1's audit table is now resolved, and the Traceability Matrix reflects the corrected state. Re-run a targeted re-check: search all 7 cards for the word "Tier" and confirm every numeric reference is now consistent with Card 06 §3/§14 as the single source of truth.

---

## Stage 4 — Document the Reconciled Anti-Bloat Decisions (discharges the reconciliation's Tool/Skill Registry and Circuit Breaker findings)

**Why this goes fourth, after contradictions are resolved but before new files are drafted:** these are decisions, not defects — but per the rubric's own "Alternative Architecture Considered" mechanism, undocumented intentional decisions are exactly what *looks* like an open question to a future maintainer or the next independent review cycle. Documenting them now, while the reasoning is fresh, prevents this exact question from being re-litigated in a future review.

**Actions:**
1. Add a brief **"Alternative Architecture Considered"** note to Card 02 §5 and Card 04 §6 (Tool Registry / Skill Registry sections respectively): note the structural field-schema similarity, and the reconciled reasoning for keeping them separate (different lifecycle/governance semantics — executable capability vs. procedural knowledge — per the Synthesis's D1 rebuttal, accepted in `REVIEW-RECONCILIATION.md`).
2. Add a brief **"Alternative Architecture Considered"** note to Card 06 §11 (Circuit Breaker): note that merging into the Policy Server was considered and rejected because Policy Server *decides* and Circuit Breaker *acts* — the same Decide/Act split already established in Card 01 §2, not a redundant pair.
3. Relabel Card 03 §4's LangGraph-specific content explicitly as **`Implementation Note (LangGraph):`** per the Synthesis's D3 resolution — preserving the content, just making the framework-coupling visible and scoped rather than blended into architectural guidance.

**Exit criteria:** All three reconciled decisions are now documented in the cards themselves, not just in the reconciliation memo — a future reader of Cards 02/04/06 alone (without needing the review history) understands why these design choices were made.

---

## Stage 5 — Operationalize the Untrusted-Context-Boundary Mechanism (discharges the Missing Abstraction finding)

**Why this goes fifth, after Stage 4:** this is a genuine missing mechanism (Card 06 §26 imposes a requirement Card 03 never operationalizes), and fixing it requires the corrected, de-duplicated state of both cards from Stages 3-4 as a stable foundation — adding new content to a card that still has unresolved stale references elsewhere risks compounding the inconsistency.

**Actions:**
1. Add a new subsection to Card 03 (e.g., §12 — "Source Boundary Preservation, Closing Card 06 §26's Requirement") specifying, at minimum: a source-labeling convention for context entries (e.g., a `source_type` tag: system / tool_output / retrieved / memory / a2a_message / user_upload), and an explicit statement that the Harness must never let a non-`system` source override system or policy instructions during context assembly.
2. Update Card 06 §26 to point to this new Card 03 subsection by exact number, replacing the current vague "Card 03 §2-3" reference (itself a Stage 1 finding) with a precise, now-correct citation.
3. Update the Traceability Matrix with this new concept row.

**Exit criteria:** The untrusted-context-boundary requirement has an actual implementable mechanism, not just an assertion that one should exist. Card 06's reference to it is now accurate.

---

## Stage 6 — Close the Blast-Radius Gaps (extends Card 06 §20's proven pattern to the components found lacking it)

**Why this goes sixth:** this reuses a pattern (Card 06 §20's fail-closed-by-tier table) that is *already proven correct* by both reviews — applying a known-good pattern to the remaining gaps is lower-risk than inventing new mechanisms, and it should happen after the foundational corrections (Stages 3-5) so it's extending a now-fully-consistent card, not building on top of unresolved errors.

**Actions:**
1. Add a brief blast-radius/degradation statement for **Memory** (what happens to the Harness if `core/memory/`'s backend is unavailable — degraded mode vs. hard-fail) to Card 03.
2. Add a brief statement for **whole-registry-file failure** (distinct from the already-covered single-bad-entry case) to Card 02 §5/§16 and Card 04 §6.
3. Add a brief statement for **Evaluation-service unavailability** (does the Harness block on it or proceed and backfill) to Card 05.
4. Add a brief statement for **Harness/Orchestrator involuntary crash recovery** (distinct from the deliberate `Stop` outcome) to Card 01 §2.

**Exit criteria:** Every major component named in the rubric's Dimension 13 now has an explicit, stated blast-radius/degradation behavior — no more binary unknowns.

---

## Stage 7 — Produce the Missing Foundational Artifacts (discharges Synthesis A4)

**Why this goes seventh, deliberately late, not first:** this is the most consequential ordering decision in this entire plan. AGENTS.md, the Agent Contract template, and the model-routing-table.md are all *downstream consumers* of the concepts fixed in Stages 3-6 (corrected Tier references, the operationalized context-boundary mechanism, the documented anti-bloat decisions, the blast-radius statements). **Drafting these files before the corrections above would risk baking the same stale Tier reference, the same unoperationalized boundary requirement, and the same undocumented design decisions directly into brand-new "foundational" files** — turning today's fixable inconsistency into tomorrow's freshly-authored one.

**Actions, each drafted against the now-corrected, now-fully-cross-referenced card set:**
1. **`docs/architecture/agent-contract-template.md`** (Phase 0, file 0.12) — the full field-by-field template per Card 01 §1's specification (mission, inputs/outputs, tools, memory access, success criteria, stop conditions, escalation rules, evaluation metrics, HITL requirements).
2. **`AGENTS.md`** (Phase 0, file 0.10) — the master ecosystem specification, written as a signed instruction artifact per Card 06 §15's requirement, referencing the now-corrected Tier table and the now-operationalized context-boundary mechanism directly.
3. **`docs/architecture/model-routing-table.md`** (Phase 0, file 0.9) — populated with the provider metadata schema from Card 07 §11, using the now-documented classification-aware routing rules from Card 06 §26.
4. **`docs/architecture/tech-stack.md`** (Phase 0, file 0.8) — final stack decisions, including the explicit LangGraph implementation-note labeling convention established in Stage 4.

**Exit criteria:** All four files exist with real content, each one internally consistent with the corrected Cards 01-07, not just structurally present.

---

## Stage 8 — Should-Do Documentation Improvements (discharges Synthesis B1 + B3)

**Why this goes eighth, after the foundational files exist:** these are navigability/governance-history improvements that are most valuable once there's a complete, correct picture to navigate and a documented decision history to preserve — doing this earlier would mean indexing/recording a still-incomplete state.

**Actions:**
1. **Card 06 topic index** (Synthesis B1): add a short index at the top of Card 06 mapping related-but-separated sections (e.g., "Registries: §7, §16 · Risk Tiers: §3, §14 · Policy Server: §4, §20, §21").
2. **Architecture Decision Records** (Synthesis B3): create `docs/governance/adr/`, and write at minimum: ADR-001 (Runtime Stack ordering), ADR-002 (Policy Server centralization), ADR-003 (the Contract pattern), ADR-004 (Skills as procedural memory), ADR-005 (Evaluation/Security separation), ADR-006 (Tool Registry vs. Skill Registry — capturing Stage 4's reconciled decision formally), ADR-007 (Circuit Breaker vs. Policy Server — capturing Stage 4's other reconciled decision formally).

**Exit criteria:** Card 06 is navigable without a full read-through; every major architectural decision this entire review process surfaced has a permanent, citable record independent of any single chat history.

---

## Stage 9 — Draft the Architecture Verification Specification (discharges Synthesis B2 — planning only, not execution)

**Why this goes ninth, last among the content-producing stages:** this document's scenario list should be drawn from the *final, corrected* architecture — drafting it earlier risks writing test scenarios against requirements that were about to change in Stages 3-6.

**Actions:**
1. Create `docs/governance/architecture-verification-specification.md`.
2. Populate it with the scenario catalog already compiled across this review cycle (Policy Server unavailable/compromised, Circuit Breaker activation per scope, the newly-defined Memory/registry-file/Evaluation-service/Harness blast-radius scenarios from Stage 6, retry exhaustion, context overflow, rollback, concurrent orchestration, delegation failure), each with a stated expected-result/pass-criterion per Dimension 14's requirement.
3. Explicitly label this as a **planning artifact only** — no scenario in it is executed until Phase 1 infrastructure exists. This document is written now so it doesn't have to be reconstructed from scratch later, but its *execution* remains Category C, correctly out of scope for Phase 0 closure.

**Exit criteria:** A complete, citable verification plan exists, ready to execute the moment Phase 1 produces something to test against.

---

## Stage 10 — Final Confirmation Review (re-running the rubric against the now-corrected set)

**Why this is last:** this is the actual gate. Nothing in this plan is "done" until re-scored.

**Actions:**
1. Re-run the Ecosystem Cohesion Review Rubric v1.0 against the corrected Cards 01-07 plus the four new Phase 0 files.
2. Specifically re-score Dimensions 2, 4, 5, 7, 9, 10, 11, and 13 — the dimensions this review cycle found issues in — using the Traceability Matrix (Stage 2) as a fact-checking tool during re-scoring, not impression alone.
3. Confirm: zero remaining factual contradictions findable via the matrix; all four foundational files present and internally consistent; every Stage 4-6 documentation addition is reflected in the cards.

**Exit criteria for closing Phase 0 entirely:** A Freeze-Ready verdict, with evidence-cited scores for every dimension — not an assumption that "the punch list was completed, so it must be fine now."

---

## Summary Table — Stage-to-Deliverable Map

| Stage | Deliverable | Discharges |
|---|---|---|
| 1 | Verified forward-reference audit table | Synthesis A3 |
| 2 | `architecture-traceability-matrix.md` | Synthesis B4 |
| 3 | Corrected Cards 01, 07 (+ any others Stage 1 surfaces) | Synthesis A1, A2 |
| 4 | Documented Alternative-Architecture notes in Cards 02, 04, 06; relabeled Card 03 | Reconciliation's D1/D2/D3 resolutions |
| 5 | New Card 03 subsection + corrected Card 06 §26 citation | Missing Abstraction finding |
| 6 | Blast-radius statements added to Cards 01, 02, 03, 04, 05 | Dimension 13 gaps |
| 7 | `agent-contract-template.md`, `AGENTS.md`, `model-routing-table.md`, `tech-stack.md` | Synthesis A4 |
| 8 | Card 06 index; `docs/governance/adr/` (7 ADRs) | Synthesis B1, B3 |
| 9 | `architecture-verification-specification.md` | Synthesis B2 |
| 10 | Re-scored rubric, Freeze-Ready verdict | The actual gate |

**Nothing in this plan is optional if the goal is genuinely bulletproof, evidence-based closure — but the order above is what prevents Stage 7's new files from inheriting Stage 3's bugs, and what prevents Stage 8's documentation from describing a still-incomplete picture.**
