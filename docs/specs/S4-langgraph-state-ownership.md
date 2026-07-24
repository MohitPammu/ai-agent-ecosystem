---
spec_id: S4
title: LangGraph State Ownership, Reducers, and Concurrency
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-24
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 01 §1, Card 01 §2, Card 02 §4, Card 03 §4, agent-contract-template.md §7, tech-stack.md Decision #4]
depends_on: [S5]
gates: [1.12, 1.13]
source: identified as a genuine gap during the AVS citation walk — AVS-MA-002 cited "tech-stack.md, LangGraph reducer note" and "Card 03 §4's Implementation Note — reducers are defined per state key," neither of which exists; this spec is the real source both citations should have pointed to
avs_relationship_convention: follows S5 §11's project-wide verifies/related_avs convention — not redefined here
---

# S4 — LangGraph State Ownership, Reducers, and Concurrency

**Status:** 1.0 (Approved)

## 1. Purpose

Card 01 §1 defines `state` as "the authoritative record of an agent's current execution context and progress through the execution loop," and explicitly defers implementation detail to Card 03. The Agent Contract template (§7) requires every agent to declare `state_owned` (which keys it may write) and `state_read_only` (which keys it may read but never mutate) — but nothing in the frozen architecture defines the *mechanism* that turns those declarations into an enforced runtime behavior, rather than a documentation-only convention an agent could still violate in code.

This spec closes that gap. It defines: (1) the reducer contract every `state_owned` key must have, (2) why that contract is defined now even though Phase 1 has no scenario that produces genuine concurrent writes, and (3) what stays deferred to Phase 3, when AVS-MA-002's actual concurrent-write race scenario becomes executable.

## 2. Non-Goals

This specification does not define:

