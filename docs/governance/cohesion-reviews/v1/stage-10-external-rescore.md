# Independent Architecture Review — AI Agent Ecosystem

**Date:** 2026-07-10
**Reviewer:** External (cold-read, no prior involvement with this project)

**Reviewer Disclosure:** External reviewer — cold read, no prior involvement with this project.

**Architecture Version:** Cards 01–07, 2026-06 Freeze Candidate:

- Card 01 v4.0
- Card 02 v3.0
- Card 03 v5.0
- Card 04 v3.0
- Card 05 v2.0
- Card 06 v4.0
- Card 07 v2.0

Governance artifacts reviewed: AGENTS.md, Agent Contract Template v2.0, Model Routing Table v1.0, Tech Stack v1.0, Architecture Traceability Matrix, Architecture Verification Specification v1.0, and ADR-001 through ADR-007.

## Review Assumptions

1. LangGraph remains the orchestration framework.
2. The zero-cost model stack remains the initial deployment assumption.
3. Only public or synthetic data is used for the three portfolio projects.
4. The Master Execution Plan remains the authoritative build sequence.
5. Cards 01–07 are evaluated as one unified specification, not as independent documents.
6. “Security at the base” is intended primarily as an **authority and enforcement ordering**, even though some runtime data dependencies flow in the opposite direction.
7. Phase 0 artifacts marked Draft are intended to be approved or status-corrected before the freeze tag is created.

---

# Dimension 1 — Ownership Boundaries

**1. Score:** **8/10**

**2. Confidence:** **High —** ownership is explicitly documented in the cards, ADRs, AGENTS.md, and the traceability matrix.

**3. Evidence — Evidence Quality: Strong:** Card 06 §9a explicitly states that Security owns authorization and enforcement while domain cards retain semantic ownership. The Architecture Traceability Matrix maps individual concepts to canonical owners, including State, Memory types, Tool Registry, Policy Server, Evaluation, source-boundary preservation, and provider-routing metadata. ADR-003 similarly separates the shared Contract governance pattern from the four distinct schemas. fileciteturn2file5 fileciteturn3file2

A minor ownership ambiguity remains around the phrase “Policy Server.” Card 06 and ADR-002 describe a single centralized authority, while `tech-stack.md` selects OPA evaluated in-process as WASM. This can implement centralized policy semantics, but it is not literally a single runtime server unless policy distribution and version synchronization are separately defined. fileciteturn2file16 fileciteturn2file18

**4. Risk if Ignored:** Different processes could evaluate different policy bundles while all believing they are using the authoritative Policy Server. Audit records could then report internally valid but mutually inconsistent authorization decisions.

**5. Recommended Action:** Add a short implementation-boundary note to Card 06 §4 or `tech-stack.md` Decision 7 defining whether “Policy Server” means:

- one logical policy authority with distributed embedded evaluators, or
- one independently deployed service.

For the chosen embedded model, define the authoritative policy-bundle source, version pin, startup verification, refresh behavior, and stale-policy failure mode.

---

# Dimension 2 — Dependency Flow Integrity

**1. Score:** **6/10**

**2. Confidence:** **High —** the inconsistency is visible in the architecture’s own declared stack and runtime handoffs.

**3. Evidence — Evidence Quality: Strong:** ADR-001 calls the Runtime Stack a “strict dependency hierarchy,” places Security at the base, and says every layer above it is governed by Security. fileciteturn3file1

However, the documents also establish legitimate runtime dependencies in the reverse direction:

- Security/SecOps consumes Card 05 observability.
- Safety evaluation sends signals to the Policy Server.
- The Circuit Breaker receives signals from Evaluation and the Harness.
- The Circuit Breaker calls the Policy Server to revoke credentials.
- The Harness owns the final state transition requested by Security.

ADR-007 expressly describes the multi-hop `Evaluation/Harness → Circuit Breaker → Policy Server` path. fileciteturn2file17

The resulting architecture is coherent as an **authority hierarchy**, but not as a strict unidirectional software-dependency hierarchy. AGENTS.md reinforces the authority interpretation but still labels it a layered runtime stack. fileciteturn2file9

