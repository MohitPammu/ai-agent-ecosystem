# Model Routing Table
## AI Agent Ecosystem

**Status:** 1.0 (Draft) — structurally usable for routing, but external-provider approvals remain limited by the policy evidence below.

**Purpose:** Operationalizes Card 01 §4's specialist-model routing principle and Card 06 §26's classification-aware model routing requirement. This file is the routing control used by `core/harness/` and Agent Contracts. It applies canonical architecture rules; it does not redefine Card 01, Card 06, or Card 07.

**Binding compliance rule:** Unknown, missing, expired, or unsourced provider-policy evidence fails closed for Tier 2+ data. If a provider is not explicitly approved for a data classification, route to Ollama local or block the call.

---

# 1. Routing Decision Order

Every model-routing decision follows this order:

1. Determine the request's highest data classification under Card 06 §13/§23.
2. Reject any provider whose policy evidence is missing or expired for that classification.
3. Reject any provider whose approved classes do not include the request's classification.
4. Reject preview/beta/experimental model surfaces for Tier 2+ data unless explicitly approved.
5. Apply task-complexity routing.
6. Apply cost/quota tie-breakers only after compliance gates pass.
7. Log the routing decision with provider, model, data classification, approval basis, and evidence version.

Classification always wins over task complexity and cost.

---

# 2. Data Classification Routing Rule

| Data classification | Default route | External provider allowed? |
|---|---|---|
| `public` | Any approved provider | Yes, if policy evidence is current |
| `internal` | Ollama preferred; Gemini/GitHub allowed only if current evidence permits | Yes, if policy evidence is current |
| `sensitive-PII` | Ollama local | No, unless a provider row is explicitly approved by documented organizational approval |
| `sensitive-PHI` | Ollama local | No, unless BAA/DPA/regulatory review explicitly approves the route |
| `regulated-financial` | Ollama local | No, unless documented organizational approval explicitly approves the route |

**Default:** Projects 1 and 3 route all sensitive-PII, sensitive-PHI, Structured Records, and regulated-financial data to Ollama local unless an explicit non-local approval row exists and is current.

**Open question, explicitly flagged (not resolved here):** Each project's actual data source(s) and their classification status are a Phase 5+ scoping decision, not a Phase 0 infrastructure decision. This table establishes the routing *rules*; it does not presume which specific source(s) Healthcare Fraud Detection (or any project) will ultimately use, nor their de-identification status. Whatever source(s) are chosen, their classification must be explicitly verified against Card 06 §13's taxonomy before that project begins — not assumed from this table's conservative defaults.

---

# 3. Approval Status Values

| Status | Meaning |
|---|---|
| `approved` | Usable for listed classes |
| `conditional` | Usable only if stated conditions are true |
| `public_internal_only` | Approved only for public/internal data; never Tier 2+ |
| `verify_required` | Not usable beyond public until reviewed |
| `blocked` | Do not use |
| `expired` | Previously reviewed but stale; fail closed |

---

# 4. Provider Policy Evidence Schema

Each provider/product/model row must maintain these fields.

| Field | Purpose |
|---|---|
| `provider` | Provider name |
| `product_surface` | Exact product surface, e.g. Gemini API unpaid quota, Gemini API paid quota, GitHub Models, GitHub Copilot Individual |
| `model_id` | Exact model identifier |
| `model_status` | GA / preview / beta / deprecated |
| `account_type` | Free / Pro / paid API / Business / Enterprise / local |
| `approved_classes` | Data classifications this row may receive |
| `blocked_classes` | Data classifications this row may not receive |
| `approval_status` | One of the statuses in §3 |
| `approval_basis` | local-only / sourced-provider-policy / enterprise-contract / compliance-review / blocked-by-policy |
| `training_use_allowed` | yes / no / opt-out / unknown / n/a |
| `retention_policy` | Provider retention behavior |
| `prompt_response_logging` | Whether prompts/responses are logged |
| `human_review_possible` | Whether provider human review is possible |
| `region_residency` | Region/residency behavior |
| `zero_retention_available` | yes / no / unknown / n/a |
| `feature_flags_required` | Required disabled/enabled settings |
| `evidence_source` | Official source used for approval |
| `source_effective_date` | Effective date of cited provider terms |
| `verified_on` | Date this row was reviewed |
| `verified_by` | Reviewer |
| `next_review_due` | Expiration date for this evidence |
| `notes` | Constraints, caveats, and account-specific details |

---

# 5. Provider Approval Table

## 5.1 Ollama Local

