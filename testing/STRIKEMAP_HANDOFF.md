# Strikemap — Project Handoff
*Status: Core functionality complete — UI restructure + polish next*

---

## What This Is

A single-file static HTML app for tracking lightning strikes in real time. Point your phone at a lightning flash, tap a button, then tap again when you hear thunder. The app calculates distance and bearing, logs each strike, and plots uncertainty regions on a live map showing the storm's progression.

Designed to be hosted on GitHub Pages and shared with a few people. No server, no API keys, no accounts.

---

## Three-Screen App (after Session 4)

### Screen 1: RECORD
- Compass display showing live phone heading — rotates in real time, locks on tap after FLASH
- **COUCH MODE toggle** — disables bearing entirely; no compass shown, no bearing recorded; strike saves with `bear: null`
- **SAW FLASH** button — arms compass lock (unless couch mode), starts timer, requests compass permission on iOS
- **HEARD THUNDER** button — stops timer, calculates distance, saves strike to localStorage
- Live calc display: delay (seconds), distance (miles), bearing (degrees + cardinal, or "COUCH MODE" if toggled)
- Full-width single column — no right panel split (phone layout)

### Screen 2: TRENDS
- Distance trend bar chart (up to 8 most recent strikes)
- Approaching / receding / steady indicator
- Full strike log with per-entry delete (✕ button, no confirm needed for single strike)
- **CLEAR ALL** button — requires confirm dialog before wiping localStorage
- Timestamps refresh on a 30s interval regardless of active tab

### Screen 3: MAP
- Live Leaflet/CartoDB dark tiles, locked (no pan/zoom/drag)
- Centered on user's geolocation at zoom 12 (~10–12mi wide view)
- Strikes with bearing → pizza-crust uncertainty polygon + center dot + bearing line
- Strikes without bearing (couch mode or no compass) → **full circle ring** at `distMi` radius centered on user, same age color, no bearing line
- Age filter buttons: ALL / 10 MIN / 30 MIN + RE-CENTER button
- DISTANT notice banner when any strikes exceed 6.2mi (logged but not plotted)
- Bottom strip: obs count, closest strike, direction, trend, approaching alert
- Legend overlay — present, positioned bottom-right, out of the way; consider collapsible tap if it occludes strikes
- No CLEAR button on map — all data management lives on TRENDS tab

---

## Core Geometry

### Pizza Crust (bearing known)

Two error sources, both referenced from the observer's position:

**Bearing error (±5°):**
- Source: compass wobble
- Wedge of two rays at `bearing ± 5°` from observer
- Width grows linearly with distance: ~0.24mi at 1.4mi, ~0.52mi at 3mi

**Timing error (±500ms):**
- Source: human tap reaction on FLASH and THUNDER
- `±500ms × 343 m/s = ±171.5m ≈ ±0.107mi`
- Inner arc at `dist − 0.107mi`, outer arc at `dist + 0.107mi`
- Depth is constant regardless of distance

Shape: outer arc (left→right) + inner arc (right→left) = closed `L.polygon` of ~26 points via `strikeLatLon()`. Collapses to a pure wedge if `dist < 0.107mi` (strike under ~560ft). Center dot always plotted.

```javascript
function makePizzaCrust(userLat, userLon, bearingDeg, distMi, color) {
  const DELTA_D_MI = 0.5 * 343 * 0.000621371;  // ~0.107 mi
  const DELTA_B    = 5;                          // ±5°
  const ARC_STEPS  = 12;

  const innerDist = Math.max(0, distMi - DELTA_D_MI);
  const outerDist = distMi + DELTA_D_MI;
  const bearLeft  = bearingDeg - DELTA_B;
  const bearRight = bearingDeg + DELTA_B;

  const points = [];
  for (let i = 0; i <= ARC_STEPS; i++) {
    const b = bearLeft + (bearRight - bearLeft) * (i / ARC_STEPS);
    points.push(strikeLatLon(userLat, userLon, b, outerDist));
  }
  for (let i = ARC_STEPS; i >= 0; i--) {
    const b = bearLeft + (bearRight - bearLeft) * (i / ARC_STEPS);
    points.push(strikeLatLon(userLat, userLon, b, innerDist));
  }
  return L.polygon(points, { color, weight:1.5, fillColor:color, fillOpacity:0.20, interactive:false });
}
```

### Bullseye Ring (bearing null — couch mode or no compass)

When `bear === null`, plot a full circle at `distMi` centered on the user. Represents "definitely this far away, direction unknown." Same age color, reduced opacity, dashed stroke. No bearing line. Center dot still plotted.

```javascript
L.circle([userLat, userLon], {
  radius: distMi * 1609.34,
  color, weight:1.5, fillColor:color, fillOpacity:0.10,
  dashArray:'4 4', interactive:false
})
```

---

## Distance / Time Reference

| Thunder delay | Distance |
|---|---|
| 5 seconds | ~1 mile |
| 10 seconds | ~2 miles |
| 15 seconds | ~3 miles |
| 30 seconds | ~6.3 miles (practical outer limit) |

Strikes beyond ~6.2 miles: logged normally, not plotted, DISTANT banner shown on map tab.

---

## Age / Color Scheme

| Age | Color | Hex | Applied to |
|---|---|---|---|
| < 5 min | Yellow | `#f5c842` | log border, pizza crust / ring, bearing line, trend bar |
| 5–15 min | Orange | `#f08030` | same |
| > 15 min | Blue-grey | `#4a7aaa` | same |

---

## Tech Stack

