# Ecosystem Cohesion Review — Stage 10 Internal Re-Score

**Date:** 2026-06
**Reviewer:** Internal (architectural advisor — participated in all Closure Plan stages)
**Reviewer Disclosure:** Internal reviewer — not a cold-read. This review carries lower evidentiary independence than the concurrent external review and should be weighted accordingly in the reconciliation pass.
**Architecture Version:** Cards 01-07, 2026-06 Freeze Candidate
- Card 01: v4.0
- Card 02: v3.0
- Card 03: v5.0
- Card 04: v3.0
- Card 05: v2.0
- Card 06: v4.0
- Card 07: v2.0

Plus all Phase 0 governance artifacts: AGENTS.md (v1.0), agent-contract-template.md (v2.0), model-routing-table.md (v1.0), tech-stack.md (v1.0), architecture-traceability-matrix.md (current), architecture-verification-specification.md (v1.0), ADR-001 through ADR-007.

---

## Review Assumptions

1. LangGraph remains the orchestration framework
2. Zero-cost model stack remains the initial deployment assumption
3. Public/synthetic data only for all three portfolio projects
4. Master Execution Plan remains the authoritative build sequence
5. Cards 01-07 are evaluated as a unified specification, not 7 independent documents

---

## Dimension 1 — Ownership Clarity

**Score: 9**
**Confidence:** High — the Traceability Matrix provides direct verification of every ownership claim across all 7 cards.
**Evidence:** Strong — Card 01 §8 now reads "see Card 05 §5 for the current observability field schema; not restated here to avoid drift" (Stage 3 fix); Card 07 §11 reads "lives in `docs/architecture/model-routing-table.md` §4 — not restated here" (Stage 7 follow-up). The Architecture Traceability Matrix confirms single ownership for every tracked concept — Risk Tiers canonical owner: Card 06 §3; Memory Types: Card 03 §5; Evaluator Hierarchy: Card 05 §4; Source Boundary Preservation: Card 03 §12. AGENTS.md §2 Repository Philosophy states "when two artifacts appear to disagree, the one whose concept it actually owns — per the Architecture Traceability Matrix's single-owner discipline — governs."
**Risk if Ignored:** N/A — no material issue found.
**Recommended Action:** None. Minor note for debt register: Agent Contract Template's `related_adr` field is present but no filled contract exists yet — correctly a Phase 1 task, not a Phase 0 ownership problem.

---

## Dimension 2 — Dependency Flow

**Score: 10**
**Confidence:** High — ADR-001 now provides explicit, citable evidence for the ordering; AGENTS.md §4 reiterates it as runtime instruction.
**Evidence:** Strong — ADR-001 ("Runtime Stack Ordering") states: "Security & Governance sits at the base. Every layer above it is governed by Security, never the reverse" — the rationale section explains precisely: "the stack ordering is the single fact that makes every 'X is Card 06's responsibility, not ours' statement in the card set correct." AGENTS.md §4 lists the 7-layer stack in order. Card 06's header reads "sits at the base of the Runtime Stack — Security governs every layer above it (Tools, Memory, State, Harness, Contract), not the reverse." The dependency flow from Card 02 §5's binding-rule (Tool Registry → Policy Server for assignment authority), Card 03 §11 (Memory Contracts → Card 06 §8 for authorization), and Card 04 §6 (Skill Registry assignment → Card 06 §4) all flow in one direction only — upward to Security, never downward.
**Risk if Ignored:** N/A.
**Recommended Action:** None. This was a 7/10 in the v1 review — ADR-001 closes the gap definitively.

---

## Dimension 3 — Circular References

**Score: 10**
**Confidence:** High — the Stage 1 audit was specifically designed to catch circular reference patterns and found none.
**Evidence:** Strong — Stage 1's exhaustive forward-reference audit traversed every "See Card X §Y" / "per Card X" reference across all 7 cards and found no circular dependencies. The Architecture Traceability Matrix's single-owner discipline structurally prevents circularity: if Card A owns concept X and Card B cites X, Card B can never simultaneously own X, and Card A never needs to cite Card B for X's definition. The stack ordering (ADR-001) also prevents circular governance: Security governs Tools; Tools cannot govern Security.
**Risk if Ignored:** N/A.
**Recommended Action:** None.

