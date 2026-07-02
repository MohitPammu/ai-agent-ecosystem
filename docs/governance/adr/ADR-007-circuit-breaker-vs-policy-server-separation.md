# ADR-007 — Circuit Breaker vs. Policy Server: Kept Separate

**Status:** Accepted
**Date:** 2026-06
**Discharges:** Synthesis B3 (Architecture Decision Records) — formally records the Stage 4 reconciled decision

---

## Context

Both the Policy Server (Card 06 §4) and the Circuit Breaker (Card 06 §11) sit in Security's enforcement path. The question — surfaced explicitly during the cohesion review cycle — was whether they should be merged into a single component or kept separate.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Merge Circuit Breaker into the Policy Server (one enforcement component) | Collapses a decision-time control (Policy Server evaluates before an action happens) with a runtime-state control (Circuit Breaker acts on accumulated signal from what has already been happening) — blurring two fundamentally different enforcement semantics into one component |
| Circuit Breaker as a pure Harness concern (no Security involvement) | Removes the Policy Server's ability to receive the Circuit Breaker's trip signal and revoke in-flight JIT tokens (Card 06 §18) — the Circuit Breaker becomes a soft stop rather than a hard enforcement boundary |
| Distributed Circuit Breaker per agent (no central state) | Prevents system-level quarantine scoping (run-level, agent-level, tool-level, or system-level per Card 06 §11) and makes cross-agent blast-radius coordination impossible |

---

## Decision

Policy Server and Circuit Breaker are kept separate. Merging them was considered and rejected.

---

## Rationale

The Policy Server *decides* — it evaluates per-call authorization requests against ABAC+JIT rules and issues or denies scoped tokens. The Circuit Breaker *acts* on accumulated runtime signal — it trips, quarantines, and scopes the blast radius of an already-in-progress execution based on event patterns, not per-call decisions. This is the same Decide/Act separation Card 01 §2 already establishes for the execution loop generally (Plan/Decide vs. Act). Collapsing them would blur a decision-time control with a runtime-state control. A compromised or misbehaving Policy Server would then also compromise Circuit Breaker behavior, removing the last safety net.

---

## Tradeoffs

Two separate components means the Circuit Breaker must receive signals from Evaluation and the Harness rather than having direct access to Policy Server state. The handoff — specifically, the Circuit Breaker calling on the Policy Server to revoke a JIT token (Card 06 §18) when it trips — is an explicit dependency. This coupling is accepted because it runs in one direction only: the Circuit Breaker calls the Policy Server, never the reverse, maintaining the Decide/Act ordering.

---

## Consequences

**Positive:**
- Decide/Act separation is preserved end-to-end — the same principle Card 01 §2 establishes for the execution loop applies equally inside the Security layer itself
- Independent failure modes — a Policy Server outage and a Circuit Breaker trip are distinct, separately-handled events; Card 06 §20's fail-closed table governs the former, Card 06 §11's quarantine behavior governs the latter
- A compromised Policy Server cannot silently disable the Circuit Breaker — the two components' integrity is independently governable
- Blast-radius scoping remains flexible — the Circuit Breaker can quarantine at run/agent/tool/system level without that granularity needing to live inside the Policy Server's authorization logic

**Negative:**
- One-directional coupling — the Circuit Breaker depends on the Policy Server to revoke JIT tokens; if the Policy Server is unavailable when the Circuit Breaker trips, token revocation follows Card 06 §20's fail-closed table rather than an active revocation
- Two components to monitor and test — the Circuit Breaker and Policy Server each require independent health monitoring and failure-mode testing
- The signal path (Evaluation/Harness → Circuit Breaker → Policy Server) is a multi-hop dependency whose coordination must be explicitly maintained
- Quarantine-release-as-Tier-4-action (Card 06 §11) flows through the Policy Server — the same one-directional coupling applies to the release path as to the trip path

---

## Decision Status

Accepted — documented in Card 06 §11 as an "Alternative Architecture Considered" note (added in Closure Plan Stage 4). This ADR formally records the same decision at the governance level, making it citable independently of the card content.

**Review Trigger:** The one-directional coupling between Circuit Breaker and Policy Server creates a real latency problem at scale — revisit the handoff mechanism at that point, not the principle of separation.

---

## References

Card 06 §4 (Policy Server — decides), Card 06 §11 (Circuit Breaker — acts, blast-radius scoping, "Alternative Architecture Considered" note), Card 06 §18 (JIT token revocation — the one-directional handoff mechanism), Card 06 §20 (fail-closed table — governs Policy Server unavailability during a Circuit Breaker trip), Card 01 §2 (Decide/Act separation — the same principle applied here inside the Security layer), `cohesion-reviews/v1/review-reconciliation.md` (D3 resolution), Closure Plan Stage 4, ADR-002 (Policy Server Centralization — the Policy Server is one of the two components this ADR keeps separate, and its centralization is what makes the one-directional coupling architecturally safe).
