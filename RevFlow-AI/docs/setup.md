# Setup Guide

This guide walks another engineer through deploying RevFlow AI from scratch.

## 1. Required accounts

- **Activepieces Cloud** account (or self-hosted instance) — free tier is sufficient, and this project also depends on Activepieces' built-in AI provider for lead scoring (no separate OpenAI/Anthropic key needed)
- **Airtable** account — free tier is sufficient at this project's data volume
- **Slack workspace** with permission to install an app/bot
- (Optional) A firmographic enrichment API provider (e.g. Clearbit, Hunter, Apollo) — used by Workflow 3; the pipeline runs end-to-end without one

> **Note on API keys:** the enrichment integration in Workflow 3 is built with a real call structure and a safe fallback if the call fails. You can deploy and run the full pipeline with a placeholder enrichment key and every workflow will complete successfully — Workflow 3 will just flag `Enrichment Status: Failed` instead of populating real vendor data. Swap in a real key whenever you're ready; no other workflow needs to change.

## 2. Airtable setup

Create one base, **"RevFlow AI CRM"**, with three tables: **Leads**, **Sales Reps**, **Opportunities**. Full field-by-field detail: [airtable-schema.md](airtable-schema.md).

After creating the tables, seed **Sales Reps** with your team roster — this build uses 5 SDRs (general pool) and 3 AEs (one per territory: Enterprise, Mid-Market, SMB). If your roster differs, update Workflow 4's routing code step to match (see [engineering-decisions.md](engineering-decisions.md#why-round-robin-uses-a-deterministic-hash-not-a-stored-counter) for why this is currently hardcoded rather than read from the table).

## 3. Slack setup

1. Install an Activepieces-connected Slack app to your workspace.
2. Choose or create a channel for lead notifications (this build posts to `#leads`).
3. **Invite the bot to that channel.** This is the single most common setup gap — see [troubleshooting.md](troubleshooting.md#slack-not_in_channel).

## 4. Import workflows

Import each file from `workflows/` into Activepieces, in this order:

1. `01-lead-capture-gateway.json`
2. `02-lead-qualification.json`
3. `03-lead-enrichment.json`
4. `04-lead-routing.json`
5. `05-sales-notification.json`
6. `06-opportunity-creation.json`

After import, each workflow will have its own webhook URL. **Every workflow except the last one references the next workflow's webhook URL in its final HTTP step.** Open each one and update the URL to match your own instance — these are instance-specific and won't match the original build's URLs. See [troubleshooting.md](troubleshooting.md#webhook-failures-after-import).

## 5. Re-point connections

Each imported workflow's Airtable and Slack actions need to be re-pointed at your own connections — Activepieces connections are account-scoped and don't transfer with an exported workflow.

## 6. Activation order

Publish and enable all six workflows before testing, since Workflow 1's success depends on Workflow 2 already being live to receive its handoff (and so on down the chain).

## 7. Verifying the deployment

Send a test lead to Workflow 1's webhook:

```json
{
  "company": "Test Company Inc",
  "companySize": "250",
  "email": "test@example.com",
  "fullName": "Test Lead",
  "industry": "SaaS",
  "jobTitle": "VP of Customer Support",
  "leadSource": "Webinar"
}
```

Then check, directly in Airtable (not just that each workflow reported success):
- A new record appears in Leads with `Status: New`, then progressing through `Qualified` (or `Disqualified`, if the score is low) → `Enriched` → `Routed` → `Notified` → `Opportunity` (only if routed to an AE)
- `Score`, `Priority`, `Territory`, `Company Size`, and `Industry` are all populated — not blank (see [troubleshooting.md](troubleshooting.md#company-size--industry-show-blank-on-the-lead-record-after-qualification) if they are)
- `Assigned Rep` and `Assigned Rep Role` are set
- A message appears in your configured Slack channel
- If `Assigned Rep Role` is `AE`, a matching row appears in Opportunities; if `SDR`, no Opportunity row should exist for this lead

See [troubleshooting.md](troubleshooting.md) if any of these don't happen.
