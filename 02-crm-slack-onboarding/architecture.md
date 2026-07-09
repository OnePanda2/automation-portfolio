# Architecture — CRM + Slack Onboarding

Three Activepieces flows, each triggered by a webhook, that route a new lead/signup payload through validation, storage, and notification steps.

## Flow A — `flow-a-lead-capture.json`

- **Catch Webhook (trigger)** — no auth, accepts raw JSON body (`name`, `email`, `plan`, etc.). Kept auth-less since this is a learning/sandbox flow; a production version would add a shared-secret header check.
- **Send Email (Gmail)** — sends a plain-text welcome email directly to `trigger.body.email` using `trigger.body.name` and `trigger.body.plan` in the copy. No branching here — this is the simplest version of the flow (happy-path only), used to prove out the webhook → Gmail piece connection before adding validation logic in Flow B.

## Flow B — `flow-b-airtable-trigger.json`

- **Catch Webhook (trigger)** — same as Flow A.
- **Router** — added right after the trigger to introduce conditional logic missing from Flow A.
  - **Branch 1 (`email` is empty)** → Slack message to `#onboarding-failed`-style channel (`C0BFX7KJAC9`) with the raw payload attached for debugging → **Stop Flow**. This exists so a malformed lead (missing email) doesn't silently fail deeper in the flow (e.g. Gmail action erroring on empty receiver) — it's caught immediately and surfaced to a human.
  - **Otherwise (fallback)** → the real happy path:
    1. **Create Airtable Record** — writes `name`, `source`, `email`, `company` into the CRM base (`appju97zngNteVlGV`, table `tblsrw7HAEdQTN3MI`), and hardcodes `Status = "Onboarding"` so every new record lands in a known pipeline stage.
    2. **Send Email (Gmail)** — same welcome email as Flow A, now only sent once the lead is confirmed valid and stored.
    3. **Send Message To A Channel (Slack)** — posts a "New Customer Onboarded" notification to a separate internal channel (`C0BFX7J46V7`) with an explicit next action ("Assign account manager"), so the team knows a human still needs to act after automation hands off.
  - Router uses `EXECUTE_FIRST_MATCH` so only one branch runs — either the failure path or the success path, never both.

## Flow C — `flow-onboarding.json`

- **Catch Webhook (trigger)** — same pattern as A and B.
- **Router** — expanded from Flow B's 2-branch router to a 3-branch router, since this flow validates two required fields instead of one:
  - **Branch 1 (`name` is empty)** → Slack alert flagging a missing name, includes email/company/source for context → **Stop Flow**. Split out from the email-check branch so the Slack message can say exactly which field is missing, instead of a generic "onboarding failed" message.
  - **Branch 2 (`email` is empty)** → same idea, separate Slack alert specific to missing email → **Stop Flow**.
  - **Otherwise (fallback)** → full happy path:
    1. **Create Airtable Record** — identical field mapping to Flow B (same base/table), keeping the CRM schema consistent across flows.
    2. **Send Email (Gmail)** — a shorter, branded welcome email ("Welcome to XwhyZee") rather than the plan-based copy used in A/B — this flow represents the final onboarding-confirmation step rather than the plan-signup step, so the copy was simplified accordingly.
    3. **Send Message To A Channel (Slack)** — posts a "New Lead" notification (not "Onboarded") to the team channel, prompting them to check Airtable to assign and follow up — this flow treats the record as a fresh lead needing manual triage, not a fully onboarded customer.
  - An additional **Stop Flow** sits directly on the router's own `nextAction` (outside the branches) as a safety net/dead-code guard from the Activepieces builder — harmless but worth noting if cleaning up later.

## Why two routers with different branch counts (B vs C)

Flow B only validates `email` because at that point in the course sequence the "missing name" case hadn't been introduced yet. Flow C adds a second required-field branch (`name`) to demonstrate multi-condition routing (`EXECUTE_FIRST_MATCH` across 3 branches instead of 2) — each flow is a slightly more advanced version of the last, building up router complexity incrementally rather than writing the "final" version straight away.
