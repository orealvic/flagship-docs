# Build Log — VNet-Integrated Self-Hosted Runner

**Status:** Runner built and validated. Workflow wiring pending.

## Problem

The landing-zone pipeline failed `terraform plan` with `403 ForbiddenByRbac` when reading Key Vault secrets (`mysql-fqdn`, `mysql-admin-password`).

Root cause was **network line-of-sight, not RBAC**:

- Key Vaults are configured `public_network_access_enabled = false` (private endpoint only)
- GitHub-hosted runners (`ubuntu-latest`) run on the public internet
- The runner had no network path to the private endpoint
- Terraform's data-plane secret read during plan/refresh therefore failed

This surfaced on the diagnostics PR but is a pre-existing architectural gap — it would affect any landing-zone pipeline run, since the Day 3/4 resources were originally applied locally (where the operator had VNet line-of-sight) rather than from CI.

## Solution: ephemeral VNet-integrated self-hosted runner

Architecture chosen: **1B — ACI ephemeral runner, manually triggered.**

Considered alternatives:
- Always-on VM runner — rejected (cost, contradicts cost-engineering narrative)
- Full webhook autoscale (1C) — deferred (complexity; the autoscale layer is the least important part for solving the actual problem)
- Temporary KV IP allowlisting — rejected (fragile, weakens the private posture)

### Components (in `flagship-platform/platform/runner.tf`)

| Resource | Purpose |
|---|---|
| `azurerm_subnet.runner` (10.10.30.0/24) | Delegated to `Microsoft.ContainerInstance/containerGroups`, in the hub VNet |
| `azurerm_network_security_group.runner` | AllowVnetInbound + DenyInternetInbound |
| `azurerm_user_assigned_identity.runner` | The runner's Azure identity |
| `azurerm_role_assignment` ×2 | Contributor + Key Vault Secrets User (subscription scope) |
| `azurerm_container_group.runner` | ACI running the runner agent, VNet-integrated, gated behind `deploy_runner` toggle |

### Why it works

1. The private DNS zones (including `privatelink.vaultcore.azure.net`) were already linked to the hub VNet, so the runner resolves Key Vault names to private IPs.
2. Hub-to-spoke peering provides the network path from the runner subnet to the KV private endpoint in the prod spoke.
3. The managed identity holds Key Vault Secrets User, so the data-plane read is authorized.

### Cost model

- Persistent scaffolding (subnet, NSG, identity, role assignments): **free**
- Billable compute (ACI): only when `deploy_runner = true`; destroyed after use
- Idle cost: **zero**

### Auth model

- GitHub registration: short-lived registration token supplied at apply time (not a stored PAT)
- Azure auth: managed identity (MSI), no stored credentials
- Scope: repo-level (registers to flagship-platform)

## Validation

From inside the running ACI container, using only the managed identity:

1. Obtained a Key Vault-scoped token from the instance metadata endpoint (MSI) — **success**
2. Called the Key Vault REST API for `mysql-fqdn` over the private endpoint — **returned the secret value**

This proves all four layers end to end: MSI auth, private DNS resolution, hub-spoke network path, and RBAC. The runner read the exact secret that public runners get 403 on.

## Drift reconciled along the way

The runner plan surfaced pre-existing policy drift (manual changes made to Azure that weren't in code). Reconciled into `policy.tf`:

- `allowed-locations` policy: added `canadaeast` + `eastus` (Automation account requires eastus; the subscription tier won't permit Automation in canadacentral), plus a `not_scopes` exemption for `rg-flagship-prod-ai` (Azure OpenAI regional availability)
- Three tag-requirement policies: changed `not_scopes` exemption from the bootstrap RG to the automation RG

Plan after reconciliation: clean (`0 to change` on existing resources).

## Outstanding

1. **Workflow wiring** — parameterize the reusable workflow (`flagship-actions/.github/workflows/terraform.yml`) with a `runs_on` input so landing-zone routes to `[self-hosted, flagship-private]` while platform stays on `ubuntu-latest`; switch landing-zone's Azure auth to MSI.
2. **ADR-0006** — formalize the runner architecture decision and the production upgrade path (GitHub org → org-level runner → GitHub App auth → webhook autoscale).
3. **Runner teardown** — set `deploy_runner = false` when not actively running pipelines, to return cost to zero.

## Production upgrade path (for ADR-0006)

This implementation is repo-scoped with manual triggering — appropriate for a portfolio. At organizational scale:

- Host repos in a GitHub **org** → make the runner **org-level** (reusable across repos)
- Replace registration tokens with a **GitHub App** (short-lived installation tokens, programmatic registration)
- Add **webhook-driven autoscale** on the `workflow_job` event (architecture 1C) so runners spin up on demand and scale to zero
