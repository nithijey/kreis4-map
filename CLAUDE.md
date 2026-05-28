# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Static, client-side web visualization of building data for **Kreis 4** (District 4) of Zürich, Switzerland. Each HTML file is a self-contained page (inline CSS + JS, no modules of our own) that loads map/3D libraries from a CDN and reads data from `assets/`. UI text is in German. There is **no build system, package manager, framework, or test suite** — editing an HTML file and refreshing the browser is the entire dev loop.

## Running locally

The pages `fetch()` local files in `assets/`, which fails under the `file://` protocol, so the site must be served over HTTP:

```bash
python -m http.server 8000   # then open http://localhost:8000/
```

`index.html` is the landing page and links to the two pages considered "final": `2d.html` and `3d_kreis4_final.html`. Runtime requires internet access for CDN libraries (unpkg) and basemap tiles (openfreemap.org / OpenStreetMap).

## Pages and their rendering stacks

Three independent rendering approaches exist across the HTML files. When changing behavior, know which stack a file uses:

- **`2d.html`** — Leaflet 1.9.4. OSM raster tiles + boundary + `circleMarker` points with table popups.
- **`3d_kreis4_final.html`** *(final 3D, linked from index)* — MapLibre GL 5.19.0, `liberty` style (its 3D buildings come from the style itself). Renders points as floating HTML `.bubble` markers offset upward by `FLOAT_PIXELS` to fake a ~30 m hover, plus a live stats panel (`updateStats`).
- **`3d_maplibre.html`**, **`3d_maplibre_volumen.html`**, **`3d_maplibre_volumen_30m.html`** — earlier MapLibre iterations using the `bright` style and an explicitly-added `3d-buildings` `fill-extrusion` layer over the `openfreemap` vector source. `volumen_30m` uses the same floating-bubble technique; the others use a flat `circle` layer. These are experiments, not linked from `index.html`.
- **`3d.html`** — Three.js 0.160.0 (WebGL, OrbitControls, GLTFLoader). The only page that does **not** use a web map: it loads `buildings_simple.glb` and `points3d.json` directly into a 3D scene and raycasts for click-to-info. Not linked from `index.html`.

## Data and coordinate systems (important)

`assets/` holds two parallel representations of the same buildings in **two different coordinate systems** — pick the right pair for the stack you are editing:

- **WGS84 / CRS84 lon-lat** (web maps): `points.geojson` (points + per-building properties) and `kreis4_boundary.geojson` (district outline polygon, `properties.kreis = 4`). Used by all Leaflet and MapLibre pages.
- **Local metric EPSG:2056** (Three.js): `points3d.json` (array of `{pos:[x,y,z], props:{…}}` in meters, relative to the origin in `meta.json`) and `buildings_simple.glb` (building meshes as simplified convex hulls). `meta.json` records the `EPSG:2056` origin offset and notes that building heights come from each mesh's z-range.

The two datasets also have **different property schemas**: `points.geojson` uses `egid_int` and a smaller field set; `points3d.json` uses `egid` plus many extra fields (`gvznummer`, LV95 coordinates, `oev_*`, `strasse_*`, etc.). Common fields across both: `strasse`, `hausnummer`, `plz`, `ortschaft`, `baujahr`, `grundflaeche_m2`, `gebaeudealter_2026`, `gebaeude_nutzung`, `qualitaet`.

Note: the MapLibre popups read `p.egid`, but `points.geojson` only provides `egid_int` — so the EGID line renders blank on those pages. Keep this mismatch in mind if you touch popup rendering.

## Conventions

- Each page is fully self-contained; shared styles/logic (e.g. the `popupHtml`/`esc`/`updateStats` helpers and the `.bubble` CSS) are **copy-pasted** across files rather than imported. A change to popup format or marker styling must be replicated in every relevant page.
- MapLibre pages share a view convention: `CENTER = [8.531, 47.377]` (lon, lat) and a `START` of `zoom 15.8, pitch 60, bearing -20`, then `fitBounds` to the boundary on load.
- User-facing values are escaped with the inline `esc()` helper when injected into popup HTML.
