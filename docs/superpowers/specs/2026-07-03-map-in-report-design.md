# Map in Word Report — Design

Date: 2026-07-03
Branch: `map-in-report`
Scope: `index.html` only. Builds on the professional-report feature (cover, TOC, sections) already on main.

## Purpose

Embed maps of the plotted photo locations in the exported Word report. Hard requirement: the map must render completely and accurately on the first export — no half-loaded tiles, no silently wrong output. When the basemap cannot be fetched, the report still gets a correct (schematic) map rather than a broken one.

## 1. Map builder

New async function in `index.html`:

`buildMapImageDataUrl(points, style, opts)` → PNG data URL

- `points`: array of `{lat, lon, label}` (label = number rendered in the marker).
- `style`: `"street"` (OSM `https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`) or `"satellite"` (Esri World Imagery `https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}`). Esri is the same vendor as the app's existing ArcGIS address lookup.
- Target output ~1000×700 px.

Algorithm:
1. Bounding box of points + padding (min span ~120 m so a single point still gets context).
2. Highest zoom whose tile grid for the bbox fits the target size; hard caps: zoom ≤ 19, tile count ≤ 30 (step zoom down until both hold).
3. Fetch every tile as `Image` with `crossOrigin="anonymous"`, 8-second timeout, one retry per tile.
4. **All-or-nothing:** compose only after every tile resolves; any tile failing both attempts → throw. No partial maps.
5. Compose tiles on an offscreen canvas, crop to bbox, then draw:
   - numbered circle markers (white number on accent-color disc, small point stem),
   - scale bar (metric + feet, bottom-left),
   - attribution line bottom-right ("© OpenStreetMap contributors" or "Esri, Maxar, Earthstar Geographics"), required by both tile policies.
6. Return `canvas.toDataURL("image/png")`.

## 2. Schematic fallback

`buildSchematicMapDataUrl(points, opts)` — same signature family, cannot fail, no network:
- White canvas, light grid, north arrow (top-right), scale bar, the same numbered markers positioned by local equirectangular projection of the GPS coordinates.
- Note text on canvas: "Basemap unavailable — schematic plot".
- Used automatically when `buildMapImageDataUrl` throws.

## 3. Report placement

- **Overview map (satellite):** its own page immediately after the Contents page. One numbered pin per location section; numbers match the TOC/section numbers. Caption: "Site Overview Map". Only GPS-bearing locations are pinned; if no location has GPS, the overview page is skipped entirely.
- **Per-location map (street):** at the top of each location section, before the photos. One numbered dot per annotated GPS-bearing photo in that section; the number is that photo's global Figure number. Legend line under the caption: "Marker numbers correspond to figure numbers." Caption: "Location Map — <location name>".
  - Photos without GPS are omitted from the map.
  - A location whose annotated photos have no GPS gets no map (skipped silently; nothing fabricated).

Figure-number timing: photo figure numbers must be assigned before the per-location map is built. The export loop computes figure numbers for a section's photos first, builds that section's map, then appends map + photos to the document children.

## 4. Export flow and error handling

- Map building happens inside `buildWordDocumentBlob` (so ZIP export, which embeds the docx, gets maps too).
- Progress status: "Building site maps…" via existing `onProgress`.
- Tile failure → schematic fallback, plus status note after export: "Basemap unavailable — schematic maps used."
- Fallback failure is not possible by construction (no I/O); any unexpected exception in map building must not abort the report — catch, skip that map, continue.
- No new UI controls. Styles are fixed: satellite for overview, street for per-location.

## 5. Testing

- Headless (preview harness): generate a small JPEG with real GPS EXIF via a scratchpad script, drop it, export Word — assert status reaches "Word document download ready" and no console errors. Repeat with tile hosts blocked (override `Image.prototype.src` or point tile URLs at an unreachable host in-page) — assert export still completes (schematic path).
- Manual (user): export a real survey, eyeball marker positions against the known site on both maps; confirm attribution and scale bar visible.

## Out of scope

- Commercial static-map APIs, API keys.
- User-selectable map styles, zoom controls, map in the ZIP as a standalone image.
- Offline tile caching.
