---
spec_id: S5
title: Phase 1 Minimum Conformance Profile
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-18
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 01, Card 02, Card 03, Card 04, Card 05, Card 06, Card 07, ADR-007, tech-stack.md Decision #7]
depends_on: [S7]
gates: [1.2, 1.3, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10, 1.11, 1.12, 1.13, 1.14, 1.15, 1.16, 1.17]
source: docs/governance/cohesion-reviews/v1/stage-10-external-rescore.md (lines 330-347), stage-10-reconciliation.md (row 184, line 30)
---

# S5 — Phase 1 Minimum Conformance Profile

**Status:** 1.0 (Approved)

## 1. Purpose

Defines the minimum implementation required before any of the three portfolio projects begins, and — equally important — defines what is *explicitly not* required yet. This spec exists because the External Stage 10 review identified a real risk: the frozen architecture's full complexity (two registries, four contract types, OPA/Rego, artifact signing, Jaeger, Postgres/pgvector, per-execution Docker sandboxing, tiered evaluation, a Circuit Breaker, A2A protocol, and governance workflows) is a heavy Phase 1 burden for one developer building against public/synthetic data. The risk, if ignored: most of Phase 1 gets spent building governance infrastructure before validating whether the agent abstractions actually help the three portfolio workflows.

This profile does **not** remove any architectural boundary. It sequences which *instance* of each boundary gets built first — one of something, not none of it.

**Conformance Principle:** A Minimum Conformance implementation is conformant only if every architectural boundary, security invariant, interface contract, ownership rule, and governance requirement defined by the frozen Phase 0 architecture remains intact. Minimum Conformance reduces implementation scale only; it never weakens architectural behavior.

## 2. Non-Goals

This specification does not define:

- production scalability
- high availability
- clustering or replication
- horizontal scaling
- disaster recovery
- deployment topology beyond minimum conformance
- performance targets

S5 defines what "architecturally complete but minimally implemented" means for Phase 1. It is not a deployment or scaling specification, and later readers should not treat it as one.

## 3. Scope — what the Minimum Conformance Profile requires

All security invariants remain mandatory. What's constrained is cardinality and deployment topology, not enforcement.

