---
spec_id: S7
title: Circuit Breaker Aggregation and Threshold Configuration
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-18
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 01 §2, Card 06 §3, Card 06 §8, Card 06 §9, Card 06 §11, Card 06 §18, ADR-007, tech-stack.md Decision #3]
depends_on: [S1, S6]
gates: [Circuit Breaker implementation, within Tier 3 Security Foundation — exact Master Plan item number pending the Step 4 build-order diff]
source: identified as a genuine gap while drafting S5, when AVS-CB-002's frozen [PHASE 1] tag was found to conflict with the External Stage 10 review's Recommendation 6 (defer system-level Circuit Breaker); Card 06 §11 line 278 confirmed run-level and system-level are one configurable module, not separable builds
added_late: true — this is the 7th Mandatory Pre-Build Spec; S1-S6 were the original set defined in stage-10-reconciliation.md
---

# S7 — Circuit Breaker Aggregation and Threshold Configuration

**Status:** 1.0 (Approved)

## 1. Purpose

Card 06 §11 defines the Circuit Breaker's *behavior* — scope can be run/agent/tool/system-level, thresholds and release criteria are Policy Server configuration, a single high-severity event can trip immediately while repeated low-severity events accumulate toward a trip. What it does not define is the *mechanism*: how events actually get counted and aggregated across concurrently running agents to determine when the system-level threshold (AVS-CB-002) is crossed.

This spec closes that gap, narrowly. It defines the counting mechanism for a **single-process, single-Postgres-instance deployment** — the Minimum Conformance Profile (S5) — not a distributed or multi-node solution. The harder version of this problem (cross-node coordination) is already correctly identified elsewhere as a future concern, triggered by multi-node deployment, not by Phase 1.

## 2. Non-Goals

This specification does not define:

- multi-node or distributed event coordination
- a new message queue, pub/sub system, or in-memory cache (e.g., Redis) — `tech-stack.md`'s Message Queue decision is deferred until a genuinely asynchronous workload requires it, and introducing one here would violate that standing decision and the zero-cost/lean-dependency principle
- threshold *values* (what count, what time window) — those are Policy Server configuration per Card 06 §11, set at runtime, not fixed by this spec. This spec requires only that Policy Server implementations support externally configurable thresholds — it asserts no default numbers.
- the Policy Server's own decision logic for mapping evaluation signals to actions (Card 06 §8) — this spec covers counting/aggregation only, not the authorization decision that consumes the count
- **severity classification** (what makes an event "low" vs. "high" severity). Card 06 §8/§9 establishes severity as a real input to the Policy Server's risk-tier + severity + confidence model, and line 241's reference to "§9's severity-mapping logic" implies a formal taxonomy was intended — but no concrete severity scale exists anywhere in the frozen cards (unlike risk tier, which has an explicit table in Card 06 §3). This is a genuine open gap, not something already defined elsewhere that S7 can simply cite. It should not be defined here, since S7 is a mechanism spec, not a classification spec — it belongs as its own tracked item against Card 06 §9. S7 consumes whatever severity classification eventually gets defined there; it does not invent one to unblock itself.

## 3. Ownership of the Breaker-Candidate Classification

**The Policy Server determines whether an event is a breaker candidate.** This follows directly from Card 06 §8: the Policy Server is what maps an evaluation signal to an action (revoke a JIT token, trigger the Circuit Breaker, escalate to HITL) using risk tier + severity + confidence together. S7 consumes the resulting classification as it already exists on the written Security Event record — it never computes candidate status itself, never re-derives severity, and never second-guesses the Policy Server's classification. This closes the ownership loop completely: Policy Server classifies, Security Event Schema carries the classification, S7 counts only records already carrying it.

## 4. Definitions

