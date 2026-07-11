# ADR-006 — Tool Registry vs. Skill Registry: Kept Separate

**Status:** Accepted
**Date:** 2026-07-11
**Changelog:** Rationale corrected — removed inaccurate "binary Active/Deprecated lifecycle" claim; Tool lifecycle is Card 02 §6's actual 9-stage sequence. Patch 3, Stage 10 pre-freeze.
**Discharges:** Synthesis B3 (Architecture Decision Records) — formally records the Stage 4 reconciled decision

---

## Context

Both Tools (Card 02 §5) and Skills (Card 04 §6) have registries that serve discovery and governance functions, with structurally similar entry-field tables. The question — surfaced explicitly during the cohesion review cycle — was whether these should be unified into a single Capability Registry or kept separate.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Unified Capability Registry (one schema, one registry for both Tools and Skills) | Forces one schema to serve two fundamentally different governance semantics — executable capability vs. procedural knowledge — producing a schema that is either too broad (carrying irrelevant fields for each type) or too narrow (losing the fields each type specifically requires) |
| Shared registry with type-specific sub-schemas | Introduces a type-discriminator field and conditional schema logic, recreating the same complexity as two separate files while adding indirection that weakens the single-owner principle per type |
| No registry for Skills (free filesystem discovery) | Removes the binding-rule enforcement that prevents agents from loading unapproved Skills — the exact failure mode Card 04 §6 exists to prevent |

---

## Decision

Tool Registry and Skill Registry are kept separate. A unified Capability Registry was considered and rejected.

---

## Rationale

A Tool is a callable function with a fixed input/output contract, moving through the 9-stage lifecycle Card 02 §6 defines (Proposed → Designed → Security Reviewed → Built → Tested → Certified → Active → Deprecated → Retired) — it is executed, not read. A Skill is instructional content with no callable contract, promoted into existence via a human-reviewed process (Card 04 §8) rather than registered as a pre-built executable, moving through its own 6-stage lifecycle — it is loaded and interpreted, not called. Merging them would force one schema to serve two fundamentally different governance semantics: executable capability vs. procedural knowledge. The binding rule for each type is also different — a Tool binding check is performed at call time (Card 02 §5), while a Skill loading check is performed at context-assembly time (Card 04 §6). Separate registries means separate binding rules can evolve independently.

---

## Tradeoffs

Two registries instead of one means two files to maintain (`tool-registry.yaml` and `skills-registry.yaml`), two places where the binding-rule discipline must be enforced, and two schema definitions to keep internally consistent. This is explicitly accepted — the governance-semantic difference between the two types is real enough that a shared schema would be misleading, not efficient.

---

## Consequences

**Positive:**
- Each registry's schema and binding rule can evolve independently — adding a field to the Tool schema doesn't risk polluting the Skill schema and vice versa
- Governance semantics are clear per type — "present in the Tool Registry" and "present in the Skill Registry" are distinct, unambiguous checks with no type-discriminator logic
- Separate registries means separate blast-radius statements — a Tool Registry outage and a Skill Registry outage are independently scoped, as documented in Card 02 §5 and Card 04 §6 respectively
- Consistent with the Contract Pattern (ADR-003) — each type has its own schema, governed by the same philosophy but not forced into a shared format

**Negative:**
- Two files to maintain — binding-rule discipline must be enforced in two places rather than one
- Schema consistency between the two types requires deliberate coordination — structural drift (e.g., one registry adds a `compliance_tier` field the other should also have) is now caught by review discipline rather than structural constraint
- Discovery tooling must be aware of both registries if cross-type queries are ever needed

---

## Decision Status

Accepted — documented in-card in Card 02 §5 and Card 04 §6 as "Alternative Architecture Considered" notes (added in Closure Plan Stage 4). This ADR formally records the same decision at the governance level, making it citable independently of the card content.

**Review Trigger:** A third artifact type emerges that is so similar to either Tools or Skills that a unified registry becomes genuinely useful — not anticipated within the current three-project scope.

---

## References

Card 02 §5 (Tool Registry, binding rule, "Alternative Architecture Considered" note), Card 04 §6 (Skill Registry, binding rule, "Alternative Architecture Considered" note), `cohesion-reviews/v1/review-reconciliation.md` (D1/D2 resolutions), Closure Plan Stage 4, ADR-003 (The Contract Pattern — each registry type's schema follows its own governed Contract, the shared philosophy this ADR's separation preserves), ADR-004 (Skills as Promoted Procedural Memory — explains why the Skill Registry's governance semantics are fundamentally different from the Tool Registry's).
