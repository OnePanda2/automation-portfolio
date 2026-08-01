# Architecture

Stack: **Activepieces** (orchestration) + **Airtable** (system of record, via MCP). No Postgres, no Docker, no custom backend — every module is demoable live from a browser tab.

Four Airtable bases, one per module's data, plus a shared "Pipeline Leak Engine" base for the core CRM:

- **Pipeline Leak Engine** — Leads, Reps, ICP Scoring Rules (Module 1's system of record)
- **Pipeline Leak Engine - CRM Health** — Data Quality Issues, CRM Health Score (Module 2)
- **Pipeline Leak Engine - Deals** — Deals (Module 3)
- **Pipeline Leak Engine - Daily Brief** — Daily Brief Metrics, a running tracker fed by Modules 1 and 3, read by Module 4

```
                    ┌─────────────────────┐
   New lead ───────▶│ Module 1            │───▶ Slack #leads
                     │ Speed-to-Lead Router│         │
                     └─────────┬───────────┘         │
                               │ writes Lead                  increments
                               ▼                       Leads Routed Count
                        [ Leads table ]                       │
                               │                               ▼
                               │                     ┌───────────────────┐
                     ┌─────────▼───────────┐          │ Module 4          │
   Lead created ────▶│ Module 2            │          │ Morning Revenue    │──▶ Slack #automations
                     │ CRM Self-Healing     │          │ Brief (reads 1-3)  │
                     └─────────┬───────────┘          └─────────▲──────────┘
                               │ updates CRM Health Score                │
                               ▼                                        │ increments
                      [ CRM Health Score ]                    At-Risk Pipeline Value
                                                                          │
                     ┌──────────────────────┐                            │
   Deal touched ────▶│ Module 3              │───▶ Slack #automations ──┘
                     │ Pipeline Slippage      │
                     │ Monitor                │
                     └────────────────────────┘
```

## Module 1 — Speed-to-Lead Router

**Trigger:** webhook (lead form submission)

1. Create raw Lead record (`New`)
2. Deterministic enrichment HTTP call (placeholder key) → success/failure branch → writes Employee Band / Industry / Revenue Band, or `Unknown` + `Enrichment Incomplete` flag on failure
3. AI call normalizes messy job title + free-text need into a Persona Tag + Intent Summary → falls back to raw passthrough on failure (with an explicit HTTP-status-code check, since a 401 response does not count as a request failure in Activepieces' own success/failure branching — see "Bugs found and fixed" below)
4. Deterministic Territory router (employee count / country rules)
5. Deterministic ICP Scoring lookup against an editable rules table → falls back to Tier 3 / Low on no match
6. Deterministic Rep lookup by territory → falls back to a queue rep if none active
7. Slack notification to the assigned rep, `Status` marked `Notified`
8. Increments the shared Daily Brief `Leads Routed Count` tracker

## Module 2 — CRM Self-Healing Engine

**Trigger:** webhook (fires once a Lead record exists — from Module 1 or any other entry point)

1. Deterministic email-format check → logs an `Invalid Email Format` issue on failure
2. Deterministic exact-match dedup by email → **auto-merges** safe duplicates (deletes the redundant record, logs it as resolved)
3. Deterministic fuzzy-candidate search by company name → AI judges whether two differently-named companies are probably the same account → **always flags for human review, never auto-merges** — this split is explicit and intentional, since a wrong auto-merge destroys attribution history
4. A live CRM Health Score (0–100) recalculated on every run from cumulative valid/invalid counts

## Module 3 — Pipeline Slippage Monitor

**Trigger:** webhook (deal touched — a stage-change attempt, or a periodic check)

1. Deterministic stage-entry gates — e.g. no move to `Proposal Sent` without a proposal doc link, no move to `Closed Won` without a close date
2. Deterministic tripwires (computed in one small CODE step, since the math itself doesn't fit Activepieces' condition-builder cleanly): days-in-stage > 10, no activity > 7 days, close date pushed ≥ 2 times
3. If any tripwire fires: an AI call reads the deal's activity log and writes a one-line plain-English risk narrative distinguishing a genuinely dying deal from one that's quiet for a known, benign reason — with a deterministic fallback narrative (built from the tripwire facts themselves) if the AI call fails
4. Slack alert to the deal owner and manager, surfaced before the deal review instead of during it
5. Increments the shared Daily Brief `Deals Flagged At Risk Count` and `At-Risk Pipeline Value Caught` trackers

## Module 4 — Morning Revenue Brief

**Trigger:** webhook in this repo (would be a daily Schedule trigger in a live deployment — see "Known simplifications")

1. Fetches Module 2's CRM Health Score and the Daily Brief Metrics tracker fed by Modules 1 and 3
2. Deterministic leakage-prevented calculation: `(leads routed × $500) + (at-risk pipeline value caught × 20%)` — both constants are named assumptions, meant to be swapped for a real client's numbers, not treated as universal truths
3. AI executive-summary synthesis reads the numbers and writes a 3–4 sentence brief for a VP of Sales, with a deterministic fallback summary (built directly from the numbers) if the AI call fails
4. Delivered to Slack

## Bugs found and fixed during the build

Worth stating plainly, since this is exactly the kind of thing a technical interviewer or a skeptical client would ask about:

- **`airtable_get_record_by_id` returns the record directly** (`{id, fields}`), while `airtable_find_record` returns an array. Mixing these up silently produced blank fields downstream until caught by testing.
- **Activepieces' HTTP success/failure branching only reacts to network-level failures, not HTTP error status codes.** A 401 from OpenAI still counts as a "successful request" (a response was received) — every AI call in this system therefore has an explicit status-code check after it, not just a `continueOnFailure` flag.
- **Activepieces' `{{ }}` templates don't support inline arithmetic.** Every running counter (CRM Health Score, Daily Brief trackers) needed a small CODE step to compute the increment, rather than `{{x + 1}}` directly.
- **The Airtable connector's native "New Record" trigger threw a persistent `ENTITY_NOT_FOUND` error** in this environment. Every module uses a webhook trigger instead — which, as a side effect, makes each module independently callable by anything, not just by the module before it in the pipeline.

## Known simplifications

- Module 4 is triggered by webhook in this repo for testability; in a live deployment it would be a daily Schedule trigger instead.
- Module 2 and 3's lookups rely on Airtable substring search rather than a true table scan (no bulk-list action available in this connector) — see the README's "what would break at 10x volume" section.
- Daily Brief trackers are cumulative running totals, not reset daily; a live deployment would either reset them each morning after the brief runs or move to date-partitioned records.
