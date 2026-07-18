# IDCUP 26 — International Infectious Disease Importation Risk Dashboard

**FIFA World Cup 2026 · Northeastern University / EPISTORM x Insight Net**

A single-file interactive dashboard that estimates pathogen-specific importation risk to 11 US host cities during the 2026 FIFA World Cup, using the GLEAM-EPIRisk mobility framework and OAG airline data.

Live deployment: GitHub Pages (`index.html` is a copy of the latest stable version).

---

## Repository structure

```
dashboard/
├── IDCUP26_dashboard_v22_stable.html   # Current stable dashboard (v22, ~3.5 MB, self-contained)
├── index.html                          # GitHub Pages entry point (copy of v22)
├── IDCUP26_landing.html                # Static landing page (InsightNet branding)
├── index-insightnet.html               # Alternate landing page
├── .nojekyll                           # Disables Jekyll processing on GitHub Pages
├── insightnet-watermark.png            # Subtle floral watermark behind header
├── flower-pattern-2.png                # Unused alternate pattern asset
├── round_of_32.csv                     # R32 bracket reference (country, group, slot)
│
├── New_data_final/                     # XLSX source data — FINAL (full tournament)
│   └── IDCUP26_<ISO3>.xlsx × 29       #   One file per advancing country
├── New_data_Round_of_32_v2/            # XLSX source data — R32 phase (v2 revision)
├── New_data_round32/                   # XLSX source data — R32 phase (v1, superseded)
├── Data_Round_of_16/                   # XLSX source data — R16 phase
├── data/                               # Logo images (EPISTORM, InsightNet PNGs)
│
├── IDCUP26_dashboard_v21_stable.html   # Prior version: R16/QF phase
├── IDCUP26_dashboard_v20_stable.html   # Prior version: R32 v2 + eliminated countries
├── IDCUP26_dashboard_v19_stable.html   # Prior version: R32 phase
├── IDCUP26_dashboard_v18_stable.html   # Prior version: InsightNet visual refresh
└── IDCUP26_dashboard_v17_stable.html   # Initial committed version
```

### Version history

| Version | Snapshot date | Phase | Key changes |
|---------|--------------|-------|-------------|
| v17 | 2026-05-15 | Group stage | Initial commit, 48 countries |
| v18 | 2026-05-18 | Group stage | InsightNet visual refresh, brief generator restyled |
| v19 | 2026-06-29 | Round of 32 | R32 data from `New_data_round32/`, match calendar, "played" markers |
| v20 | 2026-07-05 | Round of 32 | Revised R32 data from `New_data_Round_of_32_v2/`, restored 13 eliminated countries |
| v21 | 2026-07-06 | Round of 16 | R16 data from `Data_Round_of_16/`, QF match names filled in |
| v22 | 2026-07-16 | Final | Full-tournament data from `New_data_final/`, all matches through Final filled in |

---

## Architecture overview

The dashboard is a **single self-contained HTML file** (~4,300 lines). All CSS, JavaScript, and data are inlined — no build step, no bundler, no framework. External dependencies are loaded from CDNs:

- **D3.js v7** — maps, bubble charts, data manipulation
- **TopoJSON Client v3** — world and US state boundary rendering
- **Google Fonts** — Lato, IBM Plex Sans, IBM Plex Mono

### High-level structure of the HTML file

| Line range (approx.) | Content |
|-----------------------|---------|
| 1–12 | `<head>`: meta, fonts, CDN scripts |
| 12–600 | `<style>`: all CSS (variables, layout, tables, maps, briefs) |
| 600–1595 | HTML body: header, overview section, pathogen view, host-city view, methodology/about section |
| 1596–1601 | **Data blobs** (JSON): `COUNTRY_DATA`, `CITY_DATA`, `HOST_CITY_VIEW`, `HOST_CITY_META`, `HOST_CITY_MATCHES`, `CITY_COORDS` |
| 1670–1710 | Configuration constants: `PATHOGEN_ORDER`, `PATHOGEN_GROUPS`, `WHO_SOURCES`, `PANEL_NOTES` |
| 1783–2270 | Brief generators: CSV export, `buildPathogenBriefHTML()` |
| 2270–2400 | Brief generators: report map renderer, `buildCityBriefHTML()` |
| 2822–3300 | Logo data URLs, `REPORT_CSS` (brief print stylesheet) |
| 3300–4355 | Core application logic: tab switching, pathogen panels, D3 map rendering, city view, match calendar |

