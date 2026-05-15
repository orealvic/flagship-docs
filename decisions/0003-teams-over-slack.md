# ADR-0003 — Microsoft Teams over Slack

**Status:** Accepted
**Date:** 2026-05-14

## Context

The FinOps copilot, Key Vault expiry alerts, and pipeline notifications need a chat target. The author uses Slack at Tradogram. The candidate target enterprises (Home Hardware, BMO, iQmetrix) are Microsoft shops running M365.

## Decision

Use **Microsoft Teams** with Incoming Webhooks. Slack notifications are not implemented in the flagship project.

## Consequences

### Positive
- Demonstrates the M365 + Azure integration pattern that target employers actually run.
- Adaptive Cards format (Teams) is materially richer than Slack blocks for tabular cost reports.
- Incoming Webhooks are free, no app registration, no OAuth flow.
- Aligns the AI FinOps copilot demo with what an enterprise customer would actually deploy.

### Negative
- Author has more Slack experience; small ramp-up on Teams webhook format.
- Free personal Teams (without M365 Dev tenant) does not support interactive bots — only one-way webhook posts.
- For the "ask the FinOps copilot a question" interaction, must use a small web UI in the demo app instead of an in-Teams bot.

### Mitigations
- Web UI is hosted in the existing demo app — no additional infra cost.
- Adaptive Card payload format is documented in `flagship-ai/README.md` for the next-engineer story.
- If a richer integration becomes useful later, signing up for the free M365 Developer Program (90-day renewable tenant) takes 10 minutes and enables full bot support.

## Alternatives considered

| Option | Why rejected |
|---|---|
| Slack (free workspace) | Generic for Microsoft-shop target market |
| Discord | Wrong signal for enterprise cloud roles |
| Email only | Misses the chatops demo entirely |
| M365 Dev tenant from day 1 | Sign-up adds friction; can be added later without rework |
