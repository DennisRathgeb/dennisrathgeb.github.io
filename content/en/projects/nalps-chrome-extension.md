---
layout: article
title: "NalpSolar BuildTrack — Chrome extension for a solar-park build"
date: 2026
description: "Chrome extension with SharePoint integration and a MapLibre map for real-time visualization of the NalpSolar solar-park construction site at STRABAG. Multi-shell architecture (Chrome, SPFx, PCF) with an integrated AI assistant."
image: /assets/images/projects/nalps-chrome-extension/cover.jpg
permalink: /en/projects/nalps-chrome-extension/
lang: en
key: project-nalps-chrome-extension
sidebar:
  nav: project-en
---

![NalpSolar BuildTrack – Chrome side panel with MapLibre map, table polygons and drill points on a swisstopo aerial basemap]({{ '/assets/images/projects/nalps-chrome-extension/cover.jpg' | relative_url }})

## Overview

**NalpSolar BuildTrack** is a Chrome extension in production use at **STRABAG** that drives site operations for the **NalpSolar** solar-park project (Axpo NalpSolar construction management). The extension lives in the Chrome side panel next to the project's SharePoint site and overlays its master data — assembly tables, drill points, adapter pieces ("Passstücke"), module-carrier stacks — as coloured polygons and points on a swisstopo aerial basemap.

The goal was to give the field teams a **spatial, real-time view of project data** without standing up a separate backend: the extension injects fetch calls directly into the user's authenticated SharePoint tab and piggy-backs on that session. The same shared codebase produces three host shells — Chrome extension, SharePoint Framework web part (SPFx), and a Power Apps Component Framework control (PCF) — so the same visualizations and workflows are available at several points of STRABAG's Microsoft 365 estate.

This is a solo build: architecture, domain model, SharePoint list schemas, stage derivation, rendering pipelines and the AI integration were all designed and implemented by me.

---

## Use context

- **Client / operator:** STRABAG, cost centre 256-RKDO, execution management for the Axpo NalpSolar project
- **Data source:** SharePoint Online in the Strabag BRVZ GmbH tenant, 20 SharePoint lists (tables, drill points, adapter pieces, modules, status, audit trail, …)
- **Coordinate system:** Swiss national grid LV95 (EPSG:2056), converted on the fly to WGS84 for MapLibre
- **Basemap:** swisstopo (aerial imagery + topographic map, WMTS)
- **Users:** planners and fitters on site; tablet and desktop workstations

---

## Architecture

Strict **four-layer architecture** whose boundaries are documented in `docs/guidelines/` and enforced in review:

```
Layer 4  UI              React + Mantine v7, MapLibre map, AG Grid
Layer 3  Composition     useDerivedData() — the one place queries meet domain logic
Layer 2  Domain          Types, stage derivation, selectors, geo pipelines
Layer 1.5 Agent          OpenAI Responses API infrastructure (see below)
Layer 1  Data Access     SP REST, Graph API, Power Automate, ApiBackend abstraction
```

### Multi-shell via an `ApiBackend` abstraction

Three production shells share the entire business logic in `src/`:

- **Chrome extension** (Manifest V3, side panel) — primary shell, injects fetch into the SharePoint tab via `chrome.scripting.executeScript({ world: 'MAIN' })`.
- **SPFx web part** — same UI embedded directly into SharePoint pages.
- **PCF control** — same components as a Canvas-App building block in Microsoft Dataverse / Power Apps.

Each shell registers a backend adapter on boot (`createChromeBackend`, `createSpfxBackend`, `createPcfBackend`, `createMockBackend`) against the shared `ApiBackend` interface. SPFx and PCF copy the `src/` tree via `scripts/sync-shared-src.js` on every build because their build chains can't traverse npm workspaces reliably.

### Config-driven by design

The whole system is built around **data, not code**:

