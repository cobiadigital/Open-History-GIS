# Open History GIS

A single-page, client-side historical mapping viewer. No build step, no backend — one `index.html` and a `data/` folder served as static files (currently via Cloudflare Workers at `openhistory.cobiadigital.workers.dev`).

## What it does now

### Globe + OHM base map
The map renders as a 3D globe (MapLibre GL v4 globe projection). The base layer is one of four **OpenHistoricalMap vector tile styles**, each with a distinct visual character:

| Button | Style | Character |
|--------|-------|-----------|
| Historical | `main/main.json` | Standard cartographic — borders, roads, labels |
| Woodblock | `woodblock/woodblock.json` | Woodblock-print aesthetic |
| Rail | `rail/rail.json` | Emphasises railway infrastructure |
| Japanese Scroll | `japanese_scroll/ohm-japanese-scroll-map.json` | Scroll-painting look |

All four styles are fetched from `raw.githubusercontent.com/OpenHistoricalMap/map-styles` because the GitHub Pages URL (`openhistoricalmap.github.io/map-styles`) returns 404. Internal `sprite` and `glyph` URLs inside each style JSON are rewritten to `raw.githubusercontent.com` before being passed to MapLibre.

### Year / date slider
The `maplibre-gl-dates` plugin (`unpkg.com/@openhistoricalmap/maplibre-gl-dates`) patches `Map.prototype.filterByDate()`. Moving the slider (1–1950 CE, default 1850) calls `map.filterByDate(year)`, which filters OHM features by their `start_date`/`end_date` properties — borders, settlements, railways etc. update to match the selected period.

### River layer
Interactive river network overlaid on top of the OHM base. Data sourced from `cobiadigital/night_time_geoguesser` (same JSON format). Six world regions, defaulting to Europe. Features:
- Zoom-aware detail filter (Major / +Secondary / All)
- Click nearest river within 75 miles → name shown in info bar
- Optional city layer (population-scaled circles, lazy-loaded per region)
- All river/city data persists across OHM style switches

### Dataset layers
An extensible `DATASETS` registry in `index.html` allows toggling named historical datasets on/off as map overlays. Each dataset loads its JSON lazily on first activation and re-renders after OHM style switches.

**Current datasets:**

| Button | File | Description |
|--------|------|-------------|
| M. Antoinette 1770 | `data/antoinette_1770.json` | Marie Antoinette's 26-day journey Vienna → Versailles, Apr–May 1770. 41 stops. Habsburg segment (red dashed), French segment (blue dashed). Click any stop for date, location, and historical notes. |

---

## Architecture

```
index.html          — entire app; no framework, no build
data/
  antoinette_1770.json   — 41-stop journey array
```

### Key external dependencies (CDN)
- `maplibre-gl@4` — globe rendering, vector tiles, GeoJSON layers
- `@openhistoricalmap/maplibre-gl-dates` — date-based feature filtering

### Key external data sources
- OHM vector tiles: `vtiles.openhistoricalmap.org/maps/osm/{z}/{x}/{y}.pbf`
- OHM style JSON + sprites/glyphs: `raw.githubusercontent.com/OpenHistoricalMap/map-styles/main/`
- Rivers/cities: `raw.githubusercontent.com/cobiadigital/night_time_geoguesser/main/`
- Landcover/hillshade raster: `static-tiles-lclu.s3.us-west-1.amazonaws.com`

### Adding a new dataset

1. Add a JSON file to `data/`.
2. Add one entry to the `DATASETS` object in `index.html`:
   - `file` — filename inside `data/`
   - `buildSources(data)` — returns `{ sourceId: geoJsonSpec, ... }`
   - `buildLayers()` — returns array of MapLibre layer specs
   - `clickLayers` — layer ids that respond to click
   - `popup(props)` — returns HTML string for the popup
3. Add one `<button class="btn dataset-btn" data-dataset="your_key">` in `#dataset-btns`.

No other plumbing needed — toggle, lazy-load, style-switch restore, and cursor management are all generic.

---

## What we want to build

### More datasets
The core use case: rich historical journeys, events, and geographic data overlaid on the historically-accurate OHM base map. Candidates:

- **Napoleon's campaigns** — routes of major campaigns with battle sites
- **Roman roads network** — via `Itinerarium Antonini` reconstructions
- **Historical trade routes** — Silk Road, Hanseatic League, Amber Road
- **Treaty boundaries over time** — borders at specific peace treaties (Westphalia 1648, Vienna 1815, Versailles 1919)
- **Migration events** — Viking expansion, Mongol expansion, Crusades routes
- **Other biographical journeys** — Grand Tour routes, Darwin's Beagle voyage, etc.

### Dataset format improvements
- Standardise on a richer JSON schema that includes metadata (title, description, date range, source citations)
- Support line datasets (routes), point datasets (sites/events), and polygon datasets (territories/empires)
- A small dataset manifest file (`data/index.json`) so the app can discover and list available datasets without hardcoding each one

### UI / UX
- **Dataset detail panel** — clicking a dataset button could expand a description and legend rather than just toggling
- **Time-synced datasets** — link the year slider to dataset visibility (e.g. only show stops that occurred in or before the selected year)
- **Mobile layout** — the current header wraps acceptably but hasn't been tuned for small screens
- **Permalink / URL state** — encode active region, year, style, and datasets in the URL hash so views are shareable

### OHM integration
- The `maplibre-gl-dates` plugin is wired up but the OHM vector tile coverage is sparse for many periods and regions; evaluate what years/areas have enough data to be useful
- Consider showing a "data density" indicator next to the year slider

### Infrastructure
- Move large or frequently-updated datasets out of the repo into a hosted API or R2 bucket
- Consider a thin Cloudflare Worker API for dataset discovery and filtering rather than shipping all data to the client
