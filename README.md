# AI Agent Ecosystem

A modular, production-grade multi-agent framework built to power three data science portfolio projects spanning healthcare, sports analytics, and fintech. Built on LangGraph with a zero-additional-cost design philosophy — every component runs on free-tier or existing-subscription infrastructure.

## Why This Exists

Most data science portfolios show isolated, one-off projects. This repo takes a different approach: build the reusable architecture once — orchestration, memory, security, evaluation, tool governance — then apply it across three distinct industries and ML paradigms. The framework is the artifact as much as any single project built on top of it.

This is also a deliberate exercise in **spec-driven, governance-first engineering practice**: every component is specified and reviewed before it's implemented, every architectural decision is documented and traceable, and nothing ships without a defined evaluation and security posture.

## Architecture Philosophy

The ecosystem is designed around a **Runtime Stack** — a strict dependency hierarchy that keeps every layer's responsibilities cleanly separated:

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
├── docs/
│   ├── reference/          # Distilled architecture reference cards (the knowledge base)
│   ├── architecture/       # Tech stack decisions, model routing, agent contract template
│   └── specs/              # BDD-style specs for each core component, written before code
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
- **Observability:** OpenTelemetry-style tracing
- **Tool protocol:** Model Context Protocol (MCP)
- **Language:** Python 3.12

Full rationale documented in `docs/architecture/tech-stack.md`.

## Build Status

This repository is under active, incremental development as part of a structured architecture build. See `00-MASTER-EXECUTION-PLAN.md` for the complete phase-by-phase roadmap, current status of every component, and a running progress log of design decisions.

## Author

Mohit Pammu — Data Analyst / Data Scientist in transition, building toward agentic systems and ML engineering roles.
[mohitpammu.github.io](https://mohitpammu.github.io) · [LinkedIn](https://linkedin.com/in/mohitpammu)

## License

MIT — see `LICENSE`.
