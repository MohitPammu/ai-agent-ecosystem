# ADR-003 — The Contract Pattern (Agent, Tool, Memory, Skill)

**Status:** Accepted
**Date:** 2026-06
**Discharges:** Synthesis B3 (Architecture Decision Records)

---

## Context

The ecosystem has four distinct governed artifact types: Agents, Tools, Memory, and Skills. Each needs a declaration of its identity, scope, ownership, access, lifecycle, and success/failure behavior before being instantiated or used. The question is whether each type gets its own bespoke specification format, or whether a shared pattern governs all four.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Independent specification format for every governed artifact type | Creates inconsistent review processes, increases cognitive load across artifact types, and weakens the spec-driven discipline established by Card 07 — a developer familiar with one contract type has no mental model for the next |
| Single inherited schema shared across every artifact type | Too rigid; forces unrelated fields onto fundamentally different artifact types (a Tool's input/output contract has no business appearing in a Memory Contract), coupling independent concerns and producing bloated, ambiguous schemas |
| Documentation created after implementation | Violates the specification-first philosophy — governance becomes descriptive rather than prescriptive, and the review gate that catches design problems before code exists disappears |

---

## Decision

All four governed artifact types follow the same Contract pattern: a structured declaration of identity, scope, ownership, access, success/failure behavior, and lifecycle status — drafted and approved before implementation, not documented after the fact. The four types (Agent Contract, Tool Contract, Memory Contract, Skill Contract) share the same governance philosophy while maintaining separate schemas appropriate to their distinct concerns.

---

## Rationale

A shared pattern means: (a) any developer familiar with one contract type can read any other without a new mental model; (b) the same review discipline (spec before code, human approval, version bumps on change) applies uniformly rather than being reinvented per type; (c) Card 07's spec-driven development discipline has a concrete, uniform artifact to enforce rather than four separate processes. The alternative — bespoke formats per type — would produce the same structural inconsistency that the Closure Plan's Stage 1 forward-reference audit was specifically designed to catch.

---

## Tradeoffs

A shared pattern requires enough abstraction to cover four genuinely different things (callable functions, instructional content, memory access, and agent-level behavior) in one conceptual framework. The ecosystem resolves this by sharing the *governance philosophy* (declaration-before-instantiation, human-approved lifecycle, version-controlled) while allowing each type's schema to carry only the fields its specific concern requires — no forced field inheritance across types.

---

## Consequences

**Positive:**
- Shared governance vocabulary — one mental model covers all four artifact types
- Consistent review workflow — the same spec-before-code discipline applies regardless of artifact type
- Predictable artifact lifecycle — Draft → Reviewed → Approved → Active → Deprecated applies uniformly
- Architectural drift is harder — a new artifact type must fit the pattern or explicitly justify deviation

**Negative:**
- Four separate schemas must be maintained independently — shared philosophy, not shared schema, means four places to evolve rather than one
- Governance philosophy changes affect every contract type simultaneously — a change to the review discipline in Card 07 ripples across all four
- Discipline is required to prevent schema drift between types — the similarity of pattern can tempt future contributors to copy fields across types inappropriately

---

## Decision Status

Accepted — `docs/architecture/agent-contract-template.md` is the primary artifact. Card 02 §4 (Tool Contract), Card 03 §11 (Memory Contracts), Card 04 §5 (Skill Contract) each document their type's specific schema.

**Review Trigger:** A fifth governed artifact type emerges that the Contract pattern genuinely cannot accommodate — not anticipated within the current three-project scope.

---

## References

Card 01 §1 (Contract as a core architectural component on par with Model/Tools/Orchestration), Card 02 §4 (Tool Contract), Card 03 §11 (Memory Contracts), Card 04 §5 (Skill Contract), `docs/architecture/agent-contract-template.md`, Card 07 (SDD discipline — the spec-first philosophy this pattern enforces), ADR-001 (Runtime Stack Ordering — Contract is the first layer of the stack, of which this ADR documents the shared governance philosophy).
