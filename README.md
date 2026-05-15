# Flagship Azure Landing Zone

A production-grade, CAF-aligned Azure landing zone built from scratch in 14 days, cost-engineered to run end-to-end (infra + app + AI + CI/CD) on a $200 Azure credit.

This is the documentation hub. Code lives in five sibling repositories.

## Repositories

| Repo | What it owns |
|---|---|
| [flagship-platform](https://github.com/orealvic/flagship-platform) | Management groups, Azure Policy, hub network, state backend, OIDC trust |
| [flagship-landing-zone](https://github.com/orealvic/flagship-landing-zone) | Prod and dev spokes, workload infrastructure (App Service, MySQL, App Gateway) |
| [flagship-app](https://github.com/orealvic/flagship-app) | Procurement-lite demo app — Node/Express API + React frontend |
| [flagship-ai](https://github.com/orealvic/flagship-ai) | RAG chatbot, FinOps copilot, AI-powered pipeline reviewer |
| [flagship-actions](https://github.com/orealvic/flagship-actions) | Reusable GitHub Actions workflows shared across repos |

## Architecture at a glance

```
┌─────────────────────────────────────────────────────────────────┐
│  Microsoft Entra ID  ←──  PIM-eligible roles, federated GH OIDC │
└─────────────────────────────────────────────────────────────────┘
            │
┌───────────┴────────────┐
│  Management Groups     │   tg-flagship (root)
│  + Azure Policy        │     ├─ platform
│  + Tag taxonomy        │     └─ landingzones
└────────────────────────┘          ├─ prod
            │                       └─ dev
┌───────────┴────────────────────────────────────────────────┐
│  Hub VNet  10.10.0.0/16  (Canada Central)                  │
│  ├─ Shared services (Log Analytics, Defender, DNS)         │
│  ├─ GatewaySubnet (P2S VPN, demo-only)                     │
│  └─ AzureBastionSubnet (demo-only)                         │
└────────────┬─────────────────────────────┬─────────────────┘
             │ peering                     │ peering
┌────────────┴──────────────┐  ┌───────────┴──────────────┐
│  Prod spoke 10.20.0.0/16  │  │  Dev spoke 10.30.0.0/16  │
│  • AGW v2 WAF             │  │  • App Service F1 web/api│
│  • App Service B1 web/api │  │  • MySQL B1ms (scheduled)│
│  • MySQL B1ms + PE        │  │  • Cosmos free tier      │
│  • Key Vault + PE         │  │                          │
└───────────────────────────┘  └──────────────────────────┘
             │
┌────────────┴────────────────────────────────────────────────┐
│  AI Layer                                                   │
│  • Azure OpenAI (gpt-4o-mini + text-embedding-3-small)      │
│  • RAG chatbot embedded in app (Cosmos vector store)        │
│  • FinOps copilot → Teams (Function App + Reader RBAC)      │
│  • Pipeline reviewer (GitHub Action, comments on PRs)       │
└─────────────────────────────────────────────────────────────┘
             │
┌────────────┴────────────────────────────────────────────────┐
│  CI/CD: GitHub Actions, OIDC federated, reusable workflows  │
│  Security: gitleaks, tfsec, checkov, trivy, CodeQL          │
│  Observability: Log Analytics + App Insights + Workbooks    │
└─────────────────────────────────────────────────────────────┘
```

## Cost engineering

Target spend: **$35–45 over 14 days** out of the $200 credit.

| Principle | Implementation |
|---|---|
| Free tier everywhere it exists | App Service F1 (dev), Cosmos DB free, Log Analytics ≤5GB, Defender free |
| Expensive resources are demo-only | App Gateway, VPN Gateway, Bastion — deployed via Terraform only when demoing, destroyed after |
| Dev shuts down nights + weekends | Logic App on schedule, tag-driven |
| Single region | Canada Central only, no geo-redundancy |
| Hard tripwires | Budget alerts at 50/80/100% with action group |

See [cost-report.md](./cost-report.md) for daily actuals.

## Build plan

### Week 1 — foundation
- **Day 1** — Bootstrap, state backend, OIDC, budget alerts ✅
- **Day 2** — Management groups, Azure Policy, tag taxonomy
- **Day 3** — Hub VNet, Log Analytics, Key Vault, private DNS zones
- **Day 4** — Prod spoke + App Service + MySQL + private endpoints
- **Day 5** — Dev spoke (mirrored, smaller SKUs), auto-shutdown schedule
- **Day 6** — Demo app v1 (Node/Express + React), CI/CD wired up
- **Day 7** — Buffer, screenshots, docs

### Week 2 — differentiators
- **Day 8** — App Gateway WAF v2, custom rules, the 3600s timeout pattern
- **Day 9** — RAG chatbot (Cosmos vector store, embeddings, chat widget)
- **Day 10** — AI pipeline reviewer (GitHub Action that comments on plans)
- **Day 11** — FinOps copilot (Function → Teams webhook)
- **Day 12** — Observability workbook, alert rules, runbooks
- **Day 13** — Documentation polish, architecture diagram, recorded demo
- **Day 14** — `terraform destroy` rehearsal, final teardown, blog post

## Decisions

Every non-obvious choice is documented as an ADR (Architecture Decision Record):

- [ADR-0001 — GitHub Actions over Azure DevOps](./decisions/0001-github-actions-over-azdo.md)
- [ADR-0002 — Single region (Canada Central)](./decisions/0002-single-region.md)
- [ADR-0003 — Microsoft Teams over Slack](./decisions/0003-teams-over-slack.md)
- [ADR-0004 — Cost-engineering principles](./decisions/0004-cost-engineering.md)

(More added as we go.)

## Portfolio evidence

- Architecture diagram (PNG + draw.io source) — `/diagrams`
- Recorded walkthrough videos — `/videos`
- Screenshots of compliance dashboards, cost reports, AI demos — `/portfolio-evidence`
- Blog-post-ready writeup — `/blog`

## Author

**Victor Ugbor** — Senior Cloud Infrastructure Engineer
- [LinkedIn](https://linkedin.com/in/victorugbor) *(update with real URL)*
- [Terraform video series](https://linkedin.com/in/victorugbor) *(update with real URL)*

## License

MIT. See [LICENSE](./LICENSE).