**4. Risk if Ignored:** An implementer may enforce an incorrect module-import or service-dependency direction to comply with the diagram. This could cause duplicated observability infrastructure, awkward callback mechanisms, or attempts to make Security independent of signals it explicitly needs.

**5. Recommended Action:** Patch ADR-001, AGENTS.md §4, and the Card 02/03/05/06 governing-structure introductions to distinguish two diagrams:

1. **Authority/semantic ownership order:** Security governs higher layers.
2. **Runtime interaction graph:** bidirectional event and request flows are permitted through named interfaces.

Replace “strict dependency hierarchy” with “authority and ownership hierarchy” unless strict software dependency direction is truly intended. State that lower-layer authority does not prohibit lower layers from consuming events emitted by higher layers.

---

# Dimension 3 — Circular Reference Check

**1. Score:** **8/10**

**2. Confidence:** **High —** all major cross-card loops have a definable bootstrap and implementation order.

**3. Evidence — Evidence Quality: Medium:** The strongest apparent loop is Evaluation → Security → Harness:

- Evaluation detects a safety condition.
- Security decides and signals enforcement.
- Harness performs the state transition.
- Harness emits observability consumed by Evaluation and Security.

Each component remains independently meaningful: Evaluation can score stored traces, Security can deny calls without Evaluation, and Harness can execute with static policies. ADR-005 clearly separates detection from enforcement, while ADR-007 separates decision-time authorization from runtime quarantine. fileciteturn2file2 fileciteturn2file17

Memory and Security similarly reference one another, but ownership remains resolvable: Card 03 defines memory semantics, while Card 06 defines who may perform governed actions.

**4. Risk if Ignored:** The event handoffs could be implemented synchronously in a way that deadlocks—for example, Harness waiting on Evaluation while Evaluation waits for the completed Harness trace.

**5. Recommended Action:** Add a small runtime interaction diagram identifying synchronous gates versus asynchronous consumers:

- Policy decision: synchronous.
- Tool binding check: synchronous.
- Evaluation scoring: asynchronous by default.
- SecOps analytics: asynchronous.
- Circuit-breaker trip signal: synchronous/preemptive when triggered.
- Final state transition: synchronous Harness action.

---

# Dimension 4 — Duplication / Redundant Definition

**1. Score:** **7/10**

**2. Confidence:** **High —** the major v1 duplication has been fixed, but two meaningful lifecycle restatements remain inconsistent.

**3. Evidence — Evidence Quality: Strong:** Card 01 §8 now points to Card 05 §5 instead of restating observability fields, and Card 07 §11 points to the Model Routing Table’s canonical schema. The traceability matrix records both corrections. fileciteturn2file5

The remaining material inconsistency is ADR-003’s statement that the lifecycle “Draft → Reviewed → Approved → Active → Deprecated” applies uniformly. fileciteturn3file2

That is not the lifecycle defined for:

- Tools: `Proposed → Designed → Security Reviewed → Built → Tested → Certified → Active → Deprecated → Retired`.
- Skills: `Proposed → Drafted → Reviewed → Active → Deprecated → Retired`. fileciteturn1file9
- Agent contracts: `Draft → Reviewed → Approved → Active → Deprecated`. fileciteturn1file16

ADR-006 also calls the Tool lifecycle “binary Active/Deprecated,” although Card 02 defines a much richer lifecycle. fileciteturn1file2

**4. Risk if Ignored:** Workflow automation or CI validation could implement the ADR lifecycle instead of the card-specific lifecycle, allowing an artifact to skip required stages such as Security Reviewed, Tested, Certified, or Retired.

**5. Recommended Action:** Amend ADR-003 to say the lifecycle **principles** are uniform—proposal, human review, activation, deprecation—while exact states are type-specific and owned by each artifact card. Remove “binary Active/Deprecated lifecycle” from ADR-006 and replace it with “call-time Active-state enforcement within a richer Tool lifecycle.”

---

# Dimension 5 — Missing Abstractions

**1. Score:** **7/10**