- **Stage registry** (`src/domain/config/`) defines each workflow stage once: colour, label, order, icon. Adding a new stage = edit three files. Components read from the registry — no hard-coded colours or labels anywhere.
- **Map presets** (`src/map/config/map-presets.ts`): each page hands in a `DeepPartial<MapConfig>` that drives selection targets, toolbar entries, camera, interactions and layer colouring.
- **PCF release pipeline** (`pcf/components.json` + `scripts/release-pcf.js`): a new PCF or a new environment = one JSON entry. Build → package → import → publish runs through Makefile targets.

---

## Map stack

MapLibre GL 4 with a 15-layer stack:

- swisstopo WMTS as raster basemap (aerial or topographic, user-switchable)
- Generated GeoJSON sources for tables (polygons), drill points (points) and deviations (LineString SOLL→IST)
- **Ghost layers** for inactive features (grey outlines/labels under the main render) — controlled by two toggles
- **Feature-state–driven hover and selection** (rectangle + circle select)
- **Print layout** with its own layer configuration (different colours, flush labels, ghost tables, fit-padding)
- Colour schemes (`MapColorScheme`) allow colouring by aggregate stage, type, phase, adapter-piece status or drill-point stage without recomputing the geometry

Coordinates are transformed once via `proj4` from LV95 to WGS84, preserving the precision of the Swiss national grid.

---

## AI assistant (Layer 1.5)

The extension ships an **in-app AI chat built on the OpenAI Responses API**. The agent sits deliberately between Data Access and Domain (Layer 1.5) — tool handlers call `spQuery()` directly and read from the Zustand stores without importing React hooks.

**Key concepts:**

- **28+ tools in 9 groups** (analysis, map, inspect, documents, communication, sp_crud, automation, navigate, meta). Tools are pure functions routed through a middleware pipeline (input validation, output-size guard, null/metadata stripping, error enrichment).
- **Group-based prompt routing** — user input is scored against tool groups by keywords; only the relevant tools and context cards enter the turn.
- **11 context cards** (one always-on, ten adaptive) inject domain knowledge per query (entity guide, stage overview, coordinate system, SP field gotchas, workflows, email templates, flow management, document access).
- **Context compression** at turn-pair boundaries: once input exceeds 70 % of the model's context limit, the older portion is summarised by `gpt-4.1-mini` and the Responses API chain is reset. Deterministic `structuredTrim` fallback on summary failure.
- **Budget tracker** with per-model pricing and compression counter.
- **Planner subsystem** (`make_plan` / `update_task` / `replan`) with a reasoning model, auto-continue, auto-finalize on loop exit, and an inline plan card with live checkboxes in the chat.
- **Three subagent profiles** (`explore_sp`, `analyze`, `planner`) — short-lived, do not compress.
- **Chat artifacts:** map commands, custom grids, detail panels — produced by the agent and rendered in the right-hand artifact panel.
- **Quality gates** (`make lint-agent`): knowledge build, OpenAI tool-schema compliance, content quality (stale refs, duplicates, token budgets), routing-gap detection across 675 test prompts.

---

## Ops tooling

The repo is not only the app; it's the toolbox around the site:

- `cdp-sp-crud.py` — universal SharePoint CRUD via Chrome DevTools Protocol, with a production-write guard that refuses writes to non-`_COPY` lists unless `--i-know-this-is-production` plus an interactive confirmation is given.
- `sp-ingest.py` — 4-phase Excel → SharePoint pipeline, dry-run by default, `--apply` to write.
- `sp-snapshot.py` — JSON dump of all 20 SP lists into `temp/snapshots/<timestamp>/` as a baseline before bulk edits.
- `ingest-crosscheck.py` — Excel-vs-SharePoint diff report.
- `analyze-agent-logs.js` — aggregate analysis of AgentLog NDJSON files: tool failures, routing gaps, prompt distribution.

Everything is orchestrated by a single Makefile (`make dev`, `make build`, `make check`, `make test`, `make lint-agent`, `make release-pcf-prod COMPONENT=tal-pdf-list`, …).

