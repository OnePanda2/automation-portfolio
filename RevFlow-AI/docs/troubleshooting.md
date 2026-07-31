# Troubleshooting Guide

Known failure modes across RevFlow AI, their root causes, and how to fix them. The first two were found and fixed during this project's own development — they're documented here because they're genuinely useful failure modes to recognize, not hypothetical ones.

## "Get Record by ID" returns every record in the table

**Symptom:** a workflow that fetches a specific lead by its Airtable record ID instead returns *every* record in the Leads table. With only one record in the table this is invisible (the one record returned happens to be the right one); it becomes obvious once there are several — a workflow meant to process one lead silently picks whichever record happens to sit first in the returned array instead.

**Root cause:** Activepieces' Airtable "Get Record by ID" action, in this build, did not actually filter by the ID passed to it — it returned the full unfiltered table content regardless of the `recordId` input.

**Resolution:** use the "Find Record" action instead, searching by a specific field value (this project searches by Email) rather than by record ID. Verified independently via `ap_run_action` with literal values before and after the fix — "Find Record" correctly returns zero or one match for a given field value; "Get Record by ID" did not reliably isolate a single record.

## Inter-workflow handoff arrives empty

**Symptom:** an HTTP request step to a downstream workflow's webhook returns `200`, but the downstream workflow's trigger shows an empty body (`rawBody: ""`, no parsed `body` object) — and if the downstream workflow's logic falls back to matching "any" record when its expected identifier is blank (see the bug above), this compounds into **the wrong lead getting silently processed**, not just a visible failure.

**Root cause:** the JSON body sent by Activepieces' HTTP piece between two workflows in the *same* Activepieces instance did not reliably arrive at the receiving webhook — content-length showed as `0` on the receiving end often enough to be a real, reproducible problem, not a one-off network blip.

**Resolution:** pass the handoff data (this project only ever needs to pass one thing — the lead's email) as a **URL query parameter** on the webhook call instead of a JSON body, and read it via `queryParams` instead of `body` on the receiving end. Query parameters are part of the URL itself and don't depend on body parsing, so they don't have this failure mode. Every workflow in this project re-fetches the lead's current record fresh using that email rather than trusting any other data forwarded through the chain.

## AI scoring gives a different score on a re-run of the same lead

**Symptom:** re-submitting the exact same lead data produces a different Score (and potentially a different Priority tier or routing outcome) than an earlier run.

**Root cause:** this is expected LLM non-determinism, not a bug. The scoring prompt is deterministic; the model's output for a given prompt is not guaranteed to be identical across calls.

**Resolution:** none needed — this is a known characteristic to be aware of, not something to fix. If perfectly reproducible scoring is required for a specific use case, that would need either a temperature of 0 (reduces but doesn't eliminate variance) or moving the scoring logic to fully deterministic business rules instead of an LLM call.

## Company Size / Industry show blank on the Lead record after Qualification

**Symptom:** `Territory` defaults to `SMB` regardless of the actual company size submitted at intake, and `Company Size`/`Industry` fields are blank.

**Root cause:** these are free-text values at intake (e.g. `"700"`) that don't match the fixed dropdown choices on the Leads table, so Workflow 1 doesn't write them directly — they need to reach Workflow 2's normalization step. If they aren't threaded through the Workflow 1 → Workflow 2 handoff (e.g. only `email` is passed as a query parameter), Workflow 2's territory/bucket logic has nothing to compute from and silently falls back to its default (SMB, 1-10).

**Resolution:** confirm Workflow 1's handoff to Workflow 2 includes `companySize` and `industry` as query parameters alongside `email`, and that Workflow 2's AI prompt and bucketing code step both read from `trigger.queryParams`, not from the Lead record's (still-blank) Airtable fields.

## Airtable `422` on an update action

**Symptom:** an Airtable update step fails with a 422 error mentioning an invalid choice for a single-select field.

**Root cause:** a value being written doesn't already exist as an option on that select field, and the update action isn't set to auto-create new choices.

**Resolution:** confirm every value a workflow writes to a single-select field is drawn from that field's existing choice list (this project's code steps map free text to a fixed set for exactly this reason — see the Company Size/Industry bucketing above) rather than passing arbitrary text through to a select field.

## Slack: `not_in_channel`

**Symptom:** the Sales Notification workflow's Slack post fails with `not_in_channel`.

**Root cause:** the Slack bot has valid credentials but hasn't been explicitly invited to the target channel — a bot token being valid doesn't grant channel membership.

**Resolution:** run `/invite @[your bot's name]` in the specific channel (`#leads` in this build).

## Enrichment vendor failures

**Symptom:** every lead shows `Enrichment Status: Failed` and no real firmographic data is ever added.

**Root cause:** expected — this build ships with a placeholder enrichment API key by design. Every AI/vendor-dependent step in this project is built with a safe fallback specifically so this doesn't block the pipeline.

**Resolution:** configure a real enrichment provider (Clearbit, Hunter, Apollo, etc.) in Workflow 3's HTTP request step if real firmographic data is needed.

## Webhook failures after import

**Symptom:** every inter-workflow HTTP handoff step has the *original* build's webhook URLs hardcoded. After importing into a new Activepieces instance, these won't match your new instance's webhook URLs and every handoff will fail (typically with a 404).

**Resolution:** update each handoff step's URL to the correct webhook for your own imported copy of the next workflow in the chain — see [setup.md](setup.md).

## Missing connections after import

**Symptom:** a step fails immediately with an authentication-shaped error after importing a workflow into a new Activepieces account.

**Root cause:** Activepieces connections (Airtable, Slack) are account-scoped and don't transfer with an exported workflow JSON.

**Resolution:** re-point every Airtable/Slack action in each imported workflow at your own connections before running anything for real.
