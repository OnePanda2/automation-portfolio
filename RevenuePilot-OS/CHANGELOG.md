# Changelog

## v1.0.0 — Initial public release

First public release of RevenuePilot OS: a 10-workflow B2B lead-to-cash automation portfolio project built on Activepieces.

### Workflows implemented

1. **Lead Capture Gateway** — normalizes and validates inbound leads from multiple source shapes (website, Typeform, HubSpot, manual)
2. **Duplicate Detection** — dedupes by email, creates or updates the CRM record accordingly
3. **Lead Enrichment** — attaches firmographic data with a safe fallback on vendor failure
4. **AI Lead Qualification** — scores and prioritizes leads via an LLM, with a safe fallback on AI-call failure
5. **Territory Routing** — assigns territory and rep pool via an ordered, auditable rules table
6. **Slack Notification System** — audience-aware Slack alerts based on score and source
7. **Customer Onboarding** — round-robin CSM assignment, welcome email, and onboarding checklist on Closed Won
8. **Weekly Executive Report** — scheduled funnel/revenue metrics with an AI-generated narrative summary
9. **Error Monitoring** — centralized failure classification, logging, alerting, and escalation for all other workflows
10. **AI Revenue Assistant** — on-demand, validated natural-language querying over lead and reporting data

### Notes on this release

- Workflows 1–6 are chained via webhook-to-webhook cascades, each independently triggerable and testable.
- Workflows 3, 4, 8, and 10 are built with real third-party API call structures (enrichment vendor, OpenAI) but ship with placeholder credentials by default — every one of these has a tested, safe fallback path so the pipeline runs end-to-end without live keys. See [docs/setup.md](docs/setup.md) to add real credentials.
- All ten workflows have been tested end-to-end against a live Activepieces instance, including the full lead cascade (1→6), the onboarding flow (7), error escalation (9), and the assistant's fallback path (10).
