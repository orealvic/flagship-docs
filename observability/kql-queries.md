# KQL Query Library

A curated reference of KQL queries for observability, security monitoring, cost governance, and audit on Azure environments. Organized for daily operational use.

These queries were developed for the Flagship landing zone and adapted from patterns proven in production SaaS operations. Resource names in examples are Flagship-specific — adapt resource group names, tag values, and identifiers to your environment.

## How this library is organized

Queries are tagged with three dimensions so you can find what you need quickly:

- **Purpose** — what category of question the query answers (Drift, Security, Health, Cost, Audit)
- **Cadence** — how often it should be run (Daily, Weekly, Monthly, On-Demand)
- **Source** — which Azure data plane the query reads from (Resource Graph, Log Analytics, Activity Logs, Monitor Metrics)

Each query includes:
- When to use it
- The KQL itself
- What the output means
- What action to take based on the result

---

## The daily 10-minute routine

Five queries to run every morning. Total time: under 10 minutes. Catches 80% of operational signals.

| # | Query | Source | Why |
|---|---|---|---|
| 1 | [Recently created resources (24h)](#1-recently-created-resources) | Resource Graph | Shadow IT detection |
| 2 | [Resource deletions (24h)](#13-resource-deletions) | Activity Logs | Change awareness |
| 3 | [Failed sign-ins (24h)](#11-failed-sign-ins) | SigninLogs | Security signal |
| 4 | [App Service 5xx errors (24h)](#14-app-service-5xx-errors) | Log Analytics | Customer impact |
| 5 | [MySQL connection counts (24h)](#22-mysql-connection-trend) | Monitor Metrics | DB health trending |

If you can only do one thing per day, do #4 (App Service 5xx). Customer-facing errors trump everything else.

---

# 1. Drift detection (Resource Graph)

Queries that answer "what changed in my environment?" Useful for catching unauthorized provisioning, shadow IT, and configuration drift.

## 1. Recently created resources

**Cadence:** Daily | **Source:** Resource Graph

**When to use:** Every morning. Catches resources provisioned outside your IaC pipelines.

```kql
Resources
| where todatetime(properties.timeCreated) > ago(24h)
| project name, type, resourceGroup, location, 
          createdTime=properties.timeCreated, 
          tags
| order by createdTime desc
```

**What the output means:** A list of all resources created in the last 24 hours. Every entry should be traceable to a Terraform apply or an approved manual operation.

**Action:** Any resource you don't recognize → investigate. Anything tagged `managed-by=manual` that shouldn't be → root-cause and consider blocking with policy.

## 2. Recently modified resources

**Cadence:** Daily | **Source:** Resource Graph

**When to use:** When investigating suspected configuration drift or after a release window.

```kql
Resources
| where todatetime(properties.timeChanged) > ago(24h)
| project name, type, resourceGroup, location, 
          changedTime=properties.timeChanged, 
          tags
| order by changedTime desc
```

**What the output means:** Resources whose configuration changed in the last 24 hours.

**Action:** Cross-reference against expected changes (releases, ticket activity). Unexpected NSG or RBAC changes warrant immediate investigation.

## 3. Untagged resources

**Cadence:** Weekly | **Source:** Resource Graph

**When to use:** Weekly governance review. Confirms tagging compliance for cost allocation.

```kql
Resources
| where isempty(tags) or 
        tags !has 'environment' or 
        tags !has 'cost-center'
| where type !in~ (
    'microsoft.network/virtualnetworks/subnets',
    'microsoft.compute/virtualmachines/extensions',
    'microsoft.network/privatednszones/virtualnetworklinks'
)
| project name, type, resourceGroup, location, tags
| order by type asc
```

**What the output means:** Resources missing required tags. Child resources (subnets, VM extensions, DNS zone links) are excluded because they inherit tags from parents.

**Action:** Each entry is a tagging gap. Backfill tags via Terraform or remediate via Azure Policy `Modify` effect.

## 4. Resources in unexpected regions

**Cadence:** Weekly | **Source:** Resource Graph

**When to use:** Monthly governance review. Catches misprovisioned or compromised resources.

```kql
Resources
| where location !in (
    'canadacentral', 'canadaeast', 'eastus', 'eastus2', 
    'centralus', 'westeurope', 'global'
)
| project name, type, resourceGroup, location
```

**What the output means:** Resources outside your approved regions.

**Action:** Should be empty. Anything that appears → investigate. Enforce with `allowed-locations` Azure Policy assignment.

## 5. Resource group sprawl

**Cadence:** Monthly | **Source:** Resource Graph

**When to use:** Monthly cleanup planning.

```kql
Resources
| summarize 
    resourceCount=count(),
    distinctTypes=dcount(type)
  by resourceGroup
| order by resourceCount desc
```

**What the output means:** Resource groups ranked by resource count and diversity. Groups with hundreds of resources of dozens of types are operational debt.

**Action:** Top offenders are candidates for resource group splits or cleanup projects.

---

# 2. Security signals (Log Analytics, Resource Graph)

Queries that answer "is anything suspicious happening?" Critical for SOC 2 / ISO 27001 audit posture.

## 6. NSGs with broad inbound access

**Cadence:** Weekly | **Source:** Resource Graph

**When to use:** Weekly security review. Common audit finding source.

```kql
Resources
| where type =~ 'microsoft.network/networksecuritygroups'
| mv-expand rule = properties.securityRules
| where rule.properties.access =~ 'Allow' 
   and rule.properties.direction =~ 'Inbound'
   and (rule.properties.sourceAddressPrefix in ('*', '0.0.0.0/0', 'Internet') 
        or rule.properties.sourceAddressPrefix has '0.0.0.0/0')
| project nsgName=name, resourceGroup, 
          ruleName=tostring(rule.name),
          port=rule.properties.destinationPortRange,
          source=rule.properties.sourceAddressPrefix
```

**What the output means:** NSG rules permitting inbound traffic from the public internet. Each row is a potential audit finding.

**Action:** Justify each rule (Front Door, Application Gateway). Remove rules with no clear business purpose. Document approved rules in flagship-docs.

## 7. Storage accounts allowing public access

**Cadence:** Weekly | **Source:** Resource Graph

**When to use:** Weekly. Highest-risk misconfiguration in Azure.

```kql
Resources
| where type =~ 'microsoft.storage/storageaccounts'
| where properties.allowBlobPublicAccess == true 
   or properties.publicNetworkAccess =~ 'Enabled'
| project name, resourceGroup, location,
          publicBlobAccess=properties.allowBlobPublicAccess,
          publicNetwork=properties.publicNetworkAccess,
          tags
```

**What the output means:** Storage accounts exposed to the internet. Should be near-zero for production SaaS.

**Action:** Disable public access. If business reason exists, document it as an ADR with the risk acceptance.

## 8. Key Vaults missing diagnostic settings

**Cadence:** Monthly | **Source:** Resource Graph (with join)

**When to use:** Before audits. Verifies security-relevant resources are logging.

```kql
Resources
| where type in~ (
    'microsoft.keyvault/vaults',
    'microsoft.storage/storageaccounts',
    'microsoft.web/sites',
    'microsoft.network/applicationgateways',
    'microsoft.dbformysql/flexibleservers'
)
| join kind=leftouter (
    Resources
    | where type =~ 'microsoft.insights/diagnosticsettings'
    | extend parentId = tostring(properties.resourceUri)
    | project parentId, diagnosticEnabled=true
) on $left.id == $right.parentId
| where isnull(diagnosticEnabled)
| project name, type, resourceGroup
```

**What the output means:** Critical resources without diagnostic settings — auditors will flag these.

**Action:** Add diagnostic settings via Terraform. Enforce via Azure Policy `DeployIfNotExists`.

## 9. Public IP addresses inventory

**Cadence:** Weekly | **Source:** Resource Graph

**When to use:** Periodic attack surface review.

```kql
Resources
| where type =~ 'microsoft.network/publicipaddresses'
| extend associated = isnotnull(properties.ipConfiguration)
| extend ip = tostring(properties.ipAddress)
| project name, resourceGroup, location, ip, associated, sku=sku.name, tags
| order by associated desc, location asc
```

**What the output means:** Every public IP your subscription owns. Unassociated IPs are wasted spend AND a reminder of past experiments.

**Action:** Unassociated IPs → delete. Associated IPs → confirm each is intentional (App Gateway, VPN Gateway, Bastion).

## 10. Privileged role assignments

**Cadence:** Weekly | **Source:** Log Analytics (AuditLogs)

**When to use:** Weekly security review.

```kql
AuditLogs
| where TimeGenerated > ago(7d)
| where OperationName has "Add member to role"
| where TargetResources has_any (
    "Owner", "Contributor", "User Access Administrator", 
    "Security Administrator", "Global Administrator"
)
| project TimeGenerated, OperationName, 
          User=InitiatedBy.user.userPrincipalName,
          Role=tostring(TargetResources[0].displayName),
          Target=tostring(TargetResources[0].userPrincipalName)
```

**What the output means:** Every privileged role grant in the last 7 days.

**Action:** Each entry should be traceable to an approved ticket or PIM activation. Anything unexpected → investigate immediately.

## 11. Failed sign-ins

**Cadence:** Daily | **Source:** Log Analytics (SigninLogs)

**When to use:** Daily routine. Brute force / compromise indicator.

```kql
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"  // 0 = success
| summarize failedAttempts=count() by 
    UserPrincipalName, 
    ResultType, 
    ResultDescription, 
    AppDisplayName
| where failedAttempts > 5
| order by failedAttempts desc
```

**What the output means:** Users with more than 5 failed sign-in attempts in 24 hours.

**Action:** Investigate spikes. >50 failures from one account → potential compromise, force MFA reset. >100 from one IP → possible brute force, consider conditional access lockout.

## 12. Sign-in failure sources by IP

**Cadence:** Weekly | **Source:** Log Analytics (SigninLogs)

**When to use:** Pattern detection across the week.

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType != "0"
| summarize failed=count() by IPAddress, Location
| where failed > 100
| order by failed desc
```

**What the output means:** IP addresses generating sustained failed sign-in activity. Often bots or attempted brute force.

**Action:** Consider IP blocking via Conditional Access. Document any IP > 1000 failures in security incident log.

## 13. Resource deletions

**Cadence:** Daily | **Source:** Log Analytics (AzureActivity)

**When to use:** Daily routine. Catches accidental and unauthorized deletions.

```kql
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue endswith "/delete"
| where ActivityStatusValue =~ "Success"
| project TimeGenerated, Caller, ResourceProviderValue, 
          ResourceId, OperationNameValue
| order by TimeGenerated desc
```

**What the output means:** Successful resource deletions in the last 24 hours.

**Action:** Every deletion should be traceable to a Terraform destroy or approved cleanup task. Unexpected deletions → investigate immediately, restore from backup if possible.

---

# 3. Production health (Log Analytics, Monitor Metrics)

Queries that answer "is the application working?" Customer-facing signals.

## 14. App Service 5xx errors

**Cadence:** Daily | **Source:** Log Analytics (AppServiceHTTPLogs)

**When to use:** Daily routine. Customer impact indicator.

```kql
AppServiceHTTPLogs
| where TimeGenerated > ago(24h)
| where ScStatus >= 500
| summarize errors=count() by 
    _ResourceId, 
    CsUriStem, 
    ScStatus, 
    bin(TimeGenerated, 1h)
| where errors > 10
| order by TimeGenerated desc
```

**What the output means:** App Service responses with HTTP 500+ status codes, grouped by endpoint and hour.

**Action:** Any 5xx > 10/hour on a customer endpoint → investigate. Look at the Application Insights `exceptions` table for stack traces.

## 15. App Service request latency p95

**Cadence:** Daily | **Source:** Log Analytics (AppServiceHTTPLogs)

**When to use:** Performance trending.

```kql
AppServiceHTTPLogs
| where TimeGenerated > ago(24h)
| where ScStatus < 500
| summarize 
    p95 = percentile(TimeTaken, 95),
    p99 = percentile(TimeTaken, 99),
    requestCount = count()
  by _ResourceId, bin(TimeGenerated, 1h)
| order by p95 desc
```

**What the output means:** 95th and 99th percentile latency per App Service per hour. P95 > 2000ms sustained is a degradation signal.

**Action:** Investigate latency spikes. Common causes: slow MySQL queries, cold-start issues, downstream service degradation, cache misses.

## 16. MySQL slow queries

**Cadence:** Daily | **Source:** Log Analytics (AzureDiagnostics)

**When to use:** Daily DB health check. Requires MySQL slow query log shipping to be enabled.

```kql
AzureDiagnostics
| where Category =~ "MySqlSlowLogs"
| where TimeGenerated > ago(24h)
| where toint(query_time_d) > 5
| project TimeGenerated, _ResourceId, 
          query=sql_text_s, 
          duration_sec=query_time_d, 
          rows_examined_s
| order by toreal(duration_sec) desc
| take 50
```

**What the output means:** Slowest 50 queries in the last 24 hours. Queries taking >5 seconds.

**Action:** Cross-reference with application logs to find calling code path. Common fixes: missing indexes, full table scans, suboptimal joins, lock contention.

## 17. Front Door backend health

**Cadence:** Daily | **Source:** Monitor Metrics (via portal or KQL)

**When to use:** When customers report regional issues.

```kql
AzureMetrics
| where ResourceProvider =~ "MICROSOFT.CDN" or ResourceProvider =~ "MICROSOFT.NETWORK"
| where MetricName == "BackendHealthPercentage"
| where TimeGenerated > ago(24h)
| summarize avgHealth=avg(Average) by Resource, bin(TimeGenerated, 15m)
| where avgHealth < 100
| order by TimeGenerated desc
```

**What the output means:** Front Door backend health dropping below 100%.

**Action:** Sub-95% sustained → investigate the backend, check origin App Service health, NSG rules, certificates.

---

# 4. Cost governance (Resource Graph, Cost Management API, Monitor Metrics)

Queries that answer "where is money going?" Critical for FinOps.

## 18. Unassociated public IPs

**Cadence:** Monthly | **Source:** Resource Graph

**When to use:** Monthly orphan cleanup.

```kql
Resources
| where type =~ 'microsoft.network/publicipaddresses'
| where isnull(properties.ipConfiguration)
| project name, resourceGroup, location, sku=sku.name, tags
```

**What the output means:** Public IPs not attached to any resource. Each costs ~$4 CAD/month.

**Action:** Delete each one. They serve no purpose.

## 19. Unattached managed disks

**Cadence:** Monthly | **Source:** Resource Graph

**When to use:** Monthly orphan cleanup.

```kql
Resources
| where type =~ 'microsoft.compute/disks'
| where properties.diskState =~ 'Unattached'
| project name, resourceGroup, location, 
          sizeGB=properties.diskSizeGB, 
          sku=sku.name, 
          tags
| order by toint(sizeGB) desc
```

**What the output means:** Disks not attached to any VM. Costs accrue per GB-month.

**Action:** Verify nothing depends on them (sometimes used as data disks waiting for re-attach), then delete or snapshot+delete.

## 20. Stopped (not deallocated) VMs

**Cadence:** Daily | **Source:** Resource Graph

**When to use:** Daily. Catches the most common Azure cost mistake.

```kql
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend powerState = tostring(properties.extended.instanceView.powerState.displayStatus)
| where powerState == "VM stopped"
| project name, resourceGroup, location, powerState, tags
```

**What the output means:** VMs that were stopped via OS shutdown rather than deallocated via Azure. **You're still being charged for compute.**

**Action:** Use `az vm deallocate` (not `az vm stop`) to stop billing. Configure auto-shutdown to deallocate, not stop.

## 21. Empty App Service plans

**Cadence:** Monthly | **Source:** Resource Graph

**When to use:** Monthly cleanup.

```kql
Resources
| where type =~ 'microsoft.web/serverfarms'
| extend numberOfSites = toint(properties.numberOfSites)
| where numberOfSites == 0
| project name, resourceGroup, location, sku=sku.name, tier=sku.tier, tags
```

**What the output means:** App Service plans with zero hosted sites. Still billed at full tier rate.

**Action:** Delete the plan. If kept for a planned site, downgrade to a smaller SKU.

## 22. MySQL connection trend

**Cadence:** Daily | **Source:** Monitor Metrics

**When to use:** Daily DB health.

```kql
AzureMetrics
| where ResourceProvider =~ "MICROSOFT.DBFORMYSQL"
| where MetricName == "active_connections"
| where TimeGenerated > ago(24h)
| summarize 
    avgConnections=avg(Average), 
    maxConnections=max(Maximum) 
  by Resource, bin(TimeGenerated, 1h)
| order by Resource, TimeGenerated
```

**What the output means:** Active connection counts per MySQL Flex server, hourly.

**Action:** Connection exhaustion is a top production incident cause. Trending up week-over-week → investigate connection pooling, leaked connections, application restarts. Approaching max_connections limit → tune `max_connections` parameter.

---

# 5. Audit and compliance (Log Analytics)

Queries that produce evidence for SOC 2, ISO 27001, and other audits.

## 23. RBAC role assignment changes

**Cadence:** Weekly | **Source:** Log Analytics (AuditLogs)

**When to use:** Weekly audit-evidence collection.

```kql
AuditLogs
| where TimeGenerated > ago(7d)
| where Category =~ "RoleManagement"
| project TimeGenerated, OperationName, 
          InitiatedBy=tostring(InitiatedBy.user.userPrincipalName),
          target=tostring(TargetResources[0].displayName),
          result=tostring(Result)
```

**What the output means:** Every RBAC change in the last 7 days. Audit evidence.

**Action:** Export weekly to compliance evidence storage. Spot-check unexpected entries.

## 24. Service principal activity

**Cadence:** Monthly | **Source:** Log Analytics (AzureActivity)

**When to use:** Audit prep — proves you can trace automated identity actions.

```kql
AzureActivity
| where TimeGenerated > ago(30d)
| where Caller has "ServicePrincipal" or 
        isnotempty(tostring(parse_json(Properties).servicePrincipalId))
| summarize operations=count() by Caller, ResourceProviderValue
| order by operations desc
```

**What the output means:** Service principals ranked by activity. Useful for identifying which SPs are actively used vs candidates for cleanup.

**Action:** Document active SPs in flagship-docs runbook. Stale SPs → rotate or delete.

## 25. Key Vault access patterns

**Cadence:** Monthly | **Source:** Log Analytics (AzureDiagnostics)

**When to use:** Audit prep — proves you can trace secret access.

```kql
AzureDiagnostics
| where ResourceType =~ "VAULTS"
| where TimeGenerated > ago(30d)
| where OperationName !has "List"
| summarize accesses=count() by 
    Resource, 
    identity_claim_oid_g, 
    OperationName
| order by accesses desc
```

**What the output means:** Who accessed what secrets in the last 30 days, by operation type.

**Action:** Validate against expected service-principal access. Unknown OIDs → investigate. Unusual access spikes → investigate.

## 26. Resources accessed by anonymous identity

**Cadence:** Weekly | **Source:** Log Analytics (multiple tables)

**When to use:** Anomaly detection.

```kql
AzureDiagnostics
| where TimeGenerated > ago(7d)
| where identity_claim_oid_g == "" or isnull(identity_claim_oid_g)
| where OperationName !in ("GetCertificates", "ListCertificates")
| project TimeGenerated, Resource, OperationName, CallerIPAddress
| order by TimeGenerated desc
| take 100
```

**What the output means:** Operations performed without an authenticated identity.

**Action:** Should be near-zero for properly-configured environments. Anything appearing → investigate immediately.

---

# 6. Configuration drift (Resource Graph)

Queries that answer "does reality match my IaC?"

## 27. Resources without `managed-by=terraform` tag

**Cadence:** Weekly | **Source:** Resource Graph

**When to use:** Weekly governance. Catches manual provisioning.

```kql
Resources
| where type !in~ (
    'microsoft.network/virtualnetworks/subnets',
    'microsoft.compute/virtualmachines/extensions',
    'microsoft.network/privatednszones/virtualnetworklinks',
    'microsoft.web/sites/slots',
    'microsoft.web/sites/config'
)
| where tags !has 'managed-by' or tostring(tags['managed-by']) != 'terraform'
| project name, type, resourceGroup, managedBy=tags['managed-by']
| order by type asc
```

**What the output means:** Resources not declared as Terraform-managed. Each represents drift or manual creation.

**Action:** Either bring under Terraform management (preferred) or document as intentionally manual with an ADR.

## 28. Compute SKU inventory

**Cadence:** Monthly | **Source:** Resource Graph

**When to use:** Right-sizing review.

```kql
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| summarize count() by 
    vmSize=tostring(properties.hardwareProfile.vmSize), 
    location
| order by count_ desc
```

**What the output means:** Distribution of VM sizes across regions.

**Action:** Identify over-provisioning candidates. Cross-reference with Azure Advisor right-sizing recommendations.

## 29. Database tier inventory

**Cadence:** Monthly | **Source:** Resource Graph

**When to use:** Cost review.

```kql
Resources
| where type =~ 'microsoft.dbformysql/flexibleservers' 
   or type =~ 'microsoft.sql/servers'
| project name, resourceGroup, location, 
          sku=sku.name, tier=sku.tier, 
          tags
| order by sku asc
```

**What the output means:** All database servers with their SKUs.

**Action:** Verify non-prod environments aren't running on production-grade SKUs.

---

# 7. Useful one-offs (On-demand)

Queries you don't run on schedule but want available when needed.

## 30. Find a resource by partial name

**Source:** Resource Graph

**When to use:** "Where did I put that resource called..."

```kql
Resources
| where name contains 'finops'  // change this term
| project name, type, resourceGroup, location, tags
```

## 31. Resources in a specific resource group

**Source:** Resource Graph

**When to use:** Quick inventory of one RG.

```kql
Resources
| where resourceGroup =~ 'rg-flagship-platform-prod'  // change this
| project name, type, location, tags
| order by type asc
```

## 32. All resources with a specific tag value

**Source:** Resource Graph

**When to use:** "Show me everything in production."

```kql
Resources
| where tags['environment'] =~ 'production'  // change tag/value
| project name, type, resourceGroup, location
| order by type asc
```

---

## Operational notes

### Permissions required

| Query type | Required role |
|---|---|
| Resource Graph (most queries here) | Reader on subscription |
| Log Analytics (Signin, Activity, AppService logs) | Log Analytics Reader on workspace |
| Audit logs (AuditLogs table) | Security Reader at tenant scope |
| Monitor Metrics | Monitoring Reader on subscription |

### Running these queries

- **Azure Portal:** Resource Graph Explorer for Resource Graph queries; Log Analytics workspace for Log Analytics queries
- **Azure CLI:** `az graph query -q "..."` for Resource Graph
- **PowerShell:** `Search-AzGraph -Query "..."` for Resource Graph
- **REST API:** Direct calls via `https://management.azure.com/providers/Microsoft.ResourceGraph/resources`

### Saving queries for reuse

In Log Analytics workspace, save frequently-used queries via the "Save" button. Saved queries become accessible across the workspace and can be embedded in Workbooks.

In Resource Graph Explorer, pin queries to dashboards for one-click execution.

### From queries to alerts

Several of these queries should become Azure Monitor alert rules:

- Query #11 (Failed sign-ins > 50 in 24h)
- Query #13 (Any production resource deletion)
- Query #14 (App Service 5xx > 50/hour)
- Query #20 (Stopped-not-deallocated VMs)
- Query #22 (MySQL connections > 80% of max)

Alert rules let you stop running these queries manually — Azure runs them on your schedule and notifies you via Action Group when thresholds are exceeded.

---

## Maintenance

This library is a living document. As the environment evolves, queries need updating. Review quarterly:

- Are the daily routine queries still surfacing useful signal?
- Have new resource types appeared that need coverage?
- Have query patterns become more efficient?

Last reviewed: see commit history.

## See also

- `flagship-docs/decisions/` — Architecture decisions including observability strategy
- `flagship-docs/runbooks/` — Incident response procedures referencing these queries
- [Azure Monitor documentation](https://learn.microsoft.com/azure/azure-monitor/)
- [Resource Graph query language reference](https://learn.microsoft.com/azure/governance/resource-graph/concepts/query-language)
