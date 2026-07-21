# Architecture

## System diagram

```mermaid
flowchart TD
    Source["Lead Source (website / Typeform / HubSpot / manual)"] -->|"webhook"| F1["1. Lead Capture Gateway"]
    F1 -->|"valid lead"| F2["2. Duplicate Detection"]
    F2 -->|"Airtable create/update"| AT1[("Airtable: CRM Leads")]
    F2 --> F3["3. Lead Enrichment"]
    F3 -->|"firmographic lookup"| Enrich["Enrichment API"]
    F3 --> F4["4. AI Lead Qualification"]
    F4 -->|"scoring prompt"| AI1["OpenAI"]
    F4 --> F5["5. Territory Routing"]
    F5 -->|"rule lookup"| AT2[("Airtable: Territory Rules")]
    F5 --> F6["6. Slack Notification System"]
    F6 -->|"post"| Slack1["Slack: #leads / #automations"]

    F2 -.->|"Closed Won"| F7["7. Customer Onboarding"]
    F7 -->|"CSM pool"| AT3[("Airtable: CSM Pool / Checklist")]
    F7 -->|"welcome email"| Gmail1["Gmail"]
    F7 -->|"announce"| Slack2["Slack: #automations"]

    Schedule["Weekly schedule (Mon 8am)"] --> F8["8. Weekly Executive Report"]
    F8 -->|"read"| AT1
    F8 -->|"narrative summary"| AI2["OpenAI"]
    F8 -->|"archive"| AT4[("Airtable: Reports")]
    F8 -->|"post"| Slack3["Slack: #automations"]

    F1 -.->|"on failure"| F9["9. Error Monitoring"]
    F2 -.->|"on failure"| F9
    F3 -.->|"on failure"| F9
    F4 -.->|"on failure"| F9
    F5 -.->|"on failure"| F9
    F6 -.->|"on failure"| F9
    F9 -->|"log"| AT5[("Airtable: Failures")]
    F9 -->|"alert"| Slack4["Slack: #automations"]
    F9 -->|"escalate"| Gmail2["Gmail: Admin"]

    Question["User question"] --> F10["10. AI Revenue Assistant"]
    F10 -->|"generate query"| AI3["OpenAI"]
    F10 -->|"read"| AT1
    F10 -->|"read"| AT4
    F10 -->|"explain result"| AI3
```

## Data flow, in order

1. **Lead enters the system** through any source (a web form, Typeform, HubSpot, or a manual entry), all normalized into one shape by Lead Capture Gateway.
2. **Duplicate Detection** looks the contact up by email in the CRM. New contact → create. Existing contact → update, preserving history instead of creating a second record.
3. **Lead Enrichment** attaches firmographic data (company size, industry, country) from a third-party provider. Failure here is expected and handled — the record is flagged `Enrichment Incomplete` and continues rather than blocking the pipeline.
4. **AI Qualification** scores the lead and assigns a priority using an LLM prompt built from whatever data is available (enriched or not). On AI failure, a safe default (score 0, priority medium) is used instead of blocking.
5. **Territory Routing** evaluates an ordered rules table (company size, location, etc.) and assigns a territory and rep pool. No matching rule → falls back to an unassigned/SDR pool rather than failing.
6. **Slack Notification** tells the right channels: sales always, founders for high scores, marketing for inbound-organic sources.
7. **Customer Onboarding** fires separately, only when a deal is marked Closed Won — not part of the automatic new-lead cascade above. It round-robins a Customer Success Manager by current open-account load, creates a checklist, sends a welcome email, and announces the new customer.
8. **Weekly Executive Report** runs on a schedule (not lead-triggered), independently computing the previous week's funnel metrics and revenue, summarizing them with an LLM, archiving the result, and posting it to Slack.
9. **Error Monitoring** is called by any of the other workflows' failure-handling logic, not chained into the success path. It classifies whether a given failure is retryable, logs it, alerts the team, and escalates by email if the failure is fatal or retries are exhausted.
10. **AI Revenue Assistant** runs on-demand, translating a plain-English question into a validated, read-only query against the CRM and reports data, then explaining the result in plain English.

## Why webhook-to-webhook chaining instead of one workflow

Workflows 1 through 6 form a logical pipeline, but each is a *separate* Activepieces flow with its own webhook trigger, wired together by having each workflow's final step call the next workflow's webhook. This is a deliberate structural choice — see [engineering-decisions.md](../docs/engineering-decisions.md#why-modular-workflows-instead-of-one-flow) for the full reasoning.
