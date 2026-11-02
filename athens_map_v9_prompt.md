# Prompt: build the Athens public transport map (v9) as a single-page vector PDF

You are working in a Linux sandbox with Python 3 (geopandas, pyogrio, shapely 2, pyproj, networkx, numpy, matplotlib) and internet access to www.oasa.gr. Build a single, very large, vector PDF diagram of all Athens bus, trolleybus, tram, metro and suburban rail routes, in the spirit of the Budapest BKK network map (schematic-looking, parallel colour-coded lines, route-number pills, stop names). Work step by step and verify each step. Do not bypass blocked domains; if oasa.gr is blocked, tell me and wait.

## 1. Download and parse the data

Base URL: `https://www.oasa.gr/wp-content/uploads/IndexAllNet/`. It is a qgis2web export. Download `layers/layers.js` and `layers/_1.js` … `layers/_8.js` (plus any other file needed). Each `_N.js` is `var json__N = {...GeoJSON...};` in CRS84 (lon/lat). Strip the wrapper and parse the JSON. Layer meaning (check against `layers.js` and the attributes):

| Layer | Content |
|---|---|
| `_1` | tram lines |
| `_2` | metro lines |
| `_3` | bus + trolleybus network (one feature per route/direction, route number and mode in the attributes) |
| `_4` | suburban rail lines |
| `_5` | suburban rail stations |
| `_6` | metro stations |
| `_7` | tram stops |
| `_8` | bus and trolleybus stops |

Inspect the attributes of every layer first and print them; do not assume field names.

## 2. Exports (do these as well, they are separate deliverables)

* Per-route geometry: one GPKG and one SHP file per route, for bus, trolleybus, tram (split into T6 and T7, see section 4), metro and suburban rail. Zip them.
* All stops/stations (bus, trolleybus, tram, metro, suburban rail): names, IDs, coordinates (lon/lat) and every other attribute field, as GPKG, SHP and CSV. Zip them.

## 3. Build the network

1. Project everything from EPSG:4326 to EPSG:2100 (GGRS87/EGSA87, metres).
2. Node the bus/trolleybus/tram lines with `unary_union` on a 1 m precision grid (`shapely.set_precision`).
3. For every noded edge compute its route signature: the set of routes (per mode) that run along it, using a segment-level STRtree with a 4 m tolerance.
4. Group adjacent edges with the same signature and `line_merge` them into oriented polylines ("units").
5. Draw one line per mode per street segment, even if many routes share it. Consecutive route numbers of the same mode on a segment go into one pill label (e.g. `12 40 224`), sorted numerically.
6. Where bus, trolleybus and tram share a street, draw them parallel using per-mode offsets: for k modes present, slot offsets are `((k-1)/2 - i) * 2 * OFF` with the order bus, trolleybus, tram; `OFF = 1.55 pt`, line width `LW = 2.6 pt`. Use `offset_curve` and keep round caps/joins.
7. Metro and suburban rail are drawn on top of everything (higher z-order), each as its own network, with their own line codes.

## 4. Tram line numbers

