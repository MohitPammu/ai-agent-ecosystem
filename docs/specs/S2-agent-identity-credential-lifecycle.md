---
spec_id: S2
title: Agent Identity, Signing Key, and JIT Credential Lifecycle
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-25
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 01 §2, Card 06 §3, Card 06 §4, Card 06 §11, Card 06 §12, Card 06 §15, Card 06 §18, Card 06 §20, Card 06 §21, Card 06 §22, agent-contract-template.md §1, tech-stack.md Decision #11]
depends_on: [S1]
integration_dependencies: [S7 quarantine events — invalidation trigger, not a hard prerequisite]
gates: [1.7]
source: identified by the External Stage 10 review as a missing implementation-governance abstraction — Card 06 §4 establishes the conceptual model and §18 lists the JIT token's required properties, but neither defines the actual issuance/binding/revocation mechanism. Semantic owner per stage-10-reconciliation.md: Card 06 §§4/15/18/21
---

# S2 — Agent Identity, Signing Key, and JIT Credential Lifecycle

**Status:** 1.0 (Approved)

## 1. Purpose

Card 06 §4 establishes that every agent gets a unique cryptographic identity, not a shared service credential, and that JIT (Just-In-Time) token downscoping is the mechanism behind Card 02 §5's "assignment authority belongs to the Policy Server." §18 lists what properties a JIT token must have: short-lived, bound to identity + `run_id` + intent + resource scope, revocable mid-run, single-use or narrowly-scoped, never logged in plaintext, auto-invalidated on Stop/Escalate/quarantine. Neither section defines the mechanism that actually produces an agent's identity, the concrete claims a token carries, who validates it at the point of use, or what happens across the identity's full lifecycle — rotation, compromise, retirement.

This spec defines that lifecycle. It depends on S1 because JIT issuance is a Policy Server decision made against whichever policy bundle is currently active (S1 §4). It has an integration dependency, not a hard prerequisite, on S7: Circuit Breaker quarantine events are one of several triggers for token invalidation, but S2's own mechanism (issuance, validation, expiry, retirement) functions independently of S7's existence — S7 emits a signal S2 already knows how to act on, the way S1 activates a bundle S2's ABAC decisions already know how to consult.

## 2. Non-Goals

This specification does not define:

- ABAC decision logic itself — Card 06 §4's territory
- policy bundle activation or the Policy Server's own integrity model — S1's territory
- Circuit Breaker trip detection or aggregation — S7's territory
- instruction artifact signing (contracts, AGENTS.md, Skill bodies) — Card 06 §15 generally, S3 specifically
- multi-node identity federation — out of scope per S5's single-process profile; §16 states which invariants must survive a future multi-deployment implementation
- a formal canonicalization *algorithm* for attestation payloads, or a specific cryptographic construction for `token_id_hash` — these are implementation-technology choices, the same boundary S1 drew around its own signing mechanism (S1 §6, "tech-stack.md Decision #11 owns the signing mechanism itself; S1 consumes it")
- exact clock-skew tolerance or TTL numeric values — Policy Server configuration, the same deferral pattern S1 used for threshold values
- key-compromise incident response as a standalone process — compromise recovery reuses the existing Tier 4/5 approval discipline (§21), it is not a new workflow this spec invents
- identity or credential federation beyond single-deployment uniqueness — §14 states the uniqueness domain explicitly, does not build toward multi-tenant namespacing

## 3. Definitions

**Agent identity:** a unique cryptographic identity bound to one approved Agent Contract's `agent_id` (agent-contract-template.md §1), and to the specific contract version/approval-record that governed its issuance (§14). One identity per contract; an identity's lifecycle is `Active` → `Retired` (§4). Future specifications may extend this to additional states without changing current semantics — `Active` and `Retired` are the complete Phase 1 model, not a placeholder pair assumed to grow on their own.

**Signing key:** private key material bound to an agent identity, used for Risk-Stratified Attestation (§8) — never for JIT token issuance, which is a Policy Server function, not a signing operation. An identity may have multiple sequentially versioned signing keys over its lifetime, but at most one `Active` key at a time (§4). Distinct from policy-bundle signing (S1) — different keys, different purposes.

**JIT token:** a short-lived credential issued by the Policy Server, bound to exactly one (agent identity, `run_id`, intent, resource scope) tuple, carrying a defined claims set (§5), with lifecycle `Issued → Active → Consumed | Revoked | Expired` (§5). No terminal state (`Consumed`, `Revoked`, `Expired`) ever transitions back to `Active`.

