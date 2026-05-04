# bpost RouteOptimizer — Demo

A single-file, client-side prototype of a Citrix GeoRoute–style route
optimizer for bpost mail delivery in Brussels. Built for stakeholder demos
and as a scaffold for a production integration.

> **One file.** `route-demo.html` contains all HTML, CSS, and JavaScript.
> No build step, no bundler, no API keys, no backend.

---

## How to run

1. Clone or download this repo.
2. Open `route-demo.html` directly in any modern browser (double-click works).
3. The map loads tiles from OpenStreetMap and Leaflet from unpkg — an internet
   connection is required the first time.

That's it. There is no `npm install`, no dev server, no `.env` file.

### Using the demo

- Pick a vehicle (Fiets / Bestelwagen / eBestelwagen).
- Move the **Afstand ↔ Tijd** slider to bias the objective function.
- Toggle **Respecteer tijdsvensters** and **Alleen prioritaire post**.
- Choose how many mock stops to generate (10–250).
- Click **Optimaliseer route**. The polyline animates as the route is drawn.
- Click **Reset (nieuwe stops)** to regenerate a fresh randomized stop set.
- Click a stop in the right-hand schedule to zoom to it; hover to highlight
  it on the map.

---

## What this demo proves

- The full UX shape works: three-pane layout, vehicle selection, objective
  slider, time-window toggle, animated route, ordered schedule, KPIs.
- The internal contract is correct: `RoutingEngine.optimize(stops, vehicle,
  options)` returns `{ orderedStops, totalDistanceKm, totalDurationMin,
  violations[] }` — exactly what a production UI would consume.
- Vehicle-specific constraints are enforced (capacity, parcel-type
  compatibility) and surfaced as **schendingen**.
- Time windows are evaluated as **soft constraints** with violations
  logged per-stop and visible on the map and in the schedule.
- Performance is acceptable up to 250 stops with pure JS (Nearest Neighbour
  + 2-opt on a haversine matrix).

## What this demo does **not** prove

- **Real road distance.** It uses straight-line haversine, not OSRM. Routes
  that look short on the map may be long on real Brussels streets.
- **Real ETAs.** Travel time = distance / vehicle speed. No traffic, no
  one-ways, no service-area polygons.
- **Optimality.** NN+2-opt is a fast local heuristic. OR-Tools' VRP solver
  will find materially better tours, especially with hard constraints.
- **Multi-day / multi-route planning.** Single vehicle, single shift only.
- **Accurate addresses.** The "fakeAddress" field uses real Brussels street
  names but random numbers and communes — they will not geocode.
- **Production data privacy.** No real customer data is loaded. The demo
  does not exercise GDPR controls, audit logging, or auth.

---

## SWAP POINTS

The code is sectioned and explicitly marked so a future dev can replace
each piece without rearchitecting. Search the file for `SWAP POINT:`.

### 1. Distance matrix → OSRM `/table`

**Where:** `RoutingEngine.buildDistanceMatrix` in `route-demo.html`.
**Today:** O(n²) haversine table.
**Production:** one POST to a private OSRM container running a Belgium PBF
extract:

```
POST http://osrm.internal:5000/table/v1/driving/<polyline>
   ?annotations=duration,distance
```

Cache results per stop-set hash. Fall back to haversine if OSRM is down.

### 2. Solver → OR-Tools VRP

**Where:** `RoutingEngine.optimize` in `route-demo.html`, around the
`nearestNeighbour` + `twoOpt` calls.
**Today:** greedy nearest-neighbour + 2-opt local search on the cost
matrix.
**Production:** Google OR-Tools VRP solver — either compiled to WASM and run
in-browser for < 100 stops, or called as a backend microservice for larger
plans. Feed it: cost matrix, vehicle capacities, demands, time windows,
service times, multiple depots, driver shifts.

### 3. Mock stops → bpost CSV / KLARA import

**Where:** `generateStops()` in `route-demo.html`.
**Today:** seeded RNG produces fake stops with random Brussels coordinates.
**Production:** parse the daily KLARA CSV / API response (geocoded postal
addresses + parcel manifest). Reject malformed rows up front; surface
geocoding failures as filtered-count violations, the same way
vehicle-incompatible stops are surfaced today.

### 4. Vehicles → fleet API

**Where:** the static `VEHICLES` object.
**Today:** three hard-coded vehicles.
**Production:** GET from the fleet management system: live availability,
current eVan battery, driver shift, allowed zones.

### 5. (Optional) NPS / complaints overlay

Not implemented. Add a Leaflet heatmap layer fed from a complaints feed so
planners can bias routing away from problem zones.

---

## 5-step path to production

1. **Stand up OSRM** in Docker on a Belgium extract. Expose `/table` on
   the internal network. Replace `buildDistanceMatrix`. Verify totals
   change vs. haversine.
2. **Replace solver** with OR-Tools (start with backend microservice;
   optimise to WASM later). Keep the `optimize()` signature stable so the
   UI does not change.
3. **Wire the data import** — parse KLARA, run geocoding fallbacks,
   show filtered/rejected counts in the existing violations list.
4. **Plug in fleet API** — drop the static `VEHICLES` and load from
   the fleet system; respect driver shifts as additional time-window
   constraints.
5. **Productionise the UI** — split the single HTML file into a small
   framework (e.g. Lit or Svelte) only when you actually need
   multi-route planning, persistence, scenario comparison, or auth.
   Until then, the single-file footprint is a feature, not a debt.

---

## File layout inside `route-demo.html`

```
<head>           Leaflet 1.9 CDN, brand-token CSS, layout/components/markers
<body>           topbar · sidebar · map · right panel (collapsible) · toast
<script>
  TODO / PRODUCTION INTEGRATIONS  (top-of-file checklist)
  Constants           BRUSSELS, VEHICLES, address pools
  /* ===== DATA GENERATION ===== */
    generateStops(n, seed)      <- SWAP POINT: KLARA import
  /* ===== ROUTING ENGINE ===== */
    buildDistanceMatrix         <- SWAP POINT: OSRM /table
    nearestNeighbour, twoOpt    <- SWAP POINT: OR-Tools VRP
    optimize(stops, vehicle, options)
  /* ===== UI ===== */
    Leaflet init, renderers, controls, boot
```

---

## Browser support

Tested against modern Chromium, Firefox, and Safari. No polyfills required.
Requires JavaScript and an internet connection for tiles.
