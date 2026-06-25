# Reference Card 04 — Agent Skills Framework

**Source whitepaper:** Agent Skills (2026 Day 3)
**Governing structure:** Skills are the concrete implementation of **Procedural Memory** (Card 03 §5) — this card does not redefine memory types, it answers *how* procedural memory gets packaged, loaded, and governed. Skills sit conceptually between the Memory layer and the Tool layer in the Runtime Stack: a Skill can reference Tools (Card 02) but is not itself a Tool — a Tool performs an action, a Skill teaches an agent *how* to approach a class of problem, often by orchestrating several tool calls.
**Purpose:** Defines SKILL.md anatomy, progressive disclosure, the skill taxonomy, and — per the explicit requirement carried over from Card 03's review — the procedural-memory-to-Skill promotion lifecycle (creation, approval, versioning, agent authorship).

---

## 1. What a Skill Is (and Isn't)

A Skill is a folder containing a `SKILL.md` file (plus optional supporting files — scripts, templates, reference docs) that packages reusable procedural knowledge: "how things get done" for a recurring class of task. It is distinct from two things it's easily confused with:

- **Not an AGENTS.md** — AGENTS.md (Card 01, Phase 0 file 0.10) is global, always-loaded instruction for the *entire* ecosystem. A Skill is narrow, conditionally-loaded knowledge for *one* task category.
- **Not a Tool/MCP server (Card 02)** — a Tool is a callable function with a contract (input/output schema, etc.). A Skill is *instructional content* an agent reads to decide *how* to approach a problem, which often includes deciding *which* Tools to call and in what sequence. A Skill without any Tool references is still valid (e.g., a pure analysis methodology); a Tool without any Skill referencing it is also valid (e.g., a primitive utility used directly).