---

## Data structures

All data lives as inline `const` declarations in the `<script>` block. There are six primary data objects.

### 1. `COUNTRY_DATA`

**Structure:** `pathogen → ISO3 → metrics`

The primary country-level importation data. One entry per pathogen per country.

```javascript
COUNTRY_DATA = {
  "Dengue": {
    "BRA": {
      "active_cases": 29606,         // Monthly active case count
      "baseline_e_imports": 5.0852,  // E[K] under baseline OAG flows
      "p_zero": 0.0062,             // P(K=0) — probability of zero imports/month
      "p_geq1": 0.9938,             // P(K≥1) — probability of ≥1 import/month
      "e_imports_10": 5.5935,        // E[K] under +10% WC scenario
      "e_imports_20": 6.1019,        // E[K] under +20% WC scenario
      "e_imports_35": 6.8644,        // E[K] under +35% WC scenario
      "excess_10": 0.5083,           // e_imports_10 − baseline_e_imports
      "excess_20": 1.0167,           // e_imports_20 − baseline_e_imports
      "excess_35": 1.7792,           // e_imports_35 − baseline_e_imports
      "scenario_type": "current"     // "current" (surveillance) or "hypothetical" (planning)
    },
    // ... more countries
  },
  // ... more pathogens
}
```

**Pathogens (12):** Dengue, Chikungunya, Yellow Fever, Measles, Pertussis, Mumps, Rubella, Mpox Clade I, Ebola, Marburg, Cholera, Typhoid.

**Countries:** 29 advancing + 13 eliminated group-stage countries = 42 total.

### 2. `CITY_DATA`

**Structure:** `ISO3 → pathogen → scenario → [{city, flag, rr}]`

City-level relative risk vectors for each country/pathogen combination across all four scenarios.

```javascript
CITY_DATA = {
  "BRA": {
    "Dengue": {
      "baseline": [
        { "city": "New York/New Jersey", "flag": "N", "rr": 0.234567 },
        { "city": "Miami",              "flag": "Y", "rr": 0.198234 },
        // ... ~400 indexed US cities, sorted by RR descending
      ],
      "+10%": [ /* same structure */ ],
      "+20%": [ /* same structure */ ],
      "+35%": [ /* same structure */ ]
    },
    // ... more pathogens
  },
  // ... more countries
}
```

- **`flag`**: `"Y"` = host city where this country's team plays; `"N"` = other indexed city
- **`rr`**: relative risk = conditional probability that an imported case from this country arrives in this city
- Cities are sorted descending by `rr` within each scenario
- The city list covers ~400 US cities with international air connectivity

### 3. `HOST_CITY_VIEW`

**Structure:** `city → [{country, pathogen, flag, baseline: {rr, e_country, local_imports, local_excess}, "+10%": {...}, ...}]`

Pre-aggregated importation flows for each of the 11 host cities. This powers the "By host city" tab.

```javascript
HOST_CITY_VIEW = {
  "Miami": [
    {
      "country": "BRA",
      "pathogen": "Dengue",
      "flag": "Y",
      "baseline": {
        "rr": 0.198234,           // Relative risk for BRA→Miami
        "e_country": 5.0852,      // Country-level E[K] (same as COUNTRY_DATA)
        "local_imports": 1.008,   // rr × e_country
        "local_excess": 0         // local_imports − baseline_local_imports (0 for baseline)
      },
      "+10%": {
        "rr": 0.201,
        "e_country": 5.5935,
        "local_imports": 1.124,
        "local_excess": 0.116     // Difference vs baseline
      },
      // "+20%", "+35%" ...
    },
    // ... sorted by baseline local_imports descending
  ],
  // ... 11 cities total
}
```

### 4. `HOST_CITY_META`

**Structure:** `city → {state, stadium, capacity, risk_tag}`

Static metadata for the 11 US host cities.

```javascript
HOST_CITY_META = {
  "Miami": {
    "state": "FL",
    "stadium": "Hard Rock Stadium",
    "capacity": 64767,
    "risk_tag": "Aedes aegypti vector presence; subtropical climate increases arbovirus seeding risk."
  },
  "San Francisco Bay Area": {  // NOTE: key is "San Francisco Bay Area", not "San Francisco"
    "state": "CA",
    "stadium": "Levi's Stadium",
    "capacity": 68500,
    "risk_tag": ""
  },
  // ... 11 cities
}
```

