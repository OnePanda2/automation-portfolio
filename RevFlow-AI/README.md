# RevFlow AI

A 6-workflow B2B lead-to-cash automation MVP built on Activepieces, covering lead capture through opportunity creation for a B2B SaaS company selling an AI-powered customer support platform.

## Portfolio highlights

- 6 production-tested Activepieces workflows, chained into one lead-to-cash pipeline
- AI-powered lead qualification using Activepieces' built-in AI provider — a live model call, not a mock
- Deterministic business rules layered on top of AI output (priority tiers, territory segmentation) — judgment calls go to the AI, fixed rules stay in code
- Modular, webhook-to-webhook architecture — every workflow independently triggerable and testable
- Airtable CRM integration across leads, sales rep pool, and opportunities
- Slack automation for real-time rep notification
- Two real bugs found and fixed during development, both documented with root cause — see [docs/troubleshooting.md](docs/troubleshooting.md)
- Full engineering documentation: architecture, setup, schema, troubleshooting, and decisions doc

## The problem

RevFlow AI (the fictional client for this project) grew quickly and now has marketing, sales, and customer success teams each running their own disconnected automations. Marketing says sales ignores leads; sales says lead quality is terrible; nobody trusts the data. The brief called for an "Automation Operating System" — not another one-off automation, but a system where every important business event automatically triggers the right next step, without an engineer manually fixing workflows every week.

This project delivers the MVP slice of that system: the full lead-to-cash core loop. A lead enters once, and six purpose-built workflows carry it through deduplication, AI-assisted qualification, enrichment, routing, notification, and — for leads that reach an Account Executive — opportunity creation.

## What it is not

