---
spec_id: S6
title: Observability and Audit Delivery Reliability
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-30
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 05 §5, Card 06 §11, Card 06 §12, Card 06 §20, tech-stack.md Decision #5, AVS-CF-004, AVS-OB-001, AVS-OB-002]
depends_on: [S5]
gates: [core/observability/ implementation, Tier 2 Observability Foundation — Master Plan item 1.3 (spec) / 1.4 (implementation)]
source: stage-10-reconciliation.md — added as a mandatory pre-build spec (originally missing from S1-S5) after AVS-CF-004 was found to require a fallback audit buffer mechanism that no card defines; Master Plan Deferred item "Observability and Audit Delivery Reliability Spec mandatory-list flagging," resolved 2026-07-25 by adding this spec to the mandatory list
---

# S6 — Observability and Audit Delivery Reliability

**Status:** 1.0 (Approved)

## 1. Purpose

Card 05 §5 defines what observability captures (Run ID, Agent ID, Mission, tools used, cost, latency, errors, state transitions, trace/span IDs, HITL events) and Card 06 §11 extends this with a Security Event Schema for security-specific events (policy denials, Circuit Breaker trips, integrity violations). tech-stack.md Decision #5 selects OpenTelemetry + self-hosted Jaeger as the backend that receives this data. None of these define what happens when that backend is unreachable. AVS-CF-004 names the concrete failure this spec must close: the Jaeger backend becomes unavailable while a Policy denial and Circuit Breaker trip occur simultaneously — security enforcement must still happen (it does not depend on observability, per Card 05 §5's existing one-directional-dependency precedent), but the security events themselves must not be permanently lost. This spec defines the fallback audit buffer that makes that guarantee concrete: what it is, its capacity behavior, how buffered events are retried and deduplicated once the backend recovers, and how reconciliation confirms nothing was lost.

## 2. Non-Goals

