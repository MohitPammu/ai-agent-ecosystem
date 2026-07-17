# AI Agent Ecosystem

![License](https://img.shields.io/github/license/MohitPammu/ai-agent-ecosystem)
![Python](https://img.shields.io/badge/python-3.12-blue)
![Status](https://img.shields.io/badge/status-active%20development-yellow)
![CI](https://github.com/MohitPammu/ai-agent-ecosystem/actions/workflows/ci.yml/badge.svg)
![Last Commit](https://img.shields.io/github/last-commit/MohitPammu/ai-agent-ecosystem)

A modular, production-grade multi-agent framework built to power three data science portfolio projects spanning healthcare, sports analytics, and fintech. Built on LangGraph with a zero-additional-cost design philosophy — every component runs on free-tier or existing-subscription infrastructure.

## Why This Exists

Most data science portfolios show isolated, one-off projects. This repo takes a different approach: build the reusable architecture once — orchestration, memory, security, evaluation, tool governance — then apply it across three distinct industries and ML paradigms. The framework is the artifact as much as any single project built on top of it.

This is also a deliberate exercise in **spec-driven, governance-first engineering practice**: every component is specified and reviewed before it's implemented, every architectural decision is documented and traceable, and nothing ships without a defined evaluation and security posture.

## Architecture Philosophy

The ecosystem is designed around a **Runtime Stack** — an authority and ownership hierarchy that keeps every layer's responsibilities cleanly separated. It governs which layer owns and enforces each concern; runtime events and requests may flow bidirectionally through named interfaces:

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

Each layer is documented in a dedicated reference card (see `docs/reference/`), distilled from a structured review of current industry frameworks on agentic system design (MCP, A2A, SKILL.md, spec-driven development, agent evaluation, and security architecture). Every card has been through multiple rounds of senior-architecture-style critical review before being marked approved — see the build log in `00-MASTER-EXECUTION-PLAN.md`.

## What's Modular vs. What's Project-Specific

**Stays constant across every project** (`core/`, `skills/` infrastructure, `mcp-servers/`):
- Orchestrator and execution harness
- Memory system (episodic, semantic, procedural, preference, structured records)
- Evaluation and observability pipeline
- Security and governance layer (sandboxing, policy enforcement, risk tiering)
- Tool registry, contracts, and lifecycle management

**Swapped per project** (`projects/*/domain-skills/`, data connectors, model selection):
- Domain-specific SKILL.md files (e.g., Medicare claims logic vs. fantasy football scoring rules)
- Data source connectors (CMS Medicare data vs. ESPN/Sleeper APIs vs. financial market APIs)
- Problem framing and model choice (unsupervised anomaly detection vs. time-series forecasting vs. portfolio optimization)

## Portfolio Projects Built on This Framework

| Project | Industry | ML Approach | Repo |
|---|---|---|---|
| Healthcare Fraud Detection | Healthcare / Compliance | Unsupervised anomaly detection | [link once published] |
| Fantasy Football Intelligence | Sports Analytics | Time-series forecasting + recommendation | [link once published] |
| Financial Analysis Platform | Fintech | Forecasting + portfolio optimization | [link once published] |

Each project repo depends on this one for its core agent infrastructure and consumes the shared Skills/Tools layer, customizing only the domain-specific layer described above.

## Repository Structure

```
ai-agent-ecosystem/
├── AGENTS.md               # Master ecosystem specification — signed instruction artifact, v3.0 (Closure Plan Stage 7, Stage 10 patches)
├── docs/
│   ├── reference/          # Distilled architecture reference cards (the knowledge base) — all 7 APPROVED
│   ├── architecture/       # All 3 architecture artifacts complete (Stage 7): agent-contract-template.md, model-routing-table.md, tech-stack.md
│   ├── specs/              # BDD-style specs for core components, written before code (Phase 1+)
│   └── governance/         # Review/QA artifacts — emerged organically during Phase 0, not originally planned
│       ├── ecosystem-cohesion-review-rubric.md         # Frozen v1.0 — reusable beyond this project
│       ├── PHASE-0-CLOSURE-PLAN.md                     # Complete — all 10 stages done, Phase 0 frozen at v0.1.0-phase0-freeze
│       ├── architecture-traceability-matrix.md         # Concept → owner → references (Closure Plan Stage 2) — living document, updated every stage
│       ├── architecture-verification-specification.md  # Post-implementation test plan — 38 scenarios (Closure Plan Stage 9, complete, 1.0 Approved)
│       ├── adr/                                        # Architecture Decision Records — ADR-001 through ADR-007 (Closure Plan Stage 8, complete)
│       └── cohesion-reviews/v1/                        # Versioned historical review snapshot
│           ├── claude/                                 # 4-part independent review
│           ├── chatgpt/                                # 4-part independent review
│           ├── architecture-review-board-synthesis.md
│           ├── review-reconciliation.md
│           ├── stage-10-internal-rescore.md
│           ├── stage-10-external-rescore.md
│           └── stage-10-reconciliation.md
├── core/                   # The reusable "factory" — orchestrator, harness, memory, evaluation, observability, security
├── skills/                 # Reusable SKILL.md library shared across all projects
├── mcp-servers/            # MCP tool connectors (file system, database, APIs)
├── projects/               # The three portfolio project implementations
├── tests/                  # Test suites
└── 00-MASTER-EXECUTION-PLAN.md   # Full build roadmap, status tracking, and progress log
```

## Tech Stack

- **Orchestration:** LangGraph
- **Model layer:** Local inference via Ollama, Google Gemini Flash (free tier), GitHub Models (via existing Copilot subscription) — routed by task complexity, see `docs/architecture/model-routing-table.md`
- **Memory:** Local vector store (semantic memory) + PostgreSQL (structured records)
- **Observability:** OpenTelemetry + self-hosted Jaeger — see `docs/architecture/tech-stack.md` for full rationale
- **Tool protocol:** Model Context Protocol (MCP)
- **Language:** Python 3.12

Full rationale documented in `docs/architecture/tech-stack.md`.

## Build Status

**Phase 0 is frozen at tag `v0.1.0-phase0-freeze`** (commit `106a70f`). All 10 Closure Plan stages are done — the architecture has been reviewed, corrected, and verified by dual independent reviews (internal + cold-read external), producing a reconciled FREEZE-READY-WITH-MINOR-PATCHES verdict. All 5 pre-freeze patches have been applied and verified, followed by a Working Principle #8 pre-tag consistency sweep (7 commits) that corrected discrepancies the patches themselves had missed. Phase 1 (Core Infrastructure) begins next — building `core/harness/`, `core/memory/`, `core/security/`, `core/evaluation/`, and `core/observability/` against the 6 mandatory pre-build specs (S1-S6) before the first portfolio project implementation begins.

See `00-MASTER-EXECUTION-PLAN.md` for the complete phase-by-phase roadmap, current status of every component, and a running progress log of design decisions.

## Author

Mohit Pammu — Data Analyst / Data Scientist in transition, building toward agentic systems and ML engineering roles.
[mohitpammu.github.io](https://mohitpammu.github.io) · [LinkedIn](https://linkedin.com/in/mohitpammu)

## License

MIT — see `LICENSE`.
