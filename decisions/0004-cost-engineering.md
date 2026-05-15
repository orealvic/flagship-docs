# ADR-0004 — Cost-engineering principles

**Status:** Accepted
**Date:** 2026-05-14

## Context

Total budget: $200 Azure credit. Project lifespan: 14 days build + demo. The architecture must mirror enterprise patterns (CAF, multi-tier, WAF, observability, AI) while staying under ~$50 actual spend so the credit covers the project comfortably with margin for mistakes.

## Decision

The following principles are enforced from Day 1 onward. Every Terraform module and every resource decision is evaluated against them.

### Principle 1 — Free tier first
If a free tier exists and is sufficient for the demo, use it. Document the upgrade path in the module README.

| Service | Free tier used |
|---|---|
| App Service (dev) | F1 |
| Cosmos DB | 1000 RU/s + 25 GB |
| Log Analytics | 5 GB/month ingestion |
| Defender for Cloud | Foundational CSPM |
| Azure AD | Free tier (no P2 features in dev) |

### Principle 2 — Demo-only resources
Resources that cost significant money per hour but are only needed for live demos are deployed on demand and destroyed after. Terraform stacks are split so these can be applied/destroyed independently.

| Resource | Idle cost | Strategy |
|---|---|---|
| Application Gateway WAF v2 | ~$0.30/hr | Apply 30 min before demo, destroy after |
| VPN Gateway VpnGw1 | ~$0.10/hr | Apply 15 min before demo |
| Azure Bastion | ~$0.20/hr | JIT VM access preferred; Bastion only when needed |

### Principle 3 — Schedule everything in dev
All dev resources tagged `autoshutdown=true`. A Logic App on schedule stops them at 19:00 EDT weekdays and all weekend. Mandatory tag policy enforces the tag.

### Principle 4 — Single region
See ADR-0002. No geo-redundancy, no cross-region traffic.

### Principle 5 — Cost-engineered SKUs
| Resource | Standard SKU | Flagship SKU | Saving |
|---|---|---|---|
| App Service prod | P1v3 | B1 | ~80% |
| MySQL Flexible | GP_Standard_D2ds_v4 | B_Standard_B1ms | ~85% |
| Storage account | RAGRS | LRS | ~50% |
| Log Analytics | PerGB2018 unrestricted | PerGB2018 with daily cap of 1 GB | bounded |

### Principle 6 — Hard tripwires
Budget alerts at 50%, 80%, 100% of $200. The 100% threshold triggers an action group that posts to Teams *and* invokes a Logic App that runs `az group stop` against all `autoshutdown=true` tagged resource groups.

### Principle 7 — `terraform destroy` always works
Every stack must support `terraform destroy` cleanly. Tested on Day 1. Tested again before each demo. The "destroy and rebuild in 25 minutes" demo is itself a portfolio artifact.

## Consequences

### Positive
- Spend stays predictable: target $35–45 over 14 days.
- $155+ of credit headroom for screwups, extensions, or experimental side quests.
- Cost discipline is itself a portfolio talking point — every interview includes a FinOps question now.
- The destroy/rebuild cycle becomes a confidence builder and a demo highlight.

### Negative
- Demos require pre-planning (apply expensive stacks ~30 min before a live walkthrough).
- F1 App Service has cold-start latency; dev demos look slow on first request.
- B1 MySQL has no HA; not suitable for production but appropriate for this demo.

### Mitigations
- A pre-demo runbook (`flagship-docs/runbooks/pre-demo-checklist.md`) lists the apply commands to warm everything up.
- Dev cold-start latency is itself a talking point — *"In production I'd run P1v3 with Always On, here's the trade-off."*

## Review cadence

Cost report (`flagship-docs/cost-report.md`) updated daily with actual spend pulled from `az consumption usage list`. The FinOps copilot also posts daily to Teams.