**2. Confidence:** **Medium —** the major architectural abstractions exist, but some implementation-governance abstractions are still deferred or implicit.

**3. Evidence — Evidence Quality: Medium:** All four Contract types have a defined home. The source-boundary mechanism missing in v1 now exists in Card 03 §12 and is tracked as the canonical mechanism in the traceability matrix. fileciteturn2file10

Card 07 defines the Specification lifecycle, and Card 04 explicitly defines Skill composition and the rule that Skills do not directly call other Skills. Card 04 nevertheless explicitly defers dependency declarations, freshness tracking, and deeper composition precedence. These are identified rather than silently missing.

The principal remaining abstraction gap is a formal **runtime policy-bundle/configuration distribution contract** for embedded OPA evaluators. Closely related details—agent identity issuance, key custody, token issuer ownership, and secret rotation—are specified as requirements but not yet collected into one operational identity/credential component contract.

**4. Risk if Ignored:** Phase 1 may produce authorization, identity, and signing implementations that satisfy individual paragraphs but do not form one coherent lifecycle. Policy version skew and unclear key ownership would be particularly difficult to diagnose.

**5. Recommended Action:** Add two Phase 1 specifications rather than another reference card:

- **Policy Bundle Distribution and Activation Spec**
- **Agent Identity, Signing Key, and JIT Credential Lifecycle Spec**

Both should cite Card 06 as semantic owner and define concrete components, storage, rotation, bootstrap, recovery, and audit behavior.

---

# Dimension 6 — Production Readiness

**1. Score:** **8/10**

**2. Confidence:** **High —** failure and compromise behavior is unusually explicit for a pre-implementation architecture.

**3. Evidence — Evidence Quality: Strong:** The architecture covers:

- Policy Server unavailability by risk tier.
- Policy integrity and change control.
- Break-glass restrictions.
- Circuit-breaker scope, thresholds, quarantine, and release.
- Runtime registry signature verification.
- Whole-registry unavailability.
- Durable Harness recovery.
- Evaluation-service degradation.
- Memory-backend degradation.

ADR-002 explicitly accepts the Policy Server’s high blast radius and ties it to Card 06 §§20–21. fileciteturn2file16 The verification specification contains executable failure scenarios for Policy Server outages and tampering, Circuit Breaker scope and release, Registry outages, Memory failure, Evaluation failure, and Harness crash recovery. fileciteturn3file0

The primary limitation is that signing is implemented using Git commit signing. The verification specification refers both to a valid Git signature and a registered hash, but the runtime mapping from repository commit trust to individual loaded artifacts is not yet fully specified.

**4. Risk if Ignored:** A runtime may load a validly committed artifact copied from the wrong branch, repository state, or approval context. A Git signature alone proves who signed a commit, not necessarily that the exact runtime bundle was approved for a particular environment.

**5. Recommended Action:** Before Phase 1 runtime loading, define an artifact manifest containing artifact path, content digest, artifact version, policy/classification version, environment, approval record, and signer. Runtime verification should validate the manifest and content digest, not only repository commit signature.

---

# Dimension 7 — Cross-Card Terminology Consistency

**1. Score:** **7/10**

**2. Confidence:** **High —** the important v1 terminology collision was fixed, but several smaller inconsistencies remain.

**3. Evidence — Evidence Quality: Strong:** “Structured Records” is now distinguished from session `state`, and the traceability matrix records the rename and owner. fileciteturn2file10 HITL, Risk Tier, observability, and evaluation are mostly used consistently.

Remaining defects include:

- Card 02’s heading says “Six Tool Design Principles” but enumerates seven principles.
- AGENTS.md says “Layer 4 (Skills)” even though its numbered Layer 4 is Memory and ADR-001 places Skills with Tools at position 5. fileciteturn1file7
- ADR-003’s lifecycle terminology conflicts with card-specific lifecycles.
- “Policy Server” suggests a service, while `tech-stack.md` specifies embedded OPA evaluation.
- Some text calls Skills “executable procedural knowledge,” although the same card correctly states that Skills are instructional content rather than callable/executable Tools. fileciteturn2file13

