---
layout: article
title: "ThermalGuide — ML prediction of thermals and wind fields from paragliding GPS data"
date: 2025–ongoing
description: "Bachelor's thesis at ZHAW: a full-stack system that recovers latent flight states from GPS paragliding trajectories via MAP estimation and uses them to train a physics-structured multiplicative GAM for vertical-wind and wind-field prediction. Four components — React/MapLibre frontend, Django/GeoDjango/PostGIS backend, ML repo (PyTorch + FastAPI), MkDocs documentation."
image: /assets/images/projects/thermaltrack/cover.jpg
permalink: /en/projects/thermaltrack/
lang: en
key: project-thermaltrack
sidebar:
  nav: project-en
---

![ThermalGuide — interactive thermal and wind-field prediction for the Prättigau–Davos region, rendered via the frontend MapLibre map with 3D terrain]({{ '/assets/images/projects/thermaltrack/cover.jpg' | relative_url }})

## Overview

**ThermalGuide** is my ongoing Bachelor's thesis in *B.Sc. Electrical Engineering* at the **ZHAW** (examination committee: Prof. Karl Rege). The goal is a **predictive model for thermals and wind fields** in the Prättigau–Davos region of Switzerland, trained from **GPS trajectories of paragliding flights** — Lagrange-sensor data. That framing matters: the "sensor" (the glider) itself moves with the field it is meant to observe, so the directly measurable quantities (groundspeed, GPS altitude, vario) are only indirect witnesses of the underlying air mass and vertical wind. The quantities of interest — heading, airspeed, local wind — are **latent** and must be reconstructed from the trajectory *before* any actual learning happens.

The system is built as **four separate repositories** that together form one end-to-end data and prediction path: ingest & persistence (backend), learning & inference (ML), visualization (frontend), and documentation aggregation (docs). All four are private while the thesis is being written; this page describes the architecture and the scientific approach without exposing source code.

---

## System architecture

Four components, one request path from browser to prediction tile:

```
Browser
  │  HTTPS
  ▼
┌───────────────────────────────────────────────────────────────┐
│  Frontend (React 19 + Vite, MapLibre GL 5 + deck.gl 9)        │
│    /            HomePage                                      │
│    /thermal     Server-rendered thermal map (iframe)          │
│    /predict     ML prediction map (iframe)                    │
│    /map         Client-side MapLibre, 3D terrain, overlays:   │
│                 surface type / slope / aspect                 │
└──────────────────────────┬────────────────────────────────────┘
                           │ nginx proxy  /api/…
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  Backend (Django 5 + DRF + GeoDjango, gunicorn)               │
│                                                               │
│  Apps: flights · flight_analysis · weather · grids · geolayers│
│        machinelearning · layers_3d · webscraper_xc · ingest   │
│        lib/igc (pure Python, Django-free)                     │
│                                                               │
│  Data layers: RAW → CANONICAL → PRODUCTS                      │
│  Architecture lints enforced before every commit              │
└────┬──────────────┬────────────────────┬──────────────────────┘
     │ ORM          │ Celery dispatch    │
     ▼              ▼                    │
┌──────────┐  ┌─────────────────┐        │ tile/JSON
│ Postgres │  │ Redis + Celery  │        │
│ +PostGIS │  │ worker fleets   │        │
└──────────┘  └────────┬────────┘        │
                       │ "map" queue     │
                       ▼                 │
┌────────────────────────────────────────┴──────────────────────┐
│  ML repo                                                      │
│  (1) MAP solver (Celery worker): reconstructs latent flight   │
│      states via Gauss-Newton MAP, O(T) time per flight.       │
│  (2) Hierarchical energy-block GAM (training, PyTorch/GPU)    │
│  (3) FastAPI inference server: tile prediction, a model       │
│      registry that auto-discovers new runs (POST /model/refresh)│
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│  Docs repo                                                    │
│  Obsidian vault → MkDocs Material (awesome-pages, roamlinks,  │
│  mermaid2). Pulls the docs/ dirs of the module repos via the  │
│  GitHub API and stages them centrally for the docs site.      │
└───────────────────────────────────────────────────────────────┘
```

