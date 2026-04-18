---
layout: article
title: "NalpSolar BuildTrack — Chrome-Extension für Solarpark-Baustelle"
date: 2025
description: "Chrome-Extension mit SharePoint-Anbindung und MapLibre-Karte zur Echtzeit-Visualisierung der Solarpark-Baustelle NalpSolar bei STRABAG. Multi-Shell-Architektur (Chrome, SPFx, PCF) mit integriertem KI-Assistenten."
image: /assets/images/projects/nalps-chrome-extension/cover.jpg
permalink: /projekte/nalps-chrome-extension/
lang: de
key: project-nalps-chrome-extension
sidebar:
  nav: project-de
---

> Produktiv bei STRABAG eingesetzte Chrome-Extension, die die Stammdaten der Solarpark-Baustelle NalpSolar räumlich in einem Swisstopo-Luftbild visualisiert – inkl. integriertem KI-Agent.

**Jahr:** 2025–2026  ·  **Kontext:** STRABAG (Produktiveinsatz, Kostenstelle 256-RKDO)  ·  **Rolle:** Einzelentwicklung

![NalpSolar BuildTrack – Chrome Side Panel mit MapLibre-Karte, Tischpolygonen und Bohrpunkten auf Swisstopo-Luftbild]({{ '/assets/images/projects/nalps-chrome-extension/cover.jpg' | relative_url }})

**TL;DR**
- **Problem:** Baustellenteams ein räumliches Echtzeitbild der NalpSolar-Projektdaten geben, ohne eine eigene Backend-Infrastruktur aufzusetzen.
- **Meine Rolle:** Einzelentwicklung — Architektur, Domänenmodell, Multi-Shell-Release, Karten-Stack, KI-Agent, Ops-Tooling.
- **Ergebnis:** Produktiver Einsatz bei STRABAG; ein Quelltext → drei Shells (Chrome-Extension, SPFx, PCF); ~45 k LOC TypeScript, 193 Tests, 28+ Agent-Tools mit eigener Quality-Gate-Pipeline.
- **Stack:** TypeScript 5.7, React 18, Mantine v7, MapLibre GL 4, TanStack Query, Zustand, SharePoint REST + Graph API, OpenAI Responses API, Manifest V3, SPFx, PCF.

---

## Kontext

**NalpSolar BuildTrack** ist eine produktiv bei **STRABAG** eingesetzte Chrome-Extension zur Baustellensteuerung des Solarparks **NalpSolar** (Kraftwerksprojekt Axpo NalpSolar). Die Extension lebt als Chrome Side Panel neben der SharePoint-Site des Projekts und überlagert die Stammdaten der Anlage – Montagetische, Bohrpunkte, Passstücke, Modulträger-Stapel – als farbige Polygone und Punkte auf einem Swisstopo-Luftbild.

Ziel war, den Baustellenteams ein **räumliches Echtzeitbild** der Projektdaten zu geben, ohne eine eigene Backend-Infrastruktur aufzubauen: die Extension injiziert Fetch-Aufrufe direkt in den authentifizierten SharePoint-Tab der Benutzerin und nutzt so deren bestehende Session. Aus der gleichen geteilten Codebasis entstehen drei Host-Shells – Chrome-Extension, SharePoint Framework Web Part (SPFx) und Power-Apps-Komponente (PCF) – damit die gleichen Visualisierungen und Workflows an mehreren Stellen der Microsoft-365-Landschaft bei STRABAG verfügbar sind.

Das Projekt ist Einzelentwicklung: Architektur, Domänenmodell, SharePoint-Listenschemata, Stage-Herleitung, Leistungs-Pipelines und die KI-Integration wurden von mir konzipiert und umgesetzt.

![Vermessungsarbeiten auf der Solarpark-Baustelle NalpSolar – die Daten, die die Extension visualisiert, entstehen hier im Feld]({{ '/assets/images/projects/nalps-chrome-extension/site.jpg' | relative_url }})

---

## Einsatzkontext

