# Setup Guide

This guide walks another engineer through deploying RevenuePilot OS from scratch.

## 1. Required accounts

- **Activepieces Cloud** account (or self-hosted instance) — free tier is sufficient
- **Airtable** account — free tier is sufficient at this project's data volume
- **Google account** with Gmail — for transactional email
- **Slack workspace** with permission to install an app/bot
- **OpenAI API key** — used by workflows 4, 8, and 10 (optional to start; see note below)
- (Optional) An enrichment API provider (e.g. Clearbit, Hunter, Apollo) — used by workflow 3

> **Note on API keys:** every AI/vendor integration in this project (enrichment, OpenAI) is built with a real call structure and a safe fallback if the call fails. You can deploy and run the full pipeline with placeholder keys and every workflow will complete successfully — it'll just use fallback values (default lead score, "enrichment incomplete" flags, etc.) instead of real AI/vendor output. Swap in real keys whenever you're ready; no workflow logic needs to change.

## 2. Airtable setup

Create the following bases (a limitation of the Airtable API used here is that a table can't be added to an *existing* base — so each of these is its own base):

### RevenuePilot CRM
One table, **Leads**, with fields:

| Field | Type |
|---|---|
| Email | Email |
| Full Name | Single line text |
| Company | Single line text |
| Phone | Phone number |
| Source | Single select (website / typeform / tally / hubspot / manual) |
| Created At | Date/time |
| Duplicate | Checkbox |
| Notes | Long text |
| Status | Single select |
| Lead Score | Number |
| Priority | Single select (high / medium / low) |
| AI Reasoning | Long text |
| Industry | Single line text |
| Company Size | Number |
| Country | Single line text |
| Enrichment Incomplete | Checkbox |
| Territory | Single select |
| Assigned Rep | Single line text |
| Deal Value | Currency |
| Assigned CSM | Email |
| Onboarding Completed At | Date/time |

Full field-by-field detail, including which workflow writes each field: [airtable-schema.md](airtable-schema.md).

### RevenuePilot Territory Rules
One table, **Territory Rules**: Rule Order (number), Condition Field (text), Condition Op (single select: equals/in/gt/contains), Condition Value (text), Territory (text), Rep Pool (text), Active (checkbox).

### RevenuePilot Onboarding
Two tables:
- **CSM Pool**: Name, Email, Open Onboardings (number), Active (checkbox)
- **Onboarding Checklist**: Lead Record ID, Company, Folder URL, Page URL, Assigned CSM, Created At, Completed At

### RevenuePilot Reports
One table, **Reports**: Week Start, Week End, Leads Count, MQLs Count, SQLs Count, Conversion Rate, Revenue, Lost Deals, AI Summary, Created At.

### RevenuePilot Monitoring
One table, **Failures**: Flow Name, Step Name, Error Message, Payload, Retry Count, Resolved (checkbox), Created At.

## 3. Slack setup

1. Install an Activepieces-connected Slack app to your workspace.
2. Create (or choose existing) channels for lead notifications and internal automation alerts — this project uses two: one for sales-facing lead alerts, one for internal automation/error alerts.
3. **Invite the bot to every channel it needs to post in.** This is the single most common setup gap — a bot with valid credentials will still fail with a `not_in_channel` error if it hasn't been explicitly invited via `/invite @[bot name]` in each channel.

## 4. Gmail and OpenAI

- Connect Gmail via OAuth in Activepieces — used for onboarding welcome emails and error escalation emails.
- Add your OpenAI API key wherever the qualification, reporting, and assistant workflows reference it (see each workflow's first Code step).

## 5. Import workflows

Import each file from `workflows/` into Activepieces, in this order:

1. `01-lead-capture-gateway.json` (Lead Capture Gateway)
2. `02-duplicate-detection.json` (Duplicate Detection)
3. `03-lead-enrichment.json` (Lead Enrichment)
4. `04-ai-lead-qualification.json` (AI Lead Qualification)
5. `05-territory-routing.json` (Territory Routing)
6. `06-slack-notification-system.json` (Slack Notification System)
7. `07-customer-onboarding.json` (Customer Onboarding)
8. `08-weekly-executive-report.json` (Weekly Executive Report)
9. `09-error-monitoring.json` (Error Monitoring)
10. `10-ai-revenue-assistant.json` (AI Revenue Assistant)

After import, each workflow will have its own webhook URL. **The cascade steps in workflows 1–5 reference the next workflow's webhook URL directly** — after import, open each workflow's final HTTP step and update the URL to match your instance's actual webhook URL for the next workflow in the chain (these are instance-specific and will not match the original build's URLs).

## 6. Connect Airtable, Slack, and Gmail connections in each workflow

Each imported workflow will need its Airtable/Slack/Gmail actions re-pointed at your own connections (Activepieces connections are account-scoped and won't carry over from an export).

## 7. Activation order

1. Publish and enable Flows 1–6 first, and send a test lead through Flow 1's webhook to confirm the full cascade.
2. Publish Flow 9 (Error Monitoring) — other workflows' error paths reference it.
3. Publish Flow 7 (Customer Onboarding) — test by sending a mock `Closed Won` payload to its webhook.
4. Publish Flow 8 (Weekly Executive Report) — it will run on its own schedule; you can also trigger it manually to test before the first scheduled run.
5. Publish Flow 10 (AI Revenue Assistant) last — it's fully independent of the others.

## 8. Verifying the deployment

Send a test lead to Flow 1's webhook:

```json
{
  "company": "Test Company Inc",
  "email": "test@example.com",
  "name": "Test Lead",
  "phone": "+10000000000"
}
```

Then check:
- A new record appears in RevenuePilot CRM → Leads
- The record has enrichment fields (or `Enrichment Incomplete: true` if no enrichment key yet)
- The record has a Lead Score, Priority, and AI Reasoning (or the fallback default if no OpenAI key yet)
- The record has a Territory and Assigned Rep
- A message appears in your configured Slack channel

See [troubleshooting.md](troubleshooting.md) if any of these don't happen.
