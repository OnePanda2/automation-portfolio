# Pricing Guide

Each module is sellable standalone or bundled, consistent with project-based freelance/agency pricing in this space ($1.5K–$10K per system, depending on scope and customization).

## Standalone pricing

| Module | Suggested range | Why |
|---|---|---|
| Speed-to-Lead Router | $2,000 – $5,000 | Highest-visibility, fastest-to-demo win; directly tied to a conversion-rate metric every VP of Sales already tracks |
| CRM Self-Healing Engine | $1,500 – $4,000 | Foundational — unblocks every other automation the client might buy next; the easiest module to demo live via the Health Score |
| Pipeline Slippage Monitor | $2,000 – $5,000 | Directly improves forecast accuracy, a number sales leadership is measured on |
| Morning Revenue Brief | $1,000 – $2,500 as an add-on | Sold as the synthesis layer on top of at least one other module, not standalone — it has nothing to summarize without them |

## Bundled pricing

- **Any two modules:** 15% discount off combined standalone price
- **All four modules (the full system):** $6,000 – $12,000 depending on customization depth (real API keys, client-specific scoring rules, multiple territories/reps, etc.) — positioned as the "full five-minute demo" package
- **Retainer option:** a reduced upfront build fee plus a monthly retainer for monitoring, rule tuning, and adding client-specific edge cases as they come up

## What changes the price

- **Real API keys and live third-party integrations** (enrichment provider, OpenAI, a real Slack/CRM workspace) — this repo ships with placeholder keys and fully-built, tested fallback paths; wiring in real credentials is scoped separately per client
- **Volume** — the substring-search-based lookups in Modules 2–3 are demo/small-scale appropriate; a client with thousands of monthly leads needs a scoped conversation about a proper database layer underneath (see `ARCHITECTURE.md`)
- **Custom scoring/routing rules** — the ICP scoring matrix and territory rules ship with an illustrative rule set; building out a client's actual rules is where a meaningful share of real engagement time goes