**`token_id_hash`:** a one-way audit correlation identifier derived from the token's unique identifier — never derived from low-entropy data, never accepted as authentication material, never a substitute for the token itself. The only representation of a token that ever appears in any log, trace, span, metric label, crash dump, or debug output.

**Rollback target / registered version:** as defined in S1 §3 — S2 does not redefine these, only consumes them via `policy_decision_id`/`policy_version` binding (§5).

## 4. Identity and Signing-Key Lifecycle

**Identity states:** `Active` → `Retired`. An identity is `Active` from issuance (governed contract approval, §21's Tier 4/5 discipline — Tier 5 specifically if the contract declares Tier 5 capability, since issuing an identity capable of Tier 5 attestation is itself held to that standard, per the same "administering the enforcer" principle §21 already applies to Policy Server admin actions) until its governing contract is retired. There is no separate `Suspended` state: a temporary inability to act is already fully covered by Circuit Breaker quarantine (S7) invalidating the identity's tokens at agent-scope — inventing a second, identity-level suspension mechanism would duplicate what quarantine already does.

**Contract revisions.** A routine revision to an already-approved Agent Contract (e.g., updated `state_owned` keys, per S4) requires re-approval of the contract itself, per Card 06 §15's signing model, but does not require a new identity or key — the same identity persists across such revisions, now bound to the new approval record (§14). Two kinds of revision do require identity/key reconsideration: a change to the `agent_id` binding itself (which is a new agent, not a revision of the existing one) and a change to the contract's declared Tier 5 capability (which changes the approval tier §4 requires for the identity's own standing, potentially requiring re-issuance at the correct tier).

**Retirement.** When an Agent Contract is retired, its identity transitions to `Retired`: all active JIT tokens for that identity are invalidated immediately, no new token or attestation signing operation is permitted, and historical attestations produced while the identity was `Active` remain verifiable (§8's public-key retention requirement, §3).

**Key states:** `Active` → `Retired`, sequential and versioned (`key_id`, monotonically increasing version). At most one `Active` key per identity at a time. Routine rotation: a new key becomes `Active`, the prior key transitions to `Retired` — the retired key can no longer create new attestations, but its public verification material is retained (§8) so historical attestations it produced remain verifiable. Compromise response: a suspected or confirmed compromised key is disabled immediately (transitions to `Retired` outside the routine rotation path), new JIT issuance for the identity is blocked until a replacement key is issued, and replacement issuance goes through the same Tier 4/5 approval discipline as original issuance — not a separate incident-response process.

**Custody.** Private signing key material is generated and retained by a trusted key-custody component, never exposed to model context, tool output, logs, telemetry, or agent-controlled code in any form. An agent requests a signing operation through the Harness; the Harness invokes the custody component, which performs the signing and returns only the resulting signature — never the key. This boundary exists because agent-visible private key material would expose Tier 5 attestation to compromise via prompt injection or tool compromise, defeating the purpose of the attestation entirely. The exact Phase 1 storage technology for the custody component is implementation-defined, not fixed by this spec.

**Idempotent, atomic issuance.** Reprocessing the same approved Agent Contract version MUST return the existing identity record, never create a duplicate identity or key. Identity metadata and key generation are all-or-nothing: no `Active` identity may exist without usable key custody, and no orphaned `Active` key may exist without corresponding identity metadata.

**Uniqueness domain.** `agent_id` uniqueness is deployment-local, consistent with S5's single-process Minimum Conformance Profile — global or multi-tenant uniqueness is out of scope until a future multi-deployment decision actually requires it (§16).

## 5. JIT Token Lifecycle

**Claims.** Every JIT token carries: a unique token identifier, issuer (the Policy Server), subject (agent identity), the four-part binding (`run_id`, intent, resource scope — "resource scope" already names what a separate "audience" concept would duplicate), issued-at, expiry, `usage_mode` (`single_use` or `bounded_reuse`), and provenance back to the authorizing decision — `policy_decision_id`, `policy_version`, `policy_bundle_hash` (S1 §5's own logged fields, carried forward onto the token itself so a validator can confirm which decision authorized it without a separate lookup).

**States:** `Issued → Active → Consumed | Revoked | Expired`. No terminal state transitions back to `Active`. `Issued` and `Active` are effectively simultaneous for a token that passes point-of-use validation on its first presentation — the distinction exists to make issuance and first-use-eligibility separately observable, not to imply a meaningful gap between them.

**Usage modes.** `single_use`: valid for exactly one privileged action, consumed atomically on that use. `bounded_reuse`: valid for multiple uses within explicit bounds — resource set, operation set, and time window, all stated at issuance. A `bounded_reuse` token issued without explicit bounds is invalid; there is no unbounded reuse mode. Data-volume limits and inter-call sequence constraints are not required bound types for Phase 1 — nothing in the current architecture demonstrates a need for them, and adding them now would be speculative. Additional bounded dimensions may be introduced in future specifications if a real need emerges; this spec's three bound types are the complete Phase 1 set, not a partial list awaiting completion.

**Atomic single-use consumption.** The `Active → Consumed` transition is atomic. Under concurrent presentation of the same `single_use` token, exactly one use succeeds; every other concurrent presentation fails as already-consumed, and only one privileged side effect ever begins. This is what makes "single-use" an enforceable property rather than a label — a claim of single-use without atomic consumption does not prevent replay.

**Point-of-use validation.** The Harness is the final enforcement point for every privileged action — consistent with its role elsewhere in this architecture (e.g., S4 §6's state-schema ownership, memory access being harness-mediated). Before any privileged action begins, the Harness validates: token authenticity, the four-part binding against the requested action, expiry, usage state (not already `Consumed`/`Revoked`/`Expired`), and current revocation state. Any failed check prevents execution — no privileged operation begins before this validation succeeds.

**Revocation-state authority.** For Phase 1's single-process deployment, one authoritative credential-state registry (Postgres-backed, consistent with S5's single-instance profile) tracks current token state. This registry is operated by `core/security/` — the same module implementing S1's Policy Server and S7's Circuit Breaker — not by the Harness; the Harness queries it at point-of-use (§5 above) but does not own its state. Every use establishes current status against this registry — tokens are not treated as self-contained/self-validating for revocation purposes, since a self-contained token cannot reflect a revocation that happened after issuance. **If current status cannot be established, the privileged use fails closed** — this is the same fail-closed-on-unknown-state discipline Card 06 §20 already establishes for Policy Server unavailability, applied here to credential-state unavailability specifically.

**Mid-run revocation, including outage.** When S7 emits a quarantine event, S2 invalidates the affected tokens scoped exactly to the event's stated scope (§6). If the authoritative revocation-state registry itself is unavailable when a revocation needs to be recorded, the Harness blocks subsequent privileged actions locally rather than waiting for remote confirmation — it does not proceed merely because the revocation path failed, and it records that revocation confirmation was unavailable as its own event. This is the same "unknown status fails closed" principle applied to the specific case where the failure is in the revocation mechanism itself, not just an ordinary lookup.

**In-flight actions.** No new privileged action may begin once a token is invalidated. An action already granted and in progress before invalidation completes under its original grant — Phase 1's synchronous execution model (Card 01 §2) means most actions are atomic enough that "in-flight cancellation" beyond this isn't a meaningful additional state to model — but completing that action confers no further authority; any follow-on action requires its own valid, unrevoked token.

**Expiry.** Every token has mandatory expiry — no token is issued without one. Expiry duration is Policy Server configuration (not fixed by this spec, mirroring S1's own threshold-value deferral), but is bounded by the action's risk tier (Card 06 §3) — the exact bound is configuration, the requirement that higher risk tiers cannot have longer-lived tokens than lower ones is normative. Lifecycle invalidation (revocation, Stop, Escalate, quarantine) always overrides nominal time-based expiry — a token can become invalid before its stated expiry, never valid past it.

**Escalate and resume.** A run transitioning to `Escalate` (Card 01 §2) invalidates its currently active tokens — but per S7 §4, `Escalate` is not terminal; the run remains resumable while awaiting HITL response. Resuming a run after `Escalate` requires fresh Policy Server decisions and newly issued tokens — the invalidated tokens never reactivate, but the run itself is not forced to `Stop` just because its tokens were invalidated. This reconciles S2's invalidation behavior with S7's own definition of `Escalate` as non-terminal, rather than treating every `Escalate` as equivalent to `Stop`.

**Policy changes after issuance.** If a policy bundle supersession or rollback (S1) means previously-granted tokens should no longer be honored, that determination is the Policy Server's — S1/Card 06 §4's territory, not S2's. S2 applies whatever revocation instruction results, the same way it applies S7's quarantine events: as an instruction to execute, not a policy decision to originate.

**Failed issuance.** A Policy Server grant decision followed by a credential issuance failure (custody component unavailable, registry write failure, etc.) does not authorize any action — the failure is logged with `policy_decision_id`, and no privileged operation proceeds on the strength of the grant decision alone. A decision to grant and a successfully issued, usable credential are two different things, and only the second authorizes anything.

## 6. Quarantine Scope Mapping

S7 supports four quarantine scopes; S2 defines exactly which tokens each invalidates:

- **run-level:** all active tokens for the affected `run_id`
- **agent-level:** all active tokens for the affected agent identity, across all its runs
- **tool-level:** all active tokens whose resource scope targets the affected tool
- **system-level:** all active JIT tokens across the deployment (S7 §5's system-level trip, per S5 Appendix A's resolution that both Circuit Breaker scopes are Phase 1 requirements)

Tokens outside the affected scope retain their prior state unchanged — a run-level quarantine does not touch another run's tokens, an agent-level quarantine does not touch another identity's tokens.

## 7. Audit Events

Every lifecycle transition defined above produces its own audit event as it occurs (not a separate taxonomy maintained apart from the lifecycle definitions themselves): identity issued/retired; key issued/rotated/compromised; token issued/consumed/revoked/expired; point-of-use validation accepted/rejected; revocation-outage fallback triggered; attestation created/verification-failed. All such events use `token_id_hash` and `key_id` for correlation, never raw key or token material — this prohibition extends to every telemetry surface: exception messages, crash dumps, metrics labels, span attributes, dead-letter records, debug output, model context, and persisted tool arguments, not only the primary audit log.

## 8. Risk-Stratified Attestation

For Tier 5 actions specifically, Card 06 §12 requires a cryptographic signature bound to the agent's output. The attestation payload contains: the output's digest, the agent identity and `key_id` that signed it, `run_id`, `policy_decision_id`, timestamp, risk tier, and an `attestation_schema_version` field — the last of these following the same versioning discipline Card 06 §22 already applies to every other governed artifact (`policy_version`, `risk_tier_version`, `data_classification_version`), so that a future change to the payload's shape doesn't silently break verification of historical attestations produced under an earlier schema. Together these are sufficient to answer "which agent, using which key, under which authorization, produced this exact output, when." (Full technology-neutral canonicalization procedure is implementation-defined, per §2's Non-Goals — this spec states what must be verifiable, not the exact serialization format.)

Verification happens before a Tier 5 output is accepted or released — missing or invalid attestation causes rejection, the same fail-closed discipline applied throughout this spec. Historical public verification material (§4) is retained so that attestations produced under a since-retired key remain verifiable indefinitely.

**On what the attestation proves:** it proves that the trusted execution path associated with a specific agent identity finalized the recorded output, under a specific authorization, at a specific time. It does not independently prove that a language model "intended" or "authored" the output in any stronger sense — the attestation is a provenance and integrity guarantee, not a claim about model cognition.

## 9. Scope

| Concern | In scope for S2 | Out of scope |
|---|---|---|
| Identity issuance, `Active`/`Retired` lifecycle | Yes | Separate `Suspended`/`Pending`/`Compromised` identity states — covered by quarantine (token-level) instead |
| Signing key issuance, rotation, compromise response, custody boundary | Yes | Formal incident-response process — reuses existing Tier 4/5 approval |
| JIT token claims, states, usage modes, atomic consumption | Yes | Specific cryptographic token format (JWT/PASETO/opaque) — implementation-defined |
| Point-of-use validation (Harness as final enforcement point) | Yes | — |
| Revocation-state authority, fail-closed on unknown status, outage fallback | Yes | Operational ownership assigned to core/security/ (§5) |
| Quarantine scope-to-token mapping | Yes | Circuit Breaker trip detection itself (S7) |
| Escalate/resume reconciliation with S7 | Yes | — |
| Tier 5 attestation payload and verification | Yes | Canonicalization algorithm — implementation-defined |
| Audit events per lifecycle transition | Yes | A separate, independent audit taxonomy document |
| TTL/expiry requirement and risk-tier bounding | Yes | Exact numeric values, clock-skew tolerance — Policy Server configuration |
| Multi-node identity federation | No | §16 states survivable invariants only |
| ABAC decision logic, policy bundle activation, Circuit Breaker detection | No | Card 06 §4, S1, S7 respectively |

## 10. Ownership Boundaries

- **Agent Contract (§1 `agent_id`)** owns declaring which agent an identity belongs to, at which contract version.
- **S2 owns** the mechanism turning an approved contract into a live identity, the full key lifecycle, and the full JIT token lifecycle including point-of-use validation.
- **Card 06 §4's ABAC logic** owns deciding whether to grant a token; **S1** owns the policy bundle that decision is made against; **S2** owns everything from grant-decision to token-invalidated.
- **S7** owns deciding *that* a Circuit Breaker trip occurred and its scope; **S2** owns *acting* on that decision (§6).
- **The Harness** owns point-of-use validation and enforcement — S2 defines what must be checked, the Harness is where the check happens, consistent with its role elsewhere in this architecture.
- **`core/security/`** owns operating the credential-state registry itself (§5) — the same module implementing S1 and S7, not the Harness, which only queries it.

## 11. Architectural Invariants

- An identity may have sequential signing-key versions; at most one `Active` key per identity at a time.
- Raw private key material is never exposed to agent-controlled code, tools, model context, or any telemetry surface.
- Every token carries issuer, subject, the four-part binding, issued-at, expiry, and policy-decision provenance.
- The Harness, as final point-of-use validator, checks authenticity, binding, expiry, usage state, and revocation state before any privileged action begins.
- Failure to establish current credential status fails closed.
- Single-use consumption is atomic; `bounded_reuse` requires explicit bounds.
- Lifecycle invalidation (revocation, Stop, Escalate, quarantine) always overrides nominal time-based expiry.
- Contract retirement or key compromise blocks new issuance and invalidates active credentials immediately.
- Consumed, Revoked, and Expired tokens never return to `Active`.
- Identity and credential operations (issuance, rotation) are idempotent under retry.
- Historical public verification material is retained for the life of any attestation it could verify.

## 12. Future Multi-Deployment — Invariants That Must Survive

This spec's mechanism assumes deployment-local identity uniqueness and a single credential-state registry (§5, §4), per S5's single-process profile. If a future phase moves to multi-deployment, two invariants must still hold regardless of mechanism: **revocation-state fails closed when it cannot be established** (not "assumes valid" as a fallback), and **an identity's signing key never crosses a trust boundary it wasn't issued for** (i.e., federation, if it happens, extends verification, it does not relax custody). This spec does not speculate on how a future multi-deployment implementation achieves these — same boundary S1 §8 drew for its own future-distributed note.

## 13. Scenarios (Given/When/Then)

```gherkin
Scenario: S2-001 — Agent identity is issued only at contract approval, bound to agent_id and contract version
  Given an Agent Contract is approved per Card 06 §21's discipline (Tier 5 if the contract declares Tier 5 capability)
  When the identity issuance step runs
  Then a unique cryptographic identity and an associated Active signing key are produced
  And both are bound to the contract's agent_id and the specific contract version/approval record
  And no identity exists for any agent whose contract is not yet approved

Scenario: S2-002 — Identity issuance is idempotent and atomic
  Given the same approved Agent Contract version is processed more than once
  When identity issuance is retried
  Then the existing agent identity is returned, no second identity or key is created
  And no Active identity ever exists without usable key custody, and no orphaned Active key exists without identity metadata

Scenario: S2-003 — A JIT token carries its full claims set and is bound to exactly one tuple
  Given a Policy Server ABAC decision grants an agent's request
  When a JIT token is issued
  Then the token carries a unique identifier, issuer, subject, run_id, intent, resource scope, issued-at, expiry, usage_mode, and policy_decision_id/policy_version/policy_bundle_hash
  And the token is bound to exactly that agent identity, run_id, intent, and resource scope
  # verifies: AVS-PS-002

Scenario: S2-004 — Point-of-use validation is complete before any privileged action begins
  Given a presented JIT token
  When the Harness validates it at the privileged-action boundary
  Then authenticity, the four-part binding, expiry, usage state, and current revocation state are all checked
  And any single failed check prevents execution
  And no privileged operation begins before this validation succeeds

Scenario: S2-005 — Single-use consumption is atomic under concurrent presentation
  Given one valid single_use token
  When two concurrent uses present it simultaneously
  Then exactly one use atomically succeeds and transitions the token to Consumed
  And every other concurrent presentation fails as already-consumed
  And only one privileged side effect ever begins

Scenario: S2-006 — Raw token and key material never appear in any telemetry surface
  Given a JIT token is issued, used, or revoked, or a signing operation occurs
  When any related event is logged, traced, or recorded
  Then only token_id_hash and key_id appear — never the raw token or raw key material
  And this holds across every surface: logs, traces, metrics labels, crash dumps, debug output, model context, and persisted tool arguments

Scenario: S2-007 — Revocation-status unavailability fails closed
  Given a privileged action requires a current revocation-status check
  When the authoritative credential-state registry cannot be reached
  Then the use fails closed — the action does not proceed
  And this is logged distinctly from an ordinary revocation (status unknown, not status revoked)

Scenario: S2-008 — Circuit Breaker quarantine invalidates exactly the tokens in its stated scope
  Given active tokens exist across multiple runs, identities, and tools
  When S7 emits a quarantine event at run/agent/tool/system scope
  Then S2 invalidates exactly the tokens within that scope, per §6's mapping
  And tokens outside the scope retain their prior state unchanged
  # related_avs: AVS-CB-001, AVS-CB-002

Scenario: S2-009 — Revocation-path outage blocks locally rather than proceeding
  Given a Circuit Breaker trip requires token invalidation
  And the authoritative credential-state registry is unavailable to record it
  When a subsequent privileged action is attempted
  Then the Harness blocks the action locally without waiting for remote confirmation
  And records that revocation confirmation was unavailable, as its own distinct event
  And never proceeds merely because the revocation path itself failed

Scenario: S2-010 — Escalate invalidates current tokens without terminating the run
  Given a run with active tokens transitions to Escalate
  When those tokens are invalidated per this transition
  Then the run remains resumable, consistent with S7 §4's definition of Escalate as non-terminal
  And if the run later resumes, it does so with fresh Policy Server decisions and newly issued tokens
  And none of the invalidated tokens become valid again

Scenario: S2-011 — Key rotation preserves identity continuity; compromise disables immediately
  Given an Active identity with signing key K1
  When a routine rotation occurs, K2 becomes Active and K1 becomes Retired — K1's historical attestations remain verifiable
  But when K1 is instead reported compromised, K1 is disabled immediately outside the routine rotation path
  Then new JIT issuance for the identity is blocked until a replacement key is issued through the same Tier 4/5 approval as original issuance
  And in both cases the agent identity itself is unchanged

Scenario: S2-012 — Contract retirement disables the identity and its tokens, preserves history
  Given an Active identity with active JIT tokens
  When its governing Agent Contract is retired
  Then the identity becomes Retired, all its active tokens are invalidated immediately
  And no new token or attestation signing operation is permitted
  And historical attestations produced while Active remain verifiable

Scenario: S2-013 — Tier 5 attestation is verified before output acceptance
  Given a finalized Tier 5 output with an attestation payload (§8)
  When it is submitted for acceptance
  Then its digest, agent identity, key_id, and policy_decision_id are verified against the signing key's recorded version
  And missing or invalid attestation causes rejection, fail-closed
  And the verification itself is logged without exposing any private key material

Scenario: S2-014 — Credential issuance failure does not authorize action
  Given the Policy Server produces a grant decision
  When JIT credential issuance subsequently fails (custody component or registry unavailable)
  Then no privileged action proceeds on the strength of the grant decision alone
  And the failure is logged with policy_decision_id
```

## 14. What this spec explicitly does NOT do

- Does not define ABAC decision logic — Card 06 §4's territory, consumed not redefined
- Does not define policy bundle activation — S1's territory; this spec depends on it existing
- Does not detect or decide Circuit Breaker trips — S7's territory; this spec only maps and acts on S7's decisions (§6)
- Does not define instruction-artifact signing or loading verification — Card 06 §15 generally, S3 specifically
- Does not implement multi-node identity federation — §12 states only which invariants must survive it, not how
- Does not invent a formal cryptographic construction for `token_id_hash`, a canonicalization algorithm for attestation payloads, or exact clock-skew tolerances — all implementation-defined, the same boundary S1 drew around its own signing mechanism
- Does not build a separate identity-level suspension mechanism, an independent incident-response workflow for key compromise, or unbounded reusable-token semantics — each of these duplicates a mechanism this spec (or S7) already provides, or adds complexity not demonstrated as necessary for Phase 1
- Does not weaken any of §18's listed JIT token properties — this spec is the concrete, atomic, fail-closed mechanism that gives them operational meaning

---

*Once resolved, this becomes `docs/specs/S2-agent-identity-credential-lifecycle.md` at v1.0.*