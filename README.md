# EV Charging Station Site Suitability Analysis — California

**GIS Data Science Portfolio | Project 02**

A data-driven site suitability model identifying the highest-priority locations for new public EV charging stations in California. Integrates live federal station data, Census demographics, and spatial machine learning into a reproducible six-stage analytical pipeline across **2,007 ZIP Code Tabulation Areas (ZCTAs)**.

---

## Headline Results

| Metric | Value |
|--------|-------|
| Public EV stations analyzed | 18,957 |
| Total charging ports | 62,860 (45,279 L2 · 17,300 DC Fast · 281 L1) |
| California ZCTAs analyzed | 2,007 |
| ZCTAs with adequate coverage | 1,283 (63.9%) |
| **Underserved ZCTAs (gap zones)** | **343 (17.1%)** |
| **People in gap zones** | **2,573,537** |
| Total EV registrations (CA) | 1,761,634 |
| GWR Global R² | **0.9777** (vs. OLS 0.9100) |
| GWR Mean Local R² | **0.961** |
| Optimal GWR bandwidth | 52 nearest neighbours (AICc) |
| Moran's I on OLS residuals | 0.2275 (p = 0.001) — GWR justified |

---

## Interactive Map

The full interactive suitability map (MCE scores, gap zones, existing stations, top-20 candidates) is best viewed via nbviewer:

**[View Interactive Map & Full Notebook](https://nbviewer.org/github/Suvamp/ev-charging-suitability-ca/blob/main/EV_Station_Suitability_Analysis.ipynb)**

---

## Top 20 Candidate Sites

Sites ranked by MCE composite suitability score. All are confirmed gap zones (>8 km from nearest charger cluster) with meaningful population.

| Rank | ZCTA | Score | Population | EV Regs | Med. Income | Commuters | Curr. Stations | Gap (km) |
|------|------|-------|-----------|---------|-------------|-----------|---------------|----------|
| 1 | 93536 | 0.5086 | 73,417 | 3,303 | $89,987 | 29,627 | 10 | 8.9 |
| 2 | 93619 | 0.4591 | 48,320 | 2,934 | $121,444 | 21,063 | 0 | 3.5 |
| 3 | 93306 | 0.4512 | 74,518 | 2,267 | $60,857 | 28,511 | 3 | 3.9 |
| 4 | 93311 | 0.4417 | 48,722 | 2,471 | $101,447 | 21,992 | 11 | 5.2 |
| 5 | 93117 | 0.4406 | 54,915 | 2,140 | $77,964 | 26,925 | 34 | 1.5 |
| 6 | 93257 | 0.4374 | 78,754 | 1,906 | $48,411 | 30,031 | 11 | 5.8 |
| 7 | 93535 | 0.4324 | 79,522 | 2,050 | $51,560 | 26,979 | 15 | 3.7 |
| 8 | 95973 | 0.3855 | 38,490 | 1,544 | $80,249 | 19,010 | 2 | 5.1 |
| 9 | 92544 | 0.3844 | 52,364 | 1,515 | $57,881 | 19,669 | 2 | 5.8 |
| 10 | 93454 | 0.3780 | 41,324 | 1,511 | $73,166 | 17,741 | 39 | 0.4 |
| 11–20 | — | 0.376–0.349 | 26K–55K | 857–1,438 | $40K–$120K | 11K–21K | 1–21 | 0.2–6.0 |

*Full table exported to `outputs/top20_candidate_sites.csv`*

---

## Six-Stage Pipeline

```
AFDC API ──┐
Census ACS ─┼──► Data Acquisition ──► EDA ──► DBSCAN Gap Detection
OSM/TIGER ─┘                                        │
                                                     ▼
                                    MCE Scoring ◄── Gap Zones
                                         │
                                         ▼
                              Moran's I Diagnostic
                                         │
                                         ▼
                                   GWR Modeling
                                         │
                                         ▼
                          Interactive Map + Top-20 Report
```

### Stage 1 — Data Acquisition
- **AFDC API** (NREL): 18,957 open public EVSE locations with port-type breakdown
- **Census TIGER/Line 2022**: California ZCTA boundaries (2,007 units)
- **Census ACS 5-Year (2021)**: Population, median income, commute workers, housing units
- **CA DMV Open Data**: EV registration counts by ZIP code (1,761,634 total EVs)

### Stage 2 — Exploratory Data Analysis
- Port-type composition: Level 2 dominates at 72% of all ports
- Network operator breakdown: top 10 operators by station count
- Station density choropleth by ZCTA
- Stations vs. population density scatter — reveals the urban concentration pattern

### Stage 3 — DBSCAN Gap Detection
Using DBSCAN *inversely* — clustering existing stations to define coverage zones, then flagging ZCTAs outside those zones as gaps.

- `eps = 8,000 m` (8 km — typical EV range for short errands)
- `min_pts = 3`
- **149 clusters** identified · **177 isolated stations** (0.9%)
- Largest cluster: 7,475 stations (coastal LA/Orange County corridor)
- **343 underserved ZCTAs** identified with populations ≥ 500

### Stage 4 — Multi-Criteria Evaluation (MCE)
Eight-criterion weighted composite suitability score normalized to [0, 1]:

| Criterion | Weight | Rationale |
|-----------|--------|-----------|
| EV Registrations | 0.35 | Most direct signal of where EVs are |
| Population Density | 0.20 | Concentration of potential users |
| Commute Workers | 0.20 | Workplace charging demand proxy |
| Median Income | 0.15 | Purchasing power / adoption likelihood |
| Charger Gap Score | 0.10 | Inverse of existing coverage |

### Stage 5 — Moran's I Diagnostic
Confirmed that GWR is statistically justified before fitting:
- Global OLS R² = 0.9100
- Moran's I on OLS residuals = **0.2275 (p = 0.001)**
- Significant positive spatial autocorrelation → relationships vary geographically → GWR is warranted

### Stage 6 — Geographically Weighted Regression (GWR)
Local regression models fitted at each ZCTA centroid, revealing *where* and *how* demand drivers vary across California.

- **Bandwidth**: 52 nearest neighbours (selected via AICc golden-section search)
- **Kernel**: bisquare adaptive
- **Features**: population density, median income, commute workers, highway access
- **GWR R²**: 0.9777 · **Adjusted R²**: 0.9714
- **Local R² range**: 0.839 – 0.996 · **Mean**: 0.961
- Income coefficient strongest in coastal metros; commute workers dominant in inland CA

---

## Output Files

| File | Description |
|------|-------------|
| `outputs/01_stations_raw.png` | EV station map with CA boundary |
| `outputs/02_eda_panels.png` | 4-panel EDA chart |
| `outputs/03_gap_analysis.png` | DBSCAN gap zones map |
| `outputs/04_mce_suitability.png` | MCE suitability choropleth |
| `outputs/gwr_coefficient_maps.png` | 4-panel GWR local coefficient maps |
| `outputs/06_gwr_local_r2.png` | GWR local R² map |
| `outputs/06_ev_suitability_map.html` | Interactive Folium map (4 layers) |
| `outputs/top20_candidate_sites.csv` | Ranked candidate sites table |

---

## Data Sources

| Dataset | Source | Year | License |
|---------|--------|------|---------|
| EV Charging Stations | [NREL AFDC API](https://developer.nrel.gov/docs/transportation/alt-fuel-stations-v1/) | 2024 | Public |
| ZCTA Boundaries | [US Census TIGER/Line](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html) | 2022 | Public Domain |
| State Boundary | [US Census Cartographic Boundary](https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html) | 2022 | Public Domain |
| Demographics | [Census ACS 5-Year](https://www.census.gov/data/developers/data-sets/acs-5year.html) | 2021 | Public Domain |
| EV Registrations | [CA DMV Open Data](https://data.ca.gov/dataset/vehicle-fuel-type-count-by-zip-code) | 2024 | CC BY |

---

## Tech Stack

```
Python 3.11
├── geopandas       — spatial data handling and choropleth mapping
├── pandas / numpy  — data wrangling
├── scikit-learn    — DBSCAN clustering, MinMaxScaler
├── libpysal        — spatial weights (KNN)
├── esda            — Moran's I spatial autocorrelation test
├── mgwr            — Geographically Weighted Regression
├── folium          — interactive web map
├── matplotlib      — static charts and maps
├── osmnx           — highway proximity (optional)
├── requests        — API data retrieval
└── tqdm            — progress tracking
```

---

## Getting Started

**1. Clone and set up the environment**
```bash
git clone https://github.com/yourusername/ev-suitability-analysis.git
cd ev-suitability-analysis
conda env create -f environment.yml
conda activate ev_suitability
```

**2. Add your NREL API key**

Get a free key at [developer.nrel.gov/signup](https://developer.nrel.gov/signup/). In the config cell of the notebook:
```python
NREL_API_KEY = 'your_key_here'
```

**3. Run the notebook**
```bash
jupyter lab EV_Station_Suitability_Analysis.ipynb
```

Run cells top to bottom. Data is cached in `./outputs/` after first download — subsequent runs are fast.

---

## Limitations

- **OSM highway features** were excluded due to state-level query performance constraints; highway access proxied through cached distances where available
- **EV registration data** sourced from CA DMV Open Data; falls back to income-based proxy if the API is unavailable
- **ZCTAs** do not perfectly align with administrative boundaries; some gap zones near county borders may have coverage from adjacent ZCTAs not captured in this model
- **GWR bandwidth** of 52 neighbours produces highly local models; results in sparse rural areas should be interpreted cautiously due to small sample sizes within each local kernel

---

## License

MIT License — data sources retain their original licenses (see table above).
