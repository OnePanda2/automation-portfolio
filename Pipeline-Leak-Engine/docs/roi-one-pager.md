# ROI One-Pager

A reusable calculator, not a fixed claim. Swap the two assumptions below for a real prospect's own numbers before presenting this.

## The formula

```
Leakage Prevented = (Leads Routed On Time × Value Per Fast-Routed Lead)
                   + (At-Risk Pipeline Value Caught × Deal-Rescue Rate)
```

| Input | Demo default | What it represents | Where it comes from |
|---|---|---|---|
| Value per fast-routed lead | $500 | The average value protected by routing a lead in under a minute instead of hours — speed-to-lead is one of the strongest predictors of lead-to-opportunity conversion | Replace with the prospect's own average deal size × their actual conversion-rate delta between fast and slow-routed leads, if known |
| Deal-rescue rate | 20% | The average share of an at-risk deal's value recovered by surfacing it before the deal review instead of during it | Replace with the prospect's own historical save-rate on deals flagged early, if they track it — otherwise 15–25% is a defensible starting range |

## Worked example (synthetic data, from `SAMPLE_CLIENT_SCENARIO.md`)

- 1 lead routed on time → **$500**
- 1 deal worth $42,000 flagged before it went cold, at a 20% rescue rate → **$8,400**
- **Total leakage prevented: $8,900** — for one day, one lead, one deal

Scale that to a team routing 15–20 leads a day and carrying 5–10 at-risk deals a week, and the same formula compounds fast — which is exactly the pitch: this isn't a one-time save, it's a number that regenerates every single day the system runs.

## How to use this with a real prospect

1. Ask for their average deal size and rough monthly lead volume.
2. Ask whether they already track how often a "quiet" deal recovers versus goes cold — if not, 20% is a reasonable, defensible starting assumption to propose.
3. Recompute the formula live with their numbers, in front of them. That's the moment this ROI page is designed for.