| Field | Value |
|---|---|
| `provider` | Ollama |
| `product_surface` | Local inference |
| `model_id` | Project-selected local model |
| `model_status` | Local |
| `account_type` | Local |
| `approved_classes` | public / internal / sensitive-PII / sensitive-PHI / regulated-financial |
| `blocked_classes` | none, subject to local controls |
| `approval_status` | approved |
| `approval_basis` | local-only |
| `training_use_allowed` | n/a |
| `retention_policy` | No external provider retention |
| `prompt_response_logging` | Local only; must be disabled or secured for sensitive classes |
| `human_review_possible` | No provider human review |
| `region_residency` | Local machine only |
| `zero_retention_available` | n/a — local-only |
| `feature_flags_required` | No external telemetry; secure local logging; encrypted storage for sensitive classes |
| `evidence_source` | Local architecture control, Card 06 §26 |
| `source_effective_date` | n/a |
| `verified_on` | 2026-06-27 |
| `verified_by` | Project owner |
| `next_review_due` | Review with local security baseline before Phase 1 sensitive-data use |
| `notes` | Approval assumes local environment controls: encrypted disk, access control, no insecure logs, no sensitive shell history, no unencrypted backups, and no external telemetry. |

---

## 5.2 Gemini API — Unpaid Services / Free Tier

| Field | Value |
|---|---|
| `provider` | Google |
| `product_surface` | Gemini API unpaid quota / Google AI Studio unpaid services |
| `model_id` | Gemini Flash variant in use — must be filled exactly before integration |
| `model_status` | Must verify at integration |
| `account_type` | Unpaid / free tier |
| `approved_classes` | public / internal only |
| `blocked_classes` | sensitive-PII / sensitive-PHI / regulated-financial |
| `approval_status` | public_internal_only |
| `approval_basis` | sourced-provider-policy |
| `training_use_allowed` | yes / provider may use submitted content and generated responses to improve/develop products and ML technologies |
| `retention_policy` | Human review and provider processing possible; exact retention not approved for sensitive routing |
| `prompt_response_logging` | Provider processing applies |
| `human_review_possible` | yes |
| `region_residency` | Not approved for sensitive routing |
| `zero_retention_available` | not approved / not established for unpaid services |
| `feature_flags_required` | No sensitive, confidential, personal, PHI, or regulated-financial data; grounding/search disabled unless separately reviewed |
| `evidence_source` | Google Gemini API Additional Terms of Service — Unpaid Services |
| `source_effective_date` | 2026-03-23 |
| `verified_on` | 2026-06-27 |
| `verified_by` | Project owner |
| `next_review_due` | 2026-07-27 or before any production integration, whichever comes first |
| `notes` | Google states not to submit sensitive, confidential, or personal information to Unpaid Services. Therefore this row is never approved for Tier 2+ data. |

---

## 5.3 Gemini API — Paid Services

| Field | Value |
|---|---|
| `provider` | Google |
| `product_surface` | Gemini API paid quota through Cloud Project with active billing |
| `model_id` | Gemini Flash variant in use — must be filled exactly before integration |
| `model_status` | Must verify at integration |
| `account_type` | Paid API |
| `approved_classes` | public / internal only by default |
| `blocked_classes` | sensitive-PII / sensitive-PHI / regulated-financial until documented organizational approval is granted otherwise |
| `approval_status` | conditional |
| `approval_basis` | sourced-provider-policy; documented organizational approval required for Tier 2+ |
| `training_use_allowed` | no for prompts/responses improving Google products under Paid Services terms |
| `retention_policy` | Prompts/responses may be logged for a limited period for abuse monitoring/safety/security/legal reasons |
| `prompt_response_logging` | yes, limited provider logging for abuse monitoring/safety/security/legal reasons |
| `human_review_possible` | not approved for sensitive routing until reviewed |
| `region_residency` | May be transiently stored or cached in any country where Google or agents maintain facilities unless a separate product/contract constrains this |
| `zero_retention_available` | not established for this row |
| `feature_flags_required` | Grounding/Search disabled for sensitive data unless separately approved; no session/caching feature enabled unless reviewed |
| `evidence_source` | Google Gemini API Additional Terms of Service — Paid Services |
| `source_effective_date` | 2026-03-23 |
| `verified_on` | 2026-06-27 |
| `verified_by` | Project owner |
| `next_review_due` | 2026-07-27 or before any Tier 2+ use, whichever comes first |
| `notes` | Not approved until evidence satisfies this table's approval requirements. Not part of the current zero-cost stack. Do not use for sensitive-PII/PHI/regulated-financial data until DPA/BAA, region, logging, and project-specific legal requirements are explicitly approved. |

