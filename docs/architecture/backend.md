# Backend

The backend has three jobs: deliver a fresh, matched catalog of prices every week; keep signed-in users' profiles, snapshots and history; and run the engine server-side so the Thursday push can say a number. Everything else is on the device.

## Shape `[D5]` `[D6]` `[D7]` `[D9]`

- **`apps/api`**: Node 22, TypeScript, Fastify. A modular monolith: one deployable, internal modules with their own tables and a service interface. Cross-module access goes through the service, not the other module's tables. Modules: `identity`, `profile`, `catalog`, `pricing`, `recipes`, `planning`, `trips`, `progress`, `notifications`, `admin`.
- **Job runner**: pg-boss, a Postgres-backed queue in the same process (or a second instance of the same image with a `worker` flag). Scheduled jobs: weekly rebuild, catalog publish, stale-price sweep. No Redis.
- **`apps/ingestion`**: Python 3.12. One adapter per chain, a shared parse/normalise/match pipeline, a publish step. Writes to staging tables with a dedicated database role; promotion to live prices is a single API call so the API owns the invariant.
- **Database**: PostgreSQL 16 with PostGIS on Supabase, Frankfurt. Drizzle for schema and migrations `[D10]`. Row-level security on user tables as defence in depth behind the API's own checks.
- **Auth**: Supabase Auth `[D8]`. Apple, Google, email magic link. The API verifies the JWT; it never handles passwords.
- **Storage**: Supabase Storage. Raw scrape snapshots and leaflet PDFs (kept 12 months for re-parsing), and the published catalog bundles served through the CDN.
- **Hosting**: API and worker as containers on Fly.io in Frankfurt or Amsterdam. Two small machines through beta.

## API

REST, JSON, versioned under `/v1`. Request and response schemas are zod objects in `packages/contracts`, from which the app's client and an OpenAPI document are generated. Plain REST rather than tRPC because ingestion (Python), the admin tooling and any future retailer integration also need to call it.

| Endpoint | Purpose |
| --- | --- |
| `GET /v1/catalog/bundle?stores=lidl,hofer,spar` | The week's products, prices and discounts for a store set. ETag, CDN-cached, redirects to the static bundle in Storage |
| `GET /v1/stores?near=lat,lng&radius=km` | Stores with chain, address, geography, hours |
| `GET /v1/recipes/bundle` | The recipe corpus, versioned, CDN-cached |
| `GET` / `PUT /v1/me/profile` | Inputs |
| `GET /v1/me/weeks/current` | This week's snapshot |
| `POST /v1/me/weeks/current/overrides` | Append swaps, checks, extras, removals, max-stores changes |
| `POST /v1/me/weeks/current/rebuild` | Re-run the engine server-side after a profile change |
| `GET /v1/me/weeks?before=` | History |
| `POST /v1/me/trips`, `PATCH /v1/me/trips/:id`, `POST /v1/me/trips/:id/complete` | Shopping mode sync; idempotent by client-generated ID so the offline queue can replay |
| `POST /v1/me/weights`, `GET /v1/me/progress` | Weight log and the progress screen's numbers |
| `POST /v1/me/devices` | Expo push token |
| `GET /v1/me/export`, `DELETE /v1/me` | GDPR from day one |
| `/v1/admin/matching/queue`, `/v1/admin/ingestion/runs` | Review queue and run dashboard; behind an admin role |

All `/me` routes require a JWT. Catalog, stores and recipes are public and cacheable: they are the same for every user with the same store set.

## The engine (`packages/engine`) `[D11]`

Pure TypeScript with no I/O. Given `(profile, overrides, catalog, recipes, seed)` it returns `(plan, groceries, optimisation, route)` and is deterministic. It runs on the device for instant interactions and in the API for the weekly rebuild, from the same package version. Ported from the prototype's `NUTRITION + PLAN` section, then improved where the prototype cut corners:

| Module | Ported from prototype | Improvements |
| --- | --- | --- |
| `nutrition.ts` | `kcalTarget`, `servings`, `SLOT_SHARE` | Macro targets by goal; kid portion factor configurable |
| `planner.ts` | `allowedMeal`, `pool`, `mealFor`, `plan` | Score-and-pick instead of modular indexing: kcal fit, variety across the week, ingredient reuse (less waste, fewer packs), cost. Seeded, so a week is reproducible. Swaps stored as recipe IDs |
| `grocery.ts` | `groceries` | Pantry staples (oil, honey, spices) flagged "have it" and not rebought weekly, once the pantry feature exists |
| `optimizer.ts` | `optimize` | Exact search over store subsets up to `maxStores` with lower-bound pruning; products not stocked at a store cost infinity there; ties broken toward fewer stops |
| `router.ts` | `routeFor` | Nearest neighbour plus 2-opt on up to four stops; the distance function is injected (haversine now, an OSRM table later) |

The prototype's `DATA` section becomes the engine's seed fixture: 5 stores, 38 products, 27 recipes, 14 discounts. A parity test asserts the port reproduces the prototype's plan, list, store set and route for the default profile before any improvement lands.

## Data model

Core tables by module. Types and relations only; column details live in `packages/db`.

```
identity      users                  Supabase auth.users; nothing else stored here
profile       profiles               user_id, goal, sex, age, height_cm, weight_kg, target_kg, activity,
                                     diet, allergens[], dislikes[], cook_minutes, weekly_budget,
                                     meals_per_day, adults, kids, travel_mode, max_stores,
                                     usual_store_id, enabled_store_ids[], home geog, consent_health_at

catalog       chains                 id, name, colour, website
              stores                 id, chain_id, name, address, geog (PostGIS), hours, active
              products               id, name, category, unit_kind (weight | count), default_pack_size,
                                     nutrition_per_100 jsonb                      -- canonical, ~40 to start
              store_products         id, chain_id, sku, name_raw, brand, pack_size, image_url,
                                     product_id (nullable), match_confidence, matched_by, reviewed_at

pricing       prices                 store_product_id, chain_id, store_id (nullable), price, was_price,
                                     promo_pct, valid_from, valid_to, run_id
              ingestion_runs         id, chain_id, source, started_at, finished_at, status,
                                     stats jsonb, snapshot_path
              catalog_bundles        week_start, store_set_key, path, etag, published_at

recipes       recipes                id, slug, name, slot, minutes, tags[], allergens[], kcal, protein,
                                     carbs, fat, steps, art, status, version
              recipe_ingredients     recipe_id, product_id, amount_per_serving, unit, optional

planning      week_plans             id, user_id, week_start, engine_version, target_kcal, servings,
                                     inputs_snapshot jsonb, result_snapshot jsonb, created_at
              plan_overrides         week_plan_id, seq, kind (swap | check | extra | remove | max_stores),
                                     payload jsonb, created_at

trips         trips                  id (client-generated), week_plan_id, store_ids[], route jsonb,
                                     status, started_at, completed_at, spend_total
              trip_items             trip_id, product_id, store_id, packs, price, checked

progress      weight_logs            user_id, logged_at, weight_kg
notifications devices                user_id, expo_push_token, platform, last_seen_at
```

Notes:

- Prices are per chain by default: Slovenian chains price nationally with few exceptions. `store_id` is filled only when a store deviates, and the engine looks up store first, chain second.
- `result_snapshot` stores what the user actually saw. History survives engine changes and price corrections.
- `plan_overrides` is an append-only log per week, replayed onto the engine's output. It is the server-side twin of the app's overrides store and makes the offline queue trivial to reconcile.
- Trips carry client-generated UUIDs so the offline replay is idempotent.

## Price ingestion pipeline `[D3]` `[D15]`

This is the moat and the biggest risk. Every downstream number depends on knowing what roughly forty products cost at five chains every week. The pipeline is designed for that scope, not for full-catalog coverage.

