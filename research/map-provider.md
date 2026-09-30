# Map library + tile provider for an unlabeled Play Area map

Resolves GitHub issue #2 (part of #1). Researched 2026-09-29 against official pricing, terms and docs pages. Prices and terms change, so re-check the linked pages before committing money.

## Question

Which map library and tile provider can render the Play Area (Davidson, Williamson, Rutherford, Maury, Wilson, Sumner, Robertson, Cheatham, Dickson counties) with **no place labels**, at zero or near-zero cost for a hobby-scale mobile static site, with room to grow? We compare cost at ~1k and ~50k daily players, label-free styling (including toggling roads, water and county lines), terms-of-use constraints, and mobile tap/zoom UX.

## TL;DR

**Use MapLibre GL JS with a self-hosted Protomaps PMTiles extract of the Play Area, served as a static file, and a custom style that has no symbol (label) layers.** Our measured extract is **7.5 MB at zoom 0–12** and 71 MB at zoom 0–15. It costs $0 at 1k daily players and about $0–25/month at 50k. No API key, no vendor quota, and the map can't show labels because we control the style. **OpenFreeMap** (same library, public vector tiles, free and unlimited but with no SLA) is the zero-setup fallback for prototyping. Avoid Mapbox, MapTiler, Stadia and CARTO. Their free tiers are non-commercial or too small, and at 50k players they cost hundreds to thousands of dollars a month.

## Assumptions used for cost math

- One game session per player per day, so 30 days × DAU sessions per month. 1k DAU is about **30k sessions/month** and 50k DAU is about **1.5M sessions/month**.
- About **50 vector-tile requests per session** (initial fit-to-Play-Area view around z8–9, plus some pinch-zoom to z11–13 while placing 5 taps). This is my estimate, not a measurement. Raster (256px) tiles need several times more requests for the same view. Dropping labels also removes glyph/font requests.
- Tile requests per month: 1k DAU is about **1.5M** and 50k DAU is about **75M**.

## Libraries