---

## 5.4 GitHub Models

| Field | Value |
|---|---|
| `provider` | GitHub / model-hosting company |
| `product_surface` | GitHub Models |
| `model_id` | Must be filled per selected model |
| `model_status` | Must verify per selected model |
| `account_type` | GitHub account / Marketplace access |
| `approved_classes` | public / internal only |
| `blocked_classes` | sensitive-PII / sensitive-PHI / regulated-financial |
| `approval_status` | public_internal_only |
| `approval_basis` | sourced-provider-policy; model-host-specific terms required |
| `training_use_allowed` | model-host-specific; not approved for Tier 2+ without review |
| `retention_policy` | model-host-specific; not approved for Tier 2+ without review |
| `prompt_response_logging` | model-host-specific |
| `human_review_possible` | model-host-specific |
| `region_residency` | model-host-specific |
| `zero_retention_available` | model-host-specific |
| `feature_flags_required` | No sensitive data; exact model host terms must be reviewed before any elevation |
| `evidence_source` | GitHub Terms for Additional Products and Features — GitHub Models |
| `source_effective_date` | 2026-04-27 |
| `verified_on` | 2026-06-27 |
| `verified_by` | Project owner |
| `next_review_due` | 2026-07-27 or before any selected model integration, whichever comes first |
| `notes` | GitHub Models use is subject to the terms of the company hosting the model and the model license. Therefore this row cannot approve sensitive data generically. Create a model-specific row before any Tier 2+ routing. |

---

## 5.5 GitHub Copilot Individual / Free / Pro / Pro+

| Field | Value |
|---|---|
| `provider` | GitHub Copilot |
| `product_surface` | Copilot Individual / Free / Pro / Pro+ |
| `model_id` | Copilot-selected model; varies |
| `model_status` | Provider-managed |
| `account_type` | Individual consumer |
| `approved_classes` | public / internal only |
| `blocked_classes` | sensitive-PII / sensitive-PHI / regulated-financial |
| `approval_status` | public_internal_only |
| `approval_basis` | sourced-provider-policy |
| `training_use_allowed` | may be used for AI training unless opted out, per GitHub's 2026 update for Free/Pro/Pro+ |
| `retention_policy` | GitHub product-specific; not approved for sensitive routing |
| `prompt_response_logging` | yes / product interaction data exists |
| `human_review_possible` | not approved for sensitive routing |
| `region_residency` | not approved for sensitive routing |
| `zero_retention_available` | not established for this account type |
| `feature_flags_required` | AI training opt-out must be enabled if used at all; no sensitive data |
| `evidence_source` | GitHub Privacy/Terms update, March 25 2026 |
| `source_effective_date` | 2026-04-24 |
| `verified_on` | 2026-06-27 |
| `verified_by` | Project owner |
| `next_review_due` | 2026-07-27 |
| `notes` | Existing individual Copilot subscription is not approved for Tier 2+ project data. Do not route PHI, PII, or regulated-financial content here. |

---

## 5.6 GitHub Copilot Business / Enterprise

| Field | Value |
|---|---|
| `provider` | GitHub Copilot |
| `product_surface` | Copilot Business / Enterprise |
| `model_id` | Must be model-specific |
| `model_status` | Must verify; preview/beta models fail closed for Tier 2+ |
| `account_type` | Business / Enterprise |
| `approved_classes` | public / internal by default |
| `blocked_classes` | sensitive-PII / sensitive-PHI / regulated-financial until documented enterprise compliance approval is granted otherwise |
| `approval_status` | conditional |
| `approval_basis` | enterprise-contract / DPA review required |
| `training_use_allowed` | GitHub states Business/Enterprise users are not affected by the 2026 Free/Pro/Pro+ training update |
| `retention_policy` | model-specific; check model-hosting documentation and enterprise agreement |
| `prompt_response_logging` | model-specific |
| `human_review_possible` | model-specific |
| `region_residency` | model-specific |
| `zero_retention_available` | model-specific |
| `feature_flags_required` | Disable preview/beta models for Tier 2+ unless explicitly approved; verify selected model hosting path |
| `evidence_source` | GitHub Copilot model-hosting docs; GitHub DPA landing page; GitHub Privacy/Terms update |
| `source_effective_date` | 2026-04-24 / current docs at verification time |
| `verified_on` | 2026-06-27 |
| `verified_by` | Project owner |
| `next_review_due` | Before any enterprise use |
| `notes` | Not approved until evidence satisfies this table's approval requirements. Not part of the current zero-cost stack. Do not use for sensitive project data until enterprise terms, DPA/BAA/regulatory posture, selected model, and hosting chain are approved. |

