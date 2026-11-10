# shubham--demo
This is my first Git Repository
**# AGNINETRA**



**KURUKSHETRA · College-Level Hackathon Project**

**AI-Based Detection and Classification of Industrial Fires and Persistent Thermal Sources Using NASA FIRMS, OSM & Satellite Data**



NASA FIRMS tells you *\*where\** the ground is hot. AGNINETRA tells you *\*what\** is hot and \*whether it

should be\*: it classifies every VIIRS/MODIS thermal anomaly as an industrial fire / process heat, gas

flare, mining or coal-seam fire, agricultural residue burning, wildfire, or other; it tracks

**\*\*persistent thermal sources\*\*** over time; and it flags **\*\*FRP anomalies\*\*** at those sources — the

signature of an industrial fire or explosion rather than routine operation. Everything is served as

GeoJSON/CSV through a REST API and shown on an interactive GIS dashboard. It then reduces the whole

archive to a **\*\*ranked alert feed\*\*** an operator can actually work through, forecasts \*\*which sites and

which areas are likely to burn in the next seven days\*\*, and takes the analyst's verdict back as

training data.



**\*\*Live demo (no install): https\://rahulskandagal.github.io/thermal-sentinel/\*\***



\`\`\`

NASA FIRMS (VIIRS 375 m / MODIS 1 km) ─┐

OpenStreetMap / Overpass ──────────────┤   ingest → spatial clustering (DBSCAN) → temporal persistence

  industrial=\*, man_made=flare|works,  ├─► → OSM context (nearest facility, land cover) → hybrid

  power=plant, landuse=quarry|farmland │     classifier (explainable rules + gradient-boosted trees)

Sentinel-2 / Worldview deep links ─────┘   → SQLite/GeoJSON → FastAPI → React + Leaflet dashboard

\`\`\`



**## Quick start (Windows, \~2 minutes)**



\`\`\`powershell

.\run.ps1

\`\`\`



That creates a Python venv, installs dependencies, starts the API on http\://127.0.0.1:8000 (docs at

\`/docs\`) and the dashboard on http\://localhost:5173. On first start the backend generates the bundled

demo archive (22 k VIIRS-format detections, Mar–May 2025, over real Indian industrial belts), trains the

model (\~7 s) and classifies everything. No internet or API key is needed for the demo.



Manual start:



\`\`\`bash

\# backend

cd backend

python -m venv .venv && .venv\Scripts\activate      # Linux/macOS: source .venv/bin/activate

pip install -r requirements.txt

uvicorn app.main:app --reload --port 8000



\# frontend (second terminal)

cd frontend

npm install

npm run dev

\`\`\`



**## Going live with NASA FIRMS**



1\. Get a free MAP_KEY (instant): https\://firms.modaps.eosdis.nasa.gov/api/map_key/

2\. \`copy backend\\.env.example backend\\.env\` and set \`FIRMS_MAP_KEY=...\`

3\. Restart the backend, click **\*\*Run pipeline\*\*** in the dashboard → *\*Live NASA FIRMS\**, pick an area preset

   or the current map view, choose days back (NRT keeps \~2 months; use \`VIIRS_SNPP_SP\` for archives).



The pipeline downloads the hotspots in 10-day chunks, then queries Overpass \*\*only around the cells

where hotspots occurred\*\* (not all of India) for industrial infrastructure and land-use polygons, and

caches the responses in \`backend/data/cache/\`. You can also drop a FIRMS archive CSV and choose

*\*FIRMS archive CSV\** as the source. Add hand-curated facilities to \`backend/data/curated_sites.geojson\`

(same schema as \`demo_sites.geojson\`) and they are merged with OSM.



**## How the classification works**



\| Signal | Feature(s) | Why it matters |

\|---|---|---|

\| Radiometry | FRP, I-4/I-5 brightness, Δbrightness, FIRMS confidence, day/night | flares are small & steady, wildfires hot & variable |

\| Persistence (per DBSCAN cluster, 90-day window) | active days, span, max gap, detections, night fraction, FRP mean / CV / z-score, **\*\*robust median + MAD baseline\*\***, spatial spread | factories recur for months; crop burns last a day |

\| Infrastructure (OSM) | distance to nearest facility, facility type (refinery, steel, power, flare, cement, brick kiln, mine, chemical) | places the anomaly on a known industrial asset |

\| Land cover (OSM land-use) | cropland / forest / shrub-grass / industrial / mining / built / water | separates agricultural burning and wildfires |

\| Neighbourhood | detections within 1 km / 5 km, mapped facilities within 5 km, distance to the 2nd-nearest site | residue burning comes in sheets; a plant is a lone dot inside an estate |

\| Calendar | month, **\*\*local solar hour\*\*** (India spans 68°E–97°E, so UTC alone mixes overpass times), residue-burning season (Oct–Nov, Apr–May) | seasonality and time-of-day prior |



Two classifiers vote: a transparent **\*\*rule engine\*\*** (every decision returns the human-readable reasons

shown in the UI) and a **\*\*HistGradientBoosting\*\*** model. The ML label is used when its probability ≥ 0.55,

otherwise the rules take the decision back and the row says so.



**### Scoring it honestly**



A single random split would flatter this model badly, so none is used. Every number in \`GET /api/model\`

and in the dashboard's **\*\*Model\*\*** tab comes from a group-aware evaluation:



\* **\*\*5-fold GroupKFold by thermal source\*\*** — no source appears on both sides, so per-source persistence

  features cannot leak the answer. Reported as mean ± standard deviation across folds.

\* **\*\*Leave-one-block-out over \~330 km blocks\*\*** — train on the rest of India, then classify a region the

  model has never seen. This is the number that says whether it travels.

\* **\*\*Ablation\*\*** — rules only vs model only vs the shipped hybrid, on the same hold-out.

\* **\*\*Probability calibration\*\*** (isotonic, fitted on held-out folds) with expected calibration error and

  Brier score before and after, plus a reliability curve. An operator triages on the confidence number,

  so "82 %" has to mean right about 82 % of the time. Confidence is reported as the calibrated

  probability and is never nudged; whether the rules agree is a separate flag on every row.



The demo archive deliberately contains ambiguous cases — stubble fires in the fields surrounding a

refinery, kilns on cropland, plants missing from OSM — so the score is not a formality.

\`scripts/selftest.py\` prints all of it and **\*\*fails the build\*\*** if cross-validated accuracy, unseen-region

accuracy or calibration regress, so a bad change cannot reach the public demo.



**\*\*Persistent source\*\*** = ≥ 8 distinct active days and ≥ 14-day span within the window.

**\*\*FRP anomaly\*\*** = a detection at a persistent source ≥ 2.5× that source's **\*\*median\*\*** FRP with robust

z ≥ 4 (median/MAD, not mean/std: a large enough spike drags the mean up far enough to mask itself).



**## Alerts and the analyst loop**



Twenty-odd thousand detections are not a worklist. The pipeline reduces them to four kinds of alert, each scored

0–100 so the feed sorts itself:



\| Alert | Meaning |

\|---|---|

\| \`FRP_ANOMALY\` | a known source is burning far above its own baseline — incident candidate |

\| \`NEW_SOURCE\` | a persistent source that was not there before — new or unlicensed plant |

\| \`WENT_DARK\` | a persistent source has gone quiet — shutdown, or a gap in the detection chain |

\| \`UNREGISTERED\` | persistent industrial-type heat with no mapped facility nearby |



Every alert carries the evidence that produced it. From any detection or source an analyst can confirm

the class or correct it; verdicts are stored in \`feedback\` and **\*\*applied as training labels\*\*** on the next

\`POST /api/retrain\`, which reports how many detections were relabelled. In the static demo the verdicts

are queued in the browser instead, ready to replay against a real backend.



Live FIRMS observations do not include industrial-fire class labels. The model first classifies them

for review; only detections an analyst confirms or corrects are added as labelled live examples on

retraining. Unreviewed detections are never treated as ground truth.



**## ML algorithm comparison**



The local dashboard's **\*\*ML Compare\*\*** sidebar section trains and compares Decision Tree, Random Forest,

HistGradientBoosting, and XGBoost on the same labeled VIIRS-format archive. The experiment uses the

same engineered feature columns and a reproducible 5-fold \`StratifiedGroupKFold\` split by thermal

\`group_id\`; it evaluates the selected held-out fold, so no thermal group appears in both train and

test. Median imputation is fitted on training rows only. Generated benchmark files are stored under

\`backend/data/ml_models/\` and are excluded from source control.



The archive's labels are generated by the demo data generator. They are useful for a reproducible

comparison but are **\*\*not independent ground truth or live NASA FIRMS validation\*\***. Live detections

must not be treated as labeled examples unless an analyst confirms or corrects them.



With the backend running, use these endpoints (also available in FastAPI at \`/docs\`):



\| Method | Endpoint | Purpose |

\|---|---|---|

\| \`POST\` | \`/api/ml/train\` | Start training all four benchmark models asynchronously |

\| \`GET\` | \`/api/ml/comparison\` | Read current training status or the completed metrics, reports, and confusion matrices |

\| \`GET\` | \`/api/ml/models\` | Check which saved model artifacts are available |

\| \`POST\` | \`/api/ml/predict\` | Predict using a selected trained model |



After \`POST /api/ml/train\`, poll \`/api/ml/comparison\` until its \`status\` is \`ready\`. The dashboard

shows weighted accuracy/precision/recall/F1 and latency, plus macro recall (used for class-balanced

best-model selection), training time, weighted OVR ROC-AUC, per-class reports, confusion matrices,

and feature importance. The API's \`dataset.evaluation_note\` travels with the metrics.



Example prediction from existing stored detections:



\`\`\`json

{

  "model": "random_forest",

  "hotspot_ids": ["h123", "h456"]

}

\`\`\`



Alternatively, send \`samples\` as a list of feature objects using the \`dataset.features\` columns from

\`GET /api/ml/comparison\`. Supply exactly one of \`hotspot_ids\` or \`samples\`. Model files are

\`decision_tree_model.joblib\`, \`random_forest_model.joblib\`, \`hist_gradient_model.joblib\`, and

\`xgboost_model.joblib\`. The static GitHub Pages build can display exported data but cannot train or

run predictions; use the local backend for those actions.



The dashboard also has a **\*\*time-travel\*\*** control: scrub or play the 90 days a day at a time and watch a

plant burn every night while a crop fire flares once and is gone.



**## Forecasting the next seven days**



Everything above describes what already burned. Two more models ask what burns next.



\| Model | Question | Signal |

\|---|---|---|

\| **\*\*Incident risk\*\*** | will this persistent source spike far above its own baseline within 7 days? | plants run into upset conditions — unstable process heat, extra flaring — before something fails, so the recent rise against a long baseline is what carries the prediction |

\| **\*\*Outbreak risk\*\*** | will this \~28 km cell see a burst of new fires within 7 days? | recent local activity, what the neighbouring cells are doing, and where the calendar sits in the residue-burning season |



Both are **\*\*strictly causal\*\***: every feature for an origin day \`d\` is computed only from detections on

or before \`d\`, and the label is what happened in \`(d, d+7]\`. The test set is always *\*later in time\**

than the training set, because a forecast scored on a random split is not a forecast.



**### SMS area alerts**



The live dashboard can send an SMS when a forecasted area within a subscriber-selected radius reaches

50% risk or higher over the next seven days. Subscriptions are verified by a one-time code; alerts are

deduplicated to one message per subscriber and forecast horizon. This is a forecast, not an emergency

warning, and subscribers can stop alerts from the Forecast tab.



To enable delivery, add a Twilio account SID, auth token, and sending number to \`backend/.env\` using

the names in \`backend/.env.example\`, then restart the backend. Enter the recipient number in E.164

format and the center coordinates/radius in the dashboard; Twilio trial accounts may only send to

their verified destination numbers. The SMS opt-in is unsupported by the static GitHub Pages demo.



**### FRP anomaly response prioritization**



The **\*\*FRP Anomaly Finder\*\*** sidebar ranks existing flagged detections by the backend's 0–100

\`anomaly_severity\` score. The score is calculated from a persistent source's FRP relative to its

median baseline, robust z-score, and absolute FRP; detections must pass the anomaly thresholds in

\`backend/app/classifier.py\`. The panel exposes the score, FRP, baseline ratio, robust z-score, and

date, and selecting a result opens that detection on the map. This prioritizes records for analyst

review; a thermal anomaly is not confirmation of a fire or emergency, and the score is not a response

or dispatch recommendation.



Rare events make accuracy meaningless, so the scores are reported against the base rate — AUC, PR-AUC,

Brier, and the **\*\*lift\*\*** from ranking by risk instead of guessing. On the demo archive:



\| | AUC | base rate | precision in the top 10% | lift |

\|---|---|---|---|---|

\| Incident risk | 0.73 | 9.7% | 23.7% | **\*\*2.4×\*\*** |

\| Outbreak risk | 0.91 | 29.7% | 96.7% | **\*\*3.3×\*\*** |



Every score comes with plain-language drivers ("running 4.3× its own baseline in the last week",

"142 more detections in the surrounding cells"), and the area forecast is a map layer drawn *\*beneath\**

the detections, because a forecast must never be mistaken for an observation. \`GET /api/forecast\` and

\`GET /api/forecast/grid\`.



\> **\*\*Stated assumption.\*\*** The demo archive simulates the upset period before an incident: a week of

\> elevated, erratic FRP leading up to it. Without some precursor the incident date would be random by

\> construction and no forecaster could honestly learn it. On live FIRMS data the same features are

\> computed from real observations, and the CI gate (AUC and lift over the base rate) is what says

\> whether the model still earns its place.



**## API**



\| Endpoint | Purpose |

\|---|---|

\| \`POST /api/ingest\` | run pipeline: \`{source: demo|firms|csv, bbox, days, use_osm}\` (background job) |

\| \`GET /api/status\` | job state, last run metadata |

\| \`GET /api/hotspots?bbox&labels&date_from&date_to&persistent&anomaly&min_conf\` | classified detections (GeoJSON) |

\| \`GET /api/hotspots/{id}\` | full record + reasons + source time series + imagery links |

\| \`GET /api/sources\` / \`/api/sources/{group_id}\` | persistent thermal sources (GeoJSON) |

\| \`GET /api/sites\` | industrial infrastructure used as context |

\| \`GET /api/stats\`, \`GET /api/model\` | dashboard KPIs; full model evaluation (CV, spatial hold-out, ablation, calibration) |

\| \`GET /api/alerts?kinds&min_severity\` | ranked operational alert feed |

\| \`GET /api/forecast\` | 7-day incident risk per persistent source, with drivers and validation scores |

\| \`GET /api/forecast/grid\` | 7-day outbreak risk per \~28 km cell (GeoJSON) |

\| \`GET /api/sms/status\`, \`POST /api/sms/subscribe\`, \`/api/sms/verify\` | Twilio readiness and verified area-alert opt-in |

\| \`POST /api/sms/unsubscribe\`, \`/api/sms/unsubscribe/verify\` | verified SMS alert opt-out |

\| \`POST /api/feedback\` / \`GET /api/feedback\` | analyst confirms or corrects a class |

\| \`POST /api/retrain\` | refit with analyst corrections applied over the archive labels |

\| \`GET /api/export/hotspots.geojson\`, \`/api/export/sources.geojson\` | GIS export (QGIS/ArcGIS) |

\| \`GET /api/export/alerts.csv\`, \`/api/export/registry.csv\` | shift log and persistent-source registry |



**## Repository layout**



\`\`\`

backend/app/firms.py      FIRMS area API client, VIIRS/MODIS normalisation

backend/app/osm.py        Overpass queries (around hotspot cells), tag → site type / land cover

backend/app/features.py   DBSCAN clustering, persistence statistics, context features

backend/app/classifier.py rule engine + calibrated gradient boosting, anomaly flag, evaluation

backend/app/alerts.py     detections -> ranked operational alert feed

backend/app/forecast.py   causal 7-day incident and outbreak risk, scored on a time split

backend/app/pipeline.py   orchestration; persistent-source table, alerts, feedback-aware retrain

backend/app/demo.py       physically-motivated demo archive generator (labelled)

backend/app/db.py         SQLite storage & queries

backend/app/main.py       FastAPI

frontend/src/             React + Leaflet dashboard (canvas rendering for 40 k+ points)

docs/                     Hackathon idea deck & notes

\`\`\`



**## Roadmap after the hackathon**



\* Sentinel-2 SWIR (B12/B11) chip classifier on top of each persistent source (CNN) — imagery links are

  already wired; Copernicus Data Space offers free API access.

\* ESA WorldCover 10 m raster instead of OSM land-use polygons where OSM is sparse.

\* Landsat-8/9 TIRS + VIIRS Nightfire (VNF) for flare temperature / methane-flaring estimates.

\* PostGIS + tile server for national-scale multi-year archives; pushing the existing alert feed out over

  email / SMS / webhook rather than only serving it.



**## How the public demo is built**



\`.github/workflows/pages.yml\` runs the **\*\*real pipeline\*\*** on the GitHub runner on every push to \`main\`:

install the backend, run \`scripts/selftest.py\` (which fails the deploy if the model regresses), export

the static JSON, then build the dashboard with \`VITE_STATIC=1\`. The demo data is therefore never a stale

committed blob — it is regenerated from the code in that commit, and the demo URL never changes.