---

## Dimension 4 — Duplication

**Score: 9**
**Confidence:** High — three confirmed duplication instances were identified and corrected; the Traceability Matrix provides ongoing enforcement.
**Evidence:** Strong — Card 01 §8 previously restated Card 05 §5's observability field list; Stage 3 replaced it with a pointer. Card 07 §11 previously restated 6 provider-metadata fields that model-routing-table.md now owns (21 fields); Stage 7 follow-up replaced it with a pointer. Card 06 §26 previously cited "Card 03 §2-3" for a mechanism that lived nowhere; Stage 5 created Card 03 §12 as the actual canonical location. The "Alternative Architecture Considered" notes in Cards 02, 04, 06 (Stage 4) documented the same decision from two sides without duplicating each other — each card's note cites the other card's perspective rather than restating it. AGENTS.md §14 explicitly states: "if an edit to this file finds itself writing out a tier table, a field schema, or a memory-type list instead of citing one, stop."
**Risk if Ignored:** N/A for confirmed instances.
**Recommended Action:** None blocking. One residual observation: Card 03's Direct Implications section contains a LangGraph note ("LangGraph's state-as-session model is used directly") that partially overlaps with Card 03 §4's Implementation Note. Not harmful duplication — the Direct Implications bullet is a build decision, §4 is architectural explanation — but worth noting for the future editorial pass already in the Deferred list.

---

## Dimension 5 — Missing Abstractions

**Score: 10**
**Confidence:** High — all three missing abstractions identified in v1 are now present and internally consistent.
**Evidence:** Strong — (1) Card 03 §12 ("Source Boundary Preservation") now exists as the operationalized mechanism Card 06 §26 requires; its citation was corrected from "§2-3" to "§12" in Stage 5. (2) The Agent Contract Template (Stage 7) gives the ecosystem a complete, fillable specification format per Card 01 §1's requirement — no agent-building process was defined before this file existed. (3) AGENTS.md (Stage 7) gives the ecosystem its master specification, classified as a signed instruction artifact per Card 06 §15. The model-routing-table.md operationalizes Card 06 §26's classification-aware routing requirement with a concrete, evidence-backed provider approval registry. ADR-006 and ADR-007 formally document the Tool/Skill Registry split and Circuit Breaker/Policy Server split, closing the "is this documented anywhere?" gap that Stage 4's Alternative-Architecture notes identified.
**Risk if Ignored:** N/A.
**Recommended Action:** None. This dimension went from the most concerning v1 finding to fully resolved.

---

## Dimension 6 — Production Readiness

**Score: 9**
**Confidence:** High — every major component now has an explicit failure mode defined; Card 06 §20's fail-closed table is the governing pattern consistently applied.
**Evidence:** Strong — Stage 6 added blast-radius statements to five cards: Card 01 §2 (crash recovery via PostgresSaver), Card 02 §5 (whole-registry-file unavailability → system-wide fail-closed), Card 03 §12 (memory backend → degraded-read, not hard-fail), Card 04 §6 (Skill Registry unavailability → no new Skills loadable), Card 05 §5 (Evaluation unavailable → non-blocking backfill). Card 06 §20's fail-closed-by-tier table governs the Policy Server's own failure mode. Tech-stack.md Decision #4 explicitly disqualifies MemorySaver for production. Card 01 §2's Implementation Note (LangGraph) names PostgresSaver as the required backend — "defaulting to MemorySaver during early development and silently shipping that default would make this section's blast-radius guarantee false in practice."
**Risk if Ignored:** One honest gap: Card 06's failure modes for the Circuit Breaker tripping while the Policy Server is simultaneously unavailable (a correlated failure) are not explicitly addressed — covered implicitly by the fail-closed table but not named as a correlated scenario.
**Recommended Action:** Log the correlated-failure scenario as a Phase 1 verification spec addition (see Debt Register D1), not a card change.

---

## Dimension 7 — Terminology Consistency

