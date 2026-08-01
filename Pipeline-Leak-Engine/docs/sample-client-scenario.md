# Sample Client Scenario

**This is a demonstration built entirely on synthetic, made-up data.** No real company, lead, deal, or person is represented here. It exists to show the system in motion — never present these numbers as a real client's results.

---

## The setup

A mid-market B2B software company's sales team is losing pipeline in three places: leads sit unrouted for hours, the CRM has quietly accumulated duplicate accounts, and deals stall in Negotiation without anyone noticing until the weekly deal review.

## What the system caught, this run

**Speed-to-Lead Router:** A lead came in from "Northwind Health Systems" — a mid-size healthcare company. Inside seconds, it was enriched, scored (Tier 3 given the synthetic enrichment data available), routed to the matching rep by territory, and posted to Slack — instead of sitting in an inbox until someone got to it.

**CRM Self-Healing Engine:** A second lead came in from "Acme Example Corp" with the same email address already on file. Instead of creating a duplicate the team would have to notice and clean up manually, the system auto-merged it on the spot and logged the merge for audit. A separate lead from "Northwind Health" (without "Systems") was *not* auto-merged — it was flagged for a human to confirm, because a fuzzy name match is exactly the kind of judgment call that shouldn't happen silently.

**Pipeline Slippage Monitor:** The Northwind Health Systems deal — worth $42,000 — had gone quiet: 14 days in the Negotiation stage, 11 days since the last logged activity, and its close date pushed twice. All three deterministic tripwires fired, and the system flagged it to the rep and their manager *before* the next deal review, with the reasoning attached. A second deal, quiet because the champion is on parental leave, correctly did **not** get flagged — the tripwires and the (attempted) AI narrative both recognized quiet-for-a-good-reason as different from quiet-because-it's-dying.

**Morning Revenue Brief:** The next morning, one Slack message tied it all together:

> :sunrise: **Morning Revenue Brief**
> 1 lead(s) were routed on time and the CRM sits at a 75/100 health score. 1 deal(s) worth $42,000 are flagged at risk and need attention before today's deal review. Estimated leakage prevented: **$8,900**.

## The takeaway

Nobody had to remember to check anything. The system caught a stalled $42K deal, prevented a duplicate CRM record, and routed a lead in seconds — all before the sales team's first coffee.
