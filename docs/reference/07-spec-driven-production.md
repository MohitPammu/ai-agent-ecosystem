# Reference Card 07 — Spec-Driven Production

**Card Version:** 1.0 (Approved)

**Source whitepapers:** Prototype to Production / AgentOps Lifecycle (2025 Day 5) · Spec-Driven Production Grade Development (2026 Day 5)
**Governing structure:** This is the final card in the Runtime Stack's knowledge base — it does not introduce a new layer, it defines **how every layer defined in Cards 01-06 actually gets built, shipped, and operated.** Where Cards 01-06 are the architecture, this card is the *production discipline* applied to building that architecture. It also closes the one remaining tracked item from Card 06 §26: provider approval metadata for `model-routing-table.md`.
**Purpose:** Defines Spec-Driven Development (SDD) as the build methodology for this entire ecosystem, BDD/Gherkin spec format, the four agent execution modes, the three-tier code review architecture, Context Hygiene, the "last mile" production gap, CI/CD for agents, safe rollout, and the AgentOps lifecycle that closes the loop back to Card 05's Quality Flywheel.

---

## 1. Spec-Driven Development (SDD) — The Build Methodology

**The core shift: implementation is regenerable; the spec is the source of truth.** This is not a new principle for this ecosystem — it's the formal name for what the Master Execution Plan has already been enforcing since Phase 0: every component gets a spec, reviewed and approved, *before* implementation begins (Working Principle #1). SDD is why that discipline exists, not an additional layer on top of it.

**Why this matters specifically for an agent ecosystem:** when an LLM generates implementation code, that code can be regenerated, refactored, or replaced cheaply — this is not a claim that code quality doesn't matter, only that a wrong spec propagates its error into every regeneration, while a wrong line of code is comparatively cheap to fix or replace. **Investing review rigor in the spec, not the code, is the correct allocation of human attention** — this is the same logic behind why the Master Plan demands one-file-at-a-time scrutiny (Working Principle #2) at the spec stage, not after code exists.

## 2. BDD/Gherkin — The Spec Format

Specs in `docs/specs/` (Phase 1+) follow Behavior-Driven Development syntax:

```
Scenario: <name>
  Given <initial context/state>
  When <action/event occurs>
  Then <expected outcome>
```

This is deliberately the format already named in the Master Execution Plan's Working Principles — this card is where that choice is formally justified: Given/When/Then eliminates ambiguity about what a component should do *before* anyone writes code against it, and is precise enough that Card 05's automated evaluation tier (outcome-checking) can verify a `Then` clause was actually satisfied.

