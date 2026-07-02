# ADR-004 — Skills as Promoted Procedural Memory

**Status:** Accepted
**Date:** 2026-06
**Discharges:** Synthesis B3 (Architecture Decision Records)

---

## Context

Agents accumulate patterns of effective reasoning over time — step-by-step approaches to recurring domain problems. The question is how to make those patterns reusable: bake them into agent code directly, store them as raw memory records, or treat them as a distinct, governed artifact type.

---

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Embed reasoning patterns directly into agent code | Hides patterns from the governance layer entirely — no Contract, no Registry, no independent versioning — and couples reusable knowledge to a single agent's implementation, preventing cross-agent reuse |
| Leave reasoning patterns as procedural memory only | Procedural memory is subject to confidence degradation and consolidation uncertainty (Card 03 §8); it is unsuitable as a durable instructional artifact because it can be silently downgraded, contested, or overwritten without a separate approval gate |
| Automatically promote procedural memory into Skills without human review | Removes the governance gate that prevents unreviewed or incorrect reasoning patterns from propagating across multiple agents simultaneously — the known worst-case of distributed AI training failures, applied at the knowledge artifact level |

---

## Decision

Reusable reasoning patterns are stored as Skills — instructional content (SKILL.md format) that lives in a governed Skill Registry and is loaded into runtime context on demand through the Skill Registry (Card 04 §6). Skills are specifically promoted from procedural memory (Card 03 §5) through a human-reviewed process (Card 04 §8), not written directly as code or left as raw memory records.

---

## Rationale

Baking patterns into agent code makes them invisible to the governance layer (no Contract, no Registry, no version control separate from the agent itself) and non-reusable across agents. Leaving them as raw memory records means they're subject to the same confidence-degradation and consolidation uncertainty as any other memory, with no approval gate before they influence behavior. Skills as a distinct type give them: a fixed, reviewable format (SKILL.md); a defined promotion gate (human approval before activation); version control independent of any single agent; and discoverability through the Skill Registry rather than implicit agent knowledge.

---

## Tradeoffs

Skills require a promotion process — procedural memory doesn't automatically become a Skill, and the human review gate adds latency before a learned pattern becomes reusable. This is the deliberate cost of the governance benefit: an unapproved Skill cannot be loaded (Card 04 §6's binding rule), which prevents a bad pattern from propagating across multiple agents before it's caught.

---

## Consequences

**Positive:**
- Skills become reusable organizational knowledge — a pattern learned in one project context can be governed and promoted for use across agents
- Governance separates learning from execution — agents cannot directly elevate their own memory to executable instruction without human review
- Human review prevents low-quality or incorrect reasoning from propagating — the promotion gate is an architectural firewall against the distributed-failure mode of unreviewed knowledge
- Skills evolve independently of agents — a Skill can be updated, versioned, and deprecated without touching any agent's code or contract

**Negative:**
- Promotion requires manual review — new knowledge is not immediately reusable, introducing latency between learning and reuse
- Registry governance becomes critical infrastructure — if the Skill Registry is unavailable, no new Skills are loadable system-wide (Card 04 §6's blast-radius statement)
- Skill maintenance becomes an ongoing responsibility — versioning, deprecation, and quality review apply indefinitely, not just at promotion time
- The promotion pipeline (Card 03 §8 → Card 04 §8) is a dependency between two cards that must stay synchronized

---

## Decision Status

Accepted — Card 04 is the canonical definition. The Memory-to-Skill promotion pipeline bridges Card 03 §8 (memory consolidation) and Card 04 §8 (Skill lifecycle).

**Review Trigger:** The human review gate for promotion becomes a genuine bottleneck once the ecosystem is running at scale — revisit the promotion trigger criteria at that point, not the principle itself.

---

## References

Card 03 §5 (procedural memory type), Card 03 §8 (consolidation and promotion candidates), Card 04 §5 (Skill Contract), Card 04 §6 (Skill Registry), Card 04 §8 (Skill lifecycle and promotion), Card 06 §15 (Skills as signed instruction artifacts), ADR-003 (The Contract Pattern — Skills are one of the four governed artifact types whose governance philosophy this ADR extends).
