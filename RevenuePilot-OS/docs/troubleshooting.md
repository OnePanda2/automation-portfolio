# Troubleshooting Guide

Known failure modes across RevenuePilot OS, their root causes, and how to fix them.

## Slack: `not_in_channel`

**Symptom:** a Slack post step fails with `An API error occurred: not_in_channel`.

**Root cause:** the Slack bot has valid credentials and scope, but hasn't been explicitly invited to the target channel. This is Slack's standard behavior — a bot token being valid doesn't grant a bot membership in any given channel.

**Resolution:** run `/invite @[your bot's name]` in the specific channel that's failing. This is per-channel — inviting the bot to one channel doesn't grant access to others. Note this can also produce a partial-failure state where *some* workflows post successfully (because the bot was invited to that specific channel) while others fail, if they post to different channels.

## OpenAI: `401` on every AI call

**Symptom:** every workflow calling OpenAI (Qualification, Reporting, Assistant) fails or falls back with `OpenAI request failed: 401`.

**Root cause:** this is expected if you're running with the placeholder API key this project ships with by default. It is not a bug — every AI-dependent step is built with a safe fallback specifically so this doesn't block the pipeline.

**Resolution:** if you want real AI output instead of fallback values, replace the placeholder key in each workflow's relevant Code step with a real OpenAI API key. No other code changes are required.

## Enrichment vendor failures

**Symptom:** Workflow 3 always sets `Enrichment Incomplete: true` and never populates Industry/Company Size/Country.

**Root cause:** same as above — no real enrichment vendor key is configured by default.

**Resolution:** configure a real enrichment provider (Clearbit, Hunter, Apollo, etc.) and update the API call in Workflow 3's enrichment step.

## "No active CSMs found" / onboarding assigns nobody

**Symptom:** Workflow 7's CSM round-robin step returns no candidate, or errors.

**Root cause:** every row in the CSM Pool table has `Active` unchecked, or the table is empty.

**Resolution:** ensure at least one CSM Pool row has `Active` checked. If the team roster changes, deactivate departing CSMs rather than deleting their rows, so Onboarding Checklist history keeps a valid reference.

## Airtable `422` on an update action

**Symptom:** an Airtable update step fails with a 422 error mentioning an invalid choice for a single-select field.

**Root cause:** Activepieces' plain Airtable "update record" action does not set `typecast: true`, so it rejects any value that isn't already an existing option on that select field.

**Resolution:** use a `custom_api_call` PATCH request with `{ "fields": {...}, "typecast": true }` in the body for any update that might introduce a new select value, instead of the plain update-record action.

## A record's fields go missing partway through the pipeline

**Symptom:** a downstream workflow (e.g. Slack Notification) shows blank Company/Email/Name even though the record has this data in Airtable.

**Root cause:** the workflow-to-workflow HTTP cascade forwards whatever specific JSON object each workflow's own code produces. If a workflow forwards its own Code step's *partial* output (e.g. just the fields it personally added) instead of the *full record* returned by the Airtable write action, downstream workflows only see that partial slice.

**Resolution:** every cascade step should reference the Airtable action's output (which contains the complete current record) as the source for the forwarded `crmRecord.fields`, not an upstream Code step's own partial return value.

## A downstream workflow can't find the record ID

**Symptom:** an Airtable update in a downstream workflow fails because the record ID resolves to empty/undefined.

**Root cause:** field-naming inconsistency between what one workflow forwards (e.g. `crmRecord.recordId`) and what the next workflow's code expects to read (e.g. `crmRecord.id`).

**Resolution:** standardize on one key name for the record identifier across every workflow's cascade payload and confirm each receiving workflow's code reads that same key.

## HTTP cascade calls silently do nothing

**Symptom:** an HTTP request step to a downstream workflow's webhook returns 200, but the downstream workflow's trigger shows an empty body.

**Root cause:** Activepieces' HTTP piece requires a JSON body to be wrapped in a `{ "data": { ... } }` structure for `body_type: json` — passing the intended payload as a flat object at the top level results in an empty body being sent.

**Resolution:** always wrap the JSON payload as `{ "data": { <your payload> } }` when configuring an HTTP request step's body.

## Weekly report shows unexpected numbers

**Symptom:** leads/MQL/SQL counts in the Weekly Executive Report don't match manual expectations.

**Root causes to check, in order:**
1. **Timezone boundary:** the report computes its week window in UTC, but the CRM's Created At field displays in local time — a record created near midnight can land in a different week than expected when converted to UTC.
2. **Unfiltered fetch:** Workflow 8 pulls the *entire* Leads table (up to the first 100 records, unfiltered) and buckets by date client-side — pre-existing or unrelated test data in the target week window will be included.
3. **Conversion rate formula:** it's SQLs ÷ Leads, not MQLs ÷ Leads — a common point of confusion when sanity-checking by hand.
4. **Status string matching:** bucketing is case-sensitive and exact-match (e.g. `Customer`, not `customer` or `Closed Won`) — a mismatched status string silently excludes a record from revenue/lost-deal counts.

## Rate limits

None of the external services used in this project (Airtable, Slack, Gmail, OpenAI) have explicit rate-limit handling built into these workflows. At this project's scale that hasn't been an issue, but a production deployment with meaningfully higher lead volume should add retry-with-backoff around each external call — Workflow 9's `retryable` classification (429/5xx/timeout) is designed to support this if a caller-side retry loop is added.

## Missing environment variables / credentials

Every workflow that calls an external service (Airtable, Slack, Gmail, OpenAI, enrichment) needs its own connection configured in Activepieces after import — connections are account-scoped and do not transfer with an exported workflow JSON. If a step fails immediately with an authentication-shaped error, check that workflow's connection is set to a valid, currently-authorized connection in your own Activepieces account first.

## Webhook failures after import

Every inter-workflow HTTP cascade step (Workflows 1–5) has the *original* build's webhook URLs hardcoded. After importing into a new Activepieces instance, these URLs won't match your new instance's webhook URLs and every cascade call will fail (typically with a 404). Update each cascade step's URL to the correct webhook for your own imported copy of the next workflow — see [setup.md](setup.md).
