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
- Google Fonts loaded via CDN: Fraunces (display) + Instrument Sans (body)
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
    dir: 'there'|'back'     // which leg (absent on trips logged before this existed)
  }],
  debts: { [name: string]: number },  // positive = owes you, negative = credit
  settings: {
    homeAddr: string,
    clubAddr: string,
    consumption: number,   // L/100km
    gasPrice: number,      // €/L
    surcharge: number      // €/km extra charged to passengers only
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

`applyAutoKm()` fills the km field from memory, but only while `kmSource` is `null` or
`'auto'` — a number the user typed (`'manual'`) or picked from a preset (`'preset'`) is
never overwritten. `kmSource` is module-level session state.

## Pending drive flow
`openMaps()` → `armPendingDrive()` snapshots the trip and persists it → on return (or a
fresh page load, via `restorePendingDrive()`) the crew/route/km are restored and
`renderPending()` shows a banner with one-tap **Log trip** / **Log there + back** /
**Discard**. Armed drives older than `PENDING_MAX_AGE` (12h) are dropped silently.

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
- **Colors**: dark warm bg `#1a1612`, coral accent `#e8674a`, cyan `#4ecdc4`, green `#7bc67a`, amber `#e8b96a`
- **Fonts**: Fraunces (italic, light 300) for big numbers and titles — Instrument Sans for all UI text
- **Chip states**: coral = normal passenger, amber = spared (free), cyan = came-to-me but pays, default = not selected
- **No gradients, no shadows** — flat surfaces only
- All colors defined as CSS variables in `:root` — always use vars, never hardcode hex in new code

---

## Key functions
| Function | What it does |
|---|---|
| `renderTrip()` | Re-renders crew chips, preset chips, calls `applyAutoKm()` + `updatePreview()` |
| `updatePreview()` | Calculates split, renders big number + route viz SVG |
| `calcSplit(km)` | **Single source of truth for the split math** — used by preview, banner and `logTrip()` |
| `routeSignature()` / `rememberRoute(km)` | Build the route key / store the learned km |
| `applyAutoKm()` / `renderKmHint()` | Auto-fill km from memory / explain where the number came from |
| `armPendingDrive()` / `restorePendingDrive()` / `renderPending()` | The open-Maps-then-log-on-return flow |
| `repeatLastTrip()` | Restores the crew + km of the most recent trip in one tap |
| `renderRouteViz(km)` | Draws the SVG path from home → pickups → club |
| `openChipModal(id)` | Opens per-person config modal |
| `setCameToMe(mode)` | Sets spare/pays state in chipModalTemp |
| `closeChipModal(confirm)` | Writes chipModalTemp to tripOverrides if confirmed |
| `logTrip(legs)` | Saves 1 or 2 legs, updates debts, learns the route km, resets session state |
| `openMaps()` | Builds Google Maps URL with waypoints, arms the pending drive, opens in new tab |
| `openPayModal(name)` | Opens debt payment modal for a crew member |
| `persist()` / `hydrate()` | Save/load S to localStorage |

---

## Google Maps URL pattern
```
https://www.google.com/maps/dir/{home}/{stop1}/{stop2}/{club}
```
Stops are URL-encoded addresses. Members with `cameToMe` set (either mode) are excluded from waypoints — their address doesn't affect the route.

---

## Things to keep in mind when editing
- **localStorage key is `carpool_v4`** — if you change the state shape significantly, bump this to `carpool_v5` to avoid hydration errors from old saved data. Purely *additive* keys (like `routeMemory`) don't need a bump — `hydrate()` spreads over the defaults — and bumping would orphan the user's real debt balances, so don't do it lightly
- **`calcSplit()` is the only place the split formula lives** — preview, pending banner and `logTrip()` all call it, so they can't drift apart
- **SVG route viz** is built dynamically in `renderRouteViz()` — viewBox is `0 0 460 110`, nodes spaced evenly across the width with `pad=36`
- **No `position:fixed`** anywhere — the app is designed to be saved as a local file and opened in a mobile browser; fixed positioning causes issues in some mobile browsers
- **Single file constraint** — keep everything in one HTML file; don't split into separate CSS/JS files unless the user explicitly asks to set up a proper project

---

## Common tasks
**Add a new field to member profiles** → update `addMember()`, the member object shape, `renderSettings()` members-list HTML, and `renderTrip()` chip modal

**Change the split formula** → edit `calcSplit()` only — preview, banner and `logTrip()` all read from it

**Add a new chip visual state** → add CSS class, add color vars if needed, update `renderTrip()` chip class logic

**Change the color palette** → update CSS vars in `:root` only — everything references vars

**Add a new screen** → add nav button, add `<div id="s-newscreen" class="screen">`, add case in `showScreen()`