**4. Risk if Ignored:** These terms can leak into schemas and class names, turning editorial ambiguity into incompatible lifecycle enums, deployment assumptions, or misleading security controls.

**5. Recommended Action:** Apply a terminology-only patch:

- Rename Card 02 §3 to “Seven Tool Design Principles.”
- Change AGENTS.md to “Skills occupy the same capability tier as Tools.”
- Replace “executable procedural knowledge” with “runtime-loadable procedural knowledge.”
- Use “logical Policy Authority” for the architectural abstraction and “OPA embedded evaluator” for the implementation component.
- Make lifecycle names explicitly type-specific.

---

# Dimension 8 — Scope Discipline

**1. Score:** **8/10**

**2. Confidence:** **High —** most sections explicitly state what they do not own.

**3. Evidence — Evidence Quality: Strong:** AGENTS.md consistently points to canonical owners rather than restating tables and schemas. It states that Registries record while the Policy Server grants, Memory defines semantics while Security defines authorization, and Evaluation consumes observability without owning it. fileciteturn2file0

Card 06 §9a provides a useful general rule preventing Security from becoming a god layer. The traceability matrix also distinguishes domain semantics from enforcement ownership.

There is minor scope pressure in Card 06 because it contains security requirements for memory, evaluation, registry behavior, model routing, artifact integrity, A2A, audit, and runtime state signaling. The explicit enforce-not-own rule keeps this from becoming a current ownership violation, but the card is operationally dense.

**4. Risk if Ignored:** Future additions may be placed in Card 06 merely because they have a security implication, gradually making the domain cards incomplete and Security the effective owner of the entire architecture.

**5. Recommended Action:** Keep Card 06 as the policy owner, but require each new security requirement to identify:

- the domain concept owner,
- the security decision/enforcement owner,
- the interface between them,
- and which card owns failure semantics.

Add these four fields to the traceability matrix maintenance rule.

---

# Dimension 9 — Long-Term Maintainability

**1. Score:** **7/10**

**2. Confidence:** **High —** traceability is strong, but artifact-state drift is already visible.

**3. Evidence — Evidence Quality: Strong:** The Architecture Traceability Matrix substantially improves change-impact discovery and records canonical ownership, referencing locations, and last verification. It also records the blast-radius additions and foundational artifacts. fileciteturn2file6

However, current artifacts disagree on their own governance state:

- The matrix says AGENTS.md was reviewed and approved, while AGENTS.md says “Draft — awaiting review and approval.” fileciteturn1file4 fileciteturn1file12
- The matrix says the Architecture Verification Specification was reviewed and approved, while that document says Draft with approval pending. fileciteturn1file4 fileciteturn3file0
- The verification matrix says “18 scenarios,” despite the document containing 24 functional scenarios plus four performance scenarios.
- The user’s review charter describes AGENTS.md and other artifacts as frozen, while the embedded status text does not consistently reflect that.

**4. Risk if Ignored:** Future maintainers will not know whether artifact content, matrix metadata, or the freeze manifest is authoritative. Automated governance gates may either reject approved artifacts or accept drafts.

**5. Recommended Action:** Create a machine-readable `architecture-manifest.yaml` with:

- artifact path,
- version,
- lifecycle status,
- approval date,
- approving identity,
- content digest,
- supersedes/superseded-by,
- required review version.

Generate status tables in the matrix and documentation from that manifest rather than manually maintaining them.

---

# Dimension 10 — Implementation Feasibility

**1. Score:** **7/10**

**2. Confidence:** **Medium —** a competent team can start Phase 1, but a few architectural implementation boundaries require resolution.

**3. Evidence — Evidence Quality: Strong:** The Agent Contract Template includes dependencies, state ownership, execution limits, recovery mapping, escalation, evaluation, blast radius, assumptions, and artifact references. fileciteturn2file7 The Tech Stack chooses concrete technologies for orchestration, storage, durable checkpointing, observability, sandboxing, policy evaluation, deployment, testing, and signing. fileciteturn2file18

The main unresolved questions are:

- logical centralized Policy Server versus embedded policy evaluators;
- policy/configuration distribution and stale-version behavior;
- runtime artifact-signing verification;
- identity and key bootstrap;
- the exact reducer/locking rule for concurrent LangGraph state mutations;
- memory-write queue durability when the memory backend is unavailable;
- whether Tier 1–2 calls require live policy checks or can be locally authorized, because the verification scenario wording is broader than Card 06 §20’s project-specific table.

**4. Risk if Ignored:** Separate engineers could produce locally reasonable but incompatible implementations of the security boundary, especially around policy evaluation, state concurrency, and failure recovery.

**5. Recommended Action:** Permit Phase 1 to begin only after producing four implementation specifications:

1. Policy authority and bundle distribution.
2. Artifact verification and signing.
3. Agent identity/JIT token issuance.
4. LangGraph state ownership, reducers, and concurrency.

These are implementation specs, not new architecture cards.

---

# Dimension 11 — Architectural Simplicity

**1. Score:** **7/10**

**2. Confidence:** **Medium —** the abstractions are defensible, but their practical cost has not yet been measured.

**3. Evidence — Evidence Quality: Medium:** The two most questionable splits now have explicit ADRs:

- Tool Registry versus Skill Registry.
- Policy Server versus Circuit Breaker.

ADR-006 identifies different schemas, lifecycles, and binding times for Tools and Skills. fileciteturn1file2 ADR-007 distinguishes per-request authorization from accumulated-signal quarantine. fileciteturn2file17

Applying the rubric’s anti-bloat test:

| Component | Capability lost if removed or merged |
|---|---|
| Policy authority | Consistent authorization, JIT scope, audit provenance |
| Circuit Breaker | Preemptive quarantine based on runtime signal rather than individual request policy |
| Tool Registry | Governed callable-capability discovery |
| Skill Registry | Governed instruction-artifact discovery and project scoping |
| Memory Router | Type-specific storage and retrieval semantics |
| Skill Loader | Progressive disclosure and controlled runtime loading |
| Evaluation hierarchy | Cost-tiered probabilistic quality assessment |
| Observability | Traceability independent of judgments |

Each component therefore provides a distinct capability.

The complexity concern is cumulative: two registries, multiple contract types, OPA/Rego, signing, Jaeger, Postgres/pgvector, per-execution Docker, evaluation tiers, a Circuit Breaker, and numerous governance workflows are a heavy Phase 1 burden for one developer and public/synthetic portfolio data.

**4. Risk if Ignored:** The developer may spend most of the build implementing governance infrastructure before validating whether the three portfolio workflows benefit from the agent abstractions.

**5. Recommended Action:** Preserve the architecture but define a **Phase 1 minimum conformance profile**. It should implement all security invariants while allowing:

- one Tool Registry file,
- one Skill Registry file,
- one process with embedded OPA,
- one durable Postgres instance,
- deterministic evaluators first,
- run-level Circuit Breaker before system-level aggregation,
- no A2A until Phase 3.

This reduces initial machinery without removing architectural boundaries.

---

# Dimension 12 — Evolutionary Resilience

**1. Score:** **8/10**

**2. Confidence:** **Medium —** extension paths are well described, but no implementation has yet tested them.

**3. Evidence — Evidence Quality: Medium:** A fourth project fits the existing project-scoped contracts, domain Skills, model-routing classification, and policy overlays. New providers fit the Model Routing Table’s evidence schema. New Tools and Skills fit separate registries. The contract pattern permits additional governed artifact types if justified. ADR-001 claims future layers can be added without renegotiating existing ownership. fileciteturn3file1

Scenario walkthrough:

- **Fourth project:** clean extension through project contracts, policy, and domain Skills.
- **New model provider:** clean row addition to the routing table.
- **New memory type:** requires changes to Card 03, Memory Contracts, router, metadata, retrieval, and likely the precedence model; manageable but cross-cutting.
- **New risk tier:** intentionally cross-cutting and would affect Card 06, contracts, policies, tests, and approval workflows.
- **Skill calling another Skill:** currently prohibited as a control-flow mechanism; supporting it would require a deliberate redesign of Skill composition.
- **Framework replacement:** core concepts survive, but LangGraph-specific state, reducer, checkpointer, and session assumptions require implementation replacement and some card-note changes.

