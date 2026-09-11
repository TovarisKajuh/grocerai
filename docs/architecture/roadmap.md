# Roadmap and risks

Durations assume two developers working through Claude Code, roughly full time, with the module split below. Calendar dates are indicative from a mid-September 2026 start. Phases 1 and 2 overlap because they touch different directories.

## Phases

| Phase | What ships | Duration | Indicative dates | Exit criterion |
| --- | --- | --- | --- | --- |
| **0. Decide and scaffold** | Decisions D1 to D11 resolved. Monorepo, CI, engine package with the prototype's seed data and the parity test. | 1 week | 15–19 Sep | Parity test green: the ported engine reproduces the prototype's plan, list, store set and route |
| **1. Local-first app** | The whole prototype as a real app on the seed catalog: wizard, plan, list, deals, route, shopping mode, progress. No backend, no account. Internal TestFlight and Play internal track. | 4–5 weeks | 22 Sep – 24 Oct | Both developers use it for a real shop |
| **2. Real prices** | Ingestion for the five chains, matching with review queue, catalog bundles on the CDN, app switches from seed to live catalog with the stale fallback. | 5–6 weeks, overlapping phase 1 from its second week | 29 Sep – 7 Nov | Coverage of recipe products at or above 90 % for three consecutive weeks |
| **3. Accounts and the weekly rhythm** | Supabase Auth, profile and override sync, week snapshots and history, Thursday rebuild with push, trips recording spend, weight logging, export and delete. | 3–4 weeks | 10 Nov – 5 Dec | A signed-in user gets the Thursday push with a real saving |
| **4. Ljubljana beta** | 50 to 200 users. Recipe corpus to 100 or more. Receipt check on savings claims. OSRM travel times. Store hours. | Ongoing from | 8 Dec | Savings shown match receipts within 10 % for most users |
| **Later** | More cities, pantry tracking, receipt scanning, shared household lists, retailer partnerships, LLM-assisted recipe expansion `[D1]`. | | 2027 | |

Four months from scaffold to beta. The ordering is the point: phase 1 is usable without a single server, so the team can validate the product loop while the hard data work in phase 2 is under way.

## Team split

Mirrors module ownership in [collaboration/claude-code.md](../collaboration/claude-code.md).

| | Owns | Reviews |
| --- | --- | --- |
| Developer A | `apps/mobile`, `packages/engine`, `packages/contracts` | Developer B's PRs |
| Developer B | `apps/api`, `apps/ingestion`, `packages/db` | Developer A's PRs |

`packages/contracts` is the seam. A change to a contract is a small PR that lands first and both sides build on it.

## Non-functional requirements

| Requirement | Target through beta |
| --- | --- |
| Data residency | EU only; Frankfurt for database, storage, hosting, analytics |
| Offline | Shopping mode fully functional with no connectivity; the current week available on cold start |
| Catalog freshness | Published within 24 hours of each chain's leaflet; stale flag after 8 days |
| Coverage | At least 90 % of recipe products priced at each enabled chain |
| Latency | Catalog bundle from CDN under 300 ms; engine recompute on device under 50 ms; API p95 under 200 ms |
| Availability | Best effort; the app works from cache if the API is down |
| Cost | Infrastructure under 100 € a month through beta: Supabase Pro, two Fly machines, Sentry and PostHog free tiers, LLM matching a few euros per week |
| Privacy | Health data consent, export and delete before the first external user |

## Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Price data access: terms of service, legal exposure, site changes breaking adapters | High | High | Legal review per chain before phase 2. Adapters isolated per chain with raw snapshots kept. Stale carry-forward so one broken chain does not break the week. Partnerships pursued in parallel |
| Product matching quality: wrong product priced, wrong pack size | Medium | High | Scope to the ~40 recipe products, not full catalogs. Human review queue below confidence threshold. Coverage and price-move alerts. Matching memory reduces review load weekly |
| Recipe corpus too small: weeks feel repetitive, users churn | High | Medium | Variety scoring in the planner. Corpus to 100 or more before beta with LLM-assisted drafting and dietitian review `[D14]` |
| Health data compliance | Low | High | Consent, EU residency, skippable body metrics, export and delete in phase 3 `[D18]` |
| Two-developer bandwidth; the plan slips | Medium | Medium | Phase 1 ships value with no backend. Managed infrastructure. Module ownership keeps the two streams from blocking each other |
| Savings claims not credible to users | Medium | High | Always against the user's own usual store. Receipts collected in beta to validate. `savings_shown` events compared with recorded spend |
| App store review delays | Medium | Low | Internal tracks from phase 1; store listing prepared during phase 3 |
| Engine outputs differ between device and server | Low | Medium | Single package version pinned by the monorepo; snapshots carry `engine_version`; the parity suite runs on both targets |

## Open decisions

All in [decisions.xlsx](decisions.xlsx). D1 to D11 block phase 0 and 1; the rest can wait for the phase that needs them, listed in the sheet.
