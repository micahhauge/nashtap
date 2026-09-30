# Research: public data sources for candidate Locations and Fun Facts

Issue: [#3](https://github.com/micahhauge/nashtap/issues/3). Researched 2026-09-29.

**Question.** Which public sources can give us candidate Locations (with coordinates) and material for one-line Fun Facts inside the Play Area? The Play Area is Davidson, Williamson, Rutherford, Maury, Wilson, Sumner, Robertson, Cheatham and Dickson counties. We need about 1,800 Locations a year.

**Method.** Where a source could be queried, I ran real queries and clipped the results to the nine counties. For the clip I used Census 2023 cartographic boundary polygons ([cb_2023_us_county_500k](https://www2.census.gov/geo/tiger/GENZ2023/shp/cb_2023_us_county_500k.zip)), point-in-polygon. Counts are as of 2026-09-29. They are raw candidates, before curation or deduplication across sources.

## Summary table

| Source | Raw candidates in Play Area | Has coordinates | Fun Fact material | License for reuse | Ease of extraction |
|---|---|---|---|---|---|
| **Wikidata** | 4,491 items with coordinates; **1,200** also have an English Wikipedia article (about 870 after dropping schools, radio stations, businesses and similar) | Yes (P625) | Structured facts (inception, named after, height, NRHP ID and more). Also links to the Wikipedia article | **CC0**, no attribution required | Very easy: one SPARQL query |
| **English Wikipedia** | About 1,200 articles (reached through Wikidata sitelinks) | Yes (via Wikidata / REST summary) | **Best** prose for facts | **CC BY-SA 4.0**: copied text needs attribution and share-alike | Easy: REST `page/summary` gives a lead extract |
| **USGS GNIS (Domestic Names)** | **3,380** named features: 1,102 populated places, 885 streams, 361 valleys, 207 summits, 139 reservoirs, 88 springs, and others | Yes | Almost none (name, class, county, dates only) | **Public domain**, USGS credit "appreciated" | Very easy: one pipe-delimited TN file |
| **NRHP (NPS + TDEC GIS)** | NPS spreadsheet: **592** listing rows. TDEC point layer: **576** points | TDEC layer: yes. NPS spreadsheet: address only | Name, listing date, area of significance. Nomination PDFs are rich but long | Federal data is **public domain**. Third-party photos and media on NPS pages are not | Easy: ArcGIS REST query or xlsx |
| **OpenStreetMap (Overpass)** | 1,315 named places (city to locality), 492 named parks/reserves, 499 named river/stream ways, 1,454 named features with a `wikidata=` tag. Only **133** `historic=*` and **8** plaque/board markers | Yes (best geometry) | Almost none (tags only) | **ODbL**: attribution, and share-alike on any derived *database* | Easy, but the public Overpass server timed out (HTTP 504) on bigger queries |
| **HMdb (Historical Marker Database)** | Not measurable: site blocks automated access (Cloudflare). A TDEC mirror of *Civil War* HMdb markers has **96** in the Play Area | Yes | Marker inscriptions are excellent fact material | **© HMdb.org LLC. Non-commercial reuse only, with attribution + link**. Commercial use needs written permission | Hard: no API, scraping blocked |
| **Tennessee Historical Commission** | Marker program: "over 2,000 markers" *statewide*, with no public list or dataset. The THC GIS layer is the architectural *survey*: 40,434 points in the Play Area, mostly private houses | Survey: yes | Markers: good but not available. Survey: poor | State data. No explicit open license found | Markers: not available. Survey: easy but wrong content |
| **TN State Parks / protected areas** | About 10 state parks and natural areas plus federal sites (e.g. Radnor Lake, Long Hunter, Cedars of Lebanon, Montgomery Bell, Harpeth River, Bledsoe Creek, Bicentennial Mall, Stillhouse Hollow Falls, Couchville Cedar Glade, Stones River NB) | Yes (via Wikidata/OSM) | Good (Wikipedia and park pages) | Use via Wikidata/Wikipedia | Trivial, but a tiny set |

## Details per source

### Wikidata (strongest backbone)

- **Query.** `wikibase:box` over the Play Area bounding box, then clipped to the county polygons. Result: **4,491** items with coordinate location (P625). Of these, **1,200** have an English Wikipedia sitelink and **491** carry an NRHP reference number (P649). SPARQL endpoint: <https://query.wikidata.org/sparql>. A transitive `P131+` ("located in", county) query timed out, so the geographic box approach is the one to use.
- **By county** (all items / items with an enwiki article):
  - Davidson 1,169 / 540
  - Williamson 687 / 202
  - Maury 662 / 97
  - Rutherford 486 / 105
  - Sumner 392 / 83
  - Dickson 353 / 26
  - Wilson 324 / 70
  - Robertson 242 / 53
  - Cheatham 176 / 24
- **Item types.** Many items without an article are bulk GNIS/NRHP imports: cemetery (1,013), valley (302), river (214), "mountain" (204), dam (111). Items with articles are mostly houses and historic districts, unincorporated communities (101), towns and cities, battles, venues and parks. They also include types to filter out, such as radio stations (64), schools (84) and shopping centers.
- **Surviving candidates.** Dropping obviously unsuitable types (schools, broadcasters, businesses, hotels, medical facilities and similar) leaves about **870** items with articles. This is a rough type filter. It also drops some historic churches that could stay.
- **License.** "All structured data in the main, property and lexeme namespaces is made available under the Creative Commons CC0 License" ([Wikidata:Licensing](https://www.wikidata.org/wiki/Wikidata:Licensing)). We can bundle derived data in a static site freely, with no attribution needed.

### English Wikipedia (best Fun Fact prose)

- **Access.** The REST endpoint `https://en.wikipedia.org/api/rest_v1/page/summary/{title}` returns a plain-text lead `extract` plus `coordinates`. This was tested on `Radnor_Lake_State_Natural_Area`: it returned the extract and `{lat: 36.0619, lon: -86.8075}`.
- **License.** Article text is under **CC BY-SA 4.0** (and GFDL) ([Wikimedia Terms of Use §7](https://foundation.wikimedia.org/wiki/Policy:Terms_of_Use)). If we ship Wikipedia sentences verbatim or lightly edited, we must credit the article (link it) and license that text CC BY-SA.
- **Using facts without copying text.** Facts themselves are not copyrightable. A Fun Fact **written fresh in our own words** from a fact in the article is not a copy of the expression. This is my reading, not legal advice. The simple safe practice is:
  1. Write one-liners in our own words.
  2. Keep a `source_url` per Location.
  3. Show a "Learn more" link to the article. This also covers attribution if any wording leans on the article.

### USGS GNIS: Domestic Names (best for natural features and small communities)

- **Download.** `DomesticNames_TN_Text.zip` from the USGS staged products bucket ([listing](https://prd-tnm.s3.amazonaws.com/?prefix=StagedProducts/GeographicNames/DomesticNames/)). It is a pipe-delimited file with `county_name` and decimal lat/lon, published 2026-09-29.
- **Play Area rows: 3,380.**
  - Populated Place 1,102
  - Stream 885
  - Valley 361
  - Civil 240
  - Summit 207
  - Reservoir 139
  - Spring 88
  - Census 71
  - Bend 52
  - Crossing 47 (historic ferries and fords)
  - Ridge 39
  - Military 33
  - Cliff 30
  - Island 29
  - Lake 22
  - Gap 16
  - and a few others
- **Cultural features are missing.** Churches, schools, cemeteries and buildings no longer appear in Domestic Names. Only physical, populated and civil classes do.
- **License.** From the metadata (`DomesticNames_TN_Text.xml`): `useconst`: "None. Acknowledgment of the U.S. Geological Survey would be appreciated for products derived from these data." The data is public domain.
- **Fun Facts.** GNIS gives none, so facts must come from elsewhere or from research. Many features, such as unnamed-looking valleys and small streams, are too obscure to be fair tap targets. Populated places, summits, rivers, lakes, springs and bends are the useful subset.

### National Register of Historic Places

- **NPS national spreadsheet.** [`national-register-listed_20260522.xlsx`](https://www.nps.gov/subjects/nationalregister/upload/national-register-listed_20260522.xlsx), linked from [NPS Database & Research](https://www.nps.gov/subjects/nationalregister/database-research.htm).
  - **592** rows for the nine counties (Davidson 211, Williamson 139, Maury 68, Rutherford 49, Sumner 39, Robertson 29, Wilson 25, Dickson 23, Cheatham 9). This includes some "Additional Documentation" rows.
  - Fields include name, city, street address, area of significance, category and listed date. There are **no coordinates**, and some addresses are "Address Restricted", which is typical for archaeological sites.
- **TDEC GIS point layer.** [`HISTORICAL/NATIONAL_REGISTER_TN/MapServer/0`](https://tnmap.tn.gov/arcgis/rest/services/HISTORICAL/NATIONAL_REGISTER_TN/MapServer) has **576** points in the Play Area, with REFNUM, name, address, county, city, listed date and NHL date. It is queryable as JSON/GeoJSON with `outSR=4326`.
- **License.** Government content is public domain, but "photographs, audio and video clips contributed or licensed by third parties … may be protected by copyright" ([NPS National Register Content and Copyright](https://www.nps.gov/subjects/nationalregister/national-register-content-and-copyright.htm)). The names, dates and significance fields are fine to use.
- **Fun Facts.** "Listed on the National Register in 1973 for its Greek Revival architecture"-style facts are easy to template. Richer facts need the nomination PDFs, which exist per REFNUM via NPGallery or the National Archives catalog.
- **Overlap.** Wikidata already has 491 of these items via P649.

### OpenStreetMap / Overpass

- **Counts.** Queries used `(poly:…)` with a simplified Play Area outline through <https://overpass-api.de/api/interpreter>, run with `out count`:
  - named `place` (city, town, village, hamlet, suburb, neighbourhood, quarter, locality): **1,315**
  - named `leisure=park|nature_reserve`: **492**
  - named `waterway=river|stream` ways: **499**
  - named `tourism=attraction|museum|viewpoint`: **81**
  - named `boundary=protected_area`: **27**
  - any named feature with a `wikidata` tag: **1,454**
  - `historic=*` of any kind: **133**
  - `historic=memorial` + `memorial=plaque|board`: **8**
- **Timeouts.** The named `natural=*` query timed out (HTTP 504) twice. The public instance is rate- and time-limited, so plan on small queries or a Geofabrik Tennessee extract.
- **Coverage.** OSM has the best **geometry** (neighborhood polygons and river lines), which matters for scoring a tap against an area Location. Historic content is thin here.
- **License.** ODbL 1.0: "as long as you credit OpenStreetMap and its contributors". If we alter or build upon the data, we may distribute the result only under the same licence ([openstreetmap.org/copyright](https://www.openstreetmap.org/copyright)). A static game that bundles a derived Locations JSON built from OSM geometry is a "Derivative Database". That JSON would have to be offered under ODbL, and the site needs a "© OpenStreetMap contributors" credit. The map basemap needs this credit anyway if it uses OSM tiles.

### HMdb (Historical Marker Database)

- **Access.** Blocked. `curl`, WebFetch and a headless browser all received a Cloudflare challenge or HTTP 403 on <https://www.hmdb.org>. HMdb has no public API or bulk export, so extraction would mean manual browsing.
- **Indirect sample.** TDEC republishes a subset: [`HISTORICAL/SECRETARY_OF_STATE_CIVIL_WAR/MapServer/1`](https://tnmap.tn.gov/arcgis/rest/services/HISTORICAL/SECRETARY_OF_STATE_CIVIL_WAR/MapServer/1), "Agency: The Historical Marker Database (hmdb.org) … Locations of Civil War-related historic markers". It has **96** markers in the Play Area (Rutherford 56, Davidson 23, Williamson 10, Maury 4, Wilson 2, Sumner 1), each with full inscription text and a link. The Nashville area as a whole likely has several hundred markers across all topics on HMdb, but I could not measure that.
- **License.** From the [HMdb copyright page](https://www.hmdb.org/copyright.asp), as summarized by search because the page itself is blocked:
  - Content is © HMdb.org LLC.
  - Copying is allowed with attribution to HMdb.org and a link.
  - **Commercial use requires written permission.**
- **Verdict.** Marker inscriptions (the words on the physical plaque) are great *fact leads*. Our Fun Facts should be written fresh and verified, not copied from HMdb pages.

### Tennessee Historical Commission

- **Marker program.** "Has erected over 2,000 markers commemorating sites, persons, and events significant in Tennessee history" statewide ([THC Historical Markers Program](https://www.tn.gov/historicalcommission/state-programs/markers.html)). No marker list or dataset is published. The THC newsletter *The Courier* announces new markers in prose, e.g. [Nineteen New State Historical Markers](https://tn.gov/historicalcommission/about-us/the-courier/the-courier-winter-2022/nineteen-new-state-historical-markers.html).
- **THC GIS layer.** [`HISTORICAL/HISTORICAL_COMMISSION/MapServer/0`](https://tnmap.tn.gov/arcgis/rest/services/HISTORICAL/HISTORICAL_COMMISSION/MapServer) is described as "Historic sites from the TN Historical Commission. Conversion from paper files to digital data is in progress so the whole state is not available yet."
  - It is the historic-architecture **survey**: 40,434 points in the Play Area, heavily skewed (Davidson 28,745, Wilson 73).
  - Only about 6,600 have a historic name, and about 35,000 are "Single family dwelling".
  - It is mostly private homes, so it is **not suitable** as a Location source.

### TN State Parks and other protected areas

- **State parks and natural areas.** The Play Area has a small set, found via Wikidata, each with a Wikipedia article:
  - Radnor Lake, Long Hunter, Bicentennial Capitol Mall, Couchville Cedar Glade (Davidson)
  - Cedars of Lebanon (Wilson)
  - Montgomery Bell (Dickson)
  - Harpeth River (Cheatham)
  - Bledsoe Creek (Sumner)
  - Stillhouse Hollow Falls (Maury)
- **Federal and other sites.** Stones River National Battlefield (Rutherford), Nashville National Cemetery, and several WMAs.
- **Verdict.** These are good Locations, but only about 15 to 20 in total. There is no need for a separate dataset: take them from Wikidata/OSM, with facts from Wikipedia or [tnstateparks.com](https://tnstateparks.com) written in our own words.

## Can these sources reach 1,800 Locations?

**No single source reaches 1,800 good Locations.** Combined, they give plenty of raw candidates:

- **Wikipedia-backed items:** about 870. These are notable places with fact material ready.
- **GNIS populated places, summits, rivers, lakes, springs and bends:** about 1,500 more. Some overlap with Wikidata, which imported GNIS.
- **NRHP properties:** about 580. About 490 are already in Wikidata.
- **OSM neighborhoods:** added on top.

Curation is the real bottleneck, not supply. Many GNIS and NRHP entries are too obscure, too tiny, or are private houses. Expect roughly 1,000 to 1,500 genuinely playable Locations after curation. That suggests either repeating Locations across years or loosening "one year = 1,800 unique" (e.g. allowing repeats after N months). That is a game-design decision, not a data one.

## Recommendation

1. **Use Wikidata as the backbone (CC0).** Run one SPARQL `wikibase:box` query clipped to the nine counties to get IDs, coordinates, types and enwiki links. Filter out businesses, schools and broadcasters. This yields about 870 Wikipedia-backed candidates.
2. **Add USGS GNIS (public domain)** for natural features and small communities: populated places, summits, rivers, lakes, springs, bends and historic ferries. Deduplicate against Wikidata via the GNIS ID (P590).
3. **Use NRHP (TDEC point layer, public domain)** for historic sites. Most overlap with Wikidata already. Its "significance" field seeds facts.
4. **Write Fun Facts in our own words, using Wikipedia as the research source.** Store a `source_url` per Location and show it as a "Learn more" link. Don't ship Wikipedia sentences verbatim unless we accept CC BY-SA on that text.
5. **Use OSM only for geometry** where we need polygons (neighborhoods, rivers, parks). If we ship OSM-derived data, credit "© OpenStreetMap contributors" and keep that derived file under ODbL.
6. **Don't build on HMdb or the THC survey.** HMdb is scrape-blocked and restricted to non-commercial use. The THC survey is mostly private homes. Use HMdb inscriptions only as manual fact leads.
7. **Credits to ship.** A footer or About line: "Location data from Wikidata (CC0), USGS GNIS and the National Register of Historic Places (public domain); map data © OpenStreetMap contributors (ODbL); facts researched from Wikipedia (links on each Location)."