| Component | Full architecture | Minimum Conformance Profile | Source |
|---|---|---|---|
| Tool Registry | Registry supports multiple files/namespaces at scale | **One** `core/tool-registry.yaml` file | External rescore, recommendation 1 |
| Skill Registry | Same | **One** `core/skills-registry.yaml` file | External rescore, recommendation 2 |
| Policy Authority (OPA) | Distributable policy bundles, potentially multi-instance | **One process**, OPA WASM-embedded (already the frozen deployment choice per tech-stack.md Decision #7 — this profile doesn't change the model, it constrains everything else around it to match a single-process reality) | External rescore, recommendation 3; reconciliation confirms no OPA deployment model change |
| Memory backend | PostgreSQL+pgvector, potentially scaled/replicated | **One durable Postgres instance** | External rescore, recommendation 4 |
| Evaluation (Card 05 §4 tiers) | Three tiers: deterministic, LLM-as-Judge, Agent-as-Judge | **Deterministic evaluators first**; LLM-as-Judge and Agent-as-Judge deferred (see §6 trigger) | External rescore, recommendation 5 |
| Circuit Breaker (ADR-007, Card 06 §11) | Run-level and system-level scope, one configurable module | **Both scopes required in Phase 1 — not deferred.** Aggregation mechanism specified separately in S7. Full rationale for why the originally-recommended deferral was not adopted is in Appendix A | Revised — Appendix A |
| A2A (multi-agent protocol) | In scope, Card 06 threat model covers it | **Deferred to Phase 3.** No A2A implementation in Phase 1. Single-agent and Coordinator-delegated sub-agent patterns (Card 01 §6) are in scope; cross-agent A2A messaging is not | External rescore, recommendation 7; Patch 4 added this deferral note; reconciliation confirms "no A2A removal," only deferral |

## 4. Conformance vs. Production

**Conformance** means the implementation satisfies the architecture — every boundary, invariant, and contract from Cards 01–07 is intact and enforced, at minimum scale.

Conformance does **not** mean production-ready. Production readiness (availability targets, scaling, disaster recovery — see §2 Non-Goals) is out of scope for S5 and belongs to later phases, once actually scoped.

## 5. Deferred Capability Summary

Only A2A has a frozen phase assignment. Circuit Breaker is no longer deferred (Appendix A). Every remaining deferral has a defined *trigger condition* (§6) but no committed phase number — assigning one now would commit this project to a Phase 2/3 roadmap that hasn't been reviewed or approved. Phase numbers get filled in when that roadmap is actually scoped, not invented here.

| Deferred Capability | Trigger for next stage | Phase |
|---|---|---|
| Multi-file/namespaced Tool Registry | Not yet defined — no frozen source specifies a growth trigger | Not yet phase-assigned |
| Multi-file Skill Registry | Not yet defined | Not yet phase-assigned |
| Distributed policy bundle deployment | Not yet defined | Not yet phase-assigned |
| Clustered/replicated Postgres | Not yet defined | Not yet phase-assigned |
| LLM-as-Judge evaluation (tier 2) | Deterministic tier implemented and passing for at least one portfolio project's primary workflows (§6.1) | Not yet phase-assigned |
| Agent-as-Judge evaluation (tier 3) | LLM-as-Judge operational (§6.1) | Not yet phase-assigned |
| A2A protocol | — | **Phase 3** (frozen, Patch 4) |

## 6. Evolution Triggers

### 6.1 Evaluation tier progression

LLM-as-Judge implementation begins once `core/evaluation/`'s deterministic tier is implemented and passing for at least one portfolio project's primary workflows — a concrete, checkable milestone tied to real work, not a subjective readiness judgment or a coverage percentage.

Agent-as-Judge implementation begins only after LLM-as-Judge integration is operational.

### 6.2 Registry, policy distribution, Postgres scaling

No trigger is defined for these in this spec. They remain single-instance until a future phase-scoping exercise defines one — this is an explicit gap, not an oversight, and should not be filled in speculatively here.

## 7. Success Criteria

S5 is considered satisfied when:

- every minimum-capability scenario in §8 passes
- every architectural invariant from Cards 01–07 remains intact under the minimum-cardinality implementation
- no capability required by Cards 01–07 has been silently omitted (only deferred, per §5, with a stated trigger or explicit "not yet defined")
- every Phase 1 subsystem can be extended toward its full architecture later without requiring changes to any frozen architectural contract already established in Cards 01–07 (not a claim that zero code changes will be needed — only that no *contract* gets broken)

## 8. Scenarios (Given/When/Then)

```gherkin
Scenario: S5-001 — A single Tool Registry file satisfies the binding rule for Phase 1
  Given core/tool-registry.yaml exists with at least one entry
  And no second Tool Registry file exists anywhere in the runtime path
  When core/harness/ performs a tool-call binding check (Card 02 §5)
  Then the binding rule is fully enforced against the single file
  And this establishes the minimum topology AVS-CF-003 (compound Registry outage) executes against
  # related_avs: AVS-BR-002, AVS-CF-003, AVS-PERF-003

Scenario: S5-002a — A single Skill Registry file satisfies the promotion pattern for Phase 1
  Given an approved Skill candidate exists in Draft state
  When it is promoted to Active per Card 04 §8's promotion pattern
  Then the lifecycle transition is recorded in the single core/skills-registry.yaml file
  And no additional registry sharding is required to pass conformance
  # related_avs: AVS-BR-003, AVS-CF-003, AVS-PERF-003

Scenario: S5-002b — Skill loading checks the single registry at call time
  Given an Active Skill is present in core/skills-registry.yaml
  When the Harness requests that Skill
  Then binding and assignment rules are evaluated against the single registry
  # related_avs: AVS-CF-003

Scenario: S5-003 — Single-process OPA WASM-embedded satisfies Policy Authority for Phase 1
  Given the Policy Server is deployed as OPA WASM-embedded in a single process (tech-stack.md Decision #7, unchanged by this profile)
  When any agent action requires authorization (Card 06)
  Then policy evaluation occurs in-process
  And no distributed/multi-instance policy bundle distribution mechanism is required to satisfy conformance
  And this does not block S1 (Policy Bundle Distribution and Activation) — S1 still governs how the single bundle activates and rolls back
  # related_avs: AVS-PS-001, AVS-PS-002, AVS-PS-003, AVS-PERF-001, AVS-CF-005

Scenario: S5-004 — One Postgres instance satisfies memory conformance for Phase 1
  Given a single PostgreSQL+pgvector instance is running
  When core/memory/ stores or retrieves any of the five persistent memory types (episodic, semantic, procedural, preference, Structured Records — Card 03 §5)
  Then all five resolve against that one instance
  And in-run working state remains owned by the Harness/LangGraph state model (Card 01 §1), not treated as a sixth persistent memory type
  And one instance does not imply one schema or one transaction boundary for every concern
  And no replication or sharding is required for Phase 1 conformance
  # related_avs: AVS-BR-001, AVS-CF-001, AVS-PERF-002

Scenario: S5-005 — Deterministic evaluators alone satisfy Phase 1 evaluation conformance
  Given core/evaluation/ implements a minimum deterministic evaluator set: contract/schema conformance, observability-field completeness, authorization outcome correctness, stop-condition compliance, and source-boundary preservation where deterministic
  When an agent run completes
  Then evaluation produces a pass/fail result using deterministic checks only
  And LLM-as-Judge and Agent-as-Judge tiers are not required to be implemented for Phase 1 exit
  But the evaluation-harness-spec (1.16) must explicitly note where tier-2/3 hooks will attach later, so Phase 2 doesn't require re-architecting the evaluation interface
  # related_avs: AVS-BR-004

Scenario: S5-006 — Circuit Breaker conformance requires both run-level and system-level scope, per S7
  Given core/security/ implements the Circuit Breaker as one configurable module per Card 06 §11 and S7's aggregation mechanism
  When a run-level failure threshold is crossed
  Then the run is halted per AVS-CB-001
  When accumulated low-severity events cross the system-level threshold, per S7's counting mechanism
  Then all active runs are halted per AVS-CB-002
  And neither scope is treated as optional or deferred for Phase 1 conformance
  # verifies: AVS-CB-001, AVS-CB-002

Scenario: S5-007 — A2A is absent without violating architectural completeness
  Given no A2A protocol implementation exists in core/ during Phase 1
  When a Coordinator pattern requires delegation to a sub-agent (Card 01 §6)
  Then delegation occurs via direct LangGraph sub-graph invocation, not cross-process A2A messaging
  And this is documented as an explicit Phase 3 deferral, not a silent gap
  And Card 06's A2A threat-model coverage remains valid and unexecuted until Phase 3 implements A2A
  # related_avs: AVS-MA-003
  # deferred_avs: AVS-MA-001 (A2A Delegation — Untrusted Content Boundary) — requires A2A, deferred to Phase 3
```

## 9. Profile Invariant (non-scenario)

The following is a profile-level invariant, not an executable Given/When/Then — it doesn't validate one runtime behavior, it constrains every scenario above:

**Minimum Conformance never weakens enforcement.** For every component operating under its minimum-cardinality form in §3, when exercised by a relevant AVS scenario, all Security Boundary (SB), Blast Radius (BR), and Compound Failure (CF) assertions still hold in full. "Minimum" refers only to instance count or tier count — never to skipped or weakened enforcement. This is the same principle stated in §1's Conformance Principle, restated here as the invariant that governs how §8's scenarios and §10's traceability matrix should be read.

## 10. Conformance Traceability Matrix

For every AVS scenario declared executable under Phase 1 Minimum Conformance, the minimum topology must not weaken its trigger, expected behavior, or pass criterion (§9).

| Minimum choice | Mandatory AVS scenarios (must pass, unweakened) | Deferred AVS scenarios | Deferral authority |
|---|---|---|---|
| Single Tool/Skill Registry | AVS-CF-003, AVS-BR-002, AVS-BR-003 | Multi-file entry-conflict scenarios | Not yet phase-assigned (§5) |
| Single-process OPA | AVS-PS-001/002/003, AVS-CF-005 | Distributed bundle scenarios | Not yet phase-assigned (§5) |
| Single Postgres instance | AVS-BR-001, AVS-CF-001 | Replication/sharding scenarios | Not yet phase-assigned (§5) |
| Deterministic evaluation only | AVS-BR-004 | LLM/Agent-as-Judge scenarios | §6.1 trigger |
| Circuit Breaker, both scopes | AVS-CB-001, AVS-CB-002, AVS-CB-003 | None — no longer deferred | Appendix A |
| No A2A | AVS-MA-003 | AVS-MA-001 | Phase 3 (frozen) |

## 11. Scenario ID and AVS-relationship convention

`S5-###` scenario IDs are stable and use a separate namespace from `AVS-[GROUP]-[NNN]` — these are implementation-spec scenarios, not architecture-verification scenarios. Two relationship tags are used, and they are not interchangeable:

- **`verifies:`** — the S5 scenario actually reproduces the AVS scenario's trigger and asserts its pass criterion. Reserved for cases where this is literally true (e.g., S5-006).
- **`related_avs:`** — the S5 scenario establishes the minimum topology an AVS scenario runs against, or is enabled by/constrains it, without itself reproducing the AVS trigger. This is the more common relationship in this document — most S5 scenarios describe topology, not failure-mode execution.

## 12. What this spec explicitly does NOT do

- Does not change any frozen architectural decision (ADR-001 through ADR-007 stand unmodified)
- Does not remove A2A from the architecture — Card 06's threat model coverage for A2A remains valid, just unexecuted until Phase 3
- Does not reduce which AVS scenarios must eventually pass — it sequences *when* higher-cardinality scenarios (e.g., multi-file registry entry conflicts) become testable, deferring them the same way S1/S2/S3 already defer their own AVS categories per the Verification Coverage Scope note
- Does not exempt any Tier 1–8 item in the Phase 1 build order from being built — every item still gets built, just at minimum scale first (except Circuit Breaker, which is now full-scope per Appendix A)
- Does not commit this project to any Phase 2/3 roadmap beyond A2A's already-frozen Phase 3 deferral — §5's "Not yet phase-assigned" rows are deliberate, not an oversight
- Does not silently override the External Stage 10 review — Appendix A documents the supersession explicitly, with both reasons named, rather than quietly picking one interpretation

---

## Appendix A — Design Rationale: Circuit Breaker Scope Supersedes External Recommendation 6

The External Stage 10 review's Recommendation 6 proposed deferring system-level Circuit Breaker aggregation past Phase 1. That recommendation is **not adopted**, for two independent reasons discovered while drafting this spec:

1. **AVS-CB-002 (System-Level Circuit Breaker Trip) is frozen at `[PHASE 1]` execution** in the live Architecture Verification Specification — a later Phase 0 artifact than the External review, and one that was never reconciled against Recommendation 6 before both were frozen. Adopting the deferral would put S5 in direct conflict with an already-frozen AVS scenario.
2. **Card 06 §11 (line 278) establishes run-level and system-level Circuit Breaker as one configurable module** — scope (run/agent/tool/system) is a runtime parameter read from Policy Server configuration, not a structural choice between two separate implementations. There was never a clean way to build "just run-level" without either hardcoding the scope parameter (which the card forbids) or building the full configurable mechanism anyway. Deferral was never architecturally achievable, independent of the AVS conflict.

**Resolution:** both scopes are built together, as one module, in Phase 1. This is a heavier Phase 1 lift than the External review anticipated, and that tradeoff is accepted deliberately (capex-over-opex: build it correctly once now rather than defer into a false economy that the architecture doesn't actually support). The aggregation mechanism this requires is specified separately in **S7 (Circuit Breaker Aggregation and Threshold Configuration)**, approved 2026-07-18. This is not a case of S5 silently overriding a frozen document — it's S5 documenting which of two conflicting Phase 0 artifacts governs, and why, per Working Principle #13 (cross-check new specs against all frozen governance documents, not just the cards).

---

*This spec gates the Phase 1 build order per Master Plan item 1.1. Status flips from NOT STARTED to APPROVED as of this commit.*