| | MapLibre GL JS | Leaflet | Mapbox GL JS v2+ |
|---|---|---|---|
| License | Open source (BSD-3) | Open source (BSD-2) | **Proprietary**: "for use only with the relevant Mapbox product(s)… developers with a current active Mapbox account" ([LICENSE](https://github.com/mapbox/mapbox-gl-js/blob/main/LICENSE.txt)) |
| Vector tiles / runtime styling | Yes (WebGL). Hide or show any layer at runtime | **No**. Core is raster `TileLayer`/`GridLayer` only ([reference](https://leafletjs.com/reference.html)). Vector needs a plugin such as [protomaps-leaflet](https://github.com/protomaps/protomaps-leaflet) | Yes, but tied to Mapbox |
| PMTiles | First-class via `maplibregl.addProtocol("pmtiles", protocol.tile)` ([docs](https://docs.protomaps.com/pmtiles/maplibre)) | Via protomaps-leaflet | n/a |
| Mobile UX knobs | `maxBounds`, `minZoom`/`maxZoom`, `dragRotate`, `touchZoomRotate` (rotation can be disabled), `touchPitch`, `clickTolerance`, `cooperativeGestures`, `renderWorldCopies` ([MapOptions](https://maplibre.org/maplibre-gl-js/docs/API/type-aliases/MapOptions/)) | `tapTolerance` (default 15px), `touchZoom`, `bounceAtZoomLimits`, `maxBounds`, `maxBoundsViscosity` ([reference](https://leafletjs.com/reference.html)) | Same as MapLibre |

**Verdict:** MapLibre GL JS. It's the only open option that natively supports the "toggle roads/water/county lines" requirement. You style vector layers instead of choosing among a vendor's pre-baked raster variants. Pinch-zoom is smooth and fractional, and it works with every free vector source below. Leaflet is lighter and fine for a CARTO `*_nolabels` raster basemap, but it can't toggle individual features. For NashTap, set `dragRotate: false`, `touchPitch: false`, disable touch rotation, `maxBounds` a bit larger than the Play Area, and a small `clickTolerance` so a pan isn't read as a guess. For a full-screen game map, leave `cooperativeGestures` off; it forces two-finger panning on mobile.

## Tile providers

### 1. Self-hosted Protomaps PMTiles (recommended)

- **What:** Protomaps publishes daily planet builds from OpenStreetMap ([builds](https://maps.protomaps.com/builds/)). `pmtiles extract <url> out.pmtiles --bbox=minlon,minlat,maxlon,maxlat` cuts a region ([getting started](https://docs.protomaps.com/guide/getting-started)). The file is a single static archive that the browser reads with HTTP Range requests, so no tile server is needed ([cloud storage](https://docs.protomaps.com/pmtiles/cloud-storage)).
- **Measured for the Play Area** (bbox `-87.75,35.35,-85.85,36.72`, build `20260929`, pmtiles CLI):
  - `--maxzoom=12`: **7.5 MB**
  - full detail (to z15): **71 MB**
- **Layers** ([docs](https://docs.protomaps.com/basemaps/layers)): `earth, landcover, landuse, water, roads, buildings, places, pois, boundaries, transit`. The `boundaries` layer has `kind` values of `country, region, county, locality`. I decoded a z10 Middle-Tennessee tile from the extract and confirmed that **`kind=county` (admin level 6) lines are present**. So roads, water and county lines are separate layers you can toggle with `map.setLayoutProperty(id, 'visibility', ...)`.
- **Label-free styling:** the `@protomaps/basemaps` package generates the style. Labels only appear if you pass `{lang: "en"}` to `layers()`, and there's also a `labelsOnly` option ([MapLibre styling](https://docs.protomaps.com/basemaps/maplibre)). Flavors: light, dark, white, black, grayscale. Simplest approach: generate the style and drop every `type: "symbol"` layer, or omit the `places`/`pois` layers entirely. Because we ship the style, no label can leak in.
- **Hosting cost:**
  - The z0–12 file (7.5 MB) fits comfortably under the Cloudflare Pages 25 MiB per-asset limit ([limits](https://developers.cloudflare.com/pages/platform/limits/)) and the GitHub Pages 1 GB site limit ([limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits)). It could ship alongside the static site. Note that GitHub Pages has a 100 GB/month soft bandwidth limit and prohibits running an online business or commercial SaaS on it. Also verify that the chosen static host honours HTTP Range requests before relying on it.
  - Cloudflare R2 is Protomaps' recommended bucket because it has "no bandwidth fees, only per-request fees" ([cloud storage](https://docs.protomaps.com/pmtiles/cloud-storage)). R2 free tier: 10 GB storage and **10M Class B (read) ops/month**, then $0.36/M, with free egress ([R2 pricing](https://developers.cloudflare.com/r2/pricing/)).
  - **1k DAU** (~1.5M reads): **$0**.
  - **50k DAU** (~75M reads): at worst ~65M × $0.36/M, about **$23/month** if every read hits R2. If Cloudflare's CDN cache is put in front (custom domain plus cache rules), most reads never reach R2, so real cost trends toward $0. The whole Play Area is only ~7.5 MB, so the cache hit rate should be very high.
- **Terms:** OSM data is ODbL, and Protomaps is BSD/ODbL ([getting started](https://docs.protomaps.com/guide/getting-started)). You must show "© OpenStreetMap" attribution. There's no vendor ToS, quota or key.
- **Trade-off:** you re-run `pmtiles extract` whenever you want fresher OSM data, which is rare for a regional basemap. You also own the style JSON.

### 2. OpenFreeMap (public instance)

- **Cost:** free, with "no limits on the number of map views or requests", no registration and no API keys. Commercial use: "Yes" ([openfreemap.org](https://openfreemap.org/)). **$0 at both 1k and 50k DAU.**
- **Terms:** no SLA. The service "may discontinue it at any time without notice" and is provided as-is ([ToS](https://openfreemap.org/tos/)). Attribution "OpenFreeMap © OpenMapTiles Data from OpenStreetMap" is required ([openfreemap.org](https://openfreemap.org/)). The whole planet is also downloadable for self-hosting.
- **Styling:** it uses the OpenMapTiles schema, with separate `boundary`, `transportation`, `water`, `waterway`, `place`, `transportation_name` and `water_name` layers ([schema](https://openmaptiles.org/schema/)). I fetched a live z10 tile and confirmed `boundary` has **admin_level 6 (county)** lines. The stock Positron style has 19 `symbol` layers, which you'd strip to go label-free. Styles: Positron, Bright, Liberty, Dark, Fiord.
- **Verdict:** the best zero-effort start and a fine fallback. The risk is a single donation-funded operator with no SLA for a daily game.

### 3. MapTiler Cloud

- Free: 5,000 sessions or 100,000 API requests per month, **non-commercial only**, and the service pauses when exceeded. Flex: $30/month for 25k sessions / 500k requests, with overage billed ([pricing](https://www.maptiler.com/cloud/pricing/)).
- With plain MapLibre GL JS, billing is **per request**. Session billing (6 h / 10k requests per session) only applies with MapTiler SDK JS or its Leaflet plugin ([sessions](https://docs.maptiler.com/guides/maps-apis/maps-platform/what-is-map-session-in-maptiler-cloud/)).
- **1k DAU:** 30k sessions or ~1.5M requests, which already exceeds Free and Flex quotas, so paid overage applies. **50k DAU:** 1.5M sessions puts you in Custom (prepaid contract) territory.

### 4. Stadia Maps

- Free: 200k credits, **non-commercial or evaluation only** ("a personal site" is allowed). Starter $20 for 1M credits, then 3¢/1k. Standard $80 for 7.5M, then 2¢/1k. Professional $250 for 25M, then 1.5¢/1k. **1 credit per vector/raster tile** ([pricing](https://stadiamaps.com/pricing/), [ToS](https://stadiamaps.com/terms-of-service/)). Server-side caching is prohibited ([ToS](https://stadiamaps.com/terms-of-service/)).
- **1k DAU** (~1.5M tiles): about **$35/month** (Starter plus overage). **50k DAU** (~75M): about **$1,000/month** (Professional plus overage).

### 5. Mapbox

- GL JS: 50,000 free map loads per month, then $5 / $4 / $3 / $2.50 per 1k in tiers, with unlimited tile requests per load ([pricing](https://www.mapbox.com/pricing)).
- **1k DAU** (30k loads): **$0**. **50k DAU** (1.5M loads): 50k×$5 + 100k×$4 + 800k×$3 + 500k×$2.50, about **$4,300/month**.
- Constraints: the Mapbox logo is required on the map, even when rendering non-Mapbox data ([attribution](https://docs.mapbox.com/help/getting-started/attribution/)). The v2+ SDK is proprietary and requires a Mapbox account ([LICENSE](https://github.com/mapbox/mapbox-gl-js/blob/main/LICENSE.txt)). Mapbox Studio would make unlabeled styling easy, but the cost curve and lock-in rule it out.

### 6. CARTO basemaps (incl. `*_nolabels`)

- Ready-made raster `light_nolabels`, `dark_nolabels`, `voyager_nolabels` (plus `*_only_labels`), and vector Positron/Voyager/Dark Matter ([basemap-styles](https://github.com/CartoDB/basemap-styles)).
- **An API key is now required.** Free: **5M tile requests/month non-commercial, 1M/month commercial**, aggregated across keys. "Commercial" includes anything that "generates revenue (including through advertising)". Above that you need a paid Basemaps Commercial plan with unpublished pricing ([terms](https://carto.com/legal/basemap-terms/), [basemap-styles](https://github.com/CartoDB/basemap-styles)).
- `*_nolabels` rasters are the quickest way to get "no labels" with Leaflet. But raster means you can't toggle roads, water or county lines. **1k DAU** probably fits the non-commercial 5M (raster needs more tiles per view). **50k DAU** is far over, at an unknown price.

## Cost summary (per month)

| Option | ~1k DAU | ~50k DAU | Key / quota | Label-free + layer toggles | Main risk |
|---|---|---|---|---|---|
| **Protomaps PMTiles, self-hosted (R2 / static)** | **$0** | **$0–~$23** | None | Yes, full control | You maintain the extract + style |
| OpenFreeMap public | $0 | $0 | None | Yes (strip symbol layers) | No SLA, may disappear |
| Mapbox | $0 | ~$4,300 | Token, logo required | Yes | Cost, proprietary SDK |
| Stadia | ~$35 (free tier non-commercial) | ~$1,000 | Key | Partial | Cost |
| MapTiler | Over free/Flex quota | Custom contract | Key | Yes | Cost, non-commercial free tier |
| CARTO | ~$0 if non-commercial | Paid, unpublished | Key (new) | Labels only, no toggles (raster) | Commercial definition, cost |

## Recommendation

1. **Library: MapLibre GL JS.** It's open source, vector, and lets us toggle layers at runtime. Configure it for a game: no rotate or pitch, `maxBounds` around the Play Area, and `click` events for taps with a tight `clickTolerance`.
2. **Tiles: a Protomaps PMTiles extract of the Play Area bbox**, capped around z12–13 to keep it at single-digit MB, served as a static file alongside the site. If the static host doesn't support Range requests or the file grows, move it to Cloudflare R2 behind the CDN. Both cost essentially $0 at 1k DAU and stay near $0 at 50k.
3. **Style: our own minimal style** built from `@protomaps/basemaps` with **no symbol layers**. Expose `water`, `roads` (maybe filtered by `kind` to highways and major roads) and `boundaries` (`kind=county`) as togglable layers. We'll likely also overlay our own Play Area outline (for example, Census county polygons) for scoring and display.
4. **Prototype on OpenFreeMap first** (zero setup, same library and a similar style approach), then swap the source URL to the PMTiles file before launch to remove the no-SLA dependency.
5. Show "© OpenStreetMap" attribution (MapLibre's `AttributionControl` in compact mode keeps it small on mobile).
