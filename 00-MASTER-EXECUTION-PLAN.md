# AI Agent Ecosystem — Master Execution Plan

**Owner:** Mohit Pammu
**Purpose:** Build a modular, production-grade, multi-agent ecosystem that powers three (potentially four) data science portfolio projects, demonstrating industry-current agentic engineering practices for the job search.
**Cost constraint:** Zero additional spend. All tooling drawn from existing subscriptions (Claude Pro, ChatGPT Plus/Copilot) or free tiers (Ollama local, Gemini Flash free tier, GitHub Models, LangSmith free tier).
**Working environment:** `ds-core` conda environment, Python 3.12, M5 Pro MacBook.
**Knowledge base:** 7 distilled reference cards derived from 10 Google/industry whitepapers on agentic systems (architecture, tools/MCP, context/memory, skills, evaluation, security, spec-driven production).

---

## How to Use This Document

This is the **single source of truth** for build order and progress. It does three jobs:

1. **Defines the build sequence** — what gets built, in what order, and why that order minimizes rework
2. **Tracks status** — every file/artifact has a status flag you update as we go
3. **Anchors session continuity** — see the companion document `01-SESSION-PROMPT-GUIDE.md` for exactly how to resume work in any session

**Status flags used throughout:**
- `NOT STARTED` — not yet begun
- `IN REVIEW` — drafted, currently being scrutinized/refined together
- `APPROVED` — finalized, locked, safe to build on top of
- `BUILT` — code/config implemented and tested against the approved spec

**Golden rule for this build:** No file moves to `BUILT` until its spec is `APPROVED`. No downstream file starts until its upstream dependency is `APPROVED` or `BUILT`. This is the discipline from the Spec-Driven Development framework (Day 5) — specs before code, one layer fully vetted before the next begins.

---

## GitHub Strategy

**Decision:** Public repos, split by concern (Option A). The ecosystem architecture is documented and pushed publicly alongside the three portfolio projects — not kept private. Rationale: the underlying concepts (MCP, SKILL.md, agent harnesses, evaluation pipelines) are drawn from public industry whitepapers, not proprietary IP. What's valuable to demonstrate is disciplined, spec-driven engineering practice and the ability to explain the architecture — not secrecy. Consistent commit activity across these repos is itself a signal to reviewers.

**Repo structure:**

| Repo | Contents | Depends on |
|---|---|---|
| `ai-agent-ecosystem` | `core/`, `skills/`, `mcp-servers/`, `docs/` — the shared framework | — |
| `healthcare-fraud-detection` | Project 1 implementation | `ai-agent-ecosystem` |
| `fantasy-football-intelligence` | Project 2 implementation | `ai-agent-ecosystem` |
| `financial-analysis-platform` | Project 3 implementation | `ai-agent-ecosystem` |

**Commit cadence:**
- Commit to `ai-agent-ecosystem` as each file in Phases 0–4 reaches `APPROVED` (spec) and again at `BUILT` (implementation) — i.e., two natural commit points per file
- Once a project enters Phase 5/6/7, commits split: framework-level changes still go to `ai-agent-ecosystem`, project-specific work goes to that project's own repo
- Each project repo's README links back to `ai-agent-ecosystem` and briefly explains the shared-framework relationship — this is the narrative that shows the compounding architecture story in an interview

