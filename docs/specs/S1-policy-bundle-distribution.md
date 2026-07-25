---
spec_id: S1
title: Policy Bundle Distribution and Activation
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-25
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 06 §4, Card 06 §11, Card 06 §12, Card 06 §20, Card 06 §21, Card 06 §22, Card 06 §25, tech-stack.md Decision #7, tech-stack.md Decision #11, ADR-002]
depends_on: [S5]
gates: [1.7]
source: identified by the External Stage 10 review as a missing implementation-governance abstraction — Card 06 §21 defines what properties a policy must have (signed, versioned, Tier 4/5 approval), but not the actual bundle lifecycle mechanism that gets a policy from source to enforced. Semantic owner per stage-10-reconciliation.md: Card 06 §4/§21/§22
---

# S1 — Policy Bundle Distribution and Activation

**Status:** 1.0 (Approved)

## 1. Purpose

ADR-002 centralizes all authorization decisions in a single logical Policy Authority, implemented as OPA compiled to WASM and evaluated in-process (tech-stack.md Decision #7). Card 06 §21 establishes what properties the policies that authority enforces must have: signed and versioned like §15's instruction artifacts, Tier 4/5 changes requiring human approval, Tier 5 changes requiring dual control. What's missing is the mechanism connecting those two things — how does a policy actually get from Rego source to an enforced decision inside the running OPA evaluator, safely, with a way back out if it's wrong?

This spec defines that lifecycle: compile → sign → activate → verify-at-load → rollback. It is scoped to the single-process deployment S5's Minimum Conformance Profile establishes for Phase 1 — not a distributed bundle-synchronization mechanism, which stays out per S5 §3's explicit note that this profile "doesn't change the model, it constrains everything else around it to match a single-process reality."

## 2. Non-Goals

This specification does not define:

- distributed or multi-instance policy bundle synchronization — out of scope per S5 §3/§5, "Not yet phase-assigned." §8 states which invariants must survive into a future distributed implementation, without specifying how one would achieve them.
- the content of any specific policy (which agents get which permissions) — that's what the Rego source encodes, not this spec's concern
- ABAC+JIT decision logic itself (Card 06 §4) — this spec is about how the bundle containing that logic gets activated, not how the logic evaluates once active
- Tier 5 dual-control *workflow tooling* (the actual UI/process two humans use to co-approve) — this spec defines that dual control is required and what must be true before/after, not how the approval interface works
- Agent Identity, signing keys, or JIT credential issuance — that's S2's territory; this spec assumes Card 06 §21's signing model (same as §15's instruction artifacts) without redefining the signing mechanism itself
- a bundle approval queue — approval and activation are one coupled sequence (§4); this spec does not define pending-approval queue management, because no such queue exists

## 3. Definitions

**Policy bundle:** the compiled WASM artifact produced from Rego policy source, along with its accompanying metadata — `policy_version`, `policy_bundle_hash`, signature, and the Tier 4/5 approval record that authorized it. This is the unit of distribution and activation; individual Rego files are not activated independently. **Once signed, a policy bundle is immutable** — any content change produces a new bundle with a new hash, never a mutation of an existing signed bundle. This immutability guarantee holds for the full retention period (§4), including long-term/archival storage — a retained bundle must remain verifiable against its original hash and signature for as long as it is retained, not just at the moment of its original activation.

**Activation:** the point at which a policy bundle becomes the one the in-process OPA evaluator uses to answer authorization queries. Per S5 §3, there is exactly one active bundle in a Phase 1 deployment — no partial rollout, no A/B bundle serving, since there is only one process to serve from.

**Registered version:** the `policy_bundle_hash` value recorded as authoritative at the time of the bundle's Tier 4/5 approval — this is what AVS-PS-003's trigger checks a presented bundle's hash against. **Registration is transactional with approval** — the same event, not two sequential steps that could be interrupted between them. A bundle cannot be Approved without simultaneously becoming the registered version; there is no intermediate state where a bundle is approved but not yet registered.

**Rollback target:** the specific previously-active, already-verified bundle a rollback re-activates. A rollback target is always a bundle that has previously passed activation successfully — never an unverified or newly-presented bundle.

## 4. Bundle Lifecycle

