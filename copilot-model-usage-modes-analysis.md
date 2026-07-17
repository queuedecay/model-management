# GitHub Copilot model usage modes: comparative analysis

This document compares two ways to use models with GitHub Copilot workflows:

1. **GitHub-managed models**
2. **Self-hosted or third-party model via provider API key in VS Code/Copilot CLI**
   - with a Copilot subscription
   - without a Copilot subscription

## At-a-glance comparison

| Factor | 1) GitHub-managed models | 2a) Self-hosted model via API key **with** Copilot subscription | 2b) Self-hosted model via API key **without** Copilot subscription |
|---|---|---|---|
| **Cost** | Predictable Copilot licensing (plus any premium usage, if applicable to plan/model). | Split cost model: Copilot entitlement + external provider/self-hosted runtime cost. | No Copilot seat cost, but full inference/platform cost is on you/provider; can be cheaper or more expensive depending on usage scale. |
| **Visibility of cost** | Strong GitHub-side seat visibility; model-level unit economics may be less granular than cloud billing systems. | Depends on provider tooling; typically better raw token/request billing visibility than seat-only views. | Highest dependency on provider/self-hosted metering discipline; no Copilot seat reporting to reconcile. |
| **Administrative effort** | Lowest: no model hosting, scaling, or endpoint operations. | Medium-high: manage provider keys/endpoints, routing, quotas, and potential reliability concerns. | Highest in practice: all integration, access management, and user enablement are self-managed without Copilot-admin scaffolding. |
| **Observability** | Good product-level usage visibility; limited low-level model runtime telemetry control. | Variable: depends on provider APIs and what telemetry the team builds around them. | Variable-to-low unless you build full telemetry stack yourself. |
| **Performance** | Usually strong out of the box, globally optimized by GitHub/provider partnerships; limited tuning knobs. | Potentially best-fit performance if you control model choice/infrastructure, but requires active tuning. | Can be excellent in niche scenarios, but consistency is hardest without managed orchestration. |
| **Governance** | Centralized Copilot policy controls are simplest to apply consistently. | Governance depends on both Copilot controls and provider controls; policy consistency can drift. | Weakest default governance unless organization builds equivalent controls independently. |
| **Data security** | Strong baseline enterprise controls in GitHub ecosystem; less control over underlying hosting topology. | Security posture depends on provider and key hygiene; risk increases with key sprawl and ad hoc endpoints. | Highest operational risk if key management, access control, and audit practices are immature. |

## Detailed contrast by mode

### 1) GitHub-managed models

- **Best for simplicity and fast adoption.**
- Minimal setup, centralized Copilot administration, and low operational burden.
- Tradeoff: less direct control over model runtime internals, cloud placement details, and deep infrastructure telemetry.

### 2) Self-hosted/provider model via API key in VS Code and Copilot CLI

#### 2a) With Copilot subscription

- **Best for teams that want Copilot UX/governance plus model flexibility.**
- Lets teams combine Copilot productivity features with custom model choice and provider economics.
- Tradeoff: split ownership across Copilot and model provider; governance and security controls are harder to keep uniform.

#### 2b) Without Copilot subscription

- **Best for highly customized or cost-optimized workflows outside standard Copilot seat licensing.**
- Gives maximum independence in model/runtime selection.
- Tradeoff: you lose Copilot subscription-backed capabilities and must self-manage almost everything (identity, policy, usage controls, reliability, telemetry).

## Main use cases

- **GitHub-managed models**
  - Rapid rollout across engineering teams
  - Standardized developer experience with low ops overhead
  - Organizations prioritizing productivity and administrative simplicity

- **Self-hosted/provider API-key models (with or without Copilot subscription)**
  - Teams requiring specific model families not available in managed offerings
  - Organizations optimizing for custom cost/performance envelopes
  - Advanced platform teams capable of handling security, policy, and operations end to end

## Practical selection guidance

- Choose **GitHub-managed** when speed, low friction, and standardization matter most.
- Choose **self-hosted/provider API-key** when model flexibility is the top priority and your team can sustain the operational/security burden.