**Score: 10**
**Confidence:** High — the v1 collision ("structured state" vs. "Structured Records") is resolved; the Stage 1 audit found no other terminology inconsistencies.
**Evidence:** Strong — Card 03's "Structured Records" rename is explicitly flagged in Card 03's Direct Implications: "This card corrects and supersedes Card 01's original 'long-term memory = RAG' framing, and renames the fifth memory type from 'structured state' to **Structured Records** to avoid colliding with Card 01's `state` definition — Card 01 should be read with this card's five-type model and corrected terminology in mind." Card 03 §4's "Implementation Note (LangGraph)" correctly scopes LangGraph-specific terminology (state-as-session, no formal Session object) to an explicitly labeled section, distinguishing it from general architectural language. The Architecture Traceability Matrix uses consistent section-reference notation (`Card X §Y`) throughout.
**Risk if Ignored:** N/A.
**Recommended Action:** None.

---

## Dimension 8 — Scope Discipline

**Score: 9**
**Confidence:** High — every card's header explicitly states what it defers to other cards; AGENTS.md §14 enforces non-restatement at the runtime artifact level.
**Evidence:** Strong — Card 02's header: "it deliberately does not define risk tiers, permission/scope enforcement, or sandboxing — those are Card 06's authority." Card 03 §11 (Memory Contracts): "this field does not redefine authorization, it points to where authorization already lives." Card 05 §4 (Evaluator Hierarchy): "Evaluation detects — it does not enforce." Card 06 §9a explicitly names the Safety-Evaluation-enforcement-boundary, preventing Card 05 from absorbing enforcement responsibility. AGENTS.md §14: "if an edit to this file finds itself writing out a tier table, a field schema, or a memory-type list instead of citing one, stop." Tech-stack.md's "Replacement Philosophy" section explicitly states each decision is an implementation choice, not an architectural dependency.
**Risk if Ignored:** N/A.
**Recommended Action:** None. Minor observation: Card 01's Direct Implications section contains build-guidance content ("core/harness/ implements the 4-step...") that is slightly outside Card 01's pure-architecture scope — consistent with every other card's Direct Implications section and serves as a legitimate bridge between architecture and implementation, not a scope violation.

---

## Dimension 9 — Maintainability

**Score: 10**
**Confidence:** High — every card has a self-contained version history, the Traceability Matrix provides the single lookup table for all cross-references, and the ADR collection provides permanent, implementation-independent decision records.
**Evidence:** Strong — every card carries a stacked changelog (Card 01: 3 entries; Card 03: 4 entries; Card 06: 3 entries). The Traceability Matrix's maintenance rule states: "no fix is applied from memory or assumption alone." ADR-001 through ADR-007 provide durable decision records that survive any individual card being edited — a future maintainer who changes Card 03 can check ADR-004 and ADR-006 to understand why the Skill Registry and Memory separation exists before making a change that might break it. Working Principle #7 ensures no card is bumped without a real content change.
**Risk if Ignored:** N/A.
**Recommended Action:** None.

---

## Dimension 10 — Implementation Feasibility

**Score: 9**
**Confidence:** High — all four foundational files now exist; tech-stack.md resolves every implementation technology decision with explicit rationale.
**Evidence:** Strong — Stage 7 produced all four previously-missing files: AGENTS.md (master runtime specification), agent-contract-template.md (17-section spec template), model-routing-table.md (11-section provider approval registry with evidence metadata and expiration), tech-stack.md (11 technology decisions with Decision Status, Review Trigger, and Tradeoffs per decision). Tech-stack.md explicitly resolves: PostgreSQL+pgvector (one DB for all memory types + checkpointing), PostgresSaver (durable checkpointing), OpenTelemetry+Jaeger (no usage caps), Docker per-execution (isolation), OPA WASM-embedded (authorization without a separate service), pytest, Git commit signing. No technology decision was left as "TBD" except those explicitly flagged as Intentionally Deferred (CI platform, secret manager, container registry, etc.).
**Risk if Ignored:** N/A. Note: agent-contract-template.md §12 Project-Specific Notes table has the correct project names but per-agent `hitl_required_for` remains placeholder — correctly a Phase 1 task when agent contracts are actually filled.
**Recommended Action:** None blocking.