This is not a CRM replacement, and it doesn't cover the client's full requested scope. The brief's stated success criteria also include customer onboarding, customer health monitoring, AI-powered ticket classification, and weekly executive reporting — all deliberately deferred to a later phase in favor of shipping the core revenue loop first (see [Future Improvements](#future-improvements)). Airtable stands in for the client's stated CRM (HubSpot) because no HubSpot/Postgres/Stripe connection was available to build against — see [engineering-decisions.md](docs/engineering-decisions.md#why-airtable-instead-of-the-clients-stated-stack-hubspot-postgres-stripe).

## Features

| # | Workflow | Trigger | Purpose | Business value |
|---|---|---|---|---|
| 1 | Lead Capture Gateway | Webhook | Validates inbound leads and upserts by email | One consistent entry point; duplicate submissions update, not duplicate, the record |
| 2 | Lead Qualification | Webhook | AI-scores fit/intent, then applies deterministic priority/territory rules | Consistent scoring without a human reviewing every lead |
| 3 | Lead Enrichment | Webhook | Firmographic lookup with a fail-open fallback | Enriches when possible without ever blocking the pipeline |
| 4 | Lead Routing | Webhook | High-priority → AE by territory; else → SDR round-robin | Hot leads reach a closer immediately; everything else still gets worked |
| 5 | Sales Notification | Webhook | Slack alert with score, priority, and AI reasoning | Reps know *why* a lead matters within seconds |
| 6 | Opportunity Creation | Webhook | Creates an Opportunity only for AE-assigned leads | Opportunity data reflects real pipeline state, not every lead ever routed |

## Workflow overview

### 1. Lead Capture Gateway
- **Trigger:** webhook, any lead source posts here
- **Steps:** validate/normalize required fields → find existing lead by email → branch update/create → get canonical record → forward to Qualification
- **External services:** Airtable
- **Result:** exactly one Leads record per email, contact info kept current on repeat submission

### 2. Lead Qualification
- **Trigger:** webhook, `?email=&companySize=&industry=`
- **Steps:** fetch the lead → AI scoring call (Activepieces built-in provider) → business-rule code step (priority tier, territory bucket, disqualify cutoff) → write back to Airtable → branch: Qualified continues, Disqualified stops here
- **External services:** Airtable, Activepieces AI (google/gemini-3.5-flash-lite)
- **Result:** every qualified lead carries a Score, Priority, Territory, and AI Reasoning

### 3. Lead Enrichment
- **Trigger:** webhook, `?email=`
- **Steps:** fetch the lead → derive company domain → call firmographic API → branch success/fail → update CRM → forward to Routing regardless of outcome
- **External services:** third-party enrichment API (placeholder key in this build — see [Setup](docs/setup.md))
- **Result:** enrichment data attached when available; the lead is never blocked on a vendor being down or unconfigured

### 4. Lead Routing
- **Trigger:** webhook, `?email=`
- **Steps:** fetch the lead → branch on Priority: High → assign the AE who owns the lead's Territory; Medium/Low → round-robin across the SDR pool → stamp a deal-value estimate → forward to Notification
- **External services:** Airtable
- **Result:** deterministic, auditable assignment — no ad hoc "whoever's free" routing

### 5. Sales Notification
- **Trigger:** webhook, `?email=`
- **Steps:** fetch the lead → post to Slack with rep name, lead context, score, and AI reasoning → mark Notified → forward to Opportunity Creation
- **External services:** Slack
- **Result:** the assigned rep hears about a lead within seconds, with the context to act on it immediately

### 6. Opportunity Creation
- **Trigger:** webhook, `?email=`
- **Steps:** fetch the lead → branch on Assigned Rep Role: AE → create an Opportunity record (Discovery stage, +45 day close estimate) and mark the lead Opportunity; SDR → no-op
- **External services:** Airtable
- **Result:** Opportunity data exists only for leads that have actually reached a closer

## System diagram

```
Lead source → 1. Lead Capture Gateway → 2. Lead Qualification (AI + rules)
                                              ↓ (Qualified only)
                                         3. Lead Enrichment (fail-open)
                                              ↓
                                         4. Lead Routing (AE-by-territory or SDR round-robin)
                                              ↓
                                         5. Sales Notification (Slack)
                                              ↓
                                         6. Opportunity Creation (AE-assigned leads only)
```

Each workflow is independently triggerable via its own webhook and independently testable — the pipeline is wired by having each workflow call the next one's webhook (via URL query parameters, not a JSON body — see [why](docs/troubleshooting.md#inter-workflow-handoff-arrives-empty)) rather than being one monolithic flow. Full diagram with data stores: [architecture/architecture.md](architecture/architecture.md).

## Tech stack

| Technology | Why |
|---|---|
| **Activepieces** | Visual builder with native code steps when logic needs more than a drag-and-drop action, first-class pieces for every external service this project touches, and (specific to this project) built-in AI credits that made real LLM-based scoring possible without a separate API key. |
| **Airtable** | Zero-cost, zero-infrastructure relational store standing in for the client's stated CRM (HubSpot), which wasn't available to build against. See [engineering-decisions.md](docs/engineering-decisions.md). |
| **Slack** | Real-time rep notification. |
| **Activepieces AI (Gemini Flash Lite)** | Lead scoring. Chosen for cost-efficiency against a limited AI-credit budget; the scoring prompt and business-rule split are provider-agnostic. |

## Installation

See [docs/setup.md](docs/setup.md) for the full setup guide.

## Repository layout

```
RevFlow-AI/
├── README.md                    — this file
├── LICENSE
├── CHANGELOG.md
├── workflows/                   — exported Activepieces flow JSON, one per workflow
├── screenshots/
│   └── workflow/                — clean canvas screenshots of each workflow
├── payloads/                    — example request/response payloads, captured from real executions
├── architecture/
│   └── architecture.md          — system diagram and data flow
└── docs/
    ├── setup.md                 — deployment guide
    ├── engineering-decisions.md — why, not just what
    ├── airtable-schema.md       — every table and field
    └── troubleshooting.md       — known failure modes and fixes, including two real bugs found during development
```

## Payload examples

Example payloads in [`/payloads`](payloads/) are captured from real executions run against this project's live Activepieces instance during development and testing — not hand-written. See each file's `_meta` block for the specific record/run it was captured from.

## Lessons learned

- **"The action worked" and "the action did the right thing" are different claims.** Every workflow in this project reported success on every test run, including the two that were silently broken — one repeatedly returned the wrong Airtable record, the other silently misrouted a lead. Both were only caught by checking actual Airtable state after each test, not by trusting execution-success status.
- **A platform action's name is not a guarantee of its behavior.** "Get Record by ID" sounds unambiguous; in this build it didn't filter by ID at all. Verify behavior with a deliberately-different test case (a second record in the table), not just a happy-path test with one record where a bug like this is invisible.
- **Deterministic logic belongs in code, not in a prompt.** Priority thresholds and territory bucketing are asked of a code step immediately after the AI call, not the model itself — a rule that should always give the same output for the same input shouldn't be probabilistic.
- **Fail-open beats fail-closed for third-party dependencies you don't control.** Enrichment failing (no real vendor key) doesn't stop a lead from being routed and announced — it just gets flagged. A vendor outage or a missing key shouldn't be able to stall revenue-generating activity.
- **Workflow modularity has a real cost, and query params paid it down.** Splitting into six independently-triggered workflows means data has to be explicitly re-sent between them — which is exactly what surfaced the inter-workflow handoff bug. Moving that handoff to URL query parameters instead of a JSON body fixed the reliability problem and, as a side effect, made every workflow re-fetch current state instead of trusting a possibly-stale forwarded copy.

## Future improvements

- **The remaining client success criteria** — customer onboarding, customer health monitoring, AI-powered ticket classification, and weekly executive reporting were all in the original brief and deliberately deferred to ship the core lead-to-cash loop first.
- **Real CRM integration** — swap Airtable for HubSpot as the system of record; the workflow logic (upsert by email, field updates, status transitions) maps directly onto it.
- **A formal Leads ↔ Opportunities relation** — currently a soft email-based reference rather than a linked-record field, a limitation of the table-creation tooling available for this build (see [engineering-decisions.md](docs/engineering-decisions.md)).
- **A live-queried Sales Reps roster** — routing currently uses a hardcoded rep list in a code step rather than reading the Sales Reps table directly; fine at 8 people, brittle if the roster changes often.
- **Secrets management** — the enrichment API key is a placeholder value wired into a workflow's HTTP step; a production deployment would pull this from a secrets manager rather than embedding it in workflow configuration.
- **Authentication on inbound webhooks** — the lead-capture and inter-workflow webhooks are currently unauthenticated by design (simplifies the portfolio build); production webhooks should validate a shared secret or signature.
- **Rate limiting** — none of the six workflows currently rate-limit inbound traffic; a public-facing lead-capture endpoint should have this in front of it.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

See [LICENSE](LICENSE).