- **Kunde / Betreiber:** STRABAG, Kostenstelle 256-RKDO, Projekt Ausführungsabwicklung Axpo NalpSolar
- **Datenbasis:** SharePoint Online im Tenant Strabag BRVZ GmbH, 20 SharePoint-Listen (Tische, Bohrpunkte, Passstücke, Module, Status, Audit-Trail, …)
- **Koordinatensystem:** Schweizer Landesvermessung LV95 (EPSG:2056), on-the-fly konvertiert auf WGS84 für MapLibre
- **Basemap:** Swisstopo (Luftbild + topografische Karte, WMTS)
- **Nutzer:** Planer und Monteure am Bau; Tablet- und Desktop-Arbeitsplätze

---

## Meine Rolle

Einzelentwicklung, end-to-end:

- **Architektur & Kern-Codebase:** 4-Schichten-Modell, `ApiBackend`-Abstraktion, Multi-Shell-Release (Chrome, SPFx, PCF) aus einem `src/`.
- **Domänenmodell:** Tische, Bohrpunkte, Passstücke, Modulträger-Stapel; Stage-Registry, Selektoren, Stage-Herleitung.
- **SharePoint-Integration:** 20 Listen-Schemata, REST-Batching + Versionshistorie, Microsoft-Graph-Integration, Power-Automate-Trigger.
- **Karten-Stack:** MapLibre GL 4, LV95↔WGS84-Transformation, Swisstopo-WMTS, Druck-Layout-Pfad, Ghost-Layer, Farbschemata.
- **KI-Agent (Layer 1.5):** 28+ Tools, adaptive Context-Cards, Kontext-Kompression, Planner-Subsystem, `make lint-agent`-Quality-Gates.
- **Ops-Tooling:** CDP-basierter SP-CRUD, Excel→SP-Ingest-Pipeline, SP-Snapshots, Agent-Log-Analyzer, Production-Write-Guards.
- **Release-Pipelines:** CRA/Craco (Chrome), Gulp (SPFx), pac CLI (PCF), Makefile als einzige Orchestrierung.

---

## Architektur

Strikte **4-Schichten-Architektur** mit in `docs/guidelines/` dokumentierten und im Review durchgesetzten Grenzen:

```
Layer 4  UI              React + Mantine v7, MapLibre-Karte, AG Grid
Layer 3  Composition     useDerivedData() – einzige Stelle, wo Queries auf Domain treffen
Layer 2  Domain          Typen, Stage-Herleitung, Selektoren, Geo-Pipelines
Layer 1.5 Agent          OpenAI-Responses-API-Infrastruktur (siehe unten)
Layer 1  Data Access     SP-REST, Graph API, Power Automate, ApiBackend-Abstraktion
```

### Multi-Shell über eine `ApiBackend`-Abstraktion

Drei produktive Shells teilen sich die gesamte Business-Logik in `src/`:

- **Chrome Extension** (Manifest V3, Side Panel) – primärer Shell, injiziert Fetch in den SharePoint-Tab via `chrome.scripting.executeScript({ world: 'MAIN' })`.
- **SPFx Web Part** – gleiche UI direkt in SharePoint-Seiten einbettbar.
- **PCF Control** – gleiche Komponenten als Canvas-App-Baustein in Microsoft Dataverse / Power Apps.

Jeder Shell registriert beim Start einen Backend-Adapter (`createChromeBackend`, `createSpfxBackend`, `createPcfBackend`, `createMockBackend`) gegen das gemeinsame `ApiBackend`-Interface. SPFx und PCF kopieren den `src/`-Baum über `scripts/sync-shared-src.js` bei jedem Build, weil deren Build-Chains npm-Workspaces nicht zuverlässig auflösen.

### Config-driven Design

Das ganze System ist auf **Daten statt Code** ausgelegt:

- **Stage-Registry** (`src/domain/config/`) definiert jede Workflow-Stufe einmal: Farbe, Label, Reihenfolge, Icon. Eine neue Stufe hinzufügen = 3 Dateien editieren. Komponenten lesen aus der Registry – keine hart kodierten Farben oder Labels.
- **Map-Presets** (`src/map/config/map-presets.ts`): jede Seite reicht ein `DeepPartial<MapConfig>` herein, das Selektionsziele, Toolbar-Einträge, Kamera, Interaktionen und Layer-Einfärbung steuert.
- **PCF-Release-Pipeline** (`pcf/components.json` + `scripts/release-pcf.js`): ein neuer PCF oder eine neue Umgebung = ein JSON-Eintrag. Build → Package → Import → Publish läuft über Makefile-Targets.

