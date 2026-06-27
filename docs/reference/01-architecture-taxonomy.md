# Reference Card 01 — Agent Architecture & Taxonomy

**Card Version:** 4.0
**Changelog:** §2 — added "Implementation Note (LangGraph)" clarifying that the crash-recovery blast-radius statement depends on a durable checkpointer backend, not the default `MemorySaver`. Sourced from a Phase 0 platform/framework docs scan, post-Closure Plan.
**Changelog (v3.0):** §2 — added "Blast radius — involuntary Harness/Orchestrator crash" statement (checkpoint-based recovery, idempotency discipline for in-flight calls). Closure Plan Stage 6.
**Changelog (v2.0):** §8's observability field list replaced with a pointer to Card 05 §5 — the restated list had drifted stale (still listed "evaluation score," which Card 05 §5 explicitly excludes). Closure Plan Stage 3.

**Source whitepapers:** Introduction to Agents and Agent Architectures (Nov 2025) · Spec-Driven Production Grade Development (Day 5, May 2026, foundational framing only)
**Purpose:** Defines what an agent is, the 5-level capability taxonomy, the universal problem-solving loop, the 3 core architectural components, and the major multi-agent design patterns. This is the conceptual bedrock every other card and every component in `core/` builds on.

---

## 1. What Makes Something an "Agent"

An agent is not just a chatbot with extra steps. The defining trait is **closing the loop between reasoning and the real world** — a standard LLM call generates text and stops; an agent generates text, *acts on it through tools*, observes the result, and reasons again. Four components make this possible:

- **Model** ("the Brain") — the LLM doing the reasoning
- **Tools** ("the Hands") — functions/APIs/databases the agent can invoke to affect or sense the world
- **Orchestration** ("the Nervous System") — the logic that manages state, sequences steps, and routes between Model and Tools
- **Contract** ("the Job Description") — the standard specification every agent is built against before implementation: mission, inputs/outputs, tools, memory access, success criteria, stop conditions, escalation rules, evaluation metrics, HITL requirements. Originally treated as a downstream artifact, but on reflection this belongs at the same foundational tier as Model/Tools/Orchestration — every agent in this ecosystem should be instantiated *from* a contract, not designed ad hoc and documented after the fact. Full template lives at `docs/architecture/agent-contract-template.md` (Phase 0, file 0.12).

