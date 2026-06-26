# Reference Card 05 — Quality & Evaluation

**Card Version:** 2.0
**Changelog:** §5 — added "Blast radius — Evaluation-service unavailability" statement (non-blocking backfill model, Tier 4-5 HITL exception deferring to Card 06 §20). Closure Plan Stage 6.

**Source whitepaper:** Agent Quality (2025 Day 4, updated May 2026)
**Governing structure:** Occupies the "Evaluation & Observability" position in the Runtime Stack — sits above Tools, below Security. Inherits two binding obligations from earlier cards, both discharged in this card: (1) Card 01 §8 requires Observability and Evaluation kept as clearly separated disciplines, not conflated; (2) Card 01 and Card 03 both flagged a missing **failure taxonomy** as a gap to close here.
**Purpose:** Defines the Four Pillars of Agent Quality, the Outside-In/Inside-Out evaluation hierarchy, the evaluator hierarchy (automated metrics → LLM-as-Judge → Agent-as-Judge → HITL), the three pillars of Observability kept distinct from evaluation, the formal failure taxonomy, and the Quality Flywheel that ties it all into a continuous loop.

---

## 1. Why Traditional QA Fails for Agents

Traditional software testing verifies: "did we build the product right?" against a fixed spec — deterministic, repeatable, and failure is explicit (a crash, a wrong calculation). Agents fail differently: the system keeps running, tool calls return success, the output looks plausible — but it's wrong. The shift required is from **verification** (checking against a spec) to **validation** (judging real-world value against intent). This is why Card 01 §2's Validate/Decide stage exists in the execution loop itself, not just as an external test suite — validation has to happen at runtime, not only in CI.

## 2. The Four Pillars of Agent Quality

Every agent we build is evaluated against all four, not just task success:

- **Effectiveness (Goal Achievement)** — did the agent actually achieve the user's intent, not just produce *an* output? (e.g., not "did it write code" but "did the code produce the correct insight")
- **Efficiency (Operational Cost)** — total tokens, latency, and trajectory complexity (number of steps, retries, self-corrections). An agent that succeeds after 5 failed tool calls is lower quality than one that succeeds cleanly — this ties directly to Card 03's context/token budgeting and our zero-cost model-routing goal
- **Robustness (Reliability)** — does the agent fail gracefully under adversity (API timeouts, ambiguous input, missing data)? Does it retry appropriately, ask for clarification, or report what it couldn't do — rather than crashing or hallucinating a recovery?
- **Safety & Alignment (Trustworthiness)** — the non-negotiable gate. Stays within ethical/operational boundaries, resists prompt injection and data leakage. This pillar is the evaluation-side counterpart to Card 06's security controls — Card 06 builds the guardrails, this pillar measures whether they held. **Evaluation may detect safety/alignment failures, but permissioning and enforcement remain Card 06's responsibility** — this card observes and reports, it does not authorize or block risky behavior itself

## 3. The Outside-In / Inside-Out Evaluation Hierarchy

A strategic, two-stage process — start with the black box, then open it only if needed:

**Outside-In (the "black box"):** the first question is always "did the agent achieve the user's goal?" Measured via Task Success Rate, completeness, and (for interactive contexts) user satisfaction. **If this stage passes, evaluation can stop here for that case.** Most cases should pass at this stage; opening the box on every single run is wasteful.