- multi-writer conflict-resolution or arbitration logic for genuinely concurrent writes from independent agents — that requires A2A, which is deferred to Phase 3 per S5 Appendix A, and is covered by AVS-MA-002 (execution tag `[PHASE 3+]`, confirmed correctly assigned — not a conflict like the Circuit Breaker case)
- a new state-management framework or abstraction layer on top of LangGraph — this spec adopts LangGraph's existing native mechanism, it does not invent one
- memory system design (Card 03's job) — this spec is about in-run execution state (Card 01 §1's `state`), not persistent memory (episodic/semantic/procedural/preference/Structured Records)
- checkpointer backend selection — already frozen (tech-stack.md Decision #4, `PostgresSaver`)
- tool-call idempotency and completion tracking — already Card 02 §4's territory; S4 does not redescribe it, only relies on it (§3)

## 3. Mechanism

LangGraph state schemas declare each key with a type and an associated reducer function invoked by LangGraph to merge an update into existing state for that key, rather than defaulting to an implicit overwrite. This is native LangGraph behavior (`Annotated[type, reducer_fn]` in the state schema), not a bespoke mechanism this project is inventing.

**Definition — reducer:** the sole authoritative function responsible for transforming the current value of a state key into its next value. A reducer owns state *transitions*, not merely merging two values. **Reducers MUST be pure state-transition functions — no external side effects.** This is what makes the idempotence guarantee below meaningful: a pure reducer can be safely re-applied because it only ever transforms state, never triggers an action a second time. Idempotency of externally observable actions (a tool call, a write to an external system) is a distinct concern, already owned by Card 02 §4's retry/idempotency discipline, and out of scope here (§2).

**Definition — update identity, for replay purposes:** an update's identity is tied to the PostgresSaver checkpoint transaction that persisted it (tech-stack.md Decision #4). "The same update" being replayed means: the update(s) captured in the last confirmed checkpoint are re-applied deterministically from that checkpoint's persisted pre-image, not re-derived by re-running arbitrary upstream logic. This is what AVS-BR-005's crash-recovery replay actually replays — the checkpoint content, not a freshly recomputed equivalent.

- **Every `state_owned` key declared in an Agent Contract (§7) MUST have an explicit reducer** at state-schema-definition time. There is no implicit "last write wins" default — an undeclared reducer is a schema error, not a silent behavior.
- **A state key MUST have exactly one reducer**, and **exactly one Agent Contract may declare a given key under `state_owned`.** Two contracts both claiming ownership of the same key is a contract validation error, not a case the reducer needs to arbitrate — ownership collisions are prevented before the schema is ever defined, not resolved at runtime.
- **Every reducer is declared in exactly one authoritative state schema.** The schema — not individual agents — is the source of truth for which reducer governs which key. This reinforces the separation between contract ownership (§7's declaration of *who* may write) and runtime implementation (the schema's declaration of *how* the write is applied).
- **Reducers are deterministic.** Determinism concerns ordered sequences of updates: given the same ordered sequence of updates to a key, the reducer always produces the same final value.
- **Reducers are idempotent-safe.** Idempotence concerns replay of identical updates (per the update-identity definition above), a distinct property from determinism: applying the same committed update more than once MUST produce the same final state as applying it exactly once. Because reducers are pure (no side effects), this guarantee is about state alone — it does not need to account for external actions being duplicated, since a pure reducer cannot duplicate one.
- **`state_read_only` keys have no reducer requirement** for the reading agent, since it never writes them — but per the "exactly one reducer" rule above, the key still has exactly one reducer overall, owned by whichever agent's contract declares it under `state_owned`.
- **Reducer assignment is per-key, not per-agent.** An agent's contract declares which keys it owns; the reducer for each owned key is a property of the key in the state schema, written once when the schema is defined, not re-derived per agent.

## 4. Why Reducers Are Phase 1

Two independent reasons this can't wait for Phase 3, even though the adversarial race test (AVS-MA-002) correctly can:

1. **Interface stability.** If Phase 1 agents get built against an undeclared/implicit merge behavior, and Phase 3 later needs real per-key reducer semantics for concurrent writes, every Phase 1 Agent Contract's `state_owned` declarations would need revisiting. Declaring the reducer contract now costs little and forecloses that rework.
2. **It already benefits Phase 1, independent of concurrency.** A well-defined, idempotent reducer makes crash-recovery replay (AVS-BR-005) and ordinary retries safer even under single-writer conditions — this isn't speculative complexity for a scenario that can't occur yet, it's a mechanism Phase 1 already needs for a different reason.

**The Phase 1 single-writer assumption, stated explicitly:** under the Phase 1 execution model, each state key has at most one active writer during execution — single-agent and Coordinator-delegated sub-agent patterns only, no A2A (S5 Appendix A). This is precisely why multi-writer arbitration is deferred rather than built now: it can't even be exercised under Phase 1's architecture, and building it now would be speculative. What stays out: any actual multi-writer arbitration, locking, or ordering guarantee under true concurrency.

## 5. Scope

| Concern | In scope for S4 | Out of scope |
|---|---|---|
| Reducer required per `state_owned` key, exactly one per key | Yes | — |
| Ownership-collision prevention (one contract per key) | Yes | — |
| Deterministic, idempotent-safe, pure (no side effects) reducer semantics | Yes | — |
| Crash-recovery replay safety and update identity (AVS-BR-005 interaction) | Yes | — |
| Multi-writer conflict arbitration under true concurrency | No | AVS-MA-002, correctly deferred to Phase 3 (requires A2A) |
| Checkpointer backend | No | Already frozen, tech-stack.md Decision #4 |
| Tool-call idempotency / external side-effect deduplication | No | Card 02 §4's territory |
| Persistent memory (episodic/semantic/procedural/preference/Structured Records) | No | Card 03's job |

## 6. Ownership Boundaries

- **Agent Contract (§7)** owns declaring *which* keys an agent may write (`state_owned`) or read-only access (`state_read_only`) — a per-agent, per-instantiation decision, and now explicitly exclusive per key (§3).
- **S4 owns** the requirement that every declared `state_owned` key has a reducer, and the reducer's semantic properties (deterministic, idempotent-safe, pure, exactly-one-per-key) — a schema-level, cross-agent requirement.
- **`core/harness/` (1.13)** owns the actual state schema implementation — writing the reducer functions themselves, per key, according to each agent's declared `state_owned` scope, not designed generically and reconciled with contracts after the fact (per the Master Plan's existing Deferred Enhancement note on this exact point). Harness also owns tool-call idempotency (Card 02 §4), kept distinct from reducer purity (§3).

