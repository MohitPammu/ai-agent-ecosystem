# Reference Card 02 — Tools, MCP & Interoperability

**Source whitepapers:** Agent Tools & Interoperability (2025 Day 2, full depth) · Agent Tools & Interoperability (2026 Day 2, MCP architecture)
**Governing structure:** This card occupies the "Tool Layer" position in the Master Execution Plan's Runtime Stack (sits below Memory, above Evaluation/Observability and Security/Governance). Because Security sits *below* Tools in that stack, Security governs Tools — not the reverse. This card therefore defines tool mechanics, contracts, registry, and lifecycle; it deliberately does not define risk tiers, permission/scope enforcement, or sandboxing — those are Card 06's authority. It does not redefine Contract, Harness, State, or Memory either — see Card 01.
**Purpose:** Defines what makes a tool well-designed, the MCP standard for connecting agents to tools, the A2A protocol for agent-to-agent communication, the tool contract specification referenced from Card 01 §8, and the honest security gaps in MCP's current ecosystem.

---

## 1. The NxM Problem — Why Standards Exist

Without a shared standard, every agent framework needs custom integration code for every tool/data source it connects to — N agents × M tools = N×M custom integrations. MCP (Model Context Protocol) and A2A (Agent-to-Agent) solve this by giving both sides a common interface: write one MCP server for a tool, and *any* MCP-compatible agent can use it; expose one A2A-compatible agent, and *any* A2A-compatible agent can collaborate with it. This is the same problem USB solved for hardware peripherals.

## 2. MCP Architecture

Three roles:

- **Host** — the application the user interacts with (e.g., our orchestrator, or Claude Desktop)
- **Client** — the connector living inside the host that speaks MCP to a server
- **Server** — exposes a specific tool/data source's capabilities (e.g., our PostgreSQL connector, file system connector)

Communication runs over **JSON-RPC** — a lightweight, language-agnostic message format. This is why an MCP server can be written in Python while the host is in TypeScript, or vice versa; the protocol, not the language, is the contract.

**Direct implication:** `mcp-servers/file-system-connector/`, `mcp-servers/postgres-connector/`, and `mcp-servers/api-connectors/` (Phase 4) are each independent MCP servers exposing a narrow, well-defined capability. The orchestrator is the host; LangGraph's MCP client integration is the client role.

**Server vs. tool — not the same unit:** one MCP server can expose multiple distinct tools (e.g., the PostgreSQL connector might expose both a `read_claims_table` tool and a `read_provider_table` tool). Each exposed tool gets its own Tool Contract (§4) and its own Registry entry (§5) and moves through its own Lifecycle (§6) independently — a server can remain `Active` while one specific tool it exposes is `Deprecated`. Registry and lifecycle tracking therefore applies at the tool-capability level, not merely at the server level.

## 3. Six Tool Design Principles

Good tool design determines whether an agent uses a tool correctly or hallucinates its way through it. Apply these to every tool we build:

1. **Narrow, single-purpose tools** — a tool that does one thing well is easier for the model to select correctly than a multi-purpose tool with branching behavior
2. **Descriptive names and parameters** — the tool's name and parameter names *are* the documentation the model reasons from; vague names produce vague tool use
3. **Return structured, parseable output** — JSON/typed responses, not free-form text the model has to re-parse
4. **Fail loudly and specifically** — a tool that fails silently or returns an ambiguous error causes the agent to either hallucinate a recovery or loop uselessly. Errors should be specific enough that the Validate stage (Card 01 §2) can act on them
5. **Idempotency where possible** — a tool safe to call twice with the same input is far easier to retry safely; tools with side effects (writes, sends) should be explicitly marked as non-idempotent so the retry policy (below) treats them differently
6. **Right-sized scope of access** — a tool should expose only the capability it needs (e.g., a "read claims table" tool, not a generic "execute arbitrary SQL" tool) — this is also a security principle, formalized fully in Card 06
7. **Data minimization** — a tool returns only the minimum data required for the current task, not a full record dump because it was convenient to implement that way. This reduces what ends up in the agent's context (cost) and what could leak if that context is ever exposed (security) — direct complement to principle 6, but specifically about output shape rather than access scope

## 4. The Tool Contract — Specification Every Tool Must Define

This is the formal answer to Card 01 §8's forward reference. Every tool in `mcp-servers/` must define:

| Field | Purpose |
|---|---|
| **Input schema** | Typed parameters, required vs. optional, validation rules |
| **Output schema** | Typed return shape — what a successful call looks like |
| **Timeout behavior** | How long the agent waits before treating the call as failed |
| **Error contract** | What error types/codes are possible and what they mean (feeds Validate stage). Error categories should eventually align with a platform-wide failure taxonomy rather than being invented per-tool — see the Master Execution Plan's note that Card 05/06 owns formal failure taxonomy design; this card's error contracts should consume that taxonomy once it exists, not diverge from it |
| **Retry policy** | Is this tool safe to retry automatically? How many times, with what backoff? |
| **Idempotency expectation** | Explicitly marked idempotent or non-idempotent (per principle 5 above) |
| **Observability metadata** | What gets logged on every call — tool name, args (redacted if sensitive), duration, outcome (feeds Card 05's observability standard) |
| **tool_version** | Semantic version of this specific tool |
| **contract_version** | Version of the contract schema itself (this table), independent of the tool's own version |
| **breaking_change** | Boolean/flag — does this version break callers relying on the prior contract? |
| **deprecated_after / supported_until** | Lifecycle dates — see §6 below |
| **risk_tier** | Reference only — the tool's assigned tier from Card 06's risk classification. This card does not define the tiers, only requires every tool to declare which one it carries |
| **owner / registry_reference** | Either the responsible owner directly, or a pointer to this tool's Registry entry (§5) where ownership lives — since contracts are often reviewed independently of the Registry, this link must not be assumed implicit |

**This table is the spec every MCP server build (Phase 4) and every domain-specific tool (Phases 5-7) must satisfy before being marked `BUILT` in the Master Execution Plan.**

## 5. Tool Registry — Discovery Is Not Authorization

A critical governance rule: **agents discover approved tools through a central Tool Registry, not through arbitrary/open MCP discovery.** Open discovery is exactly the surface that enables tool shadowing (§6) — if any agent can find and bind to any MCP server that announces itself, a malicious server can simply announce itself convincingly. The Registry is the single approved source of truth for "what tools exist and which agents may use them."

Registry entry fields (one row per tool, living alongside the Tool Contract):

| Field | Purpose |
|---|---|
| **Tool ID** | Unique, stable identifier. **Tool IDs must be globally unique and stable — display names are never authoritative identifiers**, since confusable/duplicate names are a direct tool-shadowing vector |
| **Name** | Human-readable |
| **Version** | Current `tool_version` in production |
| **Owner** | Who is responsible for this tool (us, for now — but the field exists for when this scales) |
| **Status** | Current lifecycle stage — see §6 |
| **Allowed Agents** | Which agents/agent-roles may bind to this tool — the actual permission enforcement mechanics (scopes, approval requirements) are Card 06's job; the Registry just records *that* a restriction exists and what agents are cleared |
| **Allowed Environments** | dev / staging / prod availability, plus `mock_available` and `local_only` vs. `remote_allowed` flags. A tool safe in dev (e.g., `send_email`, `write_to_database`, `delete_file`) may be unsafe or forbidden in prod — this field exists so that distinction is explicit, not assumed |
| **Contract reference** | Pointer to the full Tool Contract entry |

**Binding rule — this is what makes the Registry a runtime control, not just documentation:**

> A tool is callable only if it is (1) present in the Tool Registry, (2) assigned to the requesting agent or agent role, (3) in `Active` lifecycle state, and (4) compatible with the orchestrator's currently supported MCP protocol version.

All four conditions must hold. This is checked at call time by the orchestration layer, not just at registration time — a tool that was Active yesterday but got Deprecated today must fail this check immediately, without requiring a code change anywhere else.

**Assignment authority — single source of truth:** condition (2) raises a fair question — who has the authority to decide that an agent/role is assigned to a tool? The Registry *records* the assignment (it's queried at call time), but the *authority* to grant or revoke an assignment belongs to Card 06's Policy Server, not the Registry itself. This keeps the Runtime Stack discipline intact: Security governs Tools, not the reverse. The Registry is the lookup table; the Policy Server is what writes to it.

**Implementation note:** for our scale (one developer, three projects), this does not need to be a separate database initially — a structured `tool-registry.yaml` or `.json` file in `core/` is sufficient, but the *discipline* of enforcing the binding rule above (rather than letting agents free-discover MCP servers) is what matters, regardless of storage format.

## 6. Tool Lifecycle

Every tool moves through defined states — this prevents the common failure of tools quietly becoming unmaintained or unreviewed while still being actively called:

**Proposed → Designed → Security Reviewed → Built → Tested → Certified → Active → Deprecated → Retired**

This maps onto our existing Master Execution Plan status flags but is more granular for tools specifically. A tool should not move to `Active` (callable by agents) without passing `Security Reviewed` — this is the checkpoint where the §8 MCP risk list (tool shadowing, malicious definitions, etc.) gets explicitly checked off, not assumed.

## 7. A2A Protocol — Agent-to-Agent Communication

Where MCP connects an agent to a *tool*, A2A connects an agent to *another agent* as a peer, not a function call. This matters specifically for our Coordinator pattern (Card 01 §6) — when a coordinator delegates to a specialist agent, A2A is the standard for how that handoff is structured (capability discovery, task delegation, result return), rather than inventing a bespoke calling convention per agent pair.

**Application to our ecosystem:** Within a single project's pipeline, lightweight internal function calls between agents (via LangGraph's native graph edges) are sufficient — we don't need full A2A overhead for in-process coordination. A2A becomes relevant if/when an agent from one project needs to call an agent from another (e.g., a future cross-project capability), or if we ever expose an agent externally for a third party to call. Noted here so the distinction is deliberate, not accidental.

**Trust boundary note:** if/when A2A is actually used, identity verification, capability declaration, authorization, audit trail, and approval requirements for that agent-to-agent handoff are Card 06's responsibility (same logic as MCP's risk gaps in §8 below) — this card describes the mechanism, not its governance.

