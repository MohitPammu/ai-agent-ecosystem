# Ecosystem Cohesion Review — Independent Assessment
## Part 3 of 4: Dimensions 9-11 (Long-Term Maintainability, Implementation Feasibility, Architectural Simplicity)

*(Continued from Parts 1-2. Same Architecture Version, Reviewer Independence, and Assumptions apply — see Part 1 header.)*

---

## Dimension 9 — Long-Term Maintainability

**Score: 8/10**
**Confidence: High**

**Evidence (Strong):** Most dependency relationships are discoverable from a single card's header — every card opens with a "Governing structure" line naming what it inherits and what it forward-references, which is a genuinely strong maintainability signal. I tested this directly: reading Card 04's header alone correctly told me it depends on Card 03 (procedural memory) and sits below Card 02 (Tools) in the conceptual layer — no need to cross-reference the full card body just to understand its place in the system.

**Finding (Strong evidence) — Card 06's internal navigability:** Card 06 has grown to 26 numbered sections covering risk tiers, the permission model, sandboxing, A2A trust, registry integrity, memory governance, instruction-artifact signing, Policy Server self-integrity, classification inheritance, HITL authorization, break-glass, and five more "closing remaining gaps" topics in §26 alone. Finding *everything* Card 06 says about, for example, Tool Registries specifically requires checking §7 (registry integrity discharging Card 02) **and** §16 (runtime verification, fail-closed scoping) — two non-adjacent sections, separated by ~9 other sections covering unrelated topics (sandboxing, A2A, JIT tokens). A maintainer asked "what does Card 06 say about registries" has to read the whole card or already know exactly which two section numbers to jump to.

**Risk if Ignored:** Medium — this is a navigability cost, not a correctness one. The risk is future editing: when Card 06 is revised again, an editor who updates §7 but forgets §16 exists (or vice versa) could introduce exactly the kind of unpropagated-correction problem already found in this review (Card 01/05's evaluation-score drift, Card 06/07's tier-number drift). Length without an index is a structural maintainability risk, not just an inconvenience.

**Recommended Action:** Add a short topic index at the top of Card 06 (e.g., "Registries: §7, §16 · Risk Tiers: §3, §14 · Policy Server: §4, §20, §21") so related-but-separated sections are discoverable without a full read-through. This is a low-effort fix relative to the maintainability risk it closes.

---

## Dimension 10 — Implementation Feasibility

**Score: 6/10**
**Confidence: High** — directly testable by asking "could I, with zero institutional context, start Phase 1 right now using only what's in front of me."

**Evidence (Strong) — direct application of the rubric's required test:** I evaluated this from the position the rubric specifies: an experienced engineer with no participation in this architecture's design history. Three concrete blockers prevent that engineer from starting cleanly:

1. **Missing foundational files** (carried forward from Dimension 5): the Agent Contract template, AGENTS.md, and `model-routing-table.md` are referenced as load-bearing by multiple cards but don't exist in the set under review. A stranger-engineer cannot instantiate a single agent per Card 01 §1's own stated requirement ("every agent should be instantiated *from* a contract") without that contract template existing. This is not an architectural ambiguity — the cards are clear about what these files should contain — but it is a hard implementation blocker nonetheless.

2. **The Tier 3/4 contradiction** (carried forward from Dimension 7): a stranger-engineer implementing the CI/CD code-review gate per Card 07 §6 would hit a genuine fork requiring a clarifying question — "Card 07 says Tier 3+ needs human review, Card 06's own table says Tier 4+. Which one governs?" This is exactly the kind of *architectural* question the rubric says should not be necessary, as distinct from acceptable implementation-detail questions.

3. **The untrusted-context-boundary mechanism** (carried forward from Dimension 5): Card 06 §26 requires Card 03's context assembly to "preserve source boundaries," but no card specifies *how* — no source-labeling scheme, no delimiter convention, nothing implementable. A stranger-engineer building `core/harness/`'s context assembly would need to invent this mechanism from scratch or ask for clarification, since neither card actually operationalizes the requirement.

**What the architecture gets right, weighed against the above:** every *other* deferred-to-implementation item I checked is correctly and explicitly flagged as such rather than silently missing — e.g., Card 02 §6's deliberate non-specification of tool-lifecycle transition criteria ("not fully specified in this card to avoid over-specifying before any tool exists") is exactly the right kind of acceptable implementation-detail deferral the rubric wants to see, and it's clearly labeled. The three blockers above are qualitatively different — they're either missing artifacts the cards assume exist, or genuine unresolved contradictions, not deliberately-scoped-out detail.

**Risk if Ignored:** High — this is the dimension that converts the other findings into a concrete "can work begin" answer, and the honest answer right now is "not yet, on three specific counts."