**Format choice — hybrid Markdown + YAML, not pure JSON:** for specs with nested structure (a scenario with multiple steps, each step with parameters), hybrid Markdown+YAML parses more reliably for LLM-driven implementation than pure JSON for deeply nested content. Markdown carries the human-readable narrative (the Scenario/Given/When/Then prose); YAML frontmatter or fenced blocks carry structured parameters where precision matters (e.g., a Tool Contract's exact field types per Card 02 §4). This is the format every spec in `docs/specs/` should use going forward.

## 3. Four Agent Execution Modes

When delegating actual implementation work to an AI coding agent (Claude Code, Codex, or similar — per the zero-cost tooling already established), the agent operates in one of four modes, and which mode is active should be explicit, not ambient:

- **Architect** — designs the spec/structure, does not write implementation code. This is the mode for drafting `docs/specs/*.md` itself
- **Builder** — implements code against an *already-approved* spec. Never operates without an approved spec to build against — this enforces SDD (§1) at the tooling level, not just as a stated principle
- **Forensic Specialist** — investigates an existing failure (a Card 05 Inside-Out evaluation finding, or a Card 06 Red Team finding) without making changes — diagnosis only, separate from fixing
- **Librarian** — maintains and organizes existing documentation/specs without changing behavior (e.g., updating the Master Execution Plan's progress log, reorganizing `docs/reference/`)

**Direct implication:** every work session in this ecosystem's build (including the reference-card drafting we've been doing) maps to one of these modes. Architect mode is what's been happening for all 7 reference cards; Builder mode begins at Phase 1.

## 4. Context Hygiene — Preventing PII Leakage at the Spec/Code Level

A specific technique for keeping sensitive data out of specs, code, and any artifact that might be logged, shared, or fed back into a model: **use `[[VARIABLE]]` placeholder syntax for any value that would otherwise be sensitive** (e.g., a real patient ID, a real account number) when writing examples in specs, test fixtures, or documentation. The placeholder is resolved to a real value only at runtime, within the sandboxed execution environment (Card 06 §5) — never written into a spec, commit, or shared artifact as literal sensitive data.

**Direct implication for Projects 1 and 3:** any spec, test case, or documentation example involving healthcare claims or financial records uses `[[CLAIM_ID]]`, `[[PATIENT_REF]]`, `[[ACCOUNT_NUMBER]]`-style placeholders, never real or even realistic-looking sensitive values. This is a cheap, mechanical control that directly supports Card 06 §13's data classification discipline — a spec containing only placeholders carries no sensitive classification at all, regardless of what it describes.

## 5. The Policy Server Pattern, Applied to Specs

Card 06 §21 established the Policy Server as the single enforcement point separating security governance from execution logic. The same separation principle applies to specs: **a spec describes *what* a component does and *why* (the governance/intent layer); it should not embed *how* permission checks are implemented (the enforcement layer)**. A spec for a Tool, for example, states "this tool requires Tier 4 approval per Card 06 §3" — it does not re-implement the Policy Server's ABAC logic inline. This keeps specs stable even as the Policy Server's internal implementation evolves, and prevents the same enforcement logic from being redefined slightly differently in every spec that touches it.

## 6. Three-Tier Code Review

Mirroring Card 05's Evaluator Hierarchy (automated → LLM-as-Judge → Agent-as-Judge → HITL), code review for this ecosystem runs three tiers before anything merges:

1. **Automated** — linting, type checking, the CI workflow already running in the `ai-agent-ecosystem` GitHub repo (`.github/workflows/ci.yml`)
2. **AI review** — an LLM or Agent-as-Judge (Card 05 §4 tier 3) reviews the diff against the approved spec specifically — does the implementation actually satisfy the Given/When/Then clauses, not just "does it look reasonable"
3. **Human review** — final approval, required for anything reaching Tier 4+ by default (Card 06 §3), or Tier 3 specifically where Card 06 §14's regulated-project override applies (Projects 1/3, writes affecting downstream decisions) — consistent with the HITL discipline already established across Cards 01/03/04/05/06

This is not a new evaluation system — it's Card 05's Evaluator Hierarchy applied specifically to the code-review checkpoint, with the same escalation logic (cheap/automated first, human reserved for what needs it).

## 7. The "Last Mile" Gap — Why Most Effort Is Infrastructure, Not Intelligence

A grounding observation for scoping expectations honestly: in a real agentic system, roughly 80% of the engineering effort is infrastructure — the harness, observability, security, evaluation, deployment pipeline — and only about 20% is "the AI part" (the actual model calls and prompts). **This maps precisely onto why Cards 01-06 exist before any project-specific code does** — the Master Execution Plan's Phase 1-4 (core infrastructure, security, skills, MCP connectors) *is* the 80%, and Phases 5-7 (the three portfolio projects) are where the "20%" finally becomes visible as project-specific logic sitting on top of that infrastructure.

## 8. CI/CD for Agents

Extends standard CI/CD with agent-specific gates, layered onto the existing `.github/workflows/ci.yml`:

- **Spec-gate**: a PR implementing a component cannot merge without referencing its approved spec (§1) — enforced by convention now, by CI check once Phase 1 tooling exists
- **Golden-set regression**: Card 05 §4 tier 1's automated evaluation runs against a golden test set on every change — a regression here blocks merge, the same trend-detection logic Card 05 established
- **Security gate**: Card 06's Tier 4+ checks (signed instruction artifacts, registry verification) run as part of CI, not just at runtime — catching a tampered or unsigned artifact before it ships, not only when an agent tries to load it

## 9. Safe Rollout Strategy

Once a component is `BUILT` (per the Master Execution Plan's status flags), deployment itself follows a staged rollout, not an all-at-once switch:

- **Canary**: new version handles a small fraction of traffic/tasks, monitored via Card 05's observability before scaling up
- **Rollback readiness**: every deployed version is reversible — ties directly to Card 02's tool/skill versioning (`tool_version`/`skill_version`) and Card 06's signed-artifact versioning, both of which already make "what version was active" a well-defined question
- **Feature flagging at the Skill/Tool level**: a new Skill or Tool can be registered (Card 02/04's Registries) without immediately being set `Active` for all agents — enabling gradual exposure rather than a hard cutover

## 10. The AgentOps Lifecycle — Closing the Loop Back to Card 05

**Observe → Act → Evolve**, operating at the level of the whole ecosystem rather than a single evaluation run:

- **Observe**: Card 05's observability infrastructure plus Card 06's SecOps consumption of that same data (Card 06 §11) — the combined picture of what's happening
- **Act**: Card 05's Quality Flywheel's Improve step (a named, versioned contract/policy/prompt change) and Card 06's enforcement actions (circuit breaker, policy updates) — the combined response
- **Evolve**: the *systemic* version of the above two — not just fixing one component, but updating the Master Execution Plan itself (new Phase, new deferred item resolved, a reference card amended) when a pattern repeats across multiple Observe/Act cycles. This is the lifecycle the Master Plan's Progress Log has been informally running since Card 01 — every review cycle on these reference cards has been one Evolve turn on the *architecture itself*

**Fleet governance** (relevant once Phases 5-7 produce three running projects rather than one): when multiple agents/projects operate simultaneously, governance decisions (a Card 06 policy change, a Card 04 Skill update) need a defined propagation rule — does a core Skill update apply to all three projects immediately, or roll out per-project on the same canary logic as §9? This is flagged here as a Phase 5+ design question, not resolved now, since it's premature before more than one project exists.

## 11. Provider Approval Metadata — Closing Card 06 §26's Tracked Item

Card 06 §26 required classification-aware model routing but deferred the metadata schema to this card. `docs/architecture/model-routing-table.md` (Phase 0, file 0.9) must include, per provider entry:

| Field | Purpose |
|---|---|
| `provider_approved_for_classes` | Which Card 06 §13 data classifications this provider may legitimately receive |
| `retention_policy` | The provider's stated data retention behavior |
| `training_use_allowed` | Whether the provider may use submitted data for model training |
| `region` / `data_residency` | Where data is processed/stored |
| `zero_retention_available` | Whether a zero-data-retention tier/setting exists for this provider |
| `approval_status` | Whether this provider is currently approved for use in this ecosystem, and for which classification levels |

This table is what Card 06 §26's "classification-aware model routing" principle actually checks against at runtime — Ollama (local) trivially satisfies the most restrictive classifications since data never leaves the machine; Gemini Flash and GitHub Models must be evaluated against this table per classification level before being used for Projects 1 or 3's sensitive data specifically.

**Fleet-governance rollout discipline:** when this question is eventually addressed in Phase 5+, fleet-governance changes (a core Skill update, a Card 06 policy change propagating across multiple simultaneous projects) follow the **same canary and rollback discipline already defined in §9** — this is not a new mechanism to invent later, just an application of one already established here.

## 12. Specification Lifecycle

Every other governed artifact in this ecosystem has an explicit lifecycle — Tools (Card 02 §6), Skills (Card 04 §8), Memory contracts (Card 03 §11), security policies (Card 06 §21). Specs themselves complete this pattern with the same lightweight discipline:

**Draft → Review → Approved → Implemented → Verified → Deprecated → Archived**

This maps directly onto the Master Execution Plan's existing status flags (`NOT STARTED` → `IN REVIEW` → `APPROVED` → `BUILT`) with two additions at the end: **Verified** (Card 05's evaluation confirms the implementation actually satisfies the spec's Given/When/Then clauses — closing the loop §6 describes) and **Deprecated/Archived** (a spec superseded by a newer version, kept for audit history rather than deleted — consistent with Card 06 §16's registry versioning, which never overwrites old signed versions).

---

## Direct Implications for This Build

- Every spec in `docs/specs/` (Phase 1+) follows BDD/Gherkin syntax in hybrid Markdown+YAML format (§2) — this is now formally justified, not just stated as a working principle
- Implementation work maps explicitly to one of the four execution modes (§3); Builder mode never operates without an approved spec, mechanically enforcing SDD
- Any spec, test fixture, or example touching Projects 1/3's sensitive domains uses `[[VARIABLE]]` placeholder syntax (§4) — never literal sensitive-looking values, even as fictional examples
- Specs describe governance intent (e.g., "Tier 4 approval required") without re-implementing Card 06's Policy Server logic inline (§5) — one source of truth for enforcement mechanics
- Code review runs the same three-tier escalation as Card 05's Evaluator Hierarchy, applied specifically to spec-conformance checking (§6)
- `.github/workflows/ci.yml` gains agent-specific gates over time: spec-reference enforcement, golden-set regression, and security artifact verification (§8) — extending, not replacing, the existing CI workflow
- Deployment of any `BUILT` component follows canary rollout with rollback readiness, leveraging the versioning already established in Cards 02/04/06 (§9)
- The AgentOps Observe→Act→Evolve lifecycle (§10) is the operational frame for what the Master Plan's Progress Log has already been doing across all 6 prior review cycles — Phase 1+ automates the same loop on the running system
- `docs/architecture/model-routing-table.md` (Phase 0, file 0.9) must include the provider approval metadata schema in §11 before it can be considered complete — this closes Card 06 §26's tracked dependency
- Fleet governance propagation rules (§10) are explicitly deferred to Phase 5+, once more than one project exists to govern jointly — but already bound to follow §9's canary/rollback discipline rather than inventing a new mechanism when that phase arrives
- Every spec moves through the explicit Specification Lifecycle (§12: Draft→Review→Approved→Implemented→Verified→Deprecated→Archived), completing the same lifecycle pattern already established for Tools, Skills, Memory contracts, and security policies — mapped directly onto the Master Plan's existing status flags with `Verified` and `Deprecated/Archived` as the two additions beyond `BUILT`
- This card closes the Phase 0 knowledge base: all 7 reference cards now exist, and every cross-card forward-reference accumulated since Card 01 has been resolved somewhere in Cards 01-07
