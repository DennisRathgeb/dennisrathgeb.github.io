---
layout: article
title: "ThermalGuide — ML-Vorhersage von Thermik und Windfeldern aus GPS-Gleitschirmdaten"
date: 2025–laufend
description: "Bachelorarbeit an der ZHAW: ein Full-Stack-System, das aus GPS-Gleitschirmflügen latente Flugzustände via MAP-Schätzung rekonstruiert und damit ein physikalisch strukturiertes, multiplikatives GAM zur Vorhersage von Vertikalwind und Windfeldern trainiert. Vier Teilsysteme – React/MapLibre-Frontend, Django/GeoDjango/PostGIS-Backend, ML-Repo (PyTorch + FastAPI), MkDocs-Dokumentation."
image: /assets/images/projects/thermaltrack/cover.jpg
permalink: /projekte/thermaltrack/
lang: de
key: project-thermaltrack
sidebar:
  nav: project-de
---

![ThermalGuide — interaktive Thermik- und Windfeldvorhersage für die Region Prättigau–Davos, gerendert über die frontend-seitige MapLibre-Karte mit 3D-Gelände]({{ '/assets/images/projects/thermaltrack/cover.jpg' | relative_url }})

## Überblick

**ThermalGuide** ist meine laufende Bachelorarbeit im Studiengang *B.Sc. Elektrotechnik* an der **ZHAW** (Prüfungsausschuss: Prof. Karl Rege). Ziel ist ein **vorhersagefähiges Modell für Thermik und Windfelder** in der Region Prättigau–Davos, das aus **GPS-Trajektorien von Gleitschirmflügen** lernt – also aus Lagrange-Sensordaten. Das bedeutet: der "Sensor" (der Gleitschirm) bewegt sich selbst mit dem zu beobachtenden Feld, die direkten Messgrössen (Groundspeed, GPS-Höhe, Vario) sind nur indirekte Zeugen von Luftmasse und Vertikalwind. Die gesuchten Grössen – Heading, Airspeed, lokaler Wind – sind **latent** und müssen vor dem eigentlichen Lernen aus der Trajektorie rekonstruiert werden.

Das System besteht aus **vier separaten Repositories**, die zusammen einen durchgehenden Daten- und Vorhersagepfad bilden: Ingest & Persistenz (Backend), Lernen & Inferenz (ML), Visualisierung (Frontend) und Dokumentationsaggregation (Docs). Alle vier sind wegen der laufenden Arbeit privat; diese Seite beschreibt die Architektur und den wissenschaftlichen Ansatz ohne Quellcode.

---

## Systemarchitektur

Vier Komponenten, ein Request-Pfad vom Browser zum Vorhersage-Tile:

```
Browser
  │  HTTPS
  ▼
┌───────────────────────────────────────────────────────────────┐
│  Frontend (React 19 + Vite, MapLibre GL 5 + deck.gl 9)        │
│    /            HomePage                                      │
│    /thermal     Server-gerenderte Thermikkarte (iframe)       │
│    /predict     ML-Vorhersagekarte (iframe)                   │
│    /map         Client-side MapLibre, 3D-Gelände, Overlays:   │
│                 Bodenbeschaffenheit / Hangneigung / Exposition│
└──────────────────────────┬────────────────────────────────────┘
                           │ nginx-Proxy  /api/…
                           ▼
┌───────────────────────────────────────────────────────────────┐
│  Backend (Django 5 + DRF + GeoDjango, gunicorn)               │
│                                                               │
│  Apps: flights · flight_analysis · weather · grids · geolayers│
│        machinelearning · layers_3d · webscraper_xc · ingest   │
│        lib/igc (reines Python, kein Django)                   │
│                                                               │
│  Datenschichten: RAW → CANONICAL → PRODUCTS                   │
│  Architektur-Lints pflichtig vor jedem Commit                 │
└────┬──────────────┬────────────────────┬──────────────────────┘
     │ ORM          │ Celery dispatch    │
     ▼              ▼                    │
┌──────────┐  ┌─────────────────┐        │ tile/JSON
│ Postgres │  │ Redis + Celery  │        │
│ +PostGIS │  │ Worker-Flotten  │        │
└──────────┘  └────────┬────────┘        │
                       │ "map"-Queue     │
                       ▼                 │
┌────────────────────────────────────────┴──────────────────────┐
│  ML-Repo                                                      │
│  (1) MAP-Solver (Celery-Worker): rekonstruiert latente        │
│      Flugzustände via Gauss-Newton-MAP, O(T)-Zeit pro Flug.   │
│  (2) Hierarchisches Energie-Block-GAM (Training, PyTorch/GPU) │
│  (3) FastAPI-Inferenzserver: Kachelvorhersagen, Modellregistry│
│      entdeckt neue Runs automatisch (POST /model/refresh).    │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│  Docs-Repo                                                    │
│  Obsidian-Vault → MkDocs Material (awesome-pages, roamlinks,  │
│  mermaid2). Holt via GitHub-API die docs/-Verzeichnisse der   │
│  Modul-Repos und staged sie zentral für das Docs-Portal.      │
└───────────────────────────────────────────────────────────────┘
```

