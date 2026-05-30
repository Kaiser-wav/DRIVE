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
    spared: string[]        // names of passengers who rode free
  }],
  debts: { [name: string]: number },  // positive = owes you, negative = credit
  settings: {
    homeAddr: string,
    clubAddr: string,
    consumption: number,   // L/100km
    gasPrice: number,      // €/L
    surcharge: number      // €/km extra charged to passengers only
  },
  presets: [{ id: number, name: string, km: number }]
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
| `renderTrip()` | Re-renders crew chips, preset chips, calls `updatePreview()` |
| `updatePreview()` | Calculates split, renders big number + route viz SVG |
| `renderRouteViz(km)` | Draws the SVG path from home → pickups → club |
| `openChipModal(id)` | Opens per-person config modal |
| `setCameToMe(mode)` | Sets spare/pays state in chipModalTemp |
| `closeChipModal(confirm)` | Writes chipModalTemp to tripOverrides if confirmed |
| `logTrip()` | Saves trip, updates debts, resets session state |
| `openMaps()` | Builds Google Maps URL with waypoints, opens in new tab |
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
- **localStorage key is `carpool_v4`** — if you change the state shape significantly, bump this to `carpool_v5` to avoid hydration errors from old saved data
- **SVG route viz** is built dynamically in `renderRouteViz()` — viewBox is `0 0 460 110`, nodes spaced evenly across the width with `pad=36`
- **No `position:fixed`** anywhere — the app is designed to be saved as a local file and opened in a mobile browser; fixed positioning causes issues in some mobile browsers
- **Single file constraint** — keep everything in one HTML file; don't split into separate CSS/JS files unless the user explicitly asks to set up a proper project

---

## Common tasks
**Add a new field to member profiles** → update `addMember()`, the member object shape, `renderSettings()` members-list HTML, and `renderTrip()` chip modal

**Change the split formula** → edit `updatePreview()` and `logTrip()` (both must use identical logic)

**Add a new chip visual state** → add CSS class, add color vars if needed, update `renderTrip()` chip class logic

**Change the color palette** → update CSS vars in `:root` only — everything references vars

**Add a new screen** → add nav button, add `<div id="s-newscreen" class="screen">`, add case in `showScreen()`