**Active run:** a run is active unless it has reached the `Stop` Decide outcome (Card 01 §2 — terminal) or is currently in Circuit Breaker quarantine (Card 06 §11 — halted pending Tier 4+ release). A run in `Continue`, `Retry`, or `Escalate` state (Card 01 §2's four Decide outcomes) is active — `Escalate` in particular is not terminal; a run awaiting HITL response can still resume and remains active during that wait.

**Wall-clock time:** measured from commit timestamp (defined next), not client-generated event-creation time or ingestion time.

**Deterministic (as used in §5's concurrency guarantee):** given the same ordered sequence of committed events, threshold evaluation always produces the same trip decision. Determinism is defined relative to commit order, not wall-clock arrival order at any particular evaluator instance.

## 5. Mechanism

Circuit-breaker-relevant events are already written to the Security Event Schema (Card 06 §11) as part of the Security Event pipeline, extending Card 05's observability infrastructure and backed by the single Postgres instance (S5 §3, tech-stack.md Decision #3). No new datastore, cache, or message system is introduced.

- **Storage:** circuit-breaker-relevant security events (fields per Card 06 §11's schema: `risk_tier`, `circuit_breaker_state`, `policy_decision_id`, etc.) are written to Postgres as part of the same write path S6 already defines for audit/observability events — not a separate table with separate write semantics.
- **Which events participate:** only committed security events already carrying a circuit-breaker-candidate classification from the Policy Server (§3) enter the aggregation count. Aggregation operates only on committed security events — deleted, rolled-back, or quarantined-draft events never participate. Not every security event is a candidate; routine allowed/denied policy decisions that don't cross into Safety/Alignment violation territory (per §8) do not count toward the threshold.
- **Visibility guarantee (replaces a specific transaction-implementation requirement):** an event MUST NOT become visible to downstream aggregation consumers unless threshold evaluation against it has completed successfully. Equivalent implementations (transactional write, write-then-confirm, or any other mechanism) must preserve this guarantee — the requirement is the outcome, not a specific database mechanism.
- **Concurrency guarantees:** under concurrent writers, exactly one evaluation occurs per committed event; no duplicate system-level trips result from concurrent events crossing the threshold near-simultaneously; behavior is deterministic per §4's definition.
- **Timestamp source:** aggregation window boundaries are evaluated against commit timestamp (§4), not client-generated event-creation timestamp or ingestion timestamp — this avoids clock-skew issues between whatever produced the event and the aggregation logic, and keeps the window's meaning tied to what the system actually observed.
- **Window semantics:** the sliding window is **rolling**, not tumbling — evaluated continuously as the interval `(now-W, now]` — exclusive of events committed exactly W-and-older, inclusive of events committed within the last W up to and including now. This avoids the pathological case where events cluster just before and after a tumbling window's reset point and never trigger despite being close together in real time.
- **Counting:** system-level threshold evaluation is a query against that table, scoped to the Policy Server-configured rolling window `(now-W, now]`, counting only breaker-candidate low-severity events across all runs (not scoped to one run — that's what makes it "system" rather than "run" level). Queries are indexed and bounded to the configured window — never a full-table scan of complete event history.
- **Evaluation trigger:** the count is evaluated synchronously at each new low-severity, breaker-candidate security event write, not via a separate polling job — this avoids introducing a scheduler dependency (already a Non-Goal per the Master Plan's own deferred Scheduler item) and keeps trip detection immediate rather than batched.
- **Idempotency and recovery:** once a system-level quarantine is active, additional qualifying events do not create repeated trip transitions or duplicate quarantine records. While a quarantine remains active, additional qualifying events extend the audit trail but never create nested quarantines. The existing quarantine state is authoritative until released per the Tier 4+ approval discipline (§6 below).
- **Run-level (AVS-CB-001)** requires no aggregation at all — a single high-severity event is sufficient by itself, evaluated in-line at write time with no counting logic involved.
- **Threshold configuration** (the count and window values themselves) is read from live Policy Server configuration at evaluation time, per Card 06 §11 — this spec defines how the count is produced, not what the configured number is, and asserts no default.

## 6. Ownership Boundaries

- **S7 owns:** the transition *into* quarantine — detecting that a threshold (run-level or system-level) has been crossed, and emitting the quarantine event.
- **Card 06 §3/§11's existing HITL approval mechanism owns:** the transition *out of* quarantine — Tier 4+ approval is required to release, regardless of scope. S7 does not define or implement release logic.
- **S2 (Agent Identity, Signing Key, JIT Credential Lifecycle) owns:** actual credential/JIT-token revocation, performed *after* S7 emits the quarantine event (Card 06 §11: "revoke credentials... invalidate JIT tokens, §18"). S7 decides that a trip occurred; S2 is the mechanism that acts on that decision to invalidate tokens. S7 does not itself revoke anything.

## 7. Scope

| Concern | In scope for S7 | Out of scope |
|---|---|---|
| Run-level trip (single event) | Yes — trivial case, no counting needed | — |
| System-level trip (accumulated events) | Yes — the core mechanism this spec defines | — |
| Breaker-candidate classification ownership | Yes — S7 consumes it (§3); does not compute it | Classification logic itself is Policy Server's (Card 06 §8/§9) |
| Threshold value configuration | No — Policy Server's job (Card 06 §11) | Deferred to Policy Server config, not this spec |
| Cross-node event coordination | No | Deferred to a future spec, triggered by multi-node deployment adoption (per `stage-10-reconciliation.md`'s deferred-items table) |
| Quarantine release approval flow | No — Tier 4+ HITL approval mechanics are Card 06 §3/§11's existing approval discipline | Referenced (AVS-CB-003), not defined here |
| Credential/JIT-token revocation | No — S2's mechanism, triggered by S7's decision | S7 emits the event; S2 acts on it |
| Severity classification | No | Genuine open gap against Card 06 §9 (§2 above) |

## 8. Normative Language

- **MUST / MUST NOT** — required for conformance (e.g., an event MUST NOT become visible to aggregation before threshold evaluation completes)
- **SHOULD / SHOULD NOT** — recommended unless a documented exception exists
- **MAY** — optional, implementer's discretion within the spec's constraints

## 9. Architectural Invariants

Design constraints this spec establishes, distinct from the executable scenarios in §10 — these validate architecture, not runtime behavior, and aren't independently testable as Given/When/Then:

- No new datastore, cache, or message queue is introduced for circuit-breaker aggregation. The single Postgres instance defined by S5's Minimum Conformance Profile is reused, satisfying the zero-cost/lean-dependency principle and the Master Plan's standing Message Queue deferral.
- Failure guarantees: aggregation is fail-closed (an aggregation-evaluation failure does not silently permit continued operation as if no threshold had been crossed), no committed event is ever missed by the count, and no sequence of committed events produces more than one active trip for the same quarantine scope.
- While a quarantine remains active, no sequence of additional qualifying events can produce a nested or duplicate quarantine for the same scope — this is the invariant-level statement of §5's idempotency requirement.

## 10. Scenarios (Given/When/Then)

```gherkin
Scenario: S7-001 — A single high-severity event trips the run-level breaker with no aggregation
  Given an agent run is active (per §4's definition)
  When a single high-severity security event is written
  Then the run-level Circuit Breaker trips immediately, evaluated at write time
  And no counting or time-window logic is invoked — a single event is sufficient
  And no further privileged actions (tool calls, memory writes) succeed after the trip
  And a run-level quarantine record is written to the Security Event Schema
  # verifies: AVS-CB-001
  # evidence:
  #   - security_event_record
  #   - otel_trace

Scenario: S7-002 — Accumulated low-severity breaker-candidate events within the configured window trip the system-level breaker
  Given the Policy Server has configured a system-level threshold of N events within rolling window W
  And fewer than N committed, breaker-candidate low-severity events have been written within (now-W, now]
  When an additional committed, breaker-candidate low-severity security event is written, bringing the count to N within (now-W, now]
  Then the system-level Circuit Breaker trips
  And all active runs (per §4) are halted, not just the run that produced the Nth event
  And only committed events participate in the count — an event that failed to persist does not count
  # verifies: AVS-CB-002
  # evidence:
  #   - security_event_record
  #   - policy_decision_record
  #   - otel_trace

Scenario: S7-003 — Events outside the rolling window, and events exactly at the window boundary, are handled correctly
  Given the Policy Server has configured a system-level threshold of N events within rolling window W
  And N-1 low-severity breaker-candidate events were written, with one falling outside window W relative to now
  When commit timestamps are evaluated against the current rolling window (now-W, now]
  Then only events within (now-W, now] count toward the threshold
  And the system-level breaker does not trip
  And an event whose commit timestamp equals exactly now-W is excluded (the window is exclusive at its lower bound, per §5's interval definition)
  # verifies: AVS-CB-002 (negative case and boundary case — confirms the window is enforced precisely, not just approximately)
  # evidence:
  #   - security_event_record

Scenario: S7-004 — Threshold evaluation reads live Policy Server configuration, not a cached or hardcoded value
  Given a system-level threshold is configured at N events
  When the Policy Server's configuration is updated to a new threshold N' before the next evaluation
  Then the next event-count evaluation uses N', not the previously configured N
  And this satisfies Card 06 §11's requirement that thresholds are "Policy Server configuration, not hardcoded"
  # related_avs: AVS-PS-001, AVS-PS-002 (Policy Server configuration scenarios — this scenario confirms S7 reads from that mechanism correctly, it does not itself verify Policy Server behavior)
  # evidence:
  #   - policy_decision_record

Scenario: S7-005 — A system already in system-level quarantine does not re-trigger on additional qualifying events
  Given a system-level Circuit Breaker trip is currently active (per S7-002)
  When another breaker-candidate low-severity event is written that would otherwise qualify toward a new trip
  Then no duplicate trip transition occurs
  And no duplicate or nested quarantine record is created (per §9's invariant)
  And the event is still recorded in the Security Event Schema, extending the audit trail
  And the existing quarantine remains authoritative until released per the Tier 4+ approval discipline
  # verifies: AVS-CB-002 (idempotency case)
  # evidence:
  #   - security_event_record
  #   - policy_decision_record

Scenario: S7-006 — System-level quarantine release still requires Tier 4+ approval
  Given a system-level Circuit Breaker trip has occurred per S7-002
  When a release is attempted
  Then release requires the same Tier 4+ HITL approval discipline as any other high-risk action (Card 06 §3/§11)
  And this scenario is owned by the existing approval mechanism (§6 above), not by S7's counting logic
  # verifies: AVS-CB-003
  # evidence:
  #   - policy_decision_record
  #   - reviewer_id
  #   - attestation_id
```

## 11. What this spec explicitly does NOT do

- Does not introduce any new infrastructure dependency (Redis, message queue, scheduler) — reuses the already-frozen single Postgres instance and existing Security Event Schema write path
- Does not define threshold values — those remain Policy Server configuration, set at runtime, per Card 06 §11
- Does not define severity classification — that's a genuine open gap against Card 06 §9, tracked separately (§2), not solved here
- Does not compute breaker-candidate classification — consumes it from the Policy Server (§3), never derives it independently
- Does not solve multi-node event coordination — that problem is correctly deferred elsewhere, triggered by multi-node deployment adoption, not by this spec's existence
- Does not change Card 06 §11's architecture in any way — it fills in a mechanism the card intentionally left as implementation detail
- Does not implement quarantine release or credential revocation — those belong to Card 06's existing approval mechanism and S2, respectively (§6)