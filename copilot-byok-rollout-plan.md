# Copilot BYOK rollout plan

This document plans the rollout of GitHub Copilot with a Bring Your Own Key (BYOK) configuration — connecting an LLM provider's API key to GitHub Copilot and the GitHub Copilot CLI — within a well-managed software delivery organization. The goal is to achieve full AI traceability, cost management, model lifecycle management, and governance/compliance parity with what GitHub-managed models provide out of the box.

---

## Approved model list

Only models on this list may be used via BYOK. The list is intentionally short to limit the governance surface area. New models must pass the onboarding gate (see [Model lifecycle management](#model-lifecycle-management)) before being added.

| Provider | Model | Approved use | Notes |
|---|---|---|---|
| OpenAI | `gpt-4.1` | General code completion, chat, CLI | Current primary model |
| OpenAI | `gpt-4o` | General code completion, chat, CLI | Fallback; review for removal when `gpt-4.1` is stable at scale |
| Anthropic | `claude-sonnet-4-5` | Long-context code review, large-file analysis | Requires Anthropic API key; separate key vault secret |
| Google | `gemini-2.0-flash` | High-throughput, latency-sensitive tasks | Approved for performance-optimized workflows only |

> **Guiding principle:** Prefer depth over breadth. Fewer approved models means fewer keys to rotate, fewer billing accounts to reconcile, and narrower security and compliance scope.

---

## Rollout phases

### Phase 1 — Foundation (Weeks 1–4)

**Goal:** Establish infrastructure, policies, and governance tooling before any users are onboarded.

1. **Secrets management**
   - Provision a dedicated secrets vault (Azure Key Vault, AWS Secrets Manager, or HashiCorp Vault) to hold all provider API keys.
   - Create one secret per approved provider. Do not store keys in `.env` files, source control, or developer machines.
   - Configure automatic 90-day rotation for each key with rotation alerting.

2. **Key distribution architecture**
   - Deploy an internal API proxy/gateway (e.g., Azure API Management, AWS API Gateway, or a lightweight self-hosted reverse proxy) that holds provider keys server-side and issues short-lived scoped tokens to developers.
   - Developers configure Copilot and Copilot CLI to point to the internal proxy endpoint, not directly to provider endpoints. This ensures no raw provider key ever touches a developer machine.
   - The proxy enforces per-user and per-team rate limits, logs every request, and attaches identity context (GitHub username, team, repository) to each outbound call.

3. **Logging and tracing pipeline**
   - Route all proxy traffic logs to a centralized log aggregator (Splunk, Datadog, Azure Monitor, or equivalent already in use).
   - Define the minimum required log fields per request: timestamp, user identity, model ID, model version/snapshot, prompt token count, completion token count, latency, HTTP status, provider response ID.
   - Tag every request with a `correlation_id` that spans the Copilot client, proxy, and provider response for end-to-end traceability.

4. **Cost reporting baseline**
   - Identify the provider billing accounts for each approved model and confirm read access to usage/billing APIs.
   - Create a cost dashboard with daily breakdowns by provider, model, user, and team.
   - Set budget alerts at 50%, 80%, and 100% of the monthly target for each provider.

5. **Policy and compliance review**
   - Review each approved provider's data processing agreement (DPA), data residency commitments, and zero-retention options against organizational compliance requirements (SOC 2, ISO 27001, GDPR, HIPAA, or equivalent).
   - Document the compliance status of each provider/model combination in a shared register.
   - Obtain legal/security sign-off before proceeding to pilot.

---

### Phase 2 — Pilot (Weeks 5–8)

**Goal:** Validate the full stack with a small, representative group of early adopters under close observation.

1. **Pilot cohort selection**
   - Select 10–20 developers across 2–3 teams. Include a mix of seniority, IDE preferences, and operating systems.
   - Confirm all pilot participants have completed AI usage policy training.

2. **Developer configuration**
   - Distribute internal proxy endpoint and scoped token generation instructions via an internal runbook (not by sharing raw provider keys).
   - Configure `GITHUB_COPILOT_LLM_PROVIDER_API_KEY` (or equivalent Copilot CLI config) to use the internal proxy token.
   - Validate Copilot and Copilot CLI connectivity against each approved model.

3. **Observability validation**
   - Confirm that every request from the pilot cohort appears in the log pipeline with full required fields.
   - Validate that correlation IDs propagate correctly across client, proxy, and provider.
   - Confirm cost dashboard reflects pilot usage within expected latency (target: under 1 hour lag).

4. **Feedback collection**
   - Run a structured weekly feedback session with pilot participants.
   - Track issues in the team's standard backlog. Prioritize anything blocking traceability, compliance, or security.

5. **Go/no-go review**
   - At the end of Week 8, review: security posture, cost tracking accuracy, log completeness, compliance sign-offs, and developer experience.
   - Require explicit approval from engineering leadership, security, and finance before Phase 3.

---

### Phase 3 — Controlled expansion (Weeks 9–16)

**Goal:** Extend access to all engineering teams while refining operational playbooks.

1. **Staged onboarding**
   - Onboard teams in batches of 20–30 developers per week.
   - Require each team lead to acknowledge the AI usage policy before their team is enabled.

2. **Self-service provisioning**
   - Automate scoped token issuance via an internal developer portal or CLI tool. Developers request access; the system issues a short-lived proxy token tied to their GitHub identity.
   - Eliminate all manual key distribution steps.

3. **Cost allocation enforcement**
   - Enable per-team cost tagging on the proxy. Map proxy routing labels to cost centers.
   - Publish weekly cost-per-team reports to engineering managers and finance partners.
   - Implement automatic throttling if a team exceeds 120% of its monthly model budget.

4. **Incident response rehearsal**
   - Run a tabletop exercise: simulate a suspected key leak and measure time-to-revoke, re-provision, and restore service.
   - Target recovery time objective (RTO): under 30 minutes from detection to restored access.

5. **Compliance audit trail hardening**
   - Enable immutable log storage (WORM or equivalent) for all proxy request logs. Retention period aligned to audit policy (typically 1–3 years).
   - Confirm audit trail is accessible to compliance and security teams without requiring developer access.

---

### Phase 4 — Full production (Weeks 17–24)

**Goal:** Complete organization-wide rollout with all controls stable and fully automated.

1. **Full rollout completion**
   - Complete onboarding of all remaining engineering teams.
   - Publish user-facing documentation: how to configure Copilot BYOK, how to request access, how to report issues.

2. **Model governance committee**
   - Establish a standing committee (engineering, security, finance, legal) that meets quarterly to review the approved model list, audit results, cost trends, and provider security posture.

3. **Automated compliance checks**
   - Integrate proxy log completeness checks into CI/CD pipelines or scheduled jobs. Alert on any gap (missing fields, dropped requests, correlation ID failures).
   - Add automated checks to confirm no raw provider keys are committed to source control (secret scanning on all repositories).

4. **SLA definition**
   - Define availability and latency SLAs for the internal proxy (recommended baseline: 99.9% availability, p95 latency ≤ 200 ms added overhead).
   - Set up alerting and on-call rotation for proxy incidents.

---

### Phase 5 — Steady state (ongoing)

Ongoing operations after full rollout is complete.

---

## AI traceability

Traceability means being able to answer: *who made this AI request, using which model and version, at what time, with what input, and what output was returned?*

| Traceability requirement | Implementation |
|---|---|
| User identity | Internal proxy attaches GitHub user identity to every request via the scoped token. |
| Model identity | Proxy records the full model ID and, where available, the model version or snapshot identifier returned by the provider API. |
| Request/response logging | Full token counts logged; raw prompt and completion content logged to a restricted-access store (see [Data sensitivity](#data-sensitivity)). |
| End-to-end correlation | Every request carries a `correlation_id` linking Copilot client logs, proxy logs, and provider response IDs. |
| Immutable audit trail | Logs written to WORM-compliant storage; tamper evidence enforced. |
| Retention | Minimum 1 year for metadata logs; prompt/completion content retention per DPA and data classification policy. |

### Data sensitivity

Raw prompt and completion content is sensitive. Access to content logs must be restricted to security and compliance investigations under a documented process. Operational dashboards should show only metadata (token counts, latency, error rates) — not prompt text.

---

## Cost management

| Control | Mechanism |
|---|---|
| Budget setting | Monthly per-provider and per-team budgets set in cost dashboard with automated alerts. |
| Real-time visibility | Proxy emits per-request cost estimates (token count × known provider rate) to the cost pipeline; reconciled against provider invoices weekly. |
| Chargeback/showback | Teams receive weekly usage reports. Finance teams receive monthly cost-center allocations. |
| Throttling | Automatic soft throttle at 80% of team budget (warning); hard throttle at 120% pending manager approval. |
| Model cost governance | Before approving a new model, the model governance committee reviews its cost profile relative to alternatives. High-cost models require explicit finance approval. |
| Invoice reconciliation | Automated job compares proxy-estimated spend against provider invoice line items monthly. Variance >5% triggers investigation. |

---

## Model lifecycle management

Models must be actively managed throughout their lifecycle — from evaluation to retirement.

### Onboarding a new model

1. **Proposal:** Any team may propose a new model. Proposal includes use case, provider DPA status, cost estimate, and security considerations.
2. **Security and compliance review:** Security team reviews provider data handling, key isolation, and any new compliance obligations.
3. **Technical pilot:** 2-week internal pilot with the proposing team. Collect latency, cost, and quality data.
4. **Governance committee approval:** Committee votes to add the model to the approved list, sets allowed use cases, and assigns a model owner.
5. **Proxy configuration:** Add model to proxy routing; enable logging and cost tagging.
6. **Documentation update:** Update the approved model list in this document.

### Version pinning and upgrades

- The internal proxy pins requests to a specific model version where the provider API supports versioning.
- Model version upgrades follow a defined process: pilot with a subset of users, validate quality and cost, then promote to all users.
- Deprecated model versions are removed from the approved list on a date published at least 30 days in advance.

### Retiring a model

1. Governance committee identifies models for retirement (low usage, high cost, superseded by newer model, provider deprecation).
2. Publish retirement date at least 30 days in advance.
3. Migrate affected users to an approved alternative.
4. Remove model from proxy routing and approved list.
5. Archive associated cost and usage data per retention policy.

---

## Governance and compliance

### Policy controls

- **Allowlist enforcement:** The internal proxy only routes requests to approved provider endpoints and model IDs. Any request for a non-approved model is rejected and logged.
- **User eligibility:** Only users who have completed AI usage policy training and have an active Copilot license are issued proxy tokens.
- **Data residency:** Approved models are pre-screened for data residency. Requests are routed only to provider regions permitted by organizational policy.
- **Zero-retention options:** Where available (OpenAI Enterprise, Anthropic Enterprise), zero-retention API agreements are in place. Documented for each provider.

### Access management

- Proxy tokens are scoped to the individual developer and the set of models they are authorized to use.
- Tokens expire after 8 hours and must be re-issued via the developer portal. This limits blast radius of a compromised token.
- Privileged access (proxy administration, key vault access) is restricted to a named operations team, reviewed quarterly.

### Audit and reporting

| Audit activity | Frequency | Owner |
|---|---|---|
| Log completeness check | Daily (automated) | Platform engineering |
| Cost reconciliation | Monthly | Finance + platform engineering |
| Access review (who has proxy token access) | Quarterly | Security |
| Approved model list review | Quarterly | Model governance committee |
| Provider DPA and compliance status review | Annually (or on provider change) | Legal + security |
| Penetration test of proxy and key vault | Annually | Security |
| Key rotation | Every 90 days (automated) | Platform engineering |

### Incident response

- A runbook documents steps for suspected key compromise: revoke key, re-provision, notify affected users, file security incident report.
- Security incidents involving AI-generated content (e.g., prompt injection leading to exfiltration) are handled under the standard security incident process.

---

## Administrative effort summary

This section estimates the ongoing administrative work required throughout the lifetime of the BYOK configuration, grouped by role.

### One-time setup (Phases 1–4, approximately 6 months)

| Activity | Estimated effort |
|---|---|
| Secrets vault setup and key onboarding | 2–3 days (platform engineering) |
| Internal proxy deployment and configuration | 1–2 weeks (platform engineering) |
| Logging pipeline configuration | 3–5 days (platform/observability engineering) |
| Cost dashboard and alerting setup | 3–5 days (platform engineering + finance) |
| Provider DPA review and compliance documentation | 1–2 weeks (legal + security) |
| Policy authoring and training development | 1 week (security + HR/training) |
| Developer portal / self-service token flow | 1–2 weeks (platform engineering) |
| Pilot execution and feedback cycle | 4 weeks (platform + 1 embedded engineering manager) |
| Staged rollout coordination | 8 weeks (platform engineering, part-time) |
| **Total one-time setup** | **~3–4 months of part-time effort across 4–6 people** |

### Recurring steady-state effort (per year, post-rollout)

| Activity | Estimated effort per year |
|---|---|
| Key rotation (automated, with human verification) | 4 rotations × 1 hour = ~4 hours |
| Monthly cost reconciliation | 12 × 2 hours = ~24 hours (finance + platform) |
| Quarterly access reviews | 4 × 4 hours = ~16 hours (security) |
| Quarterly model governance committee meetings | 4 × 2 hours = ~8 hours (committee of 4–5 people) |
| Model onboarding (estimated 1–2 new models/year) | 2 × 3 days = ~6 days (platform + security + legal) |
| Model retirement (estimated 1–2/year) | 2 × 1 day = ~2 days (platform) |
| Annual DPA/compliance review | ~3 days (legal + security) |
| Annual penetration test | ~5 days (security team or external) |
| Incident response (estimated 1–2 minor incidents/year) | ~2 days per incident |
| User onboarding and support | ~1 hour/week = ~52 hours |
| **Total recurring steady-state** | **~30–40 person-days per year** |

### Comparison with GitHub-managed models

GitHub-managed models eliminate proxy operations, key management, provider DPA management, and custom cost reconciliation. The equivalent recurring overhead with GitHub-managed models is approximately **5–10 person-days per year** (primarily Copilot seat administration and policy configuration).

The BYOK configuration therefore carries a **sustained overhead premium of roughly 20–35 person-days per year** in exchange for model choice flexibility, deeper cost visibility, and tighter control over data handling and traceability. This overhead should be weighed against the concrete benefits of the specific models approved: if the approved models provide meaningfully better outcomes than GitHub-managed alternatives, the overhead is justified. If not, GitHub-managed models remain the operationally simpler choice.

---

## Related reading

- [GitHub Copilot model usage modes: comparative analysis](./copilot-model-usage-modes-analysis.md)
