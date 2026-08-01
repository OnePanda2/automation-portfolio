# The Pipeline Leak Engine

A focused revenue-recovery automation system for B2B GTM teams — four flagship workflows, each solving one named, quantified pipeline leak, wired together into a five-minute live demo.

Built with **Activepieces** (orchestration) + **Airtable** (system of record), using Claude to write and test every flow.

---

## Built with AI — and that's a feature, not a footnote

This project was built using Claude (Anthropic) as the implementation partner: directing the architecture, the deterministic/AI split in each module, and the business logic, while Claude wrote, wired, and tested the actual Activepieces flows and Airtable schemas.

I'm not disclosing this reluctantly. If you're evaluating this repo as a potential client or employer, you were always going to be able to tell — and I'd rather make the case directly than have you wonder why I didn't:

- **Automation exists to make delivery faster — using AI to build automations is the same principle applied one level up.** If speed is the whole point of the deliverable, it should also be the point of the delivery process.
- **AI-assisted development compresses the time between "here's the brief" and "here's a working, tested system."** For a freelance engagement, that directly shortens time-to-value for the client.
- **The real choice a client faces isn't "AI-assisted vs. fully manual expert" — it's who can deliver the same precision faster.** Given two consultants with comparable judgment, the one who uses AI well to eliminate boilerplate and iteration time is the better hire, not the weaker one.
- **Every day a GTM team runs without automation is a day of measurable leakage** — industry estimates put the pipeline lost to slow lead routing and stale CRM data at roughly 10–20% before a rep ever engages, compounding daily. Faster delivery of the fix has a direct dollar value.

A few more reasons this is a legitimate way to work, not a shortcut around the work:

- **AI didn't make the decisions that matter.** The deterministic-vs-AI split in each module, the specific tripwire thresholds, the stage-entry gates, the fallback behavior when a third-party API is unreachable — those are judgment calls a working RevOps engineer has to own and be able to defend. AI wrote code; it didn't design the system.
- **Every AI call in the delivered system is itself documented and has a tested deterministic fallback** — see "Deterministic vs. AI, by design" below. The same transparency I'm applying to how this was *built* is built into what gets *delivered*. A client isn't buying a black box either way.
- **The market has already made its call on this.** Clients hiring automation consultants increasingly expect AI-fluency as a baseline skill — the question isn't "did you use AI" but "do you know exactly where to use it and where not to."
- **It's a more honest demo of the actual job.** Freelance RevOps/automation work in 2026 genuinely involves directing AI tools inside platforms like Activepieces — showing that skill directly is more representative of what a client is hiring for than pretending otherwise.

If you'd rather see judgment than code-typing speed, the sections below — especially the deterministic/AI split and the "what would break at 10x volume" answers — are where to look.

---

## Why four modules, not fifteen

The original brief for this project called for a much broader system — routing, CRM hygiene, forecasting, discount approvals, executive copilots, all in one release. That instinct is understandable but works against a freelance pitch in two ways:

1. **Breadth signals a hobbyist, not an operator.** A real RevOps consultant finds the highest-leverage leak, fixes it, proves the ROI, then expands — not the reverse.
2. **Clients buy stopped bleeding, not a feature list.** What justifies a project fee is a specific dollar amount of leakage the system stops.

So this repo ships the three highest-leverage, most-automatable leaks as fully-realized systems, plus a fourth AI-native layer that ties them together:

| Module | Problem it solves | Automatability |
|---|---|---|
| 1. Speed-to-Lead Router | Leads sit for hours before anyone touches them | ~90% |
| 2. CRM Self-Healing Engine | Duplicate accounts, bad data quietly break everything downstream | ~80% |
| 3. Pipeline Slippage Monitor | Deals go quiet and nobody notices until the deal review | ~75% |
| 4. Morning Revenue Brief | Ties 1–3 together into one daily, dollar-denominated summary | AI synthesis layer |

---

## Deterministic vs. AI, by design

Every module states this split explicitly, because it's the actual technical-credibility question a sharp buyer asks:

- **A rule does it if a rule can do it** — territory routing, stage-entry gates, exact-match dedup, tripwire thresholds. All deterministic, all inspectable, all free to run.
- **AI is reserved for genuinely fuzzy judgment** — normalizing messy free-text into a clean persona tag, deciding whether two company names are probably the same account, writing a plain-English narrative for why a deal looks at risk.
- **AI never has unilateral authority to change data.** The fuzzy-duplicate step flags for human review — it does not auto-merge. Every AI call has a tested, deterministic fallback path that fires cleanly when the AI call fails (which it does in this repo, since it ships with placeholder API keys — see below).

## A note on API keys

No OpenAI or third-party enrichment key is wired into these flows — each AI-dependent step is built as a real, complete HTTP call structure against a placeholder key, with the failure path built and tested (not mocked, not skipped). This was a deliberate choice: it proves the fallback logic actually works, and it means dropping in a real key is the only step left before this runs live for a client.

## Modules

See `ARCHITECTURE.md` for the full flow-by-flow breakdown, `SAMPLE_CLIENT_SCENARIO.md` for a synthetic-data walkthrough, `ROI_ONE_PAGER.md` for the leakage-prevented methodology, and `PRICING.md` for how each module is priced standalone vs. bundled.

## What would break at 10x volume

Worth stating plainly rather than waiting to be asked: the Airtable connector used here has no bulk-list action, so duplicate/fuzzy-match detection in Module 2 relies on single-field substring search rather than a true table scan. It works at demo scale and catches the most common real-world pattern (one company name being a substring of the other), but a client running thousands of leads a month would need a proper indexed lookup or a real database layer underneath — a legitimate, disclosed next step, not a hidden gap.