---

## Kartenschicht

MapLibre GL 4 mit einem 15-stufigen Layer-Stack:

- Swisstopo-WMTS als Raster-Basemap (Luftbild oder Karte, vom Benutzer umschaltbar)
- Generierte GeoJSON-Quellen für Tische (Polygone), Bohrpunkte (Points) und Abweichungen (LineString SOLL→IST)
- **Ghost-Layer** für nicht aktive Features (graue Umrisse/Labels unter der Hauptdarstellung) – über zwei Toggles steuerbar
- **Feature-State-gesteuertes Hover + Selektion** (rectangle + circle select)
- **Druck-Ansicht** mit eigener Layer-Konfiguration (andere Farben, bündige Labels, ghost tables, fit-padding)
- Farbschemata (`MapColorScheme`) erlauben Färbung nach Aggregatstufe, Typ, Phase, Passstück-Status oder Bohrpunkt-Stufe, ohne die Geometrie neu zu berechnen

Koordinaten werden einmalig über `proj4` von LV95 auf WGS84 gebracht; die Präzision der Schweizer Landesvermessung bleibt erhalten.

![Modulträger-Wizard: Tisch auf der Karte wählen, Reihenfolge festlegen, Aufbau dokumentieren – die farbigen Polygone zeigen den Baufortschritt pro Tisch]({{ '/assets/images/projects/nalps-chrome-extension/ui-wizard.png' | relative_url }})

---

## KI-Assistent (Layer 1.5)

Die Extension enthält einen in die App integrierten **KI-Chat auf Basis der OpenAI Responses API**. Der Agent ist bewusst zwischen Data Access und Domain angesiedelt (Layer 1.5) – Tool-Handler rufen `spQuery()` direkt auf und lesen aus den Zustand-Stores, ohne React-Hooks zu importieren.

**Schlüsselkonzepte:**

- **28+ Tools in 9 Gruppen** (analysis, map, inspect, documents, communication, sp_crud, automation, navigate, meta). Tools sind reine Funktionen, werden durch eine Middleware-Pipeline (Input-Validierung, Output-Size-Guard, Null/Metadaten-Stripping, Error-Enrichment) geroutet.
- **Gruppen-basiertes Prompt-Routing** – die Nutzereingabe wird per Keyword-Scoring auf Tool-Gruppen abgebildet. Nur die relevanten Tools und Kontextkarten kommen in den Turn.
- **11 Context Cards** (eine immer aktiv, zehn adaptiv) injizieren pro Anfrage Domänenwissen (Entitätsguide, Stage-Übersicht, Koordinatensystem, SP-Feldfallen, Workflows, E-Mail-Templates, Flow-Management, Dokumentzugriff).
- **Context Compression** an Turn-Paar-Grenzen: überschreitet der Kontext 70 % des Modell-Limits, wird der ältere Teil durch `gpt-4.1-mini` zusammengefasst und die Responses-API-Kette neu aufgesetzt. Deterministischer `structuredTrim`-Fallback bei fehlschlagender Zusammenfassung.
- **Budget-Tracker** mit modellspezifischen Preisen und Kompressionszähler.
- **Planner-Subsystem** (`make_plan` / `update_task` / `replan`) mit Reasoning-Modell, Auto-Continue, Auto-Finalize bei Loop-Exit und Inline-Plan-Card mit Live-Checkboxen im Chat.
- **Drei Subagent-Profile** (`explore_sp`, `analyze`, `planner`) – kurzlebig, komprimieren nicht.
- **Artefakte im Chat:** Map-Kommandos, benutzerdefinierte Grids, Detail-Panels – vom Agent erzeugt und im rechten Artefakt-Panel gerendert.
- **Quality Gates** (`make lint-agent`): Knowledge-Build, OpenAI-Tool-Schema-Compliance, Inhaltsqualität (stale refs, Duplikate, Token-Budgets), Routing-Gap-Erkennung über 675 Testprompts.