**Important:** The city name `"San Francisco Bay Area"` must match across all data structures. The XLSX files use `"San Francisco"` and must be mapped via `CITY_NAME_MAP` during the build.

### 5. `HOST_CITY_MATCHES`

**Structure:** `city → [{match_number, date, time_local, stage, round, home_team, away_team}]`

Full match calendar for each host city's venue.

```javascript
HOST_CITY_MATCHES = {
  "Miami": [
    {
      "match_number": 13,
      "date": "2026-06-15",
      "time_local": "6:00 p.m.",
      "stage": "Group stage",
      "round": "NA",
      "home_team": "Saudi Arabia",
      "away_team": "Uruguay"
    },
    // ... more matches, including knockout rounds
    {
      "match_number": 103,
      "date": "2026-07-18",
      "time_local": "5:00 p.m.",
      "stage": "Knockout stage",
      "round": "Match for third place",
      "home_team": "France",
      "away_team": "England"
    }
  ],
  // ... 11 cities
}
```

Matches with `date < SNAPSHOT_DATE` are rendered with a "Played" tag in the UI.

### 6. `CITY_COORDS`

**Structure:** `city_name → [longitude, latitude]`

Geographic coordinates for ~400+ US cities referenced in `CITY_DATA`. Used by D3 to position bubbles on the map.

```javascript
CITY_COORDS = {
  "New York/New Jersey": [-74.174, 40.774],
  "Miami": [-80.191, 25.762],
  "San Francisco Bay Area": [-122.016, 37.385],
  // ... ~400 cities
}
```

---

## Configuration constants

### `SNAPSHOT_DATE`

```javascript
const SNAPSHOT_DATE = '2026-07-16';
```

Defined on line ~4089. Used to determine which matches show the "Played" tag (`m.date < SNAPSHOT_DATE`, strict less-than). Also displayed in the masthead and brief footers.

### `PATHOGEN_GROUPS`

Four display groups for sidebar navigation:

| Group | Pathogens |
|-------|-----------|
| Arboviruses | Dengue, Chikungunya, Yellow Fever |
| Vaccine-preventable | Measles, Pertussis, Mumps, Rubella |
| High consequence | Mpox Clade I, Ebola, Marburg |
| Enteric / waterborne | Cholera, Typhoid |

### Hypothetical vs. surveillance-derived