All services share one `../.env.production` on the server — a deliberate design choice, because DB credentials, media paths and SSL paths are shared across backend, frontend and the docs subdomain anyway.

---

## Component 1 — Backend (Django / GeoDjango / PostGIS)

**Role:** ingest, persistence, orchestration, tile serving.

- **Stack:** Django 5, Django REST Framework, **GeoDjango** on **PostgreSQL + PostGIS**, Celery + Redis + Flower, gunicorn, whitenoise.
- **Scraping subsystem (`webscraper_xc`):** two-stage pipeline for XContest — (a) discovery by coordinate + radius and year range (2011–2025), (b) bulk IGC download with retry and rate limiting. **Tor integration** (`stem`) for IP rotation with circuit management, multi-account authentication with credential rotation, undetected-chromedriver + Selenium. Custom Django admin operations dashboard.
- **IGC parser (`lib/igc/`)** — deliberately kept as a **pure Python library with no Django imports** so the same parser can be used offline from the ML repo.
- **Data pipeline layering (`backend/docs/data-architecture.md`):** `RAW → CANONICAL → PRODUCTS`. RAW stores untouched input, CANONICAL is the lint-checked single source of truth, PRODUCTS are derived artefacts (tiles, aggregates). Import directions are enforced by a custom **architecture lint** (`make arch-lint`, 20+ rules in `backend/lint/lint_registry.yaml`).
- **Geolayers app:** server-side tile rendering for PostGIS rasters (`raster_renderers/`) and Zarr stores (`zarr_renderers/`). DEM, slope, aspect, WorldCover. The MapLibre 3D-terrain `raster-dem` tiles are encoded as Terrain-RGB.
- **Enrichment:** ICON-DREAM numerical weather, ERA5, Payerne radiosonde and 250 m-hesse analyses are joined server-side onto flight trackpoints. Speedup trick: deduplicating `(time, x, y)` triples reduces Zarr queries by a factor of ~15 000.
- **Test discipline:** 117 test files, golden-file regression tests for tile output, pytest markers (`unit`, `integration`, `api`, `golden`, `slow`, `e2e`).

---

## Component 2 — ML repo (PyTorch, FastAPI)

**Role:** the scientific core — recover latent flight states, train the physics-structured GAM, serve predictions.

Three subsystems, one shared configuration idea (YAML).

### 2a — MAP estimation of flight states (`src/map/`)

A paraglider GPS track directly provides only **groundspeed vectors**. Heading ψ(t), airspeed V_a(t) and local wind w(t) are **latent** — they must be inferred from the track plus a physical forward model. Formally, as a maximum-a-posteriori problem:

$$
\psi^* \;=\; \arg\min_{\psi}\; \tfrac{1}{2}\!\sum_i r_i(\psi)^{\!\top} R_i^{-1} r_i(\psi)\;+\;\tfrac{1}{2}\,\psi^{\!\top} Q^{-1} \psi\;+\;\tfrac{1}{2}\,\tfrac{(\psi_1-\chi_1)^2}{\sigma^2}
$$

with residual $r_i = \mathbf{v}_\text{gnd,i}^\text{obs} - \mathbf{v}_\text{gnd}(\psi_i, V_a, w_\text{prior})$. Because the trajectory prior Q is tridiagonal (smoothness on adjacent time steps), the normal equations are also tridiagonal and solvable by a **banded solve in $\mathcal{O}(T)$** per flight. A `dense_debug` validation path is kept in lock-step with the `banded` solver and they agree within 1e-6 degrees.

The implementation is deliberately **config-driven**: latent variables are typed by `Scope` (trajectory / window / flight) and `Role` (`state` / `nuisance` / `calibration` / `diagnostic`). Derived quantities (bank angle, load factor, wing area, total energy) carry an **explicit `frozen` / `differentiated` policy** in the YAML — each is either recomputed per Gauss-Newton step without entering the Jacobian, or threaded through the Jacobian via the chain rule.