![KI-Agent beantwortet eine Baustellen-Frage, erzeugt eine E-Mail als Artefakt und hebt die betroffenen 196 Tische auf der Karte hervor]({{ '/assets/images/projects/nalps-chrome-extension/ui-agent.png' | relative_url }})

---

## Ops-Tooling

Das Repository versteht sich nicht nur als App, sondern als Werkzeugkasten rund um die Baustelle:

- `cdp-sp-crud.py` – universelles SharePoint-CRUD via Chrome DevTools Protocol, mit Production-Write-Guard (verweigert Schreibzugriffe auf nicht-`_COPY`-Listen ohne explizites `--i-know-this-is-production` und interaktiver Bestätigung).
- `sp-ingest.py` – 4-Phasen-Pipeline Excel → SharePoint, Dry-Run standardmässig, `--apply` für Schreibzugriff.
- `sp-snapshot.py` – JSON-Dump aller 20 SP-Listen in `temp/snapshots/<timestamp>/` als Baseline vor Bulk-Änderungen.
- `ingest-crosscheck.py` – Excel-vs-SharePoint-Diff-Report.
- `analyze-agent-logs.js` – aggregierte Auswertung von AgentLog-NDJSON-Dateien: Tool-Fehler, Routing-Gaps, Prompt-Verteilung.

Alles wird über ein einziges Makefile orchestriert (`make dev`, `make build`, `make check`, `make test`, `make lint-agent`, `make release-pcf-prod COMPONENT=tal-pdf-list`, …).

---

## Ergebnis & Impact

- **Ein Quelltext, drei produktive Shells** (Chrome, SPFx, PCF) über eine sauber definierte `ApiBackend`-Abstraktion. Mock-Backend ermöglicht komplette Entwicklung ohne Chrome- oder SharePoint-Zugriff (`make dev`).
- **Config-driven an jedem Ort, der sonst rot wird:** Stages, Farben, Map-Presets, PCF-Releases, Agent-Tools, Kontextkarten. Neue Stufe/Umgebung/Tool = Konfigurationseintrag, nicht Code-Fork.
- **Präzises Geo-Modell** mit LV95↔WGS84-Transformation, Abweichungslayern SOLL→IST, separaten Druck- und Bildschirm-Layoutpfaden.
- **Voll integrierter KI-Agent** mit Prompt-Routing, adaptiven Kontextkarten, automatischer Kontextkomprimierung, Budget-Tracking, Planner-Subsystem mit Live-Checkboxen und 28+ geprüften Tools – mit eigenem Quality-Gate-Pipeline (`make lint-agent`).
- **Saubere Arbeitsweise:** 366 Commits über zwei Wochen im Conventional-Commit-Stil, 193 Jest-Tests, strenge Layer-Grenzen, umfangreiche Blueprint-Dokumentation (14 Module, ~6 400 Zeilen).
- **Produktions-Guardrails:** Production-Write-Guards in den SP-Skripten, `_COPY`-Tabellen-Konvention per `.env`-Flag, Audit-Store (NDJSON) für jede Schreib-Intention, Chrome-Web-Store-Publikations-Checkliste gepflegt.

---

## Tech-Stack (Auszug)

- **Sprache:** TypeScript 5.7 (strict mode)
- **Framework:** React 18, Mantine v7, i18next (DE primär, EN alternativ)
- **Karte:** MapLibre GL 4.7, proj4 (LV95 ↔ WGS84), swisstopo WMTS
- **Datenhaltung:** TanStack Query (Server-State), Zustand (UI-State), IndexedDB (Agent-Persistenz)
- **Grid:** AG Grid Community 35 mit generischem `NalpsGrid<T>`-Wrapper
- **Parsing / Export:** `xlsx`, `mammoth` (DOCX), `pdfjs-dist`, `jspdf` + `jspdf-autotable`
- **Suche:** Fuse.js
- **Integration:** SharePoint REST (inkl. Batch + Versionshistorie), Microsoft Graph (Files, People, Calendar, Tasks, Search, Subscriptions), Power Automate (Flow Trigger + Registry)
- **KI:** OpenAI Responses API mit Streaming, Mock-Client für Dev-Mode
- **Build & Release:** CRA via Craco für die Extension, Gulp für SPFx, pac CLI für PCF
- **Ops:** Python 3.14 in `.venv/`, Chrome mit `--remote-debugging-port=9222` für die CDP-Skripte
- **Tests:** Jest via `react-scripts` – 193 Tests in 15 Suites

