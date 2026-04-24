# BC Ferries Tracker — Project Memory

## Project Overview
A single-file static HTML/JS web app that displays BC Ferries sailing schedules and capacity data. No server-side processing. Hosted on GitHub Pages.

## Repository
- **GitHub:** https://github.com/dfslioch/bcferries
- **Live URL:** https://dfslioch.github.io/bcferries/
- **Local dev folder:** C:\Dev\BCFerries
- **Main file:** `index.html` (everything is in one file — HTML, CSS, JS)
- **Git remote:** `https://dfslioch@github.com/dfslioch/bcferries.git` (username embedded to avoid multi-account prompt)

## Owner / Context
- David, based in Sechelt, BC (Sunshine Coast)
- Primary route of interest: **Horseshoe Bay ↔ Langdale** (HSB↔LNG)
- Existing PHP reference page at: https://www.sliochsoftware.ca/bcf_hsb_lng.php

---

## Data Sources

### BC Ferries API (independent, scrapes BC Ferries)
- **Root:** `https://www.bcferriesapi.ca/v2/`
- **Capacity only:** `https://www.bcferriesapi.ca/v2/capacity/`
- **Non-capacity:** `https://www.bcferriesapi.ca/v2/noncapacity/`
- App uses `/v2/capacity/` endpoint
- Response key is `routes` (not `capacityRoutes` — that was a v1 key)
- Each route object: `{ routeCode, fromTerminalCode, toTerminalCode, sailingDuration, sailings[] }`
- Each sailing: `{ time, arrivalTime, sailingStatus, fill, carFill, oversizeFill, vesselName, vesselStatus }`
- `sailingStatus` values: `"past"`, `"current"`, `"future"`
- Fill data (0–100%) only meaningful on **future** sailings — past/current sailings always return 0
- API returns today's data only, no date field
- Times are in 12-hour format with am/pm, e.g. `"7:30 am"`, `"1:04 pm"` — use `parseTimeToMins()` to convert

### Non-capacity endpoint notes (investigated 2026-04-24)
- Returns 116 routes — mostly small inter-island connectors with unknown terminal codes
- No `sailingStatus` field — past/current/future must be inferred from clock
- `vesselName` is always blank in this endpoint
- HSBBOW/BOWHSB have a data bug — returns ~7 days of sailings concatenated (~103/109 entries)
- All routes already in the capacity picker are also in noncapacity — no new major routes to add
- Decision: **not adding noncapacity routes for now**

### Terminal Camera Images (BC Ferries)
- URL pattern: `https://apigateway.bcferries.com/api/currentconditions/1.0/images/terminals/cam1_HSB.jpg`
- Cameras: `cam1_XXX.jpg` and `cam2_XXX.jpg` per terminal code
- Confirmed working for: HSB, LNG (all 4 URLs tested)
- Images embedded with cache-busting `?t=timestamp` on refresh
- Camera cards auto-hide via `onerror` if camera doesn't exist for a terminal
- Cameras are inside each route panel (anchored to bottom), showing departure terminal only
- Clicking a camera opens a full-screen lightbox overlay; Escape or tap to close

### Vessel Tracker (BC Ferries iframe)
- URL pattern: `https://apigateway.bcferries.com/api/currentconditions/1.0/images/vessels/route4.html`
- Native content size: 500px wide × 650px tall (500×500 image + info text below)
- App scales iframe to fit container using CSS transform + JS resize listener
- Route numbers (confirmed: route 4 = HSB↔LNG; others are best-guess estimates):
  - Route 1: TSA ↔ SWB (Tsawwassen ↔ Swartz Bay) — **NEEDS VERIFICATION**
  - Route 2: TSA ↔ DUK (Tsawwassen ↔ Duke Point) — **NEEDS VERIFICATION**
  - Route 3: TSA ↔ SGI (Tsawwassen ↔ Southern Gulf Islands) — **NEEDS VERIFICATION**
  - Route 4: HSB ↔ LNG (Horseshoe Bay ↔ Langdale) — **CONFIRMED**
  - Route 5: SWB ↔ FUL (Swartz Bay ↔ Fulford Harbour) — **NEEDS VERIFICATION**
  - Route 6: HSB ↔ NAN (Horseshoe Bay ↔ Departure Bay) — **NEEDS VERIFICATION**
  - Route 7: HSB ↔ BOW (Horseshoe Bay ↔ Bowen Island) — **NEEDS VERIFICATION**
  - Route 9: SWB ↔ SGI (Swartz Bay ↔ Southern Gulf Islands) — **NEEDS VERIFICATION**

---

## Terminal Codes
```
TSA = Tsawwassen
SWB = Swartz Bay
SGI = Southern Gulf Islands
DUK = Duke Point (Nanaimo)
FUL = Fulford Harbour (Salt Spring Island)
HSB = Horseshoe Bay
NAN = Departure Bay (Nanaimo)
LNG = Langdale (Sunshine Coast)
BOW = Bowen Island
```