The solver runs as a **Celery consumer on the `map` queue** that the backend dispatches; solved states are written back to the flight parquets.

### 2b — Hierarchical energy-block GAM (`src/gam/`)

Vertical wind is **not** modelled by an opaque deep network, but as a **product of physically-motivated blocks**:

$$
u_z \;=\; E_\text{rad}(t,\mathrm{sun}) \,\cdot\, \gamma(\mathrm{moisture}) \,\cdot\, A_\text{surf}(\mathrm{surf}) \,\cdot\, C_\text{aero}(\mathrm{terrain}) \,\cdot\, \mathrm{inhibition}(\mathrm{cloud},\mathrm{precip}) \,\cdot\, \mathrm{spatial\_prior}(\mathrm{hotspots}) \;+\; \mathrm{vertical\_structure}(\mathrm{AGL},\mathrm{BL})
$$

Each block is a **mini-GAM in its own right**, with its own basis system, link function and coefficients:

- **Source block $E_\text{rad}$** — `softplus` link (strictly positive), tensor product of day-of-year × hour-of-day + solar radiation + horizon shadow factor + sun incidence angle.
- **Modifier blocks**, multiplicative, centered at 1.0:
  - $\gamma$ (moisture partition) — `sigmoid`, features: soil moisture, VPD.
  - $A_\text{surf}$ (surface absorption) — `softplus`, features: surface thermal potential.
  - $C_\text{aero}$ (aerodynamic coupling) — `softplus`, features: surface roughness, terrain slope.
  - inhibition (cloud / precipitation) — `sigmoid`, features: cloud cover, precipitation.
  - spatial_prior — `softplus`, **Spatial-RBF basis** placed at thermal-hotspot cluster centers.
- **Additive block** (`vertical_structure`) — `identity` link, captures height-dependent structure that the multiplicative chain cannot. Features: AGL, altitude fraction, forecast vario, thermal-top / inversion indicators.

Training uses **block-coordinate descent** (outer loop up to 15 iterations): in each outer step every multiplicative block is updated by **inner IRLS** (Gauss-Newton with trust region), followed by a weighted least-squares update of the additive block. Modifiers are renormalised to geometric mean 1.0 after every step.

Basis terms are explicitly separated: `SplineTerm`, `CyclicSplineTerm` (for `time_hour_float` with period [0,24] and `time_day_of_year` with period [1,366] — allowing sharp diurnal transitions without the artefacts of a sin/cos encoding), `LinearTerm`, `CategoricalTerm`, `TensorProductTerm` (for seasonal × diurnal), `ModulatedSplineTerm` (for shadow × incidence) and `SpatialRBFTerm`. B-spline evaluation uses de Boor recursion on GPU.

### 2c — Inference server (`serve/`, FastAPI)

A dedicated **FastAPI inference server** with a GPU backend. Endpoints for tile prediction (`/predict/tile`, `/predict/tile/arrow`, `/predict/tile/raw`), profiles (`/predict/profile`) and model management (`/model/info`, `/model/refresh`, `/model/list`). A **model registry** auto-discovers new training runs under `runs/gam/` — new models enter service without a restart. The tile format **TTDT v1** encodes predictions as gzip-compressed uint16, tuned for MapLibre raster sources.

### Data

- 12 yearly parquet files (2012–2025, 2015 and 2026 excluded — 2015 is 100 % NaN in the r570 export, 2026 has too few flights), **175 columns**, region 570 (Prättigau–Davos, ~41 × 47 km).
- After bounding-box cleaning: **~60 million rows from ~11 000 flights**.
- **Content-addressed caching** in two stages (clean + GPU subsample) — invalidation only on config or data changes; exactly one parquet row group per flight plus a flight index for O(1) per-flight access.

---

## Component 3 — Frontend (React / MapLibre)

**Role:** interactive visualization of predictions and flight data.

