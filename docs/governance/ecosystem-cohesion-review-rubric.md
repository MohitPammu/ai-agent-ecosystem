# Ecosystem Cohesion Review Rubric — Cards 01-07 as a Single Specification

**Rubric Version:** 1.0 (frozen)
**Architecture Version Under Review:** *(state explicitly per use, e.g., "Cards 01-07, 2026-06-25 Freeze Candidate")*

**Reviewer Charter:** The objective of this review is not to validate that the architecture reflects the author's intent. It is to determine whether an independent engineering team could safely implement, evolve, and operate the architecture as written. Review findings should favor long-term maintainability, correctness, and clarity over preserving existing wording or design decisions. You are not reviewing the author — you are reviewing whether the specification can survive independent implementation and years of evolution.

**Reviewer Disclosure (state before scoring):** Original author? Independent reviewer (no prior involvement)? Internal reviewer (familiar with the project but not its author)? External reviewer (cold read)? This matters because different reviewer relationships carry different evidentiary value when reconciling multiple reviews — a review from the original author should be weighted accordingly against a cold-read external review.

> **Reviewer Charter:** The objective of this review is not to validate that the architecture reflects the author's intent. It is to determine whether an independent engineering team could safely implement, evolve, and operate the architecture as written. Review findings should favor long-term maintainability, correctness, and clarity over preserving existing wording or design decisions.

**Before scoring, every review must declare:**
- **Architecture Version** — which version of Cards 01-07 this review covers (e.g., "Cards 01-07, 2026-06-25 Freeze Candidate") — critical for making sense of a review six months later
- **Reviewer Independence** — Original author? Independent reviewer? Internal reviewer? External reviewer? — independent reviews carry different evidentiary value than self-review, and this should be stated rather than left implicit
- **Assumptions** (see below)

**Purpose:** This rubric evaluates the AI Agent Ecosystem as an **executable architectural specification**, not merely as a set of documents. It validates not only internal consistency, but also production viability, long-term maintainability, evolutionary resilience, and readiness for both implementation and future operational verification. A reviewer using this rubric should not ask "is this document well written" — they should ask "does this ecosystem function as one coherent architecture that a team of engineers will build, maintain, and evolve over the next five to ten years."

**How to use this rubric:** Score each dimension 1-10 using the anchors provided. Do not average dimensions into a single number — report all dimensions plus a qualitative verdict. A high average can hide a single fatal dimension (e.g., a circular dependency) that should block freeze regardless of how well other dimensions score.

**Freeze Criteria vs. Future Validation Criteria:** Not every dimension below carries equal weight toward whether the architecture can freeze and move to implementation. Dimensions 1-11 (Ownership, Dependency Flow, Circular References, Duplication, Missing Abstractions, Production Readiness, Terminology Consistency, Scope Discipline, Maintainability, Implementation Feasibility, Architectural Simplicity) are **freeze criteria** — they must be resolved before Phase 1 begins. Dimensions 12-14 (Evolutionary Resilience, Blast Radius Analysis, Operational Validation Readiness) are important but are **future validation criteria** — they inform what an Architecture Verification Suite should test *after* Phase 1 infrastructure exists, not blockers to freezing the architecture documents today.

**Mandatory gate rule, stated explicitly to remove ambiguity:** any score of ≤4 on a freeze-criteria dimension (1-11) automatically results in an overall verdict of **Not-Freeze-Ready**, regardless of how well every other dimension scores. Scores are never averaged to produce an overall verdict — a circular dependency cannot be compensated for by excellent terminology consistency. This is a deliberate design choice: architecture failures are often binary, and an averaging scheme would hide exactly the kind of single fatal flaw this rubric exists to surface.

