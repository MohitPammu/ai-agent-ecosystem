# Reference Card 03 — Context Engineering & Memory

**Card Version:** 1.0 (Approved)

**Source whitepaper:** Context Engineering: Sessions, Memory (2025 Day 3, May 2026 update)
**Governing structure:** Occupies the "State Management" and "Memory System" positions in the Runtime Stack (Contract → Harness → **State → Memory** → Tools → Evaluation/Observability → Security). Builds directly on the `state` definition established in Card 01 §1. Does not redefine Tool Contracts (Card 02) or Security (Card 06).
**Purpose:** Defines Context Engineering as a discipline, the Session vs. Memory distinction, the per-turn context lifecycle, the five memory types (correcting Card 01's original RAG-only framing), and multi-agent history-sharing patterns.

---

## 1. Context Engineering — The Core Discipline

LLMs are stateless outside their training data — all reasoning is confined to whatever is in the context window for that single call. **Context Engineering** is the dynamic, per-turn process of assembling exactly what the model needs — no more, no less — from several distinct sources. It's an evolution beyond Prompt Engineering: prompt engineering crafts static instructions; context engineering constructs the *entire payload* fresh each turn, blending instructions, history, retrieved knowledge, and the immediate task.

**The mise en place analogy (use this when explaining the concept):** giving a chef only a recipe (the prompt) produces an inconsistent meal. Giving them the recipe *plus* properly prepped, high-quality ingredients (engineered context) produces a reliably excellent result. The goal is the most relevant information, not the most information.

**The two failure modes this discipline guards against:**
- **Cost/latency blowup** — naively appending all history to every call scales token usage linearly with conversation length
- **Context rot** — even within a large context window, a model's attention to critical information degrades as irrelevant volume grows. More context is not automatically better context

## 2. What Goes Into the Payload — Three Categories

| Category | Components | Role |
|---|---|---|
| **Context to guide reasoning** | System instructions, tool definitions (Card 02's contracts), few-shot examples | Defines the agent's behavior and capabilities |
| **Evidential & factual data** | Long-term memory, external knowledge (RAG), tool outputs, sub-agent outputs, artifacts | The "evidence" the agent reasons over |
| **Immediate conversational info** | Conversation history, state/scratchpad, the user's current prompt | Grounds the agent in the current turn |

Note tool outputs and sub-agent outputs are themselves context inputs — this is the direct link back to Card 02 (every tool call's result becomes part of the next turn's assembled context, which is exactly why Card 02's data-minimization principle matters for cost here too).

## 3. The Per-Turn Context Lifecycle

Every turn runs this 4-step cycle — this is the operational mechanism `core/harness/` must implement around the Plan/Act/Observe stages from Card 01 §2:

1. **Fetch Context** — retrieve relevant memories, RAG documents, recent events, using the current query/metadata to decide what's relevant
2. **Prepare Context** — assemble the full prompt. This is a blocking, "hot-path" step — the agent cannot proceed until context is ready
3. **Invoke LLM and Tools** — iterate calls until a final response is produced; tool/model output gets appended to context as it's generated
4. **Upload Context** — persist new information from this turn to storage. Often run as a background process so the agent doesn't block on it. This step is not a simple "save" — it expands into three sub-steps detailed in §7 (Memory Consolidation): **Consolidate** (deduplicate, merge with existing related memories, summarize where needed) → **Validate** (check for contradiction against existing verified memory, assign confidence/source/timestamp metadata per §5) → **Store** (write to the appropriate type-specific backend)

## 4. Session vs. Memory — The Fundamental Split

- **Session**: manages turn-by-turn state for a *single* conversation. Think of it as a filing cabinet with two folders — one for **events** (the chronological, append-only conversation history) and one for **state** (mutable working memory/scratchpad for in-progress data, per Card 01's state definition)
- **Memory**: long-term persistence *across* sessions — consolidating key facts so the agent doesn't start from zero in the next conversation

**Framework implementation differs** — relevant because we're using LangGraph: ADK uses an explicit Session object (events list + separate state object). **LangGraph has no formal "session" object — the state IS the session.** State holds conversation history (as a Message list) plus all other working data, and unlike an append-only event log, LangGraph state is mutable — it can be transformed, summarized, or compacted directly. This is directly useful for managing context rot (§1) — compaction strategies operate on this same mutable state object.

## 5. The Five Memory Types (Correction to Card 01's Original Framing)

Card 01 originally implied long-term memory was primarily RAG/vector-database driven. That was too narrow. Long-term memory spans five distinct types, with RAG/vector retrieval as just one mechanism among several:

| Type | Captures | Typical storage |
|---|---|---|
| **Episodic** | Past executions, specific decisions, and their outcomes ("what happened last time") | Structured logs, sometimes vectorized for retrieval |
| **Semantic** | Domain and project knowledge — facts about the world/problem space | Vector DB (RAG) is well-suited here |
| **Procedural** | Reusable workflows, playbooks, processes — "how things get done" | This is the direct conceptual bridge to **SKILL.md** (Card 04) — a Skill is procedural memory made explicit and loadable |
| **Preference** | User and project-specific preferences | Structured key-value store; small, frequently accessed |
| **Structured Records** *(renamed from "structured state")* | Authoritative records — the kind of data you wouldn't want a model paraphrasing | SQL/document store, not vector retrieval |

**Naming note:** the fifth type is deliberately called **Structured Records**, not "structured state." Card 01 already defines `state` as the authoritative record of an agent's *current execution context* — a distinct, session-scoped concept. Reusing "state" here for a long-term memory type would collide with that definition. Structured Records are authoritative *external* data (e.g., a Medicare claims table, a confirmed financial transaction) — not the agent's own execution progress.

**Memory metadata — every stored memory, regardless of type, carries:**

| Field | Purpose |
|---|---|
| **confidence** | How reliable is this memory — was it directly observed, inferred, or model-generated? |
| **source** | Where did this come from — a tool output, user statement, model inference, consolidation process? |
| **timestamp** | When was this captured/last updated |
| **verification_status** | Verified / unverified / contested — feeds the source precedence model in §7 below |

Without these fields, the system has no way to distinguish a verified Medicare claims record from a model's unconfirmed inference about a user's preference — a real risk in regulated domains (Projects 1 and 3).

**Direct implication:** `core/memory/` is not "a vector database." It is a system that routes to the right storage mechanism per memory type — semantic facts go to a vector store, procedural knowledge is expressed as Skills (Card 04) rather than duplicated here, Structured Records live in PostgreSQL (consistent with our existing PG18 setup), and preference/episodic memory get lightweight structured storage. Every record carries the metadata fields above regardless of type. Building `core/memory/` as "just RAG" would be a design mistake this card exists to prevent.

## 6. Multi-Agent History Sharing — Two Patterns

When multiple agents collaborate (Coordinator pattern, Card 01 §6), history can be shared two ways:

- **Shared, unified history** — all agents read/write to one central log. Best for tightly coupled tasks where one agent's output directly feeds the next (our Sequential pipeline steps: ingest → clean → feature-engineer → model). A sub-agent can still filter/label the shared log before using it.
- **Separate, individual histories** — each agent keeps a private log and is a black box to others; communication happens only via explicit final-output messages (Agent-as-a-Tool, or full A2A per Card 02 §7). Best for loosely coupled specialist agents where internal reasoning shouldn't leak across boundaries.

**Application to our ecosystem:** the core project pipeline (ingest→clean→feature-engineer→model→evaluate→report) uses **shared, unified history** — these steps are tightly coupled and benefit from a single source of truth. If a project ever delegates to a genuinely independent specialist (e.g., a future cross-project agent call per Card 02 §7's A2A scenario), that interaction uses **separate histories** — only the final output crosses the boundary, not the specialist's internal reasoning trace. This decision should be made explicitly per-agent-relationship, not defaulted blindly.

**Shared-history write hygiene:** shared write access creates a contamination risk — one agent's noisy intermediate scratch work can be mistaken by another agent as settled evidence. Every event written to a shared history should carry a label identifying its nature: `observation`, `intermediate`, `final`, `tool_output`, `human_approved`, or `rejected`. Consuming agents should treat unlabeled or `intermediate`-labeled events with lower trust than `final` or `human_approved` ones — this label set is a lightweight implementation detail to carry into Phase 1's harness build, not a new architectural layer.

## 7. Memory Retrieval Policy

Defining what memory types exist (§5) is necessary but not sufficient — the system also needs a defined strategy for pulling memories back into context. The pipeline:

**Retrieve → Rank → Filter → Compress → Inject**

- **Retrieve**: pull candidate memories matching the current query/task, per-type (vector similarity for semantic, recency for episodic, direct lookup for preference/structured records)
- **Rank**: order candidates by relevance — not just similarity score, but weighted by confidence and recency (per the metadata in §5)
- **Filter**: drop low-confidence, stale, or contested memories below a threshold; this is where verification_status matters most
- **Compress**: summarize or truncate retrieved content to fit the context budget (§9) — raw retrieval results are rarely injected verbatim
- **Inject**: place the final, compressed set into the assembled context per §2's payload structure

**Conflict handling during retrieval:** when multiple retrieved memories disagree (e.g., two episodic records imply different outcomes for a similar past task), the system defers to the **Source Precedence** order in §10 rather than picking arbitrarily or always trusting the most recent.

## 8. Memory Consolidation — Expanding "Upload"

The naive version of Upload ("persist new information") degrades memory quality over time through duplication and drift. The full consolidation process:

- **Consolidate**: check new information against existing memory of the same type — deduplicate near-identical entries, merge related ones, summarize verbose raw content before storing
- **Validate**: check the consolidated memory for contradiction against existing *verified* memory (per §5 metadata and §10 precedence); contradictions are flagged rather than silently overwritten
- **Store**: write to the type-appropriate backend with full metadata (confidence, source, timestamp, verification_status)

This is a background process by default (per §3), but the Validate sub-step's contradiction flags should surface to a human review queue for high-risk projects (Projects 1 and 3) rather than auto-resolving silently — this is a HITL touchpoint (Card 01 §6) applied to memory specifically.

**Consolidation provenance risk:** when Consolidate uses an LLM to summarize or merge memories (rather than simple deduplication), that summarization step is itself a generation step subject to the same risks as any model output — incorrect summaries, lost caveats, or outright hallucinated detail. For high-risk projects (1 and 3), the raw pre-consolidation source must be retained alongside the consolidated summary, not discarded — so a contested or suspicious consolidated memory can always be traced back to what was actually observed, not just what the model said it observed.

## 9. Context Budget Management

Context rot (§1) is a quality problem; budget overflow is a cost and hard-limit problem. Both are managed by enforcing explicit budgets, allocated as a fraction of the model's total context window per category:

- **Token budget** — overall ceiling for the assembled payload
- **History budget** — how much raw conversation history is retained before compaction kicks in
- **Retrieval budget** — how many memories/RAG chunks may be injected per turn (output of the Filter/Compress steps in §7)
- **Tool-output budget** — how much of a tool's return value is retained in context (ties directly to Card 02's data minimization principle — a tool that already minimizes its output makes this budget easier to hold)

When a budget is exceeded, the system must have a defined fallback (summarize further, drop lowest-ranked items first, or escalate per Card 01's Decide stage) rather than silently truncating or erroring.

## 10. Source Precedence — Resolving Conflicts

When sources disagree (a tool output conflicts with episodic memory; a user preference conflicts with semantic memory; structured records conflict with retrieved knowledge), the system needs a default precedence order rather than resolving conflicts arbitrarily:

**Structured Records → Verified Tool Outputs → Preferences → Semantic Memory → Conversation History**

Rationale: authoritative external records (a confirmed claims entry, a confirmed transaction) outrank everything else since they are ground truth, not inference. Verified tool outputs come next since they reflect a real-time check against the world. Stated preferences outrank inferred semantic knowledge since the user said it directly. Conversation history sits last because it's the least curated, highest-volume, and most prone to containing outdated statements superseded by later turns.

This precedence is a default, not an absolute rule — specific projects may need to override it (documented per-project if so), but the default must exist rather than leaving conflict resolution undefined.

**Regulated-domain exception:** Preferences are advisory, not authoritative, when they conflict with compliance or safety requirements. A user preference such as "ignore small anomalies" must not override fraud-detection logic if records indicate suspicious activity — Projects 1 (healthcare) and 3 (financial) in particular must treat Preferences as capped below regulatory/compliance requirements regardless of where Preferences sit in the general precedence order above. The precise boundary of where preferences are binding vs. advisory per project is Card 06/project-policy territory, not redefined here — this note exists so the exception isn't silently missed when §10's general ordering is applied literally.

## 11. Memory Governance and Memory Contracts

Borrowing the same discipline established for Agent Contracts (Card 01) and Tool Contracts (Card 02), every memory **type** (not every individual memory record) should have a lightweight contract defining:

| Field | Purpose |
|---|---|
| **memory_type** | Episodic / Semantic / Procedural / Preference / Structured Records |
| **retention_policy** | How long this type persists by default before archival/expiration consideration |
| **retrieval_policy** | Which step(s) of §7's pipeline apply, and any type-specific weighting |
| **update_policy** | Can this type be corrected/overwritten, or only appended/superseded? |
| **ownership** | Who/what can write to this memory type (full enforcement is Card 06's authority — this field records that a restriction exists) |
| **confidence_model** | How confidence is assigned for this type (e.g., structured records default to maximum confidence since they're authoritative; model-inferred preferences default lower) |

**Scope discipline note, consistent with how Cards 01 and 02 handled similar territory:** this card defines *that* retention/deletion/correction policies exist and what fields they need — it does not define *who is authorized* to invoke deletion or correction, or how that authorization is enforced at runtime. That enforcement layer is Card 06's responsibility (the same pattern as risk tiers in Card 02 — defined here as a reference field, governed there). Memory Contracts should be drafted per-type once `core/memory/` enters active build (Phase 1), not fully populated in this card before any memory type has been implemented.

---

## Direct Implications for This Build

- `core/harness/` implements the 4-step Fetch→Prepare→Invoke→**Consolidate/Validate/Store** context lifecycle (the expanded Upload step from §3/§8) wrapping each Plan/Act/Observe cycle from Card 01 §2
- `core/memory/` is built as a multi-type router, not a single vector database — semantic memory → vector store, Structured Records → PostgreSQL, procedural memory → deferred entirely to Card 04's Skills system, preference/episodic → lightweight structured storage. Every stored record carries confidence/source/timestamp/verification_status metadata (§5) regardless of type
- Memory retrieval follows the Retrieve→Rank→Filter→Compress→Inject pipeline (§7); raw retrieval results are never injected verbatim
- Context budgets (token/history/retrieval/tool-output, §9) are explicit and enforced, with a defined fallback behavior on overflow — never silent truncation
- Conflicts between sources default to the precedence order in §10 (Structured Records → Verified Tool Outputs → Preferences → Semantic Memory → Conversation History) unless a project explicitly overrides it
- Memory Contracts (§11) are drafted per-type once `core/memory/` enters active build in Phase 1 — not pre-populated here. Authorization/enforcement for memory ownership and deletion is explicitly Card 06's responsibility, consistent with how Card 02 deferred tool permission enforcement
- LangGraph's state-as-session model is used directly — no separate Session object is built; compaction/summarization strategies operate on this same mutable state to manage context rot and token cost
- Within a single project's pipeline, agents use **shared, unified history** by default (tightly coupled, sequential); separate histories are reserved for genuinely independent specialist relationships, decided explicitly per case
- Tool outputs and sub-agent outputs are context inputs subject to the same data-minimization discipline from Card 02 — bloated tool returns inflate every subsequent turn's context, not just the current one, and make the tool-output budget (§9) harder to hold
- This card corrects and supersedes Card 01's original "long-term memory = RAG" framing, and renames the fifth memory type from "structured state" to **Structured Records** to avoid colliding with Card 01's `state` definition — Card 01 should be read with this card's five-type model and corrected terminology in mind
- High-risk projects (1 and 3) route memory-consolidation contradiction flags (§8) to human review rather than auto-resolving — a HITL touchpoint specific to memory, not just to output shipping