- **Stack:** React 19.1, TypeScript 5.8, Vite 7.1 (SWC), Mantine v8, AG Grid 35, **MapLibre GL 5.18 + react-map-gl 8 + deck.gl 9** (aggregation / geo / mapbox layers), ECharts 6, Apache Arrow 21, axios, react-router-dom 7, react-resizable-panels.
- **Routes:**
  - `/` — navigation page
  - `/thermal` — backend-rendered thermal map (iframe)
  - `/predict` — ML prediction map (iframe)
  - `/map` — client-side MapLibre map with OpenTopoMap base, dynamically fetched overlays from the backend (surface type, slope, aspect), **2D/3D terrain toggle** (MapLibre `raster-dem` Terrain-RGB, 1.5× exaggeration), slope and aspect sliders that parameterize the tile URL (MapLibre re-fetches tiles automatically).
- **Container architecture:** multi-stage Dockerfile. Dev stage with Vite HMR, build stage for production, nginx stage for release. The `docker-compose.prod.yml` references the shared `../.env.production`.
- **Delivery efficiency:** Apache Arrow IPC for 3D trajectories (`layers_3d` backend app); tile-based layers for everything raster-like.

---

## Component 4 — Docs (Obsidian → MkDocs)

**Role:** an **aggregated documentation site** that combines (a) curated material from an Obsidian vault with (b) the `docs/` directories of the frontend and backend repos.

- **Authoring tool:** Obsidian, including wiki-link syntax `[[…]]`, resolved by the MkDocs plugin `roamlinks` for the static site.
- **Build:** MkDocs Material with `awesome-pages` (automatic navigation from `.pages` files, **no** hand-edited YAML), `mermaid2` for diagrams.
- **Aggregation:** custom Python scripts (`stage_vault.py`, `fetch_sources.py`, `generate_navigation.py`) — **cross-platform**, no rsync, no bash, works on Windows without WSL. `.vaultignore` controls which vault content reaches the public site (private protocols, drafts, scratch directories stay out).
- **Deployment:** GitHub Actions → rsync to a subdomain (`docs.thermaltrack.*`) or a `/docs` nginx fallback.

---

## Full-stack integration

Four things worth noting about how the four repos work together:

1. **Shared `.env.production` on the server.** Rather than three parallel env files per service, a single file lives one directory above each repo and every `docker-compose.prod.yml` references it. Each repo's `.env.*.example` documents only its own slice of the keys.
2. **Celery queue as the backend ↔ ML bridge.** Instead of importing the ML module as a library in the Django worker (which would make the worker heavy and GPU-dependent), the MAP solver runs as a **separate Celery consumer** on the `map` queue. The backend dispatches `solve_heading_task` messages to the same Redis broker that Celery already uses. GPU isolation, horizontal scalability and clean deploy separation come for free.
3. **IGC parser in `lib/igc/`, Django-free.** The same Python library can be imported by the Django backend *and* by the offline ML training job without booting the Django app registry. Enforced by the architecture lint.
4. **Architecture lint in the backend.** `backend/lint/arch_lint.py` checks 20+ rules on every commit: import directions, stub-task prohibitions, library purity (no Django imports in `lib/`), task-vs-service-vs-command layering. Violations with expiry dates live in `waivers.yaml`. This kept the codebase structurally coherent during fast feature growth.

---

## Status — Bachelor's thesis in progress

**This project is an active Bachelor's thesis, not a finished product.** Rough timeline:

- **September 2025** — start, three repos initially created (backend, frontend, docs), server setup, XContest webscraper as the first ingest source, weekly supervisor meetings with Karl Rege (logged in the Obsidian vault).
- **October–December 2025** — Django apps `flights`, `weather`, `flight_analysis`, `grids`, `geolayers`, `machinelearning` come online; data pipeline RAW → CANONICAL → PRODUCTS is put in place and the architecture lints are introduced. Frontend routes `/`, `/thermal`, `/predict`, `/map` take shape.
- **January–February 2026** — enrichment layer (ICON-DREAM, ERA5, radiosondes, hesse-250 m), Zarr tile renderers, 3D terrain via Terrain-RGB tiles.
- **Since mid-March 2026** — a dedicated ML repo (`ThermalTrackML`); first the **MAP solver**, then the **hierarchical block GAM**, most recently the **DAG refactor** for multi-output training (`w` + `wind_u` + `wind_v` in one run).

