# Satellite imagery source for the Play Area

Resolves GitHub issue #10. Researched 2026-09-29 against official pricing, terms and docs pages, plus measurements I took against live services (noted as *measured*). Prices and terms change, so re-check the linked pages before committing money.

## Question

Which imagery source covers the Play Area (Davidson, Williamson, Rutherford, Maury, Wilson, Sumner, Robertson, Cheatham, Dickson counties) at good quality, with licensing that allows a commercial-leaning hobby game showing imagery without labels, at ~$0 for ~1k daily players and a reasonable cost at ~50k? For self-hosting: storage per max zoom, build effort, and whether Vercel can serve it or Cloudflare R2 is needed. And what max zoom makes landmark-level Locations identifiable?

## TL;DR

**Self-host USDA NAIP 2023 imagery (0.6 m, public domain) as a single PMTiles archive of 512 px WebP tiles to z15 (the same detail as 256 px z16), stored on Cloudflare R2 behind a custom domain, rendered by MapLibre as a `raster` source.** The archive is about **1.1–1.6 GB** clipped to the counties. It costs **$0/month at 1k DAU** and roughly **$0–30/month at 50k DAU**. Building it takes about a day of work and a few hours of compute. Every hosted option (Esri, Mapbox, MapTiler, Google) is $0 only at tiny volumes: at 1k DAU they cost about $100–$3,000/month, and at 50k DAU about $4,000–$14,000/month. Vercel shouldn't serve the tiles. Hobby forbids commercial use, caps CLI uploads at 100 MB and includes only 100 GB/month of transfer, and on Pro the bandwidth alone would be about $800/month at 50k DAU.

## Assumptions used for cost math

- Sessions: 1k DAU is about **30k sessions/month** and 50k DAU about **1.5M sessions/month** (same as `research/map-provider.md`).
- Tiles per session: prior research estimated ~50 *vector* tile requests per session. Raster is similar if we use **512 px tiles**. I assume **~60 × 512 px tiles/session**, or **~240/session for 256 px tile services** (4× the count for the same screen area). This is an estimate, not a measurement.
- Bytes per tile (*measured*): I rendered 16 random 2023 NAIP quarter-quads in the Play Area, plus Nissan Stadium, the Parthenon and the Capitol, through Planetary Computer's tiler at z14–z17. Mean size per **256 px** tile was **23.3 KB JPEG** and **16.2 KB WebP** (tiler default quality). A 512 px tile is about 4×, so ~65 KB WebP.
- Result: **~3.5–4 MB of imagery per session**. At 1k DAU that is about **1.8M tiles and ~105 GB/month**. At 50k DAU it is about **90M tiles and ~5.3 TB/month**.

## What max zoom do Locations need?

Ground resolution for a 256 px tile at 36°N is **z14 = 7.7 m/px**, **z15 = 3.9 m/px**, **z16 = 1.9 m/px** and **z17 = 0.97 m/px** (computed from Web Mercator). A 512 px tile at zoom *z* has the resolution of a 256 px tile at *z+1*.

I viewed rendered NAIP 2023 tiles (*measured*):

- **z15 (3.9 m/px):** Nissan Stadium, the Cumberland River bridges and the storage tanks beside the stadium are clearly recognizable. Stadium- and campus-scale landmarks work at this zoom, and so should towns and large natural features (lakes, river bends).
- **z16 (1.9 m/px):** the Parthenon and Centennial Park's lawn and paths are identifiable. Building-scale landmarks (the Capitol, the Parthenon) need this.
- **z17 (0.97 m/px):** the Capitol's dome, parking and individual cars are crisp. This is nice but not needed, and NAIP's native 0.6 m only just resolves it.