Alle Dienste teilen sich auf dem Server ein gemeinsames `../.env.production` – eine bewusste Designentscheidung, weil DB-Credentials, Medien-Pfade und SSL-Pfade zwischen Backend, Frontend und Docs-Subdomain eh geteilt sind.

---

## Komponente 1 — Backend (Django / GeoDjango / PostGIS)

**Rolle:** Ingest, Persistenz, Orchestrierung, Kachelserving.

- **Stack:** Django 5, Django REST Framework, **GeoDjango** mit **PostgreSQL + PostGIS**, Celery + Redis + Flower, gunicorn, whitenoise.
- **Scraping-Subsystem (`webscraper_xc`):** zweistufige Pipeline für XContest — (a) Discovery nach Koordinate + Radius und Jahr (2011–2025), (b) IGC-Bulk-Download mit Retry und Rate-Limit. **Tor-Integration** (`stem`) für IP-Rotation mit Circuit-Management, Multi-Account-Authentifizierung mit Credential-Rotation, undetected-chromedriver + Selenium. Eigenes Django-Admin-Operations-Dashboard.
- **IGC-Parser (`lib/igc/`)** — bewusst als **reine Python-Bibliothek ohne Django-Importe** gehalten, damit derselbe Parser auch offline im ML-Repo verwendbar ist.
- **Datenpipeline-Schichten (`backend/docs/data-architecture.md`):** `RAW → CANONICAL → PRODUCTS`. RAW speichert Rohdaten, CANONICAL ist die lint-geprüfte Einzelwahrheit, PRODUCTS sind abgeleitete Produkte (Tiles, Aggregate). Die Importrichtungen sind per **Architektur-Lint** (`make arch-lint`, 20+ Regeln in `backend/lint/lint_registry.yaml`) erzwungen.
- **Geolayers-App:** Server-side Kachelrendering für PostGIS-Raster (`raster_renderers/`) und Zarr-Stores (`zarr_renderers/`). DEM, Slope, Aspect, WorldCover. Die `raster-dem`-Kacheln für MapLibre-Terrain werden als Terrain-RGB encodiert.
- **Enrichment:** ICON-DREAM-Wettermodell, ERA5, Payerne-Radiosonden und Hessen-250m werden server-seitig auf die Flugtrackpunkte gejoint. Speedup-Ansatz: Dedup von `(time, x, y)`-Triples reduziert die Zarr-Anfragen um Faktor 15 000.
- **Test-Disziplin:** 117 Test-Dateien, Golden-File-Regression-Tests für Tile-Outputs, pytest-markers (`unit`, `integration`, `api`, `golden`, `slow`, `e2e`).

---

## Komponente 2 — ML-Repo (PyTorch, FastAPI)

**Rolle:** das wissenschaftliche Herz der Arbeit — latente Flugzustände rekonstruieren, physikalisch strukturiertes GAM trainieren, Vorhersagen ausliefern.

Drei Teilsysteme, eine gemeinsame Konfigurationsidee (YAML).

### 2a — MAP-Schätzung der Flugzustände (`src/map/`)

Ein Gleitschirm-GPS-Track liefert direkt nur **Groundspeed-Vektoren**. Heading ψ(t), Airspeed Va(t) und lokaler Wind w(t) sind **latent** — sie müssen aus dem Track und einer physikalischen Vorwärtsmodellierung inferiert werden. Formal als Maximum-a-posteriori-Problem:

