# Contributing

This repository is currently a solo-developed portfolio project under active, structured architecture development. It is not yet open for external code contributions, but the development process itself is intentionally transparent:

- **`00-MASTER-EXECUTION-PLAN.md`** — the full build roadmap, current status of every component, and a running progress log of architectural decisions
- **`docs/reference/`** — the distilled architecture reference cards every component is built against
- **`docs/specs/`** — BDD-style specifications written before any implementation, following a spec-driven development discipline

## Development Discipline

Every component in this repo follows the same process before being marked complete:

1. **Spec first** — a BDD-style specification (`Scenario / Given / When / Then`) is drafted and reviewed against the architecture reference cards
2. **Critical review** — specs undergo a structured architecture review pass before approval
3. **Implementation** — code is written only after its spec is approved
4. **Status tracking** — every file's status (`NOT STARTED` → `IN REVIEW` → `APPROVED` → `BUILT`) is tracked in the master execution plan, not left implicit

## Reporting Issues

If you spot an architectural inconsistency, a bug, or have a suggestion, feel free to open an issue describing:
- What you observed
- Which file/component it relates to
- Why it matters (architectural conflict, security gap, etc.)

## Questions

For questions about the project's design rationale, the master execution plan's progress log documents the reasoning behind most major decisions. Feel free to open an issue if something isn't clear from the docs.
