# Reference Card 06 — Security & Governance

**Card Version:** 5.0
**Changelog:** §4 — added "Terminology clarification — logical vs. implementation" paragraph distinguishing the logical Policy Authority role from its current OPA WASM-embedded implementation. Patch 2, Stage 10 pre-freeze.
**Changelog (v4.0):** Added topic index after Purpose — maps all 26+ sections to topic clusters for navigation without a full read-through. Closure Plan Stage 8, Action 1.
**Changelog (v3.0):** Untrusted context boundary citation corrected from "Card 03 §2-3" to "Card 03 §12" — now points to the actual operationalized mechanism added in Card 03's Stage 5 edit, closing the last Stage 1 defect. Closure Plan Stage 5.
**Changelog (v2.0):** §11 — added "Alternative Architecture Considered" note documenting why the Circuit Breaker is kept separate from the Policy Server, per Closure Plan Stage 4.

**Source whitepaper:** Vibe Coding Agent Security and Evaluation (2026 Day 4, security half)
**Governing structure:** Sits at the base of the Runtime Stack — Security governs every layer above it (Tools, Memory, State, Harness, Contract), not the reverse. This card discharges the largest backlog of any card in the ecosystem so far. Every item below references exactly which earlier card raised it.
**Purpose:** Defines the 7-Pillar Security Architecture, risk tiers, the full Permission Model, sandboxing requirements, A2A trust boundaries, registry integrity, memory governance authorization, and the enforcement boundary for Safety evaluation — closing every forward-reference accumulated across Cards 01-05.

---

## Topic Index

**7 Pillars overview:** §2
**Risk Tiers (definition + table):** §3 · **Regulated-project override (Projects 1/3):** §14
**Permission Model (ABAC + JIT):** §4 · **JIT Token lifecycle:** §18
**Policy Server (mechanics):** §4 · **Failure mode (fail-closed table):** §20 · **Integrity + change control:** §21 · **Policy versioning:** §22
**Circuit Breaker (blast radius, quarantine):** §11
**Sandboxing:** §5
**Data Classification (taxonomy, assignment, inheritance):** §13 · §23
**Registries (integrity, runtime verification):** §7 · §16
**Memory governance authorization:** §8
**A2A trust + delegation:** §6 · §17
**Safety Evaluation enforcement boundary:** §9 · §9a
**Instruction-artifact signing:** §15
**Audit trail (minimums, provenance):** §12 · §26
**HITL reviewer authorization + separation of duties:** §24
**Break-glass (hardened constraints):** §25
**Egress governance:** §19
**Security Event Schema:** §11
**Red Team / Green Team definitions:** §10 · §11
**Supply chain scope:** §26
**Classification-aware model routing:** §26
**Untrusted context boundary:** §26 (see Card 03 §12 for the mechanism)

---

## 1. Core Security Principle

A raw model is not an agent — it becomes one when wrapped in a harness that gives it state, tool execution, and enforceable constraints (Card 01 §1). **Security secures the harness, not the model.** The central question Security answers, distinct from Evaluation (Card 05): *did the agent stay inside the boundary* — Evaluation then asks whether what happened inside that boundary was actually good. This card builds the boundary; Card 05 measures behavior within it.

## 2. The 7-Pillar Security Architecture

| Pillar | Owner concern | Core mechanism |
|---|---|---|
| **1 — Infrastructure** | Sandboxing untrusted execution | Ephemeral, network-isolated sandboxes (containers/VMs/gVisor); state resets between runs; raw host access blocked |
| **2 — Data** | Context/memory leakage, poisoned retrieval | Encryption at rest (CMEK) and in transit (mTLS); least-privilege data access; vector-store tenant partitioning against cross-tenant poisoning |
| **3 — Model** | Semantic attacks on instructions | System instructions and prompt templates treated as sensitive, cryptographically attested artifacts — the "new source code" |
| **4 — Application & Runtime** | Tool/agent autonomy | LLM firewalls, lifecycle hooks (before tool call / after file edit), centralized Agent Gateway governing A2A to prevent unauthorized lateral movement |
| **5 — Identity & Access (IAM)** | The Confused Deputy problem | Unique cryptographic agent identity, ABAC + JIT token downscoping, Intent × User × Time permission matrix |
| **6 — Observability & SecOps** | Invisible failures, drift | Red/Blue/Green security teaming on top of Card 05's observability infrastructure |
| **7 — Governance** | Regulatory/audit accountability | Immutable audit trail tying every action to an agent + the human who approved it; risk-stratified attestation |

This card is organized around discharging accumulated debt first (§§3-9), then covering Pillars 1, 6, and 7's remaining implementation detail (§§10-12) that isn't already satisfied by resolving that debt.

## 3. Risk Tiers — Discharging Card 02 §4/§8

Card 02 deferred risk tier *definition* here, requiring only that tools *reference* a tier. Final tiers:

| Tier | Name | Examples | Default approval requirement |
|---|---|---|---|
| **0** | Computation | Pure calculation, no I/O | None |
| **1** | Read-only data | Query a table, fetch a record | None |
| **2** | Sensitive data read | Read PII/PHI/financial fields | Logged, no approval gate |
| **3** | Reversible write | Draft creation, tagging, metadata update | None, but reversible-write audit logged |
| **4** | External side effect | Send email, write to external system | HITL required (Card 05 §4 tier 4) |
| **5** | Regulated/high-impact | Financial transaction, medical/compliance action, production deployment | HITL required + Risk-Stratified Attestation (§9) |

Every Tool Contract's `risk_tier` field (Card 02 §4) and every Skill Contract's implicit risk exposure (Card 04 §5) resolves against this table — defined here exactly once, ecosystem-wide.

**Note:** these are global defaults. **Project-specific policy may strengthen these defaults; Projects 1 and 3 do so explicitly in §14** — read this table in conjunction with §14 rather than in isolation, since Tier 2/3 in particular are tightened for regulated-data work.

## 4. The Permission Model — Discharging Card 02 §4/§8

Full mechanics for the fields Card 02 deferred:

- **Identity**: every agent gets a unique cryptographic identity (conceptually a SPIFFE-style ID) — not a shared service credential. This is what makes the audit trail (§12) meaningful and resolves the Confused Deputy problem (Pillar 5)
- **ABAC (Attribute-Based Access Control)**: access decisions evaluate attributes — which agent, which project, which risk tier, current time window — rather than a static role list
- **JIT (Just-In-Time) token downscoping**: credentials are issued fresh per task, scoped to only what that task needs, and expire immediately on completion. **This is the mechanism behind Card 02 §5's "assignment authority belongs to the Policy Server"** — the Policy Server is what issues these JIT tokens. Full lifecycle rules (issuance, binding, revocation, logging) are specified in §18.
- **The Intent × User × Time matrix**: the actual access decision is a function of all three — not just "is this agent allowed to use this tool" in isolation, but "is this agent, acting on this specific intent, on behalf of this user, within this time window, allowed to do this right now"

**Direct implication for Card 02's binding rule:** the 4-condition check (registered/assigned/Active/MCP-compatible) is necessary but not sufficient — the Policy Server's ABAC+JIT evaluation is the actual enforcement layer underneath that check, evaluated fresh per call, not cached.

**Terminology clarification — logical vs. implementation:** "Policy Server" throughout this card names a **logical Policy Authority** — the architectural role responsible for ABAC+JIT decisions, registry write authority, memory governance, and Safety enforcement. It does not imply a standalone deployed service. The current implementation (`tech-stack.md` Decision #7) realizes this role via OPA compiled to WASM and evaluated in-process — no separate server process runs. Wherever this card or others say "Policy Server," read it as "the logical Policy Authority, currently implemented as an embedded OPA evaluator" — the distinction matters because a future implementation change (e.g., moving to OPA's server mode at scale) would not require any change to this card, only to `tech-stack.md`.

## 5. Sandboxing — Discharging Card 02 §4/§8

Any agent-generated or skill-invoked code that executes dynamically (not just pre-built Tool calls) must run in an ephemeral, network-isolated sandbox:

- Hardened environments — dedicated containers, VMs, or kernel-level isolation
- Sandboxes block raw host access and **fully reset state between runs** — a compromised execution cannot persist or escape to the host
- **Supply chain defense**: dependencies sourced only from vetted/internal registries with cryptographic version pinning — this directly defends against "slopsquatting" (malicious packages published under names LLMs commonly hallucinate). CI/CD verifies SBOM entries and signatures via Binary Authorization before anything reaches production
- **Egress governance**: outbound network access from a sandbox is non-deterministic by nature (driven by dynamically chosen tools), so a simple domain allowlist is insufficient — agents fetch external data only through offline caches or pre-sanitized crawling services, never direct, interactive internet access

## 6. A2A Trust Boundaries — Discharging Card 02 §7

Card 02 described A2A mechanics but deferred trust enforcement here:

- **Identity**: each agent in an A2A exchange presents its own cryptographic identity (§4) — not a borrowed or ambient one
- **Authorization**: the centralized Agent Gateway (Pillar 4) governs A2A calls, preventing unauthorized lateral movement — one agent cannot silently reach into another's scope without going through this gateway
- **Audit trail**: every A2A handoff is logged with both agents' identities, the capability invoked, and the outcome (feeds the immutable audit trail in §12)
- **Data sharing policy**: what crosses an A2A boundary is governed by the same data minimization principle as tools (Card 02 §3) — a receiving agent gets only what its task requires, not the sender's full internal state

## 7. Registry Integrity — Discharging Card 02 §6/§8

Card 02's Tool Registry (and Card 04's parallel Skill Registry) rely on uniqueness and the binding rule for baseline anti-shadowing protection, but explicitly deferred *tamper protection* here:

- Registry entries are signed — a write to the Tool or Skill Registry requires the same cryptographic identity verification as any other Tier 4/5 action (§3)
- The Policy Server (§4) is the sole writer to both registries' `Allowed Agents`/assignment fields — no agent or process writes to its own registry entry directly
- Creating a brand-new registry entry, modifying non-assignment metadata (e.g., a description or version bump), and retiring an entry are each themselves Tier 4 actions requiring the same approval discipline as any other high-risk write; full runtime verification minimums (append-only, versioned, fail-closed on bad signature) are specified in §16

