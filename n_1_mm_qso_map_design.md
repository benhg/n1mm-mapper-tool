# N1MM QSO Map Extension — DESIGN.md

## 1) Summary

A sidecar extension for N1MM+ that listens on the local UDP broadcast network, parses QSO events from all operator PCs, and renders a **live great-circle map** from the station QTH to each DX location. Designed for **Field Day displays** and **contest debriefs**, it requires no workflow changes for operators, works offline, and provides exports (PNG, SVG, KML, GeoJSON).

---

## 2) Goals / Non-Goals

### Goals
- Passive ingestion of N1MM+ network UDP broadcasts.
- Zero-friction setup: only configure station QTH and UDP port.
- Live great-circle map visualization with per-band coloring.
- Support multi-PC, multi-op Field Day environments.
- Public-display friendly “Field Day Mode.”
- Offline-first geolocation (Maidenhead, DXCC prefix lookup).
- Session persistence and export (PNG/SVG, KML, GeoJSON).

### Non-Goals
- No direct modification of N1MM+ logs.
- Not a compiled N1MM plugin (no DLL injection).
- No reliance on internet lookups (QRZ, Club Log) for baseline functionality.
- Not a scorekeeping or cluster/skimmer tool.

---

## 3) User Stories

- **Operator:** “When I log a contact, I want to see it appear on the map instantly.”
- **Field Day coach:** “Show where we’ve worked and which bands are open.”
- **Public demo:** “Display a clean, colorful map on a big screen to engage visitors.”
- **After-action review:** “Export the map to KML or PNG for post-event analysis.”

---

## 4) Integration with N1MM+

- **Method:** UDP broadcast listener (no plugin).
- **Setup in N1MM+:**
  - `Config > Configure Ports, Mode Control, Winkey, etc. > Broadcast Data`
  - Enable **UDP Broadcasts**, at least **Contact Info**.
  - Set broadcast address = display PC IP (or subnet broadcast).
  - Set port (default: 12060).
- **Parsed events:** QSO add/edit/delete, optional Status/Radio/Score.
- **Multi-PC:** All stations send to same port → extension merges & deduplicates.

---

## 5) System Architecture

```
+-------------------------+      UDP (LAN)       +-------------------+
| N1MM+ PCs (operators)   | ------------------>  | UDP Listener      |
+-------------------------+                      | + Parser          |
                                                 +---------+---------+
                                                           |
                                                           v
                                                 +-------------------+
                                                 | Session Store     |
                                                 | (in-mem + SQLite) |
                                                 +---------+---------+
                                                           |
                             WS/SSE                        | API Queries
         +---------------------------+          +----------+----------+
         | Web Frontend (Map Display)| <------> | HTTP API (FastAPI) |
         | Leaflet / deck.gl UI      |          +---------------------+
         +---------------------------+
```

---

## 6) Data Handling

### Inputs
- UDP QSO events: timestamp, my_call/grid, dx_call/grid, band, mode.
- Auxiliary data: `cty.dat` (DXCC entities), optional overrides (`overrides.yml`).

### Geolocation strategy
1. If DX grid present → decode Maidenhead to lat/lon.
2. Else → callsign prefix → DXCC entity centroid.
3. Apply overrides (e.g., `/MM`, `/AM`, rare expeditions).

### Great-circle path generation
- WGS84 sphere model.
- Sampled points along arc, antimeridian splitting.
- Optional screen-space smoothing.

---

## 7) Visualization

- **Frontend:** Browser app (Leaflet for simplicity, deck.gl for scaling).
- **Projection:** Web Mercator (globe later).
- **Layers:**
  - Great-circle arcs: colored by band.
  - Points: per-QSO faint; per-entity bold.
  - Hover: callsign, band, time, distance.
- **Filters:** by band, mode, time window, operator.
- **Replay mode:** step through contest timeline.

### Field Day Mode
- Fullscreen dark map with high-contrast arcs.
- Band-colored legend, auto-cycling or fixed.
- Minimal controls (for public display).
- Runs continuously on TV/projector with no operator input.

---

## 8) Session Store & Schema

- In-memory cache for fast live updates.
- SQLite persistence for review/export.

Schema:
- `qso(id, ts, my_call, my_grid, dx_call, dx_grid, band, mode, freq_hz, source_station, action)`
- `point_cache(key, lat, lon, source)`
- `session_meta(key, value)`

---

## 9) Configuration

```yaml
listener:
  udp_bind: "0.0.0.0"
  ports: [12060]

station:
  my_call: "KN6UBF"
  my_grid: "CM87ux"

geodata:
  cty_dat_path: "./data/cty.dat"
  overrides: "./data/overrides.yml"

render:
  band_colors:
    "160m": "#7b3294"
    "80m":  "#c2a5cf"
    "40m":  "#a6dba0"
    "20m":  "#008837"
    "15m":  "#fdae61"
    "10m":  "#d73027"
  max_arcs: 5000

storage:
  sqlite_path: "./state/session.db"
  persist: true

fieldday:
  mode: true
  fullscreen: true
```

---

## 10) API & Events

- REST endpoints:
  - `GET /api/summary`
  - `GET /api/paths` (GeoJSON)
  - `POST /api/export/kml`
- WebSocket/SSE stream:
  - Events: `{ type: qso_add|qso_edit|qso_delete, payload: {...} }`

---

## 11) Error Handling

- Deduplicate on `(dx_call, ts±δ, band, station)`.
- Invalid grids → fallback to DXCC centroid.
- `/MM` or `/AM` → skip by default, configurable.
- Out-of-order timestamps → accepted; sort by arrival.

---

## 12) Security

- LAN-only listener.
- Configurable IP allowlist.
- Minimal retention (rotate DB if desired).

---

## 13) Performance

- Cache grid and prefix lookups.
- Throttle redraws (≤20 FPS).
- Batch WS events.
- WebGL for >5k arcs.

---

## 14) Testing

- Unit tests: Maidenhead, DXCC prefix matching, great-circle math.
- Golden files: sample UDP payload → expected QSO struct.
- Integration: UDP replay to validate live updates.
- Visual regression: snapshot maps for baseline diff.

---

## 15) Milestones

- **MVP:** UDP listener, geolocation, live Leaflet map, Field Day fullscreen mode.
- **v0.2:** Persistence, filters, replay mode, exports.
- **v0.3:** WebGL performance, overrides, public-ready packaging.
- **v1.0:** Installer, robust test suite, docs.

---

## 16) Open Questions

- Confirm UDP broadcast schema for current N1MM+.
- Decide default frontend stack: Leaflet (easier) vs deck.gl (scalable).
- Prioritize exports: PNG vs SVG vs KML.
- Should we color by **band only**, or allow **per-operator**?

---

## Appendix A — Field Day Setup Checklist

- Enable UDP broadcasts in N1MM+ on all PCs.
- Configure same port & broadcast destination.
- Open firewall for UDP port (e.g., 12060).
- Run map extension on display PC.
- Open `http://localhost:8080` in browser.
- Fullscreen browser on projector/TV.
- Verify by logging a dummy QSO.

---

## Appendix B — Great Circle Math

- Use spherical interpolation for arcs.
- Sample: `ceil(angular_distance/10) * 5` points.
- Split at ±180° longitude crossings.
- Plot polyline segments on map.

---