---

# 6. Task-Complexity Routing

This table applies only after the classification gate in §§1-5 has approved the provider.

| Task class | Task shape | Default route | Rationale |
|---|---|---|---|
| Deterministic validator | Schema checks, structural validation, simple pass/fail | No LLM call | Plain code is more reliable and cheaper |
| Level 1 retrieval / simple transformation | Simple lookup, classification, summarization | Ollama local first; Gemini API unpaid only for public/internal overflow | High-volume, low-complexity |
| Level 2 planning | Multi-step reasoning, context engineering, ambiguous planning | Gemini/GitHub only for public/internal and only with current evidence; otherwise Ollama | Higher reasoning need, but compliance gate still wins |
| Level 3 coordination | Cross-agent routing, delegation planning | Same as Level 2 | Coordination is planning, not a separate compliance class |

---

# 7. Tie-Breaking Rules

When more than one provider is compliant for the request classification:

1. Prefer no LLM call for deterministic tasks.
2. Prefer Ollama for trivial/high-volume work.
3. Prefer the lowest-cost provider with current approval evidence.
4. Prefer the provider with remaining quota/headroom.
5. If all external providers are exhausted or non-compliant, fall back to Ollama.
6. If Ollama cannot satisfy the task and no compliant external provider exists, Escalate or Stop — never route sensitive data to a non-approved provider.

---

# 8. Feature Flags That Change Approval

The following features materially change data handling and require separate review before use with Tier 2+ data:

| Feature | Default for Tier 2+ |
|---|---|
| Gemini Grounding with Google Search | Disabled |
| Gemini Grounding with Google Maps | Disabled |
| Gemini session resumption / prompt caching features | Disabled unless reviewed |
| GitHub Copilot web search | Disabled |
| GitHub Copilot preview/beta models | Disabled |
| GitHub Models selected model without model-host terms reviewed | Blocked |
| BYOK through GitHub Copilot | Blocked until provider-specific review |

---

# 9. Maintenance and Review Cadence

Provider policy evidence must be reviewed:

- before Phase 1 begins,
- before any Tier 2+ external-provider use,
- whenever a provider changes terms,
- whenever the selected model changes,
- whenever account type changes,
- whenever a feature flag that affects data handling is enabled,
- at least every 30 days for non-local providers during active development.

If `next_review_due` has passed, the provider row is treated as `expired` and fails closed for Tier 2+ data.

---

# 10. Current Routing Verdict

| Provider row | Current use |
|---|---|
| Ollama Local | Approved for all classes, subject to local security controls |
| Gemini API Unpaid | Public/internal only |
| Gemini API Paid | Public/internal only by default; sensitive approval deferred |
| GitHub Models | Public/internal only; model-specific terms required |
| GitHub Copilot Individual / Free / Pro / Pro+ | Public/internal only; no sensitive project data |
| GitHub Copilot Business / Enterprise | Not approved until evidence satisfies this table's approval requirements; not part of the current zero-cost stack |

**Binding current-state rule:** Until external provider rows are explicitly approved for sensitive classes, all sensitive-PII, sensitive-PHI, regulated-financial, and Structured Records route to Ollama local or are blocked if local execution is unavailable.

---

# 11. External Framework Alignment (Reference, Not Adopted Wholesale)

This table's structure — fail-closed-by-default, evidence-with-expiration, explicit approval states — is independently consistent with established AI governance frameworks, cited here for portfolio credibility and as an external sanity check, not as a substitute for the card-derived rules above:

- **NIST AI RMF 1.0** (public domain, NIST) — this table's evidence/expiration/approval-state structure maps onto RMF's Map/Measure functions (system inventory, third-party risk evaluation, documented approval criteria).
- **NIST AI RMF Agentic Profile** (Cloud Security Alliance Labs, 2026) — extends RMF specifically for autonomous multi-agent systems with formal autonomy-tier classification and delegation-chain accountability, conceptually aligned with Card 01's 5-level taxonomy and Card 06's risk tiers, though not formally cross-mapped here.

No content from either framework is reproduced or adopted as binding — Cards 01-07 remain the sole authority for this ecosystem's actual rules.

---

## Sign-off

| Field | Value |
|---|---|
| **Status** | 1.0 (Draft) |
| **Approved by** | _pending_ |
| **Approval date** | _pending_ |

**Versioning note:** follows the same convention as Cards 01-07 — version increments by one integer per substantive edit (1.0 → 2.0 → 3.0...), with a one-line changelog added at the point of each edit, once approved. No decimal sub-versioning.