- **Hypothetical** (`scenario_type: "hypothetical"`): Marburg. Case counts are set manually for planning; no confirmed active outbreaks. Displayed with a visual band and disclaimer.
- **Ebola**: Anchored to real Bundibugyo ebolavirus activity in DR Congo (315 cases in 30-day window at snapshot). Has special explanatory text and a cross-link to the [Epistorm EBV 2026 dashboard](https://epistorm.github.io/EBV2026/).
- All other pathogens are surveillance-derived with WHO/ECDC/PAHO data sources listed in `WHO_SOURCES`.

---

## XLSX data format

Each XLSX file (e.g., `IDCUP26_BRA.xlsx`) has 5 sheets:

| Sheet | Purpose |
|-------|---------|
| README | Metadata |
| baseline | Country + city data under baseline OAG flows |
| +10% | Country + city data under +10% WC-concentrated scenario |
| +20% | Country + city data under +20% WC-concentrated scenario |
| +35% | Country + city data under +35% WC-concentrated scenario |

### Sheet layout (baseline, +10%, +20%, +35%)

- **Row 3, Column B**: ISO3 country code (e.g., `BRA`, `GBR`)
- **Row 11**: Column headers for country-level data
- **Rows 12+**: Country-level data rows (one per pathogen), until a blank row

| Column | Content |
|--------|---------|
| A | Pathogen name |
| B | Active (Y/N) |
| C | Active cases |
| E | E[imports] |
| F | P(K=0) |
| G | P(K≥1) |

- **After the blank row**: City-level section
  - A header row with columns: `Pathogen`, `US City`, `Excess routed here?`, `Relative risk`
  - Pathogen separator rows: `--- Dengue ---`
  - Section headers: `WC26 host cities`, `Other indexed US cities`
  - City data rows: pathogen name (col A), city name (col B), flag Y/N (col C), relative risk (col D)

### Key mapping rules for XLSX parsing

These mappings must be applied when building JSON from XLSX:

```python
# ISO code remapping (England uses ENG in dashboard, GBR in XLSX)
ISO_MAP = {'GBR': 'ENG'}

# Pathogen name normalization
PATHOGEN_MAP = {
    'Yellow fever': 'Yellow Fever',
    'Mpox Ib': 'Mpox Clade I',
    'Marburg (50 cases)': 'Marburg'
}

# Skip this pathogen entirely (alternate scenario, not shown)
SKIP_PATHOGENS = {'Marburg (100 cases)'}

# City name normalization
CITY_NAME_MAP = {'San Francisco': 'San Francisco Bay Area'}

# Hypothetical pathogens (displayed with planning disclaimer)
HYPO_SET = {'Marburg'}
```

### Eliminated countries

13 countries eliminated in the group stage are **not** in the XLSX data folders for later rounds but are preserved in the dashboard by copying their data from the previous version:

```python
ELIMINATED_13 = [
    'CUW', 'HTI', 'IRQ', 'JOR', 'NZL', 'PAN',
    'QAT', 'SAU', 'SCT', 'TUN', 'TUR', 'URY', 'UZB'
]
```

Their data remains frozen at the last snapshot where they were actively modeled.

---

## Data build pipeline

Updating the dashboard with new XLSX data follows this pipeline:

### Step 1: Parse XLSX files → JSON blobs

A Python build script reads all XLSX files and produces three JSON blobs:

1. **`COUNTRY_DATA.json`** — aggregated from the country-level rows across all 4 scenario sheets
2. **`CITY_DATA.json`** — aggregated from the city-level sections across all 4 scenario sheets
3. **`HOST_CITY_VIEW.json`** — derived by cross-referencing COUNTRY_DATA and CITY_DATA for the 11 host cities

The build script also restores the 13 eliminated countries by extracting their data from the previous version's HTML.

**Dependencies:** `openpyxl` (for XLSX parsing), Python 3.

**Reference implementation:** The build scripts used for v22 were stored in the scratchpad during the build session. Key logic:
- Parse each XLSX: extract country-level metrics from rows 12+ and city-level data from the section after the blank row
- Apply all mapping rules (ISO, pathogen names, city names, skip list)
- Build the three JSON structures
- Restore eliminated countries from the previous HTML version

### Step 2: Inject JSON blobs into HTML

A separate injection script:

1. Copies the previous stable HTML as a starting point
2. Replaces the three JSON blobs (`COUNTRY_DATA`, `CITY_DATA`, `HOST_CITY_VIEW`) using regex substitution
3. Updates `SNAPSHOT_DATE`
4. Updates match names in `HOST_CITY_MATCHES` for any newly determined knockout matches
5. Updates methodology text to reflect the current tournament phase
6. Writes both the versioned file (`IDCUP26_dashboard_vNN_stable.html`) and `index.html`

**Critical regex patterns:**
```python
# COUNTRY_DATA sits between its declaration and CITY_DATA
r'(const COUNTRY_DATA\s*=\s*)\{.*?\}(\s*;\s*const CITY_DATA)'

# CITY_DATA sits between its declaration and HOST_CITY_VIEW
r'(const CITY_DATA\s*=\s*)\{.*?\}(\s*;\s*const HOST_CITY_VIEW)'

# HOST_CITY_VIEW sits between its declaration and HOST_CITY_META
# WARNING: Do NOT use HOST_CITY_MATCHES as the end boundary — HOST_CITY_META sits between them
r'(const HOST_CITY_VIEW\s*=\s*)\{.*?\}(\s*;\s*const HOST_CITY_META)'
```

> **Known pitfall:** In v22, the regex for HOST_CITY_VIEW accidentally consumed HOST_CITY_META because the end boundary was set to `HOST_CITY_MATCHES` instead of `HOST_CITY_META`. This deleted all host city metadata and broke the entire Host City view. The correct boundary is `HOST_CITY_META`.

### Step 3: Manual updates

After blob injection, some text updates require manual attention:

- **Snapshot date** in display text (masthead, methodology section, brief footers)
- **Phase description** in methodology text (e.g., "Round of 32 phase" → "final phase")
- **Ebola case count** if surveillance data has changed
- **Match team names** for newly played knockout matches
- **"Played" tag threshold** via `SNAPSHOT_DATE` constant

### Step 4: Verify

1. Open the HTML in a browser (e.g., `python3 -m http.server`)
2. Check for JavaScript console errors
3. Verify all three views load: Overview, By Pathogen, By Host City
4. Generate at least one pathogen brief and one city brief — check for stale dates, stale phase references, correct data
5. Verify match calendar shows correct "Played" tags

---

## Brief generation system

The dashboard can generate print-ready PDF briefs (via the browser's print dialog) for both pathogens and host cities.

### Pathogen brief (`buildPathogenBriefHTML`, line ~1890)

A 4-page report generated as a standalone HTML document opened in a new tab:

| Page | Content |
|------|---------|
| 1 (Cover) | Executive summary, stat grid (countries, max imports, top P≥1), hypothetical/surveillance badge |
| 2 (Section 1) | Country table: all source countries with baseline imports, excess at +10/+20/+35%, P(zero), P(≥1) |
| 3 (Section 2) | Twin D3 maps (baseline vs +20%) + top-20 city comparison table |
| 4 (Section 3) | Methodology: data sources, travel model, excess scenarios, outputs, aggregate RR, interpretation |

### City brief (`buildCityBriefHTML`, line ~2392)

A 4-page report for a specific host city:

| Page | Content |
|------|---------|
| 1 (Cover) | Executive summary, risk tags, stadium/capacity, stat grid |
| 2 (Section 1) | Pathogen importation table: cumulative imports by pathogen across all source countries |
| 3 (Section 2) | Match calendar at this venue, with source-country teams bolded |
| 4 (Section 3) | Methodology: travel model, city-level flow, aggregation, thresholds, interpretation |

### Brief styling

Both briefs use `REPORT_CSS` (line ~2835), a dedicated print stylesheet that produces clean letter-size pages. Logo data URLs for EPISTORM and InsightNet are extracted from the page's existing `<img>` elements and embedded as base64 in the brief.

---

## UI views

### 1. Overview (landing page)

Static introductory text explaining what the dashboard does and doesn't do. Key statistics (net-new visitors, teams, host cities). Entry cards linking to the two data views.

### 2. By Pathogen

Sidebar with 12 pathogens grouped into 4 categories. Each pathogen panel has two sub-tabs:

- **Table view**: Country-level data table with sortable columns. Includes active cases, baseline E[imports], excess at each scenario, P(zero), P(≥1). Hypothetical pathogens show a yellow banner.
- **Map view**: D3 map of the US with bubble overlay. Bubble area proportional to relative risk. Host cities (flag=Y) in accent color, others in gray. Click a bubble for city detail panel. Scenario selector to switch between baseline/+10%/+20%/+35%.

Each panel also has:
- **Download CSV** button (exports country table as CSV)
- **Download Brief** button (generates pathogen brief in new tab)

### 3. By Host City

Sidebar listing 11 host cities. Each city panel shows:

- **Header stats**: stadium, capacity, risk tags, match count
- **Route table**: all pathogen/country importation flows routed to this city, grouped by pathogen
- **Match calendar**: all matches at this venue with "Played" tags, source-country teams bolded
- **Download Brief** button

### 4. Methods & Sources

Static methodology section covering:
1. Overview and motivation
2. Data sources and active case computation (with links to WHO/ECDC dashboards)
3. Travel model (GLEAM-EPIRisk + OAG)
4. Excess traffic distribution methodology
5. Interpretation caveats

---

## Deployment

The dashboard is deployed via **GitHub Pages** from the `main` branch.

- `index.html` is a copy of the current stable version
- `.nojekyll` disables Jekyll processing (required because filenames contain underscores)
- `insightnet-watermark.png` must be in the repo root alongside `index.html`
- Logo images in `data/` are used by the landing page but not by the dashboard itself (logos are base64-encoded inline)

To deploy a new version:
1. Update `IDCUP26_dashboard_vNN_stable.html`
2. Copy it to `index.html`
3. Commit and push both files

---

## Common update procedures

### Adding new simulation data (new XLSX folder)

1. Place the new XLSX files in a subfolder (e.g., `New_data_semifinal/`)
2. Run or adapt the build script to parse all XLSX → 3 JSON blobs
3. Copy the previous stable HTML to a new versioned file
4. Inject the new JSON blobs (watch the regex boundaries!)
5. Update `SNAPSHOT_DATE` (line ~4089 and display text)
6. Update `HOST_CITY_MATCHES` with any new knockout match team names
7. Update methodology text to reflect the current tournament phase
8. Copy to `index.html`
9. Verify in browser (all views, both brief types, console errors)

### Updating match results

In `HOST_CITY_MATCHES`, find the match by `match_number` and update `home_team` and `away_team`. The match data is a single JSON blob on line ~1600.

### Changing the snapshot date

Three locations need updating:
1. `const SNAPSHOT_DATE = '...'` on line ~4089 (controls "Played" tag logic)
2. Display text in the masthead and methodology sections (search for the old date string)
3. Brief footers and methodology (both `buildPathogenBriefHTML` and `buildCityBriefHTML`)

**Warning:** Do not do a blanket find-and-replace of the date string — match dates in `HOST_CITY_MATCHES` may share the same date value and would be corrupted.

### Adding a new pathogen

1. Add the pathogen data to `COUNTRY_DATA` and `CITY_DATA` (via XLSX or manually)
2. Add to `PATHOGEN_ORDER` array
3. Add to the appropriate group in `PATHOGEN_GROUPS`
4. Add surveillance source URL and name to `WHO_SOURCES` / `WHO_SOURCE_NAMES` (if not hypothetical)
5. Add a panel note to `PANEL_NOTES`
6. If hypothetical, add to the `HYPO_SET` equivalent in the brief generation logic

---

## Known pitfalls and lessons learned

1. **HOST_CITY_VIEW ↔ HOST_CITY_META boundary**: When replacing the HOST_CITY_VIEW JSON blob, the regex end boundary must be `HOST_CITY_META`, not `HOST_CITY_MATCHES`. HOST_CITY_META sits between them and will be consumed by a greedy regex if the wrong boundary is used.

2. **GBR → ENG mapping**: England's FIFA code is `ENG` but XLSX files use ISO3 `GBR`. The build script must apply `ISO_MAP = {'GBR': 'ENG'}`.

3. **"San Francisco" → "San Francisco Bay Area"**: XLSX files use `"San Francisco"` but all dashboard data structures use `"San Francisco Bay Area"`. Apply `CITY_NAME_MAP` during parsing.

4. **Blanket date replacement danger**: Never do `html.replace('2026-07-05', new_date)` — this will corrupt match dates in HOST_CITY_MATCHES that happen to fall on that date. Update the snapshot date surgically.

5. **Eliminated countries must be preserved**: When new XLSX data covers only advancing countries, the 13 eliminated group-stage countries must be copied from the previous version's data blobs, or they'll disappear from the dashboard.

6. **Brief text audits**: After any data or phase update, audit both `buildPathogenBriefHTML` and `buildCityBriefHTML` for stale references to previous phases ("group-stage matches", old dates, "not yet included" disclaimers).

7. **SNAPSHOT_DATE for "Played" tags**: The comparison is `m.date < SNAPSHOT_DATE` (strict less-than). Matches on the snapshot date itself are NOT marked as played.

---

## Technical notes

- **File size**: The HTML file is ~3.5 MB, mostly due to the inline JSON data blobs (~2.5 MB for CITY_DATA alone, which contains ~400 cities × 29 countries × 12 pathogens × 4 scenarios).
- **No build system**: Everything is manual — parse XLSX with Python, inject JSON with regex, edit text by hand. There is no `package.json`, no bundler, no CI.
- **Browser compatibility**: Tested in Chrome and Safari. Uses CSS custom properties, `mask-image`, D3 v7.
- **Print briefs**: Generated as standalone HTML documents. Users print to PDF via the browser's print dialog. The brief CSS uses `@page` rules for letter-size formatting.
- **Map data**: World boundaries and US state boundaries are loaded from CDN TopoJSON files at runtime (not embedded).

---

## Funding acknowledgement

This project is supported by cooperative agreement CDC-RFA-FT-23-0069 from the CDC's Center for Forecasting and Outbreak Analytics. The findings and conclusions are those of the authors and do not necessarily represent the official position of the funding agencies.
