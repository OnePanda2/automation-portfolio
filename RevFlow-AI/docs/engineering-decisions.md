# Engineering Decisions

This document explains *why* RevFlow AI is built the way it is, not just what it does. It's written for another engineer evaluating whether these choices were reasonable, and for future-me revisiting the project after time away from it.

## Why Activepieces

Same reasoning as this portfolio's other projects: a visual, inspectable execution trace for every run, native code steps wherever a built-in piece isn't expressive enough, first-class pieces for every external service this project touches (Airtable, Slack), and a free tier that's sufficient at this project's scale.

## Why Airtable instead of the client's stated stack (HubSpot, Postgres, Stripe)

The brief specifies HubSpot, Postgres, Stripe, and OpenAI as the client's existing tools. None of those had a connection available in the Activepieces account this project was built in — only Gmail, Google Sheets, Slack, and Airtable did. Rather than block on credentials that don't exist, Airtable substitutes as a lightweight CRM: real REST API, per-field addressability, zero cost, zero infrastructure. The workflow logic (upsert by email, field updates, status transitions) maps directly onto a real CRM if this were swapped in later.

**One real capability this project has that a pure "everything's mocked" build wouldn't:** Activepieces' own account-level AI credits were available, so lead qualification (Workflow 2) makes a genuine LLM call rather than a placeholder — a meaningfully stronger position than mocking the AI step entirely.

## Why the Opportunities table links to Leads by email, not a formal relational field

Airtable base creation (via the tooling available for this build) defines a base's full table and field schema in one call — there's no way to add a field to a table *after* it's created. A linked-record field requires the *target* table's ID to already exist at the point the field is defined. Since Leads and Opportunities were being created in the same call, there was no way to reference Leads' ID from Opportunities' schema — the referenced table didn't have an ID yet. A shared key (email) is the practical substitute: both sides are populated exclusively by automation, not manual entry, so referential drift isn't a real risk in this build the way it would be with human-entered data.

## Why modular workflows instead of one flow

Six workflows, each independently triggered by its own webhook, rather than one large flow with the same logic inline:

- **Independent testability.** Each stage can be triggered and inspected on its own, with its own execution history.
- **Blast radius.** A broken external connection in one stage (the enrichment vendor, say) doesn't block editing or redeploying any other stage.
- **Matches the client's own stated constraint** — "every workflow should be understandable by another engineer" is a lot easier to satisfy with six small, single-purpose flows than one flow doing six things.

**Tradeoff accepted:** each workflow has to independently re-fetch the lead's current state rather than trusting a copy passed along the chain — see the query-parameter decision below for why this project leans into that rather than fighting it.

## Why query parameters instead of a JSON body for the inter-workflow handoff

The original design passed a JSON body between workflows (the common pattern for Activepieces webhook chaining). In practice, the internal Activepieces-to-Activepieces HTTP calls in this build did not reliably deliver that body — it arrived empty often enough to cause a real, silent bug (documented in [troubleshooting.md](troubleshooting.md#inter-workflow-handoff-arrives-empty)). Query parameters don't depend on body parsing at all, so the fix was to move the one piece of data that needs to travel between workflows (the lead's email) into the URL itself, and have each downstream workflow **re-fetch the lead's current record fresh** rather than trust a value forwarded through the chain. This has a second benefit beyond fixing the delivery bug: every workflow always acts on the lead's *current* state, not a potentially-stale snapshot from several steps back.

## Why territory/priority thresholds are computed in code, not left to the AI

The AI call in Workflow 2 scores fit and intent — a genuinely judgment-based task suited to an LLM. But *which* priority bucket a score falls into, and *which* territory a company size maps to, are fixed business rules with sharp, auditable thresholds (score ≥ 70 → High; company size ≥ 500 → Enterprise). Those are computed in a deterministic code step immediately after the AI call, not asked of the model. A rule that should always produce the same output for the same input shouldn't be delegated to something probabilistic.

## Why routing skips the SDR queue for high-priority leads

The client's team has 5 SDRs and 3 AEs. A well-qualified enterprise-fit lead (high score, right company size) doesn't need to wait in an SDR triage queue — it's routed directly to the AE who owns that lead's territory. Medium/low-priority leads still benefit from SDR-stage qualification and go into a round-robin pool instead. This is a judgment call about the sales process, not a hard requirement in the brief — worth being able to defend if asked, since a reasonable alternative (everything through SDRs first) also exists.

## Why round-robin uses a deterministic hash, not a stored counter

SDR assignment for medium/low-priority leads picks a rep via `hash(email) % 5` against a static list of the 5 SDRs, rather than maintaining a rotating counter in Airtable. This avoids a read-modify-write race condition on shared state (two leads arriving close together could both read the same "next" counter value before either writes back) and requires no extra table. The tradeoff: the SDR list is currently hardcoded in the routing workflow's code step, so a roster change needs a code edit rather than an Airtable row edit — seev[Future Improvements](../README.md#future-improvements).

## Why enrichment fails open, not closed

Workflow 3 calls a firmographic API with a placeholder key (no real vendor account exists for this build). Rather than block the pipeline on that failure, the workflow flags `Enrichment Status: Failed` and lets the lead proceed to routing anyway. A lead should still get routed and announced even when a third-party vendor is down or unconfigured — enrichment should enhance the data available for routing, not gate whether routing happens at all.

## Why Opportunity creation is conditional, not unconditional

Workflow 6 only creates an Opportunity record when the assigned rep is an AE. An SDR-assigned lead is still being qualified further — there's no deal yet to track, so creating an Opportunity record for it would misrepresent the pipeline's actual state (and would need to be manually cleaned up or ignored later, which is exactly the kind of inconsistency the client complained about in the brief).

## Tradeoffs accepted, summarized

| Decision | What was gained | What was given up |
|---|---|---|
| Airtable over the stated CRM stack | No blocked credentials, fast iteration | Not a real CRM; would need a real swap for production use |
| Opportunities linked to Leads by email, not a relation | Buildable within available Airtable tooling | No referential integrity if data were ever entered by hand |
| Query params over JSON body for handoffs | Reliable delivery, always-current data per workflow | Slightly more re-fetching (one extra Airtable read per workflow) |
| Deterministic hash-based round-robin | No shared-state race condition, no extra table | SDR roster is hardcoded; a roster change needs a code edit |
| Enrichment/AI fail open | Pipeline never stalls on a vendor or key outage | Some records carry a "Failed"/default value that needs periodic backfill once real keys are added |

## Future scalability

The architecture doesn't need to change shape to grow — a real CRM swap, a live-queried rep roster instead of a hardcoded list, and a formal linked-record schema (once a table-creation tool supports adding tables to an existing base) are additive changes, not redesigns. See [README.md's Future Improvements](../README.md#future-improvements) for the full list.
