# Ecosystem Cohesion Review — Independent Assessment
## Part 2 of 4: Dimensions 5-8 (Missing Abstractions, Production Readiness, Terminology Consistency, Scope Discipline)

*(Continued from Part 1. Same Architecture Version, Reviewer Independence, and Assumptions apply — see Part 1 header.)*

---

## Dimension 5 — Missing Abstractions

**Score: 6/10**
**Confidence: High** — these are concrete, checkable absences, not speculative gaps.

**Evidence (Strong) — three referenced-but-undrafted files:**
1. **Agent Contract template** (`docs/architecture/agent-contract-template.md`, file 0.12) — Card 01 §1 states *"every agent in this ecosystem should be instantiated from a contract"* and points to this file by path. Cards 02, 03, and 04 each model their own Contract type (Tool/Memory/Skill) on this concept. The file itself is referenced by path in at least four cards but its actual field-by-field content does not appear in any of the seven cards under review.
2. **AGENTS.md** (file 0.10) — Card 06 §15 calls this "the architectural North Star" and requires it be a *signed instruction artifact* the Harness verifies before loading. It is treated as load-bearing infrastructure by Card 06, yet — like the Contract template — its content is not present in this card set.
3. **`docs/architecture/model-routing-table.md`** (file 0.9) — Card 01 §4 originates the routing concept; Card 06 §26 imposes a security constraint on it (classification-aware routing); Card 07 §11 specifies its *required schema* in detail. Three cards build requirements onto a file whose actual content doesn't exist in the set reviewed.

This is not a flaw in any individual card — each one correctly defers to "Phase 0, file 0.X" rather than inventing content inline. But scored strictly against the dimension's question ("is there a concept the ecosystem clearly needs that has no home anywhere"), the honest answer for these three is: **the home is named, but empty.** A reviewer with zero institutional context (as instructed) cannot distinguish "deferred to a Master Plan file not included in this review" from "genuinely missing" without that Master Plan being supplied — and it was not part of this review's input.

**Finding (Medium evidence) — Distinguishing Missing from Incorrect, per the rubric's required split:** this is a **Missing** finding, not an **Incorrect** one — no card claims ownership of this content and gets it wrong; the content is simply absent. The contrast case from Dimension 1 (Card 04/06's Skill-activation cross-reference gap) was closer to an Incorrect-adjacent friction; this is cleanly Missing.

**Finding (Medium evidence) — the untrusted-context-boundary mechanism:** Card 06 §26 states *"Context assembly (Card 03 §2-3) must preserve source boundaries so that retrieved, tool, or A2A content cannot override system or policy instructions."* This is a real requirement imposed on Card 03's domain. However, Card 03 itself (§2-3, the sections cited) contains no discussion of source labels, instruction/data delimiters, or any mechanism for "preserving source boundaries" — the requirement is asserted from Card 06's side but not received, acknowledged, or operationalized anywhere in Card 03. This is a legitimate missing abstraction: the *mechanism* by which untrusted content is kept from being treated as instruction does not exist in either card.

**Risk if Ignored:** High for the three named files (0.9, 0.10, 0.12) — Dimension 10 (Implementation Feasibility) directly depends on whether these exist; an implementer cannot build Phase 1 without them despite all 7 reference cards being individually complete. Medium for the untrusted-context-boundary mechanism — this is a real prompt-injection defense with no implementation home yet.

**Recommended Action:** Draft files 0.9, 0.10, and 0.12 before considering Phase 0 closed — they are prerequisites the cards themselves assume exist. For the untrusted-context-boundary mechanism, add an explicit subsection to Card 03 (or a forward-reference acknowledging the requirement and naming where the mechanism will be specified) so the requirement isn't one-directional.

---

## Dimension 6 — Production Readiness (Failure Modes & Override Authority)

**Score: 9/10**
**Confidence: High**

**Evidence (Strong):** This is the ecosystem's strongest dimension by a clear margin, concentrated almost entirely in Card 06. I checked every high-authority component the rubric specifically names:
- **Policy Server**: both unavailability (§20, explicit fail-closed table by tier) and compromise (§21, signed/versioned policies, Tier 5 dual control, policy-version logging) are addressed as *distinct* failure modes — this is genuinely sophisticated; most architectures conflate "down" and "compromised."
- **Circuit Breaker**: trigger scope (run/agent/tool/system-level), threshold/release criteria, and an explicit non-ownership statement of execution state (§11: *"it never directly rewrites execution state... signals the Harness/Orchestrator to transition"*) are all present.
- **Registries** (Tool/Skill): both write-time signing and runtime (not just write-time) verification are specified (§16), with failure scoped to the single bad entry rather than the whole registry.
- **Break-glass**: time-boxed, scope-limited, justification-required, auto-expiring, with mandatory notification and post-incident review (§25) — a genuinely hardened override path, not a convenience loophole.

**Finding (Medium evidence) — one gap, outside Card 06's scope:** none of the seven cards define what happens if the **Harness/Orchestrator itself** (Card 01's core execution loop, not the Policy Server it calls into) crashes mid-trajectory — as distinct from a logical `Stop` outcome reached via the Decide stage. Card 01 §2 defines `Stop` as a deliberate decision output; it does not address an *involuntary* process crash (e.g., out-of-memory, unhandled exception in the harness code itself) and what state that leaves behind. Given Card 06 is careful to address Policy Server crash/compromise as distinct cases, the same rigor doesn't appear to have been applied to the Harness itself, which is arguably an even more central component.

