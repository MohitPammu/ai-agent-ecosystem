# Architecture Traceability Matrix

**Purpose:** Single lookup table mapping every cross-card concept to its canonical owner and every place that concept is referenced. Produced by Phase 0 Closure Plan Stage 2, seeded directly from Stage 1's exhaustive forward-reference audit. No fix in Stages 3+ is marked complete without being checked against this matrix first, and the "Last Verified" column updated after.

**Status:** Living document — updated at the end of every Closure Plan stage that touches a tracked concept, and at every future cohesion-review cycle.

---

| Concept | Canonical Owner (Card §) | Referencing Cards/Sections | Last Verified |
|---|---|---|---|
| Risk Tiers (0-5, default approval requirements) | Card 06 §3 | Card 02 §4 (`risk_tier` field, reference-only), Card 02 §8, Card 04 §5 (Skill Contract implicit risk exposure), Card 05 §4 (tier 4 = HITL), Card 06 §39 (table itself), Card 07 §5, Card 07 §6 | Stage 1 (2026-06) |
| Regulated-Project Risk Tier Override (tightened Tier 2/3 for Projects 1/3) | Card 06 §14 | Card 06 §3 (note pointing to §14), Card 07 §6 | Stage 1 (2026-06) |
| Agent Contract Standard | Card 01 §1 / `docs/architecture/agent-contract-template.md` (Phase 0, file 0.12) | Card 02 (Tool Contract mirrors pattern), Card 03 §11 (Memory Contracts mirror pattern), Card 04 §5 (Skill Contract mirrors pattern), Card 01 §8 (forward-reference to template file) | Stage 1 (2026-06) |
| Tool Contract (schema, error contract, observability metadata, risk_tier) | Card 02 §4 | Card 01 §8 (forward-reference), Card 05 §3 (tool-usage failure category), Card 05 §6 (error categories consume failure taxonomy), Card 06 §3/§4 (`risk_tier` resolves against Card 06's table), Card 06 §20 (binding-check sequencing), Card 07 §2 (YAML field types example) | Stage 1 (2026-06) |
| Tool Registry (discovery, binding rule, assignment authority) | Card 02 §5 | Card 04 §6 (Skill Registry mirrors), Card 06 §4 (Policy Server holds assignment authority), Card 06 §7 (registry integrity), Card 06 §16 (runtime verification) | Stage 1 (2026-06) |
| Tool Lifecycle | Card 02 §6 | Card 04 §8 (Skill Lifecycle mirrors), Card 05 §7 (Skill Certification mirrors `Tested→Certified` gate) | Stage 1 (2026-06) |
| MCP Version Pinning / Versioning | Card 02 §8 | Card 07 §9 (rollback readiness ties to `tool_version`/`skill_version`) | Stage 1 (2026-06) |
| A2A Protocol (mechanics) | Card 02 §7 | Card 01 §6 (Coordinator pattern), Card 03 §6 (separate-history pattern), Card 06 §6 (trust boundary enforcement) | Stage 1 (2026-06) |
| Permission Model (ABAC + JIT downscoping) | Card 06 §4 | Card 02 §5 (assignment authority deferred here), Card 06 §18 (JIT lifecycle detail) | Stage 1 (2026-06) |
| Sandboxing | Card 06 §5 | Card 02 (deferred), Card 07 §4 (Context Hygiene placeholder resolution happens inside sandbox) | Stage 1 (2026-06) |
| Policy Server (pattern, integrity, failure mode) | Card 06 §4, §20, §21 | Card 02 §5 (assignment authority), Card 06 §9 (Safety enforcement actor), Card 07 §5 (spec/governance separation principle) | Stage 1 (2026-06) |
| Circuit Breaker (state-ownership boundary) | Card 06 §11 | Card 01 §2 (Decide/Act split it respects — Harness owns state transition) | Stage 1 (2026-06) |
| State (definition) | Card 01 §1 | Card 03 §4 (Session's "state" folder builds on this), Card 03 §5 (Structured Records renamed to avoid collision) | Stage 1 (2026-06) |
| Memory Types (5: episodic, semantic, procedural, preference, Structured Records) | Card 03 §5 | Card 01 §5 (original 2-tier framing, corrected by Card 03), Card 04 §1/§6 (procedural memory → Skills bridge) | Stage 1 (2026-06) |
| Memory Contracts | Card 03 §11 | Card 01 (Agent Contract pattern), Card 02 (Tool Contract pattern), Card 06 §8 (authorization layer underneath) | Stage 1 (2026-06) |
| Memory Consolidation / Contested-status | Card 03 §8 | Card 04 §8 (promotion-candidate source), Card 06 §8 (who clears `contested` flag) | Stage 1 (2026-06) |
| Context Budget Management | Card 03 §9 | Card 04 §3 (Progressive Disclosure cites same principle) | Stage 1 (2026-06) |
| Source Precedence / Regulated-Domain Exception | Card 03 §10 | Card 06 (project-policy boundary, not redefined there) | Stage 1 (2026-06) |
| Source Boundary Preservation (`source_type` tagging + non-override rule) | Card 03 §12 | Card 06 §26 (citation corrected from "§2-3" to "§12," requirement now operationalized) | Stage 5 (2026-06) — mechanism drafted, citation corrected |
| Procedural-Memory-to-Skill Promotion | Card 04 §8 | Card 03 §8 (consolidation flags candidates), Card 01 §8 (Level 4 autonomy prohibition governs this) | Stage 1 (2026-06) |
| Skill Contract | Card 04 §5 | Card 01/02/03 (Contract pattern lineage) | Stage 1 (2026-06) |
| Skill Registry | Card 04 §6 | Card 02 §5 (mirrored pattern) | Stage 1 (2026-06) |
| Skill Certification | Card 05 §7 | Card 04 §8 (open dependency this discharges), Card 02 §6 (lifecycle gate mirrored) | Stage 1 (2026-06) |
| Observability Standard / Required Fields | Card 05 §5 | Card 01 §8 (fixed — now points to Card 05 §5 instead of restating, Stage 3), Card 02 (tool-call metadata feeds in), Card 06 §11 (Security Event Schema extends without polluting) | Stage 3 (2026-06) — fixed, Card 01 §8 now points to Card 05 §5 instead of restating |
| Evaluation vs. Observability Separation | Card 05 §5 | Card 01 §8 (binding requirement), Card 06 §11 | Stage 1 (2026-06) |
| Evaluator Hierarchy (Automated → LLM-as-Judge → Agent-as-Judge → HITL) | Card 05 §4 | Card 06 §9 (tier 4 = HITL), Card 06 §10 (tier 3 = Agent-as-Judge trace access), Card 07 §6 (mirrored for code review), Card 07 §8 (tier 1 = golden-set regression) | Stage 1 (2026-06) |
| Formal Failure Taxonomy | Card 05 §6 | Card 01 §2 (Decide stage branches on it), Card 01 §8 (gap originally flagged), Card 02 §4 (error contracts consume it), Card 03 (gap originally flagged) | Stage 1 (2026-06) |
| HITL Unification (infrastructure-level only, not one workflow) | Card 05 §4 | Card 01 §6, Card 02, Card 03 §8, Card 04 §6/§8, Card 06 §24 (authorization layer underneath) | Stage 1 (2026-06) |
| Safety/Alignment Evaluation Enforcement Boundary | Card 06 §9 | Card 05 (Pillar 4 — detects but does not enforce) | Stage 1 (2026-06) |
| Agent-as-Judge Trace Access / Redaction | Card 06 §10 | Card 05 §4 tier 3 (the dependency being discharged) | Stage 1 (2026-06) |
| Instruction-Artifact Signing | Card 06 §15 | Card 04 §8 (Skill lifecycle subject to same controls), Card 07 (AGENTS.md as signed artifact, Phase 0 file 0.10) | Stage 1 (2026-06) |
| Data Classification (monotonic inheritance) | Card 06 §13, §23 | Card 03 (memory consolidation inherits), Card 04 (promotion candidates inherit), Card 05 (trace-based evaluation inherits) | Stage 1 (2026-06) |
| Classification-Aware Model Routing | Card 06 §26 | Card 01 §1 (zero-cost routing principle being extended), Card 07 §11 (provider approval metadata schema that operationalizes it) | Stage 1 (2026-06) |
| Provider Approval Metadata Schema | Card 07 §11 | Card 06 §26 (tracked item this discharges) | Stage 1 (2026-06) |
| Tier 3/4 Code-Review HITL Citation | Card 07 §6 | Card 06 §3, §14 | **Verified correct, Stage 1 (2026-06) — Progress Log corrected same session** |
| Three-Tier Code Review | Card 07 §6 | Card 05 §4 (Evaluator Hierarchy it mirrors) | Stage 1 (2026-06) |
| Canary / Rollout Discipline | Card 07 §9 | Card 07 §11 (fleet-governance rollout reuses same discipline, Phase 5+) | Stage 1 (2026-06) |
| Context Hygiene (`[[VARIABLE]]` placeholders) | Card 07 §4 | Card 06 §5 (resolved only inside sandbox), Card 06 §13 (supports classification discipline) | Stage 1 (2026-06) |
| Alternative-Architecture-Considered Decisions (Tool/Skill Registry separation; Circuit Breaker/Policy Server separation) | Card 02 §5 / Card 04 §6 (Registry separation, documented both sides); Card 06 §11 (Circuit Breaker, documented) | Card 02 §5, Card 04 §6, Card 06 §11 — each now carries its own "Alternative Architecture Considered" note citing `cohesion-reviews/v1/review-reconciliation.md` | Stage 4 (2026-06) — documented in-card |
| LangGraph Implementation-Note Labeling | Card 03 §4 | Card 03 §4 — LangGraph-specific content now wrapped in `**Implementation Note (LangGraph):**` label, scoped separately from the general ADK-comparison sentence | Stage 4 (2026-06) — relabeled |
| Blast-Radius / Degradation Behavior (Memory, Registry-file, Evaluation-service, Harness/Orchestrator crash) | Not yet documented in-card | Card 03, Card 02 §5/§16, Card 04 §6, Card 05, Card 01 §2 | **Pending — Stage 6** |
| Four Missing Foundational Artifacts | Not yet drafted | `agent-contract-template.md`, `AGENTS.md`, `model-routing-table.md`, `tech-stack.md` | **Pending — Stage 7** |
| Card 06 Topic Index | Not yet added | Card 06 (self) | **Pending — Stage 8** |
| Architecture Decision Records (ADR-001 through ADR-007) | Not yet created | `docs/governance/adr/` | **Pending — Stage 8** |
| Architecture Verification Specification | Not yet created | `docs/governance/architecture-verification-specification.md` | **Pending — Stage 9** |

---

## Matrix Maintenance Rule

Every stage from this point forward (Stage 3 onward) must:
1. Look up the relevant concept row(s) before applying any fix.
2. Update the "Last Verified" column to the current stage/date once the fix is applied and cross-checked against every referencing card/section listed in that row.
3. Add new rows if a stage's work introduces a new cross-card concept not already tracked here (e.g., Stage 5's new Card 03 subsection, Stage 7's four new files).

This matrix is the direct input to Stage 3 and every subsequent stage — no fix is applied from memory or assumption alone.
