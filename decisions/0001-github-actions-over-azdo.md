# ADR-0001 — GitHub Actions over Azure DevOps

**Status:** Accepted
**Date:** 2026-05-14

## Context

The flagship project needs CI/CD. The author runs Azure DevOps daily at Tradogram and has built mirror pipelines, multi-stage deploys, and the Beanstalk → AZDO migration. AZDO is the "matches the day job" choice. GitHub Actions is the alternative.

## Decision

Use **GitHub Actions** for all pipelines in this project.

## Consequences

### Positive
- Workflows are public, in `.github/workflows/`, directly visible to anyone reviewing the portfolio. AZDO requires giving access to a project, which interviewers will not do.
- Unlimited free minutes on public repos vs 1,800/mo on AZDO free tier — material for a project with many short PR-plan runs.
- GitHub Actions is the de facto standard outside legacy Microsoft shops; broader job-market signal than AZDO-only.
- Reusable workflows + composite actions are first-class, enabling a `flagship-actions` repo that mirrors AZDO YAML templates.
- OIDC federation to Azure is identical to AZDO Workload Identity Federation, so the "zero long-lived secrets" story is preserved.

### Negative
- Diverges from the author's day-job toolchain.
- Some advanced features (environment approvals UX, parallel job orchestration) are slightly less polished in GH Actions than AZDO.
- No native equivalent to AZDO's "deploy job to deployment group" — workaround uses environments + matrix.

### Mitigations
- Interview narrative: *"I run AZDO daily at Tradogram, and I built the flagship project on GitHub Actions to demonstrate breadth. Same OIDC pattern, same reusable templates, different YAML dialect."*
- The Tradogram pipeline work (phase1, api1, mirror) is still on the CV; this project complements rather than replaces.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Azure DevOps | Hidden from public, narrower market signal, fewer free minutes for public OSS |
| GitLab CI | Author has no GitLab experience to leverage; would slow the build |
| Jenkins | Self-hosted runner adds infra cost and complexity outside the scope |