## 7. Architectural Invariant

No `state_owned` key **MAY** exist in a Phase 1 state schema without an explicit, declared reducer. Every `state_owned` key **MUST** have exactly one reducer, and **MUST** be declared by exactly one Agent Contract. Reducer input and output types **MUST** match the key's declared type. Validation occurs before any graph execution begins — this is not independently testable as one Given/When/Then, it constrains every scenario below, and would be checked as a schema-validation step at Harness initialization, not a runtime behavior encountered mid-execution.

## 8. Scenarios (Given/When/Then)

```gherkin
Scenario: S4-001 — Every state_owned key has a declared reducer at schema-definition time
  Given an Agent Contract declares a set of state_owned keys (§7)
  When the corresponding LangGraph state schema is defined in core/harness/
  Then every declared state_owned key has an explicit reducer function
  And reducer registration succeeds for every key before the Harness accepts any work
  And no key relies on implicit overwrite behavior
  # related_avs: AVS-MA-002 (establishes the mechanism the deferred race scenario will eventually exercise)

Scenario: S4-002 — Reducers are idempotent-safe under crash-recovery replay
  Given a state key's reducer has already been applied to a committed update, identified by its PostgresSaver checkpoint transaction (§3)
  When the Harness restarts after an involuntary crash and replays from the last persisted checkpoint (AVS-BR-005)
  Then re-applying that same checkpointed update through the reducer produces the same final state as applying it exactly once
  And no externally observable action is duplicated, because the reducer itself performs no side effects (§3) — any tool-call-level duplication is governed separately by Card 02 §4, not by this scenario
  # related_avs: AVS-BR-005

Scenario: S4-003 — Reducers are deterministic under repeated execution
  Given the same ordered sequence of updates to a state_owned key (per §3's determinism definition)
  When those updates are applied through the key's reducer, in any process run
  Then the final value is identical across runs
  # related_avs: AVS-MA-002 (this is the same determinism property the deferred race scenario will check under true concurrency; this scenario checks it under Phase 1's single-writer conditions per §4)

Scenario: S4-004 — An undeclared reducer, type mismatch, or ownership collision fails Harness initialization, not silently at runtime
  Given a state schema is being validated at Harness startup, and at least one of the following holds: (a) a state_owned key has no corresponding reducer, (b) a reducer's input or output type does not match its key's declared type, or (c) two Agent Contracts declare the same key under state_owned
  When core/harness/ validates the schema at startup
  Then Harness initialization fails before accepting any work, with an explicit error identifying which of (a)/(b)/(c) occurred and for which key
  And the Harness does not start with an implicit "last write wins" fallback, a type-coerced reducer, or an arbitrarily-chosen owning contract in effect
  # This is a Phase 1 safety net — it turns silent gaps into loud ones, caught before execution rather than during it.

Scenario: S4-005 — state_read_only access does not require a reducer for the reading agent
  Given an agent's contract declares a key under state_read_only
  When that agent reads the key
  Then no reducer is invoked, since the agent performs no write
  And the key's single owning reducer (declared by the one Agent Contract that lists it under state_owned, per §3's exactly-one-reducer and exactly-one-owner rules) remains the only mutation path
```

## 9. What this spec explicitly does NOT do

- Does not implement multi-writer conflict resolution for genuine concurrent writes — that's AVS-MA-002, correctly deferred to Phase 3, gated on A2A
- Does not invent a new state-management mechanism — adopts LangGraph's native per-key reducer pattern
- Does not touch memory system design, checkpointer backend selection, or persistent memory types — all already frozen or Card 03's territory
- Does not weaken Agent Contract §7's ownership declarations — it's the enforcement mechanism for them, not a replacement, and now explicitly forecloses ownership collisions between contracts
- Does not redescribe tool-call idempotency or external side-effect deduplication — that's Card 02 §4's territory; S4's reducers are pure by definition, so they never need to
- Does not redefine the project's AVS relationship tagging convention — follows S5 §11's existing `verifies`/`related_avs` distinction