$$
\psi^* \;=\; \arg\min_{\psi}\; \tfrac{1}{2}\!\sum_i r_i(\psi)^{\!\top} R_i^{-1} r_i(\psi)\;+\;\tfrac{1}{2}\,\psi^{\!\top} Q^{-1} \psi\;+\;\tfrac{1}{2}\,\tfrac{(\psi_1-\chi_1)^2}{\sigma^2}
$$

mit dem Residuum $r_i = \mathbf{v}_\text{gnd,i}^\text{beob} - \mathbf{v}_\text{gnd}(\psi_i, V_a, w_\text{prior})$. Weil der Trajektorien-Prior Q tridiagonal ist (Glattheit auf benachbarten Zeitschritten), sind die Normalgleichungen ebenfalls tridiagonal und per **Bandsolve in $\mathcal{O}(T)$** pro Flug lösbar. Für Validierungszwecke gibt es einen `dense_debug`-Pfad, der mit dem `banded`-Solver auf 1e-6 Grad übereinstimmt.

Die Implementierung ist bewusst **konfigurationsgetrieben**: latente Variablen werden per `Scope` (Trajektorie / Fenster / Flug) und `Role` (`state` / `nuisance` / `calibration` / `diagnostic`) typisiert. Abgeleitete Grössen (Bank-Winkel, Lastfaktor, Flügelfläche, Gesamtenergie) haben eine **explizite `frozen` / `differentiated`-Policy** in der YAML — sie sind entweder pro Gauss-Newton-Iteration neuberechnet aber nicht in der Jacobimatrix, oder über die Kettenregel differenziert.

Der Worker läuft als **Celery-Consumer auf der `map`-Queue**, die das Backend dispatcht; die gelösten Zustände werden zurückgeschrieben und persistiert.

### 2b — Hierarchisches Energie-Block-GAM (`src/gam/`)

Vertikalwind wird **nicht** durch ein opakes Deep-Net modelliert, sondern als **Produkt physikalisch motivierter Blöcke**:

$$
u_z \;=\; E_\text{rad}(t,\mathrm{sun}) \,\cdot\, \gamma(\mathrm{moisture}) \,\cdot\, A_\text{surf}(\mathrm{surf}) \,\cdot\, C_\text{aero}(\mathrm{terrain}) \,\cdot\, \mathrm{inhibition}(\mathrm{cloud},\mathrm{precip}) \,\cdot\, \mathrm{spatial\_prior}(\mathrm{hotspots}) \;+\; \mathrm{vertical\_structure}(\mathrm{AGL},\mathrm{BL})
$$

Jeder Block ist ein **eigenes Mini-GAM** mit eigenem Basis-System, eigener Link-Funktion und eigenen Koeffizienten:

- **Quellblock $E_\text{rad}$** — `softplus`-Link (strikt positiv), Tensorprodukt aus Tag-des-Jahres × Stunde-des-Tages + Solarstrahlung + Horizont-Schattenfaktor + Sonnen-Inzidenz.
- **Modifier-Blöcke**, multiplikativ, zentriert bei 1.0:
  - $\gamma$ (Feuchte-Partition) — `sigmoid`, Features: Bodenfeuchte, VPD.
  - $A_\text{surf}$ (Oberflächenabsorption) — `softplus`, Feature: Oberflächen-Thermalpotential.
  - $C_\text{aero}$ (aerodynamische Kopplung) — `softplus`, Features: Oberflächenrauigkeit, Hangneigung.
  - inhibition (Wolken/Niederschlag) — `sigmoid`, Features: Bewölkungsgrad, Niederschlag.
  - spatial_prior — `softplus`, **Spatial-RBF-Basis** an thermischen Hotspot-Cluster-Zentren.
- **Additiver Block** (`vertical_structure`) — `identity`-Link, fängt höhenabhängige Struktur auf, die die multiplikative Kette nicht erfasst. Features: AGL, Altitude-Fraction, Forecast-Vario, Thermaldach/Inversions-Indikatoren.

Das Trainingsverfahren ist **Block-Coordinate-Descent** (äussere Schleife bis 15 Iterationen): in jedem Schritt wird ein multiplikativer Block per **innerer IRLS** (Gauss-Newton mit Trust-Region) aktualisiert, danach der additive Block per gewichtetem Kleinste-Quadrate. Die Modifier werden nach jedem Schritt auf geometrisches Mittel 1.0 renormalisiert.

