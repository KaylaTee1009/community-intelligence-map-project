# Silver Spring Community Intelligence

Bay Hacks 2026 — UX University Challenge.

Public data about **Silver Spring, Maryland** is scattered across census
records, county datasets and GIS layers. This pulls seven public sources into
one warehouse, joins them on census tract, and presents them as four sectors you
can ask questions of — with **Fenton Village** as the worked application.

No API keys are required to run it.

---

## 🏆 Recognition

**2nd Place — Bay Hacks 2026, UX University Challenge**

Built in 24 hours by **Team OmniBulls** 🐂 — (Me) [Kayla Teekaram](https://www.linkedin.com/in/kayla-teekaram-121580272),
[Monisha Natarajan Ambika](https://www.linkedin.com/in/monisha-natarajan/), and
[Reyanna Hodge](https://www.linkedin.com/in/reyanna-hodge-024268421/).

The project earned a specially created second-place award from the division
sponsor, along with an internship opportunity to continue development and a
cash prize.

- 📝 [Read the full story on LinkedIn](https://www.linkedin.com/posts/kayla-teekaram-121580272_hackbay2026-tampabayinnovation-hackathon-ugcPost-7507599736615636992-nbcS/?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEKl5KkB78H_KsNFdZANIJoeyRPLphRjnzo)
- 🔗 [Project on Devpost](https://devpost.com/software/silver-springs-community-intelligence)
- 🎥 [Watch the demo video](https://youtu.be/RAH8OKLhbwo?si=-50LmfvIefb3Pd1I)

---

## Run

Two processes. Start the API first.

```bash
python -m venv .venv
.venv/Scripts/python.exe -m pip install requests pandas duckdb fastapi uvicorn
.venv/Scripts/python.exe backend/build.py        # fetch sources, build the warehouse
```

```bash
cd backend && ../.venv/Scripts/python.exe -m uvicorn api:app --port 8000
```

```bash
npm install && npm run dev                        # http://localhost:5173
```

`build.py` caches every upstream response to `backend/data/raw/`, so it only
hits the network once. After the first build the whole stack runs offline except
for basemap tiles. Delete a file in `raw/` to refresh that one source.

**If the page renders blank** with a console error about a missing default
export, Vite's transform cache is stale: `rm -rf node_modules/.vite` and
restart. It is not a code error.

---

## What it does

**Four sectors**, each a curated view over the same 40 census tracts:

| Sector | Covers |
|---|---|
| People & Households | Demographics, language, income, vulnerability |
| Local Economy | Businesses, building stock, permit activity |
| Mobility & Access | Transit dependency, stops and stations, pedestrian safety |
| Public Safety | Reported crime, pedestrian collisions |

**Fenton Village** — the district page. It straddles five tracts, so its
resident profile is estimated by areal interpolation and the page says so.

**Address search** — on every map. Type an address, get a pin, a walk radius
(400/800/1200 m), and what the warehouse knows about that spot.

**Trends** — yearly series with two-year projections, and an explicit refusal to
project where the data will not carry it.

**FAQ** — plain-language orientation, searchable, reached from a callout.

---

## Data sources

All public, all verified live. No key needed for any of them.

| Source | Agency | Rows | Coverage |
|---|---|---|---|
| Tract demographics (140 ACS-derived fields) | Montgomery County GIS | 40 tracts | vintage not published |
| Reported crime (NIBRS) | MCPD via dataMontgomery | 84,751 | Apr 2017 – Sep 2026 |
| Building permits | dataMontgomery | 23,367 | Jan 2000 – Sep 2026 |
| Ride On bus stops | dataMontgomery | 6,105 | snapshot |
| Pedestrian-struck dispatches | dataMontgomery | 574 | Apr 2017 – Sep 2026 |
| Parcels and SDAT assessments | Montgomery County GIS | 801 | built 1858–2025, assessed 2025 |
| Businesses, rail stations | OpenStreetMap (Overpass) | 873 / 8 | snapshot |
| Address geocoding | U.S. Census Bureau | — | live |

### Why no Census API key

The ACS API now rejects keyless requests — and returns **HTTP 200 with an HTML
"Missing Key" page**, so a status-code check will not catch it. Montgomery
County's own demographics MapServer exposes all 140 ACS-derived variables
without one, which is what this uses instead.

A key is still needed for **educational attainment** (ACS table B15003), which
the county layer does not carry. That gap is stated in the People sector and in
the FAQ. Get one free at <https://api.census.gov/data/key_signup.html>.

---

## Architecture

```
backend/
  probe.py       Discovery — what each upstream actually exposes
  sources.py     Fetchers, disk-cached to data/raw/
  build.py       ETL → DuckDB, spatial join of points into tracts
  sectors.py     Declarative sector definitions (metrics, notes, gaps)
  fenton.py      District profile by areal interpolation
  locate.py      Address geocoding + vicinity query
  trends.py      Yearly series, least-squares fit, projection
  coverage.py    Temporal reach per source
  api.py         FastAPI read-only query layer
  insights.py    CLI report of the same findings

src/
  Landing.jsx        Sector cards, district card, trends card
  SectorView.jsx     Choropleth + metric picker + overlays
  FentonView.jsx     The district application
  Trends.jsx         Charts and the expanded view
  Faq.jsx            Searchable plain-language FAQ
  AddressSearch.jsx  Search bar, pin, vicinity card
  Coverage.jsx       "How far back does this reach" lines
  theme.js           USF palette, shared with Leaflet
```

**Census tracts are the join key.** Demographics arrive per tract, so every
point source — businesses, permits, stops, collisions, crime — is counted into
its containing tract with `ST_Within`. That is what lets a People variable sit
beside a Mobility variable in one row and mean something.

**DuckDB with the spatial extension** is the store: in-process, no server to
provision, real spatial joins, and it ships as a single ~6 MB file.

**Sectors are declarative.** Adding a metric is a dict entry in `sectors.py`;
adding a sector is one block. The API, the map legend and the insight text all
read from the same definition, so a metric cannot reach the UI without the
source that produced it.

### API

```
GET /api/sectors              all sectors with headline figures
GET /api/sectors/{id}         one sector: metrics, per-tract rows, findings
GET /api/geo/tracts           tract polygons with every metric attached
GET /api/geo/points/{layer}   businesses | bus_stops | stations | ped_struck | crime
GET /api/fenton               district profile
GET /api/fenton/businesses    businesses inside the district
GET /api/locate?address=…     geocode + vicinity profile
GET /api/trends               yearly series and projections
GET /api/findings             cross-sector claims, each with its SQL
GET /api/coverage             temporal reach of every source
GET /api/sources              source registry
GET /api/health
```

---

## What the data says

Findings that need more than one source, each shipping the SQL that produced it.

- **Transit dependency tracks income, not lifestyle.** Car-free household share
  correlates with median household income at **r = −0.66**.
- **In four tracts, English is the second language spoken at home.** Spanish
  speakers outnumber English-only 3,396 to 1,412 in one of them.
- **14 tracts are simultaneously above-average car-free, below-average income,
  and have pedestrian collisions.** The worst is 30.9% limited-English, so
  safety outreach in English alone will not reach the most exposed.
- **Crime counts mostly measure commercial activity.** Incidents correlate with
  business count at **r = 0.89** and with population at only 0.31.
- **Reported crime is falling ~5.8% a year** (r² = 0.78), from 9,392 incidents
  in 2019 to 5,969 in 2025.

---

## Known gaps and caveats

Stated in the UI as well as here.

- **Education** is absent from the county demographics layer — needs a Census key.
- **Demographics have no published vintage.** The county does not say which ACS
  release the layer draws from and the service exposes none.
- **Median income is topcoded at $250,000.** Two tracts hit the ceiling, so the
  real spread at the top is wider than it looks.
- **Business data is OpenStreetMap**, volunteer-maintained. The brief describes
  Fenton Village as 240+ businesses; OSM maps 184 inside the district bounds.
  That is coverage, not closures.
- **Crime is reports to police, not crime.** Willingness to report varies, so a
  low count can mean a safe place or an under-reporting one.
- **Raw counts mislead.** Bus stops and pedestrian collisions correlate at 0.74
  raw but only 0.37 per capita. Compare rates.
- **No population trend.** The demographics layer is a single snapshot; a
  decennial count beside an ACS estimate is two points, not a series.
- **No crime, events, schools or parks beyond what is listed above.**
  Distances in the address search are spherical, not along the street network.

---

## Notes for whoever picks this up

- **`df95-9nn9` is a map asset, not a dataset.** It reports 508,651 crime rows
  and returns every one with no fields. The dataset behind it is `icn6-v9z3`,
  whose lat/lon are typed as text — filter on the `geolocation` point column.
  The same trap applies to `yszy-pfje` (collisions), which is why pedestrian
  data comes from `gptc-gfn8` instead.
- **The county's `F__*` demographic columns are fractions, not percentages.**
  Verified against `households_no_vehicle / households`. Converted at ETL time.
- **CARTO's dark basemaps stamp "API KEY REQUIRED" into the tile image** while
  still returning HTTP 200. Only a visual check catches it. This uses Esri's
  canvases, which swap with the light/dark theme.
- **The study bbox crosses into DC.** A wide TIGERweb query returns `11001*`
  tracts alongside Montgomery's `24031*`.
- The inherited `.gitignore` carries a Python-oriented `lib/` rule that will
  silently swallow any `src/lib/` you add. A negation is already in place at the
  bottom of the file; leave it there.

---

## Acknowledgments

Thanks to the organizers and sponsors who made Bay Hacks 2026 possible:
Tampa Bay Innovation, and sponsors Render, ElevenLabs, Nucleate, American
Circular, and User Experience University — along with on-campus partners
ACM, SHPE USF, and Google Developer Group on Campus at USF.

---

## Stack

React 18 · Leaflet · Vite · FastAPI · DuckDB (spatial) · Python 3.13
Theme follows the [USF brand palette](https://www.usf.edu/ucm/marketing/colors.aspx).