**4. Risk if Ignored:** The framework-agnostic architecture may gradually acquire LangGraph-specific assumptions in supposedly general sections, making a future replacement much larger than expected.

**5. Recommended Action:** Add a framework portability appendix listing invariants versus adapters:

- invariant: contracts, Decide outcomes, state ownership, source types, registries, policy semantics;
- adapter-specific: graph nodes/edges, reducers, checkpointer API, message representation, tool-result block mapping.

---

# Dimension 13 — Blast Radius Analysis

**1. Score:** **8/10**

**2. Confidence:** **High for specification coverage; Medium for real behavior —** the statements are explicit but untested.

**3. Evidence — Evidence Quality: Strong:** Card-level blast-radius behavior now exists for Harness crashes, Tool Registry outage, Skill Registry outage, Memory outage, and Evaluation outage. The traceability matrix records these distributed owners. fileciteturn2file6 The Agent Contract Template requires each agent to identify what capability becomes unavailable while it is down. fileciteturn1file16 The verification specification contains concrete pass criteria for all five failures. fileciteturn2file15

The principal remaining weakness is shared-infrastructure correlation:

- PostgreSQL hosts Structured Records, semantic memory, and LangGraph checkpoints.
- One outage therefore removes long-term memory, authoritative records, and crash recovery simultaneously.
- Docker Compose runs the current deployment on one machine.
- Registry files, signing metadata, and policy bundles may share the same host and filesystem.

Individual component blast radii are specified more clearly than correlated host-level failure.

**4. Risk if Ignored:** A single disk, host, Docker daemon, or PostgreSQL failure could invalidate several supposedly independent degradation paths at once. For example, the system may be unable both to recover the Harness and to read the memory needed for degraded operation.

**5. Recommended Action:** Add compound-failure scenarios for:

- PostgreSQL unavailable during Harness crash recovery.
- Host filesystem unavailable or corrupted, affecting registries, policy bundle, and signed artifacts.
- Docker daemon unavailable.
- Observability backend unavailable during a security incident.
- Policy evaluator available but policy bundle stale.
- Circuit Breaker trip while Policy Server/token revocation is unavailable.

---

# Dimension 14 — Operational Validation Readiness

**1. Score:** **6/10**

**2. Confidence:** **High —** the verification artifact is detailed enough to inspect, and its internal count and approval defects are directly observable.

**3. Evidence — Evidence Quality: Strong:** The verification specification provides Trigger, Expected behavior, Pass criterion, and Phase for each scenario. It covers Policy Server behavior, Circuit Breaker behavior, component degradation, execution-loop limits, A2A boundaries, state races, regulated approval, model routing, sandboxing, and observability completeness. It also includes four architectural performance expectations. fileciteturn3file0

There are significant document defects:

- The Coverage Matrix totals **18**, but its own card counts sum to **22** if the ADR row is excluded as a scenario count, or **23** if included.
- The actual numbered functional scenarios total **24**:
  - 3 Policy Server
  - 3 Circuit Breaker
  - 5 degradation
  - 4 execution-loop
  - 3 multi-agent
  - 4 security
  - 2 observability
- Four additional performance scenarios bring the actual total to **28**.
- Status remains Draft with approval pending even though the traceability matrix says approved.
- The specification claims “one pass/fail criterion per scenario,” but performance criteria defer their actual thresholds until Phase 1; they are measurable templates, not yet pass/fail criteria.
- Card 07 has zero direct verification scenarios, leaving spec-gate, artifact lifecycle, rollout/rollback, and context-hygiene process controls untested.

**4. Risk if Ignored:** Test coverage reports and freeze evidence will be unreliable. A suite may report “18/18 passed” while silently omitting ten documented scenarios. Performance tests cannot produce pass/fail results until thresholds are defined.

**5. Recommended Action:** Correct the catalog before treating it as the Phase 1 test plan:

