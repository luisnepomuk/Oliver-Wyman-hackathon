ROLE
You are a senior data scientist and Python engineer. Build a demo "Decision Platform" for the
Oliver Wyman STADS Hackathon: find the best pilot locations ("white spots") in Germany for a
new urban premium/specialty food concept (Mediterranean/Italian-inspired food market / deli).
Deliver a working, interactive Streamlit app with a map. Clarity beats complexity.

BUSINESS QUESTION
In which cities, districts and micro-locations are the most attractive white spots?
The platform must (1) identify white spots, (2) compare locations, (3) explain drivers.

ICP (assumption, make it configurable in config.yaml)
Age 28-55, above-average income, small households (1-2 persons), urban, walking/transit
oriented, high dining-out and quality affinity. Proxies: age shares, household size, rent level,
density, amenity mix (cafes, restaurants, culture, universities, offices), transit stops.

APPROACH B (multi-stage ranking with ICP-lookalike)
Stage 1 CITY: rank the ~40-80 largest German cities (config: min population).
Stage 2 DISTRICT: 500m-1km grid cells (EPSG:3035) within top-N cities (default N=10).
Stage 3 MICRO: 100m cells (or point candidates) within top-K district cells per city.
Core idea:
 - Positive examples = existing premium food retailers from OSM (shop=deli, shop=cheese,
   shop=wine, shop=butcher, shop=greengrocer, shop=organic, shop=pastry, shop=confectionery,
   amenity=marketplace, plus names matching a premium/Italian keyword list e.g. Eataly, Alnatura,
   Feinkost, Delikatessen, Enoteca, Gastronomia).
 - Train a model (logistic regression baseline + LightGBM) on cell-level features to predict
   "premium food presence" -> LOOKALIKE SCORE = how well a cell's profile matches
   successful premium locations.
 - SATURATION = weighted competitor density in the catchment (premium competitors + big
   supermarkets / discounters separately).
 - WHITE SPOT SCORE = f(high lookalike, low saturation). Default: rank-normalised
   lookalike * (1 - saturation)^alpha, alpha configurable. Cannibalisation check: distance
   to nearest premium competitor and own-concept candidate spacing.
 - IMPORTANT: when computing features for a cell, EXCLUDE the premium POIs themselves
   (and the target-defining tags) from the features to avoid label leakage.
 - Validation: leave-one-city-out spatial CV (AUC / PR-AUC), report calibration and
   feature stability. Sensitivity analysis on score weights and alpha (rank stability).
 - Explainability: SHAP values per cell, shown in the compare view.
 - Be honest about limitations: presence of premium shops is a proxy, not revenue;
   survivorship bias; no footfall data; OSM completeness varies.

DATA SOURCES (all open)
 - Zensus 2022 grid CSVs (10km/1km/100m): population, age classes, household size, rent,
   ownership, vacancy. Download from the Destatis Zensus 2022 publications page.
   INSPECT the real column names/encodings first, do not assume them.
 - OpenStreetMap via Overpass: POIs (food shops, restaurants/cafes, culture, education,
   offices, transit stops, supermarkets/discounters), city boundaries.
 - Destatis Regionaldatenbank / Regionalatlas for city-level indicators (optional).
 - City list with population: derive from Zensus or a documented static CSV.

OVERPASS RULES (strict)
 - Public Overpass is shared infrastructure. Bundle all POI types into few queries per city
   bounding box (one query, many tags). Cache every response to data/raw/osm/<city>_<hash>.json
   and reuse it if the query is unchanged. Exponential backoff on 429/504, sleep between
   requests, never parallelise. Provide --offline mode that uses only cache.

ENGINEERING RULES
 - Python 3.11, geopandas, shapely, pyproj, pandas, numpy, scikit-learn, lightgbm, shap,
   streamlit, pydeck or folium (streamlit-folium), pyarrow. Pin versions in requirements.txt.
 - Project layout: config.yaml, src/{data,features,models,scoring,app}/, data/{raw,interim,
   processed}/, tests/, notebooks/, README.md. Cache outputs as Parquet/GeoParquet.
 - Work in a projected CRS (EPSG:3035) for geometry; WGS84 only for display.
 - Every step is a CLI script (python -m src.xxx) and idempotent; the app only reads processed
   files and must start in <10s. Provide a tiny sample dataset mode for fallback demos.
 - Log assumptions and row counts. Add basic tests (shapes, no NaN in final scores, no leakage).
 - Do not invent data. If a source fails, say so and fall back to cached data, documenting it.
 - After each step, run it, show a summary, and fix errors before continuing.

DEFINITION OF DONE
 Streamlit app with: city ranking view, city drilldown map with district heatmap, micro-location
 markers, compare view (2-4 locations side by side), SHAP driver bars, sliders for ICP/weights,
 a methodology + limitations page. README with run instructions.