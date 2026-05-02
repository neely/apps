# Strikemap — Project Handoff
*Status: Design complete, ready for development*

---

## What This Is

A single-file static HTML app for tracking lightning strikes in real time. You point your phone at a lightning flash, tap a button, then tap again when you hear thunder. The app calculates distance and bearing, logs each strike, and plots uncertainty ellipses on a live map showing the storm's progression.

Designed to be hosted on GitHub Pages and shared with a few people. No server, no API keys, no accounts.

---

## Two-Screen App

### Screen 1: RECORD
- Compass display showing live phone heading (locks on flash tap)
- **SAW FLASH** button — locks heading, starts timer
- **HEARD THUNDER** button — stops timer, calculates distance, logs strike
- Live calc display: delay (seconds), distance (miles), bearing (degrees + cardinal)
- Observation log (right panel): each strike with distance, delay, bearing, time-ago, color-coded by age
- Distance trend: bar chart + approaching/receding/steady indicator

### Screen 2: MAP
- Static Leaflet/OSM map, locked (no pan/zoom interaction)
- Centered on user's geolocation, zoom level 12 (~10–12mi wide view)
- All interaction disabled except one RE-CENTER button (insurance)
- Each strike plotted as an **ellipse centered on the computed strike point**
- Bearing lines (faint dashed) from user to each strike center
- Bottom strip: obs count, closest strike, direction, trend, approaching alert

---

## Core Geometry — Important

The ellipses represent measurement uncertainty **around the computed strike point**, not a zone between the user and the strike. The strike is in the center of the ellipse.

**Radial axis (depth, along bearing):**
- Source: ±250ms timing error
- 250ms × 343 m/s = ±86 meters
- This is the *short* axis — timing is pretty accurate

**Tangential axis (width, perpendicular to bearing):**
- Source: ±5° compass wobble
- Width = 2 × sin(5°) × distance
- Grows with distance — this is the *long* axis
- At 1.4mi: ~±215m wide
- At 3.1mi: ~±475m wide

Ellipse is rotated to the bearing angle so its major axis is always perpendicular to the line of sight.

---

## Distance / Time Reference

| Thunder delay | Distance |
|---|---|
| 5 seconds | ~1 mile |
| 10 seconds | ~2 miles |
| 15 seconds | ~3 miles |
| 30 seconds | ~6.3 miles (practical outer limit) |

Strikes beyond ~6 miles (>30s delay): log them, show "DISTANT" notice on map tab, do not plot ellipse. The map at zoom 12 naturally covers ~5–6 mile radius, so this aligns cleanly.

---

## Age / Color Scheme

| Age | Color | Hex |
|---|---|---|
| < 5 min | Yellow | `#f5c842` |
| 5–15 min | Orange | `#f08030` |
| > 15 min | Blue-grey | `#4a7aaa` |

Applied to: log entry left border, ellipse stroke/fill, bearing lines, trend bars.

---

## Tech Stack

| Concern | Solution |
|---|---|
| Map tiles | Leaflet 1.9.4 + OpenStreetMap (CDN, free, no key) |
| Geolocation | `navigator.geolocation.getCurrentPosition()` |
| Compass | `DeviceOrientationEvent` / `webkitCompassHeading` (iOS) |
| Storage | `localStorage` — session only, CLEAR button wipes it |
| Hosting | GitHub Pages (single HTML file) |
| External data | None — Blitzortung overlay explicitly deferred |

---

## Compass / iOS Note

On iOS 13+, `DeviceOrientationEvent.requestPermission()` must be called from a user gesture. The app needs a one-time **"Enable Compass"** button or the compass permission must be requested on the first FLASH tap. Design this carefully — if the user hasn't granted compass access, bearing will be null and should fall back gracefully (log the strike with a "no bearing" flag, skip the map plot or plot on a question-mark ring).

---

## Map Configuration

```javascript
const map = L.map('map', {
  center: [userLat, userLon],
  zoom: 12,
  zoomControl: false,
  dragging: false,
  touchZoom: false,
  doubleClickZoom: false,
  scrollWheelZoom: false,
  boxZoom: false,
  keyboard: false
});
```

One RE-CENTER button in the controls strip restores `setView([userLat, userLon], 12)`.

---

## Ellipse Rendering on Leaflet

Use `L.ellipse` from the leaflet-ellipse plugin (CDN available), or approximate with a rotated `L.polygon` built from parametric ellipse points. The polygon approach has no additional dependency:

```javascript
function ellipsePolygon(centerLat, centerLon, bearingDeg, distMi) {
  const SOUND_MS = 343;
  const TIMING_ERR_S = 0.25;
  const BEARING_ERR_DEG = 5;
  
  const radialM = TIMING_ERR_S * SOUND_MS;           // short axis (meters)
  const tangM   = Math.sin(BEARING_ERR_DEG * Math.PI/180) * distMi * 1609.34; // long axis
  
  // build ellipse points in local coords, rotate to bearing, project to lat/lon
  // return as L.polygon(points, { color, fillColor, ... })
}
```

---

## localStorage Schema

```javascript
// Key: 'sm_strikes'
// Value: JSON array of strike objects
[
  {
    ts: 1711234567890,   // Date.now() at thunder tap
    delay: 2.3,          // seconds flash-to-thunder
    distMi: 1.42,        // computed miles
    bear: 43,            // degrees, null if compass unavailable
    card: "NNE"          // cardinal string, null if no bearing
  }
]
```

---

## Deferred / Out of Scope

- **Blitzortung / external strike data**: Hook exists as `fetchExternalStrikes()` returning `[]`. Add later if desired.
- **Pinch-to-zoom**: Explicitly excluded. Static map is intentional.
- **Google Sheets logging**: Not needed. localStorage covers the storm window.
- **Multi-session history**: Not needed. CLEAR and start fresh each storm.
- **Triangulation**: Single-observer bearing + distance only. True triangulation needs 2+ observers at known locations.

---

## Files in This Package

| File | Description |
|---|---|
| `STRIKEMAP_HANDOFF.md` | This document |
| `strikemap-mockup.html` | Full two-screen static mockup with your James Island screenshot as map background. Shows correct ellipse geometry. Flash/Thunder buttons wired for timing demo. |

---

## Starting Point for Development

1. Start from `strikemap-mockup.html` — all UI, layout, color system, and geometry logic is there
2. Replace the static `<img>` map with Leaflet init
3. Wire geolocation to map center
4. Wire compass permission flow
5. Replace hardcoded mock strikes with localStorage read/write
6. Replace SVG ellipse markup with computed `L.polygon` calls
7. Add the `fetchExternalStrikes()` stub (returns `[]`)
8. Test on actual phone — compass and geolocation only work over HTTPS

---

*Design conversations: Claude.ai chat, March 2026*
