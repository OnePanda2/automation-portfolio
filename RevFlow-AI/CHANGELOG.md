# Changelog

## v1.0.0 — Initial public release

First public release of RevFlow AI: a 6-workflow B2B lead-to-cash automation MVP built on Activepieces, covering lead capture through opportunity creation.

### Workflows implemented

1. **Lead Capture Gateway** — validates and normalizes inbound leads, upserts by email so duplicate submissions update the existing record instead of creating a new one
2. **Lead Qualification** — AI-scored fit/intent using Activepieces' built-in AI provider, with deterministic business-rule post-processing for priority tier and territory
3. **Lead Enrichment** — firmographic lookup with a safe fallback on vendor failure (fail-open, never blocks the pipeline)
4. **Lead Routing** — high-priority leads go directly to the territory-owning Account Executive; everything else round-robins across the SDR pool
5. **Sales Notification** — Slack alert to the assigned rep with score, priority, and AI reasoning attached
6. **Opportunity Creation** — creates an Opportunity record only when a lead lands with an AE, not an SDR

### Notes on this release

- Workflows 1–6 are chained via webhook-to-webhook cascades, using **URL query parameters** rather than a JSON body for the handoff payload — see [docs/troubleshooting.md](docs/troubleshooting.md) for why.
- Workflow 3 (Enrichment) is built with a real HTTP call structure against a placeholder firmographic API key; it ships with a tested fail-open fallback so the pipeline runs end-to-end without a live key.
- Workflow 2 (Qualification) uses a live AI call (not a placeholder) — this project's Activepieces account has built-in AI credits available, so lead scoring reflects a real model, not mock data.
- All six workflows have been tested end-to-end against a live Activepieces instance and verified directly against Airtable record state (not just execution-success status) across every branch: new lead, existing/duplicate lead, disqualification, AE routing (two territories), SDR round-robin routing, and both the Opportunity-created and Opportunity-skipped paths.
- Two real bugs were found and fixed during testing — see [docs/troubleshooting.md](docs/troubleshooting.md) for both, since they're informative failure modes for anyone extending this project.