The OASA data has no tram line numbers. Use the official Athens tram lines (Wikipedia, in service since 6 Dec 2021): **T6** Syntagma – Pikrodafni and **T7** Asklipio Voulas – Akti Poseidonos (with the Piraeus loop). Build a track graph (networkx) from layer `_1` (2 m grid, glue tiny components up to 6 m, snap tram stops to edges by projection, splitting edges). Define the Greek stop-name sequences for T6 and T7 from layer `_7`, route shortest paths between consecutive stops and assign each edge T6, T7 or both (shared track). Parallel edges inherit labels within 25 m. In the diagram the pills read `T6`, `T7` (prefix `T` to avoid confusion with trolleybus 6 and 7). For the T6/T7 split in the exports use the same derived paths (a tram feature belongs to a line if at least 50 % of its length lies within 4 m of that line's path).

## 5. Page and styling

* One page. Page scale `S = 0.24 pt per metre` (about 13,418 × 12,693 pt including header, footer and margins; the PDF page limit is 14,400 pt). Draw the map with no basemap; background of the map area `#f2efe9`, white page margin.
* Light grid 2 × 2 km (`#d6d2c8`) with column letters A, B, C … along top and bottom and row numbers 1, 2, 3 … (top to bottom) at left and right.
* Colours: bus `#1f5fbf` (blue), trolleybus `#ff7300` (bright orange), tram `#7b2d8e` (purple), metro line 1 `#00a651`, line 2 `#e30613`, line 3 `#0079c2`, suburban rail white line with a thin black outline. Every bus/trolley/tram line sits on a white casing (width `LW + 1.5`). Metro lines `LW_M = 4.2` with a white casing; suburban rail `LW_R = 3.0` with a black outline `LW_R + 1.1`.
* Stop markers: white circles with a dark outline (`#1b1b1b`), radius 2.4 pt; metro stations radius 4.6 pt (circle), suburban rail stations radius about 4.1 pt (square).
* Fonts: DejaVu Sans Condensed (regular and bold). Embed as TrueType (`plt.rcParams['pdf.fonttype'] = 42`). Greek stop names stay in Greek exactly as in the data; metro/rail station names come from `NAME`, falling back to `STATHMOI` when `NAME` is null.
* All **bus, trolleybus and tram stop names use one uniform font size: 6 pt**. **Metro and suburban rail station names: 10 pt bold.**
* Route pills: rounded boxes, white bold text on mode colour, rotated along the line, placed beside the line and offset outside the parallel bundle when several modes share a street. Size tiers in pt: 20, 11, 7.5, 5.4, 4.0 (largest tiers for segments with the most route numbers; wrap long number lists into several rows).

## 6. Label placement (the hard part)

Use a grid-hash collision index with rotated rectangles. Place in this priority order: (1) metro and rail station names, (2) stop names, (3) route pills tier by tier.

* **Rule: no stop or station name may overlap any drawn line** (buses, trolleybuses, trams, metro, rail), and no label may overlap another label or marker. Keep a small clearance (`LW/2 + 0.45` pt for ordinary lines, `LW_M/2 + 0.9` for metro).
* Stop names: first try 8 adjacent positions around the marker (two radii); if none is free, search rings around the stop (radii 7 … 92 pt, 24 angles) with horizontal, 90° rotated and two-line (split at the space closest to the middle) variants, joined to the marker by a thin grey leader line (0.35 pt, `#666`); first require leaders not to cross lines, then relax that.
* Station names that cannot be placed without touching lines may be placed in a final pass that allows overlap with lines (they must still avoid other labels).
* Pills: skip a tier's candidates that do not fit along the segment; segments left without a pill after all tiers get a fallback small pill (3.1 pt then 2.5 pt) at one spot.
* Verification (print it): number of stop labels overlapping a line must be 0; every stop labelled; every distinct (mode, route number) appears in at least one pill.

## 7. Header, footer, legend (all text in English)

* Header band (`#3a1a5c`): title **Athens Public Transport Map**; subtitle *Bus · trolleybus · tram · metro · suburban rail · Αστικές συγκοινωνίες Αθήνας*; right side: "N bus and M trolleybus routes · tram · metro · suburban rail" and "Source: OASA (oasa.gr), data downloaded <date>".
* Footer note: Built from open OASA geodata; lines follow street centrelines without a base map; 2×2 km grid, GGRS87 (EGSA87); stop names use one small font and never cross lines; station names larger and bold; pills follow the line; tram numbers T6/T7 are absent from the OASA data and were taken from the Athens tram map and matched to tracks by stop sequence.
* Legend in the **bottom-left corner of the map** (the sea area, no routes there), white box with grey border, about `1280K × 1330K` pt where `K = S·56000/6150`. Title "Legend" (50K pt font), row text 31K pt, row step 108K. Rows: Bus route(s) on this segment; Trolleybus route(s) on this segment; Tram (lines T6, T7); Bus, trolleybus, tram on one street: side by side; Metro: line 1, line 2, line 3; Suburban railway; Route numbers (pill colour = mode); Bus, trolleybus, tram stop, name beside it; Metro station (larger bold name); Suburban rail station (larger bold name). A 5 km scale bar at the bottom of the legend. Header, grid letters and legend sizes scale with `K`.
* PDF metadata: Title "Athens Public Transport Map", Author "Data: OASA".

## 8. Output and checks

1. Save the single-page PDF (about 2 MB, TrueType fonts, `pdffonts` must not show Type 3). Rendering takes about 40–120 s.
2. Rasterise a few crops with `pdftoppm` (centre, a suburb, the legend, the header) and look at them: labels readable, nothing overlapping lines, legend in the bottom-left, no Cyrillic text anywhere (`pdftotext` check).
3. Be aware that a phone PDF viewer limits zoom to roughly 8,600 px of page width, so the file must be opened in a viewer with deep zoom (Adobe Acrobat Reader, Xodo, Google Drive) to read the 6 pt stop names; mention this in your final answer.
4. Deliver the PDF, the route ZIP and the stops ZIP.