## 8. Memory Governance Authorization — Discharging Card 03 §11

Card 03 defined memory contract *fields* (retention/update/ownership policy) but deferred *who* can invoke deletion, correction, or verification-status changes:

- **Marking a memory `verified`**: requires either a Tier 2+ verified tool output as the source (automatic), or explicit human review (manual) — never a model's own self-assessment of its confidence
- **Marking a memory `contested`**: any consolidation-time contradiction (Card 03 §8) automatically sets this status; clearing a `contested` flag requires human review
- **Deletion/correction**: governed by the same ABAC+JIT model as tool access (§4) — an agent does not get standing authority to delete its own prior memories; deletion is itself a Tier 3+ action subject to the same permission check as any other write

## 9. Safety Evaluation Enforcement Boundary — Discharging Card 05's Pillar 4

Card 05 stated evaluation detects Safety/Alignment failures but does not enforce. This card is where enforcement actually lives: when Card 05's Safety pillar evaluation flags a violation, **the Policy Server (§4) is what acts on that flag**, mapping the signal to an action using three inputs together — **risk tier (§3) + severity + confidence** of the evaluation signal — not risk tier alone. A low-confidence flag on a Tier 1 action and a high-confidence flag on a Tier 5 action warrant very different responses; this mapping is a Policy Server configuration, not a fixed rule, but the three-input model is fixed. The resulting action is revoking a JIT token, triggering the stateful circuit breaker (§11), or escalating to HITL (Card 05 §4 tier 4) per the violation's risk tier. Evaluation produces the signal; this card's enforcement mechanisms are what actually respond to it.

## 9a. Layer Ownership — Security Enforces, Domains Own Semantics

A general rule, stated explicitly to prevent Security from becoming a "god layer" as this card's authority has grown across §§3-9: **Security owns authorization and enforcement decisions — who may act, under what scope, with what consequence on violation. It does not own the domain semantics of what an action means.** Concretely:

- Security answers *who may mark a memory verified* (§8); Memory (Card 03) still defines *what `verified` means* and *how retrieval treats contested memory*
- Security answers *who may write to a Tool/Skill Registry* (§7); Tools (Card 02) and Skills (Card 04) still define *what a registry entry's fields mean*
- Security answers *whether a Circuit Breaker trips*; the Harness/Orchestrator (Card 01) still owns *what execution state transition results*

This split is reinforced concretely in §11's Circuit Breaker boundary and is the lens for resolving any future ambiguity about whether something belongs in this card or in the domain card it touches.

## 10. Agent-as-Judge Trace Access — Discharging Card 05 §4 Tier 3

Card 05 flagged that trace data may contain sensitive content and required this card's redaction policy: Agent-as-Judge evaluators access traces through the same data-access controls as any other reader (Pillar 2) — least-privilege scoping and redaction of Tier 2+ sensitive fields apply *before* a trace reaches a Critic Agent's context, not after. An evaluator agent's elevated *purpose* (improving the system) does not grant it elevated *access* — it goes through the identical Policy Server check as a production agent would for the same data.

**Red Teaming clarification:** Red Team activity (§11) is a **security assurance activity**, conceptually and procedurally distinct from Card 05's quality-focused evaluation harness (Outside-In/Inside-Out, Agent-as-Judge). It is not part of the evaluation pipeline and should not be referred to as an "evaluation activity" even informally — this keeps Evaluation (Card 05) and Security Assurance (this card) from blurring into one discipline.

## 11. Observability/SecOps Teaming (Pillar 6) — Built On Top of Card 05

Card 05 built the observability infrastructure (logging/tracing/metrics) as a separate, evaluation-agnostic module. This pillar adds a security-specific consumer of that same infrastructure — a second, parallel consumer alongside Card 05's evaluation harness, maintaining the same one-directional dependency (SecOps reads observability data; observability has no SecOps dependency). Security events, however, need fields beyond generic observability — see the Security Event Schema below.