---

## Dimension 11 — Architectural Simplicity

**Score: 9**
**Confidence:** High — ADR-006 and ADR-007 provide explicit, evidence-backed justifications for the two most-questioned complexity decisions in the v1 review.
**Evidence:** Strong — applying the rubric's required 4-question test to the two challenged components:

*Tool Registry vs. Skill Registry (ADR-006):* (1) Problem: governance semantics differ — Tool is callable, Skill is instructional; (2) Replacement: unified Capability Registry; (3) What disappears: ability for binding-rule conditions to evolve independently (Tool's call-time check vs. Skill's context-assembly-time check); (4) Worth it: yes — the merged schema would be genuinely misleading, not just larger.

*Circuit Breaker vs. Policy Server (ADR-007):* (1) Problem: Decide-time authorization is different from runtime-state enforcement; (2) Replacement: merged into Policy Server; (3) What disappears: independent failure modes — a compromised Policy Server would simultaneously disable the Circuit Breaker; (4) Worth it: yes — this is a genuine security boundary, not complexity for its own sake.

The memory-type split (5 types), Risk Tier table (6 tiers), and 4-level Evaluator Hierarchy were each tested against the rubric's 4-question framework — all produce real capability losses if simplified.
**Risk if Ignored:** N/A.
**Recommended Action:** None. ADR-006/007 close this dimension's v1 concern definitively.

---

## Dimension 12 — Evolutionary Resilience *(Future Validation Criterion)*

**Score: 9**
**Confidence:** Medium — no implementation exists to stress-test, but the rubric's required test scenarios can be walked through analytically.
**Evidence:** Medium — walking through the rubric's required scenarios:

*Adding a 4th portfolio project:* Card 06 §14's regulated-project override table would need one new row; every other component extends without restructuring. Clean.

*New model provider:* model-routing-table.md's provider evidence schema accommodates a new row; tech-stack.md Decision #2 explicitly states replacements are implementation choices, not architectural dependencies. Clean.

*New memory type:* Card 03 §5's five-type taxonomy would need an extension — a Card 03 edit and Matrix update, but no other card owns memory-type definitions. Single-card change.

*New risk tier:* Card 06 §3's table would need a new row; every reference to "Tier 4+" in other cards would need review. Manageable — the Matrix makes this traceable.

*Replacing LangGraph:* Card 03 §4's Implementation Note (LangGraph) would become stale; tech-stack.md Decision #1 would change; the PostgresSaver decision would change. However, Card 01 §2's Plan→Act→Observe→Decide loop, Card 03's memory types, Card 06's security architecture, and the Contract pattern are all framework-independent. The architecture survives — the implementation notes don't. Notes are labeled `Implementation Note (LangGraph)` precisely so they're distinguishable from architectural content.
**Risk if Ignored:** N/A — but the LangGraph replacement scenario would require 5-6 files to update, worth flagging in the debt register.
**Recommended Action:** Log "LangGraph replacement scenario — if orchestration framework changes, which documents need updating" as a Phase 3+ debt register entry (see Debt Register D2).

---

## Dimension 13 — Blast Radius Analysis *(Future Validation Criterion)*

**Score: 9**
**Confidence:** High — Stage 6 explicitly addressed this dimension; every component named in the rubric now has documented blast-radius statements.
**Evidence:** Strong — rubric requires named analysis for Policy Server, Circuit Breaker, Memory, Registries, Tool Layer, Evaluation, and Harness. Both halves documented (what breaks AND what still works):

