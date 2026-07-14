# EV Charging Station Site Suitability Analysis (CA)

GIS data science portfolio project. Single notebook (`EV_Station_Suitability_Analysis.ipynb`)
runs a six-stage pipeline: AFDC/Census/OSM/DMV data acquisition → EDA → DBSCAN gap detection →
MCE suitability scoring → Moran's I diagnostic → GWR modeling → Folium map + top-20 report.

## Known issues from 2026-07-14 pre-push review

These were found during a full read-through (not just a lint pass) before the first GitHub push.
Some are fixed, some are open — check git log / diff against this list before assuming state.

### Still needs action (as of last review)
- **Leaked NREL API key in git history.** Commits `38a637b` and `d8210c8` contain a real key
  (`NREL_API_KEY = 'twIExn...'`) in the notebook source. Current HEAD has a placeholder, but the
  key is reachable in history and was already pushed to `github.com/Suvamp/ev-charging-suitability-ca`.
  Must: revoke/regenerate the key at developer.nrel.gov, then scrub history (`git filter-repo` /
  BFG) and force-push before treating this repo as public-safe. **Revoking the key is the actual
  fix; history-scrubbing is cleanup on top of that, not a substitute for it.**
- The "Interactive Dashboard" link in the README is a Google Drive link that shows a sign-in
  wall to anyone but the owner — needs to move to GitHub Pages (note: the Folium HTML is ~197MB,
  over GitHub's 100MB hard file limit, so it can't just be committed as-is — needs simplification
  or a different static host) or a properly public share link.
- **No LICENSE file** despite README claiming MIT License.

### Fixed (2026-07-14)
- **`median_income` sentinel bug** — fixed in `download_census_acs()`: ACS negative sentinel
  codes (e.g. `-666666666`) are now masked to `NaN` right after the `pd.to_numeric` conversion,
  before the `pop_total > 0` filter. Stale cached `outputs/ca_census_acs.csv` (which had the bad
  values baked in, confirmed 158/1762 rows) was deleted so the next run re-downloads clean data —
  re-run the notebook top to bottom before trusting `median_income`/MCE output again.
- **README/notebook mismatches** — README now matches the actual 8-criteria `MCE_WEIGHTS`
  (was showing a stale 5-criteria version), documents that `grid_capacity` duplicates
  `highway_access` (not an independent signal), documents that `retail_poi_density` and
  `charger_gap_score` are dropped from GWR (near-zero variance) while still counted in the MCE
  score, and the Limitations section no longer falsely claims OSM highway data was excluded.
  Stale narrative text left in the notebook's own markdown cells (e.g. "12 million Californians",
  "Moran's I of 0.38" — both contradicted by the notebook's own printed output, 2,573,537 and
  0.2275 respectively) was **not** touched — only `README.md` was edited. If polishing the
  notebook prose itself, watch for this pattern.

### Minor / lower priority
- Duplicate GWR coefficient-map plotting code (sections "7.2" and "7.3" build near-identical
  figures); `outputs/gwr_coefficient_maps.png` is the orphaned output of the first one and isn't
  referenced anywhere in the README — safe to delete once the duplicate code block is removed.
- `download_osm_features()` cache-hit print is missing an `f` prefix (`print('Loaded {len(...)}...')`),
  confirmed printing the literal `{}` in actual output — cosmetic but real.
- `download_census_acs(state_fips, ...)` and `download_ca_zctas(state_fips, ...)` both accept
  `state_fips` but don't actually use it for filtering (one uses a hardcoded ZIP numeric range,
  the other a lat/lon bbox) — misleading signatures.
- No post-merge validation (null/match-rate checks) after the ACS, EV-registration, or OSM-POI
  joins — non-matches silently `fillna(0)`, indistinguishable from genuine zeros.
- `contextily` is in `environment.yml` but never imported/used.
- `RANDOM_SEED = 42` is defined in config but unused; the one seeded operation
  (`stations_wgs.sample(...)`) hardcodes `42` directly instead of referencing it.

### Confirmed solid (don't re-litigate)
- `.gitignore` was present from the very first commit and correctly excludes all large geodata,
  the 197MB Folium HTML map, and the 207MB `docs/index.html` — nothing large is tracked.
- CRS handling is consistent: WGS84 → `EPSG:3310` (CA Albers) reprojection happens before any
  distance/area/DBSCAN calculation, except one harmless case (`download_ca_zctas`'s bbox filter
  runs `.centroid` before reprojecting — triggers a GeoPandas CRS warning but doesn't affect
  correctness for a coarse CA bbox).
- The headline numbers actually printed by the code and used in the README (Moran's I 0.2275,
  GWR global R² 0.9777, optimal bandwidth 52, gap population 2,573,537) are internally consistent
  and accurate — the pipeline's real results are fine. It's specifically the free-text narrative
  cells and the Limitations/MCE sections of the README that had drifted from the code.

## Repo/remote
- Origin: `https://github.com/Suvamp/ev-charging-suitability-ca.git`
- Local `cache/` (~294MB, 525 files) is gitignored working-directory API response cache — not a
  tracking concern, just don't `git add -A` blindly.