Die Basis-Terme sind explizit getrennt: `SplineTerm`, `CyclicSplineTerm` (für `time_hour_float` mit Periode [0,24] und `time_day_of_year` mit Periode [1,366] — erlaubt scharfe diurnale Übergänge ohne die Artefakte einer Sinus/Kosinus-Kodierung), `LinearTerm`, `CategoricalTerm`, `TensorProductTerm` (für saisonal × diurnal), `ModulatedSplineTerm` (für Schatten × Inzidenz) und `SpatialRBFTerm`. B-Spline-Auswertung via De-Boor-Rekursion auf GPU.

### 2c — Inferenzserver (`serve/`, FastAPI)

Ein eigener **FastAPI-Inferenzserver** mit GPU-Backend. Endpunkte für Tile-Vorhersage (`/predict/tile`, `/predict/tile/arrow`, `/predict/tile/raw`), Profile (`/predict/profile`) und Modellverwaltung (`/model/info`, `/model/refresh`, `/model/list`). Eine **Model-Registry** entdeckt neue Runs unter `runs/gam/` automatisch — neue Modelle landen ohne Serverneustart im Dienst. Das Tile-Format **TTDT v1** kodiert die Vorhersage als gzip-komprimiertes uint16, optimiert für MapLibre-Raster-Sources.

### Daten

- 12 Jahresparquet-Dateien (2012–2025, 2015 und 2026 ausgeschlossen — 2015 komplett NaN im r570-Export, 2026 zu wenige Flüge), **175 Spalten**, Region 570 (Prättigau–Davos, ~41 × 47 km).
- Nach Bounding-Box-Cleaning: **~60 Millionen Zeilen aus ~11 000 Flügen**.
- **Content-adressiertes Caching** in zwei Stufen (Clean + GPU-Subsample) — neuinvalidierung nur bei Config- oder Datenänderung; pro Flug genau eine Parquet-Row-Group plus Flight-Index für O(1)-Flugzugriff.

---

## Komponente 3 — Frontend (React / MapLibre)

**Rolle:** Interaktive Visualisierung der Vorhersagen und Flugdaten.

- **Stack:** React 19.1, TypeScript 5.8, Vite 7.1 (SWC), Mantine v8, AG Grid 35, **MapLibre GL 5.18 + react-map-gl 8 + deck.gl 9** (Aggregation/Geo/Mapbox-Layer), ECharts 6, Apache Arrow 21, axios, react-router-dom 7, react-resizable-panels.
- **Routen:**
  - `/` — Navigationsseite
  - `/thermal` — Backend-gerenderte Thermikkarte (iframe)
  - `/predict` — ML-Vorhersagekarte (iframe)
  - `/map` — Client-seitige MapLibre-Karte mit OpenTopoMap-Basemap, dynamisch vom Backend geholten Overlays (Bodenbeschaffenheit, Hangneigung, Exposition), **2D/3D-Geländeschalter** (MapLibre `raster-dem` Terrain-RGB, 1.5× Überhöhung), Slope- und Aspect-Slider, die die Kachel-URL parameterisieren (MapLibre lädt automatisch neu).
- **Container-Architektur:** Multi-Stage-Dockerfile. Dev-Stage mit Vite-HMR, Build-Stage für Produktion, nginx-Stage für Release. Der `docker-compose.prod.yml` referenziert das geteilte `../.env.production`.
- **Ausliefereffizienz:** Apache Arrow IPC für 3D-Trajektorien (`layers_3d`-Backend-App); Tile-basierte Layer für alles Raster-ähnliche.

---

## Komponente 4 — Docs (Obsidian → MkDocs)

**Rolle:** eine **aggregierte Dokumentations-Site**, die (a) eigenes kuratiertes Material aus einem Obsidian-Vault und (b) die `docs/`-Verzeichnisse aus Frontend- und Backend-Repo zusammenführt.

