# Tech Stack

**Status:** 2.0 (Approved)
**Changelog:** Decision #7 — added clarifying sentence distinguishing the logical Policy Authority role (Card 06) from its OPA WASM-embedded implementation, cross-referencing Card 06 §4's new terminology clarification. Patch 2, Stage 10 pre-freeze.

**Purpose:** The canonical record of *which specific technology* satisfies each card's architectural requirement, and *why*. Cards 01-07 define what's required; this file decides what's used. No requirement is re-derived here — every choice cites the card/section it satisfies. This is the single source of truth for technology decisions — README's Tech Stack section and the Master Plan's header summarize this file, they do not compete with it.

**Replacement philosophy:** every decision below is an *implementation choice*, not an architectural dependency — consistent with the same discipline Card 02 §5 applies to Tool Registry storage format ("doesn't need to be a separate database initially... the discipline of enforcing the binding rule is what matters, regardless of storage format"). The architecture requires *a* local model runtime, *a* durable checkpointer, *a* policy-evaluation engine — Ollama, PostgresSaver, and OPA are this project's current answers, replaceable without any card needing to change, as long as the replacement satisfies the same cited requirement.

**Review Trigger, defined once:** the measurable condition under which a technology decision should be formally reconsidered. A trigger firing does **not** automatically imply replacement — it initiates architectural review, the same review discipline applied everywhere else in this project, not an automatic swap.

Every technology section below follows the same five-field order: **Satisfies → Decision → Rationale → Tradeoffs → Decision Status / Review Trigger.**

---

## Technology Selection Principles

Every decision below is evaluated against this shared set of principles, rather than justified independently each time:

1. Zero recurring cost preferred
2. Production-proven technologies over novel ones
3. Mature ecosystem and community support
4. Minimal operational complexity at solo-developer scale
5. Components remain replaceable — no choice should become an unstated architectural dependency
6. Vendor lock-in avoided
7. Security and sustainability prioritized over setup speed, consistent with this project's explicit capex-over-opex posture

---

## Architectural Technologies

*(Implement a specific card requirement — distinct from the general implementation platform below.)*

### 1. Orchestration — LangGraph

**Satisfies:** Card 01 §2 (the Plan→Act→Observe→Decide loop), Card 03 §4 (state-as-session model).

**Decision:** LangGraph. No alternative seriously considered — this was a pre-existing project constraint, not a Stage 7 decision.

**Rationale:** n/a — pre-existing constraint.

**Tradeoffs:** ties state management and checkpointing conventions to LangGraph's specific model (state-as-session, no separate Session object — Card 03 §4's Implementation Note) rather than a framework-agnostic design.

**Decision Status:** Accepted
**Review Trigger:** Project abandoned; licensing changes; architecture's needs outgrow graph-based orchestration entirely.

### 2. Model Layer — Ollama (local) / Gemini Flash (free tier) / GitHub Models (Copilot subscription)

**Satisfies:** Card 01 §4 (specialist-model routing), Card 06 §13/§23/§26 (classification-aware routing).

**Decision:** Three-provider routing per the zero-cost stack. Full routing logic and provider approval evidence: `docs/architecture/model-routing-table.md` — not restated here.

**Rationale:** see routing table.

**Tradeoffs:** see routing table §1-10 (provider-specific tradeoffs already documented there in full; not duplicated here).

