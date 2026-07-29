---
spec_id: S3
title: Artifact Verification Manifest
status: APPROVED
version: 1.0
approved_by: Mohit Pammu
approval_date: 2026-07-29
format: BDD/Gherkin hybrid Markdown+YAML (Card 07 §2)
cites: [Card 02 §6, Card 04 §6, Card 04 §8, Card 06 §7, Card 06 §15, Card 06 §16, tech-stack.md Decision #11, AVS-C7-002, AVS-C7-005]
depends_on: [S2]
gates: [core/security/ runtime artifact loading, within Tier 3 Security Foundation — Master Plan item 1.8 (spec) / 1.9 (implementation)]
source: stage-10-reconciliation.md D7 (Runtime artifact-signing verification incomplete — Git signing ≠ runtime bundle trust); Master Plan Deferred list entry "Runtime artifact-signature verification" flagged as a real gap, not a future enhancement
---

# S3 — Artifact Verification Manifest

**Status:** 1.0 (Approved)

## 1. Purpose

Card 06 §15 requires that `AGENTS.md`, all four contract types, Skill bodies, evaluator rubrics, and system prompt templates are signed instruction artifacts, and that the Harness verifies signatures before loading any of them at runtime — unsigned or invalid-signature artifacts are not loaded. §16 states the same for registry entries specifically: runtime (not just write-time) verification, append-only, fail-closed scoped to the single bad entry. tech-stack.md Decision #11 closes the provenance half of this with Git commit signing, but is explicit that commit signing is not runtime verification — it is a record checked by a human reading history, not a gate checked by the running system. The gap this spec closes: what actually happens at the moment the Harness loads one of these artifacts — what gets checked, against what, and what happens on failure.

## 2. Non-Goals

- Agent identity, signing-key custody, issuance, or rotation — S2 owns this; S3 consumes S2's identity/key model as given, it does not redefine it
- JIT token lifecycle — S2 §5
- Policy Bundle signing/activation — S1 owns policy artifacts specifically, despite policies being "signed instruction/control artifacts" in Card 06 §21's own language; S1 is the more specific spec for that one artifact class
- Registry write-path authorization (who may write a registry entry, Tier 4 approval discipline for the write itself) — Card 06 §7, `core/security/` policy implementation (Master Plan 1.7); S3 covers verification at *load* time, not the write/approval path
- Skill content review and promotion workflow — Card 04 §8; S3 covers signature/integrity verification of an already-`Active` Skill body at load time, not how it became `Active`
- Cryptographic algorithm selection or canonicalization format — implementation-defined, consistent with S2 §8's precedent for attestation payloads
- Multi-node/distributed verification — out of scope per S5's single-process profile

## 3. Definitions

**Instruction artifact:** any of the six artifact classes named in Card 06 §15 — `AGENTS.md`, Agent Contract, Tool Contract, Memory Contract, Skill Contract, Skill body (Card 04), evaluator rubric (Card 05), system prompt template. (Policy bundles are a signed artifact class too, but owned by S1, not enumerated here to avoid dual ownership.)

**Verification manifest:** the signed metadata record that accompanies one specific version of one specific artifact, produced at the same time the artifact is signed (§4). The manifest is what the Harness actually checks at load time — the artifact's raw content is never compared directly; the manifest states what the artifact's content digest, version, and signer should be, and the Harness verifies the artifact against the manifest, then the manifest against its own signature. One manifest exists per signed artifact version — not one global manifest for the whole artifact class, and not one shared manifest across versions (a new version produces a new manifest, per §4's lifecycle). This mirrors S1's `policy_bundle_hash`-plus-metadata pattern (S1 §3), applied to the six Card 06 §15 artifact classes instead of policy bundles.

**Registry entry:** a single row/record in the Tool Registry or Skill Registry (Card 02 §6, Card 04 §6), each independently signed and versioned per Card 06 §16, and carrying its own manifest under the same model as any other instruction artifact.

**Load-time verification:** the check the Harness performs at the moment an artifact is read into an active run — as opposed to write-time verification (checked once, when the artifact is committed/registered). Card 06 §16 requires both exist independently; this spec's primary subject is load-time, since write-time (Git commit signing, GitHub push protection) is already covered by tech-stack.md Decision #11 and AVS-C7-002/005.

**Trust anchor.** The manifest's signature is verified against the same agent/service signing-key identity model S2 owns (S2 §4) — S3 does not define a separate root-of-trust or key hierarchy. Whatever key is authorized under S2's identity model to sign is the trust anchor S3 checks against; if that key's status changes (e.g., revoked per S2's identity lifecycle), manifests it signed are affected per §4's revocation handling below. This spec introduces no new PKI or trust infrastructure — it consumes S2's model as-is (§2's Non-Goals).

**Verification failure — two categories**, mirroring S1 §4's precedent:
- **Integrity/authenticity failure**: missing manifest, invalid manifest signature, manifest-to-artifact digest mismatch, or a manifest signed by a since-revoked key.
- **Security-policy failure**: a detected sensitive literal (AVS-C7-005) — a distinct, independent gate. It can reject an artifact whose integrity/authenticity checks all pass, and passing it does not substitute for integrity/authenticity verification. The two categories are checked separately (§4's ordering) so a failure's log entry always states which kind of failure occurred, without inventing a broader taxonomy beyond this one working distinction.

## 4. Mechanism

**Manifest lifecycle:** `Signed` (manifest generated and signed alongside the artifact at commit time, tech-stack.md Decision #11) → `Registered` (recorded as the current version for that artifact by the same actor/process that performs the Card 06 §7 registry write, or by commit for non-registry instruction artifacts — this spec does not mandate a distinct registration service) → `Active` (the version the Harness currently accepts for loading — this state describes verification eligibility, not execution; an artifact can be `Active` in the manifest sense while not currently in use by any run) → `Superseded` (a newer version has been registered; the old manifest is retained, immutable, and rejected if presented for load — it does not remain independently loadable once superseded). There is no separate "Rejected" state — a manifest that fails verification at load never reaches `Active`; that load attempt fails per this section's fail-closed behavior, and the artifact/manifest pair is simply not registered as anything other than what it already was. Once `Signed`, a manifest is immutable — any content change to the underlying artifact produces a new manifest with a new digest, never a mutation of an existing signed manifest, mirroring S1 §3's bundle-immutability precedent exactly.

**Manifest-artifact binding.** The manifest and the artifact it describes are versioned together and can never be independently substituted — presenting a valid manifest alongside a different artifact version (or vice versa) fails the digest check (§4 step 3) exactly as if the artifact had been altered. This is the security property the metadata contract's `content_digest` field exists to enforce; it is stated here as an explicit invariant rather than left implicit in the mechanism description alone.

**Manifest metadata contract.** Every manifest states, at minimum: `artifact_id`, `artifact_class` (one of the Card 06 §15 classes, or "registry entry"), `artifact_version`, `content_digest`, `signer_identity` (the S2-owned identity/key that signed it), `signature_reference`, `approval_record_id`, and `created_at`. `approval_record_id` refers to the Card 06 §15/§21 governance approval for that artifact's change (Tier 4 minimum, Tier 5 if it governs a Tier 5 capability, per §15) — not a separate artifact-class-specific approval process; every instruction artifact class shares this one approval discipline, so the field's meaning does not vary by class. This is the semantic field set required, not an implementation serialization format (consistent with S2 §8's precedent for the attestation payload) — the encoding is implementation-defined.

**What gets verified, and in what order:**
1. Locate the artifact and its accompanying manifest.
2. Verify the manifest's own signature against the trust anchor (S2's identity model).
3. Verify the artifact's content digest matches the manifest's recorded `content_digest`.
4. Scan for sensitive literals (AVS-C7-005) — independent of steps 2-3's outcome; a pass at steps 2-3 does not skip this step.
5. Only if all of 2-4 pass is the artifact accepted and loaded.

This ordering matters so different implementations perform the same checks in the same sequence, rather than each choosing its own subset.

**Fail-closed behavior, scoped:**

- Manifest missing, manifest signature invalid, or digest mismatch: the Harness does not load the artifact. The run does not start (or, if already running and the artifact is needed mid-run — e.g., a Skill invoked partway through — that specific capability is unavailable and the run proceeds without it or halts per its own Tier discipline, not a full-ecosystem halt).
- Invalid registry entry: fail-closed **scoped to that single entry** per Card 06 §16 — the affected tool/Skill is unusable until re-verified; the orchestrator and all other registry entries continue operating normally. This is the same scoping principle S7 uses for Circuit Breaker trips and S2 uses for quarantine scope — a failure in one unit does not cascade to unrelated units by default.
- Sensitive-literal detection (AVS-C7-005): an artifact containing a detected secret pattern is not loaded, independent of whether its signature is otherwise valid — a validly-signed artifact can still fail verification on this separate ground.

**Revoked signer timing.** If a manifest's `signer_identity` (a key under S2's model) is subsequently revoked, the manifest is treated as invalid on every load attempt from the moment of revocation forward — an artifact signed yesterday by a key revoked today fails verification on tomorrow's load, consistent with S2's own "fails closed on unknown/compromised status" principle (S2 §4). This spec does not define revocation itself — that is S2's territory — only that S3's load-time check always re-evaluates current signer status rather than trusting a result cached from a prior load.

**No caching across loads.** Verification (manifest signature, digest, sensitive-literal scan) is performed at every load event, in full — never served from a cached prior result, even if the same artifact was successfully verified on a previous run. This closes the loop already implied by §7's invariant and S3-005 below: it is stated here explicitly as a mechanism rule, not left to be inferred only from a scenario.

**Mid-run behavior.** Verification is a load-time gate, not a continuously-monitored runtime state. If an artifact is loaded successfully at the start of its use and later becomes invalid (e.g., its signer is revoked) before that same load is needed again, the currently-loaded instance continues in use for that invocation — the Harness does not interrupt an in-flight use. The next load event (a new run, or a new invocation of that artifact within the same run) re-verifies per the mechanism above and is rejected if the artifact is by then invalid. This spec does not define continuous mid-run monitoring — that would be new infrastructure beyond what Card 06 §15/§16 requires, which specifies load-time verification.

**What a verification failure produces:** every failure (missing manifest, invalid manifest signature, digest mismatch, revoked-signer manifest, sensitive-literal detection) is logged with the artifact identity, the artifact class, the specific check that failed (per §3's two-category distinction), and a timestamp — consistent with Card 06 §11's audit-trail model and AVS-C7-002/005's pass criteria requiring both failures to be logged with artifact identity and reason.

**What a successful verification produces:** a successful load also produces an audit record — not only failures are logged — stating `artifact_version`, `artifact_class`, `signer_identity`, and the verification timestamp, giving the same forensic traceability for the normal path that failures already have.

**Registry loading uses the same pipeline.** Registry entries do not have a separate verification path — loading a registry entry runs the identical five-step sequence above (locate, verify manifest signature, verify digest, sensitive-literal scan, accept) that any other instruction artifact does; §16's "registry integrity" requirement is satisfied by applying this spec's general mechanism to registry entries specifically, not by a parallel mechanism.

**Artifact-missing, deliberately deferred.** This spec does not define behavior for a manifest that is present and valid but whose referenced artifact cannot be located at all (as opposed to found-but-mismatched, which §4 step 3 already covers). This is intentionally left to the fuller artifact-integrity scenario set the Master Plan Deferred list already tracks as post-S3 work, not an oversight in this pass.

## 5. Scope

| Concern | In scope for S3 | Out of scope |
|---|---|---|
| Verification manifest as a first-class object — lifecycle, metadata contract, immutability | Yes | — |
| Load-time signature verification of instruction artifacts, via the manifest | Yes | Commit-time/CI signature enforcement (tech-stack.md Decision #11, already accepted) |
| Content-digest check at load time | Yes | Full artifact-integrity scenario set (wrong environment, expired approval beyond §21's existing Tier 4/5 discipline, missing artifact, partial bundle failure, rollback to older artifact version) — tracked in Master Plan Deferred list, post-S3 |
| Registry entry load-time verification, single-entry fail-closed scoping | Yes | Registry write authority / Tier 4 approval discipline for the write itself (Card 06 §7, `core/security/` policy implementation) |
| Sensitive-literal detection at load time, as an independent gate from integrity/authenticity | Yes (verification gate only) | Push-protection/CI-side secret scanning mechanics (already GitHub-native, tech-stack.md) |
| Trust anchor for manifest signature verification | Consumes S2's identity/key model as given | Defining a new root-of-trust, PKI hierarchy, or key-custody mechanism — S2's territory, not reopened here |
| Revoked-signer timing at load | Yes (re-evaluated every load) | Revocation mechanism itself, or notification/propagation timing — S2's territory |
| What key/identity signs an artifact | Consumes S2's model as given | Key custody, rotation, issuance (S2) |
| Policy bundle artifact verification specifically | No | S1 owns policy bundles as a distinct artifact class |
| Failure-category taxonomy beyond integrity/authenticity vs. security-policy | No | No card grounds a broader taxonomy (e.g., Confidentiality/Availability classes); declined per this project's established precedent (S7 similarly declined inventing a severity taxonomy without Card 06 grounding) — a candidate for a future Deferred-list item if a concrete downstream consumer emerges |
| Artifact dependency graphs (a Skill referencing a Prompt referencing an Evaluator, etc.) and cascading verification failure across that graph | No | No card describes artifact dependency graphs; this is new architecture, not a gap in existing text — candidate for a future Deferred-list item, not first-pass S3 |
| Full registry lifecycle states (Registered/Deprecated/Removed) beyond load-time verification | No | Already owned by Card 02 §6's 9-stage Tool lifecycle and Card 04 §8's Skill lifecycle — not redefined here to avoid dual ownership |
| Continuous mid-run artifact monitoring (detecting invalidation while an artifact is already in active use) | No | Card 06 §15/§16 specify load-time verification, not continuous monitoring; would be new infrastructure beyond the frozen requirement |

## 6. Ownership Boundaries

- **S2** owns agent/signing-key identity and lifecycle. **S3** owns what the Harness does with a signature at artifact-load time — it does not redefine who holds keys or how they rotate.
- **S1** owns policy bundle signing/activation specifically, since policies are one governed artifact class with their own lifecycle (Draft/Compiled/Signed/Approved/Active/Superseded) already fully specified. S3's mechanism (§4) is written to be the general pattern other artifact classes follow; S1 is not required to reference S3 retroactively, since S1 was approved first and already satisfies the same intent for its own artifact class.
- **Card 06 §7 / `core/security/` policy implementation (Master Plan 1.7)** owns registry *write* authorization — who may write, Tier 4/5 approval discipline for the write. **S3** owns registry *load-time* verification — what happens when an already-written entry is read for use.
- **Card 04 §8** owns the Skill content-review and promotion lifecycle (Proposed → Drafted → Reviewed → Active → Deprecated → Retired). **S3** owns integrity verification of a Skill body already at `Active`, at the moment it's loaded — S3 does not gate promotion between lifecycle states.

## 7. Architectural Invariants

- No instruction artifact is loaded into an active run without a valid, verified manifest signature and matching content digest.
- A verification manifest, once `Signed`, is immutable — a content change always produces a new manifest, never a mutation of an existing one.
- Verification happens at every load, not only once at commit or registration, and is never served from a cached prior result (per §4's five-step sequence).
- A registry-entry verification failure is scoped to that entry; it never halts unrelated entries or the orchestrator itself.
- Sensitive-literal detection is an independent gate from signature/digest validity — passing one does not exempt an artifact from the other.
- Manifest signature verification is checked against S2's identity/key model as the sole trust anchor — S3 introduces no separate root-of-trust.
- A manifest signed by a subsequently-revoked key is invalid on every load from the point of revocation forward — revocation status is always re-evaluated, never cached.
- Verification failures are always logged with artifact identity, artifact class, which of the two failure categories (§3) applied, and a timestamp — never a silent skip.

## 8. Scenarios (Given/When/Then)

```gherkin
Scenario: S3-001 — A valid artifact and manifest load successfully
  Given an instruction artifact with a Signed, Active manifest whose signature verifies against S2's identity/key model
  And the artifact's current content digest matches the manifest's recorded content_digest
  And no sensitive literals are detected in the artifact's content
  When the Harness loads the artifact
  Then the artifact becomes available for use
  And a success audit record is logged with artifact_version, artifact_class, signer_identity, and the verification timestamp

Scenario: S3-002 — Unsigned instruction artifact (no manifest) fails load
  Given an instruction artifact with no accompanying signed manifest
  When the Harness attempts to load it for a run
  Then the load fails closed as an integrity/authenticity failure
  And the failure is logged with artifact identity, artifact class, and reason "manifest missing"
  verifies: AVS-C7-002

Scenario: S3-003 — Manifest present but its own signature is invalid
  Given an instruction artifact with an accompanying manifest whose signature does not verify against S2's identity/key model
  When the Harness attempts to load it
  Then the load fails closed as an integrity/authenticity failure
  And the failure is logged with artifact identity, artifact class, and reason "invalid manifest signature"
  verifies: AVS-C7-002

Scenario: S3-004 — Validly-signed manifest but artifact content digest mismatch
  Given an instruction artifact whose manifest signature is valid but whose current content digest does not match the manifest's recorded content_digest
  When the Harness attempts to load it
  Then the content-digest check fails and the load is rejected as an integrity/authenticity failure
  And the failure is logged with artifact identity, artifact class, and reason "digest mismatch"
  related_avs: AVS-C7-002

Scenario: S3-005 — Sensitive literal detected at load blocks an otherwise validly-verified artifact
  Given an instruction artifact whose manifest, signature, and digest all verify successfully, but whose content contains a detected secret pattern
  When the Harness attempts to load it
  Then the load is rejected as a security-policy failure, independent of the integrity/authenticity checks having passed
  And the failure is logged with artifact identity, artifact class, and reason "sensitive literal detected"
  verifies: AVS-C7-005

Scenario: S3-006 — Invalid registry entry fails closed scoped to that entry only
  Given a Tool Registry with one entry whose manifest verification fails and all other entries verifying successfully
  When the orchestrator loads the registry at run start
  Then the invalid entry's tool is unavailable
  And all other registry entries load and remain usable
  And the orchestrator itself starts normally
  related_avs: AVS-CF-003 (registry rollback/entry-level topology this scenario partially establishes; full entry-level scenario set is Deferred per Master Plan, "Registry entry-level behavior")

Scenario: S3-007 — Registry entry re-verified on every load, not cached indefinitely
  Given a registry entry whose manifest verified successfully on a prior run
  And that entry's signer key has since been revoked per S2's identity lifecycle
  When the registry is loaded for a new run
  Then verification is re-performed against current signer status, not a cached prior result
  And the entry fails closed on this load as an integrity/authenticity failure, reason "revoked signer"
  related_avs: AVS-C7-002

Scenario: S3-008 — A manifest signed by a key valid at signing time but revoked before the next load
  Given an artifact signed and manifested yesterday, when its signer key was valid
  And that signer key is revoked today per S2's identity lifecycle
  When the artifact is loaded tomorrow
  Then the load fails closed as an integrity/authenticity failure, reason "revoked signer"
  And this is true even though the manifest's signature was valid at the time it was originally produced

Scenario: S3-009 — A newer manifest version supersedes an older one; the older manifest is rejected if presented for load
  Given an artifact with a currently Active manifest at version N
  And a new version N+1 has since been Signed and Registered, making version N Superseded
  When a load attempt presents version N's manifest
  Then the load is rejected — a Superseded manifest is never independently loadable once superseded
  And the failure is logged with artifact identity, artifact class, and reason "superseded manifest version"
```

## 9. What this spec explicitly does NOT do

- Does not define agent identity, key custody, rotation, or issuance, or any root-of-trust/PKI hierarchy — S2 owns identity and keys entirely; S3 consumes that model as its trust anchor
- Does not define registry write-path authorization or Tier 4/5 approval discipline for registry writes — Card 06 §7, `core/security/` policy implementation
- Does not define policy bundle verification — S1
- Does not define Skill content review/promotion, or any registry lifecycle states beyond load-time verification — Card 04 §8, Card 02 §6
- Does not specify a canonicalization or cryptographic algorithm, or the manifest's storage/encoding format — implementation-defined, per the metadata contract's semantic-field approach (§3)
- Does not define continuous mid-run artifact monitoring — verification is a load-time gate per Card 06 §15/§16, not a continuously-running check
- Does not invent a failure-category taxonomy beyond the two named in §3 (integrity/authenticity vs. security-policy) — no card grounds a broader one; a candidate for a future Deferred-list item, not first-pass content
- Does not define artifact dependency graphs or cross-artifact cascading failure (e.g., a Skill referencing an invalid Prompt) — no card describes such graphs; a candidate for a future Deferred-list item, not first-pass content
- Does not cover the fuller artifact-integrity scenario set beyond what §8 now includes (wrong environment, expired approval beyond existing Tier 4/5 discipline, missing artifact, partial bundle failure, unsigned supporting files, rollback to an older artifact version) — explicitly tracked in the Master Plan Deferred list as post-S3 verification-spec expansion