- **Autorenwerkzeug:** Obsidian, inklusive Wiki-Link-Syntax `[[…]]`, die via MkDocs-Plugin `roamlinks` für die statische Site aufgelöst wird.
- **Build:** MkDocs Material mit `awesome-pages` (automatische Navigationsstruktur aus `.pages`-Dateien, **ohne** manuelles YAML), `mermaid2` für Diagramme.
- **Aggregation:** eigene Python-Skripte (`stage_vault.py`, `fetch_sources.py`, `generate_navigation.py`) — **cross-platform**, kein rsync, kein Bash, funktioniert ohne WSL unter Windows. Der `.vaultignore` steuert, welche Vault-Inhalte in die öffentliche Site gelangen (private Protokolle, Drafts, Scratch-Verzeichnisse bleiben aussen vor).
- **Deployment:** GitHub Actions → rsync auf eine Subdomain (`docs.thermaltrack.*`) oder als Fallback `/docs` via nginx.

---

## Full-Stack-Integration

Vier Besonderheiten der Zusammenarbeit zwischen den Repos:

1. **Geteiltes `.env.production` auf dem Server.** Statt drei parallele Env-Dateien pro Service zu pflegen, liegt eine einzige Datei eine Ebene oberhalb der Repos, und jedes `docker-compose.prod.yml` referenziert sie. Pro Service dokumentiert die repo-eigene `.env.*.example` nur die relevanten Keys.
2. **Celery-Queue als Kommunikationskanal Backend ↔ ML.** Statt das ML-Modul als Python-Library in den Backend-Prozess zu importieren (was den Django-Worker schwergewichtig und GPU-abhängig machen würde), läuft der MAP-Solver als **separater Celery-Consumer** auf der `map`-Queue. Das Backend dispatcht `solve_heading_task`-Messages in den gleichen Redis-Broker, den Celery sowieso schon verwendet. GPU-Isolation, horizontale Skalierbarkeit und saubere Deploy-Trennung inklusive.
3. **IGC-Parser in `lib/igc/`, Django-frei.** Die gleiche Python-Bibliothek kann vom Django-Backend *und* vom ML-Trainingsjob importiert werden, ohne die Django-App-Registry zu initialisieren. Per Architektur-Lint enforced.
4. **Architektur-Lint im Backend.** `backend/lint/arch_lint.py` prüft bei jedem Commit 20+ Regeln: Import-Richtungen, Stub-Task-Verbote, Library-Purity (keine Django-Imports in `lib/`), Task-vs-Service-vs-Command-Layering. Verletzungen mit Ablaufdatum in `waivers.yaml`. Das hat die Codebase beim schnellen Feature-Wachstum strukturell stabil gehalten.

---

## Status — laufende Bachelorarbeit

**Dieses Projekt ist die aktive Bachelorarbeit, nicht ein abgeschlossenes Produkt.** Zeitlicher Stand:

- **September 2025** — Start, drei Repos initial angelegt (Backend, Frontend, Docs), Server-Setup, XContest-Webscraper als erste ingest-Quelle, wöchentliche Meetings mit Betreuer Karl Rege (dokumentiert im Obsidian-Vault).
- **Oktober–Dezember 2025** — Django-Apps `flights`, `weather`, `flight_analysis`, `grids`, `geolayers`, `machinelearning` entstehen; Daten-Pipeline RAW → CANONICAL → PRODUCTS wird implementiert, Architektur-Lints werden eingeführt. Frontend-Routen `/`, `/thermal`, `/predict`, `/map` entstehen.
- **Januar–Februar 2026** — Enrichment-Schicht (ICON-DREAM, ERA5, Sonden, Hessen-250m), Zarr-Tile-Renderer, 3D-Gelände über Terrain-RGB-Kacheln.
- **Seit Mitte März 2026** — eigenes ML-Repo (`ThermalTrackML`), dort entsteht zuerst der **MAP-Solver**, dann das **hierarchische Block-GAM**, zuletzt der **DAG-Refactor** für mehrere Ziele (`w` + `wind_u` + `wind_v`) in einem Trainingsdurchgang.

**Was steht / was noch läuft:**

- ✅ Ingest-Pipeline (Webscraper, IGC-Parser, Flight-Metadaten in PostGIS), Enrichment über externe Wetter- und Gelände-Quellen.
- ✅ MAP-Solver v1 (Heading-only) validiert gegen Dense-Debug-Pfad auf 1e-6°.
- ✅ Hierarchisches GAM trainierbar, Modellregistry und Inferenzserver laufen.
- 🔄 **In Arbeit:** MAP-Progression v2 (+Wind) → v3 (+Va) → v4 (+Kalibrierung); stagewise Co-Training von Flugzuständen und GAM (Stage A / Stage C alternierend); Gauss-Prozess auf den Residuen (Stage B) als zukünftige Erweiterung.
- 🔄 **In Arbeit:** Evaluierung und Ablationen gegen Baseline-Prädiktoren; Dokumentation der wissenschaftlichen Ergebnisse für die schriftliche Arbeit.