**What's in / what's still underway:**

- ✅ Ingest pipeline (webscraper, IGC parser, flight metadata in PostGIS), enrichment from external weather and terrain sources.
- ✅ MAP solver v1 (heading-only) validated against the dense-debug path to 1e-6°.
- ✅ Hierarchical GAM is trainable, the model registry and the inference server are up.
- 🔄 **In progress:** MAP progression v2 (+wind) → v3 (+V_a) → v4 (+calibration); stagewise co-training of flight states and GAM (Stage A / Stage C alternating); a Gaussian process on the residuals (Stage B) is the planned third stage.
- 🔄 **In progress:** evaluation and ablations against baseline predictors; writing up the scientific findings for the thesis document.

A specific submission date is not given here — it is not relevant to this presentation and still being calibrated within the project timeline. After completion and grading, the plan is to release selected code samples and the written thesis on request.

---

## Technology profile at a glance

| Layer | Technology |
|---|---|
| Languages | Python (~112 k LOC), TypeScript (~21 k LOC) |
| Web framework (backend) | Django 5, Django REST Framework, GeoDjango |
| DB | PostgreSQL + PostGIS |
| Queue | Celery + Redis + Flower |
| Scraping | Selenium + undetected-chromedriver + patchright, Tor (`stem`), multi-account auth |
| Geospatial | PostGIS rasters, Zarr stores, pyproj, DEM / slope / aspect / WorldCover, MapLibre raster-DEM |
| ML | PyTorch (GPU), numpy, scipy, custom B-spline / basis infrastructure, FastAPI inference |
| Frontend | React 19, Vite 7, Mantine 8, MapLibre GL 5, react-map-gl 8, deck.gl 9, AG Grid 35, ECharts 6, Apache Arrow |
| Docs | Obsidian → MkDocs Material + awesome-pages + roamlinks + mermaid2 |
| CI / deploy | GitHub Actions (frontend + backend + docs), docker-compose, shared `.env.production`, SSL termination via nginx |
| Quality | 117 pytest files, golden-file regression, custom architecture lint with 20+ rules |

---

## Screenshots

<!-- TODO: add images. Suggestions:
  1. /map route with 3D terrain active + slope overlay + a flight track
  2. Tile prediction (FastAPI /predict/tile) as a thermal heatmap
  3. Partial-dependence plot of a block (source or spatial_prior)
  4. Architecture diagram (ASCII version above — cleaner as a graphic)
-->

---

## Source

**Note:** This project is part of an ongoing Bachelor's thesis at ZHAW. Source code and scientific results are confidential until thesis submission and not publicly available. After completion, selected code samples and the written thesis can be shared on request.

---

## Reflection

Two decisions that, in hindsight, really mattered:

**Multi-repo, not monorepo.** The four repos have **different deploy life cycles** — the frontend builds in seconds, the backend with its geospatial dependencies takes minutes, the ML repo is GPU-dependent, the docs site rebuilds on every push independently. A monorepo would have felt faster at the start; in practice, every push would have run through one slow shared CI, and the ML code would have bloated the backend into a GPU image. The price — the shared `.env.production` and the Celery queue as bridge — was worth it.

**Physics structure instead of an end-to-end deep net.** The multiplicative GAM with physically-motivated blocks is significantly more work than a generic regressor over the same features. But in return, every block has its **own link function, its own feature set, its own lambda-tuning knobs** — which enables targeted ablations ("turn off the moisture block and retrain", "replace $A_\text{surf}$ features with WorldCover-only"), partial-dependence plots per block, and an argumentation chain for the thesis that reaches beyond "the model minimized the loss". For a Bachelor's thesis that will be reviewed by domain experts, that was the right investment.