**State your assumptions up front.** Before scoring, list the assumptions this review is conducted under — e.g., "the Master Execution Plan remains authoritative," "LangGraph remains the orchestration framework," "the seven cards under review remain frozen at the version provided," "the zero-cost model stack remains the initial deployment assumption." If any of these assumptions later changes, scores may need to be revisited — recording them up front makes that traceable years later, rather than forcing a future reader to guess what the reviewer took for granted.

**State your confidence, separately from your score.** Evidence and confidence are not the same thing — a well-evidenced score can still be low-confidence if the underlying architecture artifacts the dimension depends on don't exist yet (e.g., scoring Evolutionary Resilience is inherently lower-confidence before any implementation exists to stress-test). For each dimension, state **Confidence: High / Medium / Low** with a one-line reason (e.g., "Medium — future resilience is necessarily speculative before Phase 1 artifacts exist"). Without this, a reader may wrongly assume every score carries equal certainty.

---

## Dimension 1 — Ownership Boundaries

**Question:** Does every concept in the ecosystem have exactly one card that owns its semantics, with all other cards only referencing or enforcing against that ownership?

| Score | Anchor |
|---|---|
| 9-10 | Every concept traceable to exactly one owning card; all cross-references are clearly "enforce/consume," never "redefine" |
| 7-8 | Ownership is clear for all major concepts; minor ambiguity in 1-2 edge cases that doesn't create contradiction risk |
| 5-6 | At least one concept has split or unclear ownership (two cards could each plausibly claim to define it) |
| 1-4 | Multiple concepts are redefined differently across cards, or a card has silently absorbed another's domain ("god layer" risk) |

**Check specifically:** Does Security (Card 06) enforce-not-own per its own §9a rule? Does every "Contract" type (Agent/Tool/Memory/Skill) have undisputed ownership? Does Card 07's methodology layer ever redefine an architectural decision from Cards 01-06 rather than just operationalizing it?

---

## Dimension 2 — Dependency Flow Integrity

**Question:** Does the Runtime Stack's declared dependency order (Contract → Harness → State → Memory → Tools → Evaluation/Observability → Security) actually hold across all 7 cards, with no card depending "upward" on a card that should depend on it?

| Score | Anchor |
|---|---|
| 9-10 | Every dependency reference flows in the declared direction; no card assumes a capability from a card "above" it in the stack |
| 7-8 | Flow holds with one minor exception that's explicitly acknowledged and justified in-text |
| 5-6 | One unacknowledged upward dependency exists but doesn't create a logical contradiction |
| 1-4 | Multiple upward dependencies, or the stack order itself is violated without explanation |

**Check specifically:** Card 06 (Security) sits at the base and governs everything above it — does anything in Cards 01-05 secretly assume Security-layer behavior without referencing Card 06? Does Card 07 (methodology, sits "outside" the stack) ever get treated as something Cards 01-06 depend on?

---

## Dimension 3 — Circular Reference Check

**Question:** Are there any reference loops (Card A requires Card B's mechanism, which requires Card A's mechanism, with neither resolvable independently)?

| Score | Anchor |
|---|---|
| 9-10 | Zero circular references; every forward-reference eventually resolves in a card that doesn't loop back |
| 7-8 | One soft circularity exists (two cards mutually reference each other) but each is independently meaningful without the other |
| 5-6 | A genuine circular dependency exists but is narrow in scope and doesn't block implementation |
| 1-4 | A circular dependency blocks a clear implementation order — you cannot build Card A's component without Card B's, and vice versa |

**Check specifically:** Card 01 defines State; Card 03 defines Memory (which depends on State); Card 06 governs Memory's verification (depends on Card 03); does Card 06 ever require something from Card 03 that Card 03 in turn requires from Card 06 with no resolution order?

---

## Dimension 4 — Duplication / Redundant Definition

**Question:** Is any concept, table, or mechanism defined more than once, with any risk the two definitions could drift out of sync?

