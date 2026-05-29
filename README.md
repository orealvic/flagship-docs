# Flagship Azure Landing Zone

A production-grade, CAF-aligned Azure landing zone built from scratch in 14 days, cost-engineered to run end-to-end (infra + app + AI + CI/CD) on a $200 Azure credit.

This is the documentation hub. Code lives in five sibling repositories.

---

## Repositories

| Repo | Owns |
|---|---|
| [flagship-platform](https://github.com/orealvic/flagship-platform) | Bootstrap, management groups, Azure Policy, hub VNet, Log Analytics, remote state backend |
| [flagship-landing-zone](https://github.com/orealvic/flagship-landing-zone) | Workload spokes, App Service, MySQL Flex, Key Vault, private endpoints, auto-shutdown |
| [flagship-app](https://github.com/orealvic/flagship-app) | Procurement demo app (Node/Express + React), RAG chatbot, CI/CD pipelines |
| [flagship-actions](https://github.com/orealvic/flagship-actions) | Reusable GitHub Actions workflows (Terraform pipelines, AI PR Reviewer) |
| **flagship-docs** *(you are here)* | Architecture docs, ADRs, runbooks, portfolio evidence |

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │   GitHub (OIDC federation, no secrets)  │
                    │   flagship-actions (reusable workflows) │
                    └────────────────────┬────────────────────┘
                                         │ terraform apply
                                         ▼
       ┌─────────────────────────────────────────────────────────┐
       │              Azure Subscription (PayAsYouGo)            │
       │                                                         │
       │   ┌──────────────────────────────────────────────┐      │
       │   │  Management Groups + Azure Policy            │      │
       │   │  (flagship-platform)                         │      │
       │   └──────────────────────────────────────────────┘      │
       │                                                         │
       │   ┌────────────────┐         ┌────────────────┐         │
       │   │   Hub VNet     │◄───────►│  Prod Spoke    │         │
       │   │  10.10.0.0/16  │ peering │  10.20.0.0/16  │         │
       │   │                │         │                │         │
       │   │  Log Analytics │         │  App Service   │         │
       │   │  (shared)      │         │  MySQL Flex    │         │
       │   └────────────────┘         │  Key Vault     │         │
       │                              │  (private EP)  │         │
       │                              └────────────────┘         │
       │                                                         │
       │   ┌──────────────────────────────────────────────┐      │
       │   │  Automation Account (eastus)                 │      │
       │   │  Tag-driven auto-shutdown for dev resources  │      │
       │   └──────────────────────────────────────────────┘      │
       │                                                         │
       │   ┌──────────────────────────────────────────────┐      │
       │   │  AI layer (in flagship-app)                  │      │
       │   │  • RAG chatbot (Azure OpenAI + Cosmos)       │      │
       │   │  • AI PR Reviewer (gpt-4o-mini)              │      │
       │   └──────────────────────────────────────────────┘      │
       └─────────────────────────────────────────────────────────┘
```

Detailed diagrams in [`diagrams/`](./diagrams) *(in progress)*.

---

## Build progress

**Status as of latest update:** 7 of 14 planned days complete, 1 in progress, 1 deferred, buffer + future days outstanding.

| Day | Topic | Status |
|---|---|---|
| 1 | Bootstrap: remote state, OIDC federation, budget alerts | ✅ Done |
| 2 | Management groups, Azure Policy, tag taxonomy | ✅ Done |
| 3 | Hub networking + observability foundation (VNet, Log Analytics) | ⚠️ Partial — hub VNet and Log Analytics in code; Key Vault and private DNS handled in landing-zone instead |
| 4 | Prod spoke + App Service + MySQL Flex + private endpoints | ✅ Done |
| 5 | Dev workload + tag-driven auto-shutdown (Automation Account) | ✅ Done |
| 6 | Procurement demo app v1 (Node/Express + React) + CI/CD | ✅ Done |
| 7 | Buffer, screenshots, docs | — *(planned buffer)* |
| 8 | Application Gateway WAF v2 with custom rules and 3600s timeout | ⏳ Deferred — Terraform-ready, not yet applied (see ADR-0005) |
| 9 | RAG chatbot — Azure OpenAI + Cosmos DB vector store | ✅ Done |
| 10 | AI PR Reviewer — gpt-4o-mini reviews Terraform plans | ✅ Done |
| 11 | FinOps copilot — Azure Function summarizing daily spend | ⏳ In progress |
| 12 | Observability workbook, alert rules, KQL query library | 📋 Planned |
| 13 | Documentation polish, architecture diagrams, recorded demo | 📋 Planned |
| 14 | `terraform destroy` rehearsal, final teardown, blog post | 📋 Planned |

---

## Recent updates

- **Day 10** — AI PR Reviewer shipped: GitHub Action invokes Azure OpenAI to review Terraform plans and post structured PR comments. Uses managed identity, no API keys in source.
- **Day 9** — RAG chatbot shipped: chat widget in the procurement app, backed by Azure OpenAI `gpt-4o-mini` and Cosmos DB vector store. All secrets via `@Microsoft.KeyVault(SecretUri=...)` references — never embedded in code.
- **Day 5** — Tag-driven auto-shutdown system: Automation Account in eastus runs PowerShell runbooks against any resource tagged `autoshutdown=true`. Stops weekday 7pm EDT, restarts 7am EDT. Cross-region by necessity (subscription type restricts Automation to specific regions).

---

## Architecture decisions

ADRs in [`decisions/`](./decisions):

- **ADR-0001** — Hub-and-spoke topology over flat VNet
- **ADR-0002** — Terraform state in Azure Storage with managed lock
- **ADR-0003** — GitHub Actions OIDC federation over service principal secrets
- **ADR-0004** — Tag-driven cost controls (autoshutdown, mandatory tags)
- **ADR-0005** — Application Gateway as on-demand deployment *(in progress)*

---

## Cost engineering

This project is cost-engineered to fit a $200 Azure credit across 14 build days:

- **Always-on resources** (App Service Basic, MySQL Flex Burstable, Cosmos DB serverless, Log Analytics): ~$35–45/month at idle
- **Expensive resources are demo-only**: App Gateway, VPN Gateway, Bastion — deployed via Terraform only when demoing, then destroyed
- **Tag-driven shutdown**: anything tagged `autoshutdown=true` stops weekdays at 7pm EDT, restarts at 7am EDT
- **AI cost discipline**: `gpt-4o-mini` chosen for both RAG and PR review; estimated <$2/month at demo volumes
- **PayAsYouGo subscription** (not Free Trial): required for OIDC federation, private endpoints, and several Automation regions

---

## How to read this project

If you're a recruiter, hiring manager, or fellow engineer evaluating this work:

1. Start with the **build progress** table above — it tells you what's done vs. planned
2. Read the **ADRs** in `decisions/` to understand the major design choices and tradeoffs
3. Browse [flagship-platform](https://github.com/orealvic/flagship-platform) to see the foundation (state, identity, governance)
4. Browse [flagship-landing-zone](https://github.com/orealvic/flagship-landing-zone) to see the workload patterns
5. Browse [flagship-app](https://github.com/orealvic/flagship-app) to see the demo application and the AI integrations
6. Browse [flagship-actions](https://github.com/orealvic/flagship-actions) to see the reusable CI/CD pipeline patterns

---

## Author

**Victor Ugbor** — Senior Cloud Infrastructure Engineer

- LinkedIn: [linkedin.com/in/victor-u-5984b8242](https://linkedin.com/in/victor-u-5984b8242)
- GitHub: [orealvic](https://github.com/orealvic)

Currently building Azure platform engineering depth for senior cloud roles. AWS + Azure certified. Background in production SaaS infrastructure, FinOps, and ISO 27001 / SOC 2 compliance work.

---

## License

Code repositories: MIT. Documentation: CC BY 4.0.