**On state:** Orchestration is described above as managing "state," which deserves a precise definition rather than being left implicit. **State is the authoritative record of an agent's current execution context and progress through the execution loop** — at minimum: the current mission, execution progress (which loop stage it's in), tool outputs collected so far, retry counts, any human approvals received, checkpoints, and intermediate artifacts produced along the way. State is what makes the Validate/Decide stage in §2 possible — you cannot validate progress or decide to retry/escalate without an authoritative record of what has happened so far. Full implementation treatment (where state lives, how it's persisted, how it relates to memory) is Card 03's responsibility — this definition exists so the concept isn't left undefined until then.

## 2. The Universal Problem-Solving Loop

Every agent, regardless of level, runs the same core cycle — extended here beyond the textbook 5-step version to include the failure-handling controls a production system requires:

**Mission → Scan → Plan → Act → Observe → Validate → Decide** (then loop back to Plan, or exit)

- **Mission**: the goal/task given to the agent
- **Scan**: gather relevant context (memory, prior state, available tools)
- **Plan**: reason about what to do next — framed deliberately as planning/decision-making, not as exposed chain-of-thought. Implementation should produce a structured decision (which tool, which args, why) and a brief reasoning summary for the trace, not a raw stream-of-thought transcript. This matters for both cost (verbose CoT burns tokens) and security (Card 06 — verbose reasoning traces are an injection/leakage surface)
- **Act**: invoke a tool or produce output
- **Observe**: ingest the result of the action
- **Validate**: check the observed result against expected success criteria — did the tool call actually succeed, is the output well-formed, does it satisfy the step's goal?
- **Decide**: based on validation, choose one of four outcomes:
  - **Continue** — proceed to the next planning step
  - **Retry** — re-attempt the action (with a bounded retry policy, defined per-tool in Card 02)
  - **Escalate** — surface to a human or a higher-tier agent (HITL pattern, see §6)
  - **Stop** — terminate the loop, with a clear reason logged

This loop is the pattern to implement in `core/harness/` — it is framework-agnostic and applies whether the agent is built in LangGraph, ADK, or raw Python. The Validate/Decide stage is what separates a demo loop from a production-grade one: agents do not merely act, they operate within bounded permissions and observable, recoverable execution traces.

**Blast radius — involuntary Harness/Orchestrator crash (distinct from a deliberate `Stop`):** a `Stop` is a logged, intentional Decide outcome; a crash is not. On Harness restart after an involuntary crash, recovery resumes from the last persisted `state` checkpoint (§1's `state` definition) rather than restarting the mission from scratch — in-flight tool calls without a confirmed completion record are treated as failed and subject to the same retry/idempotency discipline as any other tool failure (Card 02 §4), never silently assumed successful. If no checkpoint exists (the crash occurred before the first checkpoint was written), the mission is treated as never having started, and any partial side effects from tool calls made before the crash are exactly why idempotency marking (Card 02 §3) matters at this boundary, not just for ordinary retries.

**Implementation Note (LangGraph):** the above blast-radius statement assumes the persisted `state` checkpoint actually survives the crash — this is not automatic. LangGraph's `MemorySaver` checkpointer only persists state in-process RAM and is lost on any restart, making it unsuitable for anything beyond local development. Production deployment must use a durable checkpointer backend (e.g., `PostgresSaver`, `SqliteSaver`) — defaulting to `MemorySaver` during early development and silently shipping that default would make this section's blast-radius guarantee false in practice.

## 3. The 5-Level Agent Taxonomy

Use this to scope what we are building at each phase — don't over-engineer Level 3/4 behavior into Phase 1's "hello world" agent.

| Level | Name | Capability | Example |
|---|---|---|---|
| **0** | Core Reasoning | Pure LLM, no tools, single-turn | Plain chat completion |
| **1** | The Connected Problem-Solver | Uses tools to access real-time info (search, API, RAG) | "What was last night's score?" → calls search tool |
| **2** | The Strategic Problem-Solver | Multi-step planning + **context engineering** (actively curating what info matters at each step) | Multi-call plan: find midpoint → search nearby → filter by rating → synthesize |
| **3** | Collaborative Multi-Agent System | Specialist agents treat *other agents* as tools; a coordinator delegates sub-missions | PM agent delegates to MarketResearch, Marketing, WebDev agents |
| **4** | Self-Evolving System | Agent identifies its own capability gaps and creates new tools/agents to fill them | PM agent spins up a new SentimentAnalysis agent on the fly |

**Where our ecosystem sits:** Not every agent in the ecosystem should default to Level 2 planning capability — that's costly and unnecessary for simple sub-tasks. The correct mix, per agent role:

- **Deterministic validators** (no LLM call at all — plain code checks) for structural validation, schema checks, simple pass/fail gates
- **Level 1 retrieval agents** for straightforward data fetch/lookup sub-tasks
- **Level 2 planning agents** reserved for genuinely multi-step, ambiguous sub-tasks requiring context engineering
- **Level 3 coordinators** at the top of each project pipeline, delegating to the appropriate tier above

This tiering is a cost and maintainability lever as much as a capability one — it directly serves the zero-cost design goal (cheaper/faster tiers handle the bulk of sub-tasks; expensive planning capability is reserved for where it's actually needed). Level 4 (self-evolving, autonomous tool/agent creation) is explicitly out of scope for this build — see §8 for the explicit boundary.

## 4. Model Selection — Not Just "Pick the Best Benchmark Score"

Model choice should be driven by the actual task, not leaderboard rank. Key guidance:

- Test candidate models against metrics that map to *your* business outcome, not generic benchmarks
- Cross-reference quality against cost and latency — the "best" model sits at the optimal intersection of all three for the specific task
- **Use a team of specialist models, not one model for everything.** Route heavy reasoning/planning to a frontier-quality model; route high-volume simple tasks (classification, summarization) to a fast/cheap model
- This routing principle is the direct ancestor of our `docs/architecture/model-routing-table.md` — model routing is a cost AND quality lever, not just a quality one

## 5. Memory — Two Architectural Tiers (full taxonomy in Card 03)

- **Short-term/working memory**: scoped to a single session — implemented as state, threads, or session objects
- **Long-term memory**: persists across sessions

This two-tier split is a simplification for orientation purposes only. Long-term memory is **not** just a vector database — Card 03 distinguishes five distinct memory types (episodic, semantic, procedural, preference, and **Structured Records** — note: not "structured state," to avoid colliding with this card's own `state` definition above), with vector/RAG retrieval as just one mechanism among several. Do not implement long-term memory as "just RAG" — consult Card 03 before building `core/memory/`.

## 6. Multi-Agent Design Patterns

Four patterns to choose from when decomposing a complex task across specialist agents — but first, a more fundamental decision: **does this workflow need an agent coordinator at all?**

**Deterministic orchestration vs. agentic orchestration:** Not every workflow benefits from an LLM-driven coordinator. Where the workflow structure is already known and stable — ETL steps, a fixed data-cleaning sequence, a fixed train/evaluate/report pipeline — use plain deterministic code (regular Python/LangGraph edges, no LLM call) to move between steps. Reserve agentic coordination for genuine ambiguity: prioritization calls, adaptive routing, research-style sub-tasks where the next step isn't knowable in advance. This split is also a direct cost lever — every unnecessary LLM-mediated routing decision is tokens spent on something a plain `if/else` could have done for free.

With that filter applied, four patterns cover the cases where agentic coordination *is* warranted:

- **Coordinator pattern** — a manager agent segments a request and routes sub-tasks to specialists, then aggregates results. Use for dynamic/non-linear tasks where routing logic itself requires reasoning.
- **Sequential pattern** — output of one agent feeds directly into the next, like an assembly line. Use for linear workflows — note many "sequential" pipelines are actually deterministic-orchestration candidates per the filter above, not agentic ones.
- **Iterative Refinement pattern** — a generator agent produces content, a critic agent evaluates it against quality standards, loop until acceptable. Use when output quality is high-stakes.
- **Human-in-the-Loop (HITL) pattern** — deliberate pause for human approval before a significant/irreversible action. Use for high-risk operations (deployments, financial transactions, schema changes). HITL escalation is also one of the four outcomes of the Decide step in §2 — the two concepts are linked, not separate mechanisms.

**Application to our 3 projects:** The core data pipeline (ingest → clean → feature engineer → model → evaluate → report) is largely deterministic orchestration with fixed steps — it does not need an LLM coordinator to decide what comes next. Agentic coordination is reserved for genuinely ambiguous sub-decisions within that pipeline (e.g., which feature engineering strategy fits this dataset's anomalies). Healthcare fraud detection and financial analysis both warrant HITL gates before any flagged-case report ships, regardless of which orchestration style produced it.

## 7. Agent Ops — Why Testing Agents Differs From Testing Code

Traditional software tests assert `output == expected`. Agent outputs are probabilistic, so this doesn't work. Agent Ops (evolution of DevOps/MLOps for agentic systems) replaces binary pass/fail with:

- **KPI-driven observability** — frame it like an A/B test: define what "better" means in business terms (goal completion rate, latency, cost per interaction) before optimizing anything
- **LLM-as-Judge evaluation** — use a strong model to score outputs against a rubric (correctness, factual grounding, instruction-following) against a curated "golden dataset" of scenarios
- **Metrics-driven deployment** — every change is tested against the full golden dataset and compared to the current production score before shipping; eliminates subjective "looks fine to me" sign-off

Full evaluation framework detail lives in Card 05. This card only establishes *why* the paradigm shift from deterministic testing is necessary.

## 8. Layers Defined Elsewhere — Forward References

This card is the conceptual bedrock, not the full specification. The following layers are *named* here so they're not forgotten, but are fully defined in their dedicated cards/documents — do not build them from this card alone:

- **Security architecture** (tool permission scopes, secret management, audit logging, sandboxing, data governance, regulatory boundaries for healthcare/finance, risk tiers for action types) → **Card 06 — Security**
- **Tool contracts** (input/output schema, timeout behavior, error contract, retry policy, idempotency, observability metadata per tool) → **Card 02 — Tools, MCP & Interoperability**
- **Observability standard** — see Card 05 §5 for the current observability field schema; not restated here to avoid drift → **Card 05 — Quality & Evaluation**
- **Agent Contract Standard** — now established as a core architectural component in §1, on par with Model/Tools/Orchestration. Full template is formalized separately as `docs/architecture/agent-contract-template.md` (Master Execution Plan, Phase 0, file 0.12) — not duplicated here.

**Explicit Level 4 boundary (scope control):** To prevent future scope creep, the following are explicitly prohibited anywhere in this ecosystem unless this document is formally revised: autonomous tool creation, autonomous code modification, autonomous permission escalation, and production deployment without human review. If a future project seems to need one of these, that is a signal to revisit this card deliberately — not to build around it quietly.

---

## Direct Implications for This Build

- Every agent is instantiated **from** a contract (mission, inputs/outputs, tools, memory access, success criteria, stop conditions, escalation rules, evaluation metrics, HITL requirements) — the contract is a core architectural component on par with Model/Tools/Orchestration, not an afterthought documented post-hoc. Template: `docs/architecture/agent-contract-template.md` (Phase 0, file 0.12)
- `core/harness/` implements the full Mission→Scan→Plan→Act→Observe→Validate→Decide loop, including the four Decide outcomes (Continue/Retry/Escalate/Stop) — not the simplified 5-step version. The Validate/Decide stage operates on **state** (defined in §1) as its authoritative input
- Agent role assignment follows the tiered model in §1's "Where our ecosystem sits": deterministic validators and Level 1 retrieval agents handle simple sub-tasks; Level 2 planning capability is reserved for genuinely ambiguous steps; Level 3 coordinators sit atop each project pipeline only where routing logic itself requires reasoning
- `core/orchestrator/` defaults to **deterministic orchestration** (plain code, no LLM call) for known/stable pipeline steps; agentic coordination (Coordinator pattern) is reserved for genuine ambiguity — this is a cost and maintainability decision, not just an architectural one
- `docs/architecture/model-routing-table.md` formalizes the "team of specialists" model selection principle using our zero-cost stack (Ollama local / Gemini Flash / GitHub Models)
- HITL checkpoints are mandatory before any fraud-flag or financial-recommendation output ships in Projects 1 and 3, and are one of the four Decide-stage outcomes, not a separate bolt-on mechanism
- Security, tool contracts, observability, and evaluation standards are explicitly deferred to Cards 06, 02, and 05 respectively — Card 05 in particular must keep observability and evaluation as separate sections, and must (along with or instead Card 06) introduce a formal failure taxonomy. See the Master Execution Plan's "Runtime Stack" section for the full governing structure across all cards
- We are explicitly building toward Level 1–3 capability depending on agent role. Level 4 (self-evolving, autonomous tool/code creation, permission escalation, unreviewed deployment) is explicitly prohibited — noted here to prevent scope creep later