---

## Tech stack (extract)

- **Language:** TypeScript 5.7 (strict mode)
- **Framework:** React 18, Mantine v7, i18next (DE primary, EN secondary)
- **Map:** MapLibre GL 4.7, proj4 (LV95 ↔ WGS84), swisstopo WMTS
- **State:** TanStack Query (server state), Zustand (UI state), IndexedDB (agent persistence)
- **Grid:** AG Grid Community 35 with a generic `NalpsGrid<T>` wrapper
- **Parsing / export:** `xlsx`, `mammoth` (DOCX), `pdfjs-dist`, `jspdf` + `jspdf-autotable`
- **Search:** Fuse.js
- **Integration:** SharePoint REST (including batch + version history), Microsoft Graph (Files, People, Calendar, Tasks, Search, Subscriptions), Power Automate (flow trigger + registry)
- **AI:** OpenAI Responses API with streaming, mock client for dev mode
- **Build & release:** CRA via Craco for the extension, Gulp for SPFx, pac CLI for PCF
- **Ops:** Python 3.14 in `.venv/`, Chrome with `--remote-debugging-port=9222` for the CDP scripts
- **Tests:** Jest via `react-scripts` — 193 tests across 15 suites

---

## Highlights

- **One source tree, three production shells** (Chrome, SPFx, PCF) via a cleanly drawn `ApiBackend` abstraction. A mock backend enables full development without Chrome or SharePoint access (`make dev`).
- **Config-driven everywhere that normally rots:** stages, colours, map presets, PCF releases, agent tools, context cards. A new stage / environment / tool is a config entry, not a code fork.
- **Precise geo model** with LV95↔WGS84 transformation, SOLL→IST deviation layers, and separate print and screen layout paths.
- **Fully integrated AI agent** with prompt routing, adaptive context cards, automatic context compression, budget tracking, a planner subsystem with live checkboxes, and 28+ validated tools — with its own quality-gate pipeline (`make lint-agent`).
- **Clean workflow:** 366 commits over two weeks in conventional-commit style, 193 Jest tests, strict layer boundaries, extensive blueprint documentation (14 module docs, ~6,400 lines).
- **Production guardrails:** production-write guards in the SP scripts, a `_COPY` table convention switchable via `.env`, an NDJSON audit store for every write intent, and a maintained Chrome Web Store publishing checklist.

---

## Screenshots

<!-- TODO: add screenshots. Suggested set:
  1. Chrome side panel with the map view (coloured tables, drill points, aerial basemap)
  2. Modulträger page with the stack wizard + map
  3. AI agent with a streamed response + map artifact + plan card
  4. Browser page with grid + filters + map in the 65/35 split
  5. Print preview of a module-carrier sign-off
-->

---

## Source

**Note:** This project is in active production use at STRABAG. The source code is confidential and not publicly available. Selected code samples can be shared on request.

---

## Reflection

The project was a deliberate bet on **"code that looks like configuration"**: the domain model (assembly tables, drill points, adapter pieces, module carriers) has enough regularity that practically everything — colours, layer order, tool availability, release targets, agent context — can be carried as data. That style only pays back if the boundaries are drawn hard; the four-layer guidelines with mandatory review kept the system coherent over 45 000 lines of TypeScript.

The AI agent was the technically hardest part. Tool routing, adaptive context, compression and the planner all have to compose cleanly, and the best investment was building a dedicated quality-gate pipeline early (`make lint-agent` with knowledge build, tool-schema validation, content lint and routing-gap tests). Regressions in agent behaviour show up in seconds rather than after a day of debugging.

The multi-shell architecture was the other big lesson: the decision to **sync** `src/` into the SPFx and PCF shells rather than link it feels inelegant at first, but it's the only robust solution given the quirks of those two build chains. It stays healthy behind a single `sync-shared-src.js` and blocks neither development nor release.