**Recommendation:** store up to **256 px z16 equivalent (= 512 px z15)**. Let MapLibre overzoom to map zoom ~17–18 (set `maxzoom` on the source to 15 with `tileSize: 512`, and the map's `maxZoom` to 18). Going to z17 quadruples storage (~4.6–6.6 GB) for detail a player won't need.

## Sources compared

### 1. USDA NAIP, self-hosted as PMTiles (recommended)

- **Coverage/quality:** NAIP captures each state at least every three years as 4-band (RGB+NIR) orthoimagery. Planetary Computer hosts 2010–present as COGs ([Planetary Computer NAIP](https://planetarycomputer.microsoft.com/dataset/naip)). A STAC query I ran for the Play Area bbox (*measured*) returned **703 Tennessee quarter-quads for 2023 at 0.6 m**, plus 0.6 m 2021 and 2018 and 1 m 2012–2016. Kentucky tiles only cover the bbox's northern edge, outside the counties. Caveat: the 2023 acquisition dates span **May–November** (374 May, 102 June, 18 Aug, 104 Oct, 105 Nov), so a mosaic will show some seasonal color seams between flight lines. The 2025 NAIP cycle is flying some states at 30 cm ([NAIP 2025 sample metadata](https://maris.mississippi.edu/METAdata/METAdata_Aerial/NAIP/NAIP_2025_Sample_metadata.pdf)). Check the [NAIP hub](https://naip-usdaonline.hub.arcgis.com/) for a newer Tennessee year before building.
- **License:** **public domain** (the Planetary Computer STAC collection's license link is titled "Public Domain", pointing at [FSA policies](https://www.fsa.usda.gov/help/policies-and-links/)). No attribution is legally required, but crediting "Imagery: USDA NAIP" is good practice. There are no restrictions on commercial use, caching, or display without labels.
- **Storage per max zoom** (*computed*: tile counts from the bbox `-87.75,35.35,-85.85,36.72` and from the 9 counties' total area of 4,751 sq mi / 12,304 km² from the [Census 2023 Gazetteer](https://www2.census.gov/geo/docs/maps-data/data/gazetteer/2023_Gazetteer/2023_Gaz_counties_national.zip) +8% edge tiles, multiplied by the measured bytes per tile):

  | Max zoom (256 px equiv.) | Tiles, clipped to counties | WebP | JPEG | Tiles, full bbox | WebP | JPEG |
  |---|---|---|---|---|---|---|
  | z14 (7.7 m/px) | 4.5k | **~72 MB** | ~103 MB | 9.2k | ~145 MB | ~209 MB |
  | z15 (3.9 m/px) | 18k | **~290 MB** | ~410 MB | 36k | ~570 MB | ~820 MB |
  | z16 (1.9 m/px) | 72k | **~1.1 GB** | ~1.6 GB | 143k | ~2.3 GB | ~3.3 GB |
  | z17 (0.97 m/px) | 290k | ~4.6 GB | ~6.6 GB | 571k | ~9.0 GB | ~13 GB |

  512 px tiles hold the same pixels in a quarter as many tiles, so the byte totals are the same at one zoom lower. Clip to the county polygons: outside the counties, just render a flat dark background (and it halves storage).
- **Build effort (about 1 day):**
  1. Query Planetary Computer STAC for `naip`, year 2023, intersecting the counties. Get a free SAS token from `https://planetarycomputer.microsoft.com/api/sas/v1/token/naipeuwest/naip` ([PC docs](https://planetarycomputer.microsoft.com/dataset/naip)). One source COG is ~450 MB (*measured* via HEAD), about 300 GB in total at full resolution. GDAL reads only the needed overview levels via `/vsicurl/`, so the real transfer is much smaller. You can also download USDA's compressed county mosaics from the [Geospatial Data Gateway](https://datagateway.nrcs.usda.gov).
  2. Build a VRT of bands 1-2-3 with `gdalbuildvrt`, then cut it to the county polygons with `gdalwarp -cutline` (with alpha).
  3. Run `rio pmtiles --format WEBP --tile-size 512 --zoom-levels 0..15 in.vrt nashtap-naip.pmtiles` ([Protomaps: creating PMTiles](https://docs.protomaps.com/pmtiles/create)). Protomaps recommends 512 px tiles for MapLibre and supports JPEG/WEBP. GDAL ≥ 3.8 also writes PMTiles, but Protomaps recommends that "only for smaller datasets".
  4. Upload to R2 via the S3 API (for example rclone), then add the `pmtiles://` protocol in MapLibre (as in the vector research).
  Rebuild only when a new NAIP year lands, which is every 2–3 years.
- **Hosting:** see "Where to host the tiles" below. **$0 at 1k DAU and about $0–30 at 50k DAU on R2.**

### 2. Esri World Imagery (ArcGIS Location Platform)

- **Quality:** the best-looking option. The service describes "Vantor [formerly Maxar] imagery at 0.3m resolution for select metropolitan areas… 0.5m resolution across the United States", plus community aerials down to 0.03 m ([World_Imagery service description](https://services.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer?f=pjson)). The service uses 256 px JPEG tiles and has no labels (labels are a separate `imagery/labels` layer).
- **Access:** `https://ibasemaps-api.arcgis.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}?token=…`. It requires an ArcGIS Location Platform account and an API key with the "Static basemap tiles" privilege ([MapLibre tutorial](https://developers.arcgis.com/maplibre-gl-js/maps/raster-tile-basemaps/display-multiple-basemap-layers/)). The Static Basemap Tiles service itself only offers `arcgis/imagery/labels` for imagery ([service docs](https://developers.arcgis.com/rest/static-basemap-tiles/)). The unlabeled imagery comes from the World_Imagery tile service or the `arcgis/imagery` style of the Basemap Styles service ([Basemap styles](https://developers.arcgis.com/rest/basemap-styles/)).
- **Pricing:** **2M basemap tiles/month free, then $0.15 per 1,000**. The alternative is **1k basemap sessions free, then $4 per 1,000 sessions** ([pricing](https://location.arcgis.com/pricing/)).
  - 1k DAU (~7.2M × 256 px tiles): about **$780/month**. With session pricing, if it applies to this layer, 30k sessions cost about **$116/month**.
  - 50k DAU: tile pricing is about **$54k/month** (360M tiles), and session pricing about **$6,000/month**.
- **Terms:** the app must show "Powered by Esri" (linked to esri.com) plus the data attribution "Esri, Vantor, Earthstar Geographics, and the GIS User Community". With MapLibre, add these via `customAttribution` ([Esri attribution](https://developers.arcgis.com/documentation/esri-and-data-attribution/)). Commercial use is allowed on the paid platform.
- **Verdict:** great for a quality benchmark or a prototype inside the free 2M tiles (about 8k sessions/month), but it gets expensive quickly.

### 3. Mapbox Satellite

- **Pricing:** raster tiles are **750k free/month**, then $0.25 / $0.20 / $0.15 per 1,000 in tiers. Mapbox Satellite is billed as Raster Tiles API requests. GL JS map loads are 50k free, then $5 → $2.50 per 1,000 ([pricing](https://www.mapbox.com/pricing)). Third-party renderers such as MapLibre are billed **per tile request** ([pricing guide](https://docs.mapbox.com/accounts/guides/pricing/)).
  - Via MapLibre, 512 px, 1k DAU (1.8M tiles): about **$260/month**. At 50k DAU (90M): about **$13.6k/month**.
  - Via Mapbox GL JS (map loads): **$0** at 1k DAU and about **$4,300/month** at 50k DAU. But that means the proprietary SDK and Mapbox logo lock-in (see `research/map-provider.md`).
- **Verdict:** high quality, but the cost curve rules it out.

### 4. MapTiler Satellite

- Free plan: 100k requests / 5k sessions, **non-commercial only**, and the service pauses when exceeded. Flex is $30/month for 500k requests / 25k sessions, then **$0.15/1k requests** or **$2.50/1k sessions** ([pricing](https://www.maptiler.com/cloud/pricing/)). Plain MapLibre is billed per request (see `research/map-provider.md`).
- 1k DAU: about **$225/month** (requests). With the MapTiler SDK billed per session, about $43/month. 50k DAU: about **$13.5k/month** by requests, or about $3.7k by sessions.

### 5. Google Map Tiles API (2D satellite)

- 2D Map Tiles are **100k free/month**, then $0.60 → $0.045 per 1,000 in tiers ([pricing](https://developers.google.com/maps/billing-and-pricing/pricing)). `mapType: "satellite"` requires a session token, and the tiles are 256 px ([2D tiles overview](https://developers.google.com/maps/documentation/tile/2d-tiles-overview)). The Google logo and data attribution are mandatory, and offline or non-visualization use is prohibited ([policies](https://developers.google.com/maps/documentation/tile/policies)).
- 1k DAU (~7.2M tiles): about **$3,250/month**. **Ruled out.**

### 6. Sentinel-2 cloudless (EOX)

- 10 m resolution, useful only to about z13. Landmarks aren't identifiable. Commercial use needs a separate license with EOX, and non-commercial use is CC BY-NC-SA 4.0 with mandatory attribution ([license summary](https://cloudless.eox.at/documentation/license)). **Ruled out** for resolution and licensing.

### 7. Other free public sources worth knowing

- **USGS Imagery Only** tile service (`basemap.nationalmap.gov/.../USGSImageryOnly/MapServer`): public-domain NAIP, but only 1 m from 2017–2021, visible to ~1:9k (~z16), with no SLA ([service description](https://basemap.nationalmap.gov/arcgis/rest/services/USGSImageryOnly/MapServer?f=pjson)). OK for a quick prototype, but not something to hotlink in production.
- **TDOT / TNMap orthoimagery** (`tnmap.tn.gov/.../BASEMAPS/IMAGERY_WEB_MERCATOR`): **6-inch imagery (2022+)**, 1 ft earlier, flown one TDOT region per year ([service description](https://tnmap.tn.gov/arcgis/rest/services/BASEMAPS/IMAGERY_WEB_MERCATOR/MapServer?f=pjson)). It has the best resolution of anything here, but the service metadata has **no license or terms**. Don't use it without written permission from TDOT / the TN GIS office. It's worth an email if NAIP quality disappoints.

## Where to host the tiles (Vercel vs Cloudflare R2)

**Vercel: not suitable for the imagery.**

- Range requests work on Vercel static files. I *measured* a `206 Partial Content` response on a Vercel-served asset, so PMTiles would technically work.
- **The Hobby plan is non-commercial only**: "Hobby teams are restricted to non-commercial personal use only", and ads or payments count as commercial ([fair use](https://vercel.com/docs/limits/fair-use-guidelines)). It includes **100 GB Fast Data Transfer/month**, and when a limit is hit the feature pauses for 30 days ([Hobby plan](https://vercel.com/docs/plans/hobby)). At 1k DAU we'd use about 105 GB.
- **CLI static upload limit: 100 MB on Hobby and 1 GB on Pro** ([limits](https://vercel.com/docs/limits)). A z16-equivalent archive (~1.1 GB WebP) doesn't fit even on Pro.
- On Pro, usage-based Fast Data Transfer is **$0.15/GB** in iad1 ([regional pricing](https://vercel.com/docs/pricing/regional-pricing/iad1)). At 50k DAU (~5.3 TB) that's about **$800/month**. Pro also offers a "Flat Rate CDN" with 1 TB included and spike protection, but tier prices aren't published on the [announcement](https://vercel.com/changelog/flat-rate-cdn-is-now-ga-for-pro-teams).
- Vercel Blob caches objects only up to **512 MB**; larger blobs miss the cache on every access ([Blob pricing](https://vercel.com/docs/vercel-blob/usage-and-pricing)).
- Keep the Next.js app on Vercel and host the imagery elsewhere.

**Cloudflare R2: recommended.**

- Free tier: **10 GB-month storage, 1M Class A and 10M Class B operations, and free egress**. Beyond that, $0.015/GB-month and $0.36 per million reads ([R2 pricing](https://developers.cloudflare.com/r2/pricing/)).
- Serve from a **custom domain, not `r2.dev`**. `r2.dev` "is rate-limited and should only be used for development purposes", and only a custom domain gets Cloudflare Cache and cache rules ([public buckets](https://developers.cloudflare.com/r2/buckets/public-buckets/)). The domain must be a Cloudflare zone.
- Cloudflare's CDN caches files only up to **512 MB on Free/Pro/Business** ([cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/)). So a 1+ GB PMTiles file read directly by range requests mostly **misses the CDN and bills R2 reads**. There are three options:
  1. **Direct PMTiles range reads (simplest):** 1k DAU uses ~1.8M reads, which is **$0**. 50k DAU uses ~90M reads, about **$29/month**.
  2. **Protomaps' Cloudflare Worker** serving `/{z}/{x}/{y}` with edge caching (needs a custom domain) ([Protomaps Cloudflare](https://docs.protomaps.com/deploy/cloudflare)). Workers cost $5/month for 10M requests, then $0.30/M, so it's no cheaper at 50k DAU. It's worth doing mainly for latency, since Protomaps notes R2 can have "500ms or higher" latency on misses.
  3. **Explode to individual tile objects** (~18–72k objects with 512 px tiles; well inside the 1M free Class A writes) behind the custom domain with a cache rule. Almost every request becomes a cache hit, so this costs **about $0 at 50k DAU**. It's the cheapest at scale but means more files to manage.
- Storage for ~1.1–1.6 GB is within the free 10 GB.

## Cost summary (per month, estimates)

| Option | Quality | 1k DAU | 50k DAU | License fit |
|---|---|---|---|---|
| **NAIP 0.6 m on R2 (PMTiles)** | Good (0.6 m, 2023, some seasonal seams) | **$0** | **$0–30** | Public domain, no constraints |
| NAIP on Vercel Pro | Same | $20 + overage, file too large for static upload | ~$800+ | Hobby prohibits commercial use |
| Esri World Imagery | Best (0.3–0.5 m) | ~$116 (sessions) – $780 (tiles) | ~$6k–54k | Commercial OK, "Powered by Esri" + attribution |
| Mapbox Satellite | Very good | $0 (GL JS) – $260 (MapLibre) | ~$4.3k–13.6k | Logo required, proprietary SDK for load pricing |
| MapTiler Satellite | Good | ~$43–225 | ~$3.7k–13.5k | Free tier non-commercial |
| Google 2D satellite | Best | ~$3,250 | far higher | Logo, no caching |
| Sentinel-2 cloudless | Too coarse (10 m) | – | – | Non-commercial unless licensed |

## Recommendation

1. **Use USDA NAIP 2023 (0.6 m, public domain), tiled once into a county-clipped PMTiles archive of 512 px WebP tiles, zoom 0–15** (the same detail as 256 px z16, 1.9 m/px). This is **~1.1 GB** and enough to identify building-scale landmarks such as the Parthenon and the Capitol. Allow MapLibre to overzoom to ~18.
2. **Host it on Cloudflare R2 behind a custom domain**, not Vercel. It's $0 at 1k DAU and at most ~$30/month at 50k DAU with direct range reads. Switch to exploded, CDN-cached tiles if you want it to be ~$0 at scale. Keep the Next.js app itself on Vercel. Note that commercial use of Vercel needs Pro, regardless of imagery.
3. Before building, check the [NAIP hub](https://naip-usdaonline.hub.arcgis.com/) for a **2025 Tennessee** acquisition, since some 2025 states were flown at 30 cm. If there is one, use it instead.
4. For a zero-effort prototype, point MapLibre at Esri World Imagery within its free 2M tiles/month (with "Powered by Esri" attribution), or at the USGS Imagery Only service. Don't launch on either.
5. Optional: email TDOT / the TN GIS office about licensing TNMap's 6-inch imagery if NAIP's seasonal seams or 0.6 m detail prove to be a problem in playtesting.
