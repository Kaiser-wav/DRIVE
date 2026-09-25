# Carpool App — CLAUDE.md

## What this is
A single-file HTML app for splitting driving costs between a regular crew (volleyball practice carpooling). No backend, no build step, no dependencies — just one `fahrtkosten.html` file opened in a browser.

---

## File structure
```
/your-folder/
  fahrtkosten.html   ← the entire app
  CLAUDE.md          ← this file
```

---

## Tech stack
- Pure HTML + CSS + vanilla JS — no frameworks, no npm, no bundler
- Google Fonts loaded via CDN: Onest (everything)
- `localStorage` key: `carpool_v4` — all state persists here
- No external API calls — everything runs client-side

---

## State shape
All app data lives in a single `S` object, serialized to localStorage:

```js
S = {
  members: [{ id: number, name: string, addr: string }],
  trips: [{
    id: number,
    date: string,           // e.g. "26 May"
    km: number,
    totalCost: number,      // fuel cost only
    perPerson: number,      // what each paying passenger owes
    passengers: string[],   // names of paying passengers
    spared: string[],       // names of passengers who rode free
    dir: 'there'|'back',    // which leg (absent on trips logged before this existed)
    ts: number,             // epoch ms — hydrate() backfills it from `id` on old trips
    setup: { crew: number[], overrides: tripOverrides, home, club }, // for "Same as last time"
    rates: { fuelPerKm: number, surchPerKm: number }  // priced-at rates; tripRates() derives them for old trips
  }],
  payments: [{ id: number, ts: number, name: string, amt: number }],  // newest first
  debts: { [name: string]: number },  // positive = owes you, negative = credit
  settings: {
    homeAddr: string,
    clubAddr: string,
    consumption: number,   // L/100km
    gasPrice: number,      // €/L
    surcharge: number,     // €/km extra charged to passengers only
    lastBackup: number     // epoch ms of the last exported backup
  },
  presets: [{ id: number, name: string, km: number }],

  // learned distances — see "Auto distance" below
  routeMemory: { [signature: string]: { km: number, n: number, ts: number } },

  // set when Maps is opened, cleared when the trip is logged or discarded
  pendingDrive: null | {
    startedAt: number, km: number|null, kmSource: string|null,
    direction: 'there'|'back', home: number|null, club: number|null,
    crew: number[], overrides: tripOverrides   // snapshot, see below
  }
}
```

---

## Per-trip overrides (not persisted)
`tripOverrides` is a session-only object reset after each saved trip:
```js
tripOverrides = {
  [memberId]: {
    addr: string|null,          // one-time pickup address override
    cameToMe: null|'spare'|'pays'
    // 'spare'  → excluded from cost split, excluded from Maps route
    // 'pays'   → included in cost split, excluded from Maps route (no detour)
  }
}
```

---

`pendingDrive.overrides` is the one exception to "not persisted" — it's a snapshot so an
armed drive survives the tab being killed when the user switches to Google Maps.

---

## Auto distance (route memory)
The app has no geocoding API, so distances are **learned, not fetched**. Every logged trip
writes its km to `S.routeMemory` under a signature built by `routeSignature()`:

```
"<homeAddr>|<sorted pickup addresses>|<clubAddr>"   — all lowercased/trimmed
```

- Direction is deliberately **not** part of the signature — there and back cover the same
  ground, so a distance learned one way auto-fills the other.
- Members with `cameToMe` set are excluded (no detour → no effect on distance).
- A member with no address falls back to their name, so crew identity still differentiates.

`learnRoutesFromHistory()` rebuilds signatures from past trips (rebuilding the crew from names
for trips without `setup`) and adds only routes memory doesn't know yet. `hydrate()` runs it once
(`settings.routesFromHistory`); Settings → "Learn distances from history" (`relearnRoutes()`) runs it again.

`applyAutoKm()` fills the km field from memory, but only while `kmSource` is `null` or
`'auto'` — a number the user typed (`'manual'`) or picked from a preset (`'preset'`) is
never overwritten. `kmSource` is module-level session state.

## Trip screen layout
The screen reads top to bottom as **set up → see the outcome → act**:

```
1  Who's coming?   crew chips + "same as last time"
2  Route           direction button + alternative from/to chips
3  Distance        km input + auto-fill hint + presets
   This trip       the outcome panel (#preview) — cost, km, fuel, route drawing
   actions         one loud button, two quiet ones
```

The numbered `<span class="step">` markers live in the `.label` of each input section;
the outcome label uses `.label.outcome`. The outcome panel shows as soon as there is a
route to draw or a crew selected — not only once km is filled in — so the km placeholder
state is a real state (`.preview-big.idle`, `renderRouteViz(null)`).

## The three actions
| Button | Function | Behaviour |
|---|---|---|
| Loud, full width | `primaryAction()` | Opens Maps **and** logs the trip in one tap |
| Quiet, left | `logOnly()` | Logs, never touches Maps |
| Quiet, right | `openMapsOnly()` | Opens Maps only — the original workflow, arms the pending banner |

`renderActions()` rebuilds them on every `updatePreview()`, so the primary always carries
the live amount. With no home/club address set, the primary degrades to a plain **Log
trip** and the quiet row becomes a hint pointing at Settings. `buildMapsUrl()` returns
`null` instead of alerting, so callers decide what to do about a missing address.

## One trip = one one-way leg
**A logged trip is always one direction.** The way back is a separate trip, because the
crew going home is often not the crew that came out — someone gets picked up by a parent,
someone stays late. That is what the direction switcher is for, and it is why there is no
round-trip or "log both legs" shortcut anywhere: it would charge the outbound crew for a
ride some of them never took. `logTrip()` takes no leg count.

After an outbound trip is logged, `returnPrompt` (session-only) offers to **set up** the
way back — flipped direction, same crew and km as a starting point, nothing logged until
the user acts. `setupReturn()` restores that into the form; `dismissReturn()` and any
manual `flipDirection()` clear it.

## Pending drive flow
`openMapsOnly()` → `armPendingDrive()` snapshots the trip and persists it → on return (or
a fresh page load, via `restorePendingDrive()`) the crew/route/km are restored and
`renderPending()` shows a banner with one-tap **Log trip** / **Log there + back** /
**Discard** — one leg only. Armed drives older than `PENDING_MAX_AGE` (12h) are dropped
silently.
`primaryAction()` deliberately does *not* arm — it has already logged the trip.

---

## Cost calculation logic
```
fuel cost     = (km / 100) × consumption × gasPrice
surcharge     = surcharge_per_km × km
total people  = paying_passengers + 1 (driver always pays a share)
per passenger = (fuel / total_people) + (surcharge / paying_passengers)
driver pays   = fuel / total_people  (gets surcharge back)
spared people = ride free, not counted in total_people
```

---

## Design system
**"Cobalt" in teal** — an Uber/Bolt-style ride app: map first, controls on floating very round white cards, one accent color.