**Green Team — explicit definition** (the term is non-standard outside this whitepaper's usage): **Green Team refers to automated defensive response — quarantine, credential revocation, and safe shutdown when security thresholds are crossed.** It is not related to sustainability/efficiency work, a meaning the term carries in some organizations; this definition is binding within this ecosystem.

- **Blue Team**: behavioral analytics over existing trace data, looking for drift — a security-specific read of the same traces Card 05's evaluation reads, not the same analysis
- **Red Team**: proactively simulates multi-hop attacks against the running system — a security assurance activity (see clarification above)
- **Green Team**: executes the stateful circuit breaker described next

**Circuit Breaker — state ownership boundary (closing the review's highest-priority gap):** the Circuit Breaker does **not** directly own or rewrite execution state. Its scope is strictly: revoke credentials (invalidate JIT tokens, §18), block further privileged actions for the affected scope, and emit a quarantine event. **It then instructs the Harness/Orchestrator (Card 01) to transition the run into Escalate or Stop according to Card 01's existing Decide semantics** — the Harness remains the sole owner of execution state; Security only ever signals, never mutates that state directly. This is the precise resolution to the review's "who owns the final state transition" question.

**Circuit Breaker — trigger and scope semantics:**
- **Scope**: a circuit-breaker event can be run-level, agent-level, tool-level, or system-level — declared explicitly per trigger, not assumed global by default
- **Trigger thresholds**: a single high-severity event can trigger immediate Stop; repeated low-severity events accumulate toward Escalate. Thresholds and release criteria are Policy Server configuration, not hardcoded, and are themselves auditable
- **Quarantine release**: clearing a quarantine is itself a Tier 4+ action (per §3) requiring the same approval discipline as any other high-risk action — quarantine does not self-expire silently
- A quarantined agent may still produce a safe, pre-approved final explanation to the user (e.g., "this task was halted for review") — quarantine blocks further privileged action, not all communication

**Security Event Schema (extends, does not pollute, Card 05's observability):** security events carry fields beyond Card 05's generic observability schema — `policy_decision_id`, `denied_resource`, `requested_scope`, `granted_scope`, `risk_tier`, `token_id_hash` (never the raw token, per §18), `circuit_breaker_state`, `quarantine_reason`, `reviewer_id`, `attestation_id`. These are security-specific event types layered on top of Card 05's observability infrastructure — observability captures events generically, this schema is what Security specifically needs from those events, and Evaluation (Card 05) remains untouched by this addition.

**Alternative Architecture Considered:** Merging the Circuit Breaker into the Policy Server (one component instead of two) was considered, given both sit in Security's enforcement path. Rejected: the Policy Server *decides* (ABAC+JIT authorization, per-call evaluation of whether an action is permitted) while the Circuit Breaker *acts* on accumulated runtime signal (tripping, quarantining, scoping the blast radius of an already-in-progress execution) — the same Decide/Act separation Card 01 §2 already establishes for the execution loop generally. Collapsing them would blur a decision-time control with a runtime-state control. Kept separate, per the reconciled position in `cohesion-reviews/v1/review-reconciliation.md`.

## 12. Governance and the Immutable Audit Trail (Pillar 7)

Every real-world action (any Tier 2+ action per §3) is logged to an immutable audit trail tying it to: the specific agent identity (§4), the human who approved or deployed it (if applicable, per the HITL mechanisms unified in Card 05 §4), and the risk tier. **Audit trail minimums:** both **allowed and denied** Tier 2+ attempts are logged (a denial is itself security-relevant signal, not a non-event); audit records are **append-only and tamper-evident** (consistent with the registry integrity model in §16); access to the audit trail itself is access-controlled and logged; retention duration is **set per-project** according to that project's regulatory requirements (Projects 1 and 3 in particular should default to the longer of any applicable regulatory minimum); sensitive field values are redacted in the audit record while preserving evidentiary value (e.g., a hash or reference rather than the raw sensitive value). For Tier 5 actions specifically, this extends to **Risk-Stratified Attestation** — a cryptographic signature bound to the agent's output, creating a verifiable ledger for both internal governance and potential third-party/regulatory audit. This satisfies the regulatory-boundary requirement Card 01 originally flagged for healthcare (Project 1) and financial (Project 3) work.

## 13. Data Classification — Foundation for Pillar 2

Pillar 2's data protections (encryption, least-privilege, tenant partitioning) operate on **classification labels, not ad hoc field names** — every piece of data handled by tools, memory, traces, and evaluator agents is classified at ingest (e.g., public / internal / sensitive-PII / sensitive-PHI / regulated-financial). Security controls (redaction in §10, audit redaction in §12, the risk tier defaults in §14) reference these labels rather than hardcoding per-field logic, so a new data source is automatically governed correctly once classified, without bespoke rules per field. Full data classification taxonomy, retention policy per class, and deletion policy are implementation detail for Phase 2 — this card establishes that classification is the governing mechanism, not the specific label taxonomy.

## 14. Risk Tier Defaults — Tightened for Regulated Projects

The base tier table in §3 sets global defaults. **For Projects 1 (healthcare) and 3 (financial) specifically, those defaults are floors, not ceilings — project policy may strengthen them, and for these two projects, it must:**

| Tier | Global default | Project 1 / 3 minimum |
|---|---|---|
| **2 — Sensitive data read** | Logged, no approval gate | Logged + policy-checked; approval gate required when reading classified PHI/PII/regulated-financial data (per §13's classification) |
| **3 — Reversible write** | Audit logged only | Audit logged + approval required if the write affects downstream decisions (e.g., a fraud-suspicion tag, a label that feeds model training, a draft that may later be sent) — "reversible" is explicitly not treated as synonymous with "low risk" |

This directly answers the review's concern that the original defaults under-protected exactly the domains this ecosystem targets.

## 15. Instruction Artifact Integrity — A Major Gap, Now Closed

Pillar 3 named system instructions and prompt templates as sensitive, cryptographically attested artifacts but the original card never operationalized this. Given this ecosystem is contract- and prompt-driven (Agent/Tool/Memory/Skill contracts, AGENTS.md, SKILL.md bodies, evaluator rubrics), this is one of the largest attack surfaces in the entire architecture if left unaddressed.

**Binding requirement:** `AGENTS.md`, all four contract types (Agent/Tool/Memory/Skill), Skill bodies (Card 04), evaluator rubrics (Card 05), and system prompt templates are **signed instruction artifacts**. The Harness verifies signatures before loading any of them at runtime; **unsigned or invalid-signature artifacts are not loaded** — this fails closed, not open. A change to any instruction artifact is itself a **Tier 4 action minimum** (Tier 5 if the artifact governs a Tier 5 capability) — it requires the same approval discipline as any other high-risk action, not a quiet file edit. Prompt *injection* (an attacker manipulating runtime input) and prompt *tampering* (an attacker or bug modifying the artifact itself) are distinct threat categories handled by different controls — injection is mitigated by the LLM firewalls and hooks in Pillar 4; tampering is mitigated by this signing requirement.

**Skills specifically are a security surface, not just a knowledge surface** (direct response to the review's gap here): Card 04 defines Skills as procedural knowledge that can guide tool usage — meaning a malicious or compromised Skill can cause harmful behavior *without ever executing code directly* (e.g., instructing an agent toward an overly broad tool, omitting a required HITL step, recommending unsafe data handling, encouraging a bypass of retrieval constraints). **Skill content is untrusted until reviewed, signed, registered, and Active** (Card 04 §8's lifecycle) — a Skill is subject to the exact same integrity controls as prompts and contracts under this section, not a lesser standard because it "looks like documentation."

## 16. Registry Integrity — Runtime Verification, Not Just Write-Time Signing

§7's signing requirement is necessary but the review correctly identified it as insufficient alone. Full registry integrity minimums: **all registry changes are append-only, signed, versioned, and auditable** (mirroring the immutable audit trail model in §12); **runtime loaders reject unsigned or invalidly-signed registry entries** — verification happens every time a registry entry is loaded, not only at write time; **registry verification failure causes fail-closed behavior for the affected tool/skill specifically** (the orchestrator can still start and operate on everything else — a single bad registry entry does not halt the whole ecosystem, but that one entry is unusable until re-verified). Old signed registry versions are retained (never overwritten) so any prior state can be audited or rolled back to.

## 17. A2A Delegation Semantics — Closing the Confused Deputy Gap More Tightly

§6 described A2A trust boundaries at a high level; delegation specifically is where multi-agent systems become dangerous and needed sharper rules. **A2A authorization is evaluated against the intersection of: the requesting agent's identity, the receiving agent's identity, the original user, the original intent, the task scope, and the time window** — not any single one of these in isolation. Delegation chains carry the original user/intent context forward across every hop (an agent two hops downstream still knows whose authority it's ultimately acting under) and **delegation depth is policy-limited** — unbounded multi-hop delegation is not permitted by default. This is the concrete mechanism that prevents the Confused Deputy problem from re-emerging one layer removed, through a chain of otherwise-individually-authorized agents.

## 18. JIT Token Lifecycle — Full Minimums

§4 established JIT downscoping conceptually; full lifecycle rules close the remaining gap: **JIT tokens are short-lived; bound to agent identity + run_id + intent + resource scope (not just "this agent can use this tool" in the abstract); revocable mid-run** (this is the mechanism the Circuit Breaker in §11 calls on); **single-use or narrowly-scoped per call** rather than broad session-lived grants; **never logged in plaintext** — only a `token_id_hash` appears in any log or audit record (per §11's Security Event Schema); and **automatically invalidated on Stop, Escalate, or quarantine** — a halted run cannot have its credentials reused or replayed afterward.

## 19. Egress Governance — Clarified, Not Absolute

§5 originally stated agents never get direct interactive internet access. The review correctly flagged this as too absolute given Card 02 explicitly includes API connectors (Project 3's financial data needs current, live information) and Card 05 evaluators may need external benchmark retrieval. **Clarified rule: direct, arbitrary internet egress by an agent is prohibited. Access to current external data occurs exclusively through approved Tools/MCP servers (Card 02) or pre-sanitized retrieval services — both of which carry scoped credentials, are logged, and pass through the same Policy Server check as any other tool call.** This preserves the security intent (no agent freely browsing the open internet, vulnerable to indirect prompt injection from arbitrary pages) while leaving a sanctioned, governed path for the live-data workflows the ecosystem genuinely needs.

## 20. Policy Server Failure Mode — Closing the Single-Point-of-Failure Gap

The Policy Server (§4) is now a high-value, high-blast-radius component by design — concentrating ABAC+JIT decisions, registry write authority (§16), memory governance (§8), and Safety enforcement (§9) in one place is architecturally clean but creates a real availability and compromise risk that must have a defined failure mode, not an implicit one.

**Binding fail-closed table, by tier:**

| Tier | Behavior if Policy Server is unavailable |
|---|---|
| **0 — Computation** | May continue under local static policy (no live decision needed) |
| **1-2 — Read-only / Sensitive read** | Fail configurable per-project; **default fail-closed for Projects 1 and 3** specifically (regulated data), fail-open-with-heavy-logging permissible only for non-regulated development work |
| **3 — Reversible write** | Fail closed |
| **4-5 — External effect / Regulated** | **Always fail closed**, no exceptions, regardless of project |

**Local fallback policy integrity:** the Tier 0 local static policy referenced above is itself a signed, versioned artifact (per §15's instruction-artifact discipline) and is **strictly limited to Tier 0 actions only** — it cannot be silently extended in scope to cover higher tiers, which would otherwise turn a legitimate availability fallback into an unguarded bypass path around the Policy Server entirely.

**Break-glass override:** a break-glass path exists for genuine emergencies, but **break-glass itself is a Tier 5 action** — it requires immutable audit logging (per §12), is restricted to a named, limited set of human roles (not "anyone with admin access"), and triggers a mandatory post-incident review before normal operation resumes. Break-glass is not a convenience path around Policy Server downtime; it is a heavily audited last resort.

**Sequencing clarification (resolves the review's Card 02 cross-reference):** Card 02's binding rule (registered/assigned/Active/MCP-compatible) and this card's Policy Server enforcement are sequential, not redundant or competing checks: **the Orchestrator performs the Card 02 binding check first; if it passes, the Policy Server then authorizes and issues a scoped JIT token (§18); the tool call proceeds only once that token is granted.** Two checks, one pipeline, each owned by its respective card.

**Memory verification tightened (resolves the review's Card 03 cross-reference):** §8's "verified tool output" trigger for automatic memory verification does **not** mean any Tier 2+ tool output. It specifically means: output from a tool that is registered, Active, policy-authorized for that call, **and whose Tool Contract (Card 02 §4) classifies its output as verified/authoritative** — a deliberate, declared property of that specific tool, not an incidental side effect of the data being sensitive. A sensitive read from a tool not carrying that classification does not auto-verify anything; it still requires human review to move from `unverified` to `verified`.

## 21. Policy Server Integrity and Change Control — Securing the Security Layer Itself

§20 addressed Policy Server *unavailability*. **Policy Server *compromise* is a different failure mode** — an unavailable server fails closed and is safe; a compromised one issues valid-looking bad decisions and is dangerous in a way fail-closed behavior cannot catch. Minimum requirements:

- **Policies themselves are signed and versioned instruction/control artifacts** — the same integrity model as §15's instruction artifacts, applied to the policies the Policy Server enforces, not just the prompts/contracts it governs
- **Tier 4/5 policy changes require human approval; Tier 5 policy changes specifically require dual control (two-person review)** — no single human, however authorized, can unilaterally change a Tier 5 policy
- **Every policy decision logs its policy version alongside the decision** (extending §11's Security Event Schema — `policy_decision_id` now references `policy_version` and `policy_bundle_hash`, plus `risk_tier_version` and `data_classification_version`, per §22) — an audit record must say not just *what* was decided but *which version of policy logic* produced that decision
- **Policy Server administrative actions (changing policy, not just enforcing it) are themselves Tier 5 actions** — administering the enforcer is held to the same standard as the highest-risk actions it enforces
- **Tier 5 policy decisions require secondary/independent verification, unless explicitly exempted by a signed emergency policy** (e.g., a documented break-glass exception per §25) — a second, separate check before a Tier 5 enforcement action executes is the default, not a discretionary "may require"

## 22. Policy and Classification Versioning

Beyond policy versions specifically (§21), every enforcement decision must be traceable to the **exact version** of every governing artifact active at the time: `risk_tier_version` (the §3 table can itself evolve), `data_classification_version` (§13's taxonomy can evolve), and the evaluator-to-enforcement mapping version (§9's severity-mapping logic). Without this, an audit record tells you *what* happened but not *which rules* produced that outcome — a critical gap for regulatory reconstruction and incident investigation in Projects 1 and 3.

## 23. Data Classification Authority and Derived-Artifact Inheritance

§13 established that classification is the governing mechanism for data protection, but left *who assigns labels* and *what derived data inherits* undefined — both are security-critical, not implementation detail:

- **Classification assignment authority**: a model/agent may *propose* a classification, but does not have standing authority to *downgrade* one unilaterally. **Initial classification at ingest may be automated under approved classification policy** (e.g., sampling-based or rule-based labeling at scale) — requiring full Tier 4 human review for every single ingested record would be operationally impractical. **However, downgrading an existing classification, declassifying an artifact, or correcting a contested classification all require explicit human/security review** — the asymmetry is deliberate: labeling conservatively at scale is cheap and safe; loosening a label is exactly the action that must stay expensive and reviewed
- **Classification is monotonic by default**: **derived artifacts inherit the highest classification of their source inputs** — a summary of PHI is PHI; a trace containing PHI inherits PHI classification; a memory consolidated from PHI-classified sources inherits PHI classification; an evaluation record or skill candidate derived from sensitive sources inherits accordingly. This applies uniformly across every derived-artifact path in the ecosystem: Card 03's memory consolidation, Card 05's trace-based evaluation, Card 04's procedural-memory-to-Skill promotion candidates, and ad hoc reports
- **Declassification is the only downgrade path**, and it requires explicit, approved human/security review — never an automatic or model-initiated downgrade. Without this inheritance rule, summaries and traces become exactly the kind of leakage path Pillar 2 exists to prevent

## 24. HITL Reviewer Authorization

Card 05 unified HITL at the infrastructure/mechanism level; this card defines the authorization layer underneath that mechanism, which was previously left implicit. **HITL approval authority is itself ABAC-governed (§4)** — not every human with system access can approve every tier. Specifically: **Tier 5 approvals require separation of duties — the requester or author of a change cannot be its sole approver.** Reviewer eligibility may vary by project (Projects 1/3's regulated context may require domain-specific reviewer qualifications beyond general system access). Conflicts of interest in reviewer assignment are tracked as part of the audit trail (§12).

## 25. Break-Glass Hardening

§20 established break-glass as a Tier 5, heavily-audited last resort; full hardening constraints close the remaining gap: break-glass grants are **time-boxed** (a fixed, short maximum duration, not indefinite), **scope-limited** (to the specific emergency, not blanket access), **justification-required** (a recorded reason precedes the grant, not follows it), **automatically expiring** (no manual step required to end access — the grant lapses on its own), and trigger **immediate notification** to designated reviewers at the moment of use, not after the fact. Every action taken during an active break-glass grant is itself logged as a Tier 5 audit event (§12), and the grant is subject to the mandatory post-incident review already established in §20.

## 26. Audit Decision Provenance, Untrusted Context, Model Routing, Supply Chain Scope, and Red Team Governance — Closing Remaining Gaps

Five smaller but real gaps, closed together:

- **Audit decision provenance**: beyond agent identity and risk tier, every audit record references the **versioned contracts, policies, models, tools, skills, and instruction artifacts active at the time of the action**, plus input/output artifact hashes and the relevant attestation ID — this is what makes full audit *reconstruction* possible after the fact, not just a record that something happened
- **Untrusted context boundary**: this card's relationship to Card 03's Context Engineering is made explicit here: **all external text entering an agent's context — tool outputs, retrieved documents, web/API data, memory records, Skill bodies prior to approval, A2A messages, user uploads — is treated as untrusted data, never as instruction.** Context assembly (Card 03 §12) must preserve source boundaries so that retrieved, tool, or A2A content cannot override system or policy instructions — this is the architectural answer to prompt/tool-output injection, distinct from the artifact-tampering threat §15 addresses
- **Classification-aware model routing**: extends the zero-cost model-routing principle (Card 01 §1, `docs/architecture/model-routing-table.md`) with a security constraint — **routing decisions are classification-aware (§13/§23): sensitive or regulated data (PHI, PII, regulated-financial) may only be sent to model providers whose retention and processing guarantees satisfy project policy; absent that guarantee, local models (Ollama) are required.** This is binding for Projects 1 and 3 specifically, where the zero-cost stack's free-tier API routing (Gemini Flash, GitHub Models) must be classification-gated, not applied uniformly
- **Supply chain scope, expanded**: §5's supply-chain defenses (SBOM, signed dependencies) extend beyond Python packages to **MCP server binaries, Skill supporting scripts, model artifacts/weights, container images, registry files, and evaluator rubrics** — anything loaded and trusted by the runtime is in scope, not only `pip`-installed code
- **Red Team finding governance**: Red Team findings (§10/§11) are not informal — each produces a **tracked security issue with severity, affected artifact, mitigation owner, and a regression-test requirement** before the finding is considered closed. This connects Red Teaming to the Quality Flywheel's Improve step (Card 05 §8) as a named, versioned artifact change, without merging Red Teaming into Card 05's evaluation discipline (§10's separation holds)

**Privacy and data-subject rights** (noted, not fully specified — outside this card's architectural scope but acknowledged so it isn't silently missing): data deletion/correction requests are governed jointly through Card 03's memory/data contracts (mechanism) and this card's authorization model (§8, §24 — who may approve such a request). Applicable regulatory rights (e.g., right-to-deletion in specific jurisdictions or healthcare/financial contexts) are handled as project-specific policy on top of this shared mechanism — this card establishes the enforcement plumbing, not the legal determination of which rights apply.

---

## Direct Implications for This Build

- `core/security/policy_server/` is the single enforcement point for: ABAC+JIT permission decisions (§4), registry write authority (§7/§16), memory governance authorization (§8), and mapping Safety evaluation flags to action via risk-tier+severity+confidence (§9) — one component, multiple responsibilities, with an explicit **fail-closed table by tier** (§20) defining exactly what happens when it's unavailable, and a Tier-5, heavily-audited break-glass path as the only override
- **Layer ownership is explicit (§9a):** Security enforces; it does not own domain semantics. Memory still defines what `verified` means, Tools/Skills still define their own registry field semantics, the Harness still owns execution state — Security only ever signals transitions, never mutates state directly
- `core/security/sandbox/` implements ephemeral, network-isolated execution per §5, including supply-chain defenses and **clarified egress governance (§19)**: no arbitrary direct internet access, but live external data flows through approved Tools/MCP servers or sanctioned retrieval services, not an absolute offline-only rule
- `core/security/circuit_breaker/` implements the Green Team's stateful response (§11) with explicit semantics: it revokes credentials and emits a quarantine event, then **signals the Harness/Orchestrator to transition via Card 01's existing Escalate/Stop outcomes** — it never directly rewrites execution state. Scope (run/agent/tool/system-level), trigger thresholds, and quarantine-release-as-Tier-4-action are all Policy Server configuration, not hardcoded
- **Instruction artifact integrity (§15)** is a new, previously-missing control: AGENTS.md, all four contract types, Skill bodies, evaluator rubrics, and prompt templates are signed artifacts; the Harness fails closed on unsigned/invalid signatures; modifying any of them is a Tier 4+ action. Skills specifically are explicitly classified as a security surface, not merely a knowledge surface
- **Registry integrity (§16)** now includes runtime (not just write-time) signature verification, append-only versioning, and fail-closed behavior scoped to the single bad entry rather than the whole ecosystem
- **A2A delegation (§17)** is authorized against the full intersection of requesting agent + receiving agent + user + intent + scope + time, with delegation chains carrying original context forward and depth-limited by policy — closing the Confused Deputy gap one layer further than §6 alone covered
- **JIT tokens (§18)** have a complete lifecycle: short-lived, bound to identity+run_id+intent+scope, mid-run revocable, never logged in plaintext (only a hash), and auto-invalidated on Stop/Escalate/quarantine
- **Risk Tier defaults are tightened for Projects 1 and 3 specifically (§14)** — Tier 2 requires an approval gate for classified sensitive data, Tier 3 requires approval when the write affects downstream decisions; "reversible" is explicitly not treated as "low risk" for these two projects
- **Data classification (§13)** is the governing mechanism behind Pillar 2's protections — security controls reference classification labels, not ad hoc field names, so newly classified data is automatically governed correctly
- **Audit trail minimums (§12)** now explicit: both allowed and denied Tier 2+ attempts logged, append-only and tamper-evident, access-controlled, per-project retention, redacted-but-evidentiary sensitive values
- **Security Event Schema (§11)** extends Card 05's observability with security-specific fields (policy_decision_id, risk_tier, token_id_hash, circuit_breaker_state, etc.) without polluting Card 05's evaluation records — observability/evaluation/security-events remain three cleanly separated concerns sharing one underlying trace infrastructure
- **Green Team is explicitly defined (§11)** to avoid the term's ambiguity outside this whitepaper's usage; **Red Teaming is explicitly named a security assurance activity, not an evaluation activity**, keeping Card 05's Evaluation discipline unblurred
- **Sequencing with Card 02 is explicit (§20):** Orchestrator's binding check runs first, Policy Server authorization/JIT issuance runs second, the tool call proceeds only with a granted token — two checks, one pipeline
- **Memory verification is tightened (§20):** "verified tool output" specifically means a tool whose own Contract declares its output authoritative — not any sensitive-but-otherwise-ordinary tool response — closing a real gap the review identified in Card 03's interaction with this card
- This card now closes every outstanding forward-reference from Cards 01-05, with each prior gap traceable to the specific section here that resolves it, and with the production-hardening gaps identified across two review cycles (failure modes, override authority, audit minimums, state-ownership boundaries, and finally **the security layer's own integrity**) explicitly closed rather than left implicit
- **The Policy Server's own integrity is now governed, not assumed (§21):** policies are signed/versioned artifacts, Tier 5 policy changes require dual control, every decision logs its policy version, and Policy Server administration is itself a Tier 5 action — compromise and unavailability are now both addressed as distinct failure modes
- **Classification has assignment authority and inheritance rules (§23):** models propose, humans/ABAC-governed review assigns; classification is monotonic by default across every derived-artifact path in the ecosystem (memory consolidation, evaluation traces, skill promotion candidates); declassification is the only downgrade path and always requires approval
- **HITL reviewer authorization is explicit (§24):** Tier 5 requires separation of duties — no self-approval — closing a real production-governance gap
- **Break-glass is fully hardened (§25):** time-boxed, scope-limited, justification-required, auto-expiring, with immediate reviewer notification — not merely "audited," but constrained at the point of grant
- **Model routing is classification-aware (§26):** the zero-cost stack's free-tier routing (Gemini Flash, GitHub Models) is gated by data classification for Projects 1 and 3 — sensitive/regulated data defaults to local models absent a satisfactory provider guarantee
- **The untrusted-context boundary (§26) explicitly connects Card 03 and this card:** all externally-sourced text in context is data, never instruction — the architectural answer to prompt/tool-output injection, distinct from artifact tampering (§15)
- **Supply chain scope is comprehensive (§26):** MCP binaries, skill scripts, model artifacts, containers, registries, and rubrics are all in scope, not just Python packages
- **Red Team findings are governed, not informal (§26):** tracked issues with severity/owner/mitigation/regression-test requirements, connected to the Quality Flywheel without merging into Card 05's evaluation discipline
- **Audit records now support full reconstruction (§26):** every record references the versioned contracts, policies, models, tools, skills, and instruction artifacts active at the time of the action, plus input/output hashes — answering not just "what happened" but "under exactly which rules"