**Recommended Action:** (1) Draft the three missing files before declaring Phase 0 complete. (2) Resolve the Tier 3/4 contradiction (see Dimension 7's recommended fix). (3) Either add a minimal source-boundary mechanism to Card 03, or explicitly move it to the Master Plan as a named Phase 1 deliverable with an owner — right now it exists only as an unfulfilled obligation from Card 06 to Card 03.

---

## Dimension 11 — Architectural Simplicity (Anti-Bloat)

**Score: 7/10**
**Confidence: Medium** — confidence is capped at Medium because this dimension's required method (the 4-question test) is inherently judgment-based; reasonable independent reviewers could weigh "is this complexity worth it" differently even from identical evidence.

I applied the rubric's required 4-question test to each major component named in the rubric, plus one the rubric didn't name but that emerged as relevant during this review:

**Policy Server** — *Problem:* centralized ABAC+JIT authorization, registry write authority, memory governance authorization, Safety-flag-to-action mapping. *Replacement:* distributed authorization checks embedded per-component. *What disappears if removed:* a single source of truth for permission decisions, consistent audit logging across every enforcement point, and the single place JIT tokens are issued/revoked from. *Worth it:* **Yes.** Given two of three target projects involve regulated healthcare/financial data, centralized, auditable enforcement is justified despite the single-point-of-failure risk — and that risk is itself directly addressed (Card 06 §20-21's fail-closed/compromise handling). **No bloat finding.**

**Circuit Breaker** — *Problem:* automated quarantine response when a security threshold is crossed. *Replacement:* this specific capability could plausibly be a *function within* the Policy Server (which already "maps Safety evaluation flags to action," §9, and issues/revokes JIT tokens, §18) rather than a separately named component family (`core/security/circuit_breaker/`, distinct from `core/security/policy_server/`). *What disappears if merged:* very little identifiable capability — the Circuit Breaker's distinct value (explicit scope/trigger/release semantics) could be expressed as a Policy Server sub-module just as easily as a sibling component. *Worth it:* **Marginal.** This is the clearest anti-bloat candidate I found: two components with adjacent, overlapping responsibility, where the separation reads more like organizational convenience (two `core/security/` subfolders) than a necessity driven by genuinely distinct capability.

**Tool Registry and Skill Registry** — *Problem:* each governs discovery/binding for its respective capability type. *Replacement:* a single, generic "Capability Registry" with a `capability_kind: tool | skill` discriminator field. I compared the two registries' field schemas directly (Card 02 §5 vs. Card 04 §6) — they are nearly identical: ID, version, owner, status, scope, trigger metadata, location/contract reference, and the same binding-rule pattern (registered + Active + in-scope). *What disappears if merged:* I could not identify a capability that depends on these being two separate mechanisms rather than one mechanism with a type field. *Worth it:* **This is the strongest anti-bloat finding in this review.** The two-registry design appears to be the result of Card 04 deliberately mirroring Card 02's pattern for consistency (a reasonable instinct) rather than a case where Tools and Skills have genuinely divergent discovery/governance needs.

**Skill Loader (Progressive Disclosure)** — *Problem:* three-tier context loading (metadata → body → supporting files) specific to Skills' size and conditional relevance. *Replacement:* could be folded into a generic "Context Loader" shared with Tool-definition loading. *What disappears if merged:* Tools don't need three loading tiers — a Tool Contract's relevant fields are small and load in one step; Skills' bodies are large enough that the 3-tier mechanism earns its keep. *Worth it:* **Yes.** This is a case where the complexity is justified by a genuine difference in the underlying problem (content size and conditional relevance), not present for Tools. **No bloat finding.**

**Evaluation Hierarchy** — *Problem:* escalating-cost evaluation (automated → LLM-as-Judge → Agent-as-Judge → HITL). *Replacement:* none identified — and notably, this mechanism is *reused* three separate times across the card set (Card 04 §7's Skill Certification, Card 07 §6's code review, and its own native use in Card 05) rather than being reinvented per use case. This is the opposite of bloat — it's the kind of abstraction reuse the rubric's anchors describe at the 9-10 band. **No bloat finding; flagged as a positive example.**

**Alternative Architecture Considered — a gap in the documentation practice itself, not the architecture:** the rubric optionally encourages documenting rejected alternatives for major decisions. None of the seven cards explicitly document an "Alternative Considered / Rejected Because" note for any of their defining decisions (Policy Server centralization, the Contract pattern, the Runtime Stack ordering itself). The reasoning behind these choices is present in prose form (e.g., Card 06 implies centralization is worth its single-point-of-failure risk because of audit needs), but it's never formally labeled as a considered-and-rejected alternative. This doesn't affect the architecture's correctness, but it's a missed opportunity for the kind of "proves the decision was intentional" documentation the rubric specifically values for long-term maintainers who weren't present for the original reasoning.

**Risk if Ignored:** Medium for the Tool/Skill Registry finding specifically — maintaining two structurally-identical-but-separately-evolving registries is exactly the kind of duplication that drifts apart over time (one gets a field the other doesn't, as nearly happened with risk tiers and HITL elsewhere in this review). Low for the Circuit Breaker/Policy Server finding — less urgent, more an organizational/clarity question than a correctness risk. Low for the missing "alternatives considered" documentation — valuable but not blocking.

**Recommended Action:** Most actionable: evaluate whether Tool Registry and Skill Registry should be unified into one Capability Registry mechanism before Phase 3/4 implementation begins — this is cheaper to do now, before either is built, than to refactor two parallel implementations later. Secondary: consider whether Circuit Breaker should be explicitly scoped as a Policy Server sub-module rather than a sibling component, or if kept separate, that the *reason* for the separation be stated explicitly in Card 06 rather than left implicit.

---

*(Continued in Part 4: Dimensions 12-14 — Evolutionary Resilience, Blast Radius Analysis, Operational Validation Readiness — plus closing sections: Blocking Issues, Architectural Debt Register, Future Validation Plan, and Overall Verdict)*