Cadence: weekly, keyed to leaflet cycles. Most Slovenian chains publish Thursday for Thursday to Wednesday; the schedule is per adapter.

```
retailer site / leaflet ─▶ fetch ─▶ parse ─▶ normalise ─▶ match ─▶ stage ─▶ publish ─▶ bundle
                            │                              │
                       raw snapshot                  review queue
                        (Storage)                   (low confidence)
```

1. **Fetch.** One adapter per chain. Online shops with prices (Mercator, Spar, Tuš) are fetched as HTML or their product APIs; discounters without an online shop (Lidl, Hofer) are fetched as leaflet PDFs and images. Every fetch is saved raw to Storage under its run ID before parsing, so a parser bug never means a re-fetch. Rate-limited, identified user agent, `robots.txt` respected. Legal review of each chain's terms before phase 2, and retailer partnerships pursued in parallel so this is not the only source forever.
2. **Parse.** Adapter output is a `RawOffer`: chain, raw name, brand, pack, price, previous price, promo percentage, validity window, image. HTML parsing is code. Leaflet pages go through an LLM vision extraction against that schema, validated, with a sampled spot-check per run.
3. **Normalise.** Units, pack sizes, price per kilogram or litre, promo percentage derived when only prices are given.
4. **Match.** Raw name to canonical `product_id`, in order: an exact hit on a previously matched SKU (the matching memory, which is most of the work from week three on); an embedding nearest-neighbour search over canonical products plus an LLM yes/no with a confidence; below the threshold, the human review queue in the admin UI. The queue is a table and a page, not a product.
5. **Stage and publish.** Matched offers land in staging. The publish step validates coverage, writes `prices` with the validity window, records the run, and materialises the catalog bundles: one JSON file per store-set key that the app fetches from the CDN.
6. **Monitor.** Per-run statistics: offers found, matched percentage, price moves over 30 % flagged for review, and the KPI that matters: coverage, the percentage of recipe products with a live price at each enabled chain. Alert under 90 %.

Failure mode: if a chain's run fails, last week's prices are carried forward with a `stale` flag that the app can surface ("Lidl prices from last week"). The week still builds.

Python because the scraping, PDF and embedding ecosystems are there. It talks to the rest of the system only through staging tables (its own database role) and the publish endpoint.

## Weekly rebuild job

After publish, for each signed-in user with a device token: load profile and this week's overrides, run the engine, write the `week_plans` snapshot, send one push with the store count and the saving against the usual store. Batched, idempotent per user and week, retried by pg-boss. At beta scale this is seconds of work; the design holds to tens of thousands of users on the same two machines because the engine is cheap and the catalog is shared.

## Security and privacy `[D18]`

- EU region for everything: database, storage, hosting, analytics, error tracking.
- Weight, target weight, goal and allergens are health data under GDPR Article 9. Explicit consent on wizard step 2, recorded as `consent_health_at`. The wizard works without them: goal-based defaults produce a plan, and the app says so.
- Export and delete endpoints ship in phase 3 with accounts, not later. Delete cascades through every `user_id` table and revokes the auth user.
- Secrets in Fly secrets and Supabase Vault. `.env.example` lists every variable, per [pitfalls.md](../collaboration/pitfalls.md).
- No advertising SDKs. Analytics events carry no body metrics, only the euro amounts shown.

## Observability

Sentry on the API and the worker. Structured JSON logs to Fly's log drain. A `/health` endpoint that reports catalog freshness per chain, which is the one thing that silently rots. Alerts on: ingestion run failed, coverage under 90 %, rebuild job errors, push failure rate.

## Environments

`local` (Supabase CLI and Docker), `staging`, `production`. Migrations run from CI with Drizzle Kit; a migration that touches a `user_id` table needs a second reviewer. Seed data for local and staging is the engine fixture, so a fresh environment produces the prototype's numbers on first run.