Ein offizieller Abgabetermin wird hier nicht genannt, weil er für die Darstellung nicht relevant und im Projektzeitplan noch justierbar ist. Nach Abschluss und Benotung ist geplant, ausgewählte Code-Teile und die schriftliche Arbeit verfügbar zu machen.

---

## Technologisches Profil auf einen Blick

| Ebene | Technologie |
|---|---|
| Sprachen | Python (~112 k LOC), TypeScript (~21 k LOC) |
| Web-Framework (Backend) | Django 5, Django REST Framework, GeoDjango |
| DB | PostgreSQL + PostGIS |
| Queue | Celery + Redis + Flower |
| Scraping | Selenium + undetected-chromedriver + patchright, Tor (`stem`), Multi-Account-Auth |
| Geospatial | PostGIS-Raster, Zarr-Stores, pyproj, DEM / Slope / Aspect / WorldCover, MapLibre-Raster-DEM |
| ML | PyTorch (GPU), numpy, scipy, eigene B-Spline / Basis-Infrastruktur, FastAPI-Inferenz |
| Frontend | React 19, Vite 7, Mantine 8, MapLibre GL 5, react-map-gl 8, deck.gl 9, AG Grid 35, ECharts 6, Apache Arrow |
| Docs | Obsidian → MkDocs Material + awesome-pages + roamlinks + mermaid2 |
| CI / Deploy | GitHub Actions (Frontend + Backend + Docs), docker-compose, geteiltes `.env.production`, SSL-Terminierung via nginx |
| Qualität | 117 pytest-Dateien, Golden-File-Regression, eigener Architektur-Lint mit 20+ Regeln |

---

## Screenshots

<!-- TODO: Bilder einfügen. Vorschläge:
  1. /map-Route mit aktiviertem 3D-Gelände + Slope-Overlay + Flug-Track
  2. Kachel-Vorhersage (FastAPI /predict/tile) als Thermik-Heatmap
  3. Partial-Dependence-Plot eines Blocks (source oder spatial_prior)
  4. Architektur-Diagramm (oben als ASCII drin — als Grafik sauberer)
-->

---

## Quellcode

**Hinweis:** Projekt ist Teil einer laufenden Bachelorarbeit an der ZHAW. Quellcode und wissenschaftliche Ergebnisse unterliegen bis zur Abgabe dem Datenschutz und sind nicht öffentlich. Nach Abschluss der Arbeit können ausgewählte Code-Beispiele und die schriftliche Arbeit auf Anfrage bereitgestellt werden.

---

## Reflexion

Zwei Entscheidungen, die sich rückblickend als wichtig herausgestellt haben:

**Multi-Repo statt Monorepo.** Die vier Repos haben **unterschiedliche Deploy-Lebenszyklen** — das Frontend baut in Sekunden, das Backend mit Geospatial-Dependencies in Minuten, das ML-Repo ist GPU-abhängig, die Docs-Site wird bei jedem Push unabhängig gerendert. Ein Monorepo hätte am Anfang schneller gefühlt; in der Praxis wäre jeder Push durch eine träge gemeinsame CI gelaufen, und der ML-Code hätte das Backend zum GPU-Image aufgebläht. Der Preis — geteiltes `.env.production` und die Celery-Queue als Bridge — war den Aufwand wert.

**Physikstruktur statt End-to-End-Deep-Net.** Der multiplikative GAM mit physikalisch motivierten Blöcken ist deutlich mehr Arbeit als ein generischer Regressor, der dieselben Features konsumiert. Aber: jeder Block hat **eigene Link-Funktion, eigenen Feature-Satz, eigene Lambda-Tuning-Knöpfe** — und das erlaubt gezielte Ablationen ("schalte den Feuchteblock aus, re-trainiere", "ersetze $A_\text{surf}$-Features durch WorldCover-only"), Partial-Dependence-Plots pro Block und eine Argumentationskette für die Arbeit, die über "das Modell hat den Loss minimiert" hinausgeht. Für eine Bachelorarbeit mit Begutachtung durch Domänen-Expertinnen und -Experten war das die richtige Investition.