| Concern | Solution |
|---|---|
| Map tiles | Leaflet 1.9.4 + CartoDB dark tiles (CDN, free, no key) |
| Geolocation | `navigator.geolocation.getCurrentPosition()` |
| Compass | `DeviceOrientationEvent` / `webkitCompassHeading` (iOS) |
| Storage | `localStorage` key `sm_strikes` — session only, CLEAR wipes it |
| Hosting | GitHub Pages (single HTML file) |
| External data | `fetchExternalStrikes()` stub returns `[]` — Blitzortung hook deferred |

---

## Compass Flow (iOS)

On iOS 13+, `DeviceOrientationEvent.requestPermission()` must be called from a user gesture. Permission is requested on the first SAW FLASH tap — no separate button needed. If denied or unavailable, bearing is stored as `null` (same behavior as couch mode). Log entry shows `--- NO HDG`.

Heading is smoothed over an 8-sample circular buffer to reduce jitter. Heading locks when the user taps the compass ring while armed; tap again to unlock.

**Couch mode** bypasses the compass entirely — no permission request, no lock UI, bearing always `null`.

---

## localStorage Schema

```javascript
// Key: 'sm_strikes'
// Value: JSON array, newest first
[
  {
    ts:     1711234567890,  // Date.now() at THUNDER tap
    delay:  2.3,            // seconds flash-to-thunder (1 decimal)
    distMi: 1.42,           // computed miles (2 decimal)
    bear:   43,             // integer degrees, null if couch mode or no compass
    card:   "NNE"           // cardinal string, null if no bearing
  }
]
```

Single-strike delete: filter array by `ts`, save, re-render log + map.

---

## Files

| File | Description |
|---|---|
| `STRIKEMAP_HANDOFF.md` | This document |
| `strikemap-mockup.html` | Static design mockup — reference only, do not edit |
| `strikemap.html` | **Live file — source of truth for all implementation** |

**Always read `strikemap.html` first. The handoff doc describes intent; the live file is what's actually built.**

---

## What's Done (Sessions 1–3)

- [x] Full UI — layout, typography, dark color system
- [x] Compass: live `DeviceOrientationEvent`, 8-sample smoothing, tap-to-lock, iOS permission flow
- [x] SAW FLASH / HEARD THUNDER timing + calculation
- [x] localStorage read/write/clear
- [x] Strike log with age classes and relative timestamps
- [x] Distance trend bar chart + approaching/receding/steady
- [x] Leaflet map: CartoDB dark tiles, geolocation centering, locked interaction
- [x] Pizza-crust uncertainty polygons (annular sector, correct polar geometry)
- [x] Bearing lines from user to each strike center
- [x] Center dot + label on most recent strike
- [x] Age filter buttons (ALL / 10 MIN / 30 MIN) scoped with `data-age`
- [x] DISTANT notice banner
- [x] APPROACHING alert — JS-controlled, starts hidden
- [x] `fetchExternalStrikes()` stub
- [x] OBS badge in header

---

## SESSION 4 — START HERE

Read `strikemap.html` first. It is the source of truth. Do all of the following in one pass.

### 1. Three-tab nav (RECORD / TRENDS / MAP)
- Add TRENDS button between RECORD and MAP in the header nav
- Add `#screen-trends` section
- Move trend bar chart + strike log into TRENDS screen
- RECORD becomes single full-width column: compass + couch toggle + buttons + calc box only
- Wire `showScreen('trends')` to call `renderLog()` and `renderTrend()` on activation

### 2. Couch mode
- Toggle button on RECORD screen, between compass and SAW FLASH
- When active: hide compass ring, show "COUCH MODE — NO BEARING" status label, set `couchMode = true`
- SAW FLASH in couch mode: skip compass permission, skip bearing lock, `bear` saves as `null`
- Calc box bearing row shows "COUCH MODE" instead of degrees
- Map: `bear === null` strikes render as `L.circle` ring at `distMi * 1609.34` meters radius — dashed, age color, `fillOpacity: 0.10`, no bearing line, center dot still plotted

### 3. UI initialization bugs
- **HEARD THUNDER button** — remove `armed` class from HTML; starts inert, JS adds it on FLASH tap
- **Calc box bearing** — add `id="c-bearing"` directly in HTML, remove the DOM-query patch in DOMContentLoaded
- **Trend bars** — initialize empty on load; call `renderTrend([])` or equivalent in DOMContentLoaded
- **Map strip** — set all stat values to `—` in HTML; `updateMapStrip()` handles real values
- **Timestamp interval** — remove the screen check; run the 30s refresh regardless of active tab
- **Strike label clipping** — use bearing quadrant to set `iconAnchor` so label stays on-screen

### 4. CLEAR / delete UX
- CLEAR ALL on TRENDS tab only — `confirm()` dialog, then wipe localStorage, re-render log + map
- Per-entry ✕ on each log row — no confirm, delete by `ts`, re-render
- Remove CLEAR button from RECORD screen
- No CLEAR on MAP tab

### 5. Map legend
- Check legend visibility on a 390px wide screen (common iPhone width)
- If it occludes strike zones, make it collapsible: tap to toggle, default collapsed

---

## After Session 4

- [ ] Field test on iOS — compass permission, heading accuracy, couch mode, tap latency
- [ ] Field test on Android — `deviceorientation` varies; test Chrome Android + Firefox
- [ ] Deploy to GitHub Pages (`strikemap.html` → `index.html`, enable Pages, verify HTTPS)
- [ ] Geolocation + DeviceOrientation require HTTPS — do not test on `file://`

## Deferred / Out of Scope

- **Blitzortung overlay** — `fetchExternalStrikes()` returns `[]`; wire later if desired
- **Pinch-to-zoom** — explicitly excluded, static map is intentional
- **Multi-session history** — not needed, CLEAR and start fresh each storm
- **Triangulation** — single observer only; needs 2+ observers at known positions

---

*Development sessions: Claude.ai, March–May 2026*