- Which fields observability captures — Card 05 §5's field list and Card 06 §11's Security Event Schema are consumed as given, not redefined here
- Retention duration/policy for audit records once durably delivered — Card 06 §12 ("set per-project") already governs this; S6 is about delivery reliability en route to the backend, not how long records are kept afterward
- The Evaluation-service blast radius (Card 05 §5's existing "Evaluation never blocks the Harness" precedent) — a separate, already-specified concern; S6 addresses the observability/audit *backend*, not the evaluation *consumer*
- Multi-node/distributed audit infrastructure — out of scope per S5's single-process Minimum Conformance Profile
- Redefining the Security Event Schema's field list, redaction rules, or append-only/tamper-evident properties — Card 06 §11/§12's territory; S6 assumes events already carry these properties correctly and focuses on their delivery path
- Policy Server fail-closed behavior itself (Card 06 §20's table) — S6 references it as a precedent for one specific edge case (§4) but does not redefine it

## 3. Definitions

**Primary observability path:** the normal case — OpenTelemetry SDK instrumentation delivers trace spans and events directly to the self-hosted Jaeger backend (tech-stack.md Decision #5) as they occur.

**Fallback audit buffer:** a durable, local, disk-backed queue that captures observability and security events when the primary path is unavailable. It exists specifically so that unavailability of the Jaeger backend does not mean event loss — it is a delivery-reliability mechanism, not a second permanent store; buffered events are always destined for the primary path once it recovers, not retained indefinitely in the buffer itself. The buffer's contents are append-only and ordered by capture time, and, being audit-destined data, tamper-evident — the same properties Card 06 §12 already requires of the audit trail generally, extended here to data in transit toward it, not a new property invented for this spec. Per S5's single-process Minimum Conformance Profile, exactly one buffer instance exists; this spec does not define a `buffer_id` or similar multi-instance identity, since there is nothing to disambiguate.

**Fallback buffer lifecycle:** `Idle` (empty, primary path healthy, no fallback writes occurring) → `Buffering` (primary path write failures are occurring; events are being durably queued) → `Reconciling` (primary path has recovered; backlog replay is in progress) → `Idle` again once the backlog is fully drained and confirmed. `Buffering` and `Reconciling` are not mutually exclusive instants — a new primary-path failure during `Reconciling` returns the buffer to `Buffering` for new events while the in-progress backlog replay continues under §4's interrupted-reconciliation rule.

**Reconciliation session:** one attempt at draining the buffer's backlog, identified by a `reconciliation_session_id`, spanning from the moment the primary path is detected reachable again to the moment the backlog is fully drained. If the session is interrupted by another primary-path failure and later resumed (§4), the resumed replay continues under the **same** `reconciliation_session_id` — a new session ID is issued only when reconciliation begins after having fully drained to `Idle`, not merely after an interruption. This identity exists to make an interrupted-and-resumed reconciliation attempt traceable as one continuous effort across the interruption, not to disambiguate multiple buffers.

**Buffered event:** any observability or security event (per Card 05 §5 / Card 06 §11's respective field sets) written to the fallback buffer because the primary path's write failed or the primary path was already known unavailable at the time the event was generated. Every buffered event carries a stable `event_id`, generated at the point of capture, that uniquely identifies it independent of how many times delivery is attempted. A buffered event's lifecycle is `Captured` (generated by the instrumented code) → `Buffered` (durably written to the fallback queue, with the buffering itself logged per §4) → `Replaying` (part of an active reconciliation session's in-flight delivery attempt) → `Delivered` (durable receipt confirmed at the primary path) → `Removed` (cleared from the buffer, only after `Delivered`). There is no state between `Replaying` and `Delivered` that is itself removable — an event is never cleared from the buffer on the strength of a send attempt alone.

**Reconciliation:** the process of replaying the fallback buffer's backlog into the primary path once it becomes reachable again, confirming each event's durable receipt, and only then removing it from the buffer.

## 4. Mechanism

**Failure detection and buffering.** When a write to the primary observability path fails or times out, the event is written to the fallback audit buffer instead — this is the write path's fallback behavior, not a separate parallel write. Detection of primary-path unavailability does not require the whole system to know it's "in fallback mode" as a distinct global state; each write attempt independently falls back on its own failure, consistent with Card 05 §5's principle that observability capture itself has no dependency on any downstream consumer's availability. The transition into `Buffering` (§3) is itself logged — auditing an outage's duration requires knowing when buffering started, not only when reconciliation completed.

**Failure classification.** Every failure this spec produces or responds to (backend unreachable, write timeout, replay delivery failure, buffer overflow, acknowledgement failure) is a subcode rolling up to Card 05 §6's existing **Tool failure** category — the primary observability backend is, architecturally, a tool dependency, and Card 05 §6's own example text ("API timeout... tool unavailable") already covers this case directly. Per Card 05 §6's direct implication that all ecosystem error handling consumes this one shared taxonomy, S6 does not define a new failure-category set; it defines the specific subcodes this spec's mechanism produces (e.g., `backend_unreachable`, `write_timeout`, `replay_delivery_failure`, `buffer_capacity_exceeded`, `ack_failure`), all rolling up to Tool failure.

**Security enforcement is unaffected.** Per Card 05 §5's existing precedent (Evaluation never gates the Harness) and AVS-CF-004's expected behavior: Policy Server denials and Circuit Breaker trips execute and their outcomes take effect regardless of whether the resulting audit event is written to the primary path or the fallback buffer. The buffer exists so the *record* of the decision is not lost — it never delays or blocks the decision itself.

**Capacity limits and overflow.** The fallback buffer has a bounded capacity (an operational configuration value, not fixed by this spec). AVS-CF-004's pass criterion requires no security event is ever permanently lost — this means overflow cannot be handled by silently dropping the oldest buffered event. When the buffer reaches capacity: new **non-security observability events** (Card 05 §5's generic fields, including metrics — the lossy-tolerant case Card 05 §5's own Logging/Tracing/Metrics distinction already implies, since metrics are aggregated signals rather than individual audit-relevant records) may be dropped, with the drop itself logged (subcode `buffer_capacity_exceeded`, rolling up to Tool failure) once capacity allows; new **security events** (Card 06 §11's Security Event Schema) are never dropped — if the buffer is genuinely full and cannot accept a new security event, this is treated as a Policy Server availability-equivalent condition and the fail-closed-by-tier table already defined in Card 06 §20 governs the triggering action, exactly as it would if the Policy Server itself were unreachable. This reuses an existing mechanism rather than inventing a second fail-closed table specific to observability.

**Capacity recovery.** Once buffer occupancy drops below capacity (through successful reconciliation or drop-eligible non-security events clearing), the buffer automatically resumes accepting new events — no manual intervention or explicit resume signal is required. Recovery from overflow is symmetric with recovery from a backend outage: both are conditions the buffer exits automatically once the underlying condition (full buffer, unreachable backend) clears.

**Retry, deduplication, and interrupted reconciliation.** Once the primary path is reachable again, a reconciliation session begins (or resumes, per below) and replays buffered events in the order they were originally captured (oldest first). Each replay attempt uses the event's `event_id` for idempotent delivery — if a prior replay attempt partially succeeded (event written to Jaeger but the buffer removal step failed before confirming), a retry of the same `event_id` does not produce a duplicate record at the primary path. This is the same idempotency principle Card 02 §4 already applies to tool calls, applied here to audit delivery instead of tool execution. **If the primary path fails again mid-reconciliation**, the session does not restart from the beginning: it resumes, on the next detected recovery, under the same `reconciliation_session_id` (§3), from the event immediately following the last event confirmed `Delivered` in that session's capture-order sequence — never re-attempting confirmed deliveries, and never skipping unconfirmed ones. Ordering (oldest-first, by capture order — the sequence is not assumed to be numerically derivable from `event_id` itself) is preserved across this kind of interruption and resumption exactly as it is in an uninterrupted session; it is not a special case. The resumption of a session after an interruption is itself a loggable event, distinct from the session's original start, giving operational visibility into how many times a given outage's reconciliation had to resume.

**Reconciliation never mutates buffered event content.** Consistent with the buffer's append-only, tamper-evident properties (§3), reconciliation changes only an event's lifecycle state (Buffered → Replaying → Delivered → Removed, §3) — it never alters the event's captured field content. The content a security or observability event carries at `Delivered` is identical to what it carried at `Captured`.

**Restart survival.** Because the buffer is disk-backed (§3), buffered events survive a process restart — this is stated here as a normative requirement, not merely a description of the storage medium. A process restart while events are in `Buffered` or `Replaying` state does not lose those events; on restart, any event left in `Replaying` (an in-flight, unconfirmed delivery attempt at the moment of restart) is treated as not yet `Delivered` and is retried under the same idempotent-replay rule above, since its delivery status was unconfirmed.

**Reconciliation completion.** An event is removed from the fallback buffer only after its durable receipt at the primary path is confirmed — never optimistically, before confirmation. A reconciliation session is considered complete once every event captured before that session began has either been confirmed `Delivered` or (for non-security events only, per the overflow rule above) confirmed dropped-and-logged. Reconciliation completion itself is a loggable event, giving an auditable record that a given outage's backlog was fully processed.

**Ownership of reconciliation.** Reconciliation is owned by `core/observability/` (Master Plan item 1.4) — it is the same component responsible for the primary write path, not a separate background worker or a Harness responsibility. This keeps the fallback buffer's write and drain paths under one component rather than splitting ownership across the Harness and the observability layer.

## 5. Scope

| Concern | In scope for S6 | Out of scope |
|---|---|---|
| Fallback audit buffer as a first-class object — lifecycle, integrity properties (append-only, ordered, tamper-evident), single-instance identity | Yes | Multi-buffer/multi-instance identity — not applicable under S5's single-process topology |
| Fallback audit buffer existence, durability, and write path | Yes | The primary OpenTelemetry/Jaeger backend itself (tech-stack.md Decision #5, already accepted) |
| Buffered-event lifecycle (Captured/Buffered/Replaying/Delivered/Removed) | Yes | — |
| Capacity limits, overflow behavior, and automatic capacity recovery | Yes | Setting the specific capacity number — operational/implementation detail |
| Retry, idempotent deduplication, interrupted-reconciliation checkpoint/resume | Yes | The tool-call idempotency mechanism itself — Card 02 §4's territory, only referenced here as precedent |
| Reconciliation ordering, session identity, and completion confirmation | Yes | — |
| Restart survival (buffer contents persist across process restart) | Yes | — |
| Failure classification for buffer-related failures | Yes, as subcodes of Card 05 §6's existing Tool failure category | A new failure-category taxonomy — declined; Card 05 §6's own direct implication requires reuse, not invention |
| Reconciliation ownership | Yes — assigned to `core/observability/` | — |
| What fields an event carries | Consumes Card 05 §5 / Card 06 §11 as given | Redefining or extending those field sets |
| Retention duration once delivered | No | Card 06 §12 ("set per-project") |
| Security enforcement continuing during observability outage | Restated as a consequence, not redefined | Card 06 §20's fail-closed table itself, and Policy Server/Circuit Breaker mechanics — S1, S7 |
| Evaluation-service blast radius | No | Card 05 §5 already specifies this fully |

## 6. Ownership Boundaries

- **Card 05 §5** owns what observability captures and the field-level standard. **Card 06 §11/§12** owns the Security Event Schema and audit-trail minimums (append-only, tamper-evident, redaction, retention). **S6** owns neither field content nor retention — it owns what happens to an already-correctly-formed event on its way to durable storage when the primary path is briefly unavailable.
- **Card 06 §20** owns the Policy Server fail-closed-by-tier table. **S6** does not redefine this table — it invokes the same table for the one specific edge case where the fallback buffer itself is exhausted for a security event, rather than inventing a parallel table.
- **S1/S7** own the Policy Server and Circuit Breaker's own decision mechanisms respectively. **S6** owns only the reliable delivery of the audit record those decisions produce — S6 does not affect whether or how those decisions are made.
- **Card 02 §4** owns tool-call idempotency as its own concept. **S6** borrows the same idempotency principle for a different purpose (audit delivery deduplication) without redefining Card 02 §4's actual mechanism.
- **Card 05 §6** owns the ecosystem's one shared Formal Failure Taxonomy. **S6** does not define a competing or parallel taxonomy — every buffer-related failure this spec produces is a subcode of Card 05 §6's existing Tool failure category, per that section's own direct implication that future error handling must reuse it.
- **`core/observability/`** (Master Plan item 1.4) owns reconciliation as an operational responsibility — the same component that owns the primary write path, not a separate service or the Harness.

## 7. Architectural Invariants

- A Policy Server denial or Circuit Breaker trip always executes and takes effect, regardless of the primary observability backend's availability.
- No security event is ever permanently lost — either delivered via the primary path (directly or via reconciliation) or, if the fallback buffer cannot accept it, the triggering action is governed by Card 06 §20's existing fail-closed table.
- Non-security observability events may be dropped only when the fallback buffer is at capacity, and only with the drop itself logged once capacity allows.
- An event is never removed from the fallback buffer before its durable receipt via the primary path is confirmed.
- Reconciliation replay is idempotent per `event_id` — replaying an already-delivered event never produces a duplicate record, whether the retry is caused by a partial-acknowledgement failure, an interrupted-and-resumed session, or a process restart.
- Reconciliation backlog is replayed in original capture order, and this ordering is preserved across interruption, resumption, and process restart alike — not only in the uninterrupted case.
- An interrupted reconciliation session resumes from the event immediately following the highest confirmed `Delivered` event in that session — never from the beginning, and never skipping an unconfirmed event.
- The fallback buffer's contents are append-only, ordered, and tamper-evident, consistent with Card 06 §12's existing audit-trail properties. Reconciliation changes only an event's lifecycle state, never its captured field content.
- Buffered events survive a process restart; an event in `Replaying` state at the moment of restart is treated as unconfirmed and retried.
- All buffer-related failures are classified as subcodes of Card 05 §6's existing Tool failure category — S6 introduces no new failure taxonomy.

## 8. Scenarios (Given/When/Then)

```gherkin
Scenario: S6-001 — Buffering itself is logged when the primary path fails
  Given the Jaeger backend is unreachable
  When an event's primary-path write fails
  Then the event is written to the fallback buffer with a preserved event_id
  And the transition into Buffering is itself logged, giving an auditable start-of-outage marker

Scenario: S6-002 — Security events survive a primary path outage
  Given the Jaeger backend is unreachable
  And a Policy denial and a Circuit Breaker trip occur during the outage
  When each event is generated
  Then the Policy denial and Circuit Breaker trip both execute correctly regardless of backend availability
  And both events are written to the fallback audit buffer with a unique event_id
  verifies: AVS-CF-004

Scenario: S6-003 — Reconciliation delivers the full backlog once the backend recovers
  Given a fallback buffer containing events captured during an outage, in original capture order
  When the Jaeger backend becomes reachable again
  Then reconciliation replays the backlog oldest-first under a new reconciliation_session_id
  And each event is confirmed durably received before being removed from the buffer
  And reconciliation completion is itself logged
  verifies: AVS-CF-004

Scenario: S6-004 — A partially-acknowledged replay does not produce a duplicate record
  Given a buffered event whose prior replay attempt wrote successfully to Jaeger but failed to confirm before the buffer removal step
  When reconciliation retries that same event_id
  Then the retry is idempotent and no duplicate record is created via the primary path
  And the event is now correctly removed from the buffer

Scenario: S6-005 — Interrupted reconciliation resumes after the last confirmed event, not from the beginning
  Given a reconciliation session has confirmed delivery of events 1 through 3 of a 5-event backlog
  When the primary path becomes unreachable again before event 4 is confirmed
  And later becomes reachable again
  Then replay resumes from event 4, the event immediately following the highest confirmed event
  And events 1 through 3 are not re-delivered
  And ordering is preserved across the interruption

Scenario: S6-006 — Buffered events survive a process restart
  Given buffered events exist, including one in Replaying state at the moment of restart
  When the process hosting the fallback buffer restarts
  Then no buffered event is lost
  And the event that was in Replaying state is treated as unconfirmed and retried under the idempotent-replay rule

Scenario: S6-007 — Fallback buffer overflow drops non-security events only, with the drop logged
  Given a fallback buffer at capacity
  When a new non-security observability event cannot be accepted
  Then the event is dropped
  And the drop itself is logged as a distinct event, subcode buffer_capacity_exceeded, once buffer capacity allows

Scenario: S6-008 — Fallback buffer overflow for a security event triggers Card 06 §20's fail-closed table
  Given a fallback buffer at capacity
  When a new security event (Policy denial, Circuit Breaker trip, or integrity violation) cannot be accepted
  Then the triggering action is governed by Card 06 §20's existing fail-closed-by-tier table, exactly as if the Policy Server itself were unreachable
  And the security event is never silently dropped
  related_avs: AVS-CF-004

Scenario: S6-009 — Buffer automatically resumes accepting events once capacity clears
  Given a fallback buffer at capacity, having previously dropped a non-security event
  When occupancy drops below capacity through reconciliation
  Then the buffer resumes accepting new events automatically, with no manual resume signal required
```

## 9. What this spec explicitly does NOT do

- Does not define the observability field standard or the Security Event Schema's field list — Card 05 §5, Card 06 §11
- Does not define audit retention duration or policy — Card 06 §12
- Does not redefine Card 06 §20's fail-closed-by-tier table — it invokes the existing table for one edge case, not a new one
- Does not define the Policy Server's or Circuit Breaker's own decision mechanisms — S1, S7
- Does not define a new failure-category taxonomy — all buffer-related failures are subcodes of Card 05 §6's existing Tool failure category, per that section's own mandate to reuse rather than invent
- Does not define multi-buffer or multi-instance buffer identity — S5's single-process topology means exactly one buffer exists; no `buffer_id` is needed
- Does not specify the fallback buffer's exact capacity number, storage technology, or serialization format — implementation-defined
- Does not cover the fuller audit-durability scenario set (trace/audit clock synchronization, A2A parent correlation, audit access logging) beyond what §8 includes — explicitly tracked in the Master Plan Deferred list as post-S6 verification-spec expansion