1. Generate scenario counts from scenario IDs.
2. Separate **24 functional scenarios** from **4 performance-baseline scenarios**.
3. Add stable scenario IDs, such as `AVS-PS-001`.
4. Add Card 07 scenarios for:
   - implementation without approved spec is rejected;
   - unsigned instruction artifact fails CI and runtime load;
   - canary rollback restores prior version;
   - `[[VARIABLE]]` context-hygiene check rejects literal sensitive fixture values.
5. Convert performance baselines into two-stage criteria:
   - Phase 1 baseline establishment;
   - Phase 1+ regression threshold.
6. Resolve Draft/Approved status before freeze.

---

# Blocking Issues

**None under the rubric’s mandatory gate rule.**

No Freeze Criterion dimension scored ≤4. The architecture therefore does not automatically receive a Not-Freeze-Ready verdict.

The Dimension 2 issue is important but patchable: the architecture is internally usable once the Runtime Stack is explicitly described as an **authority hierarchy rather than a strict unidirectional runtime dependency graph**.

---

# Non-blocking Refinements

1. Correct the Runtime Stack terminology and document the runtime interaction graph.
2. Correct lifecycle wording in ADR-003 and ADR-006.
3. Resolve all Draft/Frozen/Approved status discrepancies.
4. Fix verification scenario counts and identifiers.
5. Clarify the logical Policy Authority versus embedded OPA evaluator model.
6. Define policy-bundle distribution, signing manifests, identity bootstrap, and key custody in Phase 1 specs.
7. Correct minor terminology defects: six/seven principles, “Layer 4 Skills,” and “executable Skills.”
8. Add compound-infrastructure failure scenarios.
9. Define a Phase 1 minimum conformance profile to control solo-developer implementation burden.

---

# Architectural Debt Register

| Debt | Priority | Owner | Review Version | Target Resolution |
|---|---|---|---|---|
| Runtime Stack conflates authority order with runtime dependency direction | High | ADR-001 / AGENTS.md / Cards 02–06 governing-structure text | Cohesion Review v1 re-score | Before Phase 0 freeze tag |
| Logical Policy Server versus embedded OPA evaluator semantics | High | Card 06 §4 and `tech-stack.md` Decision 7 | Cohesion Review v1 re-score | Phase 1, before authorization implementation |
| Policy-bundle distribution and stale-version handling undefined | High | Card 06 §§21–22 / Phase 1 Security spec | Cohesion Review v1 re-score | Phase 1 |
| Artifact status disagreement: Draft versus Approved/Frozen | High | AGENTS.md, Verification Specification, Traceability Matrix | Cohesion Review v1 re-score | Before Phase 0 freeze tag |
| Verification scenario counts and approval status incorrect | High | Architecture Verification Specification | Cohesion Review v1 re-score | Before verification-suite implementation |
| Contract lifecycle wording conflicts with type-specific lifecycles | Medium | ADR-003 and ADR-006 | Cohesion Review v1 re-score | Before Phase 1 CI lifecycle checks |
| Runtime artifact manifest/signing semantics incomplete | Medium | Card 06 §15/§21 and `tech-stack.md` Decision 11 | Cohesion Review v1 re-score | Phase 1 |
| Agent identity/key/token issuer operational design incomplete | Medium | Card 06 §§4, 18, 21 | Cohesion Review v1 re-score | Phase 1 |
| Compound PostgreSQL/host failure analysis incomplete | Medium | Tech Stack Decisions 3–4 and Verification Specification | Cohesion Review v1 re-score | Phase 1 verification |
| LangGraph state reducer/concurrency contract implicit | Medium | Card 01, Card 03, Agent Contract §7 | Cohesion Review v1 re-score | Phase 1 state specification |
| Skill dependency/freshness metadata deferred | Low | Card 04 | Cohesion Review v1 re-score | Phase 3 |
| Fleet policy and Skill rollout propagation deferred | Low | Card 07 §10 | Cohesion Review v1 re-score | Phase 5+ |

---

# Future Validation Plan

## Dimension 12 — Evolutionary Resilience

### A. Fourth-project extension

**Test:** Add a synthetic fourth project with one domain Skill, two Tools, one agent contract, and a project policy override.