## Route Codes (capacity routes)
Format: `FROMTO` e.g. `HSBLNG` = Horseshoe Bay → Langdale
- TSASWB / SWBTSA
- TSASGI / SWBSGI
- TSADUK / DUKTSA
- HSBLNG / LNGHSB
- HSBNAN / NANHSB
- HSBBOW / BOWHSB
- SWBFUL / FULSWB

---

## App Features (implemented)

### Core
- Fetches `/v2/capacity/` on load
- Displays both directions of selected route side by side (desktop) or tabbed (mobile)
- Past sailings shown expanded by default with toggle to hide
- Sailings ordered chronologically: past → on-route → upcoming

### Sailing Row Types
- **Past**: compact single-line — `time → arrival  vessel  [status badge if non-standard]`
- **On-route (current)**: compact with amber background — `time → (ETA)  vessel  [On route / Delayed? / incident status]`
- **Next**: green highlighted, full capacity display
- **Future**: full capacity display

### On-Route Intelligence
- `parseTimeToMins(str)` handles 12-hour am/pm format correctly
- **Vessel cross-check**: if same vessel has departed on the return leg *after* this sailing's departure time → reclassify as past (data-driven, most reliable signal)
- **Overdue detection**: if scheduled arrival has passed but vessel cross-check hasn't confirmed → show `Delayed?` badge
- **Incident passthrough**: if `vesselStatus` or `sailingStatus` contains a non-standard value (anything other than "past"/"current"/"future") → show it as the badge instead of "Delayed?" or "On route"
- ETA shown in parentheses for on-route sailings: `12:24 pm → (1:04 pm)`

### Capacity Display (future/next sailings only)
- **Available pill**: `100 - fill%`, colour-coded green→amber→orange→red, shown as full-width bar with pill badge on right
- **Fill / Car / Oversize**: 3-column bar row below the Available bar
- Past and on-route sailings show no capacity bars (API returns 0 after departure)

### Refresh
- Manual refresh button in header (with spinning icon while loading)
- Auto-refresh options: Off / 2 min / 5 min
- "Last updated" timestamp shown in refresh bar
- Auto-refresh preference saved to cookie `bcf_interval`

### Route Selection
- Default route: HSB ↔ LNG
- Route picker modal (☰ Routes button)
- Selected route saved to cookie `bcf_route` (value: `FROM_TO` e.g. `HSB_LNG`)
- Cookies expire after 365 days

### Cameras
- Two cameras per departure terminal, side-by-side at full panel width, anchored to panel bottom
- Click any camera → full-screen lightbox overlay (fresh cache-busted image)
- Tap anywhere or press Escape to close overlay
- Cards auto-hide if camera URL returns an error

### Vessel Tracker
- Shown below route panels
- iframe scaled to container width via `scaleTracker()` using CSS transform
- Re-scales on window resize

### Layout
- Responsive: desktop = two-column side-by-side, mobile = tabbed
- Tiled background: pagewallpaper25.jpg (embedded as base64 data URI)
- Colour palette: teal/seafoam (matching sliochsoftware.ca style)

### Versioning / Cache
- No-cache meta tags in `<head>` to prevent browser caching
- Footer shows version in format `0.yy.M.d.HHmm` — update with every commit
- Current version: `0.26.4.24.1645`
- Hard refresh: Ctrl+Shift+R bypasses all caches including GitHub Pages CDN (~10 min TTL)

### Footer
- Acknowledges data source: bcferriesapi.ca
- Shows version string

---

## Known Issues / TODO
- [ ] Verify vessel tracker route numbers (all except route 4)
- [ ] Webcam URLs unverified for terminals other than HSB/LNG — auto-hides gracefully
- [ ] Non-capacity inter-island routes not added (terminal codes undocumented, Bowen Island API bug)
- [ ] The `sailingDuration` field is often blank in the API — duration inferred from time/arrivalTime

---

## Development Workflow
```bash
# Edit index.html locally
# Then deploy:
git add index.html
git commit -m "description of change"
git push
# GitHub Pages rebuilds in ~60 seconds
# Remember to update the version string in the footer
```

## File Structure
```
C:\Dev\BCFerries\
├── index.html      # Everything — HTML, CSS, JS, background image (base64)
└── CLAUDE.md       # This file
```

---

## Design Notes
- Single HTML file — no build tools, no dependencies, no CDN links
- All CSS is inline in `<style>` block
- Background texture image is embedded as base64 to keep it self-contained
- Cookie-based preferences only (no localStorage — more compatible)
- API is fetched client-side; CORS is supported by bcferriesapi.ca for browser requests
- `onerror` on camera `<img>` tags gracefully hides missing cameras
- Cache-busting on camera URLs: `?t=${Date.now()}`
- `.sailing-compact` class controls compact row layout (shared by past and on-route)
- `past` and `current` classes control background colour independently of layout