- **Policy Server:** Card 06 §20's fail-closed table (Tier 0-2 may proceed; Tier 3+ blocks). Circuit Breaker remains operational independently (ADR-007). What still works: Tier 0-2 computation, documentation/reference loading, evaluation against already-collected traces.
- **Circuit Breaker:** Card 06 §11 — quarantine scoped to run/agent/tool/system level explicitly. Policy Server token revocation is the one dependency (one-directional, documented). What still works: unaffected runs continue; pre-approved user-facing explanations may still emit.
- **Memory:** Card 03 §12 — degraded-read (in-session context only), flagged in trace. Tier 2+ Structured Records reads defer to Card 06 §20. Writes queue/drop, never block. What still works: in-session state, system/tool_output/user_upload context.
- **Tool Registry / Skill Registry:** Card 02 §5 / Card 04 §6 — whole-file unavailability = system-wide fail-closed for new bindings. What still works: in-flight JIT-tokened calls may complete; already-loaded Skills remain functional for the current turn.
- **Evaluation:** Card 05 §5 — non-blocking, backfills on recovery. What still works: all agent turns complete normally; observability traces accumulate.
- **Harness:** Card 01 §2 — PostgresSaver checkpoint recovery, in-flight calls treated as failed per Card 02 §4's idempotency. What still works: recovery resumes from last checkpoint, no full restart.

One gap: correlated failure scenario (Policy Server + Circuit Breaker both unavailable simultaneously) not explicitly named.
**Risk if Ignored:** Correlated failure could produce unexpected behavior if not tested in Phase 1.
**Recommended Action:** Add correlated-failure scenario to architecture-verification-specification.md as a new §9 before Phase 1 begins (see Debt Register D1).

---

## Dimension 14 — Operational Validation Readiness *(Future Validation Criterion)*

**Score: 10**
**Confidence:** High — architecture-verification-specification.md directly addresses this dimension's core requirement with 22 scenarios, each with a measurable pass criterion.
**Evidence:** Strong — the rubric's 9-10 anchor: "The architecture explicitly names this boundary and defines (or points to) a planned Architecture Verification Suite to run post-Phase-1." architecture-verification-specification.md's opening states: "no scenario in this document is executed until Phase 1 infrastructure exists. This document is written now, at the end of Phase 0, so the verification plan is drawn from the final corrected architecture rather than reconstructed from scratch later under implementation pressure." 22 scenarios across 7 groups plus 4 performance scenarios in §8, each with: Trigger, Expected, Pass Criterion (measurable, not vague), Execution Phase ([PHASE 1] or [PHASE 3+]). Coverage Matrix shows which cards are covered. All of the rubric's recommended verification scenarios (happy path, tool failure, policy denial, circuit breaker activation, retry exhaustion, context overflow, rollback, concurrent orchestration, delegation failure) are present with measurable pass criteria.
**Risk if Ignored:** N/A.
**Recommended Action:** One addition identified during scoring: add correlated-failure scenario (Dimension 13 note) to the verification spec before Phase 1.

---

## Summary Scorecard

| # | Dimension | Score | Category |
|---|---|---|---|
| 1 | Ownership Clarity | 9 | Freeze Criterion |
| 2 | Dependency Flow | 10 | Freeze Criterion |
| 3 | Circular References | 10 | Freeze Criterion |
| 4 | Duplication | 9 | Freeze Criterion |
| 5 | Missing Abstractions | 10 | Freeze Criterion |
| 6 | Production Readiness | 9 | Freeze Criterion |
| 7 | Terminology Consistency | 10 | Freeze Criterion |
| 8 | Scope Discipline | 9 | Freeze Criterion |
| 9 | Maintainability | 10 | Freeze Criterion |
| 10 | Implementation Feasibility | 9 | Freeze Criterion |
| 11 | Architectural Simplicity | 9 | Freeze Criterion |
| 12 | Evolutionary Resilience | 9 | Future Validation |
| 13 | Blast Radius Analysis | 9 | Future Validation |
| 14 | Operational Validation Readiness | 10 | Future Validation |

**Lowest freeze-criterion score: 9**
**Mandatory gate rule (≤4 on any freeze criterion = Not-Freeze-Ready): CLEARED**

---

## Blocking Issues

**None.** No dimension scored ≤4. No freeze-criterion dimension scored below 9.

---

## Non-Blocking Refinements

