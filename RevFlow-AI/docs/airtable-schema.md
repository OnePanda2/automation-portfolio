# Airtable Schema

RevFlow AI uses one Airtable base, **"RevFlow AI CRM"**, with three tables.

## 1. Leads

The system of record for the full lead lifecycle, from intake through routing.

| Field | Type | Written by | Notes |
|---|---|---|---|
| Full Name | Single line text | Workflow 1 | |
| Email | Email | Workflow 1 | upsert key — Workflow 1 finds-then-branches on this field rather than always creating |
| Company | Single line text | Workflow 1 | |
| Job Title | Single line text | Workflow 1 | |
| Lead Source | Single select | Workflow 1 | Website Form / Content Download / Webinar / Referral / Outbound |
| Status | Single select | Workflows 1, 2, 3, 4, 5, 6 | New → Qualified/Disqualified → Enriched → Routed → Notified → Opportunity |
| Score | Number | Workflow 2 | 0–100, from the AI scoring call |
| Priority | Single select | Workflow 2 | High / Medium / Low, computed from Score in code |
| AI Reasoning | Long text | Workflow 2 | the model's stated reasoning plus a persona label |
| Territory | Single select | Workflow 2 | Enterprise / Mid-Market / SMB — computed deterministically from company size, not left to the AI |
| Company Size | Single select | Workflow 2 | 1-10 / 11-50 / 51-200 / 201-500 / 500+ — bucketed from the raw employee count captured at intake |
| Industry | Single select | Workflow 2 | SaaS / Fintech / Healthcare IT / E-commerce / Other — normalized from free text captured at intake |
| Enrichment Status | Single select | Workflow 3 | Pending / Enriched / Failed |
| Assigned Rep | Single line text | Workflow 4 | an individual rep's name |
| Assigned Rep Role | Single select | Workflow 4 | SDR / AE — read by Workflow 6 to decide whether to create an Opportunity |
| Deal Value Estimate | Currency | Workflow 4 | a rough estimate by territory tier (Enterprise $60k / Mid-Market $25k / SMB $8k) |

**Note on Company Size and Industry:** the raw values captured at intake (e.g. `"700"`, `"SaaS"`) are free text from the webhook payload and don't necessarily match this table's fixed dropdown choices. Rather than write them directly at intake, Workflow 1 leaves both blank and passes the raw values through to Workflow 2 via query parameter, where a code step normalizes them into the fixed choice sets above before writing.

## 2. Sales Reps

The routing pool Workflow 4 assigns leads against. Seeded once at setup, not written to by any workflow.

| Field | Type | Notes |
|---|---|---|
| Name | Single line text | |
| Role | Single select | SDR / AE |
| Territory | Single select | Enterprise / Mid-Market / SMB — for AEs, this is the territory they own; SDRs are a general pool not territory-locked |
| Slack Handle | Single line text | informational only in this build — Sales Notification mentions reps by name in message text rather than resolving a real Slack user ID |
| Active | Checkbox | not currently read by any workflow — reserved for a future "exclude inactive reps from routing" rule |

Seeded with 5 SDRs (general pool, round-robin) and 3 AEs (one per territory: Enterprise, Mid-Market, SMB).

## 3. Opportunities

Created by Workflow 6, only for leads that land with an AE.

| Field | Type | Notes |
|---|---|---|
| Opportunity Name | Single line text | `{Company} - {Full Name}` |
| Lead Email | Email | soft-reference key back to Leads — see the note below |
| Lead Name | Single line text | |
| Company | Single line text | |
| Deal Value | Currency | copied from the Lead's Deal Value Estimate at creation time |
| Stage | Single select | Discovery / Proposal / Negotiation / Closed Won / Closed Lost — created at Discovery, all other stages are for manual use downstream of this MVP |
| Owner | Single line text | the assigned AE's name |
| Expected Close Date | Date | Discovery-stage default of +45 days from creation |

**Why "Lead Email" is a soft-reference key, not a linked-record field:** see [engineering-decisions.md](engineering-decisions.md#why-the-opportunities-table-links-to-leads-by-email-not-a-formal-relational-field).

## Relationships

Opportunities references Leads by a plain Email field, not a formal Airtable linked-record field — see the engineering-decisions doc linked above for why. Every write in this system is done by automation (never manual entry), which is what makes a soft key acceptable here.