| Score | Anchor |
|---|---|
| 9-10 | Every concept defined exactly once; all other mentions are explicit references, not restatements |
| 7-8 | One trivial restatement exists (e.g., a one-sentence reminder) but the authoritative definition is unambiguous |
| 5-6 | A concept is meaningfully redefined in a second location with slightly different details |
| 1-4 | Two genuinely competing definitions of the same concept exist (e.g., two different risk-tier schemes, as nearly happened in this ecosystem's Card 02 review) |

**Check specifically:** Risk Tiers (defined once, in Card 06 §3, after Card 02 deliberately deferred them) — confirm no other card redefines tier numbers/names. HITL — confirm it's described as one mechanism with multiple trigger points, not reintroduced as a new concept per card.

---

## Dimension 5 — Missing Abstractions

**Question:** Is there a concept the ecosystem clearly needs (based on what it's already built) that has no home in any of the 7 cards?

**Distinguish Missing from Incorrect when reporting findings under this dimension — they require different remediation:**
- **Missing**: no card claims ownership of a needed concept at all (e.g., "no data lineage abstraction exists anywhere")
- **Incorrect**: a concept exists and has an owner, but that ownership assignment, or the definition itself, is wrong (e.g., "memory ownership is assigned to the wrong card" or "the definition contradicts how it's used elsewhere"). An Incorrect finding is a Dimension 1 (Ownership) or Dimension 4 (Duplication) issue surfacing here — flag it under both if it's ambiguous which dimension it belongs to, rather than forcing it into one.

| Score | Anchor |
|---|---|
| 9-10 | No gaps found; every mechanism referenced has a defining home somewhere in the 7 cards |
| 7-8 | Minor gaps exist but are explicitly flagged as deferred-to-implementation in the Master Plan, not silently missing |
| 5-6 | A gap exists that isn't flagged anywhere — would surprise an implementer |
| 1-4 | A structurally necessary concept (e.g., a Failure Taxonomy, a Contract pattern) is entirely absent with no card claiming it |

**Check specifically:** Does every Contract type (Agent/Tool/Memory/Skill) have a complete field list somewhere? Is there a defined "Spec" lifecycle now that Card 07 added one? Does the Skill→Tool relationship have a clear "who calls whom" rule?

---

## Dimension 6 — Production Readiness (Failure Modes & Override Authority)

**Question:** For every centralized/high-authority component (Policy Server, Circuit Breaker, Registries), is there a defined behavior for unavailability, compromise, and override?

| Score | Anchor |
|---|---|
| 9-10 | Every high-authority component has an explicit fail-closed/fail-open table, a defined compromise-detection or integrity mechanism, and a constrained override path |
| 7-8 | Most high-authority components have this; one has only partial coverage (e.g., failure mode defined but not compromise) |
| 5-6 | At least one high-authority component has no defined failure behavior at all |
| 1-4 | Multiple high-authority components are treated as infallible, undefined-on-failure infrastructure |

**Check specifically:** Policy Server (Card 06 §20-21) — both unavailability and compromise addressed? Circuit Breaker — trigger/scope/release criteria defined? Registries (Card 02/04) — runtime verification, not just write-time signing?

---

## Dimension 7 — Cross-Card Terminology Consistency

**Question:** Does every term mean the same thing everywhere it's used, with renames/corrections fully propagated?

| Score | Anchor |
|---|---|
| 9-10 | Every term is used consistently; any historical rename (e.g., "structured state"→"Structured Records") is fully propagated to all cards that used the old term |
| 7-8 | Terminology is consistent with one minor unpropagated instance that doesn't cause confusion |
| 5-6 | A rename was only partially propagated, creating ambiguity in at least one card |
| 1-4 | The same term is used with materially different meanings in different cards |

**Check specifically:** Search all 7 cards for "structured state" (should be zero hits outside the historical correction note in Card 01/03). Confirm "HITL," "Risk Tier," and "Contract" are used identically everywhere they appear.

---

## Dimension 8 — Scope Discipline (No Unjustified Expansion)

**Question:** Does each card stay within its stated responsibility, deferring to the correct other card rather than absorbing territory to "be thorough"?

| Score | Anchor |
|---|---|
| 9-10 | Every card explicitly defers what isn't its responsibility, naming exactly which other card owns it |
| 7-8 | Scope discipline holds with one section that drifts slightly broader than necessary but doesn't create ownership conflict |
| 5-6 | One card has visibly absorbed a responsibility that reads as belonging elsewhere |
| 1-4 | Multiple cards show scope creep, making the "single owner per concept" property (Dimension 1) unreliable |

---

## Dimension 9 — Long-Term Maintainability

**Question:** If one card needs to change later, is the blast radius of that change predictable and contained?

| Score | Anchor |
|---|---|
| 9-10 | Every card's dependents are identifiable by reading that card alone (explicit forward/backward references); a change's impact can be scoped without reading all 7 cards |
| 7-8 | Mostly true; one or two dependency relationships require cross-referencing multiple cards to discover |
| 5-6 | Several implicit dependencies exist that would only surface during implementation, not from reading the docs |
| 1-4 | Changing any one card would require an unguided full re-read of all others to find impacts |

---

## Dimension 10 — Implementation Feasibility

**Question:** Could an experienced engineer who never participated in the conversations that produced this architecture successfully implement it using only Cards 01-07, the Master Execution Plan, and referenced specifications — without requiring additional architectural clarification? (Implementation-detail questions are acceptable and expected; architectural questions are not.) A mature architecture should survive personnel changes, contractor onboarding, future maintainers, and organizational growth without depending on institutional memory.

| Score | Anchor |
|---|---|
| 9-10 | Yes — every open question is already flagged as deferred-to-implementation in the Master Plan; nothing architectural is ambiguous; a new engineer could pick this up cold |
| 7-8 | Nearly — one or two architectural ambiguities would need a quick clarifying question, but nothing blocking |
| 5-6 | Several architectural questions would need resolving before Phase 1 could proceed confidently |
| 1-4 | Phase 1 cannot reasonably start without significant additional architecture work, or the architecture's coherence depends on context not captured in the documents themselves |

---

## Dimension 11 — Architectural Simplicity (Anti-Bloat)

**Question:** Is every major architectural abstraction earning its existence, or could the same capability be achieved with less machinery?

| Score | Anchor |
|---|---|
| 9-10 | Every major component solves a distinct architectural problem; no abstraction exists merely because "enterprise systems usually have one" |
| 7-8 | One or two abstractions feel heavier than strictly necessary, but each still does distinct, identifiable work |
| 5-6 | Several abstractions could plausibly be merged without losing meaningful capability |
| 1-4 | The architecture appears over-engineered relative to its actual problem scope |

**Required method — do not rely on reviewer intuition; apply this test to each major component (Policy Server, Circuit Breaker, Memory Router, Tool Registry, Skill Registry, Skill Loader, Evaluation Hierarchy, and any other component the reviewer judges "major"):**

1. What problem does this solve?
2. What existing abstraction could replace it?
3. What capability would actually disappear if it were removed?
4. Is that capability worth the architectural complexity it adds?

If question 3 produces "nothing" or "very little," the abstraction is likely bloat and should be flagged regardless of how well-documented it is.

## Dimension 12 — Evolutionary Resilience (Future Validation Criterion)

**Question:** Can the architecture absorb new capabilities through extension rather than requiring fundamental redesign?

| Score | Anchor |
|---|---|
| 9-10 | Future capability (new projects, new agent types, new model providers, new memory types, new Tool classes, new Skills, new security mechanisms, new evaluation strategies) can be introduced within existing ownership boundaries without restructuring the ecosystem |
| 7-8 | Most extension scenarios fit cleanly; one or two would require touching multiple cards simultaneously |
| 5-6 | Several plausible future scenarios would require restructuring rather than extending |
| 1-4 | The architecture would need fundamental redesign to accommodate realistic near-term growth |

**Test scenarios to walk through explicitly:** adding a 4th portfolio project; adding a new model provider to the routing table; adding a new memory type beyond the current five; adding a new risk tier; adding a Skill that needs to call another Skill directly (not just a Tool); **replacing the orchestration framework itself (e.g., LangGraph → ADK, or LangGraph → OpenAI Agents SDK)** — frameworks change faster than architecture, and a durable architecture built around contracts/ownership/registries should survive a framework swap without the contracts themselves needing to change, only their implementation.

## Dimension 13 — Blast Radius Analysis (Future Validation Criterion)

**Question:** When a given component fails, does the failure remain localized, or does it propagate uncontrolled through the rest of the ecosystem? (Distinct from Dimension 6, which asks "is a failure mode defined at all" — this dimension asks "how far does that failure actually spread.")

| Score | Anchor |
|---|---|
| 9-10 | Every major component's failure mode is explicitly scoped (run-level/agent-level/tool-level/system-level per Card 06 §11's circuit breaker model) and degrades predictably rather than catastrophically |
| 7-8 | Most components have scoped failure containment; one component's blast radius is implicit rather than explicit |
| 5-6 | At least one component's failure could plausibly cascade beyond its expected scope, with no explicit containment statement |
| 1-4 | Failures are not scoped at all — a single component failure could plausibly take down the whole ecosystem with no defined containment |

**Required method:** for each of Policy Server, Circuit Breaker, Memory, Registries, Tool Layer, Evaluation, and Harness — state explicitly what *else* breaks if this component fails, and whether that propagation is bounded by an existing architectural control (e.g., fail-closed tiers, registry-entry-scoped fail-closed behavior) or is currently unbounded. **Equally important — and easy to skip — ask what still works during that failure, not only what breaks.** Production systems are defined as much by graceful degradation as by failure handling: when the Policy Server is unavailable, does Tier 0 computation still execute (Card 06 §20)? Does documentation/reference loading still function? Can evaluation still run against already-collected traces? A component scoring well here names both halves of the picture.

## Dimension 14 — Operational Validation Readiness (Future Validation Criterion)

**Question:** Does the architecture explicitly acknowledge what documentation review cannot prove, and is there a named plan for verifying those things once implementation exists?

Documentation can validate ownership, consistency, and abstraction quality. It cannot validate orchestration correctness, latency characteristics, retry behavior, context growth under load, deadlock avoidance, policy enforcement under real traffic, runtime state transitions, or fault recovery — those require execution.

| Score | Anchor |
|---|---|
| 9-10 | The architecture explicitly names this boundary and defines (or points to) a planned Architecture Verification Suite to run post-Phase-1 |
| 7-8 | The boundary is implicitly understood but not explicitly named anywhere in the cards/plan |
| 5-6 | No verification plan exists, and the cards read as if documentation review alone is sufficient validation |
| 1-4 | The architecture conflates documentation review with operational proof, risking false confidence before implementation |

**Recommended verification scenarios to name (not perform) at this stage:** happy path, tool failure, policy denial, memory corruption, registry corruption, circuit breaker activation, retry exhaustion, context overflow, rollback after deployment, concurrent orchestration, agent delegation failure. These belong in an Architecture Verification Suite run after Phase 1, not in this documentation-stage review — but the *plan* to run them belongs here. **Each named scenario should pair with a measurable pass criterion, not just a description of the test** — e.g., "Retry exhaustion → Expected Result: no duplicate external side effect occurs" or "Policy Server compromise simulation → Expected Result: Circuit Breaker quarantines the affected scope within N seconds." Without a stated pass criterion, a future implementer building the verification suite has no definition of success to test against.

---

## Reporting Format

For each dimension, report all five of the following — do not skip any, and do not collapse them into a single paragraph:

- **Score** (1-10, using the anchors above)
- **Confidence** (High / Medium / Low, with a one-line reason — see "state your confidence" above)
- **Evidence** — exact card and section cited (e.g., "Card 06 §11, paragraph 4" not just "Card 06"), classified by **Evidence Quality** using these precise definitions: **Strong** (an exact section citation directly supports the claim — the reviewer can point to the specific sentence), **Medium** (multiple sections, whose *combined* meaning supports the claim, but no single citation does alone), or **Weak** (reviewer inference only — no specific citation supports the claim, even in combination). Weak evidence should rarely justify a low score on a freeze-criteria dimension — if the evidence is weak, the finding itself is provisional and should be flagged as needing follow-up rather than treated as conclusive
- **Risk if Ignored** — what concretely goes wrong later if this finding is left unaddressed (a vague score with no stated risk is not actionable)
- **Recommended Action** — a specific fix, or explicitly "None" if the dimension scored well enough to need nothing

This structure is what makes two independent reviewers' results reconcilable — reviewers frequently agree on the underlying evidence even when their scores differ, and Evidence + Risk + Recommended Action gives a synthesis pass something concrete to merge, rather than two unrelated narrative opinions.

**For major architectural decisions specifically (Policy Server, Circuit Breaker, Tool/Skill Registries, the Contract pattern, etc.), optionally note "Alternative Architecture Considered"** — what other design could have solved the same problem, and why it was rejected (e.g., "Policy Server — Alternative: distributed authorization — Rejected because: audit complexity"). This is not required for every dimension, but for the handful of decisions that most define this ecosystem's character, naming the rejected alternative is what proves the decision was intentional rather than accidental, and gives a future maintainer the "why" behind the choice, not just the choice itself.

**Required closing sections:**
- **Blocking issues** (anything scoring ≤4 on any *freeze-criteria* dimension — see the Freeze Criteria vs. Future Validation Criteria split above) — must be resolved before Phase 0 is considered closed
- **Non-blocking refinements** — improvements worth making but not required for freeze
- **Architectural Debt Register** — for any finding that is consciously accepted as debt rather than fixed now, log it here with: **Priority** (High/Medium/Low), **Owner** (which card/component it belongs to), **Review Version** (which review cycle surfaced it — e.g., "Cohesion Review v1"), and **Target Resolution** (which future phase/card revision should address it). This is what prevents the same recommendation from being silently rediscovered every six months
- **Future validation plan** — a named list of what Dimensions 12-14 surfaced as things to verify operationally once Phase 1 exists, distinct from blocking issues, each with the measurable pass criteria required in Dimension 14
- **Overall verdict**: Freeze-ready / Freeze-ready-with-minor-patches / Not-yet-freeze-ready, with the specific reasoning tied back to which dimension(s) drove that verdict — and explicitly noting that this verdict covers architecture validation only, not operational verification (Dimension 14's distinction)

---

## Recommended Review Process (How to Actually Run This)

1. **Run one complete review** using this rubric. Capture findings only — do not edit the architecture documents during the review itself.
2. **Run a second, independent review** using the exact same frozen rubric version. Again: findings only, no edits.
3. **Merge the findings.** Do not average scores. Reconcile Evidence, Confidence, Risk, and Recommended Action across both reviews — this naturally produces consensus where the two reviews agree, and surfaces genuine disagreement where they don't, rather than hiding it behind an averaged number.
4. **Create one consolidated action list** from the merged findings. Every action references: Card, Section, Rubric Dimension, Priority, and Reason — this becomes the actual implementation/editing backlog.
5. **Apply the edits.** Only after all actions from step 4 are addressed (fixed, or explicitly logged to the Architectural Debt Register as accepted debt) should the cards be considered candidates for freeze.
6. **Run one final confirmation review.** This should produce no new blocking issues, only minor non-blocking refinements, and a verdict of Freeze-Ready. Only then does Phase 1 implementation begin.

