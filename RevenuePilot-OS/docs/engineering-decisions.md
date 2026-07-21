# Engineering Decisions

This document explains *why* RevenuePilot OS is built the way it is, not just what it does. It's written for another engineer evaluating whether these choices were reasonable, and for future-me revisiting the project after time away from it.

## Why Activepieces over custom code

The alternative to a visual automation platform here is a small set of serverless functions (AWS Lambda, Cloud Functions) wired together with a queue or direct HTTP calls. That's a reasonable design too, and for a team with existing cloud infrastructure it might be the better one. Activepieces was chosen for this project because:

- It gives a visual, inspectable trace of every execution — every step's input and output is retained and viewable, which is what made a project like this debuggable without adding custom logging everywhere.
- Native code steps (JavaScript) are available wherever the built-in pieces aren't expressive enough, so the "visual builder" constraint never actually blocked any of the logic in this project.
- It has first-class pieces for every external service this project touches (Airtable, Gmail, Slack), which removed a meaningful amount of integration boilerplate.
- It's genuinely free to run at this project's scale, which matters for a portfolio build.

## Why Airtable

Airtable is not the "right" system of record for a production RevOps stack — a real CRM (HubSpot, Salesforce) or a relational database would be. It was chosen here specifically because:

- It has a real REST API with per-field addressability, which is what most CRMs also offer, so workflow logic written against Airtable's field-update pattern translates directly to a CRM swap later.
- Schema changes (adding a field, changing a select's options) are a UI action, not a migration — useful for a project that iterated on its schema constantly during development.
- Zero cost, zero hosting, which matters for a portfolio project with no production traffic to justify infrastructure spend.

**Tradeoff accepted:** Airtable's API has real limitations that shaped this project's design — notably, there is no tool to add a table to an *existing* base via the API used here, which is why several workflows (Territory Rules, CSM Pool, Reports, Failures) each live in their own small dedicated base rather than one unified schema. In a real CRM this would be one schema with proper relations.

## Why modular workflows instead of one flow

Workflows 1–6 form a single logical pipeline (a lead enters, and by the end it's been deduplicated, enriched, scored, routed, and announced) but are built as six independently-triggered Activepieces flows, chained by each one calling the next workflow's webhook on success.

The alternative — one large flow with all this logic inline — was rejected because:

- **Independent testability.** Each workflow can be triggered and inspected on its own, with its own execution history, without needing to replay the entire pipeline to debug one stage.
- **Blast radius.** A bug or a broken external connection in one stage (say, the enrichment vendor going down) doesn't prevent editing or redeploying any other stage.
- **Reusability.** Workflow 9 (Error Monitoring) is called from every other workflow's failure path — that's only possible because it's a separate, independently-addressable flow, not inline logic duplicated six times.

**Tradeoff accepted:** this design requires each workflow to explicitly forward the full current state (the CRM record's fields, not just what it personally changed) to the next workflow — a subtlety that caused real bugs during development (see the "field propagation" note below) and requires discipline to keep every workflow's outbound payload complete and consistently keyed.

## Why webhook-to-webhook chaining specifically

Given modular workflows, the options for connecting them were: (a) webhook-to-webhook HTTP calls, as built, (b) a shared queue/event bus, or (c) polling. Webhook chaining was chosen because Activepieces makes it a one-step HTTP action, requires no additional infrastructure, and gives each downstream workflow its own real, independently-testable trigger. The cost is that failures in the HTTP call itself (not the downstream workflow's logic) need their own handling — every cascade step in this project is configured to continue on failure rather than block the calling workflow's own response to its caller.

## Why AI qualification before routing, not after

Lead scoring happens before territory routing so that routing rules can (in principle) key off AI-derived fields like Priority, not just raw enrichment data. In this build the routing rules only use enrichment fields (company size, location), but the ordering keeps that option available without restructuring the pipeline later.

## Why round-robin CSM assignment uses live open-count, not a rotating index

A naive round-robin (cycle through CSMs 1, 2, 3, 1, 2, 3...) breaks the moment a CSM goes on leave, gets manually assigned extra accounts outside the system, or the team roster changes. This project instead tracks each CSM's *current open onboarding count* and assigns to whoever has the fewest — a form of load-based assignment that self-corrects without needing to know the assignment history.

## Why every external AI/vendor call has an explicit fallback, not a retry-until-success

Enrichment (workflow 3), AI qualification (workflow 4), report summarization (workflow 8), and the revenue assistant (workflow 10) all call third-party APIs that can fail for reasons entirely outside this system's control (vendor downtime, rate limits, invalid keys). Each of these is built with:

- The real API call structure (correct endpoint, auth header, request shape) — so the code requires no changes to go live with a real key
- A try/catch around that call
- An explicit, safe fallback value if the call fails, so the workflow completes instead of stalling

This was a deliberate choice over blocking-and-retrying indefinitely: a lead should still get created, scored (even if by default), routed, and announced even if the AI vendor is down. The alternative — halting the pipeline until the AI call succeeds — would mean a vendor outage stops new leads from being processed at all.

## Tradeoffs accepted, summarized

| Decision | What was gained | What was given up |
|---|---|---|
| Airtable over a real CRM/DB | Fast iteration, zero cost | No table-add-to-existing-base API, forced several small dedicated bases |
| Modular workflows over one flow | Independent testability, reusable error handling | Requires explicit, consistent field propagation between workflows |
| Webhook chaining over a queue | No added infrastructure, simple to build | Cascade-call failures need explicit continue-on-failure handling |
| Fallback-on-failure over retry-until-success | Pipeline never stalls on a vendor outage | Some records will carry default/placeholder values (e.g. Lead Score 0) that need periodic backfill once real API keys are added |
| Single-page fetch in reporting/assistant | Simple, no pagination logic needed | Known scaling ceiling at ~100 records per table |

## Future scalability

The architecture doesn't need to change shape to scale — a real CRM swap, pagination in the two read-heavy workflows (8 and 10), and a proper retry queue are additive changes, not redesigns. See [README.md's Future Improvements](../README.md#future-improvements) for the full list.