- **Accent**: `--p #07858B` (OKLCH hue 201) with tints `--p-soft #D4F6F8` and `--p-mid #7BD9DF`; ink `--text #001315`; page `--bg #F0F7F8`; cards white. Free rides use `--free`/`--amber` (orange). No green as an accent — `--green` is only for money coming in.
- **Dark mode** redefines the same tokens under `html[data-theme="dark"]` (text on the accent flips to ink via `--on-p`). The app is light-first: `hydrate()` moves everyone to light once (`settings.design = 'cobalt'`), the Settings toggle still switches.
- **Font**: Onest only — 800 for money and titles, 500–700 for UI.
- **Shapes**: cards 22–26px radius, pills/avatars fully round, soft shadows (`--shadow`, `--shadow-sm`). No gradients except the hue-free map drawing.
- **Trip screen**: `.map-hero` (stylised street map from `renderRouteViz()`) with a floating `.route-card`, then `.main-card` overlapping it: crew pills with avatars → distance → price → **slide to drive** (`#slide`, `initSlide()`, `slideDone()`). Sliding, not tapping, is the primary action — no accidental logs.
- **Nav**: floating dark capsule of 5 icon buttons (`nav`, fixed), active = accent circle.
- **Settings** is all tap-to-edit rows plus one "+ Add …" row per list (`.srow`, `.addrow`) that opens a sheet (`openAddMember()`, `openAddPlace()`, `openAddPreset()`, `editMainPlace()`) — no permanently visible add forms.
- **Route card** (`.route-card`) sits in the flow at the top of `.map-hero`; the hero grows under it when "Other start or destination" opens, and the drawing (`.route-viz`) stays pinned to the bottom 300px — so nothing below can be overlapped.
- **History** rows (`.hrow`): direction icon · names + "Return · 21.7 km" · amount + date. Month headers (`.hmonth`) show trips and km.
- **Sheets** all open/close through `showOverlay(id)` / `hideOverlay(id)` (slide-down close, page scroll lock via `html.locked`). `initSheetDrag()` adds pull-down-to-close; a new overlay needs an entry in `OVERLAY_CLOSE`.
- **Swipe between screens** (`initScreenSwipe()`, nav order in `SCREEN_ORDER`). Anything with its own sideways gesture must opt out — `#slide`, inputs, or `data-noswipe`.
- **Toasts** last 2.6s (5s with Undo, with a countdown bar), dismiss on tap, and plain ones clear on screen change.
- **Chip states**: accent outline = paying passenger, orange = free ride, dashed accent = came to you but pays, grey = not in the car.
- All colors are CSS variables in `:root` — never hardcode hex in new code. Old names (`--muted`, `--amber`, `--border`, `--cyan`, `--coral`) are kept as aliases.

---

## Key functions
| Function | What it does |
|---|---|
| `renderTrip()` | Re-renders crew chips, preset chips, calls `applyAutoKm()` + `updatePreview()` |
| `updatePreview()` | Calculates split, renders the outcome panel + route viz SVG, calls `renderActions()` |
| `renderActions()` | Rebuilds the loud/quiet button stack with the live amount |
| `primaryAction()` / `logOnly()` / `openMapsOnly()` | The three actions — see the table above |
| `buildMapsUrl()` | Builds the Maps URL, or `null` if home/club addresses are missing |
| `calcSplit(km)` | **Single source of truth for the split math** — used by preview, banner and `logTrip()` |
| `routeSignature()` / `rememberRoute(km)` | Build the route key / store the learned km |
| `learnRoutesFromHistory()` / `relearnRoutes()` | Backfill route memory from logged trips (once on load / from Settings) |
| `applyAutoKm()` / `renderKmHint()` | Auto-fill km from memory / explain where the number came from |
| `armPendingDrive()` / `restorePendingDrive()` / `renderPending()` | The open-Maps-then-log-on-return flow |
| `repeatLastTrip()` | Restores the crew + km of the most recent trip in one tap |
| `setupReturn()` / `dismissReturn()` / `renderReturnPrompt()` | The offer to set up the way back as its own trip |
| `renderRouteViz(km)` | Draws the generated town map + route home → pickups → club, zooming to fit |
| `openChipModal(id)` | Opens per-person config modal |
| `setCameToMe(mode)` | Sets spare/pays state in chipModalTemp |
| `closeChipModal(confirm)` | Writes chipModalTemp to tripOverrides if confirmed |
| `logTrip()` | Saves one one-way trip, updates debts, learns the route km, resets session state |
| `openPayModal(name)` | Opens debt payment modal for a crew member |
| `persist()` / `hydrate()` | Save/load S to localStorage |
| `renderLog()` | Log screen: trips and payments in one timeline, grouped by month |
| `deletePayment(id)` | Removes a payment and adds its amount back to the balance |
| `exportBackup()` / `importBackup(input)` | Download S as JSON / replace S from a backup file and reload |
| `toast()` / `withUndo()` | Feedback + undo for every data change |
| `openSheet()` / `closeSheet()` / `confirmSheet()` / `formSheet()` | One generic bottom sheet for edit forms, confirms and the person view |
| `toggleMember(id)` | Crew chip tap = in/out of the car; the chip's ⋯ opens `openChipModal()` |
| `tripDate` / `renderDatePill()` / `onTripDate()` | Session-only: log a trip on a past day (skips Maps) |
| `openPerson(name)` / `debtBreakdown(name)` | Debt detail: which rides make up the balance (payments settle oldest first) |
| `sendRequest(name)` / `copyRequest(name)` | Payment reminder via share sheet → WhatsApp link fallback / clipboard |
| `editTrip()` / `saveTripEdit()` / `editPayment()` / `editMember()` / `editPreset()` / `editAltAddr()` | Edit sheets — every list row in Log and Settings is tappable |
| `renameEverywhere(from,to)` | Moves debts, trips and payments to a member's new name |
| `renderStats()` / `renderWeekChart()` | Stats screen: week/month/all totals, 8-week stacked bar chart (crew paid vs your share), per-person |
| `esc(s)` / `fmtDate(ts)` | Escape user text for innerHTML / format a timestamp (adds year if not this year) |