1. **Card 03 Direct Implications / §4 overlap** — minor cosmetic duplication between the LangGraph note in Direct Implications and §4's Implementation Note. No semantic conflict, purely editorial. Already in the Deferred list's "Pre-publication" editorial pass.
2. **Agent Contract Template §12** — placeholder `hitl_required_for` language per agent. Correctly a Phase 1 task when agent contracts are actually filled in.
3. **Card 01 Direct Implications scope** — contains build-guidance content slightly outside Card 01's pure-architecture scope. Not a violation — every card has Direct Implications — worth noting as a future editorial consideration only.

---

## Architectural Debt Register

| # | Item | Priority | Owner | Review Version | Target Resolution |
|---|---|---|---|---|---|
| D1 | Correlated failure scenario (Policy Server + Circuit Breaker simultaneously unavailable) not explicitly addressed in cards or verification spec | Medium | Card 06 §20 / `architecture-verification-specification.md` | Cohesion Review v1 re-score | Add new §9 "Correlated Failure Scenarios" to verification spec before Phase 1 begins |
| D2 | LangGraph replacement scenario — 5-6 files require updates if orchestration framework changes (Card 03 §4 Implementation Notes, tech-stack.md Decision #1, PostgresSaver decision) | Low | Card 03 §4 / tech-stack.md Decision #1 | Cohesion Review v1 re-score | Phase 3+, if/when framework change becomes a live consideration |
| D3 | Runtime artifact-signature verification (Card 06 §15/§16) — Git commit signing covers provenance only, not runtime enforcement — a real currently-open gap | High | Card 06 §15 / tech-stack.md Decision #11 | Cohesion Review v1 re-score | Phase 1, when `core/security/` implements signing enforcement — already tracked in Polish/Deferred Enhancements |

---

## Future Validation Plan

Per Dimensions 12-14, the following must be verified once Phase 1 infrastructure exists — measurable pass criteria already specified in `docs/governance/architecture-verification-specification.md`:

| Scenario | Verification Spec Reference | Pass Criterion |
|---|---|---|
| Policy Server unavailability | §1.1 | Zero Tier 3+ actions succeed during simulated outage |
| Circuit Breaker trip and quarantine | §2.1-2.3 | No tool calls execute after trip event; quarantine requires Tier 4+ approval to release |
| Memory backend degradation | §3.1 | Run continues with in-session context only; trace flags `memory_retrieval_skipped` |
| Tool Registry unavailability | §3.2 | No new tool bindings succeed system-wide |
| Skill Registry unavailability | §3.3 | No new Skill loads succeed; already-loaded Skills functional |
| Harness crash recovery | §3.5 | Resumes from last checkpoint; no in-flight call assumed successful |
| Prompt injection via tool output | §4.3 | System instruction unchanged after injected content processed |
| Concurrent state mutation | §5.2 | LangGraph reducer resolves conflict deterministically |
| Classification-aware routing | §6.3 | No sensitive-classified request reaches non-local provider |
| Correlated failure (Policy Server + Circuit Breaker) | Not yet in spec — Debt Register D1 | To be added to verification spec §9 before Phase 1 |

---

## Overall Verdict

**FREEZE-READY.**

The architecture has materially resolved every issue identified in the v1 review cycle. All 8 specifically-flagged dimensions (2, 4, 5, 7, 9, 10, 11, 13) score 9-10 with Strong evidence citations, compared to materially lower scores in v1. All 3 non-flagged freeze-criterion dimensions (3, 6, 8) score 9-10. All 3 future-validation dimensions (12, 13, 14) score 9-10.

**No dimension scored below 9. No blocking issues found. The mandatory gate rule is cleared.**

**One honest limitation of this review:** as an internal reviewer who participated in building every artifact being scored, this review carries inherently lower independence than the concurrent external review. If the external reviewer finds a gap this internal review missed, that gap should be treated as real and addressed before Phase 0 is declared closed. The reconciliation pass should weight the external review's findings accordingly.

This verdict covers architecture validation only — not operational proof. Operational verification begins when Phase 1 produces infrastructure to test against, per `docs/governance/architecture-verification-specification.md`.

---

*This document to be filed at `docs/governance/cohesion-reviews/v1/stage-10-internal-rescore.md` once the external review is received and the reconciliation pass is complete.*