**States:** `Draft` (Rego source, not yet compiled) → `Compiled` (WASM artifact exists, unsigned) → `Signed` (immutable, hash computed) → `Approved` (Tier 4/5 human approval recorded, dual control if Tier 5 — this is also the point the `policy_bundle_hash` becomes the registered version, §3) → `Active` (currently answering queries) → `Superseded` (was Active, has been replaced by a later activation). There is no separate "Rolled Back" state — rollback re-activates a Superseded bundle, returning it to `Active`; no separate "Revoked" state — nothing in Card 06 grounds a revocation concept distinct from a bundle simply remaining Superseded and never re-activated.

**Approval-to-activation coupling.** At most one bundle may be in the `Approved` state awaiting activation at a time. Approval and activation happen as one coupled operational sequence — approve, then immediately attempt activation — not a queue of approved-but-pending bundles. If activation fails (§4's Verify-at-load step), the bundle simply does not reach `Active`; it does not enter an indefinite pending state, and a new approval-and-activation attempt is required to try again.

**Compile.** Rego policy source is compiled to a WASM bundle — this is tech-stack.md Decision #7's own stated requirement ("policies require their own compilation step before evaluation"). Compilation happens before signing, never after — a signature covers the compiled artifact that will actually be loaded, not source that still has to be transformed.

**Sign.** The compiled bundle is signed using the same integrity model Card 06 §21 specifies for §15's instruction artifacts (Git commit signing, tech-stack.md Decision #11). The bundle's `policy_bundle_hash` is computed over the compiled WASM artifact and recorded alongside the signature. Once signed, immutable (§3).

**Approve.** Per Card 06 §21: Tier 4/5 policy changes require human approval; Tier 5 changes require dual control (two-person review), no exception. The approval record — who approved, at what tier, when — becomes part of the bundle's metadata. Policy Server administrative actions (approving a policy change) are themselves Tier 5 actions per §21 — the approval step is held to the same standard as the highest-risk actions the resulting policy will enforce.

**Activate.** Activation MUST NOT begin unless: compilation succeeded, the signature is valid, a sufficient approval record exists (Tier 4/5, dual control if Tier 5), and the bundle's metadata is complete. Activation is atomic: the OPA evaluator either finishes loading the new bundle and switches to it entirely, or the activation attempt fails and the evaluator continues serving the prior bundle unchanged. There is no intermediate state where some queries are answered by the old bundle and some by the new one within a single activation attempt. Activation attempts are serialized — at most one activation is in flight at a time; a concurrent activation request while one is already in progress is rejected, not queued or interleaved. A rejected request is not automatically retried — retrying means submitting a fresh activation request once the in-flight one completes, not an automatic retry loop initiated by the system.

**Verify-at-load.** Before activation completes, the evaluator verifies the same four preconditions above. Failures are classified into two categories, since they warrant different operational responses:
- **Verification failure** — invalid signature, hash mismatch, or missing/insufficient approval record (S1-003/S1-004)
- **Load failure** — the bundle is properly signed and approved, but the WASM artifact itself fails to parse or load (a corruption or build-tooling problem, not a security violation)

Either failure category means the activation attempt fails closed — the prior bundle remains active, satisfying AVS-PS-003.

**Rollback.** Rolling back means re-activating a Superseded bundle (the rollback target, §3) through the same Activate/Verify-at-load path — rollback is not a distinct code path from forward activation, it's activation targeting an older, already-verified bundle. The rollback target's own `policy_version`/`policy_bundle_hash` are unchanged and reused exactly as they were — rollback does not create a new bundle or a new hash. What is new is the *activation event* itself: a new `activation_id` and `activation_timestamp`, referencing the rollback target's `previous_bundle_hash` (the bundle being superseded by the rollback) as well as the hash being restored.

**Version logging.** Every policy decision logs `policy_version` and `policy_bundle_hash` alongside the decision itself (Card 06 §11's Security Event Schema, extended per §21), plus `risk_tier_version` and `data_classification_version` per §22. Every activation event (forward or rollback) additionally logs `activation_id`, `previous_bundle_hash`, `initiated_by`, `approval_record_id`, and `activation_timestamp` — an audit record states not just what was decided but which exact version of every governing artifact produced that decision, and exactly how that version came to be active.

**Retention.** At minimum, the immediately-superseded bundle MUST be retained to support rollback (S1-005 below cannot function otherwise). Retained bundles remain immutable and verifiable against their original hash and signature for as long as they are retained (§3). Broader retention policy — how many historical versions, when a Superseded bundle may be deleted — is an operational decision for the `core/security/` implementation, not fixed by this spec.

## 5. Scope

| Concern | In scope for S1 | Out of scope |
|---|---|---|
| Compile → sign → approve → activate → verify → rollback lifecycle, with explicit states | Yes | — |
| Atomic activation, serialized (single-process) | Yes | Multi-instance/distributed activation — Not yet phase-assigned (S5 §5) |
| Tier 4/5 approval, Tier 5 dual control enforcement (that it's required) | Yes | The approval workflow's UI/tooling |
| Signature and hash verification at load time, 2-way failure classification | Yes | The signing mechanism itself (tech-stack.md Decision #11's territory) |
| Full audit event schema for decisions and activation events | Yes | The broader Security Event Schema itself (Card 06 §11's territory) |
| Break-glass emergency policy exception | Yes, per §25's existing constraints | Redefining break-glass hardening rules — already fully specified in §25 |
| Minimum retention (immediately-superseded bundle) | Yes | Full retention policy (count, deletion criteria) |
| ABAC+JIT evaluation logic | No | Card 06 §4's territory |
| Agent identity / JIT credentials | No | S2's territory |
| Approval queue management | No | No such queue exists (§2, §4) |

## 6. Ownership Boundaries

- **Card 06 §21** owns *what properties* a policy must have (signed, versioned, tiered approval) — a card-level architectural requirement.
- **S1 owns** the *mechanism* that produces and activates a bundle satisfying those properties — compile/sign/approve/activate/verify/rollback, and the state machine governing legal transitions.
- **Card 06 §4's ABAC+JIT logic** owns what happens *after* activation — evaluating a query against whichever bundle is currently active. S1 does not touch evaluation logic, only which bundle is being evaluated against.
- **tech-stack.md Decision #11** owns the signing mechanism itself (Git commit signing); S1 consumes it, does not redefine it.

## 7. Architectural Invariants

- **Exactly one policy bundle is active at any time** in a Phase 1 (single-process) deployment.
- **At most one bundle is Approved-but-not-yet-Active at any time** — no approval queue (§4).
- **Fail-closed on any load-time verification or load failure.** Either failure category MUST result in the prior bundle remaining active — never a partial load, never an unverified or corrupted bundle taking effect "temporarily."
- **Every policy decision is attributable to an exact bundle version**, and every activation event is attributable to who initiated it and what approval authorized it.
- **A signed bundle is immutable**, for the full duration it is retained. Any content change is a new bundle with a new hash, never a mutation.

## 8. Future Distributed Deployment — Invariants That Must Survive

This spec's mechanism is single-process (§1, per S5). If a future phase moves to a distributed or multi-instance deployment, two invariants from this spec must still hold, regardless of how that future implementation achieves them: **activation remains atomic from the perspective of any single query** (no query is ever answered by a partially-activated bundle), and **verification failures fail closed** (an invalid or unapproved bundle never takes effect, distributed or not). This spec does not speculate about the mechanism a distributed implementation would use to achieve these — that's a future spec's job, triggered by an actual multi-node deployment decision, not invented here.

## 9. Scenarios (Given/When/Then)

```gherkin
Scenario: S1-001 — A correctly signed, approved bundle activates atomically
  Given a compiled policy bundle with a valid signature, correct hash, and a recorded Tier 4/5 approval (dual control if Tier 5)
  When the bundle is presented for activation
  Then the OPA evaluator loads it completely before switching, or not at all
  And once switched, all subsequent policy queries are answered by the new bundle
  And no query during the switch is answered by a partially-loaded bundle
  # related_avs: AVS-PS-002 (this establishes the activation mechanism the denial scenario runs against)

Scenario: S1-002 — Failed activation preserves the prior bundle
  Given a policy bundle activation attempt is in progress
  When any part of the activation fails, whether a verification failure or a load failure (§4)
  Then the OPA evaluator continues serving the previously active bundle, unchanged
  And no query is ever answered by a partially-activated or corrupted bundle state
  And the failure is logged with a category distinguishing verification failure from load failure

Scenario: S1-003 — An unsigned or tampered bundle fails closed at load time (verification failure)
  Given a policy bundle without a valid Git commit signature, or whose actual hash does not match its recorded policy_bundle_hash
  When activation is attempted
  Then the evaluator refuses to load it, classified as a verification failure
  And the prior bundle remains active
  And an integrity violation event is logged per Card 06 §11/§12
  # verifies: AVS-PS-003

Scenario: S1-004 — A bundle without sufficient tiered approval fails closed at load time (verification failure)
  Given a policy bundle change classified as Tier 4/5, or Tier 5 specifically
  When activation is attempted without a recorded approval at the required tier (dual control for Tier 5)
  Then the evaluator refuses to load it, classified as a verification failure
  And the prior bundle remains active
  And this is logged distinctly from a signature/hash failure (S1-003), though both are verification failures

Scenario: S1-005 — Rollback re-activates a prior bundle through the same verified path, without creating a new bundle
  Given a previously active, now-Superseded bundle B1 has been superseded by B2 (currently Active)
  When a rollback to B1 (the rollback target) is initiated
  Then B1 is re-activated through the same Activate/Verify-at-load mechanism as any forward activation (§4)
  And B1's original signature, hash, and approval record are re-verified, not assumed valid from its prior activation
  And B1's policy_version/policy_bundle_hash are unchanged and reused — no new bundle is created
  And the rollback event itself is logged with a new activation_id and activation_timestamp, referencing B2's hash as previous_bundle_hash

Scenario: S1-006 — Every policy decision logs its exact governing bundle version
  Given an active policy bundle answering an authorization query
  When the Policy Server issues a decision (grant or deny)
  Then the decision is logged with policy_version, policy_bundle_hash, risk_tier_version, and data_classification_version (Card 06 §22)
  And an auditor can determine exactly which policy logic produced this specific decision, not just that some policy did
  # related_avs: AVS-PS-002 (the denial scenario's pass criterion requires policy_decision_id, which this scenario's logging makes possible)

Scenario: S1-007 — Every activation event logs full provenance
  Given any activation attempt, forward or rollback
  When the attempt succeeds or fails
  Then the event is logged with activation_id, previous_bundle_hash, initiated_by, approval_record_id, and activation_timestamp
  And this holds whether the activation succeeded, failed verification, or failed to load

Scenario: S1-008 — Concurrent activation attempts are serialized, not interleaved
  Given an activation attempt is already in progress
  When a second activation request arrives before the first completes
  Then the second request is rejected, not queued and not interleaved with the first
  And the requester receives an explicit "activation already in progress" response, not a silent no-op
  And retrying means submitting a fresh activation request after the in-flight one completes, not an automatic system-initiated retry

Scenario: S1-009 — Policy Server administrative actions are themselves Tier 5
  Given a human is approving a policy bundle change
  When that approval action is logged
  Then it is logged as a Tier 5 action, per Card 06 §21 — administering the enforcer is held to the same standard as the highest-risk actions it enforces
  And this applies regardless of the tier of the policy change being approved — the act of approving is Tier 5, even if the resulting policy governs Tier 2 actions

Scenario: S1-010 — Break-glass activation follows §25's existing hardening constraints without redefinition
  Given an emergency requires bypassing the normal Tier 4/5 approval path for a policy change
  When a break-glass grant is used to activate a bundle
  Then the grant is time-boxed, scope-limited, justification-required, and automatically expiring per Card 06 §25
  And every action taken during the active grant is logged as a Tier 5 audit event
  And the grant triggers immediate notification to designated reviewers, not after-the-fact notice
  # related_avs: AVS-PS-003 (break-glass is the one case where the "requires approval" branch of S1-004 is deliberately bypassed, per an already-existing exception — not a new one this spec invents)
```

## 10. What this spec explicitly does NOT do

- Does not implement distributed/multi-instance bundle synchronization — correctly out of scope for Phase 1 (S5 §5, "Not yet phase-assigned"); §8 states only which invariants must survive into a future distributed implementation, not how
- Does not define ABAC+JIT evaluation logic — Card 06 §4's territory, unchanged
- Does not define the signing mechanism itself — consumes tech-stack.md Decision #11, doesn't redefine it
- Does not redefine break-glass hardening — Card 06 §25 already fully specifies it; this spec only confirms activation respects it
- Does not touch Agent Identity or JIT credential issuance — S2's territory
- Does not invent bundle states, revocation concepts, or queue-management complexity beyond what Card 06 grounds — the 6-state lifecycle (Draft/Compiled/Signed/Approved/Active/Superseded) is deliberately minimal, and rollback/retention decisions favor the simplest answer consistent with the architecture rather than the most flexible one
- Does not change ADR-002's centralization decision or its accepted tradeoffs — this spec is the mechanism that makes §20/§21's mitigations for that tradeoff actually operable