---

## Screenshots

### Browser-Seite – Grid, Filter und Karte im Split

![Hauptansicht: Tabelle der Tische mit Filter-Chips, Abweichungsdiagramm und Luftbild-Karte mit farbkodiertem Bauzustand]({{ '/assets/images/projects/nalps-chrome-extension/ui-main.png' | relative_url }})

### Vorbereitung – STRABAG-Primärkonstruktion auf Basis der SP-Daten

![Vorbereitung-Seite: Auswahl- und Freigabe-Workflow mit Masstoleranzen und eingebetteter Primärkonstruktions-Zeichnung]({{ '/assets/images/projects/nalps-chrome-extension/ui-pdf-export.png' | relative_url }})

### Druck-Pipeline – A4/A3-Export mit Titel, Tabelle und Legende

![Druckvorschau eines Modulträger-Abnahme-Protokolls im Querformat mit beschrifteten Polygonen, Legende und Tisch-Tabelle]({{ '/assets/images/projects/nalps-chrome-extension/ui-print-preview.png' | relative_url }})

![Druckdialog: Format, Orientierung, optionale Datentabelle mit Spaltenauswahl, Legende, Mail-Versand]({{ '/assets/images/projects/nalps-chrome-extension/ui-print-dialog.png' | relative_url }})

### Responsiv – Tablet- und Mobile-Arbeitsplätze am Bau

![Tablet-Ansicht: Tisch-Detailpanel mit Bohrpunkten und Filterdialog über der Karte]({{ '/assets/images/projects/nalps-chrome-extension/ui-tablet.png' | relative_url }})

<img src="{{ '/assets/images/projects/nalps-chrome-extension/ui-mobile.png' | relative_url }}" alt="Mobile-Ansicht: vollständige Karte mit Legende und Such-/Filterleiste" style="max-width: 320px;">

---

## Quellcode

**Hinweis:** Projekt aktiv im produktiven Einsatz bei STRABAG. Quellcode unterliegt dem Datenschutz und ist nicht öffentlich. Auf Anfrage können ausgewählte Code-Beispiele bereitgestellt werden.

---

## Reflexion

Die Arbeit war eine bewusste Wette auf **„Code, der wie Konfiguration aussieht"**: das Domänenmodell (Montagetische, Bohrpunkte, Passstücke, Modulträger) hat genug Regelmässigkeit, um praktisch alles – Farben, Layer-Reihenfolge, Tool-Verfügbarkeit, Release-Ziele, Agent-Kontext – als Daten zu führen. Ein solcher Stil zahlt sich nur aus, wenn die Grenzen hart gezogen sind; die 4-Schichten-Guidelines mit Review-Pflicht hielten das System über 45 000 Zeilen TypeScript hinweg kohärent.

Der KI-Agent war der technisch anspruchsvollste Teil. Tool-Routing, adaptiver Kontext, Kompression und Planner greifen ineinander, und die beste Investition war früh in eine eigene Quality-Gate-Pipeline (`make lint-agent` mit Knowledge-Build, Tool-Schema-Validierung, Inhalts-Linter und Routing-Gap-Tests) zu stecken. So werden Regressionen im Agentenverhalten in Sekunden sichtbar statt tagelang debuggt.

Besonders lehrreich war die Multi-Shell-Architektur: die Entscheidung, `src/` in die SPFx- und PCF-Shells zu **synchronisieren** statt zu linken, wirkt auf den ersten Blick unelegant, ist aber die einzige robuste Lösung gegenüber den Eigenheiten der beiden Build-Chains. Sie lässt sich mit einem einzigen `sync-shared-src.js`-Skript aufrechterhalten und blockiert weder Entwicklung noch Release.
