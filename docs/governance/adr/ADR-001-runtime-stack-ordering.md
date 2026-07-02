# ADR-001 — Runtime Stack Ordering

**Status:** Accepted
**Date:** 2026-06
**Discharges:** Synthesis B3 (Architecture Decision Records)

---

## Context

The ecosystem requires a strict dependency hierarchy governing which layer owns which concern. Without an explicit ordering, cards could contradict each other's authority — e.g., a Tools card claiming to govern Security concerns, or a Memory card claiming to govern Orchestration behavior.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Security as a cross-cutting concern (applied at each layer independently) | Distributes enforcement responsibility across layers, recreating the exact ambiguity the stack ordering exists to eliminate — each layer would need to know how much Security applies to it, rather than Security owning that determination entirely |
| Security as a peer layer (same level as Tools, Memory, etc.) | Eliminates the unambiguous authority resolution the base position provides — a peer cannot govern its peers |
| Decentralized ownership (each layer governs its own concerns, no stack ordering) | Already the implicit status quo before this decision was made explicit; produced the "forward-reference debt" across Cards 01-05 that this review cycle was created to resolve |

---

## Decision

The Runtime Stack is ordered as follows, top-to-bottom:

1. Contract
2. Harness / Orchestration
3. State
4. Memory
5. Tools (Card 02) / Skills (Card 04) — same stack position, distinct governance semantics
6. Evaluation & Observability
7. Security & Governance

Security & Governance sits at the base. Every layer above it is governed by Security, never the reverse.

---

## Rationale

Placing Security at the base — rather than treating it as a cross-cutting concern or a peer layer — means the ownership question for every enforcement decision resolves unambiguously: if two layers conflict, the lower layer wins. This is the governing principle that allowed Cards 02, 03, 04, and 05 to all defer permission enforcement, sandbox requirements, and authorization decisions explicitly to Card 06, without any of those cards needing to know how Card 06 implements them. The stack ordering is the single fact that makes every "X is Card 06's responsibility, not ours" statement in the card set correct.

---

## Tradeoffs

Security-at-base means Security governs even the Contract and Harness layers — a Contract cannot grant itself permissions the Policy Server wouldn't authorize, and a Harness cannot bypass the Circuit Breaker by design. This is the desired behavior, but it means the Security layer is a hard dependency for every other layer's correct operation, giving it the highest blast radius of any single component in the ecosystem (explicitly documented in Card 06 §20).

---

## Consequences

**Positive:**
- Single ownership model — no ambiguity about which layer governs any given enforcement concern
- Predictable authority resolution — lower layer always wins, no case-by-case judgment
- Every "not our responsibility" deferral across Cards 02-05 is correct by definition, not coincidence
- Future cards or layers can be added without renegotiating ownership boundaries

**Negative:**
- Security & Governance becomes critical infrastructure — its availability and integrity are prerequisites for every other layer's correct operation
- Higher blast radius than a decentralized design — Card 06 §20's fail-closed table exists specifically because of this consequence
- Greater emphasis required on fail-closed behavior, break-glass discipline, and Policy Server integrity (Card 06 §20/§21/§25)

---

## Decision Status

Accepted — reflected throughout Cards 01-07, AGENTS.md §3, and tech-stack.md.

**Review Trigger:** Architecture grows beyond the current 7-layer model in a way that requires a new governing layer — not anticipated within the three reference projects.

---

## References

Card 01 §1 (Contract definition), Card 01 §2 (Harness/execution loop), Card 06 §1 (Security governs the harness, not the model), Card 06 §20 (blast radius and fail-closed consequences), AGENTS.md §3 (Runtime Stack), AGENTS.md §10 (Security is not optional configuration), ADR-002 (Policy Server Centralization — the primary consequence of this ordering within the Security layer itself).