## 8. MCP Version Pinning

MCP is an evolving protocol. To avoid silent breakage when a server or client library updates:

- **Supported MCP Version** — the exact protocol version our orchestrator (host/client) targets
- **Compatibility Version** — the range of server-side MCP versions we accept connections from
- **Upgrade Review Process** — any MCP protocol version bump is treated as a reviewed change (per the Tool Lifecycle in §6 — moves a tool back to `Security Reviewed` before returning to `Active`), not auto-adopted

## 9. Honest Security Gaps in MCP (Do Not Skip This Section)

The 2025 whitepaper's "For and Against" framing is the most operationally important content in this card — MCP is powerful but not enterprise-hardened by default. Known risk patterns:

- **Tool shadowing** — a malicious or compromised MCP server can register a tool with a name/description designed to be selected over the legitimate one, redirecting the agent's action
- **Malicious tool definitions** — a tool's description text is itself untrusted input to the model; a crafted description can act as a prompt injection vector
- **Sensitive information leakage** — tool outputs (e.g., full database rows) returned without filtering can leak PII/PHI into the model's context, where it may be echoed, logged, or retained in memory
- **The Confused Deputy problem** — an agent acting with its own broad permissions on behalf of a user can be tricked into performing an action the *user* shouldn't be authorized for, because the permission check happened at the agent's level, not the user's

**Direct implication:** None of these are fully solved by MCP itself — they are mitigated at the layers above it. Card 06 (Security) owns the actual mitigations (zero ambient authority, JIT token downscoping, sandboxing, the Policy Server). This card's job is only to make sure we never treat "we used MCP" as equivalent to "this is secure." Every MCP server we build in Phase 4 should be reviewed against this list before being marked `BUILT`.

---

## Direct Implications for This Build

- Every tool in `mcp-servers/` is specified against the expanded Tool Contract table in §4 (versioning, `risk_tier` reference, owner/registry link, error contract aligned to a future shared taxonomy) before implementation — this satisfies Card 01 §8's forward reference
- A central **Tool Registry** (`core/tool-registry.yaml` or similar) is the sole discovery mechanism for agents, and enforces the **binding rule**: a tool is callable only if registered, assigned to the requesting agent/role, `Active`, and MCP-version-compatible — checked at call time, not just registration time
- Tool IDs are globally unique and stable; display names are never treated as authoritative identifiers
- Registry entries include **Allowed Environments** (dev/staging/prod, mock availability, local-only vs. remote-allowed) — a tool safe in dev is not assumed safe in prod
- Every tool moves through the **lifecycle** in §6 (Proposed→...→Retired); no tool reaches `Active` without passing `Security Reviewed`. Formal transition criteria per lifecycle stage and minimum tool-level certification/test requirements (schema, happy-path, failure-path, timeout, retry/idempotency, redaction, data-minimization tests) are noted as a Phase 4 deliverable to define once we're actively building tools — not fully specified in this card to avoid over-specifying before any tool exists
- One MCP server may expose multiple tools; Registry and Lifecycle tracking applies at the tool-capability level, not the server level
- All custom tools follow the 7 design principles in §3 — narrow scope, descriptive naming, structured output, loud/specific failure, explicit idempotency marking, minimal access scope, and data minimization on output
- A2A is deliberately **not** used for in-process, single-project agent coordination (LangGraph native edges suffice); it is reserved for genuine cross-project or external-facing agent-to-agent scenarios, should they arise — and its trust boundary controls (identity, authorization, signing, audit) are Card 06's responsibility, not defined here
- MCP protocol version is explicitly pinned (§8) — upgrades are reviewed changes, not silent auto-adoption
- **Explicitly deferred to Card 06, not duplicated here:** risk tier definitions (this card only requires tools to *reference* a tier, never redefines the tiers), the full Permission Model, sandboxing implementation, and A2A trust boundary enforcement
- The four MCP security gaps in §9 (tool shadowing, malicious definitions, data leakage, confused deputy) remain explicitly NOT mitigated by this card alone — registry uniqueness and the binding rule reduce shadowing risk structurally, but signing/control of registry entries is still Card 06's authority. Full mitigation tracked as an open risk until Card 06's controls are built
- Tool-level observability metadata (per the Tool Contract) feeds directly into Card 05's observability standard — no duplicate logging schemes