---

## Google Maps URL pattern
```
https://www.google.com/maps/dir/{home}/{stop1}/{stop2}/{club}
```
Stops are URL-encoded addresses. Members with `cameToMe` set (either mode) are excluded from waypoints — their address doesn't affect the route.

---

## Things to keep in mind when editing
- **Always `esc()` names, labels and addresses** before putting them in innerHTML. Never inline a name into an `onclick` string — use `data-name` + `this.dataset.name` (apostrophes broke the Debts screen before)
- **Display dates from `ts`** via `fmtDate()`, not the legacy `date` string (it has no year)
- **Changes to saved data go through `withUndo(msg, mutate)`** — it snapshots S, applies, saves, re-renders and shows an Undo toast. Bulk resets additionally ask via `confirmSheet()` first. No `alert()`/`confirm()` anywhere — use `toast(msg, {error:true})` and `confirmSheet()`
- **Keep `S.trips` and `S.payments` sorted newest-first** (`sortHistory()`) — `debtBreakdown()` and "Same as last time" rely on it
- **Editing a trip reprices it at its own `rates`**, never the current settings — `splitFor()` is the same formula as `calcSplit()`, keep them identical
- The floating nav, the toast (`#toast-wrap`) and the overlays are `position:fixed` — nothing else is
- **localStorage key is `carpool_v4`** — if you change the state shape significantly, bump this to `carpool_v5` to avoid hydration errors from old saved data. Purely *additive* keys (like `routeMemory`) don't need a bump — `hydrate()` spreads over the defaults — and bumping would orphan the user's real debt balances, so don't do it lightly
- **`calcSplit()` is the only place the split formula lives** — preview, pending banner and `logTrip()` all call it, so they can't drift apart
- **Route map** (`renderRouteViz()`): `buildTown()` generates a made-up town (warped street grid, blocks, parks, river, one avenue) seeded from home+hall address and cached in `mapTown`. Home is anchored at `MAP_HOME`; the hall sits `4 + 2×pickups` columns away, so every pickup lengthens the route and the camera zooms out. Pickups spread along the way in pickup order, nudged by `townSpot()` (hash of their address). The town is one `<g>` moved by a camera matrix (`vizCam`, tweened in log space); the route and pins are drawn in screen space so they stay crisp. Road strokes use `vector-effect:non-scaling-stroke`. Map colors: `--map-bg/-road/-block/-park/-water`.
- **Avoid new `position:fixed`** — only the nav, toast and overlays use it; the app runs as a local file / mobile browser page where extra fixed layers misbehave
- **Single file constraint** — keep everything in one HTML file; don't split into separate CSS/JS files unless the user explicitly asks to set up a proper project

---

## Common tasks
**Add a new field to member profiles** → update `addMember()`, the member object shape, `renderSettings()` members-list HTML, and `renderTrip()` chip modal

**Never add a round-trip shortcut** → see "One trip = one one-way leg"; the crew can differ per direction

**Change the split formula** → edit `calcSplit()` only — preview, banner and `logTrip()` all read from it

**Add a new chip visual state** → add CSS class, add color vars if needed, update `renderTrip()` chip class logic

**Change the color palette** → update CSS vars in `:root` only — everything references vars

**Add a new screen** → add nav button, add `<div id="s-newscreen" class="screen">`, add case in `showScreen()`