**Inside-Out (the "glass box"):** only triggered when Outside-In reveals a failure. Systematically inspects the trajectory across distinct failure surfaces:
1. **LLM Planning ("the Thought")** — hallucination, off-topic reasoning, context pollution, repetitive loops
2. **Tool usage (selection & parameterization)** — wrong tool, missing necessary call, hallucinated tool/parameter names, malformed arguments (direct tie to Card 02's Tool Contract input schema — a violation here is a contract violation)
3. **Tool response interpretation ("the Observation")** — misreading numeric results, failing to extract key entities, critically: not recognizing a tool's own error state and proceeding as if it succeeded
4. **Retrieval/memory performance** — irrelevant or stale retrieval, or ignoring retrieved context and hallucinating anyway (ties to Card 03 §7's retrieval pipeline)
5. **Trajectory efficiency and robustness** — excessive steps, redundant calls, unhandled exceptions
6. **Multi-agent dynamics** (where applicable) — miscommunication, role conflicts, communication loops between agents

This hierarchy is the direct implementation target for `core/evaluation/` (Phase 1) — it is not a separate testing afterthought, it's the architecture of the evaluation harness itself.

## 4. The Evaluator Hierarchy — Escalating Cost and Nuance

Four tiers, used in this order — cheapest/most scalable first, escalating only when needed:

1. **Automated metrics** (cheapest, first CI/CD gate) — string/embedding similarity, task-specific benchmarks. Shallow but excellent as a **trend indicator**: not "is 0.8 a good score" but "did this commit drop our golden-set average from 0.8 to 0.6" — that drop is the signal, not the absolute number
2. **LLM-as-Judge** — a strong model scores another agent's output against a rubric. **Prefer pairwise comparison over absolute scoring** (run old vs. new version, ask the judge which is better, calculate win/loss/tie rate) — far more reliable than noisy 1-5 absolute scores
3. **Agent-as-Judge** — a dedicated "Critic Agent" evaluates another agent's full execution *trace*, not just its output — assessing plan quality, tool selection correctness, and context handling. This is how process failures get caught even when the final output looks fine. **Trace access for Agent-as-Judge evaluators must follow Card 06's data access and redaction policies** — a trace can contain sensitive content (PII, claims data, financial records), and an evaluator agent reading it unfiltered would be a backdoor around the access controls Card 06 establishes elsewhere
4. **Human-in-the-Loop (HITL)** — reserved for deep subjectivity, complex domain judgment, and as the final authority on anything automated tiers flag as ambiguous or high-risk. **HITL is unified at the infrastructure/mechanism level only** — one shared review/escalation pathway and audit trail — **not as a single identical workflow.** Each trigger point (runtime execution escalation per Card 01 §6, tool activation review per Card 02, memory contradiction review per Card 03 §8, skill activation review per Card 04 §6/8, and this card's evaluation-ambiguity tier) defines its own reviewer role, rubric, and approval criteria, while sharing the same underlying mechanism and audit model.

**Cost discipline, tying to our zero-cost goal:** route as much evaluation volume as possible to tier 1, escalate sparingly to tiers 2-3, and reserve tier 4 (the only tier with no cheap substitute) for what genuinely needs it.

## 5. Observability — Kept Strictly Separate From Evaluation

Per Card 01 §8's binding requirement: **Observability answers "what happened"; Evaluation answers "was it good."** They are different disciplines using different tooling, and this card does not conflate them. The Three Pillars of Observability are the data-capture layer evaluation *consumes*, not evaluation itself:

- **Logging** — discrete, timestamped records of individual events (a tool call, an error, a state transition)
- **Tracing** — the connected, end-to-end view of a single request's full trajectory across every step — this is what makes the Inside-Out hierarchy (§3) actually inspectable; without tracing, "open the box" has nothing to open
- **Metrics** — aggregated, numerical signals over time (latency percentiles, token cost per session, error rates) — this is what feeds tier-1 automated evaluation's trend-detection in §4

**Required observability fields per execution** (the concrete schema, satisfying Card 01 §8's forward reference): Run ID, Agent ID, Mission, tools used (with args/outcomes per Card 02's observability metadata field), cost, latency, errors, state transitions, trace/span IDs, and human intervention *events* (that a HITL event occurred — not its outcome or rubric). **Evaluation outputs — scores, rubric judgments, win/loss/tie comparisons, pass/fail decisions — are explicitly NOT observability fields.** They are stored in separate evaluation records, joined back to the observability trace by `run_id`/`trace_id`. This is a real boundary, not a stylistic one: observability captures what happened; evaluation captures a judgment about it, and the two must remain separately queryable.

**Direct implication:** `core/observability/` (Phase 1) implements logging/tracing/metrics as infrastructure; `core/evaluation/` (also Phase 1) is a *consumer* of that infrastructure, not a reimplementation of it. The two are separate modules with a one-directional dependency (evaluation reads observability data via the join key; observability has no dependency on evaluation, and never stores evaluation artifacts directly).

**Blast radius — Evaluation-service unavailability:** the Harness never blocks on Evaluation to complete a turn — Evaluation is a downstream consumer of observability data, not a gate in the execution loop. If `core/evaluation/` is unavailable, traces and observability records continue being captured normally (Observability has no dependency on Evaluation, per the one-directional dependency above); evaluation scoring simply backfills once the service recovers, processing the accumulated trace backlog. The one exception: Tier 4-5 actions requiring HITL escalation specifically triggered by a Safety/Alignment evaluation flag (Card 06 §9) — if Evaluation is down, that signal cannot fire, so Card 06's Policy Server fail-closed table (§20) governs those tiers regardless, independent of Evaluation's availability.

## 6. Formal Failure Taxonomy

This closes the gap explicitly flagged by both Card 01 and Card 03. Every failure in the ecosystem is classified into one of these categories — this is what gives the Decide stage (Card 01 §2) something concrete to branch on, rather than an undifferentiated "something went wrong":

| Category | Example |
|---|---|
| **Tool failure** | API timeout, malformed response, tool unavailable |
| **Model failure** | Hallucination, reasoning loop, off-topic drift |
| **Data failure** | Missing field, corrupted input, schema mismatch |
| **Validation failure** | Output fails the Validate stage's success criteria |
| **Permission failure** | Tool/skill binding rejected (Card 02 §5 / Card 04 §6 binding rules) |
| **Timeout failure** | Step or full trajectory exceeds time budget |
| **Human-approval failure** | Subtypes: rejected / no response / approval expired / wrong reviewer assigned / review system unavailable — these are operationally distinct and should not be collapsed into one undifferentiated code |

**Subcodes:** each top-level category above is a stable, shared classification — but real implementations will need finer-grained subcodes (e.g., Tool Failure → timeout/malformed_response/unavailable/dependency_failure/rate_limit). **Subcodes are implementation detail and may vary; every subcode must roll up to one of the top-level categories in this table**, so cross-ecosystem error handling and reporting stays consistent even as implementation-level detail grows.

**Direct implication:** Card 02's Tool Contract error categories (§4 of that card) and any future error handling across the ecosystem consume *this* taxonomy rather than inventing per-component categories — this is the single shared vocabulary failures get classified into, ecosystem-wide.

## 7. Skill Certification — Discharging Card 04's Dependency

Card 04 flagged that Skills have lifecycle review but no explicit *validation* process, and noted this likely belongs here. Skill Certification, occurring at the `Tested → Certified` gate (mirroring Card 02's tool lifecycle and Card 04 §8's skill lifecycle), consists of:

- **Example validation** — does the skill produce correct guidance against known worked examples?
- **Regression scenarios** — does a new version of a skill still pass cases the old version handled correctly?
- **Reference outputs** — comparison against a golden/expected output set (same mechanism as the Outside-In hierarchy's task success rate)
- **Behavioral verification** — for `procedural` skills (Card 04 §1), does following the steps actually produce the intended workflow outcome, not just plausible-sounding instructions?

This reuses the same Evaluator Hierarchy (§4) rather than inventing a skill-specific evaluation mechanism — automated checks first, LLM-as-Judge for nuance, human review as the final gate before `Certified`.

## 8. The Quality Flywheel — Closing the Loop

Evaluation data isn't valuable sitting in logs — it has to feed back into improvement. The flywheel: **Observe (capture via the 3 pillars) → Evaluate (Outside-In/Inside-Out + evaluator hierarchy) → Diagnose (classify via the failure taxonomy) → Improve → repeat.** Consistent with this ecosystem's contract-driven discipline (Agent/Tool/Memory/Skill contracts), the Improve step targets a specific contract or policy artifact rather than vaguely "fixing the agent" — concretely: the agent contract, a tool contract, a memory contract, a skill contract, the prompt/system instruction, the model routing policy, an evaluator rubric, or a security policy. Naming the artifact being changed keeps the flywheel's output traceable and versioned, the same way every other component in this ecosystem is. This loop is what the Progress Log in the Master Execution Plan is informally already doing at the architecture level (each review cycle = one flywheel turn on the *documentation*); once `core/evaluation/` is built, the same loop runs continuously on the *running system*.

---

## Direct Implications for This Build

- `core/evaluation/` (Phase 1) implements the Outside-In/Inside-Out hierarchy (§3) as its core architecture, not as a bolt-on test suite
- The Evaluator Hierarchy (§4) governs which tier handles which volume of evaluation — tier 1 (automated) as the default CI/CD gate, escalating to tiers 2-4 only as needed, consistent with the zero-cost model-routing goal
- `core/observability/` and `core/evaluation/` are built as **separate modules with a one-directional dependency** (evaluation reads observability data via a `run_id`/`trace_id` join key; observability has no dependency on evaluation, and never stores evaluation artifacts directly) — satisfies Card 01 §8's binding requirement with no schema-level leak
- The required observability fields in §5 explicitly exclude evaluation scores/judgments — those live in separate evaluation records, joined back by run/trace ID
- The formal Failure Taxonomy (§6) is the shared vocabulary every error-handling path in the ecosystem uses — Card 02's tool error contracts consume it directly, as does the Decide stage of Card 01's execution loop. Subcodes are permitted per-category but must roll up to these top-level categories; Human-approval failure in particular is treated as a category with distinct subtypes (rejected/no-response/expired/wrong-reviewer/system-unavailable), not one undifferentiated code
- Skill Certification (§7) discharges Card 04's open dependency, reusing the Evaluator Hierarchy rather than building a separate skill-specific evaluation system
- HITL is unified **at the infrastructure/mechanism level only** — shared review/escalation pathway and audit trail across Card 01 §6, Card 02, Card 03 §8, Card 04 §6/8, and this card's tier-4 — while reviewer roles, rubrics, and approval criteria remain trigger-specific, not a single identical workflow
- Safety & Alignment evaluation (Pillar 4) detects and reports failures; it does not authorize or enforce — that boundary stays with Card 06
- Agent-as-Judge trace access (§4 tier 3) is explicitly subject to Card 06's data access and redaction policies — evaluation must never become an unfiltered backdoor into sensitive trace content
- The Quality Flywheel (§8) targets a named, versioned artifact at its Improve step (agent/tool/memory/skill contract, prompt, routing policy, evaluator rubric, or security policy) — consistent with the ecosystem's contract-driven discipline rather than vague "fix the agent" framing