**Decision Status:** Accepted
**Review Trigger:** Provider terms change (per the routing table's own expiration discipline); a future local-runtime replacement (LM Studio, vLLM, llama.cpp) offers a clear advantage over Ollama specifically.

### 3. Memory Storage — PostgreSQL with `pgvector`

**Satisfies:** Card 03 §5 (five memory types — specifically Structured Records and semantic memory's vector storage need).

**Decision:** one PostgreSQL instance for both Structured Records and semantic/vector memory, via the `pgvector` extension — not a separate dedicated vector database (Chroma, Qdrant). Episodic, procedural, and preference memory use lightweight structured storage within the same instance.

**Rationale:** at solo-developer scale, running two separate database technologies adds operational surface area for no benefit this project currently needs. `pgvector` gets vector similarity search inside the same Postgres instance already required for Structured Records, reducing the stack to one database.

**Tradeoffs:** slightly slower vector search than a specialized vector database; fewer approximate-nearest-neighbor tuning options than Qdrant/Chroma offer natively.

**Decision Status:** Accepted
**Review Trigger:** Retrieval scale or query pattern genuinely outgrows what `pgvector` supports — not anticipated within the three reference projects.

### 4. LangGraph Checkpointer — `PostgresSaver`

**Satisfies:** Card 01 §2's blast-radius statement (crash recovery depends on a durable checkpointer) and its own Implementation Note (LangGraph), which explicitly disqualifies `MemorySaver` for anything beyond local dev/testing.

**Decision:** `PostgresSaver`, not `SqliteSaver`.

**Rationale:** directly follows from Decision #3, keeping the stack at one database rather than introducing SQLite solely for checkpointing.

**Tradeoffs:** ties checkpointing availability to the same Postgres instance as memory storage — a single point of failure shared across two concerns, rather than isolated.

**Decision Status:** Accepted
**Review Trigger:** Tied to Decision #3 — only reconsidered if the underlying PostgreSQL choice itself changes.

### 5. Observability — OpenTelemetry + self-hosted Jaeger

**Satisfies:** Card 05 §5 (Observability Standard's required fields, captured as trace spans), Card 06 §11 (Security Event Schema, extending the same trace infrastructure).

**Decision:** OpenTelemetry SDK instrumentation + a self-hosted Jaeger backend (`docker-compose`), not LangSmith. Resolves a pre-existing inconsistency between the Master Plan (previously named LangSmith) and README (previously named "OpenTelemetry-style" generically) — this file is now the single answer.

**Rationale:** LangSmith's free tier is lower-setup-cost and LangGraph-native, but carries usage caps that risk becoming binding exactly when observability matters most — a real-money trading agent or unattended e-commerce agent running continuously (per the Post-portfolio Deferred entry). OpenTelemetry + Jaeger has no usage ceiling, no external dependency, no recurring cost, ever — accepted deliberately per this project's capex-over-opex principle.

**Tradeoffs:** real upfront setup cost (manual span instrumentation across LangGraph nodes/tool calls/model calls, roughly a day's initial setup); a small ongoing tax per newly instrumented component; self-hosted means you own the maintenance LangSmith would otherwise handle.

**Decision Status:** Accepted
**Review Trigger:** Self-hosted maintenance burden becomes genuinely heavy; a future LangGraph-native OTel integration removes the manual-instrumentation cost entirely.

### 6. Sandboxing — Docker, per execution

**Satisfies:** Card 06 §5 (ephemeral, network-isolated execution for dynamically executed code).

**Decision:** a Docker container per sandboxed execution, not a restricted Python subprocess.

**Rationale:** a subprocess approach is faster to set up and faster per-call, but provides materially weaker isolation; adopting it now would mean a real later migration once any application carries genuine personal stakes (the trading agent, specifically). Built in from Phase 1 rather than deferred and retrofitted, consistent with this project's sustainability-over-speed principle.

**Tradeoffs:** weaker isolation than a microVM approach (e.g., Firecracker); slower per-execution startup than a bare subprocess, a real cost for fast-iteration workflows.

**Decision Status:** Accepted
**Review Trigger:** Kubernetes adoption (multi-host orchestration needed); a stronger isolation requirement emerges that Docker alone can't satisfy.

### 7. Policy Server Engine — Open Policy Agent (OPA), WASM-embedded mode

**Satisfies:** Card 06 §4 (ABAC+JIT authorization), §20 (Policy Server failure mode), §21 (Policy Server integrity — policies as signed, versioned artifacts).

**Decision:** OPA, compiled to WASM and evaluated in-process — not a custom Python authorization module, and not OPA's server mode. This is the implementation of the **logical Policy Authority** Card 06 describes — "Policy Server" in the cards is the architectural role, not a claim that a standalone service process runs; see Card 06 §4's terminology clarification.

**Rationale:** a custom Python module was the initial default, reconsidered on security grounds — a bug in general-purpose application code can more easily become an authorization bypass than a bug in a narrowly-scoped, dedicated policy-evaluation engine. OPA has real production pedigree (Kubernetes admission control, Envoy, Istio) and ships its own policy-unit-testing tooling, extending Card 07's spec-driven discipline to policies themselves. Server mode rejected to avoid a second running service at solo scale.

**Tradeoffs:** Rego is a real learning curve, distinct from Python's imperative model; policies require their own compilation step before evaluation; adds a second testing discipline (policy tests, separate from application tests) to maintain.

**Decision Status:** Accepted
**Review Trigger:** WASM-embedded evaluation becomes insufficient for required throughput/latency; centralized, multi-service policy management becomes genuinely necessary (server mode reconsidered at that point, not before).

### 8. Deployment — Docker Compose (current); Kubernetes (future, if scale requires)

**Satisfies:** no single card section directly — the deployment substrate underneath the entire Runtime Stack, closing a gap the original draft omitted.

**Decision:** Docker Compose for current solo-scale deployment — every service this stack now requires (Jaeger, PostgreSQL/pgvector, sandboxed execution containers) runs as a Compose service on one machine. Kubernetes explicitly not adopted now.

**Rationale:** Kubernetes solves multi-host orchestration problems this project doesn't have yet — adopting it now would be premature optimization against a scale that doesn't exist.

**Tradeoffs:** Docker Compose doesn't provide auto-scaling, rolling updates, or multi-host scheduling — fine at solo scale, a real limitation the moment any application needs to run across more than one machine.

**Decision Status:** Accepted
**Review Trigger:** Multi-host deployment becomes necessary (e.g., a real personal application — the trading agent or e-commerce agent — needs to run continuously on infrastructure beyond a single local machine).

---

## Implementation Platform

*(General-purpose tooling and language constraints — distinct from the architectural technologies above, which each satisfy a specific card requirement.)*

### 9. Language — Python 3.12

**Satisfies:** n/a — pre-existing project constraint, stated here for completeness.

**Decision:** Python 3.12.
**Rationale:** n/a — pre-existing constraint.
**Tradeoffs:** n/a.
**Decision Status:** Accepted
**Review Trigger:** None anticipated.

### 10. Testing — `pytest`

**Satisfies:** Card 07's spec-driven discipline (every BDD-style spec needs an executable test).

**Decision:** `pytest`. No alternative seriously considered.

**Rationale:** the standard, free, Python-native testing choice; nothing about this project's constraints suggested deviating.

**Tradeoffs:** none material at this project's scale.

**Decision Status:** Accepted
**Review Trigger:** None anticipated.

### 11. Instruction-Artifact Signing — Git commit signing (GPG/SSH)

**Satisfies (partially — see clarification):** Card 06 §15 (signed instruction artifacts — AGENTS.md, all contract types, Skill bodies, evaluator rubrics, prompt templates).

**Decision:** Git signed commits (GPG or SSH) as the current implementation.

**Rationale:** already-available infrastructure (GitHub natively supports both) — satisfies the provenance/authorship half of §15's intent with zero new tooling. Forward-compatible with the "Require signed commits" branch protection setting discussed and deliberately deferred elsewhere this session.

**Clarification, important:** Git commit signing proves *who committed* an artifact — it is **not** runtime signature verification. Card 06 §15/§16 describe the Harness/runtime failing closed on an unsigned or invalid signature *at the point of use*, which commit signing alone does not provide; it's a provenance record checked by a human reviewing history, not an automated gate checked by the running system. Runtime artifact-signature verification is deferred, not yet built (see Intentionally Deferred Technology Decisions, below).

**Tradeoffs:** provides authorship/provenance only; does not protect against a correctly-signed-but-malicious artifact being trusted at runtime — that gap is real and currently open.

**Decision Status:** Accepted (provenance-only scope); runtime verification deferred, not yet decided.
**Review Trigger:** Card 06's enforcement requirements evolve to require runtime verification explicitly, ahead of whatever phase would otherwise have triggered it.

---

## Intentionally Deferred Technology Decisions

Not yet selected — deliberately, not by oversight. Full "why deferred / when to revisit" detail lives once in the Master Execution Plan's Polish/Deferred Enhancements list, not duplicated here; this section exists only so a future reader isn't left wondering why these are absent from the list above:

- CI platform (beyond the existing GitHub Actions badge already referenced in README)
- Secret manager (beyond the `.gitignore`/secret-scanning baseline already in place)
- Container registry
- Cloud provider
- Backup solution
- Scheduler
- Message queue
- Runtime artifact-signature verification (see Decision #11's clarification, above)

---

## Sign-off

| Field | Value |
|---|---|
| **Status** | 2.0 (Approved) |
| **Approved by** | Mohit Pammu |
| **Approval date** | 2026-07-11 |

**Versioning note:** follows the same convention as Cards 01-07 and the other Stage 7 artifacts — version increments by one integer per substantive edit, with a one-line changelog added at the point of each edit, once approved.

**Version policy for all technologies above:** use the current stable major release of each technology unless a documented compatibility requirement dictates otherwise — exact version numbers are deliberately not pinned in this document, since they go stale almost immediately and add no durable value here.
