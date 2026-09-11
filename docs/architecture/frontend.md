# Frontend

The app is phone-first. Its most demanding moment is a person standing in a Lidl aisle with one hand free and no signal, so the platform, state model and offline design are chosen for that moment. The wizard, plan and progress screens work anywhere.

## Platform `[D4]`

Expo (React Native) with TypeScript and Expo Router. Targets iOS and Android from one codebase, with a web build for the landing page and wizard demo `[D22]`. EAS Build for store binaries, EAS Update for over-the-air JavaScript updates between store releases.

Why this and not a PWA: app store presence for a consumer product, reliable background sync and push, and the shopping-mode screen is better as a native surface. Why not Flutter: the engine and the API are TypeScript; sharing types and code across the whole stack is worth more than Flutter's rendering polish for two developers working through Claude Code.

## Structure

```
apps/mobile/
  app/                        Expo Router file-based routes
    (onboarding)/
      welcome.tsx
      wizard/[step].tsx       5 steps: goal, you, food, time, stores
      crunch.tsx
    (tabs)/
      plan.tsx  list.tsx  route.tsx  deals.tsx  me.tsx
    meal/[id].tsx             bottom sheet: ingredients, alternatives
    trip/index.tsx            full-screen shopping mode
  features/                   one directory per prototype section
    wizard/  plan/  list/  deals/  route/  trip/  progress/
  lib/
    engine.ts                 thin wrapper: memoised model() + invalidate()
    api/                      typed client generated from packages/contracts
    store/                    local state (Zustand) persisted with MMKV
    query/                    TanStack Query hooks and persisted cache
    sync/                     offline mutation queue
  ui/                         design system: tokens, primitives, icons
```

The mapping from the prototype's sections to feature directories is one-to-one:

| Prototype section | Feature directory | Notes |
| --- | --- | --- |
| `WIZARD` | `features/wizard` | Five steps, same fields, same validation |
| `CRUNCH` | `features/wizard` | The animation; runs while the engine computes |
| `PLAN` | `features/plan` | Hero, day strip, macro bar, meal cards, meal sheet |
| `GROCERY LIST` | `features/list` | By aisle / by store, check-off, total bar |
| `DEALS / OPTIMIZER` | `features/deals` | Compare, stores-per-trip, biggest wins, leaflets |
| `ROUTE` | `features/route` + `features/trip` | Map and stops; shopping mode split out |
| `PROGRESS` | `features/progress` | Weight, spend, streak, change goals |
| `NUTRITION + PLAN`, `HELPERS` | `packages/engine` | Leaves the app entirely |
| `DATA` | seed fixture in `packages/engine` | Becomes test data and the phase 1 catalog |

Ownership follows the directory, per [collaboration/claude-code.md](../collaboration/claude-code.md).

## State model

Three kinds of state, kept in separate stores so the derived one can never be edited by hand:

1. **Inputs**: the profile (goal, body, household, diet, allergens, dislikes, cook time, budget, meals per day, stores, usual store, max stores, travel mode, home location). Persisted locally; synced to the server when signed in.
2. **Overrides**: swaps (as explicit recipe IDs, not offsets as in the prototype), checked items, extras added to the list, items removed, trip progress, the selected day and list mode. Persisted locally; synced.
3. **Derived**: the plan, grocery list, optimisation result and route. Computed by `@grocerai/engine` from inputs + overrides + catalog, memoised, invalidated on any change to the first two. Never persisted on the device except as a cache; the server snapshots it for history.

Server-originated data (catalog bundle, week snapshot, progress history) goes through TanStack Query with a persisted cache so it is available on cold start without network.

## Offline

- The catalog bundle for the user's enabled stores (products, this week's prices, discounts, store geography) is a single JSON file of a few hundred kilobytes, fetched with an ETag and cached. The engine only ever reads from the cached copy. `[D16]`
- Shopping mode reads and writes local state only. Mutations (check item, next stop, complete trip) go into a queue in MMKV and are replayed to the API in order when connectivity returns. Conflicts are resolved last-write-wins per item; the trip belongs to one device.
- A signed-out user has everything except sync, history and push. No screen is gated on an account. `[D17]`

## Design system

Port the prototype's `:root` tokens and the CSS blocks under `/* ---------- tokens ---------- */` into `ui/tokens.ts`: ink, leaf, yellow, red, the greys, radii, and the three type families (Bricolage Grotesque for display, Instrument Sans for body, Azeret Mono for numbers). Primitives, each a direct port of a prototype class:

`Hero`, `Stat`, `DayStrip`, `MacroBar`, `MealCard`, `GroceryItem`, `StoreChip`, `Sticker`, `Sheet`, `Segmented`, `Slider`, `Counter`, `Switch`, `Chip`, `Toast`, `TabBar`, `Stop`, `DoneCard`.

Icons are the prototype's inline SVGs moved to `react-native-svg`. Meal art stays a two-colour gradient until there is photography.

## Navigation

Onboarding stack (welcome → wizard → crunch) then a five-tab layout: Plan, Groceries, Route, Deals, Progress. The meal detail is a bottom sheet over Plan. Shopping mode is a full-screen modal launched from Route; leaving it keeps the trip in progress. Deep links: `grocerai://week` (from the Thursday push) and `grocerai://trip` (from a "trip in progress" notification).

## Maps `[D12]`

MapLibre via `@maplibre/maplibre-react-native` with OpenStreetMap vector tiles. Store pins with the chain colour, numbered badges for stops on the route, a route polyline, a home marker: the prototype's SVG map made real. Distances are straight-line (haversine) to start; when travel time matters, the engine's distance provider is swapped for an OSRM table request without touching the UI.

## Web build

Expo web serves two things: the marketing page and the wizard demo (the job `index.html` does today for the team). The full app is not targeted on web in the first year; the shopping-mode experience does not benefit and the maintenance cost is real. The engine runs in the browser, so the demo computes a live plan exactly as the prototype does.

## Quality

- **Engine**: unit tests in `packages/engine`, including a parity suite that asserts the ported engine reproduces the prototype's outputs on the prototype's seed data. Every planner or optimiser change runs against it.
- **Features**: React Native Testing Library for logic in feature hooks (swap updates list, check-off updates count, trip completion records spend).
- **End to end**: Maestro flows for the three paths in the [README](README.md): onboarding to plan, swap to updated list and route, full trip.
- **CI**: GitHub Actions on every PR: typecheck, lint, engine and feature tests. EAS Build on tags. EAS Update for `main` to an internal channel.
- **Monitoring**: Sentry for crashes. PostHog (EU cloud) for product events; the initial event list is `wizard_completed`, `plan_generated`, `meal_swapped`, `list_viewed`, `trip_started`, `trip_completed`, `savings_shown` with the euro amount. These are what validate the savings claim in beta. `[D20]`

## Performance budgets

- Cold start to a rendered plan under 2 s on a mid-range Android device.
- Engine recompute under 50 ms on the same device. At prototype scale (38 products, 5 stores, 31 store subsets) it is well under 5 ms; with lower-bound pruning it holds to around 15 enabled stores.
- Catalog bundle under 500 KB; fetched at most once per price week.