**Answering whether Skills are executable instructions or reference docs (carried over from Card 03's review):** both, depending on the skill — and this should be declared explicitly per-skill via a `skill_kind` field (§5):
- **Procedural skills** are closer to executable instructions — step-by-step methodology an agent follows (e.g., "how to run exploratory data analysis on a new dataset")
- **Reference skills** are closer to documentation — domain knowledge an agent consults but doesn't follow as a literal sequence (e.g., "Medicare claims terminology and common fraud indicators")

## 2. SKILL.md Anatomy

A Skill folder's `SKILL.md` has two parts:

- **YAML frontmatter** (always loaded, lightweight): `name`, `description` — this is the part the orchestrator sees when deciding whether a skill is relevant to the current task. Keep this tight; it is the *only* part loaded by default.
- **Markdown body** (loaded only when triggered): the full procedural content — methodology, steps, examples, caveats. Can reference additional files in the same folder (scripts, templates, sample data) that are themselves only loaded if the body references them.

This two-tier structure is **Progressive Disclosure** — the same principle from Card 03's context budgeting (§9): don't pay the token cost of a skill's full content until the task actually needs it.

## 3. Progressive Disclosure — The Loading Mechanism

Three loading tiers, increasing in cost:

1. **Metadata always in context** — every registered skill's `name` + `description` is available to the orchestrator for relevance-matching, at near-zero token cost
2. **Body loaded on trigger** — once a skill is selected as relevant, its full markdown body enters context
3. **Supporting files loaded on reference** — scripts/templates/data files inside the skill folder are only pulled in if the body's content explicitly calls for them during that specific task

**Direct implication:** the Skill Loader (`skills/` infrastructure, Phase 3) must implement all three tiers — never eagerly load every skill's full body into context "just in case." This is a direct, mechanical answer to Card 03's context budget concerns (§9) as applied specifically to procedural memory.

## 4. Skill Taxonomy — Two Axes

**Axis 1 — Kind** (per §1's answer to the executable-vs-reference question):
- `procedural` — step-by-step methodology
- `reference` — domain knowledge consulted, not followed sequentially

**Axis 2 — Scope** (where the skill lives and who uses it):
- **Core/shared skills** (`skills/` at repo root) — generic, reusable across all three projects: `data-ingestion`, `eda`, `feature-engineering`, `modeling`, `evaluation`, `reporting` (Phase 3 of the Master Plan)
- **Domain skills** (`projects/*/domain-skills/`) — project-specific knowledge: Medicare claims terminology, fantasy football scoring rules, financial technical indicators (Phases 5-7)

This scope split is the direct mechanism behind the modularity goal stated at the top of the Master Plan: the same core skills get reused project to project; only domain skills get swapped.

## 5. The Skill Contract — Completing the Contract Pattern

Consistent with Agent Contracts (Card 01) and Tool/Memory Contracts (Cards 02/03), every Skill declares:

| Field | Purpose |
|---|---|
| **skill_id** | Unique, stable identifier (same anti-shadowing rationale as Card 02 §5's Tool ID rule) |
| **skill_kind** | `procedural` or `reference` (§1) |
| **scope** | `core` or `domain` (§4), and if domain, which project |
| **skill_version** | Semantic version |
| **status** | Lifecycle stage — see §6 |
| **owner** | Who authored/maintains it (us, for now) |
| **trigger_description** | The frontmatter `description` — what makes the orchestrator select this skill |
| **promoted_from** | If this skill originated from consolidated episodic/procedural memory (Card 03 §8) rather than being authored directly, a reference to that origin — answers the "when does memory become a Skill" question below |

## 6. Skill Registry — Discovery Governance, Mirroring Card 02

Consistent with the Tool Registry pattern (Card 02 §5), Skills are discovered through a central registry, not by an agent freely scanning the `skills/` filesystem at will. The same binding-rule discipline applies:

> A skill is loadable only if it is (1) present in the Skill Registry, (2) in `Active` lifecycle state, and (3) within scope for the requesting context (a `domain` skill for Project 1 is not loadable while operating on Project 2).

Registry entry fields (mirroring Card 02 §5's Tool Registry table):

| Field | Purpose |
|---|---|
| **skill_id** | Unique, stable identifier |
| **version** | Current `skill_version` in production |
| **owner** | Responsible party |
| **status** | Current lifecycle stage (§7) |
| **scope** | `core` or `domain` + project, if applicable |
| **trigger_metadata** | The frontmatter `description` used for relevance-matching |
| **location** | Path to the skill folder |

**Implementation note:** same as Card 02's Tool Registry — a structured `skills-registry.yaml` is sufficient at our scale; the discipline of treating it as the sole discovery mechanism is what matters, not the storage format.

## 7. Skills vs. Prompt Templates — Boundary Clarification

A Skill is distinct from prompt templates, runbooks, policies, or checklists that might superficially look similar as markdown files:

> A Skill is executable procedural (or reference) knowledge specifically packaged for **dynamic runtime loading** into an agent's context via the Progressive Disclosure mechanism (§3) — selected by the orchestrator based on task relevance. A prompt template, runbook, or policy document is typically **statically embedded** into a system prompt or referenced by a human, not dynamically discovered and loaded by an agent at runtime.

If a piece of content is always loaded regardless of task (e.g., core system instructions), it belongs in AGENTS.md or the agent's static system prompt — not packaged as a Skill. If it's conditionally relevant and benefits from being loaded only when needed, it's a Skill candidate.

## 8. Skill Lifecycle and the Procedural-Memory-to-Skill Promotion Question

Mirroring Card 02's Tool Lifecycle, with one addition specific to how Skills can originate from memory:

**Proposed → Drafted → Reviewed → Active → Deprecated → Retired**

**Two paths to `Proposed`:**
- **Authored directly** — we deliberately write a new Skill because we know a task category needs one (most skills in Phase 3 and per-project domain skills will start here)
- **Promoted from procedural memory** — Card 03 §8's memory consolidation process notices a workflow pattern repeating across episodic memory and flags it as a *candidate* for promotion to a formal Skill. This is explicitly a **suggestion, never an automatic action** — promotion always requires a human review step before reaching `Drafted`.

**Answering the remaining questions raised in Card 03's review explicitly:**
- *Who approves a new or promoted skill?* — human review at the `Reviewed → Active` gate, same discipline as Card 02's `Security Reviewed → Active` gate for tools. No skill reaches `Active` without this, and per §6's binding rule, no skill is loadable until it's both `Active` *and* registered.
- *Can agents create or modify skills themselves?* — **No, not autonomously.** Agents may *flag* a candidate (the promotion-suggestion path above), but actually drafting, editing, or activating a Skill is a human-initiated action. This is consistent with Card 01 §8's explicit Level 4 prohibition (no autonomous capability creation) — a Skill is a capability, and self-authored capabilities are exactly what that prohibition exists to prevent.
- *Are skills versioned?* — yes, via `skill_version` in the contract (§5), same discipline as Tool versioning (Card 02 §4).

## 9. Composing Skills

A single complex task may require multiple skills loaded together (e.g., `data-ingestion` + a domain skill like `medicare-claims-terminology`). Composition rules:

- Skills are additive, not hierarchical — there's no "parent skill" calling "child skills" as a control-flow mechanism (that's the Orchestrator's job per Card 01 §6, not the Skill system's)
- Loading order matters for context construction: core/shared skills load before domain skills, since domain skills often assume the general methodology is already in context and only need to add the specific terminology/rules layer on top
- If two loaded skills conflict (e.g., a core `evaluation` skill's generic methodology vs. a domain skill's project-specific evaluation criteria), the domain skill's guidance takes precedence for that specific task — general methodology, specific override, same pattern as software config layering

---

## Direct Implications for This Build

- `skills/` (Phase 3) implements the 6 core/shared skills as `procedural` or `reference` kind per skill, each declaring the full Skill Contract (§5)
- A central **Skill Registry** (`core/skills-registry.yaml` or similar, mirroring Card 02's Tool Registry exactly) is the sole discovery mechanism — a skill is loadable only if registered, `Active`, and in-scope for the current project context (§6's binding rule)
- The Skill Loader implements all three Progressive Disclosure tiers (§3) — metadata-only by default, body on trigger, supporting files only on explicit reference
- `projects/*/domain-skills/` (Phases 5-7) follow the same contract, registry, and lifecycle, scoped as `domain` rather than `core`
- No skill reaches `Active` without passing human review at the `Reviewed → Active` gate — mirrors Card 02's tool lifecycle discipline exactly
- Promotion from consolidated procedural memory (Card 03 §8) is a **suggested candidate only** — never an automatic write to `skills/` or the Registry; this directly satisfies Card 01 §8's Level 4 prohibition on autonomous capability creation
- Skills are explicitly distinguished from prompt templates/runbooks/policies (§7) — static, always-loaded content belongs in AGENTS.md or the system prompt, not packaged as a Skill
- Composition follows core-before-domain loading order, with domain-skill guidance taking precedence over core-skill guidance when the two conflict on a specific task
- Skill IDs are globally unique and stable, same anti-shadowing rationale as Card 02's Tool Registry
- **Explicitly deferred, tracked in the Master Plan rather than built into this card:** skill dependency declarations (`depends_on`/`conflicts_with`), skill freshness tracking (`last_reviewed`/`validation_date`), formal skill validation/certification (likely Card 05's territory), and deeper composition precedence layers (Global→Core→Project→Task→Session) beyond the current Core→Domain split
