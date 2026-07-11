# ADR-002 — Policy Server Centralization

**Status:** Accepted
**Date:** 2026-07-02
**Discharges:** Synthesis B3 (Architecture Decision Records)

---

## Context

Every layer above Security (Tools, Memory, Skills, Evaluation) needs authorization decisions made consistently — who can call what, under what conditions, with what scope. The question is whether those decisions live distributed across each layer's own logic, or centralized in a single authoritative enforcement point.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Authorization enforced independently by each architectural layer | Creates duplicated policy logic, inconsistent enforcement, fragmented auditing, and multiple failure modes — each layer's enforcement gap becomes a silent security hole the others can't compensate for |
| Hybrid model (central policy with selective local overrides) | Reintroduces ambiguity over which component is authoritative for any given decision, weakening the single-owner principle established throughout the architecture and making audit reconstruction unreliable |
| Tool-level authorization only | Insufficient because authorization also governs memory access (Card 03 §11), registry write authority (Card 06 §7/§16), Safety evaluation enforcement (Card 06 §9), and any future architectural components beyond tools |

---

## Decision

All authorization decisions — ABAC+JIT permission grants, registry write authority, memory governance authorization, Safety evaluation enforcement mapping — are centralized in a single Policy Server (Card 06 §4).

---

## Rationale

Distributing authorization logic across layers would mean: (a) a permission bug in any one layer creates a gap the others can't compensate for; (b) audit records would need to be assembled from multiple sources to reconstruct a complete decision history; (c) changing a policy rule requires finding and updating every place it's enforced rather than one. Centralization means a single point of configuration, a single audit source, and a single, explicitly-defined failure mode (Card 06 §20's fail-closed table) rather than multiple silent failure modes spread across the stack.

---

## Tradeoffs

The Policy Server is now a high-blast-radius single point of failure — explicitly the known and accepted tradeoff, addressed in Card 06 §20 (fail-closed behavior by tier) and §21 (the Policy Server's own integrity is itself governed). A distributed approach would have lower blast radius per component, but higher aggregate risk due to enforcement gaps and audit fragmentation.

---

## Consequences

**Positive:**
- Single authorization authority — no ambiguity over which component makes a permission decision
- Central audit trail — every authorization decision and its governing policy version are recorded in one place (Card 06 §11/§12)
- Easier policy evolution — changing a rule means changing one place, not hunting across all layers
- Consistent enforcement — the same ABAC+JIT logic applies regardless of which layer is making the request

**Negative:**
- Policy Server becomes critical infrastructure — its availability and integrity are prerequisites for all Tier 3+ actions (Card 06 §20)
- Fail-closed operation is mandatory, not optional — Card 06 §20's table must be implemented correctly or the entire authorization model fails silently
- Greater testing and integrity discipline required — Card 06 §21 exists specifically as a consequence of this centralization
- Policy Server administration is itself a Tier 5 action (Card 06 §21) — the enforcer must itself be governed at the highest tier

---

## Decision Status

Accepted — Card 06 §4 is the canonical definition. tech-stack.md Decision #7 records OPA (WASM-embedded) as the implementation choice.

**Review Trigger:** Multi-region or multi-tenant deployment requires geographically distributed policy enforcement — not applicable at current solo/single-machine scale.

---

## References

Card 06 §4 (Permission Model), §9 (Safety enforcement), §20 (failure mode), §21 (integrity), Card 02 §5 (Assignment Authority defers here), Card 03 §11 (Memory Contracts defer here), Card 04 §6 (Skill Registry assignment defers here), tech-stack.md Decision #7, ADR-001 (Runtime Stack Ordering — establishes that Security sits beneath every layer, of which this ADR is the primary consequence within the Security layer itself).
