# ADR-0002 — Single region (Canada Central)

**Status:** Accepted
**Date:** 2026-05-14

## Context

Tradogram production runs across three Azure regions (Canada Central, Central US, West Europe) with Front Door routing. The author has experience designing multi-region. The flagship project must decide whether to mirror that pattern or simplify.

## Decision

Deploy **only to Canada Central**. No multi-region, no Front Door, no geo-replicated databases.

## Consequences

### Positive
- Egress costs ≈ $0 (intra-region traffic is free).
- Removes Front Door Premium (~$165/mo base + $0.22/GB) from the bill — single largest cost saving.
- Removes MySQL geo-redundant backup storage charges.
- Simplifies the demo: one diagram, one set of resources, one IP space.
- Canada Central is the author's known territory — lower risk of region-specific surprises.

### Negative
- Cannot demonstrate HA across regions in the live environment.
- Cannot demonstrate Front Door WAF + Premium routing rules in the deployed system.

### Mitigations
- A second ADR (TBD) will document the multi-region design as a *paper architecture* in `/diagrams/multi-region-future.png`, with Terraform code scaffolded but not deployed — proving the author can design it without paying to run it.
- App Gateway WAF v2 deployed in the single region preserves the WAF demo and directly resolves the Front Door 240-second timeout issue from Tradogram (which becomes the talking point).

## Alternatives considered

| Option | Why rejected |
|---|---|
| Two regions (CA + US) with Front Door | Adds ~$170/mo, would consume the entire $200 credit in a week |
| Two regions, no Front Door, DNS-based failover | Still doubles compute cost, marginal demo value |
| Different single region (e.g., East US) | Canada Central matches author's known good config |
