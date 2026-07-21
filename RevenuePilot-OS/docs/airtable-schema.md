# Airtable Schema

RevenuePilot OS uses five Airtable bases. They're split this way because the Airtable API integration used in this project can create a new base with tables in one call, but has no way to add a table to an *existing* base — so each new data need became its own small base rather than a growing single schema. See [engineering-decisions.md](engineering-decisions.md) for the reasoning.

## 1. RevenuePilot CRM

**Table: Leads** — the system of record for the full lead lifecycle.

| Field | Type | Written by | Notes |
|---|---|---|---|
| Email | Email | Workflow 2 | Used as the dedup key |
| Full Name | Single line text | Workflow 2 | |
| Company | Single line text | Workflow 2 | |
| Phone | Phone number | Workflow 2 | |
| Source | Single select | Workflow 2 | website / typeform / tally / hubspot / manual |
| Created At | Date/time | Workflow 2 | ISO timestamp |
| Duplicate | Checkbox | Workflow 2 | true on repeat-contact updates |
| Notes | Long text | Workflow 2 | activity/touch log, appended on each update |
| Status | Single select | Workflows 2, 5, 7 | grows via typecast as new statuses appear (New, Scored, Routed, Notified, Customer, Closed Lost, In progress, etc.) — see the note on checkbox/select behavior below |
| Lead Score | Number | Workflow 4 | 0 on AI-call failure (fallback) |
| Priority | Single select | Workflow 4 | high / medium / low; "medium" on AI-call failure |
| AI Reasoning | Long text | Workflow 4 | includes the failure reason when the AI call falls back |
| Industry | Single line text | Workflow 3 | blank if enrichment fails |
| Company Size | Number | Workflow 3 | blank if enrichment fails |
| Country | Single line text | Workflow 3 | blank if enrichment fails |
| Enrichment Incomplete | Checkbox | Workflow 3 | true if the enrichment call failed |
| Territory | Single select | Workflow 5 | grows via typecast (enterprise/india/us/unassigned, etc.) |
| Assigned Rep | Single line text | Workflow 5 | a rep pool name, e.g. `sdr_pool`, or an individual rep once assigned downstream |
| Deal Value | Currency | manual / external | set when a deal closes; read by workflow 8 for revenue |
| Assigned CSM | Email | Workflow 7 | set on customer onboarding |
| Onboarding Completed At | Date/time | Workflow 7 | set when onboarding checklist completes |

**Checkbox/select behavior note:** several select fields (Status, Territory) accept new values via the Airtable API's typecast option, which auto-creates a new choice the first time it's used. This means the live choice list can include values beyond what's documented here — check the field's actual options in Airtable before assuming this table is exhaustive.

## 2. RevenuePilot Territory Rules

**Table: Territory Rules** — ordered routing rules evaluated by Workflow 5.

| Field | Type | Notes |
|---|---|---|
| Rule Order | Number | evaluated ascending; first match wins |
| Condition Field | Single line text | e.g. `company_size`, `location` |
| Condition Op | Single select | equals / in / gt / contains |
| Condition Value | Single line text | e.g. `1000`, `india` |
| Territory | Single line text | assigned when this rule matches |
| Rep Pool | Single line text | assigned when this rule matches |
| Active | Checkbox | inactive rules are skipped |

## 3. RevenuePilot Onboarding

**Table: CSM Pool** — the round-robin assignment pool.

| Field | Type | Notes |
|---|---|---|
| Name | Single line text | |
| Email | Email | |
| Open Onboardings | Number | incremented by Workflow 7 on each new assignment; this is what round-robin selection minimizes |
| Active | Checkbox | inactive CSMs are excluded from assignment |

**Table: Onboarding Checklist** — one row per new customer onboarding.

| Field | Type | Notes |
|---|---|---|
| Lead Record ID | Single line text | links back to the CRM Leads record |
| Company | Single line text | |
| Folder URL | URL | populated once a Drive integration is connected |
| Page URL | URL | populated once a Notion integration is connected |
| Assigned CSM | Single line text | |
| Created At | Date/time | |
| Completed At | Date/time | |

## 4. RevenuePilot Reports

**Table: Reports** — one row per week, written by Workflow 8.

| Field | Type | Notes |
|---|---|---|
| Week Start | Single line text | ISO date, e.g. `2026-07-13` |
| Week End | Single line text | ISO date |
| Leads Count | Number | all leads created in the window |
| MQLs Count | Number | leads with status at or beyond "Scored" |
| SQLs Count | Number | leads with status at or beyond "Routed" |
| Conversion Rate | Number | SQLs ÷ Leads × 100, rounded to 2 decimals — **not** MQL-based |
| Revenue | Number | sum of Deal Value where Status = Customer |
| Lost Deals | Number | count where Status = Closed Lost |
| AI Summary | Long text | falls back to a plain-text notice if the OpenAI call fails |
| Created At | Date/time | |

**Known limitation:** Workflow 8 fetches Leads with a single, unpaginated API call — if the Leads table grows past 100 records, older records may be silently excluded from the report. See [troubleshooting.md](troubleshooting.md).

## 5. RevenuePilot Monitoring

**Table: Failures** — written by Workflow 9.

| Field | Type | Notes |
|---|---|---|
| Flow Name | Single line text | the name of the workflow that failed |
| Step Name | Single line text | the specific step that failed |
| Error Message | Long text | |
| Payload | Long text | JSON-stringified context at time of failure |
| Retry Count | Number | how many retries had already been attempted |
| Resolved | Checkbox | manually marked once investigated |
| Created At | Date/time | |

## Relationships

None of these bases use Airtable's native linked-record fields — every cross-base reference (e.g. a Failures row's connection to a Leads record, or an Onboarding Checklist row's `Lead Record ID`) is a plain text field holding the other record's ID, not a formal Airtable link. This was a deliberate simplification: linked-record fields require both tables to live in the same base, which the multi-base structure here doesn't allow.