**Pass criterion:** No existing Card schema changes are required. Only new project artifacts, registry entries, contracts, and policy records are added.

### B. New model provider

**Test:** Add a provider using the routing table evidence schema.

**Pass criterion:** Provider becomes routable for approved classifications without changing Harness routing code other than configuration/schema-compatible registration.

### C. Framework portability

**Test:** Implement one simple agent using a second orchestration adapter while preserving the same Agent Contract.

**Pass criterion:** Mission, tools, memory access, state ownership, Decide outcomes, and evaluation fields remain unchanged. Only orchestration/checkpoint/message adapters differ.

### D. New memory type

**Test:** Prototype a sixth memory type.

**Pass criterion:** Required changes are identified through the traceability matrix, and existing five-type behavior remains backward compatible. No unrelated Tool, Skill, or Evaluation schema must change.

## Dimension 13 — Blast Radius

### A. Compound Postgres failure

**Test:** Crash the Harness while PostgreSQL is unavailable.

**Pass criterion:** The system does not claim checkpoint recovery. It fails or escalates with an explicit combined-dependency reason and does not repeat non-idempotent side effects.

### B. Registry and policy-bundle corruption

**Test:** Corrupt one Registry entry, the entire Registry file, and the active policy manifest separately.

**Pass criterion:** One bad entry disables only that entry; whole-file loss disables the corresponding capability type; policy-manifest failure follows tier-specific fail-closed behavior.

### C. Circuit Breaker with Policy Authority unavailable

**Test:** Trip the Circuit Breaker while JIT-token revocation infrastructure is unavailable.

**Pass criterion:** Harness blocks all subsequent privileged actions locally, transitions to Stop or Escalate, and records that remote revocation confirmation was unavailable.

### D. Observability outage during incident

**Test:** Disable Jaeger or its storage while a Policy denial and Circuit Breaker trip occur.

**Pass criterion:** Security enforcement still happens. Events are durably buffered or written to a fallback audit path and later reconciled without loss.

## Dimension 14 — Operational Validation Readiness

### A. Scenario-catalog integrity

**Test:** CI parses every `### n.n` scenario heading and compares it with the Coverage Matrix.

**Pass criterion:** Functional and performance counts match automatically; duplicate or missing IDs fail CI.

### B. Trace-schema completeness

**Test:** Validate all completed turns and security events against machine-readable schemas.

**Pass criterion:** 100% of completed turns contain Card 05 fields; 100% of security events contain applicable Card 06 fields. Non-applicable nullable fields are explicitly defined rather than silently omitted.

### C. Performance baselines

**Test:** Establish Phase 1 baselines for policy evaluation, context assembly, registry lookup, and Circuit Breaker response.

**Pass criterion:** Each baseline produces a numeric threshold and regression budget. Subsequent CI fails on a statistically meaningful threshold breach.

### D. Card 07 process controls

**Test:** Attempt to merge implementation without a spec, load an unsigned artifact, deploy without canary evidence, and commit a fixture containing a prohibited literal sensitive value.

**Pass criterion:** All four attempts are blocked by automated controls, with an auditable reason.

---

# Overall Verdict

## **Freeze-Ready-with-Minor-Patches**

All eleven Freeze Criteria score above the rubric’s automatic-failure threshold. The architecture has clear semantic owners, resolved major forward references, robust failure-mode coverage, defensible abstraction boundaries, concrete foundational artifacts, and an unusually strong traceability model.

The freeze should nevertheless include a small corrective patch set before the final tag:

1. Recast the Runtime Stack as an **authority hierarchy**, with a separate runtime interaction/dependency graph.
2. Correct the lifecycle conflicts in ADR-003 and ADR-006.
3. Resolve Draft/Approved/Frozen status discrepancies.
4. Correct the Architecture Verification Specification’s scenario totals and add stable scenario IDs.
5. Clarify how the centralized logical Policy Authority is realized through embedded OPA evaluators.

These are specification-cohesion defects, not evidence that the core architecture requires redesign. Once patched, an independent engineering team could begin Phase 1 without needing the original author’s conversational context.
