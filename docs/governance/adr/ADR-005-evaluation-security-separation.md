# ADR-005 — Evaluation/Security Separation

**Status:** Accepted
**Date:** 2026-06
**Discharges:** Synthesis B3 (Architecture Decision Records)

---

## Context

Both Evaluation (Card 05) and Security (Card 06) have roles in safety-related behavior: Evaluation detects Safety/Alignment failures; Security enforces consequences for those failures. The question is whether these concerns should be unified — one system that both detects and enforces — or kept strictly separated.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Evaluation both detects and enforces | Creates a single point of failure where incorrect detection immediately becomes incorrect enforcement, eliminating independent verification — a bug in evaluation logic is automatically also a bug in enforcement with no governance layer between them |
| Security performs its own evaluation | Couples enforcement policy to evaluation logic, preventing each from evolving independently, duplicating decision-making responsibility, and blurring the ownership boundary between Card 05 and Card 06 |
| Shared Evaluation/Security subsystem | Merges detection with authority, weakening audit traceability (a single record can no longer separately show what was detected vs. what was decided) and violating the single-owner principle established throughout the architecture |

---

## Decision

Evaluation detects — it does not enforce. Security enforces — it does not evaluate. When Card 05's Safety pillar evaluation flags a violation, it is Card 06's Policy Server that acts on that flag (Card 06 §9). Neither layer assumes the other's responsibilities.

---

## Rationale

Unifying detection and enforcement in one system creates a single point of failure for safety behavior — if that system's detection logic is wrong, its enforcement is also wrong, with no independent check between them. Separating them means: (a) Evaluation can be improved (better models, better rubrics) without touching enforcement logic; (b) enforcement policy can be tightened or loosened without retraining or reconfiguring detection; (c) the audit trail can independently record what was detected and what was decided, making post-hoc review possible. This also aligns with the broader Observability/Security separation: Card 05's observability infrastructure captures events generically, Card 06's Security Event Schema extends it with security-specific fields (Card 06 §11) rather than the two systems merging into one.

---

## Tradeoffs

Separation introduces a handoff: a detection signal must flow from Evaluation to the Policy Server before enforcement occurs. This handoff is a potential latency source and a potential failure point (if the Policy Server is unavailable when a Safety flag fires, Card 06 §20's fail-closed table governs, not Evaluation). This is the known and accepted tradeoff — the independence of the two systems is worth more than the cost of the handoff.

---

## Consequences

**Positive:**
- Independent evolution — Evaluation's detection logic (models, rubrics, thresholds) and Security's enforcement policy can each improve without coupling to the other
- Clear audit separation — the record of what was detected (Card 05's trace) and what was decided (Card 06 §11's Security Event Schema) are independently queryable, not merged into one ambiguous entry
- Better testing boundaries — Evaluation and Security can each be tested against simulated inputs from the other without requiring the full pipeline
- Reduced coupling — a failure in Evaluation (e.g., a model scoring incorrectly) does not automatically cascade into incorrect enforcement

**Negative:**
- Handoff latency — every Safety enforcement decision has a detection-to-enforcement gap; for time-critical violations this is a real operational cost
- Additional integration point — the Evaluation-to-Policy-Server signal path is a dependency that must be monitored, tested, and fail-closed (Card 06 §20 covers this explicitly)
- Policy Server becomes responsible for enforcement availability — if the Policy Server is unavailable when Evaluation fires a Safety flag, enforcement silently degrades to the fail-closed table rather than to Evaluation's own fallback
- More infrastructure coordination — two independent systems must agree on the signal format between them, and changes to that format require coordinated updates

---

## Decision Status

Accepted — Card 05 §4/§5 (Evaluation, detection only) and Card 06 §9/§9a (Security, enforcement boundary) document the separation. Card 05's direct implications explicitly state: "Evaluation is a downstream, non-blocking consumer of observability data."

**Review Trigger:** A use case emerges where the detection-to-enforcement latency is genuinely unacceptable for a safety-critical decision — revisit the handoff mechanism at that point, not the principle of separation.

---

## References

Card 05 §4 (Evaluator Hierarchy — detection), Card 05 §5 (Observability — non-blocking, downstream consumer), Card 06 §9 (Safety Evaluation Enforcement Boundary), Card 06 §9a (layer ownership explicit), Card 06 §11 (Security Event Schema extends, does not pollute, Card 05's observability), Card 06 §20 (fail-closed table governs when Policy Server is unavailable during an Evaluation signal), ADR-002 (Policy Server Centralization — the Policy Server is the enforcement actor this ADR separates from Evaluation's detection role).