**Risk if Ignored:** Medium — this is implementation-detail-adjacent (crash recovery is often handled at the infrastructure level regardless), but for a system claiming production-grade rigor everywhere else, this is a real asymmetry worth naming rather than leaving implicit.

**Recommended Action:** Add a brief note to Card 01 (or as a Phase 1 implementation requirement in the Master Plan) addressing Harness-level crash recovery — at minimum, what state is preserved/resumable versus lost on an involuntary process failure, distinct from the deliberate `Stop` outcome.

---

## Dimension 7 — Cross-Card Terminology Consistency

**Score: 6/10**
**Confidence: High** — this score is driven by one specific, directly-quotable, repeatable finding, not impression.

**Evidence (Strong) — the "structured state" rename, checked specifically per the rubric's required test:** I searched all seven cards for "structured state." It appears zero times as an active term — only as historical correction notes in Card 01 §5 (*"not 'structured state,' to avoid colliding..."*) and Card 03 §5 (*"renamed from 'structured state'"*). This rename was fully and correctly propagated. **Strong evidence this specific historical risk was avoided.**

**Evidence (Strong) — a genuine, repeatable terminology/fact inconsistency, the same root issue flagged in Part 1 but now confirmed as a distinct Dimension 7 violation in its own right:** Card 06 §3's risk tier table states explicitly: Tier 3 ("Reversible write") has a default approval requirement of *"None, but reversible-write audit logged"* — no human review by default. HITL is first required starting at **Tier 4** (*"HITL required (Card 05 §4 tier 4)"*). Card 07 §6, however, states: *"Human review — final approval, required for anything touching **Tier 3+** per Card 06 §3."* This is a direct, checkable contradiction between what Card 07 claims Card 06 says, and what Card 06 actually says, citing the exact same section (§3) as its source. This is not a matter of interpretation — Card 07 misquotes Card 06's own table.

I want to be precise about severity here: this is a **Strong-evidence, high-confidence, unambiguous factual error**, not a stylistic inconsistency. A reviewer or implementer who trusts Card 07's citation and doesn't independently re-verify against Card 06 §3 will build a code-review gate that's stricter than the architecture actually specifies — requiring human sign-off on Tier 3 changes that Card 06 deliberately classified as not requiring it (audit-logged only, specifically to avoid approval-fatigue overhead on routine reversible writes).

**Risk if Ignored:** Medium-High — this directly affects how restrictively the CI/CD code-review gate (Card 07 §6, §8) gets implemented, with a real operational cost (unnecessary human approval friction) if implemented as literally written, or a real security gap (under-enforcement) if an implementer instead trusts Card 06's table and silently ignores Card 07's stricter claim without noticing the two disagree.

**Recommended Action:** Correct Card 07 §6 to read "Tier 4+ per Card 06 §3 (or Tier 3 specifically where Card 06 §14's regulated-project override applies)" — matching Card 06's actual table and its own documented Project 1/3 exception.

---

## Dimension 8 — Scope Discipline (No Unjustified Expansion)

**Score: 9/10**
**Confidence: High**

**Evidence (Strong):** Scope discipline is consistently well-maintained. Every card I reviewed explicitly names what it does *not* own, rather than silently overstepping: Card 02's header explicitly disclaims risk tiers/permissions/sandboxing; Card 03 §11 explicitly disclaims *who* may authorize memory deletion (Card 06's job); Card 04 §6/§8 stays within Skill lifecycle and explicitly defers certification (correctly picked up by Card 05 §7) and security signing (correctly picked up by Card 06 §15) rather than absorbing either itself.

**Finding (Medium evidence) — Card 06's breadth, evaluated specifically for whether it constitutes scope creep or justified debt absorption:** Card 06 is by far the longest and most structurally complex card (26 numbered sections). I evaluated whether this represents Security becoming a "god layer" (the specific risk the rubric's Dimension 11 — Architectural Simplicity — and Dimension 1 both warn about). My assessment: **this is justified debt absorption, not scope creep.** Every one of Card 06's sections opens by naming exactly which prior card deferred that specific topic to it ("Discharging Card 02 §4/§8," "Discharging Card 03 §11," etc.) — the breadth was accumulated by other cards' deliberate, explicit deferrals, not unprompted expansion by Card 06 itself. This is a meaningfully different failure mode than scope creep, and I don't think it should be scored as a Dimension 8 problem — though it is worth flagging under Dimension 11 (Architectural Simplicity) in Part 3, since breadth and necessity are related but distinct questions.

**Finding (Weak evidence, flagged for completeness only) — Card 06 §26's reach into Card 03's territory:** the untrusted-context-boundary requirement (also flagged under Dimension 5) is Card 06 imposing a behavioral constraint on Context Engineering's domain. Given Card 06 sits at the base of the Runtime Stack and explicitly governs every layer above it, this is consistent with the stack's own declared direction of authority — not a violation. I flag it here only because it's the kind of cross-card reach that *could* be scope creep in a less disciplined architecture, and is worth a second look if the stack's directionality is ever revisited.

**Risk if Ignored:** Low for this dimension specifically — the real action items live in Dimensions 5 and 11, not here.

**Recommended Action:** None required for Dimension 8 itself. Cross-reference: see Dimension 11 in Part 3 for whether Card 06's accumulated breadth, while justified in origin, still warrants a "could this be simplified" pass independent of why it grew.

---

*(Continued in Part 3: Dimensions 9-11 — Long-Term Maintainability, Implementation Feasibility, Architectural Simplicity)*