**Commit message discipline (per Day 5 whitepaper's review guidance):** Use clear, descriptive messages tied to the master plan's file IDs, e.g. `0.10 AGENTS.md: initial approved draft` or `1.2 core/orchestrator: implement LangGraph state machine`. This makes the commit history itself a readable build log alongside the progress table in this document.

---

## Runtime Stack — Governing Structure for Cards 02–06

This dependency chain emerged from the Card 01 review process and should govern how every subsequent reference card and `core/` component is written, so that responsibilities stay cleanly separated with minimal overlap:

```
Agent Contract
      ↓
Execution Harness
      ↓
State Management
      ↓
Memory System
      ↓
Tool Layer
      ↓
Evaluation & Observability
      ↓
Security & Governance
```

**Binding constraints this implies for cards not yet written:**

- **Card 05 (Quality & Evaluation)** must keep **Observability** ("what happened" — traces, logs, runtime telemetry, cost tracking, tool execution history) and **Evaluation** ("was the result good" — golden dataset testing, LLM-as-judge scoring, KPI measurement, regression evaluation) as clearly separated sections, not conflated into one undifferentiated "quality" topic.
- **Card 05 or Card 06** must introduce a formal **failure taxonomy** — explicit classification of failure types (tool failures, model failures, data failures, validation failures, permission failures, timeout failures, human-approval failures). This is currently missing ecosystem-wide and directly strengthens the Validate/Decide stage of the execution loop (Card 01 §2) by giving "Decide" something concrete to branch on.
- Every card from 02 onward should be checked against this stack at approval time: does it respect the layer below it, and does it avoid re-defining a concern owned by another layer?

---

## Directory Structure (Target State)

```
ai-agent-ecosystem/
├── README.md
├── AGENTS.md
├── docs/
│   ├── reference/                     (7 distilled whitepaper cards)
│   ├── architecture/                  (model routing, tech stack, diagrams)
│   ├── specs/                         (BDD-style specs, one per feature)
│   └── governance/                    (review/QA artifacts — emerged organically, not originally planned)
│       ├── ecosystem-cohesion-review-rubric.md   (frozen v1.0)
│       ├── PHASE-0-CLOSURE-PLAN.md    (active, 10-stage sequential remediation plan — updated as stages complete)
│       ├── architecture-traceability-matrix.md   (Closure Plan Stage 2 — created, seeded from Stage 1 audit)
│       ├── adr/                       (Closure Plan Stage 8 — not yet populated)
│       ├── architecture-verification-specification.md   (Closure Plan Stage 9 — not yet created)
│       └── cohesion-reviews/v1/       (versioned historical snapshot — claude-independent/, chatgpt-independent/, synthesis, reconciliation)
├── core/                              (orchestrator, harness, memory, eval, observability)
├── skills/                            (SKILL.md library — reusable across all projects)
├── mcp-servers/                       (tool connectors)
├── projects/
│   ├── 01-healthcare-fraud-detection/
│   ├── 02-fantasy-football-intelligence/
│   ├── 03-financial-analysis-platform/
│   └── README.md
├── tests/
└── .agents/
```

Full rationale for this structure was established in our architecture discussion — see `docs/architecture/tech-stack.md` once written.

---

## PHASE 0 — Foundation Documents

**Goal:** Establish the knowledge base and top-level specification before any code exists. Nothing in Phase 1+ starts until Phase 0 is fully `APPROVED`.

| # | File | Purpose | Status |
|---|---|---|---|
| 0.1 | `docs/reference/01-architecture-taxonomy.md` | Distilled: agent anatomy, 5-level taxonomy, 5-step loop | APPROVED |
| 0.2 | `docs/reference/02-tools-mcp.md` | Distilled: tool design, MCP architecture, A2A, NxM problem | APPROVED |
| 0.3 | `docs/reference/03-context-memory.md` | Distilled: session mgmt, 4 memory types, retrieval patterns | APPROVED |
| 0.4 | `docs/reference/04-skills-framework.md` | Distilled: SKILL.md anatomy, progressive disclosure, taxonomy. Must answer the procedural-memory-to-Skill promotion question raised in Card 03's review: when does procedural memory become a Skill, who approves it, can agents create/modify Skills, are Skills versioned, are Skills executable instructions or reference docs | APPROVED |
| 0.5 | `docs/reference/05-quality-evaluation.md` | Distilled: outside-in/inside-out eval, LLM-as-judge, flywheel | APPROVED |
| 0.6 | `docs/reference/06-security.md` | Distilled: 7-pillar security, zero ambient authority, sandboxing | APPROVED |
| 0.7 | `docs/reference/07-spec-driven-production.md` | Distilled: SDD, BDD/Gherkin, code review tiers, guardrails | APPROVED |
| 0.8 | `docs/architecture/tech-stack.md` | Final stack decisions: LangGraph, vector DB, model routing | NOT STARTED |
| 0.9 | `docs/architecture/model-routing-table.md` | Which model/tier handles which task category (zero-cost design) | NOT STARTED |
| 0.10 | `AGENTS.md` | Master ecosystem specification — the architectural North Star | NOT STARTED |
| 0.11 | `README.md` | Top-level repo overview | NOT STARTED |
| 0.12 | `docs/architecture/agent-contract-template.md` | Standard specification template every agent is built against (mission, inputs/outputs, tools, memory access, success criteria, stop conditions, escalation rules, evaluation metrics, HITL requirements) | NOT STARTED |
| 0.13 | `docs/governance/ecosystem-cohesion-review-rubric.md` | 14-dimension rubric for evaluating Cards 01-07 as one integrated specification — not part of the original Phase 0 plan; emerged organically out of the Card 01-07 review cycles. Frozen at Version 1.0 | APPROVED (frozen v1.0) |
| 0.14 | `docs/governance/cohesion-reviews/v1/` | Per the rubric's own 6-phase process: two independent reviews → merged findings → consolidated action list → edits applied → final confirmation review. v1 cycle complete through Phase 3 (both independent reviews done, findings reconciled in `review-reconciliation.md`, action list consolidated in `PHASE-0-CLOSURE-PLAN.md`). Phases 4-6 (apply edits, final confirmation review) remain. Phase 0 is not considered fully closed until this completes with a Freeze-Ready verdict | IN REVIEW |

**Exit criteria for Phase 0:** All 14 files `APPROVED` (12 originally planned + 2 that emerged organically from the review process: the cohesion rubric and the cohesion review cycle itself). AGENTS.md in particular must be scrutinized line by line since every later file inherits from it. The agent contract template (0.12) must be approved before any agent in Phase 1+ is specced. **Critically: Phase 0 is not closed simply because all 7 reference cards (0.1-0.7) are individually approved** — the ecosystem-cohesion review (0.14) must also reach a Freeze-Ready verdict per the rubric's own 6-phase process before Phase 1 begins. Individual card approval and ecosystem-level cohesion are two distinct gates.

---

## PHASE 1 — Core Infrastructure (The Factory)

**Goal:** Build the parts that never change across projects. This is the highest-leverage phase — flaws here propagate into all three projects. Housekeeping items carried over from Card 03's review cycle, to be addressed during this phase rather than pre-specified in the card: (1) per-type Memory Contracts (Card 03 §11) get drafted during 1.5/1.6, not before; (2) default context budget allocations (Card 03 §9) need concrete numbers (e.g., system/instructions reserved, history 20-30%, retrieved memory 20-30%, tool outputs bounded per tool, completion reserve always preserved) — defined at implementation time; (3) verification_status state-transition rules (who can mark verified/contested, can contested memory be injected) are Card 05/06 territory, consumed by Card 03's implementation once defined. Card 01 should also receive a brief terminology correction noting Card 03 renamed "structured state" to "Structured Records" to avoid colliding with Card 01's own `state` definition.

| # | File/Component | Purpose | Status |
|---|---|---|---|
| 1.1 | `docs/specs/orchestrator-spec.md` | BDD spec for the orchestrator agent | NOT STARTED |
| 1.2 | `core/orchestrator/` | Orchestrator implementation (LangGraph) | NOT STARTED |
| 1.3 | `docs/specs/harness-spec.md` | BDD spec for the agent harness (state, tools, feedback loops) | NOT STARTED |
| 1.4 | `core/harness/` | Harness implementation | NOT STARTED |
| 1.5 | `docs/specs/memory-spec.md` | BDD spec for memory system (working/episodic/semantic/procedural/preference/structured-records) | NOT STARTED |
| 1.6 | `core/memory/` | Memory implementation (local vector DB + PostgreSQL + structured stores, per Card 03) | NOT STARTED |
| 1.7 | `docs/specs/observability-spec.md` | BDD spec for OpenTelemetry logging (session/think/tool spans) | NOT STARTED |
| 1.8 | `core/observability/` | Observability implementation | NOT STARTED |
| 1.9 | `docs/specs/evaluation-harness-spec.md` | BDD spec for the 7-dimension evaluation pipeline | NOT STARTED |
| 1.10 | `core/evaluation/` | Evaluation harness implementation | NOT STARTED |

**Exit criteria for Phase 1:** A "hello world" agent can run through the full loop (mission → scan → think → act → observe), log a complete trace, store and retrieve a memory, and pass a basic evaluation check — with zero project-specific code involved.

---

## PHASE 2 — Security Layer

**Goal:** Wrap the Phase 1 factory in the production-grade safety net before any real data touches it. Card 06 (Security & Governance) is APPROVED and now fully specifies risk tiers, the Permission Model (ABAC+JIT), sandboxing, A2A trust boundaries, registry integrity, memory governance authorization, instruction-artifact signing, Policy Server integrity/failure modes, data classification with inheritance rules, HITL reviewer authorization, and break-glass governance — closing every forward-reference accumulated from Cards 01-05. Remaining items are genuine implementation detail (not architecture gaps), to be made concrete during this phase: full data classification taxonomy and per-class retention/deletion policy (Card 06 §13); named break-glass role list (Card 06 §20/§25); specific circuit-breaker trigger thresholds (Card 06 §11); JIT token storage mechanism (Card 06 §18); model-routing-table.md provider approval metadata (provider_approved_for_classes, retention_policy, zero_retention_available, etc. — Card 06 §26, tracked for Card 07/final cohesion); and the untrusted-context-boundary runtime mechanism (source labels, instruction/data delimiters, tool-output sanitization — Card 06 §26, a Harness/Context Engineering implementation item).

| # | File/Component | Purpose | Status |
|---|---|---|---|
| 2.1 | `docs/specs/sandboxing-spec.md` | BDD spec for ephemeral execution sandboxing | NOT STARTED |
| 2.2 | `core/security/sandbox/` | Sandbox implementation | NOT STARTED |
| 2.3 | `docs/specs/policy-server-spec.md` | BDD spec for the Policy Server (structural + semantic checks) | NOT STARTED |
| 2.4 | `core/security/policy_server/` | Policy Server implementation | NOT STARTED |
| 2.5 | `docs/specs/context-hygiene-spec.md` | BDD spec for PII masking / placeholder injection | NOT STARTED |
| 2.6 | `core/security/context_resolver.py` | Context resolver implementation | NOT STARTED |
| 2.7 | `docs/specs/circuit-breaker-spec.md` | BDD spec for intent drift detection + rollback | NOT STARTED |
| 2.8 | `core/security/circuit_breaker/` | Circuit breaker implementation | NOT STARTED |

**Exit criteria for Phase 2:** A deliberately malicious or malformed tool call is caught, logged, and gracefully refused by the system — verified with a test case.

---

## PHASE 3 — Skills Library

**Goal:** Build the reusable, swappable capability layer. Items deliberately deferred from Card 04 to be addressed during this phase: skill dependency declarations (`depends_on`/`optional_dependencies`/`conflicts_with` fields on the Skill Contract), skill freshness tracking (`last_reviewed`/`validation_date`), and — if a future ecosystem need arises — deeper composition precedence layers beyond the current Core→Domain split (Global→Core→Project→Task→Session).

| # | File/Component | Purpose | Status |
|---|---|---|---|
| 3.1 | `skills/data-ingestion/SKILL.md` | Generic data ingestion skill | NOT STARTED |
| 3.2 | `skills/eda/SKILL.md` | Exploratory data analysis skill | NOT STARTED |
| 3.3 | `skills/feature-engineering/SKILL.md` | Feature engineering skill | NOT STARTED |
| 3.4 | `skills/modeling/SKILL.md` | Model training/selection skill | NOT STARTED |
| 3.5 | `skills/evaluation/SKILL.md` | Model evaluation skill (distinct from agent evaluation in core/) | NOT STARTED |
| 3.6 | `skills/reporting/SKILL.md` | Stakeholder-ready reporting skill | NOT STARTED |

**Exit criteria for Phase 3:** Each skill loads correctly via progressive disclosure (metadata-only by default, full content on trigger) and has been smoke-tested with a trivial synthetic dataset.

---

## PHASE 4 — MCP Connectors

**Goal:** Build the tool-connection layer. Before or during this phase, two items deferred from Card 02 must be defined: (1) explicit transition criteria for each Tool Lifecycle stage (e.g., what specifically must be true for Designed→Security Reviewed), and (2) minimum tool-level certification/test requirements (schema validation, happy-path, failure-path, timeout, retry/idempotency, redaction/logging, data-minimization tests) referenced in Card 02 §6. These were deliberately left undefined in Card 02 to avoid over-specifying before any real tool existed.

| # | File/Component | Purpose | Status |
|---|---|---|---|
| 4.1 | `mcp-servers/file-system-connector/` | Local file system MCP server | NOT STARTED |
| 4.2 | `mcp-servers/postgres-connector/` | PostgreSQL MCP server (your existing PG18 setup) | NOT STARTED |
| 4.3 | `mcp-servers/api-connectors/` | Generic REST API MCP connector template | NOT STARTED |

**Exit criteria for Phase 4:** Orchestrator can discover and call each MCP server's tools successfully.

---

## PHASE 5 — Project 1: Healthcare Fraud Detection

| # | File/Component | Purpose | Status |
|---|---|---|---|
| 5.1 | `projects/01-healthcare-fraud-detection/README.md` | Project overview, scope, goals | NOT STARTED |
| 5.2 | `domain-skills/medicare-claims/SKILL.md` | Domain knowledge skill | NOT STARTED |
| 5.3 | Data MCP connector config (CMS Medicare) | Data source wiring | NOT STARTED |
| 5.4 | Unsupervised modeling pipeline | Isolation forest / clustering | NOT STARTED |
| 5.5 | Compliance-aware reporting agent | Output layer | NOT STARTED |
| 5.6 | Full evaluation run + documentation | Proof of working system | NOT STARTED |

*(Detailed file-by-file breakdown to be expanded once Phase 0-4 are approved — premature to over-specify this far ahead.)*

---

## PHASE 6 — Project 2: Fantasy Football Intelligence
*(To be expanded after Phase 5 begins — same reasoning as above.)*

## PHASE 7 — Project 3: Financial Analysis Platform
*(To be expanded after Phase 6 begins.)*

---

## Working Principles for Every File We Build

1. **Spec before code, always.** Every component gets a BDD-style spec (`Scenario / Given / When / Then`) reviewed and approved before implementation begins.
2. **One file at a time.** We do not parallelize drafting. Each file is read, scrutinized, revised, and approved before the next is opened.
3. **No silent assumptions.** If a design decision is ambiguous, it gets surfaced as a question, not resolved unilaterally.
4. **Token economy is a design constraint, not an afterthought.** Every spec considers which model tier (local Ollama / Gemini Flash / GitHub Models) it will run on, per the model routing table.
5. **Update this document after every session.** Status flags and the session log (companion document) are not optional housekeeping — they are how continuity across sessions is maintained.
6. **Build locally first, then push.** Every file is created and verified in the local `ds-core` environment before being committed. Commits follow the cadence and message convention defined in the GitHub Strategy section above.
7. **Every reference card carries its own version number, independent of the Master Plan's status flag.** Each card now opens with a `**Card Version:**` line (e.g., "1.0 (Approved)"). The Master Plan's `APPROVED`/`BUILT` status tracks *workflow state*; the card's own version line tracks *content state* and must be self-sufficient if the card is ever read in isolation, outside this repo. **Rule:** a card touched during the Phase 0 Closure Plan (or any future edit cycle) gets its version bumped (e.g., 1.0 → 2.0) at the point the edit is made, with a one-line changelog note added directly below the version line stating what changed and why. A card that is *not* touched during a given remediation cycle remains at its current version — version numbers are per-card and only increment on actual content change, never incremented in lockstep just because other cards changed.

---

## Polish / Deferred Enhancements
Items that are genuinely worth doing but deliberately deferred to a better-timed moment in the build — tracked here so they aren't lost. Revisit this list at the start of any session when there's bandwidth beyond the core build sequence. **Ordered soonest-trigger-first**, so the top of the list is the most relevant to check next.

| Item | Why deferred | Best time to revisit |
|---|---|---|
| **`implementation_reference` naming** (generic pointer vs. `implementation_module`/`implementation_package`) on the Agent Contract Template | Stage 7 production-readiness review (Revision 2) | Once `tech-stack.md` (Stage 7, Action 4) settles actual language/packaging conventions |
| **`related_adr` field** on the Agent Contract Template's Artifact References section | Stage 7 production-readiness review (Revision 2) | Once Stage 8 produces ADR-001 through 007 — adding a pointer to documents that don't exist yet is premature |
| **Docs site** (e.g., `mkdocs` + GitHub Pages, turning `docs/` into a browsable site) | Premature with only 3-7 reference cards; pays off once specs, reference cards, and architecture docs are numerous enough that flat markdown browsing on GitHub becomes unwieldy | Once all 7 reference cards + Phase 0 architecture docs are approved |
| **Visual architecture diagram** (SVG/PNG of the 7-layer Runtime Stack, for `docs/architecture/`) | The current text-based diagram is sufficient for active build work; a polished visual is most worth the effort once the stack is partially or fully built, so the diagram reflects reality rather than aspiration | Once Phase 1 (Core Infrastructure) is substantially complete |
| **Critical vs. optional `runtime_dependencies` distinction** on the Agent Contract Template | Stage 7 production-readiness review (Revision 2) | Phase 3+, once degraded-mode logic for multi-agent pipelines becomes a live concern |
| **Contract Compatibility fields** (`minimum_contract_version`/`maximum_supported_version`) on the Agent Contract Template | Stage 7 production-readiness review | Phase 4+, if multi-agent cross-version compatibility becomes a live problem |
| **Editorial pass on repeated cross-reference phrasing** (e.g., "mirroring Card 02" appears frequently across Cards 02-04) | Purely stylistic — repetition is fine and even helpful during active development for traceability, but reads better varied once the architecture is frozen (e.g., "following the same governance pattern as Card 02") | A final editorial pass once all 7 cards are approved and frozen, before any external-facing publication of the reference docs |

**Review strategy note (from the Card 04 review cycle, applies going forward to Cards 05-07):** the foundation (Cards 01-04) is now strong enough that review emphasis should shift from "is content missing within this card" toward **cross-card consistency, governance symmetry, and hidden contradictions** between cards. This is the more likely failure mode in mature, multi-card architectures — keep this framing in mind for Cards 05, 06, and 07 specifically, since they sit furthest downstream and have the most surface area to silently contradict something already locked in.

---

## Progress Log

*(Append entries here after each working session — newest at top.)*

| Date | Session Focus | Files Touched | Outcome |
|---|---|---|---|
| 2026-06 | Ran Closure Plan Stage 7, Action 1 (Agent Contract Template) | `docs/architecture/agent-contract-template.md`, `docs/governance/architecture-traceability-matrix.md`, this plan | Drafted the full field-by-field Agent Contract Template per Card 01 §1's specification, cross-referencing the now-fully-corrected card set (Stages 3-6) throughout rather than restating any owned concept. Put through two external production-architecture review passes; accepted 7 substantive additions across both passes — Dependencies, State Ownership, Execution Constraints, Failure Recovery Strategy (scoped to map onto Card 01 §2's existing four Decide outcomes, explicitly closing the loop against future invented outcomes), Assumptions, Artifact References, and a sharpened Blast Radius framing ("what capability is unavailable" vs. "what happens to this agent") — plus formatting tightening on Mission/Success Criteria/Stop Conditions toward single-sentence/checklist/bullet conventions. Renamed `owner` to `contract_owner` for disambiguation. Four items explicitly deferred rather than added or silently dropped (Contract Compatibility fields, critical/optional dependency distinction, `related_adr` field, `implementation_reference` naming) — added to the existing Polish / Deferred Enhancements section, which was also re-sorted soonest-trigger-first (was chronological-by-review-session) and had its bolding made consistent across all entries. Rejected outright, no tracking needed: Risk Matrix, Permission Tables, Tool/Memory Schemas, Evaluation Rubrics, Security Controls, display_name, and a standalone Change History table (the existing stacked-changelog convention already covers this). Matrix's combined "Four Missing Foundational Artifacts" placeholder row split into four individual rows now that one is complete. This is the first of the four Stage 7 artifacts — Stage 7 itself remains open pending AGENTS.md, model-routing-table.md, and tech-stack.md. |
| 2026-06 | Ran Closure Plan Stage 6 (Close the Blast-Radius Gaps) | `docs/reference/03-context-memory.md`, `docs/reference/02-tools-mcp.md`, `docs/reference/04-skills-framework.md`, `docs/reference/05-quality-evaluation.md`, `docs/reference/01-architecture-taxonomy.md`, `docs/governance/architecture-traceability-matrix.md`, this plan | Extended Card 06 §20's proven fail-closed-by-tier pattern to the four components the rubric's Dimension 13 flagged as lacking explicit blast-radius statements, each stating both halves per Dimension 13's requirement (what breaks AND what still works). Card 03 §12: Memory backend unavailability degrades reads (in-session state + non-memory context only, explicitly flagged in trace) rather than hard-failing, with Tier 2+ Structured Records reads deferring to Card 06 §20 instead. Card 02 §5 / Card 04 §6: whole-registry-file unavailability (distinct from Card 06 §16's single-bad-entry case) fails closed system-wide for new bindings, while in-flight JIT-tokened calls may still complete — documented symmetrically on both the Tool and Skill Registry sides. Card 05 §5: Evaluation-service unavailability never blocks the Harness (Observability has no dependency on Evaluation), backfilling once recovered, except Tier 4-5 HITL escalations gated on a Safety flag, which fall back to Card 06 §20 regardless. Card 01 §2: involuntary Harness/Orchestrator crash (distinct from a deliberate `Stop`) recovers from last persisted `state` checkpoint, treating unconfirmed in-flight calls as failed under Card 02's existing idempotency discipline rather than assuming success. Five cards edited, each receiving its own version bump with a stacked, self-contained changelog (Card 01: 2.0→3.0, Card 02: 2.0→3.0, Card 03: 3.0→4.0, Card 04: 2.0→3.0, Card 05: 1.0→2.0 — its first edit since freeze). One real catch during drafting: Card 02 has no §16 (that section lives in Card 06, Registry Integrity) — corrected to anchor the registry-file blast-radius statement in Card 02 §5 instead, where the Registry and binding rule actually live, before any edit was applied — a reminder to verify section numbers against the live file rather than the matrix's shorthand citations when drafting new content. Matrix's Blast-Radius row updated to reflect the now-distributed, fully-documented state across all four components. Stage 6 exit criteria met — every major component the rubric names now has an explicit, stated degradation behavior. |
| 2026-06 | Ran Closure Plan Stage 5 (Operationalize the Untrusted-Context-Boundary Mechanism) | `docs/reference/03-context-memory.md`, `docs/reference/06-security.md`, `docs/governance/architecture-traceability-matrix.md`, this plan | Closed the last live Stage 1 defect. Added Card 03 §12 ("Source Boundary Preservation") defining a `source_type` tagging convention (system/tool_output/retrieved/memory/a2a_message/user_upload) and an explicit non-override rule — no non-`system` source may be interpreted as overriding a `system`-tagged instruction, regardless of phrasing — giving Card 06 §26's untrusted-context-boundary requirement an actual mechanism rather than an assertion. Enforcement authority for violations is correctly deferred to Card 06, consistent with §8/§11's existing deferral pattern. Card 06 §26's citation corrected from the vague "Card 03 §2-3" to the precise "Card 03 §12." Both cards received a second edit each this cycle — Card 03 now Version 3.0 (1.0→2.0 Stage 4 LangGraph relabel, 2.0→3.0 Stage 5 new §12), Card 06 now Version 3.0 (1.0→2.0 Stage 4 Alt-Architecture note, 2.0→3.0 Stage 5 citation fix) — both with stacked, self-contained changelog histories per the convention established this session. Matrix's Untrusted-Context-Boundary row replaced with a new resolved row tracking the §12 mechanism directly. Stage 5 exit criteria met — zero remaining Stage 1 defects of any kind. |
| 2026-06 | Ran Closure Plan Stage 4 (Document Reconciled Anti-Bloat Decisions) | `docs/reference/02-tools-mcp.md`, `docs/reference/04-skills-framework.md`, `docs/reference/06-security.md`, `docs/reference/03-context-memory.md`, `docs/governance/architecture-traceability-matrix.md`, this plan | Checked against the matrix before editing — three of these four rows were "Pending — Stage 4" with no live defect, since these are documented decisions, not contradictions. Added "Alternative Architecture Considered" notes to both sides of the Tool/Skill Registry split (Card 02 §5, Card 04 §6) and to the Circuit Breaker/Policy Server split (Card 06 §11), each citing the reconciled reasoning from `review-reconciliation.md` (Decide/Act separation for the latter, executable-capability vs. procedural-knowledge governance semantics for the former). Relabeled Card 03 §4's LangGraph-specific passage as `Implementation Note (LangGraph)`, leaving the general ADK-comparison sentence as unscoped prose — preserves content, makes framework-coupling visible per the Synthesis's D3 resolution. Four cards bumped to Version 2.0 each (Card 02, 04, 06, 03) with individual changelog notes — no card bumped without an actual content change. Matrix's two Stage-4 rows updated to reflect completion. Stage 4 exit criteria met — these are now intentional, citable decisions rather than undocumented choices that could be mistaken for oversights in a future review. |
| 2026-06 | Ran Closure Plan Stage 3 (Resolve Factual Contradictions) | `docs/reference/01-architecture-taxonomy.md`, `docs/governance/architecture-traceability-matrix.md`, this plan | Checked the one live Stage 1 defect against the Traceability Matrix before fixing, per the matrix's governing rule. Card 01 §8's stale observability field list (still including "evaluation score," contradicting Card 05 §5's schema) replaced with a pure pointer to Card 05 §5 — implementing the Synthesis's A1 principle of one canonical definition, not duplicated restatement. Card 01 bumped to Version 2.0 with changelog note. Matrix's Observability Standard row updated to reflect the fix. Card 06 §26's citation of Card 03 §2-3 remains correctly out of scope for this stage — it requires a new Card 03 subsection (Stage 5) before the citation itself can be corrected, not a Stage 3 fix. Re-ran the "search all cards for 'Tier'" and "evaluation score" sanity checks per Stage 3's exit criteria — no other occurrences found. Stage 3 exit criteria met. |
| 2026-06 | Ran Closure Plan Stage 2 (Architecture Traceability Matrix) | `docs/governance/architecture-traceability-matrix.md`, this plan | Created the matrix, seeded directly from Stage 1's audit table — populated with every concept identified as having a single canonical owner (Risk Tiers, Agent/Tool/Memory/Skill Contracts, Memory Types, Evaluator Hierarchy, Formal Failure Taxonomy, Data Classification inheritance, HITL unification, etc.), each row carrying its canonical owner and every referencing card/section. Both Stage 1 defects (Card 01 §8's stale "evaluation score" field; Card 06 §26's citation of a not-yet-defined Card 03 mechanism) are carried into the matrix as explicitly flagged STALE rows with their scheduled fix stage noted, rather than silently dropped. Every Stage 4-9 pending deliverable from the Closure Plan's summary table (Alternative-Architecture notes, LangGraph relabeling, blast-radius statements, the four foundational files, Card 06 index, ADRs, verification spec) was pre-seeded as a row marked Pending with its stage, so nothing from the remaining plan can be lost track of. Directory structure section updated to reflect the matrix now exists. This matrix is now the required lookup tool for every fix from Stage 3 onward — no fix may be marked done without checking it here first and updating its "Last Verified" column after. |
| 2026-06 | Ran Closure Plan Stage 1 (Full Forward-Reference Audit) | this plan | Audited every "See Card X §Y" / "per Card X" cross-reference across all 7 cards against the three known findings plus an exhaustive sweep for additional staleness. Result: only 2 of the 3 tracked findings are still live defects. Card 01 §8's observability field list still includes "evaluation score," contradicting Card 05 §5's schema, which explicitly excludes it — carried forward to Stage 3. Card 06 §26's citation of Card 03 §2-3 for "source boundary preservation" is confirmed stale — no such mechanism exists in Card 03 §2-3 — carried forward to Stage 5, where the missing abstraction gets drafted before the citation is corrected. The third tracked finding, Card 07 §6's Tier 3/4 citation of Card 06 §3, is verified ALREADY CORRECT in the current file — it reads "Tier 4+ by default (Card 06 §3), or Tier 3 specifically where Card 06 §14's regulated-project override applies," matching Card 06 §3/§14 exactly. This corrects the prior entry's implication that this fix was still pending; no card edit was needed, only this log correction. No other cross-reference in the ecosystem was found stale during the exhaustive pass. Stage 1 exit criteria met — this audit table is the direct input to Stage 2.
| 2026-06 | Ran v1 cohesion review cycle (2 independent reviews + reconciliation + 10-stage closure plan); reorganized `docs/governance/` | `docs/governance/cohesion-reviews/v1/*`, `docs/governance/PHASE-0-CLOSURE-PLAN.md`, this plan | Conducted my own independent 4-part cohesion review against the frozen rubric, scoring all 14 dimensions with cited evidence — overall verdict Not-Yet-Freeze-Ready, driven by two confirmed factual contradictions (Card 07 §6 misquoting Card 06 §3's risk tiers; Card 01 §8's stale evaluation-score field contradicting Card 05 §5's fix) and three referenced-but-undrafted foundational files (AGENTS.md, agent-contract-template.md, model-routing-table.md). A separately-run ChatGPT 4-part review scored uniformly high (9.3-10/10 every dimension) and verdicted Freeze-Ready/Approved — direct comparison confirmed this was not a disagreement but a miss: ChatGPT's review did not catch either confirmed contradiction. A Review Board Synthesis document reconciled both reviews into Must-Do/Should-Do/Future/Reject categories; I responded formally in `review-reconciliation.md`, conceding two of my own findings (withdrew the Circuit-Breaker/Policy-Server merge recommendation entirely — Card 01 §2's existing Decide/Act split makes the separation correct; downgraded the Tool/Skill Registry merge from "should fix" to "should document why kept separate") and adopting two new artifacts neither original review proposed (Architecture Decision Records, a Traceability Matrix). Produced `PHASE-0-CLOSURE-PLAN.md` — a 10-stage sequential remediation plan (not independent patches) where each stage's output is a dependency for the next: audit → traceability matrix → fix contradictions → document reconciled decisions → operationalize the one genuine missing mechanism → close blast-radius gaps → draft the 4 missing foundational files (deliberately late, against a now-corrected base) → Card 06 index + ADRs → verification spec → final re-scored confirmation review. Reorganized `docs/governance/` into a versioned structure (`cohesion-reviews/v1/`) so future review cycles don't mix with this one. File 0.14 updated to `IN REVIEW` (Closure Plan Stages 1-9 remain). |
| 2026-06 | Approved Card 07; drafted, reviewed, and froze the Ecosystem Cohesion Review Rubric at v1.0 | `docs/reference/07-spec-driven-production.md`, `docs/governance/ecosystem-cohesion-review-rubric.md`, this plan | Card 07 (Spec-Driven Production) APPROVED after one senior-architecture-review pass (10/10 architecture, 10/10 cross-card cohesion) — three polish refinements applied: "code is disposable" reworded to "implementation is regenerable," a Specification Lifecycle added (§12, completing the lifecycle pattern every other artifact type already has), and fleet-governance rollout explicitly bound to §9's canary discipline. One real cross-card inconsistency caught and fixed during my own follow-up cohesion check: Card 07 §6 incorrectly claimed Tier 3+ requires human review by default, when Card 06's own table starts HITL at Tier 4 — corrected. Separately, commissioned and froze a 14-dimension Ecosystem Cohesion Review Rubric across three review rounds (10 dimensions → +4 production dimensions [Architectural Simplicity, Evolutionary Resilience, Blast Radius Analysis, Operational Validation Readiness] → 9 governance refinements [Confidence, Assumptions, Evidence Quality, Architectural Debt Register, Missing-vs-Incorrect split, Alternative-Architecture-Considered, framework-replacement scenario, partial-degradation testing, measurable pass criteria] → Reviewer Charter + mandatory freeze-gate rule + reviewer-independence/version declarations). This rubric was not part of the original Phase 0 plan — it emerged organically from the review discipline itself and is explicitly designed to be reusable beyond this project. New Phase 0 files added: 0.13 (the rubric itself, frozen) and 0.14 (the cohesion review cycle, not yet run). Phase 0's exit criteria updated to distinguish individual-card approval from ecosystem-level cohesion as two separate gates — sequencing corrected so Card 07 approval precedes rubric creation in the documented history, per explicit instruction. |
| 2026-06 | Drafted, reviewed, and revised Card 06 across three review cycles | `docs/reference/06-security.md`, this plan | Card 06 (Security & Governance) APPROVED after three senior-architecture-review passes — the most heavily scrutinized card in the ecosystem, appropriately given Projects 1 (healthcare) and 3 (financial) exposure. Round 1 ("brutal review") demanded 10 required + 7 recommended fixes: Policy Server fail-closed behavior by tier, Circuit Breaker state-ownership boundary (signals Harness, never mutates state directly), JIT token full lifecycle, registry runtime verification, instruction-artifact signing (AGENTS.md/contracts/Skills/prompts), A2A delegation semantics, tightened Tier 2/3 defaults for regulated projects, security event schema, clarified egress governance, and explicit Green Team definition. Round 2 pushed for true 10/10 — "who secures the security layer itself" — adding Policy Server integrity/change control (signed policies, Tier 5 dual control), classification inheritance (monotonic, derived artifacts inherit highest source classification), HITL reviewer separation-of-duties, hardened break-glass (time-boxed/auto-expiring/notified), full audit decision provenance, classification-aware model routing, the untrusted-context-boundary principle, expanded supply-chain scope, and Red Team finding governance. Round 3 was five precision patches: removed ambiguous §7a/§8a pseudo-references, added a regulated-project override footnote to the global risk-tier table, strengthened Tier 5 secondary verification from optional to required, protected the Tier 0 local fallback policy from becoming a bypass path, and distinguished automated initial classification from human-reviewed downgrade. This card discharges every outstanding forward-reference from Cards 01-05 in full. |
| 2026-06 | Drafted, reviewed, and revised Card 05 | `docs/reference/05-quality-evaluation.md`, this plan | Card 05 (Quality & Evaluation) APPROVED after one senior-architecture-review pass focused on cross-card consistency (per the review-strategy shift logged in the Card 04 entry below). Discharged two cross-card debts: closed the formal Failure Taxonomy gap flagged by both Card 01 and Card 03 (with subcode support and human-approval-failure subtypes added per review), and discharged Card 04's Skill Certification dependency by reusing the Evaluator Hierarchy. Fixed one genuine architectural leak caught by review: removed `evaluation score` from the required observability schema — observability and evaluation are now cleanly separated with a `run_id`/`trace_id` join key, satisfying Card 01 §8's binding requirement without ambiguity. Tightened HITL-unification wording to "unified at the infrastructure/mechanism level, not as one identical workflow" across all 5 trigger points (Cards 01, 02, 03, 04, 05). Added Card 06 forward-references for Safety pillar enforcement boundary and Agent-as-Judge trace-access redaction. Quality Flywheel's Improve step now targets named contract artifacts, consistent with the ecosystem's contract-driven pattern. |
| 2026-06 | Drafted, reviewed, and revised Card 04 | `docs/reference/04-skills-framework.md`, this plan | Card 04 (Skills Framework) APPROVED after one senior-architecture-review pass — strongest review yet (9.8/9.2). Answered all procedural-memory-to-Skill promotion questions from Card 03 explicitly (human-reviewed suggestion only, never autonomous). Added Skill Registry mirroring Card 02's Tool Registry, and a Skills-vs-prompt-templates boundary clarification. Confirmed the Agent→Tool→Memory→Skill contract pattern is now a consistent architectural signature across Cards 01-04. Deferred: skill dependencies, freshness tracking, deeper composition precedence (Phase 3); formal skill validation/certification flagged as a Card 05 dependency. Review strategy shifts for Cards 05-07: emphasis moves to cross-card consistency and hidden contradictions rather than missing content per-card. |
| 2026-06 | Drafted, reviewed, and revised Card 03; fixed Card 01 terminology | `docs/reference/03-context-memory.md`, `docs/reference/01-architecture-taxonomy.md`, this plan | Card 03 APPROVED after two senior-architecture-review passes. Added memory metadata (confidence/source/timestamp/verification_status), Retrieve→Rank→Filter→Compress→Inject retrieval pipeline, Consolidate→Validate→Store consolidation process, context budgeting, source precedence with regulated-domain exception, memory contracts, shared-history write-hygiene labels, and LLM-consolidation provenance requirement. Renamed "structured state" to "Structured Records" to avoid colliding with Card 01's `state` — Card 01 corrected to match. Tracked forward: context budget defaults and verification-status transitions (Phase 1), procedural-memory-to-Skill promotion rules (now a stated requirement for Card 04). |
| 2026-06 | Drafted, reviewed, and revised Cards 01 and 02 | `docs/reference/01-architecture-taxonomy.md`, `docs/reference/02-tools-mcp.md`, this plan | Both cards APPROVED after multiple senior-architecture-review passes. Card 01 elevated Agent Contract to a core component alongside Model/Tools/Orchestration and defined State; established the Runtime Stack governing structure for Cards 02-06. Card 02 added Tool Registry (with binding rule + assignment-authority clarification), Tool Lifecycle, versioning, MCP version pinning, and data minimization, while correctly deferring risk tiers/permissions/sandboxing/A2A trust to Card 06. New artifact added to Phase 0: `docs/architecture/agent-contract-template.md` (0.12). Deferred items flagged in Phase 4 (lifecycle transition criteria, tool certification tests). |
| — | — | — | — |
