# Lanterne System Architecture Context


---


This document represents the canonical technical architecture of the Lanterne system.

Sources included:

• architecture
• analysis model
• design specifications
• architectural decision records


---

## Source File: docs/02-architecture/arch-001-data_model.md

# Lanterne Data Model

## Core Design Principle

The data model separates four concerns that must never be collapsed:

1. **Canonical identity** — what the route is geometrically
2. **Normalized facts** — what OSM and spatial analysis says about each slice
3. **Analysis outputs** — what Lanterne computed from those facts
4. **Ride-instance conditions** — what the route will feel like at a specific time

See `RECOMMENDED_SCHEMA_SHAPE.md` for the full migrated schema.

---

## Route Identity Layer

### `canonical_routes`
The authoritative geometry record for a route corridor. One row per unique road experience. Sources and variants attach to canonical routes — they do not create new ones.

### `imported_routes`
Source artifacts (RWGPS, GPX, RUSA, manual) resolved to a canonical route. Preserves provenance without polluting canonical identity.

### `external_route_catalog`
Routes fetched from external platforms before import resolution.

### `event_routes`
Ordered references from events onto canonical routes. Implements ADR-031 (multi-day events as ordered references onto canonical geometry).

---

## Analysis Layer

### `route_slices`
The atomic analysis unit (ADR-020). Physical subdivision of a canonical route into ~200–500m segments broken at every scoring-relevant semantic boundary. Ordered, stable, versioned.

### `route_slice_osm_facts`
Normalized OSM variables per slice (ADR-021, ADR-022). One row per slice per OSM extraction pass. Road environment, bike support, surface, terrain, infrastructure flags, and traceability fields.

### `route_slice_support_facts`
Corridor and proximity-derived support context (ADR-019). Settlement proximity, service proximity, water, food, lodging, medical, bailout access.

### `route_analysis_runs`
Container for any scoring pass. Tracks what version of logic ran over which route under which assumptions (stable_route or ride_instance family).

### `route_slice_analysis`
Per-slice scoring outputs for an analysis run. Safety Score, Traffic Index, Bike Support Index, Remoteness Index, Surface Quality Index, Fatigue Index, Descent Risk Index.

### `route_analysis_summary`
Route-level rollup per analysis run. Fast retrieval for cards and summary views.

---

## Expedition Layer

### `route_expeditions`
Durable multi-day progress record for one rider on one route. Survives browser close, app crash, phone restart, GPS disable, and overnight charging. The system of record for where a rider is in a larger journey. See ADR-034 and DS-034.

### `route_expedition_windows`
Bounded detailed-analysis windows for one expedition. Defines what subsection of the full route receives full corridor analysis, heatmap rendering, and navigation context at any given time.

### `route_expedition_events`
Sparse append-only audit trail for expedition transitions and recovery checkpoints. Supports crash recovery and resume debugging without storing every GPS tick.

---

## Comparative Traffic Context Layer (schema exists, canonical mapper not yet built)

### `canonical_segments`
Stable directed segment identity. The long-lived entity that traffic behavior facts, cohort memberships, and future observations attach to. Keyed by surrogate UUID + deterministic identity scaffold. See ADR-033 and DS-013.

### `route_segment_instances`
Route-analysis-local mapping layer. One row per segment occurrence in a specific analysis pass. `canonical_segment_id` is nullable in v1 — set to `unresolved` until the canonical mapper is built.

### `segment_behavior_inputs`
Canonical per-segment traffic behavior facts. Keyed by `canonical_segment_id`. Not yet populated — deferred until canonical mapper exists.

### `traffic_behavior_baselines`
Regional and contextual comparison priors. Starts mostly empty; populates as baseline data is sourced.

### `cohorts`
Typed catalog of comparative groups a canonical segment can belong to. V1 seed rows for geography, urbanicity, and road class are present.

### `segment_cohort_memberships`
Many-to-many association between canonical segments and cohorts. Not yet populated — deferred until canonical mapper exists.

### `segment_observations`
Raw evidence landing zone for future Varia radar and rider-contributed data. Stub table — no ingestion pipeline yet.

---

## Planned (not yet migrated)

### `ride_instance_runs`
A route at a specific start time. Contextual overlay layer.

### `ride_instance_slice_conditions`
Time-aware environmental outputs per slice. Wind, temperature, precipitation, light state, glare flag, sun azimuth, moon phase live here — not in stable analysis tables.

### `route_slice_overrides`
Community/admin corrections to OSM-derived facts.

### `route_slice_effective_facts`
Materialized view combining OSM facts + support facts + approved overrides into current resolved truth for scoring.

---

## Hard Rules

**Do not collapse:**
- Slice geometry and OSM facts (separate tables)
- OSM facts and support/proximity facts (separate tables)
- Stable analysis and ride-time conditions (separate tables)
- Canonical identity and source provenance (separate tables)
- Expedition state and live ride session state (different durability contracts)

**Do not write:**
- Wind, temperature, precipitation into `route_slice_analysis`
- Itinerary or event semantics into `canonical_routes`
- Analysis outputs into `route_slices`
- Raw OSM tags as the only record — always normalize into columns
- Expedition-critical progress only into ephemeral browser memory

---

## Provenance

Routes may come from:
- RWGPS import
- GPX upload
- RUSA permanent routes
- Manual drawing
- Ride history ingestion
- Future external sources

Source provenance is preserved in `imported_routes` and never allowed to determine canonical route identity. Two routes representing the same road corridor resolve to the same canonical route regardless of source.


---

## Source File: docs/02-architecture/arch-002-system_architecture.md

# Lanterne System Architecture

## Overview

Lanterne analyzes cycling routes by combining:

- Route geometry
- OpenStreetMap data
- Elevation data
- Weather forecasts
- Astronomical calculations

The system transforms raw geographic data into **route intelligence for long-distance riders**.

---

## Core System Components

### 1. Route Ingestion

Routes enter the system through several pathways:

- Manual drawing
- RWGPS import
- GPX upload
- RUSA permanent route import
- Ride history ingestion

All routes are normalized into a canonical geometry format and resolved against existing canonical routes to avoid duplication (ADR-026).

---

### 2. Route Storage

Routes are stored with strict separation of:

- Canonical geometry identity
- Source provenance
- Route variants
- Slice-level analysis
- User save relationships

A route is primarily the line on the map and its canonical identity — not the source it came from or the analysis attached to it. See `DATA_MODEL.md`.

---

### 3. Analysis Engine

The analysis engine processes route geometry and generates indices on **small internal slices** (ADR-020), not large averaged segments.

**Stable analysis inputs:**
- OSM road attributes
- Elevation profiles
- Geographic context
- DOT/HPMS traffic data

**Stable analysis outputs:**
- Safety Score (Traffic Index + Bike Support Index)
- Remoteness Index
- Surface Quality Index
- Fatigue Index
- Descent Risk Index

**Contextual (ride-time) outputs — computed separately:**
- Wind (bearing-relative)
- Temperature
- Precipitation
- Light state (daylight / twilight / night)
- Sun glare risk
- Moonlight conditions

Stable and contextual outputs are never mixed. See ADR-023.

---

### 4. Environmental Modeling

Environmental conditions are modeled along the route timeline using estimated rider arrival time per slice (ADR-024).

Systems include:

- Solar position and sun glare detection (ADR-010)
- Moon phase and cloud cover (ADR-009)
- Weather forecast integration
- Time-of-day traffic multipliers

---

### 5. Expedition Layer

For ultra-distance routes (>400 miles, >8,000 GPX points, or >500 roads/mile density), Lanterne uses a four-layer expedition model (ADR-034):

```text
master route
    ↓
route expedition       ← durable multi-day progress record
    ↓
active analysis window ← bounded working set (default 250 miles)
    ↓
live ride session      ← transient runtime state


---

## Source File: docs/02-architecture/arch-003-project_map.md

# Lanterne Project Map

## Purpose

This document is a high-level map of the product so future work has a stable reference point.

It explains what Lanterne is, how data flows through it, where major systems live, how route intelligence is produced, and how the product should feel.

---

## 1. What Lanterne Is

Lanterne is a route intelligence system for long-distance cyclists.

It is built for people riding in conditions where ordinary route tools break down:

- Long rural routes
- Overnight rides
- Self-supported riding
- Bikepacking and gravel contexts
- Remote areas with no services
- Traffic-stress tradeoffs
- Weather and light transitions
- Multi-day and multi-week expeditions

Lanterne is not mainly a ride recorder. It is a route understanding and pre-ride decision tool — and for ultra-distance riders, a durable expedition companion.

---

## 2. Core User Promise

Before a rider leaves home, Lanterne should help them answer:

- How traffic-safe is this route?
- How remote is it really?
- How hard will it be?
- What will the weather and light feel like?
- Where are the dangerous or notable parts?
- How does this route compare to another option?

During a multi-day ride, Lanterne should also answer:

- Where was I when I stopped?
- What section am I analyzing next?
- How do I resume from here?

---

## 3. System Overview

Lanterne consists of six major layers:

1. **Ingestion**
2. **Storage**
3. **Analysis**
4. **Expedition model** (for ultra-distance routes)
5. **Presentation**
6. **Decision support**

---

## 4. End-to-End Flow

```
Route source
    ↓
Normalization / canonical route resolution
    ↓
Stored route + provenance
    ↓
Windowing decision (full vs expedition model)
    ↓
Stable route analysis (on active window)
    ↓
Ride-time conditions analysis
    ↓
Map / drawer / cue-sheet presentation
    ↓
Rider decision
```

**For ultra-distance routes (>400 miles / >8K GPX points / >500 roads/mile density):**

```
Master route
    ↓
Route expedition (durable multi-day progress)
    ↓
Active analysis window (bounded working set, default 250 miles)
    ↓
Live ride session (transient runtime state)
```

The rider thinks in terms of one route and one expedition. The system manages bounded windows internally without fragmenting route identity.

---

## 5. Ingestion Layer

Routes can enter the system from:

- Manual route drawing
- RWGPS URL import
- GPX upload
- RUSA permanent route import
- Ride-history ingestion
- Future: additional sources

**Ingestion goals:**
- Normalize geometry
- Preserve provenance
- Avoid duplicate routes via canonical resolution (ADR-026)
- Support derivations and saved versions

---

## 6. Storage Layer

The storage layer separates the core route from everything attached to it.

```
canonical_route         ← the line and its identity
    ├ imported_routes   ← where it came from
    ├ route_variants    ← meaningful branches
    ├ route_slices      ← atomic analysis units
    │    ├ osm_facts    ← extracted variables
    │    └ analysis     ← computed intelligence
    ├ rollups           ← rider-facing summaries
    └ user_routes       ← personal relationships
```

A route is primarily the line on the map and its canonical identity. It is not the source it came from, the full analysis payload, or the rider's save record.

---

## 7. Analysis Layer

The analysis engine operates on **small internal slices** (ADR-020), not giant averaged segments, because truth changes along the route.

### Analysis families

**Safety** (narrow definition — motor vehicle risk only)
- Safety Score
- Traffic Index
- Bike Support Index

**Route Reality** (stable, not time-dependent)
- Remoteness Index
- Surface Quality Index
- Fatigue Index
- Descent Risk Index

**Conditions** (time-dependent, computed per ride plan)
- Wind
- Temperature
- Precipitation
- Light

**Light / Sky signals**
- UV halo
- Sun glare warning (ADR-010)
- Moon phase (ADR-009)
- Cloud overlay

---

## 8. Stable vs Contextual Analysis

| Layer | Derived from | Cached? |
|-------|-------------|---------|
| **Stable route intelligence** | Geometry, OSM, elevation, geography | Yes — long-lived |
| **Contextual ride-time intelligence** | Start time, pace, forecasts, solar/lunar timing | No — recomputed per ride plan |

This split reduces recomputation and keeps the architecture clean. Stable and contextual outputs live in separate tables and must never be mixed. See ADR-023.

---

## 9. Expedition Layer

For ultra-distance routes, the analysis working set is bounded. The expedition model (ADR-034) separates:

| Layer | Durability | Purpose |
|-------|-----------|---------|
| Master route | Permanent | Full journey identity |
| Route expedition | **Durable** — survives all interruptions | Multi-day progress record |
| Active analysis window | Session-scoped | Bounded working set (default 250 miles) |
| Live ride session | **Transient** | Current outing state |

**Windowing is triggered when any of these are true:**
- Route > 400 miles
- GPX > 8,000 points
- Estimated road density > 500 roads/mile

Expedition progress survives browser close, phone restart, GPS disable, and overnight charging. Live session state does not. These have different durability contracts by design.

---

## 10. Presentation Layer

The UI turns complex route analysis into something riders can read quickly.

**Main UI surfaces:**
- Map overlays and safety heatmap
- Analysis drawer and score explanation
- Cue-sheet / swipe views
- Ride conditions panel
- Saved route and history views
- Expedition resume card (on reopen when active expedition exists)

**Core presentation principles:**
- Show the important thing first
- Let users drill into detail
- Use icons where possible
- Use tooltips for short plain-English meaning
- Never turn the app into a data cockpit

---

## 11. Environmental Light System

One of the most distinctive parts of Lanterne is that it treats sun and moon as part of route intelligence.

**Day:** Sun icon, UV halo, glare detection (ADR-010)

**Night:** Moon phase, cloud cover overlay (ADR-009)

**Why this matters:** Long-distance riders care about darkness timing, glare windows, moonlit riding, exposed all-day sunlight, and overnight route character. This is part of what gives Lanterne its identity.

---

## 12. Product Modes / Profiles

Indices keep the same meaning across profiles. What changes is emphasis, ordering, explanatory framing, and weighting for presentation.

| Profile | Emphasis |
|---------|---------|
| Road | Safety Score, Traffic Index, Bike Support Index, Fatigue |
| Bikepacking / Gravel | Surface Quality, Remoteness, Fatigue, Temperature, Precipitation |

Urban may be added later but the center of gravity remains long-distance riding.

---

## 13. Decision Support Layer

The ultimate job of the system is to help riders make better choices:

- Choose one route over another
- Adjust start time to avoid glare or poor light
- Identify dangerous sections before encountering them
- Understand whether a route becomes harder at night
- Decide if a route is too remote for the current plan
- Resume a multi-day expedition after an overnight stop

---

## 14. Guiding Architectural Rules

1. Keep Safety Score narrow — motor vehicle risk only
2. Keep indices distinct from one another
3. Compute on small slices, present with human-readable summaries
4. Separate route identity from analysis output
5. Preserve provenance
6. Use structured columns for core concepts; JSON only for bounded flexibility
7. Prefer route intelligence over dashboard clutter
8. Build for riders who go long
9. Expedition state is durable; live session state is transient — never conflate them

---

## 15. Simplified Mental Model

```
A route is the line.
Sources tell where the line came from.
Analyses tell what Lanterne thinks about the line.
Conditions tell what the ride may feel like right now.
An expedition tells where the rider is in the larger journey.
The UI turns all of that into decisions a rider can act on.
```

---

## 16. What Makes Lanterne Special

Lanterne should feel like it was made by someone who has actually ridden:

- Into a brutal headwind
- Under a full moon at 2am
- Through dangerous glare at dawn
- On remote roads with no bailout for 80 miles
- Through the night when ordinary route tools stop helping
- On a TransAm or 1200K where losing context overnight means starting from scratch

That lived truth should remain visible in every major system decision.


---

## Source File: docs/02-architecture/arch-004-system_guide.md

# Lanterne System Guide

_Internal Documentation — Updated 2026-03-24 (v3)_

---

## 1. Overview

### What Lanterne Is

Lanterne is a mobile-first Progressive Web App that provides cyclists with segment-level safety analysis of their routes. Users upload a GPX file or create a route on-map, and the platform analyzes every road segment for risk factors — speed, traffic volume, shoulders, bike infrastructure, railroad crossings, bridge hazards, and left turns — producing a composite safety grade (A+ through F) and a color-coded heatmap overlay.

### The Problem

Cyclists lack granular, road-level safety data. Existing tools show bike lanes but don't quantify risk. Lanterne answers: "How dangerous is this specific road, given real traffic data, infrastructure, and hazard exposure?"

### Core Workflow

```
GPX Upload → Corridor Fetch → Road Matching → Data Enrichment → Hazard Detection → Safety Scoring → Visualization
```

### What Makes It Unique

1. **Client-side analysis** — all computation runs in the browser; infrastructure cost is ~$0.01 per cold-cache route.
2. **Multi-source data fusion** — OSM road geometry + federal HPMS traffic + state DOT attributes + micro-hazard detection from OSM tags.
3. **Shared geographic tile caching** — every user's analysis warms a global tile cache for future users in the same geography.
4. **Truth-segment scoring** — scoring operates on canonical segments broken at every semantic boundary, not visual approximations.
5. **Two-pass forensic matching** — coarse matching followed by targeted forensic re-analysis of suspicious zones.

---

## 2. Architecture

### Computation Model

All CPU-intensive work — GPX parsing, spatial indexing, road matching, forensic analysis, scoring, heatmap building, cue generation — runs in the browser. The backend is a data-and-proxy layer: it stores caches/user data and proxies rate-limited external APIs. No analysis runs server-side.

### Component Map

```
┌─────────────────────────────────────────────────────────────────┐
│  USER DEVICE (Browser)                                          │
│                                                                 │
│  ┌─ Ingestion ─────────────────────────────────────────────┐    │
│  │  GPX Parser (gpx.ts) → Route Hash (route-cache.ts)      │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Corridor Builder ──────────────────────────────────────┐    │
│  │  Tile Keys (corridor.ts) → Cache Lookup → Overpass Fetch│    │
│  │  → CorridorGraph adjacency model (corridor-graph.ts)    │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Analysis Engine (Two-Pass) ────────────────────────────┐    │
│  │  Spatial Grid Index → Window Matcher → Pass 1 Capture   │    │
│  │  → Forensic Matcher → Boundary Refinement (Pass 2)      │    │
│  │  → Transition Chain → Matcher Invariant Checks          │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Enrichment ────────────────────────────────────────────┐    │
│  │  HPMS Client → DOT Client → Hazard Detector             │    │
│  │  POI Subsystem (independent parallel stream)             │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Scoring & Output ─────────────────────────────────────┐     │
│  │  Safety Scoring → Heatmap Builder → Cue Generation      │    │
│  │  → Map Card Store → Corridor Reveal Animation           │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Route Editing ─────────────────────────────────────────┐    │
│  │  Detour System → CorridorGraph local pathfinding         │    │
│  │  → OSRM routing → Undo/Redo stack                       │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Navigation Engine ─────────────────────────────────────┐    │
│  │  GPS duty cycling → Corridor snapping → Cue detection    │    │
│  │  → Off-route detection                                   │    │
│  └──────────────────────────────────────────────────────────┘    │
│  ┌─ Presentation ─────────────────────────────────────────┐     │
│  │  Leaflet Map │ Safety Card │ Cue Sheet │ Lanterne Orb   │    │
│  │  Analysis Progress │ Drawers │ Inspector Panels          │    │
│  └──────────────────────────────────────────────────────────┘    │
└───────────────────┬─────────────────────────────────────────────┘
                    │ Supabase JS SDK
┌───────────────────▼─────────────────────────────────────────────┐
│  EDGE FUNCTIONS (9 total)                                       │
│  overpass-proxy (dual-server failover)                          │
│  hpms-proxy │ dot-proxy │ dot-aadt-proxy                       │
│  check-subscription │ create-checkout │ customer-portal         │
│  admin-users │ admin-manage                                     │
└───────────────────┬─────────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────────┐
│  DATABASE (Lovable Cloud / Supabase)                            │
│  ┌─ Cache Layer ───────────────────────────────────────────┐    │
│  │  osm_road_tile_cache (0.05° grid, 2yr TTL)                       │    │
│  │  hpms_osm_road_tile_cache (0.05° grid + state code, 1yr TTL)     │    │
│  │  route_cache (hash-keyed, versioned via data_version)    │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─ User Data ─────────────────────────────────────────────┐    │
│  │  route_history │ profiles │ user_roles                   │    │
│  │  user_events │ user_usage                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─ Model & Safety ───────────────────────────────────────┐     │
│  │  safety_model_versions │ safety_model_factors            │    │
│  │  route_hazard_detections │ route_perf_events             │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─ Payments ──────────────────────────────────────────────┐    │
│  │  subscription_grants │ promo_codes │ promo_redemptions    │    │
│  └─────────────────────────────────────────────────────────┘    │
│  Auth (Supabase Auth)                                           │
└─────────────────────────────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────────┐
│  EXTERNAL APIs                                                  │
│  Overpass API (primary + fallback servers) — OSM road geometry   │
│  HPMS Federal API — traffic volumes (AADT)                      │
│  State DOT APIs (22 verified states) — road attributes          │
│  OSRM (public demo server) — turn-by-turn routing               │
│  Nominatim — geocoding search                                   │
│  Open-Elevation — elevation profiles (unreliable)               │
│  Stripe — subscription payments                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Caching Architecture

| Cache | Key | TTL | Scope | Contents |
|-------|-----|-----|-------|----------|
| `osm_road_tile_cache` | 0.05° lat/lon grid (SW corner) | ~2 years | Global (shared across all users) | OSM road geometry + tags as JSON |
| `hpms_tile_cache` | 0.05° grid + state code | ~1 year | Global | HPMS traffic volume segments |
| `route_cache` | Deterministic hash (start/end coords rounded to ~11m, distance, 5 sampled points) | Until `data_version` bump | Global | Completed scoring results (not road geometry — too large) |

**Version invalidation:** A `data_version` integer (currently 2) is stored with each cache entry. Bumping this constant in `route-cache.ts` transparently invalidates all stale entries without migration.

---

## 3. Analysis Pipeline

### Pre-Analysis: Geometry Scan

Before matching begins, two preparatory scans run:

- **Track Geometry Zones** (`track-geometry-zones.ts`): Identifies curves, grades, and geometric features along the GPX track that affect matching behavior.
- **Cue Hint Zones** (`cue-hint-zones.ts`): Pre-samples areas near turns and intersections for higher-precision matching in cue-relevant regions.

### Step 1 — GPX Ingestion

**Purpose:** Parse raw GPX XML into structured points.
**Input:** `.gpx` file via drag-drop or file picker.
**Output:** `GpxRoute { points[], totalDistanceMi, cuePoints[], bounds }`.
**Code:** `gpx.ts → parseGpx()`.

### Step 2 — Route Hashing & Cache Check

**Purpose:** Generate a deterministic cache key; skip analysis on hit.
**Input:** Parsed points + total distance.
**Output:** Hash string. On cache hit (same hash + current `data_version`), scoring results are returned immediately — skip to Step 12.
**Code:** `route-cache.ts → computeRouteHash()`, `getCachedRoute()`.

### Step 3 — Corridor Tile Generation

**Purpose:** Divide route into a grid of independently fetchable tiles.
**Input:** Route points (sampled every `points.length/500` entries).
**Output:** Array of tile keys (SW corner coordinates at 0.05° intervals).
**Code:** `corridor.ts → routeTileKeys()`.
**Note:** TILE_SIZE of 0.05° ≈ 3.5mi lat × 2.5mi lon at 39°N.

### Step 4 — Tile Cache Lookup

**Purpose:** Batch-query database for already-cached tiles.
**Input:** Tile keys.
**Output:** Cached road geometry for warm tiles; list of cold tile keys.
**Code:** `corridor.ts → getCachedTiles()` (single Supabase batch query).

### Step 5 — Overpass Fetch

**Purpose:** Download OSM road geometry for uncached tiles.
**Input:** Cold tile keys.
**Output:** `SpeedRoad[]` with tags (name, highway type, maxspeed, surface, lanes, cycleway, shoulder).
**Code:** `corridor.ts → fetchTileWithRetry()` via `overpass-proxy` edge function. Dual-server failover with automatic retry. Results upserted to `osm_road_tile_cache` asynchronously. Roads are deduplicated by OSM way ID across tile boundaries.

### Step 5b — Corridor Graph Construction

**Purpose:** Build a client-side adjacency model for local pathfinding.
**Input:** All fetched corridor roads.
**Output:** `CorridorGraph { nodes (intersections), edges (road segments) }`.
**Code:** `corridor-graph.ts → buildCorridorGraph()`. Built once per corridor load; reused for all detour evaluations without external API calls.

### Step 6 — Spatial Indexing

**Purpose:** Enable O(1) road lookups during matching.
**Input:** Deduplicated corridor roads.
**Output:** `RoadGridIndex` — a spatial hash grid with 220m cells.
**Code:** Built inline in `route-analysis.ts`.
**Note:** The indexer interpolates along road segments in sub-cell steps to ensure every cell containing road geometry is populated, preventing "misses" where a segment crosses a cell without a vertex..

### Step 7 — Road Matching (Pass 1: Road Capture)

**Purpose:** Snap each GPX sample to the nearest corridor road.
**Input:** GPX points + spatial index.
**Output:** Ordered sequence of matched road segments with transition points.
**Code:** `route-analysis.ts` (Pass 1), `window-matcher.ts → windowMatchRoads()`.

**Matching rules:**
- Two-tier rejection threshold: **60m** for unnamed roads/paths; **~100m** for named motor roads (secondary, residential, etc.)
- Rejection triggers a metadata-aware null-gap fill that prioritizes named neighbors (Named motor > Unnamed motor > Named path > Unnamed path)
- **Suspicion scan** identifies incoherent zones: non-touching handoffs, null gaps, weak margins, heading discontinuities

### Step 8 — Forensic Re-Analysis

**Purpose:** Surgically re-analyze suspicious zones with dense sub-sampling.
**Input:** Suspicious zone boundaries from Pass 1.
**Output:** Corrected road assignments in those zones.
**Code:** `forensic-matcher.ts → runForensicPipeline()`.
**Note:** Uses reduced continuity bias (0.05) and "Synthetic Sample Anchors" to represent short segments between 200m samples. Only suspicious zones are re-analyzed — not the entire route.

### Step 9 — Boundary Refinement (Pass 2)

**Purpose:** Align road transitions to real intersections; attach traffic data.
**Input:** Matched road sequence from Pass 1 + forensic corrections.
**Output:** Refined transitions, HPMS traffic volumes, DOT road attributes per segment.
**Code:**
- `boundary-refinement.ts → computeBoundaryRefinements()`, `applyBoundaryRefinements()`
- `boundary-snapping.ts → snapBoundaries()`
- `transition-chain.ts → computeTransitionChain()`
- `hpms.ts` (via `hpms-proxy`) — federal traffic volumes
- `dot-enrichment.ts` (via `dot-proxy`/`dot-aadt-proxy`) — state DOT road attributes

**DOT coverage:** State DOT AADT is authoritative; 22 verified states covering ~65% of US miles. Matching uses geometry-aware rules: 50m for line segments, 200m for point-geometry stations.

### Step 10 — Hazard Detection

**Purpose:** Identify micro-hazards from OSM tags (no additional Overpass calls).
**Input:** Corridor roads already in memory.
**Output:** `SegmentHazard[]` with type and severity (1–3).
**Code:** `hazards.ts → detectHazardsFromRoads()`.

**Detected hazard types:**
| Hazard | Severity | Detection |
|--------|----------|-----------|
| Railroad crossing | 2–3 | `railway=level_crossing` |
| Metal grate bridge | 3 | `bridge + surface=grate/grid/metal_grid` |
| Metal plate bridge | 2 | `bridge + surface=metal/steel` |
| Cattle guard | 2 | `barrier=cattle_grid` |
| Single-lane underpass | 2 | `tunnel=yes + lanes=1 or maxwidth<4` |
| Covered bridge | 1 | `covered=yes or bridge:structure=covered` |
| No-shoulder bridge | 2 | `bridge + no cycling infra + narrow` |

### Step 11 — Segment Scoring

**Purpose:** Compute per-segment and aggregate route risk.
**Input:** Enriched truth segments.
**Output:** `TotalRisk`, `RiskPerMile`, `SafetyScore` (0–100), `LetterGrade` (A+ to F).
**Code:** `safety-scoring.ts → computeRouteSafetyScore()`.

**V2 Scoring Model:**

The core risk for each segment is computed as:

```
rawRisk = (W_SPEED × speedRisk + W_TRAFFIC × trafficRisk + W_RAIL × railRisk) × segmentLength
```

Where `W_SPEED=0.60`, `W_TRAFFIC=0.30`, `W_RAIL=0.10` (sum to 1.0).

Then modifiers are applied:
- **Bike infrastructure** — multiplicative factor (0.25 for protected track, 0.40 buffered lane, 0.70 painted lane, 0.90 shared lane, 1.0 none). At speeds >40 mph, even protected infra cannot reduce below 0.50×.
- **Shoulder credit** — subtractive, capped at 40% of raw risk. 0.1 per side per mile, bonus for width >1.5m.
- **Left turn penalty** — flat per-event cost (weight 0.21), not distance-proportional.

The aggregate `RiskPerMile` is transformed to a 0–100 score via a logistic curve (midpoint 2.5, steepness 1.4).

### Step 11b — Post-Analysis Validation

**Purpose:** Verify matching quality and emit diagnostics.
**Code:**
- `matcher-invariants.ts → runInvariantChecks()` — validates matched sequence consistency
- `analysis-diagnostics.ts → emitAnalysisOutputDiag()` — summary diagnostics
- `divergence-audit.ts → emitDivergenceAudit()` — first-divergence diagnostic, contamination audit

### Step 12 — Cue Sheet Generation

**Purpose:** Produce turn-by-turn navigation instructions.
**Input:** Matched road sequence with heading changes.
**Output:** `CueEntry[]` with road names, distances, turn directions.
**Code:** `topology-cues.ts → generateTopologyCues()`, `cue-hint-zones.ts`.

### Step 13 — Cache Write & History Save

**Purpose:** Persist results for future cache hits and user recall.
**Input:** Scoring result + route hash.
**Output:** Upsert to `route_cache` (if match quality sufficient). Save to `route_history` for authenticated user.
**Code:** `route-cache.ts → setCachedRoute()`.
**Note:** Road geometry is NOT stored in route_cache (was causing 33MB+ payloads). Only scoring results are cached.

### Step 14 — Visualization

**Input:** Truth segments, safety result, cue sheet.
**Output:** Heatmap overlays, grade card, cue sheet table, corridor reveal animation.
**Code:** `heatmap/builder.ts`, `RouteMap.tsx`, `useCorridorReveal.ts`.

**Heatmap architecture:** Truth segments → display segments via zoom-banded merging strategies (Low ≤12, Mid 13–14, High ≥15) using worst-case risk scoring. A Strava-style overlap rule (±1 coordinate extension) eliminates visual seams from browser rasterization.

**Corridor reveal:** After analysis completes, a 2.2-second staged animation illuminates nearby corridor roads (near → far → hold → fade) before leaving only the selected route.

### Parallel: POI Enrichment

POI enrichment is **not sequential** — it runs as an independent parallel streaming pipeline.

**Code:** `pois/index.ts` and submodules.
**Architecture:** Non-blocking, category-independent batching via Overpass with a 90-day tile cache. Uses a Centerline-First Scheduler with two-pass sweep. Categories yielding 0 results after 3 batches are auto-terminated. Max 2 concurrent Overpass requests. POIs are filtered by progress distance to correctly handle out-and-back/loop routes.

### Parallel: IO Metrics

Throughout the pipeline, `IoMetrics` (in `route-analysis.ts`) tracks: corridor tiles total/cached/fetched, enrichment fetch counts, forensic zone counts, HPMS/DOT tile cache hit rates, and log event counts by category. These are emitted as `[IO-BREAKDOWN]` and `[PERF-SUMMARY]` console logs (gated behind `IO_DEBUG` and `PERF_DETAIL` flags).

---

## 4. Data Model

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `osm_road_tile_cache` | Cached Overpass road geometry by 0.05° grid tile | `tile_key`, `roads` (JSONB), `fetched_at` |
| `hpms_tile_cache` | Cached HPMS traffic data by tile + state | `tile_key`, `state_code`, `data` (JSONB) |
| `route_cache` | Cached analysis results by route hash | `route_hash`, `safety_result` (JSONB), `data_version`, `hit_count`, `last_accessed` |
| `route_history` | Per-user saved routes and detours | `user_id`, `route_data`, `name`, `created_at` |
| `safety_model_versions` | Scoring model version history | `id`, `version`, `description`, `created_at` |
| `safety_model_factors` | Per-version factor weights | `version_id`, `factor_name`, `weight` |
| `route_hazard_detections` | Micro-hazard occurrences per analysis | `route_id`, `hazard_type`, `location`, `severity` |
| `route_perf_events` | Analysis performance telemetry | `route_hash`, `stage`, `duration_ms` |
| `profiles` | User profile data | `id` (refs `auth.users`), `email`, `display_name` |
| `user_roles` | Role assignments (admin/moderator/user enum) | `user_id`, `role` |
| `user_events` | Analytics events (fire-and-forget) | `user_id`, `event_type`, `payload`, `created_at` |
| `user_usage` | Monthly usage counters | `user_id`, `month`, `gpx_uploads`, `logins` |
| `subscription_grants` | Manual admin subscription grants | `user_id`, `granted_by`, `tier` |
| `promo_codes` | Promotional codes | `code`, `tier`, `expires_at` |
| `promo_redemptions` | Promo code redemption tracking | `user_id`, `code`, `redeemed_at` |
| `route_expeditions` | Durable multi-day expedition progress per user/route | `user_id`, `route_id`, `expedition_status`, `last_confirmed_route_mile`, `active_window_index` |
| `route_expedition_windows` | Bounded analysis windows per expedition | `expedition_id`, `window_index`, `core_start_mile`, `core_end_mile`, `load_start_mile`, `load_end_mile`, `window_status` |
| `route_expedition_events` | Sparse audit trail for expedition transitions | `expedition_id`, `event_type`, `route_mile`, `point_index` |
| `canonical_segments` | Stable directed segment identity (mapper not yet built) | `id`, `start_anchor_key`, `end_anchor_key`, `semantic_signature`, `is_active` |
| `route_segment_instances` | Route-local segment occurrences (canonical_segment_id unresolved in v1) | `route_id`, `segment_index`, `canonical_segment_id`, `match_method` |
| `cohorts` | Typed comparative group catalog | `cohort_type`, `cohort_key`, `name` |
| `traffic_behavior_baselines` | Regional traffic behavior comparison priors (mostly empty) | `geography_level`, `geography_key`, `baseline_passes_per_mile` |

**Cache sharing:** `osm_road_tile_cache` and `route_cache` are anonymous and global. User A fetching tiles in Austin warms the cache for User B riding through Austin months later. `route_history` is per-user. Expedition tables are per-user.

---

## 5. Features

### Core Analysis

| Feature | Description | Key Code |
|---------|-------------|----------|
| **Safety Scoring** | 0–100 score with A+–F grade. V2 model: speed/traffic/rail core weights + multiplicative bike infra + shoulder credit + left turn penalty | `safety-scoring.ts` |
| **Heatmap** | Speed-colored road overlays with zoom-banded truth-to-display rendering | `heatmap/builder.ts`, `RouteMap.tsx` |
| **Corridor Analysis** | Fetches and indexes the broader road network; builds CorridorGraph for local pathfinding | `corridor.ts`, `corridor-graph.ts` |
| **Micro-Hazard Detection** | 7 hazard types from OSM tags (railroad crossings, metal grate bridges, cattle guards, covered bridges, etc.) | `hazards.ts` |
| **Forensic Matching** | Dense sub-sampling re-analysis of suspicious zones | `forensic-matcher.ts` |

### Route Interaction

| Feature | Description | Key Code |
|---------|-------------|----------|
| **Detour Editing** | Drag-to-reroute with ghost handles (0.5mi), 6–8 candidate waypoints, undo/redo, brevet compliance | `detour-routing.ts`, `useDetourHistory.ts`, `DetourDeltaPanel.tsx` |
| **Route Creation** | Tap-to-place waypoints with live OSRM routing, real-time distance/elevation metrics | `useRouteCreation.ts` |
| **Cue Sheet** | Turn-by-turn from topology-based cue generation, with print/export | `topology-cues.ts`, `CueSheet.tsx` |
| **POI Enrichment** | ~20 categories (gas, food, water, bike shops) with detour distances and budget allocation | `pois/index.ts`, `PoiToggles.tsx` |
| **Route History** | Per-user saved/analyzed routes and detours | `RouteHistory.tsx`, `route_history` table |

### Navigation & GPS

| Feature | Description | Key Code |
|---------|-------------|----------|
| **GPS Tracking** | Live blue dot with accuracy circle and cumulative distance | `useGpsTracking.ts` |
| **GPS Look-Ahead** | Predictive road/POI loading ahead of GPS position | `useGpsLookAhead.ts` |
| **Navigation Engine** | Corridor-locked navigation with GPS duty cycling, corridor snapping, cue detection, off-route detection | `navigation/engine.ts` |

### Platform

| Feature | Description | Key Code |
|---------|-------------|----------|
| **Onboarding** | Multi-row swipeable flow with inline auth, safety-grade mockup, feature showcase | `OnboardingGate.tsx` |
| **Subscription Gating** | Free vs Nerd tier with Stripe checkout, admin grants, promo codes | `SubscriptionGate.tsx`, `TierPicker.tsx` |
| **PWA Install** | Home Screen prompt, standalone refresh button (iOS-only) | `AddToHomeScreenPrompt.tsx`, `StandaloneRefreshButton.tsx` |
| **Backend Resilience** | Auth timeout → cached subscription fallback → banner with retry | `AuthContext.tsx`, `BackendUnavailableBanner.tsx` |
| **Brevet Mode** | Randonneuring control checkpoint constraints for detour compliance | `brevet.ts`, `brevet-constraints.ts` |

---

## 6. UI System

### State Architecture

| System | Purpose | Code |
|--------|---------|------|
| **AuthContext** | User session, subscription status, backend-down detection, cached fallback (4hr TTL) | `contexts/AuthContext.tsx` |
| **LayoutContext** | Drawer occupancy state: `leftDrawerOpen`, `rightDrawerOpen`, `bottomDrawerOpen`, `topDrawerOpen`, `anyRightPanelOpen`, `anyDrawerOpen`, `bottomHeight` (collapsed/category/full) | `contexts/LayoutContext.tsx` |
| **UnitsContext** | Imperial/metric preference | `contexts/UnitsContext.tsx` |
| **LanternState** | Visual state machine (idle/analyzing/complete) with one-shot feedback pulses (route-loaded, route-edited, detour-applied, score-complete). Drives CSS glow/flicker classes. | `hooks/useLanternState.ts` |
| **Orb Control Router** | Maps orb gestures to actions — separate from visual state | `lib/orb-control-router.ts` |
| **Map Card Store** | Centralized overlay manager with standardized dismissal and stacking | `lib/map-card-store.ts` |

### Z-Index Layer Model

The UI uses a 5-tier z-index system defined in `ui-layers.ts`:

| Tier | Range | Examples |
|------|-------|---------|
| 0 — Map | 0–100 | Map canvas, MapIdleOverlay |
| 1 — Floating controls | 200–1020 | RoadInfoHud, MapControls, instrument tags, PoiStatusPanel |
| 2 — Drawers | 500–630 | Bottom/side/top drawers and their handles |
| 3 — Elevated floating | 700–800 | DetourDeltaPanel, AnalysisProgress, LanterneOrb |
| 4 — Overlays | 900–950 | Mismatch overlay, InstallNudgeBubble |
| 5 — Modals | 9000–9500 | SpotlightOverlay, AddToHomeScreenPrompt |

### Map System

Built on **Leaflet** via `RouteMap.tsx` (~3,000 lines — see Technical Debt). Layers include: base tiles, speed-colored heatmap overlays, GPX route polyline, POI markers (clustered via `usePoiClustering.ts`), ghost edit handles, detour candidates, GPS blue dot, corridor reveal animation, and road info cards.

### Drawer Architecture

- **TopActionDrawer** — persistent "liquid glass" top bar: road stats, search, upload GPX, create route
- **RouteAndAnalysisDrawer** — primary bottom drawer: analysis results, route history, POI toggles
- **CueDrawer** — cue sheet bottom drawer
- **StopsLayersDrawer** — POI category/layer settings
- **RouteBottomDrawer** — POI peek/expanded (note: operates outside centralized `activeDrawer` state)

### Analysis Progress

`AnalysisProgress.tsx` implements a paced progress engine that decouples UI progress from backend milestones. It enforces minimum dwell times per stage band, runs a burndown animation at completion, and auto-retracts. Designed to feel premium regardless of actual analysis speed.

### Debug/Inspection Panels

`SegmentInspectorDrawer`, `CandidatesDrawer`, `FragmentsDrawer`, `RawRoadsDrawer`, `CandidateAuditPanel`, `TurnPointAuditPanel`, `MatchedRoadSequencePanel`, `RawNearbyRoadsAuditPanel`, `FragmentContinuityPanel` — all gated behind debug flags.

---

## 7. Performance & Scaling

| Lever | Mechanism | Impact |
|-------|-----------|--------|
| **Tile cache reuse** | Global 0.05° grid, ~2-year TTL | Overpass calls approach zero for populated US regions |
| **Route cache reuse** | Hash-keyed, versioned | Popular routes (club rides, Strava segments) analyzed once globally |
| **Client compute** | Zero server CPU for analysis | Cost scales with users' devices, not infrastructure |
| **Spatial hash grid** | 220m cells with segment interpolation | O(1) road lookups; sub-minute analysis on mobile devices |
| **POI streaming** | Non-blocking category-independent batching, auto-termination | No single category blocks others; watchdog timer prevents hangs |
| **Corridor cap** | MAX_CORRIDOR_TILES = 200 | Prevents runaway fetches on transcontinental routes |
| **Adaptive corridor** | Density-based width reduction (250m → 60m) | Urban routes shrink corridor to prevent memory exhaustion; reuses guardrail density logic |
| **Road density guardrail** | 500 roads/mile normalized threshold | Warning-only; never blocks results from rendering |
| **Analysis timeout** | MAX_ANALYSIS_RUNTIME_MS = 120,000 | Client-side watchdog prevents infinite analysis |
| **GPX limit** | MAX_GPX_POINTS = 50,000 | Prevents browser memory exhaustion |

**Cost at scale:**
- Per cold-cache route: ~$0.01 (edge function invocations + Supabase writes)
- 10K MAU: ~$50–80/mo (Supabase Pro + edge function overages)
- 100K MAU: Primary cost is Supabase storage growth (osm_road_tile_cache); self-hosted Overpass eliminates external API dependency

**Primary bottleneck:** Overpass API rate limits for cold geographic regions. Mitigated by tile cache (each tile fetched at most once every ~2 years). Escape valve: self-hosted Overpass instance.

---

## 8. Admin Tools

| Tool | Location | Purpose |
|------|----------|---------|
| **User Management** | `/admin` | List users, grant/revoke subscriptions, manage promo codes |
| **Safety Model Console** | `/admin/safety-model` | Edit scoring factor weights per version, preview impact via Route Simulator |
| **Hazard Dashboard** | `/admin/safety-model` | Track micro-hazard frequency from `route_hazard_detections` for calibration |
| **Model Versioning** | `/admin/safety-model` | Version scoring models via `safety_model_versions` + `safety_model_factors` |
| **Performance Dashboard** | `PerfDashboard.tsx` | Analysis timing telemetry from `route_perf_events` |
| **Dev Tuning Panel** | Floating panel (debug flag gated) | Runtime parameter tuning for developers |
| **Debug Flags** | `localStorage.DEBUG_FLAGS` | 13 flags: `PERF_DETAIL`, `BOUNDARY_DEBUG`, `FORENSIC_DEBUG`, `ROAD_MATCH_DEBUG`, `SHORT_ROAD_DEBUG`, `OVERLAY_DEBUG`, `IO_DEBUG`, `SCORING_DEBUG`, `UI_DEBUG`, `BIKE_INFRA_DEBUG`, `USE_TOPOLOGY_CUES`, `TRANSITION_DEBUG`, `DRY_RUN_SIGNUP` |
| **Dry-Run Signup** | Admin Settings tab | Intercepts auth calls, logs to console instead of committing to database |

---

## 9. Limitations

| Constraint | Value | Notes |
|------------|-------|-------|
| Max GPX points | 50,000 | Hard limit |
| Max route length | 500 miles (soft) | Routes above this trigger windowed expedition mode (ADR-034); hard cap removed |
| Max corridor tiles | 200 | ~550 km² coverage cap |
| Analysis timeout | 120 seconds | Client-side watchdog |
| Overpass API | Public, no SLA | Dual-server failover; tile cache is primary mitigation |
| OSRM | Public demo server, no SLA | Used for detours and route creation only |
| Supabase query limit | 1,000 rows default | Must paginate or use RPC for large result sets |
| HPMS/DOT coverage | ~22 US states | ~65% of US miles; remaining states fall back to OSM speed tags |
| Open-Elevation | Public, unreliable | Elevation data may be unavailable |
| Nominatim | 1 req/sec, requires User-Agent | Geocoding rate limit |

### Known Technical Debt

- `RouteMap.tsx` is a ~3,000-line monolith with 70+ props managing all modes via internal conditionals
- Route registration / Supabase history-save logic is duplicated across 4 callsites
- `RouteBottomDrawer` operates outside the centralized `activeDrawer` / LayoutContext governance
- Right-side panel visibility managed by 6 independent booleans instead of a discriminated union
- `ManualSpanRange` type duplicated in `gpx.ts` and `useRouteCreation.ts`
- Stale root-level `detour-routing.ts` file persists alongside the canonical `src/lib/detour-routing.ts`
- `onReanalyze` prop in `RouteAndAnalysisDrawer` is a dead prop (not wired to a handler)

---

## 10. Future Extensions

| Extension | Architectural Hook |
|-----------|-------------------|
| **AI route optimization** | Scoring engine + CorridorGraph + OSRM are composable; an optimizer can iterate detour candidates against the scoring function without external dependencies |
| **Predictive crash modeling** | `route_hazard_detections` accumulates micro-hazard data globally; ML models can train on this corpus |
| **Community hazard reports** | `hazards.ts` and `_future/hazards.ts` contain schema scaffolding for Waze-style hazard confirmations |
| **Segment analytics** | Truth segments + `route_perf_events` enable per-road aggregate risk statistics across all users |
| **Safety factor tuning** | `safety_model_versions` + `safety_model_factors` + admin console already support versioned weight iteration without code changes |
| **Turn-by-turn navigation** | `navigation/engine.ts` is a full corridor-locked engine with GPS duty cycling, corridor snapping, cue detection, and off-route detection — ready for UI integration |
| **Expedition resume UX** | `route_expeditions` table exists; Phase 1 plumbing shipped; Phase 2 resume card with mismatch handling is next |
| **Windowed long-route analysis** | ADR-034 defines the four-layer model (master route → expedition → window → session); window trigger logic and expedition creation are in Phase 1 |
| **Local rerouting** | `CorridorGraph` adjacency model enables graph-based detour generation without any external API call |
| **International expansion** | Tile-based architecture is geography-agnostic; HPMS/DOT enrichment degrades gracefully to OSM-only outside the US |


---

## Source File: docs/02-architecture/arch-005-recommended_schema_shape.md

# Lanterne Schema — Current State
*Last updated: 2026-03-24*
*Status: Reflects actual migrated database state as of Phase 1 + Expedition + Comparative Traffic Context*

---

## Overview

The schema is organized into five hard separations:

1. **Canonical identity** — what the route is geometrically
2. **Normalized facts** — what OSM and spatial analysis says about each slice
3. **Analysis outputs** — what Lanterne computed from those facts
4. **Expedition state** — durable multi-day rider progress (ADR-034)
5. **Ride-instance conditions** — what the route will feel like at a specific time (Phase 2)

Do not collapse these. The architecture depends on this separation.

---

## Existing Tables (pre-Phase 1)

### `canonical_routes`

The authoritative geometry record for a route corridor.

Columns as migrated:
- `id` uuid PK
- `canonical_name` text
- `geometry` geometry(LineString, 4326) — **authoritative geometry for all downstream analysis**
- `geometry_hash` text
- `geometry_fingerprint` text — direction-normalized, 100-point, SHA-256
- `fingerprint` text
- `distance_km` numeric
- `length_m` numeric
- `elevation_gain_m` numeric
- `created_at` timestamptz

**Notes:**
- `geometry_fingerprint` uses direction normalization (ST_Reverse if needed), 100 resampled points, SHA-256 via pgcrypto
- Both `canonical_routes` and `imported_routes` must use the identical fingerprint formula or matching breaks
- A unique constraint on `geometry_fingerprint` should be added before the next ingestion run
- `distance_km` is stored here; downstream tables use `distance_m` / `length_m` — be consistent when querying across tables

---

### `imported_routes`

Source artifacts that have been resolved to a canonical route.

Key columns:
- `id` uuid PK — **this is the join key**, not `source_route_id`
- `canonical_route_id` uuid FK → `canonical_routes.id`
- `source_route_id` bigint — the external platform integer ID (e.g. RWGPS)
- `source_platform` text
- `geom` geometry — the imported geometry before canonicalization
- `geometry_fingerprint` text
- `geometry` jsonb — legacy field, prefer `geom` for spatial work

**Critical join note:** `organization_published_routes.imported_route_id` stores the UUID of `imported_routes.id`, NOT the integer `source_route_id`. Any join on `source_route_id` will return zero rows.

---

### `external_route_catalog`

Catalog of routes fetched from external platforms before import resolution.

---

### `event_routes`

Ordered references from events onto canonical routes. Partially implements ADR-031. The full event/day model (`events`, `event_days`, `event_route_part_segments`) is not yet migrated..

---

### `app_config`, `dot_tile_cache`, `hpms_tile_cache`

Infrastructure/ops tables. Not part of the route intelligence model.

---

### `hazard_comments`, `hazard_confirmations`

Community-reported hazard layer. Separate from OSM-derived slice facts by design.

---

### `hud_metric_layout`, `map_messages`, `map_message_replies`

UI/ride-mode tables. Not part of the route intelligence model.

---

## Phase 1 Tables (migrated 2026-03-22)

### `route_slices`

The atomic analysis unit (ADR-020, ADR-011).

Columns:
- `id` uuid PK
- `canonical_route_id` uuid NOT NULL FK → `canonical_routes.id`
- `sequence` integer NOT NULL — ordered position, must be > 0
- `start_distance_m` / `end_distance_m` numeric NOT NULL
- `length_m` numeric NOT NULL
- `start_lat` / `start_lng` / `end_lat` / `end_lng` double precision NOT NULL
- `geometry` geometry(LineString, 4326) NOT NULL
- `bearing_deg` numeric — dominant bearing, used for wind calculation
- `slice_boundary_reason` text[] — why this slice boundary was created
- `slice_builder_version` text NOT NULL DEFAULT '1.0'
- `created_at` timestamptz NOT NULL

Constraints: UNIQUE (canonical_route_id, sequence), CHECK sequence > 0, CHECK length_m > 0

**slice_boundary_reason values:** distance_threshold, road_class_change, intersection_boundary, surface_change, bike_infra_change, bridge_tunnel_transition, grade_transition, light_timing_change, remoteness_transition

---

### `route_slice_osm_facts`

Normalized OSM variable registry per slice (ADR-021, ADR-022). One row per slice per OSM extraction pass.

Key columns: road_class, speed_limit_value, speed_environment_class, car_speed_value, car_speed_source, lane_count_value, lane_count_class, bike_facility_type, shoulder_class, shoulder_width_value, surface_type, surface_quality_class, offroad_context_class, elevation_m, grade_percent, descent_flag, curvature_class, bridge_flag, tunnel_flag, rail_crossing_flag, raw_osm_tags_json, normalized_variable_evidence_json, slice_variable_confidence_json.

Constraints: UNIQUE (route_slice_id, osm_snapshot_version)

**Note:** `light_state` is NOT stored here. Light is ride-time-dependent and lives in `ride_instance_slice_conditions` (Phase 2).

---

### `route_slice_support_facts`

Corridor and proximity-derived support context (ADR-019). Split from OSM facts because these come from spatial search, not OSM tag extraction.

Key columns: settlement_proximity_m, service_proximity_m, water_proximity_m, food_proximity_m, lodging_proximity_m, medical_proximity_m, bailout_access_proximity_m, support_context_class, evidence_json, confidence_json.

Constraints: UNIQUE (route_slice_id, support_snapshot_version)

---

### `route_analysis_runs`

Container for any scoring pass (ADR-018). Tracks exactly what version of logic ran over which route under which assumptions.

Key columns: canonical_route_id, analysis_family (stable_route|ride_instance), analysis_version, mode_profile (road|bikepacking_gravel), osm_snapshot_version, support_snapshot_version, weather_snapshot_version, status (pending|running|complete|failed), is_current.

Partial unique index on (canonical_route_id, analysis_family, mode_profile) WHERE is_current = true.

---

### `route_slice_analysis`

Per-slice scoring outputs for an analysis run. Stable indices only.

Key columns: analysis_run_id, route_slice_id, safety_score, traffic_index, bike_support_index, remoteness_index, surface_quality_index, fatigue_index, descent_risk_index, index_breakdown_json, flags_json, confidence_json.

Wind, temperature, precipitation do **not** belong here.

---

### `route_analysis_summary`

Route-level rollup per analysis run. Fast retrieval for cards and drawer summaries.

Key columns: analysis_run_id, canonical_route_id (denormalized for convenience), safety_score through descent_risk_index, worst_mile_json, worst_sections_json, summary_breakdown_json.

---

## Expedition Tables (migrated 2026-03-24 — ADR-034, DS-034)

### `route_expeditions`

Durable multi-day progress record for one rider on one route. The database source of truth for where the rider is in a larger journey. Survives all session and device interruptions.

Key columns: user_id, route_id, expedition_status (planned|active|paused|completed|abandoned), entry_mode, detail_mode (windowed|full), target_detail_miles (default 250), max_detail_miles (default 400), window_overlap_miles (default 10), preload_trigger_miles (default 25), start_route_mile, last_confirmed_route_mile, last_matched_point_index, last_matched_lat/lon, last_match_confidence, last_progress_source, last_progress_at, active_window_index, next_window_index, route_total_miles, route_point_count.

Partial unique index: (user_id, route_id) WHERE expedition_status IN ('planned', 'active', 'paused') — enforces one open expedition per user/route.

**v1 migration note:** References `route_history(id)`, not a canonical route. Future migration should reference `canonical_route_id`.

**Windowing trigger (any condition forces windowed mode):**
- Route distance > 400 miles
- GPX point count > 8,000
- Estimated road density > 500 roads/mile

---

### `route_expedition_windows`

Bounded detailed-analysis windows for one expedition. Core span = rider-visible section. Load span = actual analysis working set including overlap.

Key columns: expedition_id, window_index, core_start_mile, core_end_mile, load_start_mile, load_end_mile, core/load point indexes, window_status (planned|queued|analyzing|ready|active|completed|failed|stale), analysis_cache_key, route_cache_key.

Constraints enforce load span always contains core span.

---

### `route_expedition_events`

Sparse append-only audit trail. Not a GPS log — sparse checkpoints only.

Key columns: expedition_id, event_type (started|resumed|paused|progress_checkpoint|window_queued|window_ready|window_activated|window_failed|manual_reposition|completed|abandoned), source_type, route_mile, point_index, lat/lon, window_index, payload.

**Progress checkpoint cadence:** Write when rider has moved ≥ 2 miles AND ≥ 10 minutes since last checkpoint.

---

## Comparative Traffic Context Tables (migrated 2026-03-24 — ADR-032, ADR-033, DS-013)

> **Note:** These tables exist. The canonical segment mapper is not yet built. `canonical_segment_id` in `route_segment_instances` will be NULL (match_method = 'unresolved') until the mapper is implemented.

### `canonical_segments`

Stable directed segment identity. The long-lived entity that traffic facts, cohort memberships, and observations attach to.

Key columns: network_source, direction, segmentation_schema_version, start_anchor_key, end_anchor_key (5dp snapped coordinates, format `{lat},{lon}`), start/end_anchor_type, start/end_osm_node_id (nullable enrichment), geometry_hash_normalized, semantic_signature, is_active, superseded_by_id.

---

### `route_segment_instances`

Route-analysis-local mapping layer. One row per segment occurrence per analysis pass.

Key columns: route_id, analysis_id, segment_index, local_segment_key, local_geometry, canonical_segment_id (nullable — unresolved in v1), match_method (exact|near_exact|new|unresolved), match_confidence.

---

### `segment_behavior_inputs`

Canonical per-segment traffic behavior facts. Keyed by canonical_segment_id. **Not yet populated** — deferred until canonical mapper exists.

Key columns: inferred_posted_speed_mph, inferred_aadt, inferred_lane_count, inferred_shoulder_width_m, inferred_bike_facility_class, predicted_passes_per_mile, predicted_vehicle_speed_mph, predicted_driver_slowdown_mph, confidence_overall.

---

### `traffic_behavior_baselines`

Regional comparison priors. **Starts mostly empty.** Not used to rescale Safety Score.

Key columns: geography_level, geography_key, road_class, urbanicity_class, baseline_passes_per_mile, baseline_vehicle_speed_mph, p25/p50/p75/p90 percentiles.

Unique index uses `coalesce()` expressions — implemented as a unique index, not a table constraint.

---

### `cohorts`

Typed catalog of comparative groups. V1 seed rows for geography (US, HI, NJ, CA, TX, NY), urbanicity, and road class are present.

---

### `segment_cohort_memberships`

Many-to-many canonical segments ↔ cohorts. **Not yet populated** — deferred until canonical mapper exists.

---

### `segment_observations`

Raw evidence landing zone for future Varia radar and rider-contributed data. **Stub table** — no ingestion pipeline yet.

---

## Phase 2 Tables (not yet migrated)

### `ride_instance_runs`
A route at a specific start time. Contextual overlay layer.

### `ride_instance_slice_conditions`
Time-aware environmental outputs per slice. Wind, temperature, precipitation, light_state, glare_flag, sun_azimuth, moon_phase live here — not in stable analysis tables.

### `route_slice_overrides`
Community/admin corrections to OSM-derived facts.

### `route_slice_effective_facts`
Materialized view combining OSM facts + support facts + approved overrides.

### `events` / `event_days` / `event_route_part_segments`
Full multi-day event model per ADR-031. `event_routes` is a partial implementation; this is the full expansion.

---

## Build Order (remaining)

1. Seed `route_analysis_runs` for a small set of canonicals
2. Populate `route_slice_analysis` and confirm indices are believable
3. Populate `route_analysis_summary` and verify rollups
4. Build `route_slice_effective_facts` view
5. Then: `ride_instance_runs` + `ride_instance_slice_conditions`
6. Then: `route_slice_overrides`
7. Then: canonical segment mapper (to populate `canonical_segments` and resolve `route_segment_instances`)
8. Then: `segment_behavior_inputs` + `segment_cohort_memberships` population
9. Then: full `events` / `event_days` / `event_route_part_segments` model per ADR-031

---

## Hard Rules

**Do not collapse:**
- Slice geometry and OSM facts (separate tables)
- OSM facts and support/proximity facts (separate tables)
- Stable analysis and ride-time conditions (separate tables)
- Canonical identity and source provenance (separate tables)
- Expedition state and live ride session state (different durability contracts)

**Do not write:**
- Wind, temperature, precipitation into `route_slice_analysis`
- Itinerary or event semantics into `canonical_routes`
- Analysis outputs into `route_slices`
- Raw OSM tags as the only record (always normalize into columns)
- Expedition-critical progress only into ephemeral browser memory
- Segment-level traffic facts against a free-text route-local segment_id


---

## Source File: docs/02-architecture/analysis/anal-001-indices_calculation.md

# Lanterne Indices Calculation

## Purpose

This document explains how individual **indices** are calculated.

Indices describe specific aspects of a route and are the building blocks behind route intelligence. Only some indices feed the Safety Score. Others are independent.

---

## Current Implementation vs Planned

The live scoring engine currently models safety primarily through Traffic Exposure and Bike Support. The broader index family (Remoteness, Surface Quality, Fatigue, Descent Risk, Conditions) is architecturally defined and partially implemented.

This document distinguishes:
- **Currently implemented** — running in the live scoring engine
- **Architecturally defined** — specified in ADRs and DS files, being introduced gradually

---

## 1. Safety Score (Currently Implemented)

Safety Score represents:

> The relative likelihood of a rider being struck by a motor vehicle and the expected severity of the outcome.

Normalized to a **0–100 scale**, mapped to letter grades (A through F).

**Current composition:**
- Traffic Exposure Index (dominant contributor — higher = more risk)
- Bike Support Index (mitigating factor — higher = less risk)

See `SCORE_CALCULATION.md` for the full pipeline.

---

## 2. Traffic Exposure Index (Currently Implemented)

**Rider question:** How dangerous is the motor vehicle environment on this road?

**Major inputs:**
- Road classification
- Speed environment (posted limit → inferred class → observed if available)
- Lane count
- Traffic volume (AADT if available via HPMS/DOT)
- Time-of-day multiplier
- Intersection density
- Rail crossings and hazard penalties

**Output:** Higher value → higher risk. Dominant contributor to Safety Score.

**Speed data hierarchy:**
1. Observed car speed (radar data with ≥3 samples)
2. Posted speed limit
3. Inferred speed environment class

---

## 3. Bike Support Index (Currently Implemented)

**Rider question:** How much does the road support cyclists?

**Major inputs:**
- Bike lanes
- Protected cycling facilities
- Paved shoulders (width and quality)
- Cycling infrastructure tags

**Output:** Higher value → lower risk. Reduces the Traffic Exposure effect.

---

## 4. Hazard Modifiers (Currently Implemented)

Micro-hazards add penalties to segment risk.

**Examples:**
- Bridges
- Cattle guards
- Railroad crossings
- Underpasses

Detected in `hazards.ts`. Each hazard adds a small penalty to the segment risk score.

---

## 5. Time-of-Day Traffic Model (Currently Implemented)

Traffic exposure adjusts based on estimated rider arrival time (`traffic-time.ts`).

**Inputs:** Estimated rider pace, start time, segment distance.

The system calculates expected arrival hour and scales traffic risk accordingly. This is especially important for randonneuring where overnight and dawn riding are common.

---

## 6. Remoteness Index (Architecturally Defined — ADR-019, DS-004)

**Rider question:** How far am I from help, services, food, water, and bailout options?

**Inputs:** Settlement proximity, service density, resupply access, bailout road access, route isolation, road network sparsity.

**Rollup strategy:** Longest unbroken remote stretch + peak isolation point. Not a simple average — a short access window should not flatten the remoteness score for the surrounding region.

**Remoteness priority (rider fear order):**
1. No escape roads nearby
2. No people / towns nearby
3. No services nearby

---

## 7. Surface Quality Index (Architecturally Defined — DS-004)

**Rider question:** How rideable is the surface on this part of the route?

**Inputs:** Paved vs unpaved, surface type, roughness/smoothness proxies, degraded pavement proxy, gravel/dirt/trail character.

Especially important for gravel and bikepacking contexts.

---

## 8. Fatigue Index (Architecturally Defined — ADR-025)

**Rider question:** How much cumulative rider burden does this part of the route contribute?

**Inputs:** Grade, cumulative climbing, accumulated distance, repeated rollers, stop/start burden, traffic stress contribution, surface drag. Future: weather burden, personal fatigue arc.

**Rollup strategy:** Cumulative burden across slices — not an average.

**Important:** Fatigue is not just climbing. A flat but exposed, windy, rough, stressful route can still be highly fatiguing. The model is designed to grow more sophisticated over time (ADR-025).

---

## 9. Descent Risk Index (Architecturally Defined — DS-004)

**Rider question:** How risky is this downhill section if something goes wrong?

**Inputs:** Negative grade, descent length, curvature, road width, shoulder availability, surface quality. Future: weather interaction, darkness interaction.

Not currently part of the narrow Safety Score unless future evidence justifies it.

---

## 10. Conditions Indices (Architecturally Defined — ADR-023)

Wind, Temperature, and Precipitation are ride-time contextual conditions — not stable route analysis outputs.

They depend on:
- Rider start time
- Estimated arrival time per slice (DS-003)
- Forecast data

They live in `ride_instance_slice_conditions` (Phase 2 schema), not in stable analysis tables.

**Wind:** Modeled relative to rider bearing, not absolute compass direction.

**Temperature:** Air temperature + apparent temperature + extreme threshold flags.

**Precipitation:** Probability, intensity, type, surface interaction.

---

## 11. Light Signals (Currently Implemented — ADR-008, ADR-009, ADR-010)

Light is modeled as a system of signals, not a major score.

**Light state:** Daylight / Twilight / Night — computed from solar altitude.

**Sun glare:** Flagged when sun elevation is 0–6° above horizon AND rider bearing is within ±30° of sun azimuth. Affects driver visibility, not just rider comfort.

**Moon phase:** Communicates whether night conditions feel moonlit or dark. High value for overnight randonneurs.

---

## 12. Slice-Level Analysis

All indices are calculated on **small internal slices** (ADR-020), not large visible segments. This preserves truth at transitions — remoteness dips near towns, traffic changes at intersections, surface transitions, light changes across the ride.

Display segments aggregate slices for readability. The analysis granularity and the display granularity are independent.

---

## 13. Storage Principles

- Core index values stored as structured columns
- Component breakdowns and confidence signals stored in JSON
- Stable analysis and ride-time conditions in separate tables
- Do not reduce core route intelligence to one generic key/value table

---

## 14. Design Principle

Each index answers a **clear rider question**.

If an index cannot be explained in one sentence to a rider, it should not exist.

| Index | One-sentence answer |
|-------|-------------------|
| Traffic Index | How stressful is the motor vehicle environment? |
| Bike Support Index | How well does the road infrastructure protect me? |
| Remoteness Index | How far am I from help if something goes wrong? |
| Surface Quality Index | How rideable is the pavement? |
| Fatigue Index | How physically demanding is this stretch? |
| Descent Risk Index | How risky is this downhill? |


---

## Source File: docs/02-architecture/analysis/anal-002-score_calculation.md

# Lanterne Score Calculation

## Purpose

This document explains how rider-facing **scores** are produced from underlying analysis data.

It reflects the **current scoring engine implementation** in the codebase (`safety-scoring.ts`, `traffic-time.ts`, `hazards.ts`).

The goal is that this document always matches the real system.

---

## 1. Score Philosophy

Lanterne does **not** collapse all route intelligence into one number.

Instead it uses:
- A primary **Safety Score**
- Several **indices describing route reality**
- Environmental **conditions**

The Safety Score is the only true "headline score" today. Everything else is supporting analysis.

---

## 2. Safety Score

**Definition:**

> The relative likelihood of a rider being struck by a motor vehicle and the expected severity of the outcome.

Normalized to a **0–100 scale** and mapped to letter grades.

---

## 3. Score Pipeline

```
Route geometry
    ↓
Segment risk modeling
    ↓
Risk-per-mile calculation
    ↓
Route risk aggregation
    ↓
Logistic normalization
    ↓
Safety Score (0–100)
    ↓
Letter grade
```

---

## 4. Segment Risk Modeling

Each route segment contributes to risk.

### Traffic exposure
Estimated using:
- Road classification
- Estimated speed environment
- Lane count
- AADT (if available via HPMS/DOT)
- Time-of-day traffic multiplier

### Infrastructure mitigation
Risk is reduced by:
- Bike lanes
- Protected cycling facilities
- Paved shoulders

### Road geometry penalties
Risk is increased by:
- Multi-lane roads
- Lack of shoulder
- Complex intersections
- Rail crossings

### Hazard penalties
Micro-hazards add additional risk penalties:
- Bridges
- Cattle guards
- Railroad crossings
- Underpasses

Detected in `hazards.ts`.

---

## 5. Time-of-Day Traffic Adjustment

The engine estimates **arrival hour per segment** and scales traffic exposure using a multiplier derived from `traffic-time.ts`.

This reflects rush hour, midday traffic, and overnight riding — particularly important for randonneuring scenarios.

---

## 6. Risk Per Mile

After segment penalties and mitigations are applied, the engine produces **Risk Per Mile** — the primary intermediate metric used for route comparison.

---

## 7. Route Aggregation

Segment risks are combined into a route-level risk value, weighted by segment length and accumulated penalties. This prevents a short segment from dominating the route unless the risk is extreme.

---

## 8. Logistic Normalization

Raw risk values vary widely. A logistic transform produces the final Safety Score in the range **0–100**, keeping the score readable while still differentiating routes.

---

## 9. Letter Grades

| Grade | Meaning |
|-------|---------|
| A | Very safe cycling conditions |
| B | Good conditions |
| C | Moderate caution required |
| D | Elevated risk |
| F | Dangerous conditions |

Exact thresholds may evolve as calibration improves.

---

## 10. Road Density Guardrail

The analysis engine includes a road density guardrail that fires when normalized road density exceeds **500 roads/mile** (total roads in memory ÷ route distance in miles).

**Behavior on violation:**
- Log a warning
- Attach `highRoadDensity: true` to `analysisWarnings`
- **Never block results from rendering**

The guardrail is informational only. It signals that the adaptive corridor should have narrowed the road set further, but it must not suppress output or prevent the results card from displaying.

**What the guardrail must not do:**
- Return early or throw
- Prevent the results card from showing
- Suppress downstream analysis

---

## 11. Adaptive Corridor

Before scoring, the road corridor width adapts based on road density (DS-008):

| Estimated density | Corridor width |
|------------------|----------------|
| < 100 roads/mile | 250m |
| 100–250 roads/mile | 200m |
| 250–400 roads/mile | 150m |
| 400–600 roads/mile | 100m |
| > 600 roads/mile | 60m |

Width only shrinks, never expands. Adaptation is logged: `[ADAPTIVE-CORRIDOR] density=340 roads/mi width=200m→150m`.

---

## 12. Diagnostics and Guardrails

The analysis engine includes diagnostics and guardrails implemented in `analysis-diagnostics.ts` and `analysis-guardrails.ts`:

- Missing data detection
- Incomplete analysis warnings
- Sanity checks on risk calculations
- Road density monitoring (warning-only, never blocks output)

The UI can surface these when necessary.

---

## 13. Windowed Scoring (Ultra-Distance Routes)

For routes using the expedition model (ADR-034), scoring operates on the **active analysis window** rather than the full route at once.

**Windowing is triggered when any of these are true:**
- Route distance > 400 miles
- GPX point count > 8,000
- Estimated road density > 500 roads/mile

Within a window, scoring operates identically to non-windowed routes. Window rollups are assembled into a full-route summary. The active window receives full heatmap and detail; the full route may be shown in simplified overview form.

---

## 14. Future Score Extensions

Possible future scores:
- Route Difficulty Score
- Exposure Score
- Service Availability Score
- Night Riding Risk Score

These will remain separate from Safety Score to keep its definition clean and trustworthy.

---

## 15. Design Rule

Safety Score must remain:
- Explainable
- Grounded in traffic safety research
- Resistant to feature creep

A dangerous road in light rain should not score better than the same road in sunshine. Weather is a different question with a different answer.


---

## Source File: docs/02-architecture/design/design-doc-index.md

# Lanterne Design Document Index

**Status:** Current as of 2026-03-24  
**Maintained by:** Derek

---

## Convention

**ADRs** record architectural decisions — what was decided, why, and what the tradeoffs are. They are immutable once Accepted; supersession creates a new ADR.

**DS files** are design specifications — how something is built, what tables it needs, what the interaction grammar is, what gets deferred. They are living documents that evolve as implementation proceeds.

**Cross-reference rules:**

- Every ADR lists its companion DS file(s) in its header
- Every DS file lists its ADR parent(s) in its header
- Self-contained ADRs are marked "No companion DS required"
- Proposed ADRs with existing DS files note the pending status in the DS header
- Reserved DS numbers are listed for future work

---

## ADR Inventory

| ADR | Title | Status | Companion DS |
|-----|-------|--------|--------------|
| ADR-001 | Route Acquisition Model | Accepted | Self-contained |
| ADR-002 | Vault Concept | Accepted | Self-contained |
| ADR-003 | Mode-Aware Vault Filtering | Accepted | Self-contained |
| ADR-004 | Rider Field Notes (Deferred) | Deferred — post-alpha | DS-016 (not yet written) |
| ADR-005 | Route Analysis Model | Accepted | See DS-001, DS-007 |
| ADR-006 | Safety Definition | Accepted | Self-contained |
| ADR-007 | Index Families | Accepted | Self-contained |
| ADR-008 | Environmental Light System | Accepted | Self-contained |
| ADR-009 | Sun and Moonlight Visualization | Accepted | Self-contained |
| ADR-010 | Sun Glare Detection | Accepted | Self-contained |
| ADR-011 | Route Slice Model | Accepted | DS-007, DS-005, DS-010 |
| ADR-012 | Predicted vs Experienced Conditions | Accepted | See ADR-023, DS-015 |
| ADR-013 | Personalized Emergency Alert Model | Accepted | Self-contained |
| ADR-014 | Ride Narrative Event Model | Accepted | See ADR-016 |
| ADR-015 | Route Vulnerability Feature Model | Accepted | Self-contained |
| ADR-016 | Ride Session Data Model | Accepted | Self-contained (DS may be written later) |
| ADR-017 | Local OSM-Derived Data Strategy | Accepted | DS-010 |
| ADR-018 | Server-Cached Slice Analysis Model | Accepted | DS-010, DS-005 |
| ADR-019 | Route Corridor & Proximity Rules | Accepted | DS-008 |
| ADR-020 | Atomic Route Analysis Unit & OSM Variable Architecture | Accepted | DS-001, DS-002, DS-007, DS-010 |
| ADR-021 | OSM Variable Registry | Accepted | DS-004 (living spec) |
| ADR-022 | Phase 1 Enum Registry | Accepted | Self-contained; enum values in DS-004 |
| ADR-023 | Predicted vs Observed Condition Layers | Accepted | DS-015 (not yet written) |
| ADR-024 | Ride Timeline Plans | Accepted | DS-003 |
| ADR-025 | Fatigue Index as Extensible Model Family | Accepted | Self-contained |
| ADR-026 | Canonical Route Identity | Accepted | DS-005, DS-006, DS-009 |
| ADR-027 | Lantern Screen Model | Accepted | DS-011, DS-012 |
| ADR-028 | Field Note Confirmation Model | Accepted | DS-016 (not yet written) |
| ADR-029 | Ride-Time Situational Awareness Mode | Proposed | DS-011, DS-012 |
| ADR-030 | Ride Mode Display, Power, and Sensor Architecture | Proposed | DS-012 (partial); power/sensor specs not yet written |
| ADR-031 | Multi-Day Events as Ordered References onto Canonical Geometry | Proposed | DS-014 (not yet written) |
| ADR-032 | Comparative Traffic Context and Segment Cohorts | Accepted | DS-013 |
| ADR-033 | Canonical Segment Identity and Route-to-Canonical Mapping | Accepted | DS-013 |
| ADR-034 | Master Route Expeditions and Windowed Long-Route Analysis | Draft | DS-014 |

---

## DS File Inventory

| File | Title | ADR Parent(s) | Status |
|------|-------|---------------|--------|
| DS-001 | Route Intelligence Pipeline | ADR-020, ADR-026 | Draft |
| DS-002 | Analysis Rollup | ADR-020 | Draft |
| DS-003 | Ride Timeline Model | ADR-024 | Draft |
| DS-004 | OSM Variable Registry *(living spec)* | ADR-021, ADR-022 | Draft — living document |
| DS-005 | Canonical Route Schema | ADR-026, ADR-031 | Draft |
| DS-006 | Route Canonicalization | ADR-026 | Draft |
| DS-007 | Route Slice Generation | ADR-020 | Draft |
| DS-008 | Route Corridor Model | ADR-019 | Draft |
| DS-009 | Route Corridor Fingerprint | ADR-026 | Draft |
| DS-010 | Slice Analysis Cache | ADR-020, ADR-017, ADR-018 | Draft |
| DS-011 | Ride-Time Situational Awareness Interface | ADR-029 | Draft (pending ADR-029 acceptance) |
| DS-012 | Ride Computer Tile System | ADR-027, ADR-029, ADR-030 | Draft (pending ADR-029/030 acceptance) |
| DS-013 | Comparative Traffic Context Schema | ADR-032, ADR-033 | Draft |
| DS-014 | Route Expedition State and Windowed Analysis | ADR-034 | Draft |
| DS-015 | Multi-Day Event Schema | ADR-031 | **Reserved — not yet written** |
| DS-016 | Predicted vs Observed Conditions Schema | ADR-023 | **Reserved — not yet written** |
| DS-017 | Field Note Schema and Confirmation Model | ADR-004, ADR-028 | **Reserved — not yet written** |

---

## Lineage Notes

### ADR-011 and ADR-020
ADR-011 is the original route slice model decision record. ADR-020 supersedes and expands it with the full atomic analysis unit architecture and OSM variable architecture. Both are retained for lineage. **ADR-020 is authoritative.**

### ADR-012 and ADR-023
ADR-012 is the original predicted vs experienced conditions decision, focused on ride-time conditions. ADR-023 supersedes and expands it to cover all predicted/observed condition layers. Both are retained for lineage. **ADR-023 is authoritative.**

### ADR-021 and DS-004
ADR-021 governs the existence and rules of the OSM variable registry. DS-004 is the living specification where the actual variable list evolves. **Add new variables and enum values to DS-004, not to ADR-021.**

### DS-008 number collision (resolved)
An earlier draft of the ride computer tile system was incorrectly numbered DS-008. DS-008 is the Route Corridor Model (ADR-019 companion). The ride computer tile system is DS-012. The earlier draft is superseded and should be deleted.

---

## Gaps to Fill

| Gap | Trigger |
|-----|---------|
| DS-014 — Multi-Day Event Schema | When ADR-031 is accepted |
| DS-015 — Predicted vs Observed Conditions Schema | When conditions persistence work begins |
| DS-016 — Field Note Schema | When field notes implementation begins (post-alpha) |
| ADR-029 → Accepted | When ride-time mode ships |
| ADR-030 → Accepted | When ride mode power/sensor ships |
| ADR-031 → Accepted | When multi-day event work begins |
| Power mode implementation spec | After ADR-030 acceptance |
| Sensor connection lifecycle spec | After ADR-030 acceptance |

---

## Design Principle

ADRs decide. DS files specify. Neither replaces the other.

When a decision changes → write a new ADR.  
When an implementation changes → update the DS file.  
When a new implementation area opens → check whether it needs an ADR first.


---

## Source File: docs/02-architecture/design/ds-001-route-intelligence-pipeline-spec.md

# DS-001 — Route Intelligence Pipeline Spec

**Status:** Draft  
**Date:** 2026

**ADR parents:** ADR-020 (Atomic Analysis Unit), ADR-026 (Canonical Route Identity)  
**Companion specs:** DS-002 (Analysis Rollup), DS-003 (Ride Timeline Model), DS-004 (OSM Variable Registry), DS-005 (Canonical Route Schema), DS-006 (Route Canonicalization), DS-007 (Route Slice Generation), DS-008 (Route Corridor Model), DS-009 (Route Corridor Fingerprint), DS-010 (Slice Analysis Cache)

---

## Purpose

Define the end-to-end pipeline that transforms an incoming route into **route intelligence**.

Lanterne is not a route file manager and not primarily a ride recorder. Its job is to transform route geometry into rider-facing understanding before the rider leaves home.

---

## Core Principle

```
Route source
    ↓
Normalization
    ↓
Canonical route identity
    ↓
Route corridor model
    ↓
Slice generation
    ↓
Stable analysis
    ↓
Ride timeline modeling
    ↓
Contextual conditions
    ↓
Rollups
    ↓
Presentation
    ↓
Rider decision
```

Route source, stored route identity, stable analysis, ride-time conditions, and rider-facing presentation are **separate concerns** throughout.

---

## Stage 1 — Route Acquisition

**Purpose:** Accept a route from one of Lanterne's supported entry paths.

**Supported inputs:**
- Manual route drawing
- RWGPS import
- GPX upload
- RUSA permanent import
- Ride history ingestion
- Future external sources

**Stored metadata:**

| Field | Description |
|-------|-------------|
| `source_type` | Origin of the route |
| `source_id` | External ID if applicable |
| `source_url` | Source URL if applicable |
| `imported_at` | Ingestion timestamp |
| `raw_name` | Original route name |
| `raw_geometry` | Unmodified source geometry |
| `provenance` | Source-specific metadata |

> **Guiding principle:** Acquisition captures what came in. It does not determine canonical identity.

---

## Stage 2 — Geometry Normalization

**Purpose:** Convert incoming route data into a consistent internal geometry format.

**Normalization tasks:**
- Decode source geometry
- Remove duplicate points
- Normalize coordinate precision
- Simplify GPS noise
- Standardize geometry format
- Compute distance and bounds

> **Guiding principle:** Normalization removes source-specific mess without changing the underlying route experience.

---

## Stage 3 — Canonical Route Resolution

**Purpose:** Determine whether the incoming route represents an existing canonical route or a new one.

**Canonical identity rule:** Identity is based on **experienced corridor geometry**, not on source platform, route title, start location, control placement, file format, or administrative event differences. See ADR-026.

```
normalized route
    ↓
candidate lookup
    ↓
corridor fingerprint filter (DS-009)
    ↓
geometry similarity comparison (DS-006)
    ↓
existing canonical route OR new canonical route
```

**Output:** `canonical_route_id`, `route_source` record, optional `route_variant` record

> **Guiding principle:** Lanterne models route experiences, not file artifacts.

---

## Stage 4 — Route Corridor Construction

**Purpose:** Represent the route as a corridor of real-world space rather than a thin polyline.

**The corridor supports:** hazard lookup, infrastructure detection, service proximity, canonical similarity comparison, and environmental exposure modeling.

**Output:** `corridor_geometry`, `corridor_width`, `spatial_envelope`

See DS-008 for corridor width rules including adaptive density-based sizing.

> **Guiding principle:** The rider experiences a corridor, not an infinitely thin line.

---

## Stage 5 — Slice Generation

**Purpose:** Divide the canonical route into small internal analysis slices — the atomic unit of route intelligence.

**Slice boundary triggers:** distance threshold, road classification change, intersection boundary, surface transition, bike infrastructure transition, bridge/tunnel transition, major grade transition.

See DS-007 for full boundary trigger rules and slice data structure.

> **Guiding principle:** Slices capture meaningful changes in route reality, not arbitrary chunks.

---

## Stage 6 — Stable Route Analysis

**Purpose:** Compute baseline route intelligence that does not depend on ride start time.

**Inputs:** Route geometry, OSM, elevation, geographic context

**Typical stable metrics:**

| Metric | Family |
|--------|--------|
| Traffic Index | Safety |
| Bike Support Index | Safety |
| Remoteness Index | Route Reality |
| Surface Quality Index | Route Reality |
| Fatigue baseline | Route Reality |
| Descent Risk baseline | Route Reality |

**Storage:** Results attach to slices, route rollups, and the canonical route.

**Cache strategy:** Stable slice analysis is cacheable and reusable across routes sharing the same underlying corridor. See DS-010.

> **Guiding principle:** Stable intelligence should be computed once and reused whenever possible.

---

## Stage 7 — Ride Timeline Modeling

**Purpose:** Estimate when the rider reaches each slice.

**Inputs:** Start time, expected rider pace, terrain burden, stop assumptions (including sleep and long rest per ADR-024)

**Output:** Arrival timestamp per slice

Once slice arrival time is known, the system can model ride-time conditions (light, weather, traffic multipliers) instead of static route averages. See DS-003.

> **Guiding principle:** Conditions must be tied to when the rider arrives, not just where the route goes.

---

## Stage 8 — Contextual Conditions Analysis

**Purpose:** Compute conditions that vary by time and depend on the ride plan.

**Typical outputs:**

| Signal | Notes |
|--------|-------|
| Wind | Speed and direction relative to rider bearing |
| Temperature | Thermal exposure per slice |
| Precipitation | Probability and intensity |
| Light state | Daylight / twilight / night |
| Sun glare risk | Bearing vs solar azimuth (ADR-010) |
| Moonlight context | Phase and cloud cover (ADR-009) |

**Separation rule:** Contextual conditions must remain distinct from the Safety Score per ADR-006 and ADR-007.

> **Guiding principle:** Show what the ride may feel like at that time, without contaminating the meaning of safety.

---

## Stage 9 — Rollups

**Purpose:** Aggregate slice-level intelligence into rider-facing summaries.

**Rollup strategies by metric:**

| Metric | Strategy |
|--------|---------|
| Traffic Index | Weighted mean + danger-sensitive percentile |
| Bike Support Index | Weighted mean |
| Remoteness Index | Longest remote stretch + peak isolation |
| Surface Quality | Weighted mean with worst-surface emphasis |
| Fatigue Index | Cumulative burden |
| Descent Risk | Concentrated-risk emphasis |

See DS-002 for full rollup strategy definitions.

> **Guiding principle:** Rollups must preserve route character rather than washing important sections away.

---

## Stage 10 — Route Intelligence Assembly

**Purpose:** Assemble the complete rider-facing intelligence object for a route.

```
route_intelligence
 ├ canonical_route
 ├ route_sources
 ├ stable_indices
 ├ contextual_conditions
 ├ hazards
 ├ services
 ├ route_rollups
 ├ timeline_plan
 └ presentation_metadata
```

> **Guiding principle:** Produce a coherent object of understanding, not scattered metrics.

---

## Stage 11 — Presentation Preparation

**Purpose:** Transform route intelligence into UI-ready layers.

**Presentation outputs:** map overlays, heatmap segments, analysis drawer cards, condition icons, tooltips, cue-sheet views, reveal/tutorial sequencing.

**Display rule:** The system may aggregate or simplify for readability, but presentation must remain grounded in slice-level truth.

> **Guiding principle:** The UI should feel like an intelligent reveal of the ride, not a dump of raw data.

---

## Stage 12 — Rider Decision Support

**Purpose:** Support a real rider decision.

**Typical decisions enabled:** route comparison, identifying dangerous sections, adjusting start time, judging remoteness, comparing variants, evaluating detours.

**Final product rule:** Lanterne succeeds when the rider understands *what the route will be like* — not just where it goes.

---

## Stable vs Contextual Boundary

| Layer | Derived from | Cached? |
|-------|-------------|---------|
| **Stable route intelligence** | Geometry, OSM, elevation, geography | Yes — long-lived |
| **Contextual ride-time intelligence** | Start time, pace, forecasts, solar/lunar timing | No — recomputed per ride plan |

See ADR-023 for the full separation decision.

---

## Storage Boundaries

| Entity | Contains |
|--------|---------|
| Canonical route | Experienced corridor identity |
| Route source | Where the route came from |
| Route variant | Meaningful geometry differences |
| Slice analysis | Atomic stable analysis units |
| Ride plan / timeline | Start-time-dependent modeling |
| Rider save relationship | User-specific storage and history |

Do not collapse identity, analysis, source, and ownership into one blob.

---

## Cache Boundaries

**Long-lived cacheable:** canonical route identity, normalized geometry, corridor geometry, slice generation, stable slice analysis, route rollups from stable metrics.

**Recomputed per ride plan:** ride timeline, weather overlays, light conditions, time-of-day traffic effects, user presentation preferences.

---

## Failure / Degradation Rule

If contextual layers fail, the system must still return stable route intelligence. Forecast unavailable, moonlight unavailable, incomplete observed conditions — all degrade gracefully to stable analysis.

---

## Non-Goals

This specification does not define exact canonicalization thresholds (→ DS-006), exact slice generation math (→ DS-007), exact cache table schemas (→ DS-010), exact UI sequencing, or exact fingerprint algorithm internals (→ DS-009).

---

## Guiding Principle

Lanterne transforms route geometry into a layered understanding of what the road is, what the route feels like, what may happen at the chosen time, and what decision the rider should make.


---

## Source File: docs/02-architecture/design/ds-002-analysis-rollup-spec.md

# DS-002 — Analysis Rollup Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-020 (Atomic Analysis Unit)  
**Related specs:** DS-001 (Route Intelligence Pipeline), DS-010 (Slice Analysis Cache)

---

## Purpose

Define how slice-level analysis is aggregated into route-level metrics.

Different metrics require different rollup strategies. Using a single strategy (e.g. weighted mean for everything) would hide short dangerous sections and distort route character.

---

## Rollup Strategies

### Weighted Mean

Used for metrics where overall exposure matters along the full route.

```
weight = slice distance / total route distance
result = Σ(slice_value × weight)
```

**Used for:** `traffic_index`, `bike_support_index`

---

### Worst Reasonable Percentile

Used for danger-sensitive metrics where a short terrible section must not be hidden by a long average.

**Used for:** Traffic exposure — 95th percentile ensures that a dangerous mile on a 100-mile route is not averaged away.

---

### Maximum Stretch Detection

Used for metrics representing continuous conditions where the longest unbroken stretch matters more than the average.

**Output values:**
- Longest unbroken stretch (distance)
- Peak isolation point (worst single location)

**Used for:** `remoteness_index`

---

### Cumulative Burden

Used for fatigue modeling where the effect accumulates across the entire route rather than averaging.

```
result = Σ(slice_fatigue_contribution)
```

**Used for:** `fatigue_index`

---

## Rollup Summary Table

| Metric | Strategy | Why |
|--------|---------|-----|
| `traffic_index` | Weighted mean + 95th percentile | Overall exposure + worst section |
| `bike_support_index` | Weighted mean | Overall infrastructure quality |
| `remoteness_index` | Longest stretch + peak isolation | Continuous exposure matters most |
| `surface_quality_index` | Weighted mean + worst-surface emphasis | Overall quality + worst patch |
| `fatigue_index` | Cumulative burden | Accumulates over the whole route |
| `descent_risk_index` | Concentrated-risk emphasis | Short technical descents dominate |

---

## Guiding Principle

Rollups must preserve important route character rather than smoothing everything into averages. A route with one genuinely dangerous mile should not score the same as a uniformly moderate route of the same length.


---

## Source File: docs/02-architecture/design/ds-003-ride-timeline-model-spec.md

# DS-003 — Ride Timeline Model Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-024 (Ride Timeline Plans)  
**Related specs:** DS-001 (Route Intelligence Pipeline), DS-007 (Route Slice Generation)  
**Related ADRs:** ADR-023 (Predicted vs Observed Condition Layers), ADR-025 (Fatigue Index as Extensible Model Family)

---

## Purpose

Map route position to time during a ride.

This enables modeling of conditions that vary with time of day — weather, light state, sun glare risk, traffic patterns, and temperature — all of which depend on knowing *when* the rider arrives at each slice.

---

## Inputs

| Input | Description |
|-------|-------------|
| `start_time` | Planned ride start timestamp |
| `estimated_rider_pace` | Base speed assumption |
| `terrain_effects` | Grade and surface adjustments to pace |
| `expected_stop_behavior` | Short stops, control stops |
| `interruption_events` | Sleep breaks, long rest stops (required per ADR-024) |

The model must support interruption events. Predicted arrival times must **not** assume a single uninterrupted pacing function — this is especially critical for overnight brevets and multi-day events.

---

## Slice Arrival Time Calculation

```
slice distance
→ travel time estimate (adjusted for grade and surface)
→ cumulative ride time
→ stop/sleep event insertion at appropriate positions
→ arrival timestamp per slice
```

---

## Stop and Sleep Event Model

The timeline plan may contain an ordered list of interruption events:

**Stop event:**
- Position along route (distance from start)
- Estimated duration

**Sleep event:**
- Position along route
- Sleep start time
- Sleep end time (or duration)

Arrival times for all slices **after** a stop or sleep event are computed by adding the event duration to the cumulative ride time at that position.

---

## Condition Modeling

Once slice arrival time is known, the system computes per-slice contextual conditions:

| Condition | Depends on arrival time |
|-----------|------------------------|
| Traffic exposure multiplier | Time-of-day traffic patterns |
| Wind conditions | Forecast at arrival time |
| Temperature | Forecast at arrival time |
| Precipitation | Forecast at arrival time |
| Light state | Solar position at arrival time and location |
| Sun glare risk | Solar azimuth vs bearing at arrival time (ADR-010) |
| Moonlight | Moon phase + cloud cover at arrival time (ADR-009) |

These are contextual predictions per ADR-023, kept distinct from stable route analysis outputs.

---

## Timeline Adjustment

Riders may adjust timeline parameters:
- Pace
- Start time
- Stop duration
- Sleep location and duration

Any adjustment must **recalculate all slice arrival times** downstream of the changed parameter. Partial recalculation is not acceptable — a changed start time affects every downstream condition.

---

## Guiding Principle

Conditions must reflect when the rider actually arrives at each part of the route, accounting for planned stops and sleep. A pacing model that cannot be interrupted is not a model for long-distance riding.


---

## Source File: docs/02-architecture/design/ds-004-osm-variable-registry-spec.md

# DS-004 — OSM Variable Registry Spec

**Status:** Draft (living document — variable list evolves here)  
**Date:** 2026

**ADR parents:** ADR-021 (OSM Variable Registry), ADR-022 (Phase 1 Enum Registry)  
**Related ADRs:** ADR-020 (Atomic Analysis Unit)  
**Related specs:** DS-001 (Route Intelligence Pipeline), DS-005 (Canonical Route Schema), DS-007 (Route Slice Generation)

> **Note:** This is the **living specification** for the variable registry. ADR-021 governs the existence and rules of the registry. ADR-022 governs the string-enum approach and Phase 1 allowed values. New variables and enum value additions are made **here**, not in the ADRs.

---

## Purpose

Define the OpenStreetMap variables used by the analysis engine and ensure consistent interpretation of OSM tags across the entire system.

---

## Registry Structure

Each variable entry defines:

| Field | Description |
|-------|-------------|
| `osm_tag` | Source OSM tag(s) |
| `normalized_variable` | Internal Lanterne field name |
| `description` | What it represents |
| `analysis_usage` | Which index or system uses it |
| `confidence_class` | observed / explicit / inferred / weak |

**Registry maintenance rule:** New variables may only be added when they answer a clear rider question. All variables must trace to an OSM source tag and carry a confidence class.

---

## A. Road Environment Variables

Support Traffic and Safety analysis.

| Variable | Description |
|----------|-------------|
| `road_class` | Normalized road classification bucket |
| `speed_limit_value` | Posted speed limit (exact value) |
| `speed_limit_confidence` | Confidence in speed limit source |
| `car_speed_value` | Observed or inferred car speed |
| `car_speed_sample_n` | Number of radar samples (if observed) |
| `car_speed_confidence` | Confidence in car speed value |
| `car_speed_source` | osm / radar / inferred |
| `speed_environment_class` | Enum: very_low / low / moderate / high / very_high |
| `lane_count_value` | Exact lane count |
| `lane_count_confidence` | Confidence in lane count |
| `traffic_proximity_class` | Class of nearby traffic environment |
| `intersection_density_value` | Intersections per mile |
| `major_intersection_flag` | Boolean |
| `crossing_complexity_class` | none / minor / caution |
| `ramp_or_interchange_flag` | Boolean |
| `pinchpoint_flag` | Boolean |

**Observed speed hierarchy** — traffic speed variables follow this priority:
1. Observed car speed (radar data with ≥3 samples)
2. Posted speed limit
3. Inferred speed environment

Observed speed overrides other speed values when confidence is sufficient.

---

## B. Bike Support Variables

Describe cycling infrastructure.

| Variable | Description |
|----------|-------------|
| `bike_facility_type` | none / marked_bike_route / painted_bike_lane / protected_bike_lane / bike_path / multiuse_path |
| `bike_facility_confidence` | Confidence in facility classification |
| `shoulder_presence` | Boolean |
| `shoulder_class` | none / narrow / usable / wide |
| `shoulder_width_value` | Exact width when known |
| `shoulder_confidence` | Confidence in shoulder data |

Shoulder width may be stored when known but falls back to `shoulder_class` when uncertain.

---

## C. Surface Variables

Surface type and rideability are stored separately.

| Variable | Description |
|----------|-------------|
| `surface_type` | paved / gravel / dirt / trail / unknown |
| `surface_subtype` | More specific surface descriptor |
| `surface_quality_class` | Rideability classification |
| `surface_confidence` | Confidence in surface data |
| `offroad_context_class` | none / paved_path / gravel_road / forest_road / trail / singletrack |

This allows meaningful distinctions for bikepacking and gravel riding.

---

## D. Remoteness and Support Variables

Remoteness is determined by proximity to multiple forms of support. See ADR-019 for the full proximity model.

| Variable | Description |
|----------|-------------|
| `settlement_proximity_m` | Distance to nearest settlement |
| `service_proximity_m` | Distance to nearest service cluster |
| `water_proximity_m` | Distance to nearest water source |
| `food_proximity_m` | Distance to nearest food source |
| `lodging_proximity_m` | Distance to nearest lodging |
| `medical_proximity_m` | Distance to nearest medical facility |
| `bailout_access_proximity_m` | Distance to nearest bailout road |
| `support_context_class` | Composite support availability classification |

Remoteness scoring considers these variables together rather than collapsing them into a single distance measure.

---

## E. Geometry and Terrain Variables

Support fatigue and descent analysis.

| Variable | Description |
|----------|-------------|
| `elevation_m` | Elevation at slice midpoint |
| `grade_percent` | Average grade across slice |
| `descent_flag` | Boolean — slice is a descent |
| `curvature_class` | straight / gentle / curvy / technical |
| `technical_descent_flag` | Boolean — descent requires active control |

Elevation is a required input for route analysis.

---

## F. Environmental Timing Variables

Capture riding conditions dependent on time of day and forecast. These variables depend on ride start time and are **contextual variables per ADR-023** — not inputs to the core safety model.

| Variable | Description |
|----------|-------------|
| `light_state` | daylight / civil_twilight / night |
| `glare_flag` | Boolean — sun glare risk active |
| `cloud_cover_class` | clear / partly_cloudy / mostly_cloudy / overcast |
| `uv_class` | low / moderate / high / very_high |
| `wind_speed_value` | Wind speed (mph or kph) |
| `wind_gust_value` | Gust speed |
| `wind_bearing_value` | Wind direction (degrees) |
| `relative_headwind_class` | Headwind class relative to rider bearing |
| `temperature_value` | Temperature |
| `apparent_temperature_value` | Feels-like temperature |
| `precip_probability` | Precipitation probability 0..1 |
| `precip_intensity_class` | none / light / moderate / heavy |

---

## G. Traceability Fields

Each slice stores traceability metadata enabling debugging and transparent model behavior.

| Field | Description |
|-------|-------------|
| `raw_osm_tags_json` | Raw OSM tags as received |
| `normalized_variable_evidence_json` | How each variable was derived |
| `slice_variable_confidence_json` | Per-variable confidence signals |
| `community_override_json` | Community corrections if applicable |

---

## Future Variables (Reserved)

Not required for v1 but reserved for future addition:
- Road lighting presence
- Tunnel and bridge flags
- Guardrail presence
- Tree cover proxy
- Commercial artery crossing risk

---

## Guiding Principle

OSM data must be interpreted consistently across the entire system. Variables are the contract between raw map data and route intelligence. Add new variables only when they answer a clear rider question.


---

## Source File: docs/02-architecture/design/ds-005-canonical-route-schema-spec.md

# DS-005 — Canonical Route Schema Spec

**Status:** Draft  
**Date:** 2026

**ADR parents:** ADR-026 (Canonical Route Identity), ADR-031 (Multi-Day Events as Ordered References)  
**Related ADRs:** ADR-020 (Atomic Analysis Unit), ADR-023 (Predicted vs Observed Condition Layers)  
**Related specs:** DS-001 (Route Intelligence Pipeline), DS-006 (Route Canonicalization), DS-007 (Route Slice Generation), DS-009 (Route Corridor Fingerprint), DS-010 (Slice Analysis Cache)

> **Scope note:** This spec covers the canonical route intelligence schema only. Event and multi-day ride tables (`events`, `event_days`, `event_route_parts`) are explicitly out of scope and belong in DS-014.

---

## Purpose

Define the core storage model for canonical routes in Lanterne.

This schema supports:
- Geometry-first route identity
- Provenance tracking
- Route variants
- Slice-based analysis
- Stable reuse of route intelligence

---

## Core Principle

Lanterne stores routes as **experienced route identities**, not as raw file artifacts.

The schema separates:
- The canonical route
- Where the route came from
- Meaningful route variants
- Internal slice structure
- Extracted variables
- Computed analysis outputs
- Rider-specific save relationships

These concerns must not be collapsed into a single table or JSON blob.

---

## Entity Overview

```
canonical_routes
route_sources
route_variants
route_slices
slice_variables
slice_analysis
route_rollups
user_routes
```

---

## 1. `canonical_routes`

Represents the stable geometry-first identity of a route.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `canonical_name` | Human-readable name |
| `canonical_geometry` | Normalized route line |
| `corridor_geometry` | Buffered route corridor |
| `distance_m` | Route distance in meters |
| `elevation_gain_m` | Total elevation gain |
| `bounding_box` | Geographic bounds |
| `centroid` | Route centroid |
| `fingerprint_hash` | Fast candidate lookup hash (DS-009) |
| `canonicalization_version` | Algorithm version used |
| `created_at` | Creation timestamp |
| `updated_at` | Last update timestamp |
| `status` | active / superseded / review_flagged |

**Rules:**
- Canonical routes are immutable in meaning
- If the route materially changes, create a new canonical route or variant
- Small metadata changes do not alter canonical identity

---

## 2. `route_sources`

Tracks where a route came from. Multiple sources may point to one canonical route.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `canonical_route_id` | FK to canonical_routes |
| `source_type` | rwgps / gpx / rusa / manual / history_import / future_external |
| `source_id` | External source ID |
| `source_url` | Source URL if applicable |
| `source_name` | Original name at source |
| `raw_geometry` | Incoming artifact geometry — preserved verbatim |
| `normalized_geometry` | Post-normalization geometry before canonical assignment |
| `imported_at` | Import timestamp |
| `provenance_json` | Source-specific metadata |
| `is_primary_source` | Boolean — primary display source |

**Rules:** Source metadata never determines canonical identity by itself. Immutable after import except for metadata backfill.

---

## 3. `route_variants`

Stores meaningful variations of a canonical route — branches that are close enough to belong to the same broader identity but different enough to warrant separate representation.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `canonical_route_id` | FK to canonical_routes |
| `variant_type` | event_variant / user_detour / construction_detour / alternate_start / extended_version / shortened_version / surface_variant |
| `variant_name` | Human-readable name |
| `variant_geometry` | Variant route geometry |
| `distance_m` | Variant distance |
| `bounding_box` | Geographic bounds |
| `parent_variant_id` | FK to parent variant (nullable) |
| `created_from_source_id` | FK to source that created this variant |
| `created_at` | Creation timestamp |
| `variant_reason` | Why this variant exists |

**A route variant should be created when:** geometry differs meaningfully from the canonical route AND the rider experience changes in a way that matters AND the route should remain grouped under the same broader route identity.

---

## 4. `route_slices`

Stores the ordered internal slices used for route analysis. Slices are the atomic analysis unit per ADR-020.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `canonical_route_id` | FK to canonical_routes |
| `route_variant_id` | FK to route_variants (nullable) |
| `slice_index` | Ordinal position — must remain ordered |
| `slice_geometry` | Slice geometry |
| `start_distance_m` | Distance from route start |
| `end_distance_m` | Distance from route start |
| `length_m` | Slice length |
| `bearing` | Direction of travel |
| `average_grade` | Average grade across slice |
| `slice_builder_version` | Rules version used to generate boundaries |
| `created_at` | Creation timestamp |

**Rules:** Slices must remain ordered. Slices should be stable across re-analysis unless geometry or slice-building rules change.

---

## 5. `slice_variables`

Stores extracted normalized variables for each slice — the structured result of OSM and related variable extraction. Variable definitions are maintained in DS-004.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `route_slice_id` | FK to route_slices |
| `osm_registry_version` | Registry version used |
| `road_class` | Normalized road class |
| `surface_type` | Surface classification |
| `bike_infra_type` | Bike facility type |
| `shoulder_type` | Shoulder classification |
| `bridge_flag` | Boolean |
| `tunnel_flag` | Boolean |
| `rail_crossing_flag` | Boolean |
| `intersection_density_class` | Intersection density |
| `lane_count` | Lane count |
| `speed_class` | Speed environment class |
| `traffic_signal_flag` | Boolean |
| `variable_confidence_json` | Per-variable confidence signals |
| `source_tags_json` | Bounded raw tag evidence |
| `created_at` | Creation timestamp |

**Rules:** Every variable must come from DS-004. No ad hoc variables written directly into storage.

---

## 6. `slice_analysis`

Stores computed analysis results for each slice — where extracted variables become rider-relevant intelligence.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `route_slice_id` | FK to route_slices |
| `analysis_version` | Scoring model version |
| `traffic_index` | 0–100 |
| `bike_support_index` | 0–100 |
| `remoteness_index` | 0–100 |
| `surface_quality_index` | 0–100 |
| `fatigue_baseline` | Cumulative burden contribution |
| `descent_risk_index` | 0–100 |
| `hazard_score` | Composite hazard score |
| `analysis_breakdown_json` | Subcomponents and explanations |
| `computed_at` | Computation timestamp |

**Rules:** Stable slice analysis must remain separate from ride-time conditions (ADR-023). Re-analysis creates new versioned rows rather than mutating history.

---

## 7. `route_rollups`

Stores route-level summaries derived from slice analysis.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `canonical_route_id` | FK to canonical_routes |
| `route_variant_id` | FK to route_variants (nullable) |
| `analysis_version` | Scoring model version |
| `traffic_index` | Route-level rollup |
| `bike_support_index` | Route-level rollup |
| `remoteness_index` | Route-level rollup |
| `surface_quality_index` | Route-level rollup |
| `fatigue_index` | Cumulative burden |
| `descent_risk_index` | Route-level rollup |
| `rollup_method_version` | Rollup strategy version |
| `rollup_breakdown_json` | Per-metric rollup details |
| `computed_at` | Computation timestamp |

Rollup strategies are defined in DS-002. Rollups summarize routes for rider-facing presentation — they do not replace slice-level truth.

---

## 8. `user_routes`

Stores the rider-specific relationship to a canonical route or route variant. Separates personal ownership from route identity.

| Field | Description |
|-------|-------------|
| `id` | UUID primary key |
| `user_id` | FK to auth.users |
| `canonical_route_id` | FK to canonical_routes |
| `route_variant_id` | FK to route_variants (nullable) |
| `saved_name` | Rider's name for this route |
| `save_source_id` | FK to the source that was saved |
| `saved_at` | Save timestamp |
| `notes` | Rider notes |
| `visibility` | private / shared / public |

This allows multiple users to reference the same canonical route intelligence object without duplicating analysis.

---

## Relationship Model

```
canonical_route
  ├ route_sources         (provenance)
  ├ route_variants        (meaningful branches)
  ├ route_slices
  │    ├ slice_variables   (extracted evidence)
  │    └ slice_analysis    (computed intelligence)
  ├ route_rollups          (rider-facing summaries)
  └ user_routes            (personal relationships)
```

---

## Versioning Strategy

The schema supports independent versioning of:

| Version field | Controls |
|--------------|---------|
| `canonicalization_version` | How routes are deduplicated |
| `slice_builder_version` | How slice boundaries are generated |
| `osm_registry_version` | How OSM tags are normalized |
| `analysis_version` | How indices are scored |
| `rollup_method_version` | How slice results are aggregated |

Using a single blunt version number creates unnecessary recomputation and migration pain when only one component changes.

---

## Mutation Rules

| Entity | Mutation rule |
|--------|--------------|
| `canonical_routes` | Immutable in meaning; metadata updates only |
| `route_sources` | Immutable after import except metadata backfill |
| `route_variants` | Immutable after creation except metadata |
| `route_slices / variables / analysis` | Versioned — create new rows when logic changes |

---

## Non-Goals

This schema spec does not define:
- Exact SQL types or PostGIS implementation details
- Exact fingerprint algorithm → DS-009
- Exact variable registry contents → DS-004
- Exact rollup formulas → DS-002
- Event or multi-day ride tables → DS-014

---

## Guiding Principle

```
canonical route = identity
source          = provenance
variant         = meaningful branch
slice           = atomic truth
variables       = extracted evidence
analysis        = computed intelligence
rollup          = rider-facing summary
user_route      = personal relationship
```

The database should reflect the real structure of the product — not collapse it into a blob.


---

## Source File: docs/02-architecture/design/ds-006-route-canonicalization-spec.md

# DS-006 — Route Canonicalization Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-026 (Canonical Route Identity)  
**Related specs:** DS-005 (Canonical Route Schema), DS-009 (Route Corridor Fingerprint), DS-001 (Route Intelligence Pipeline)

---

## Purpose

Define how Lanterne determines whether an imported route corresponds to an existing canonical route.

The goal is to identify routes representing the same real-world road experience while allowing meaningful route variants to exist.

---

## Core Concept

A canonical route represents a unique corridor of roads. Different route files representing the same corridor should map to that canonical route. Variants attach to the canonical route rather than creating duplicates.

---

## Canonicalization Pipeline

```
route import
    ↓
geometry normalization
    ↓
candidate search (using fingerprint from DS-009)
    ↓
similarity measurement
    ↓
canonical match  OR  new canonical route
```

---

## Step 1 — Geometry Normalization

Before comparison, routes are normalized:
- Remove duplicate points
- Resample geometry to uniform spacing
- Simplify extreme GPS noise
- Normalize coordinate precision

This ensures similarity comparisons are stable regardless of source GPS quality.

---

## Step 2 — Candidate Search

Search for existing canonical routes within a geographic bounding box.

**Candidate selection may use:**
- Bounding box intersection
- Route centroid proximity
- Spatial index lookup
- Fingerprint hash pre-filter (DS-009)

This step reduces the comparison cost — only plausible nearby routes are fully compared.

---

## Step 3 — Similarity Measurement

Routes are compared using corridor similarity metrics.

**Possible methods:**
- Fréchet distance
- Hausdorff distance
- Corridor overlap percentage

Practical implementation may use a hybrid approach. Fingerprints (DS-009) are a fast pre-filter; full geometry comparison is the final arbiter.

---

## Corridor Overlap Principle

Two routes are considered equivalent if they share most of the same road corridor.

**Typical threshold:** overlap ≥ 85–90%

Minor deviations are tolerated. This accounts for GPS sampling differences, minor control relocations, and small reroutes around temporary closures.

---

## Start / End Tolerance

Start and end points may shift within a small radius without affecting identity.

**Typical tolerance:** 200–500 meters

This accounts for: different parking lots, control placement changes, start location adjustments.

---

## Direction Handling

Routes that follow the same corridor in opposite directions should generally resolve to the **same canonical route**.

Direction is treated as a variant attribute, not a canonical identity attribute.

---

## Loop Routes

Loop routes may start at any point along the loop.

**Normalization should:**
- Detect loop closure
- Rotate geometry for best alignment before comparison

This prevents multiple canonical entries for the same loop based on different starting points.

---

## Variant Storage

Variants attach to canonical routes:

```
canonical_route
   ├ RWGPS variant
   ├ RUSA variant
   ├ GPX variant
   └ user-edited variant
```

Variants may store: original geometry, source metadata, control locations, user annotations.

---

## New Canonical Route Criteria

A new canonical route is created when:
- Corridor overlap falls below threshold
- Route diverges significantly from all candidates
- Route length differs substantially
- Geometry represents a materially new alignment

**Examples requiring a new canonical:** major detours, alternate highways, gravel vs pavement branch, extended route variants.

---

## Manual Review Cases

Some cases should be flagged for manual review:
- Overlap near threshold (80–90%)
- Highly braided routes
- Dense urban grid routes
- Multi-loop routes

These are stored with `status = 'review_flagged'` on `canonical_routes` and resolved through admin tooling.

---

## Performance Considerations

Canonicalization must remain efficient at scale:
- Spatial indexing for candidate lookup
- Corridor hashing (DS-009) to filter non-candidates early
- Cached canonical geometry envelopes

Comparison should only occur against plausible nearby candidates — never against the full canonical table.

---

## Non-Goals

This spec does not define:
- Exact Fréchet/Hausdorff implementation details
- Exact fingerprint algorithm → DS-009
- The full canonical route schema → DS-005

---

## Guiding Principle

The canonical route represents the experienced road corridor, not the file artifact. Two riders following the same road experience should land on the same canonical route even if their source files differ.


---

## Source File: docs/02-architecture/design/ds-007-route-slice-generation-spec.md

# DS-007 — Route Slice Generation Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-020 (Atomic Analysis Unit)  
**Related specs:** DS-001 (Route Intelligence Pipeline), DS-005 (Canonical Route Schema), DS-010 (Slice Analysis Cache), DS-004 (OSM Variable Registry)

---

## Purpose

Define how Lanterne divides routes into **internal analysis slices**.

Slices are the atomic unit of route intelligence and the foundation of the analysis engine. Routes are never analyzed as a single averaged entity — the system evaluates many small sections and aggregates results upward.

---

## Target Slice Length

| Condition | Target length |
|-----------|--------------|
| Default | 200–500 meters |
| Frequent characteristic changes | Shorter — down to ~100m |
| Stable long stretches | Longer — up to maximum |
| **Maximum (hard cap)** | **750–1000 meters** |

---

## Slice Boundary Triggers

A new slice must be created when any of the following occur:

| Trigger | Examples |
|---------|---------|
| **Distance threshold** | Slice length exceeds target range |
| **Road classification change** | tertiary → secondary, residential → trunk |
| **Intersection boundary** | Route transitions across a major intersection node |
| **Surface change** | asphalt → gravel, paved → dirt |
| **Bike infrastructure change** | Bike lane begins, shoulder disappears, protected path starts |
| **Bridge or tunnel transition** | Entering or exiting a bridge or tunnel |
| **Major grade transition** | Grade change ≥ 3–4% |
| **Environmental event boundary** | daylight → twilight, sunrise/sunset glare onset |

---

## Slice Data Structure

Each slice stores:

| Field | Description |
|-------|-------------|
| `slice_id` | UUID |
| `route_id` | FK to canonical_routes |
| `start_distance_m` | Distance from route start |
| `end_distance_m` | Distance from route start |
| `length_m` | Slice length |
| `geometry` | Slice geometry (LineString) |
| `bearing` | Direction of travel |
| `average_grade` | Average grade across slice |
| `osm_way_ids` | Source OSM ways used |
| `road_class` | Normalized road class |
| `surface_type` | Surface classification |
| `infrastructure_tags` | Raw infrastructure tag summary |
| `slice_builder_version` | Rules version used to generate boundaries |

Full schema is in DS-005.

---

## Slice Stability Principle

Slice boundaries should remain **stable** whenever possible.

Slices should only change if:
- Route geometry changes
- Underlying OSM data changes
- Slice generation algorithm version changes

Stable slices allow reuse of cached analysis results (DS-010). Unnecessary regeneration breaks the cache and forces reanalysis.

---

## Relationship to Display Segments

Slices are **not** directly visible to users. The UI aggregates slices into display segments for readability.

```
display_segment
   └ multiple analysis slices
```

This separation means the display granularity can be changed without affecting analysis accuracy.

---

## Expected Slice Density

Typical routes should produce approximately **200–400 slices per 200 km** route.

Denser slicing occurs in: urban areas with frequent intersections, routes with many infrastructure transitions, routes with significant terrain variation.

---

## Guiding Principle

Slices represent meaningful changes in the riding experience rather than arbitrary equal-length segmentation. The slice boundary is where something real changes — not where the distance counter rolls over.


---

## Source File: docs/02-architecture/design/ds-008-route-corridor-model-spec.md

# DS-008 — Route Corridor Model Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-019 (Route Corridor & Proximity Rules)  
**Related specs:** DS-001 (Route Intelligence Pipeline), DS-005 (Canonical Route Schema), DS-006 (Route Canonicalization), DS-009 (Route Corridor Fingerprint)

> **Note:** DS-008 is the Route Corridor Model. The Ride Computer Tile System is DS-012. An earlier draft incorrectly assigned the tile system to DS-008; that draft is superseded by DS-012.

---

## Purpose

Define the route corridor concept used throughout Lanterne.

Routes are not treated as infinitely thin lines. Instead they occupy a **corridor of real-world space**, allowing better modeling of hazards, infrastructure, services, and nearby features.

---

## Corridor Geometry

A route corridor is generated by buffering the route polyline.

**Default width:** 50–150 meters (before adaptive adjustment)

Each route stores:
- `corridor_geometry` — the buffered corridor polygon
- `corridor_width` — the active width used

The corridor may be simplified for faster spatial queries.

---

## Adaptive Corridor Width

Corridor width adapts based on **road density** after the initial tile fetch. This prevents urban routes from pulling in excessive OSM ways while ensuring rural routes capture parallel alternatives.

**Density thresholds:**

| Roads per mile | Corridor width |
|---------------|----------------|
| < 100 | 250m |
| 100–250 | 200m |
| 250–400 | 150m |
| 400–600 | 100m |
| > 600 | 60m |

**Rules:**
- Corridor width **only shrinks** during adaptation — never expands
- Density is computed as total roads / route distance in miles after first fetch pass
- Adaptation is logged: `[ADAPTIVE-CORRIDOR] density=340 roads/mi width=200m→150m`

**Rationale:** Urban routes (e.g. through Honolulu) can return 44,000+ nearby roads at a fixed 111m corridor, causing multi-minute processing times and memory exhaustion. Adaptive narrowing reduces the scoring road set to a manageable size without sacrificing accuracy on the roads that actually matter.

---

## Corridor Uses

The corridor model supports:

| Use | Description |
|-----|-------------|
| **Infrastructure detection** | Bike lanes, shoulders, paths near the route |
| **Service detection** | Food, water, bike shops within reasonable reach |
| **Hazard proximity** | Bridges, rail crossings, tunnels adjacent to the route |
| **Environmental exposure** | Wind exposure, terrain context |
| **Canonical similarity** | Corridor overlap used during route deduplication (DS-006) |

---

## Three-Level Proximity Model

The corridor is the first level of the proximity model defined in ADR-019:

| Level | Radius | Purpose |
|-------|--------|---------|
| **Tight route adjacency** | 25–75m | Features that physically affect the ridden road |
| **Route-side POI adjacency** | 100–200 yards | Features visibly close — "I can see it from the road" |
| **Dynamic enrichment horizon** | Varies | Reachable support that expands/contracts by context and remoteness |

See ADR-019 for the full proximity model and dynamic search behavior rules.

---

## Guiding Principle

The corridor represents the experienced road environment, not a mathematical line. Corridor width must adapt to the density of the surrounding road network to remain both accurate and computationally tractable.


---

## Source File: docs/02-architecture/design/ds-009-route-corridor-fingerprint-spec.md

# DS-009 — Route Corridor Fingerprint Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-026 (Canonical Route Identity)  
**Related specs:** DS-006 (Route Canonicalization), DS-005 (Canonical Route Schema), DS-008 (Route Corridor Model)

---

## Purpose

Provide a fast method to detect similar or duplicate routes before expensive geometry comparison.

Full geometry comparison (Fréchet distance, Hausdorff distance) is computationally expensive. Route fingerprints allow fast candidate filtering — comparing fingerprint hashes takes microseconds; comparing full geometries takes seconds.

---

## Fingerprint Components

A route fingerprint is derived from five characteristics, each capturing a different aspect of the route experience:

| Component | What it captures |
|-----------|----------------|
| `road_class_sequence` | The sequence of road environments along the route |
| `bearing_signature` | Simplified direction changes — the route's shape |
| `segment_length_signature` | Relative segment length pattern |
| `surface_signature` | Surface transitions along the route |
| `elevation_profile_signature` | Terrain pattern |

---

## Component Definitions

### Road Class Sequence
The normalized sequence of road class transitions.

```
Example: tertiary → tertiary → secondary → residential
```

Captures the road environment without storing full geometry.

### Bearing Signature
Route direction changes represented as simplified bearing buckets.

```
Example: 45° → 60° → 90° → 120°
```

Captures route shape. Two routes with the same shape but different locations will share a similar bearing signature.

### Surface Signature
Surface type transitions along the route.

```
Example: asphalt → asphalt → gravel → asphalt
```

Differentiates road routes from gravel routes following the same corridor. A critical signal for distinguishing route variants.

### Elevation Profile Signature
Terrain pattern in simplified form.

```
Example: climb → climb → flat → descent
```

Captures the route's terrain character without storing full elevation data.

---

## Fingerprint Hash

Fingerprint components are serialized and hashed into a single `fingerprint_hash` value stored on `canonical_routes`.

This hash is used as the first-pass candidate filter in the canonicalization pipeline (DS-006). Routes with sufficiently different fingerprints are excluded from full geometry comparison.

---

## Usage

| Use case | How |
|----------|-----|
| Candidate lookup | Find routes with similar fingerprints in the same geographic area |
| Duplicate detection | Flag near-identical routes for review |
| Clustering | Group similar routes for analysis |
| Canonicalization speed | Eliminate non-candidates before expensive comparison |

---

## Limitations

Fingerprints are **approximate** — they are a fast filter, not a definitive identity test.

- Two different routes may share a similar fingerprint (false positive)
- A route with minor GPS noise may produce a slightly different fingerprint from an otherwise identical route (false negative)

Final canonical route decisions must always use full geometry comparison (DS-006). **Fingerprints eliminate expensive comparisons — they do not replace them.**

---

## Guiding Principle

Fingerprints are a gate before identity, not identity itself. Use them to eliminate candidates quickly. Never use them as the final word on whether two routes are the same.


---

## Source File: docs/02-architecture/design/ds-010-slice-analysis-cache-spec.md

# DS-010 — Slice Analysis Cache Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-020 (Atomic Analysis Unit)  
**Related ADRs:** ADR-017 (Local OSM-Derived Data Strategy), ADR-018 (Server-Cached Slice Analysis Model)  
**Related specs:** DS-005 (Canonical Route Schema), DS-007 (Route Slice Generation), DS-001 (Route Intelligence Pipeline)

---

## Purpose

Define how Lanterne caches analysis results at the slice level.

The cache prevents recomputation and enables large-scale route analysis by allowing routes sharing the same corridor to reuse cached slice results. It is the primary defense against Overpass rate limiting and redundant computation.

---

## Cache Scope

Analysis is cached per slice:

```
slice
   └ cached analysis result (keyed by slice_hash + analysis_version)
```

Route analysis is assembled by aggregating slice-level cache hits.

---

## Cache Key

```
cache_key = slice_hash + analysis_version

slice_hash = hash(slice_geometry + osm_tags)
```

If the geometry or OSM tags for a slice change, the hash changes and the cache misses — triggering fresh computation.

---

## Cached Metrics

| Metric | Description |
|--------|-------------|
| `traffic_index` | 0–100 |
| `bike_support_index` | 0–100 |
| `remoteness_index` | 0–100 |
| `surface_quality_index` | 0–100 |
| `fatigue_baseline` | Slice fatigue contribution |
| `descent_risk` | 0–100 |
| `analysis_metadata` | Version, confidence, breakdown |

Optional component breakdown values may also be stored for explainability and debugging.

---

## Cache Invalidation

Slice analysis must be recomputed when:
- `analysis_engine_version` changes
- Slice geometry changes
- OSM data affecting the slice changes (tag changes, way splits, geometry edits)

---

## Tile-Based Road Fetching

Road data is fetched using a **fixed 0.05° grid tile system** rather than route-polyline chunks.

**Benefits of tile-based fetching:**
- Route-independent cache sharing — different routes traversing the same tiles share the same fetched road data
- Per-tile retry logic when Overpass returns errors
- Match quality tracking per tile
- Avoidance of Overpass rate limiting (429/504 errors) by reusing cached tile data

---

## Road Density Guardrail

A guardrail fires when the normalized road density in memory exceeds **500 roads/mile** (total roads ÷ route distance in miles).

**On violation:**
- Log a warning: `[GUARDRAIL] HIGH_ROAD_DENSITY: N roads/mi exceeds 500 limit`
- Attach `highRoadDensity: true` to `analysisWarnings`
- **Never block results from rendering**

The guardrail is **informational only**. It signals that the adaptive corridor (DS-008) should have narrowed the road set further, but it must not suppress output.

**What the guardrail must not do:**
- Return early or throw
- Prevent the results card from showing
- Suppress any downstream analysis output

---

## Cache Storage

Slice cache is stored server-side via Supabase (`tile_cache` table).

The cache layer sits between the analysis engine and Overpass:

```
analysis request
    ↓
tile cache lookup (Supabase tile_cache)
    ↓
cache hit → return cached road data
    ↓
cache miss → fetch from Overpass → store in tile_cache
```

---

## Performance Goal

Once analyzed, a slice should rarely need to be recomputed. Cache hit rate should approach 100% for routes traversing previously analyzed regions.

---

## Guiding Principle

The cache is what makes Lanterne fast after first analysis. Fetch once, cache, reuse. The guardrail warns — it never blocks.


---

## Source File: docs/02-architecture/design/ds-011-ride-time-situational-awareness-interface-spec.md

# DS-011 — Ride-Time Situational Awareness Interface Spec

**Status:** Draft  
**Date:** 2026

**ADR parent:** ADR-029 (Ride-Time Situational Awareness Mode)  
**Related ADRs:** ADR-027 (Lantern Screen Model), ADR-030 (Ride Mode Display, Power, and Sensor Architecture)  
**Related specs:** DS-012 (Ride Computer Tile System)

> **Note:** ADR-029 is currently Proposed. This spec is written against its direction and will be confirmed when ADR-029 is accepted.

---

## Purpose

Define the visual and interaction design for Lanterne's ride-time interface.

Ride-time mode surfaces **environmental and navigational context** while the rider is moving. The interface must be glanceable while riding, readable at arm's length, cognitively minimal, and map-aware.

The interface is intentionally **not a cycling computer dashboard**. Its job is to answer one question:

> *"What is the environment doing to me right now?"*

---

## Relationship to Lanterne Architecture

Ride-time mode exposes data already produced by the Lanterne system. No new data sources are introduced — the interface is a presentation layer over existing outputs.

| Signal type | Source |
|-------------|--------|
| Wind, temperature, precipitation | Environmental modeling layer |
| Sun glare, moonlight | Astronomical modeling (ADR-010, ADR-009) |
| Route bearing, distance to next cue | Navigation context |
| Remoteness, hazard warnings | Slice intelligence layer |

---

## Display Philosophy

**Avoid data cockpit design.** Most bike computers attempt to show many metrics simultaneously. Lanterne does the opposite.

**Rules:**
- Show only a few signals at once
- Make them large
- Eliminate clutter
- Let the map remain visible

---

## Signal Types

### Navigation Signals
- Current speed
- Distance to next turn
- Distance to destination
- Route bearing

### Environment Signals
- Wind direction relative to travel bearing
- Temperature
- Precipitation state
- UV exposure
- Sun glare risk
- Moonlight conditions

### Context Signals
- Remoteness warning
- Upcoming hazard
- Long descent warning

Context signals derive from slice-level route intelligence — they are not live sensor data.

---

## Screen Layout System

The layout adapts to the number of signals shown.

### One signal — large center display
```
        18 mph
```
Used for speed, wind indicator, distance to next turn.

### Two signals — vertical thirds
```
       18 mph
  ────────────
       64°F
```

### Four signals — quadrant layout
```
  18 mph   ↗ wind
  64°F     1.3 mi
```

**Map preservation rule:** Signals must never cover the rider position indicator. If conflict occurs, signals shift upward automatically.

---

## Typography

**Hierarchy:**
- Value: very large
- Unit: small, beneath value
- Label: small, quiet

**Example:**
```
18
mph
```

Large typography prioritizes readability at distance and in motion over information density.

---

## Icon System

Environmental signals prefer **icons** instead of text where possible.

| Signal | Icon |
|--------|------|
| Wind | Directional arrow (↗) — always relative to rider bearing, not compass |
| Sun glare | ☀︎ |
| Moonlight | ☾ |
| Precipitation | ☂︎ |

Icons must remain readable at large scale with a single glance.

---

## Wind Visualization

Wind must always be shown **relative to rider bearing**, not compass direction.

```
↗  =  wind coming from front-right
```

This is more useful than "Wind: NW" — the rider needs to know how it affects them, not its absolute direction.

---

## Map Contrast

| Map style | Text color |
|-----------|-----------|
| Light map | Charcoal |
| Dark map | White |
| Satellite | Dynamic contrast sampling from background |

---

## Lantern Interaction

Screens are navigated using the Lantern control per ADR-027.

| Gesture | Action |
|---------|--------|
| Scroll lantern | Screen scrolls |
| Release lantern | Screen locks |

No tap confirmation required.

---

## Screen Order

**Default sequence:**
1. Lantern home
2. Ride instrumentation (DS-012)
3. Environmental signals
4. Field notes (future)

**Visibility rules:**
- Instrumentation screen activates only when GPS tracking is active
- Navigation signals appear only when a route is engaged
- Controls do not appear if the underlying capability is unavailable

---

## Motion Behavior

Transitions should feel calm and deliberate:
- Smooth horizontal scroll between screens
- No sudden animation
- No flashing alerts
- Signals update quietly

---

## Error Handling

If signal data is unavailable, the signal **simply does not appear**. Do not show placeholder text such as "data unavailable." The interface should remain clean regardless of signal availability.

---

## Safety Rule

The rider should never need to read a paragraph. Every signal must be interpretable in **under one second**.

---

## Strategic Role

Ride-time mode allows Lanterne to function as:
- A **second screen companion** to cycling computers
- A **primary display** for riders who prioritize environmental awareness over instrument data

---

## Design Constraint

```
clarity > density
context > metrics
glanceability > precision
```


---

## Source File: docs/02-architecture/design/ds-012-ride-computer-tile-system-spec.md

# DS-012 — Ride Computer Tile System Spec

**Status:** Draft  
**Date:** 2026

**ADR parents:** ADR-027 (Lantern Screen Model), ADR-029 (Ride-Time Situational Awareness Mode), ADR-030 (Ride Mode Display, Power, and Sensor Architecture)  
**Related specs:** DS-011 (Ride-Time Situational Awareness Interface)

> **Note:** ADR-029 and ADR-030 are currently Proposed. This spec is written against their direction and will be confirmed when those ADRs are accepted. This spec supersedes the earlier DS-008 tile system draft.

---

## Purpose

Define the ride computer tile system used inside Lanterne ride mode.

This system gives riders access to a minimal but credible set of live ride metrics while preserving Lanterne's primary role as a route intelligence system — not a fitness dashboard.

**The tile system must feel:**
- Calm
- Intentional
- Map-first
- Powerful without looking dense

---

## Design Principle

The ride computer is not a separate product inside Lanterne. It is a **live metric layer** within ride mode.

> *"This gives me the few core ride stats I need, plus road intelligence my bike computer doesn't."*

**The system must not drift into:** power dashboard design, workout screen design, tiny-metric grid clutter, or Garmin clone behavior.

---

## Core Concept

Ride mode contains **tiles**. Each tile has:
- A **category**
- One or more **sub-metrics**
- A **front face** default
- Optional alternate faces

A tile is not static text. It is a small, touchable metric object with hidden faces.

---

## V1 Supported Categories and Sub-Metrics

| Category | Sub-Metrics |
|----------|------------|
| **Time** | elapsed time, moving time, total ride time, ETA to destination |
| **Speed** | current speed, average speed |
| **Distance** | distance traveled, distance remaining |
| **Elevation** | elevation gain, current grade |
| **Route** | route progress, distance to next turn, distance to destination |

Categories are intentionally limited. Every category must answer a clear rider question.

---

## Default Curated Layout

Lanterne ships with a polished default. Users are not asked to build their own screen from scratch.

| Position | Default |
|----------|---------|
| Top left | Time → elapsed time |
| Top right | Speed → current speed |
| Bottom left | Distance → distance traveled |
| Bottom right | Route → route progress |

**Rule:** Lanterne owns the first-run layout. After the first user customization, the rider owns the layout.

---

## Persistence Rule

Out-of-box defaults apply only until the rider changes something. After the first edit:
- Tile category choices persist
- Tile sub-metric choices persist
- Tile layout density persists
- Persistence is per user / per device until synced account preferences exist

User customization must survive app relaunch.

---

## Layout Model

Each half of the screen is independently configurable.

| Half | Options |
|------|---------|
| Top | Single tile or double tile |
| Bottom | Single tile or double tile |

**V1 layout combinations:**

| Layout | Tiles |
|--------|-------|
| 1 top / 1 bottom | 2 tiles |
| 2 top / 1 bottom | 3 tiles |
| 1 top / 2 bottom | 3 tiles |
| 2 top / 2 bottom | 4 tiles |

Do not support denser layouts yet. Readability while riding matters more than feature count.

---

## Interaction Grammar

| Gesture | Action |
|---------|--------|
| **Tap tile** | Advance to next category: Time → Speed → Distance → Elevation → Route |
| **Swipe left/right on tile** | Advance through sub-metrics within the current category |
| **Long press tile** | Enter layout edit mode for that half of the screen |
| **Tap lantern** | Clear overlays / return to pure map state |
| **Long press lantern** | Reveal temporary affordances and edit hints |

---

## Animation Model

Tile changes must **not** use generic fades or slides.

| Change type | Animation |
|-------------|-----------|
| Category change | **Vertical flip** — implies the tile is changing topic |
| Sub-metric change | **Horizontal flip** — implies the tile has another face |

The effect should feel like a physical instrument tile turning over — subtle and quick.

---

## Typography Rules

**Font:** Georgia for large display typography.

**Hierarchy:**

| Element | Size |
|---------|------|
| Value | Very large |
| Unit | Small, beneath value |
| Label | Small, quiet |

**Example:**
```
18.4
mph
```

**Sizing rule:** The smaller the conceptual payload, the larger it can appear. Speed and temperature can be huge. Route progress and ETA are smaller.

---

## Color Rules

| Map style | Tile text |
|-----------|----------|
| Light map | Charcoal |
| Dark map | White |
| Satellite | White |

No heavy card chrome. No boxed dashboard panels. Tiles float anchored to the map.

---

## Speed Smoothing Rule

Current speed must feel responsive without jitter. Do not use naïve always-on 3-second smoothing.

| Riding state | Behavior |
|-------------|---------|
| From stop (0→moving) | Near-real-time speed |
| Stable cruising | 3-second smoothing |
| Approaching stop | Reduced smoothing — speed drops quickly |

The rider should feel snappy starts, a stable cruising number, and quick stop recognition.

---

## Ride Mode as "God Mode"

Ride mode is not a locked page. It is the rider's live operational layer and may surface both ride metrics and Lanterne intelligence simultaneously.

Ride mode can coexist with: hazards, stops, weather, route context, navigation state.

**Separation:**
- **Tiles** = local metric objects
- **Lantern** = global ride/overlay controller

---

## Edit Model

**V1 edit model** — fast, in-place, no settings page:
- Tap tile → change category
- Swipe tile → change sub-metric
- Long press half → change half layout
- Long press lantern → reveal edit hints

**V1 non-goals:**
- No drag-and-drop tile editor
- No full layout builder
- No multi-screen settings matrix

---

## Underlying State Model

```typescript
type TileCategory = 'time' | 'speed' | 'distance' | 'elevation' | 'route';

type TimeMetric = 'elapsed_time' | 'moving_time' | 'total_time' | 'eta';
type SpeedMetric = 'current_speed' | 'average_speed';
type DistanceMetric = 'distance_traveled' | 'distance_remaining';
type ElevationMetric = 'elevation_gain' | 'current_grade';
type RouteMetric = 'route_progress' | 'distance_to_next_turn' | 'distance_to_destination';

type HalfLayout = 'single' | 'double';

type MetricTile = {
  category: TileCategory;
  subMetric: string;
};

type RideComputerLayout = {
  topLayout: HalfLayout;
  bottomLayout: HalfLayout;
  topTiles: MetricTile[];
  bottomTiles: MetricTile[];
  customized: boolean;  // default layout applies only when false
};
```

---

## Persistence Requirements

Persist: top layout, bottom layout, tile category per position, tile sub-metric per position, `customized` flag.

Persistence begins with local storage and may move to user profile sync later.

---

## V1 Non-Goals

Do not add: power, cadence, heart rate, lap metrics, workout intervals, training zones, full Karoo-style page editor. These push Lanterne toward fitness-device parity and away from its differentiation.

---

## Guiding Principle

> *"Just enough bike computer, plus the part nobody else has."*

Not: a weaker Garmin clone.


---

## Source File: docs/02-architecture/design/ds-013-comparative-traffic-context-schema-spec.md

# DS-013 — Comparative Traffic Context Schema Spec

**Status:** Draft  
**Date:** 2026-03-23

**ADR parents:**
- [ADR-032 — Comparative Traffic Context and Segment Cohorts](../adrs/adr-032-comparative-traffic-context-and-segment-cohorts.md)
- [ADR-033 — Canonical Segment Identity and Route-to-Canonical Mapping](../adrs/adr-033-canonical-segment-identity.md)

**Related ADRs:** ADR-020 (Atomic Analysis Unit), ADR-026 (Canonical Route Identity)  
**Related specs:** DS-005 (Canonical Route Schema)

---

## Scope

This spec defines the greenfield schema for Lanterne's three-layer comparative traffic context model, incorporating the canonical segment identity model from ADR-033.

**Covers:**
- `canonical_segments` — stable directed segment identity
- `route_segment_instances` — route-analysis-local mapping layer
- `segment_behavior_inputs` — canonical per-segment traffic behavior facts
- `traffic_behavior_baselines` — regional and contextual comparison priors
- `cohorts` — typed catalog of comparative groups
- `segment_cohort_memberships` — many-to-many segment-to-cohort associations
- `segment_observations` — raw evidence landing zone for future sensor and rider data

**Does not cover:**
- Scoring logic changes
- Baseline data population
- Rider-facing relative context UI
- Varia/radar ingestion pipeline (observations table is a stub only)
- Backfilling legacy route analyses to canonical segments

All tables are greenfield. Existing Lanterne tables (`tile_cache`, `route_cache`, `route_history`, etc.) are unaffected.

---

## Naming Discipline

| Prefix | Meaning |
|--------|---------|
| `observed_*` | Measured in the field from real rides or sensors |
| `inferred_*` | Deterministic derivation from known segment truth |
| `predicted_*` | Model output |
| `baseline_*` | Regional or cohort prior — belongs in `traffic_behavior_baselines` |
| `confidence_*` | Evidence strength, 0..1 |
| `score_*` | Normalized Lanterne output — belongs in scoring tables, not here |

Do not mix provenance levels in the same column. Do not use generic names like `traffic_value` or `behavior_metric`.

---

## 1. `canonical_segments`

**Purpose:** Stable, directed segment identity. The long-lived entity that traffic facts, cohorts, and observations attach to. One canonical segment per directed semantic truth segment, independent of any route.

```sql
create table canonical_segments (
  id uuid primary key default gen_random_uuid(),

  -- Identity scaffold
  network_source text not null,              -- osm / dot / custom
  direction text not null,                   -- forward / backward / undirected
  segmentation_schema_version integer not null,

  -- Anchor keys: snapped coordinate string at 5dp, format '{lat},{lon}'
  -- This is the v1 matching key. OSM node IDs are opportunistic enrichment only.
  start_anchor_key text not null,
  end_anchor_key text not null,
  start_anchor_type text not null default 'snapped_coord',  -- snapped_coord / osm_node
  end_anchor_type text not null default 'snapped_coord',

  -- OSM node IDs: nullable, populated opportunistically when analysis emits node refs
  start_osm_node_id text,
  end_osm_node_id text,

  geometry_hash_normalized text not null,    -- hash of simplified directed geometry
  semantic_signature text not null,          -- deterministic hash of: speed bucket,
                                             -- lane count, bike facility class, shoulder class
  map_snapshot_version text,                 -- OSM or source snapshot this was derived from

  -- Geometry
  geometry jsonb,                            -- GeoJSON LineString, directed

  -- Provenance
  country_code text,
  admin1_code text,
  road_class text,
  urbanicity_class text,

  -- Lineage
  is_active boolean not null default true,
  superseded_by_id uuid references canonical_segments(id),

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index on canonical_segments (network_source, direction, start_anchor_key, end_anchor_key);
create index on canonical_segments (geometry_hash_normalized);
create index on canonical_segments (semantic_signature);
create index on canonical_segments (is_active);
create index on canonical_segments (country_code, admin1_code);
create index on canonical_segments (start_osm_node_id) where start_osm_node_id is not null;
create index on canonical_segments (end_osm_node_id) where end_osm_node_id is not null;
```

**Notes:**
- `semantic_signature` is a deterministic hash of speed bucket + lane count + bike facility class + shoulder class
- `start_anchor_key` / `end_anchor_key` are snapped coordinate strings at 5 decimal places (format: `{lat},{lon}`). 5dp gives ~1.1m grid precision — sufficient to distinguish close urban intersections while surviving GPS noise
- `start_anchor_type` / `end_anchor_type` record whether the key came from a coordinate snap or an OSM node id
- `start_osm_node_id` / `end_osm_node_id` are nullable; populated opportunistically when the analysis pipeline emits boundary node refs; **never a hard matching dependency**
- DOT and non-OSM sources always use `snapped_coord`; OSM node IDs are never assumed available
- `is_active = false` when superseded; `superseded_by_id` points to the replacement
- `geometry` is stored as GeoJSON for portability; spatial index deferred until PostGIS is available

---

## 2. `route_segment_instances`

**Purpose:** Route-analysis-local mapping layer. Preserves per-analysis segment occurrence, order, and the resolved canonical mapping result. One row per segment occurrence in a specific route analysis pass.

```sql
create table route_segment_instances (
  id uuid primary key default gen_random_uuid(),

  route_id uuid not null references route_history(id) on delete cascade,
  analysis_id text,                          -- analysis pass identifier if tracked separately
  segment_index integer not null,            -- ordinal position within this route analysis

  -- Route-local identity from analysis output
  local_segment_key text not null,           -- current matcher output key; preserved for audit only
  local_road_id text,
  local_geometry jsonb,                      -- GeoJSON slice of route geometry for this instance

  -- Canonical mapping result
  canonical_segment_id uuid references canonical_segments(id),
  match_method text,                         -- exact / near_exact / new / unresolved
  match_confidence numeric,                  -- 0..1

  created_at timestamptz not null default now(),

  unique (route_id, segment_index)
);

create index on route_segment_instances (route_id);
create index on route_segment_instances (canonical_segment_id);
create index on route_segment_instances (match_method);
```

**Notes:**
- `canonical_segment_id` is nullable; unresolved until the canonical mapper runs
- `local_segment_key` is preserved for audit and backfill but must **not** be used as a join key in canonical tables
- `match_method = 'unresolved'` is the v1 default for all rows written before the canonical mapper exists
- This is the **only** table where `route_id` appears in the segment schema

---

## 3. `segment_behavior_inputs`

**Purpose:** Canonical per-segment traffic behavior facts, keyed by `canonical_segment_id`. Answers: *what do we know or infer about how vehicles behave around cyclists on this segment?*

```sql
create table segment_behavior_inputs (
  id uuid primary key default gen_random_uuid(),

  canonical_segment_id uuid not null references canonical_segments(id),

  -- Geographic scope (denormalized from canonical_segments for query convenience)
  country_code text,
  admin1_code text,
  metro_key text,
  road_class text,
  urbanicity_class text,

  -- Inferred from segment truth (deterministic)
  inferred_posted_speed_mph numeric,
  inferred_aadt numeric,
  inferred_lane_count integer,
  inferred_shoulder_width_m numeric,
  inferred_bike_facility_class text,         -- none / shared / lane / protected / path
  inferred_oneway boolean,
  inferred_parking_presence boolean,

  -- Predicted traffic behavior (model outputs; nullable until model exists)
  predicted_passes_per_mile numeric,
  predicted_vehicle_speed_mph numeric,
  predicted_driver_slowdown_mph numeric,

  -- Confidence
  confidence_overall numeric,                -- 0..1
  evidence_sources jsonb not null default '[]'::jsonb,

  -- Lineage
  model_version integer,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (canonical_segment_id)
);

create index on segment_behavior_inputs (canonical_segment_id);
create index on segment_behavior_inputs (country_code, admin1_code);
create index on segment_behavior_inputs (road_class, urbanicity_class);
```

**Future columns** *(add when ingestion pipeline or model exists — do not add prematurely):*

```sql
  -- Observed (from real ride data or sensor imports)
  observed_passes_per_mile numeric,
  observed_passes_per_hour numeric,
  observed_vehicle_speed_mph numeric,
  observed_speed_delta_mph numeric,
  observed_driver_slowdown_mph numeric,
  observed_pass_duration_sec numeric,
  observed_sample_count integer,
  observed_ride_count integer,
  observed_distance_miles numeric,

  -- Extended predicted fields
  predicted_passes_per_hour numeric,
  predicted_speed_delta_mph numeric,
  predicted_pass_duration_sec numeric,
  predicted_close_pass_rate numeric,

  -- Extended confidence
  confidence_pass_frequency numeric,
  confidence_vehicle_speed numeric,
  confidence_driver_behavior numeric,
  evidence_summary jsonb not null default '{}'::jsonb,

  -- Extended inferred
  inferred_intersection_density_per_mile numeric,
  inferred_signal_density_per_mile numeric,
  inferred_centerline_class text,
```

**Notes:**
- `route_id` does not appear in this table; it belongs in `route_segment_instances` only
- Geographic fields are denormalized from `canonical_segments` for query performance; keep in sync on insert
- `unique (canonical_segment_id)` enforces one fact row per canonical segment; update in place rather than inserting duplicates

---

## 4. `traffic_behavior_baselines`

**Purpose:** Regional and contextual comparison priors for benchmarking, explanation, and future model calibration. **Not used to rescale the headline Safety Score.**

```sql
create table traffic_behavior_baselines (
  id uuid primary key default gen_random_uuid(),

  baseline_version integer not null default 1,

  geography_level text not null,             -- country / admin1 / metro / custom_region
  geography_key text not null,               -- US / US-HI / US-HI-HONOLULU / etc.

  road_class text,                           -- null = all road classes
  urbanicity_class text,                     -- null = all urbanicity bands
  bike_facility_class text,                  -- null = all facility classes

  sample_ride_count integer,
  sample_distance_miles numeric,

  -- Central tendency
  baseline_passes_per_mile numeric,
  baseline_vehicle_speed_mph numeric,
  baseline_driver_slowdown_mph numeric,

  -- Percentile bands (required; averages alone cannot support relative context explanations)
  p25_passes_per_mile numeric,
  p50_passes_per_mile numeric,
  p75_passes_per_mile numeric,
  p90_passes_per_mile numeric,

  p25_vehicle_speed_mph numeric,
  p50_vehicle_speed_mph numeric,
  p75_vehicle_speed_mph numeric,
  p90_vehicle_speed_mph numeric,

  p25_driver_slowdown_mph numeric,
  p50_driver_slowdown_mph numeric,
  p75_driver_slowdown_mph numeric,
  p90_driver_slowdown_mph numeric,

  confidence_baseline numeric,               -- 0..1
  source_summary jsonb not null default '{}'::jsonb,

  created_at timestamptz not null default now()
);

-- Unique index required (coalesce expressions cannot be used in a table constraint in Postgres)
create unique index traffic_behavior_baselines_scope_uq
on traffic_behavior_baselines (
  baseline_version,
  geography_level,
  geography_key,
  coalesce(road_class, ''),
  coalesce(urbanicity_class, ''),
  coalesce(bike_facility_class, '')
);

create index on traffic_behavior_baselines (geography_level, geography_key);
create index on traffic_behavior_baselines (road_class, urbanicity_class);
```

**Notes:**
- Percentiles are required, not optional — central tendency alone cannot support "higher than typical" explanations
- This table starts mostly empty at v1; populate as baseline data is sourced or computed
- Do **not** import MyBikeTraffic aggregate stats without explicit written permission; treat as inspiration only until rights are confirmed
- p10 omitted from v1 for table cleanliness; add when useful

---

## 5. `cohorts`

**Purpose:** Typed catalog of all comparative groups a canonical segment can belong to.

```sql
create table cohorts (
  id uuid primary key default gen_random_uuid(),

  cohort_type text not null,                 -- geography / urbanicity / road_class / event /
                                             -- route_family / usage / custom
  cohort_key text not null unique,           -- stable machine key,
                                             -- e.g. geo:us-hi / event:ironman-kona / family:rusa-600k
  name text not null,
  description text,

  is_system boolean not null default true,   -- false = user or admin-created cohort
  definition_json jsonb,                     -- optional: geometry, filter rules, membership criteria

  created_at timestamptz not null default now()
);
```

**v1 seed rows** *(geography, urbanicity, road class — required per ADR-032 §8):*

```sql
insert into cohorts (cohort_type, cohort_key, name) values
  -- Geography (expand as routes are analyzed in new states)
  ('geography',   'geo:us',             'United States'),
  ('geography',   'geo:us-hi',          'Hawaii'),
  ('geography',   'geo:us-nj',          'New Jersey'),
  ('geography',   'geo:us-ca',          'California'),
  ('geography',   'geo:us-tx',          'Texas'),
  ('geography',   'geo:us-ny',          'New York'),

  -- Urbanicity
  ('urbanicity',  'urban:urban',        'Urban'),
  ('urbanicity',  'urban:suburban',     'Suburban'),
  ('urbanicity',  'urban:rural',        'Rural'),
  ('urbanicity',  'urban:remote',       'Remote'),

  -- Road class
  ('road_class',  'road:primary',       'Primary Highway'),
  ('road_class',  'road:secondary',     'Secondary Road'),
  ('road_class',  'road:tertiary',      'Tertiary Road'),
  ('road_class',  'road:residential',   'Residential'),
  ('road_class',  'road:path',          'Path / Trail');
```

**Notes:**
- `cohort_key` must be stable — it is referenced by `segment_cohort_memberships` and baselines
- Event and route family cohorts (Ironman, RUSA, etc.) are valid future rows; do not block on them
- `is_system = false` reserved for future admin-curated or user-defined cohorts

---

## 6. `segment_cohort_memberships`

**Purpose:** Many-to-many association layer between canonical segments and cohorts.

```sql
create table segment_cohort_memberships (
  canonical_segment_id uuid not null references canonical_segments(id) on delete cascade,
  cohort_id uuid not null references cohorts(id) on delete cascade,

  membership_source text not null,           -- derived / curated / observed / imported
  membership_weight numeric,                 -- optional; 0..1 for weighted comparisons
  notes text,

  created_at timestamptz not null default now(),

  primary key (canonical_segment_id, cohort_id)
);

create index on segment_cohort_memberships (canonical_segment_id);
create index on segment_cohort_memberships (cohort_id);
```

**Notes:**
- `membership_source = 'derived'` covers automatic assignment from canonical segment attributes (e.g. `country_code → geo:us-hi`, `urbanicity_class → urban:urban`)
- `membership_source = 'curated'` covers manually assigned cohorts (Ironman corridor, RUSA perm, etc.)
- Derived memberships are created when canonical segments are created. Because canonical mapping is deferred in v1, cohort assignment is also deferred — **do not wire this into the analysis path yet**
- `membership_weight` is nullable at v1

---

## 7. `segment_observations`

**Purpose:** Raw evidence landing zone for future rider-contributed or sensor-imported data. Raw evidence must **not** be written directly into `segment_behavior_inputs` — it must pass through this table and a quality/review step first.

```sql
create table segment_observations (
  id uuid primary key default gen_random_uuid(),

  -- Canonical linkage is nullable; observations may land before mapping is complete
  canonical_segment_id uuid references canonical_segments(id),

  observation_type text not null,    -- radar_pass_summary / speed_sign_tap / field_note / imported_fit
  source_type text not null,         -- user / device / import / admin
  source_user_id uuid references auth.users(id),

  observed_at timestamptz not null,
  ride_context jsonb not null default '{}'::jsonb,
  payload jsonb not null,

  quality_score numeric,             -- 0..1; populated by review or ingestion logic
  review_status text not null default 'unreviewed',  -- unreviewed / accepted / rejected

  created_at timestamptz not null default now()
);

create index on segment_observations (canonical_segment_id);
create index on segment_observations (review_status);
create index on segment_observations (observation_type);
```

**Notes:**
- This is a stub at v1 — table exists, no ingestion pipeline required yet
- `canonical_segment_id` is nullable to allow ingestion before canonical mapping resolves
- Varia radar pass summaries are the primary anticipated future input: `observation_type = 'radar_pass_summary'`
- `payload` is intentionally schemaless at this stage; structure will emerge from real ingestion patterns
- `review_status` gate prevents raw observations from affecting analysis until validated

---

## 8. What to Build Now vs Later

### Build now

| Item | Notes |
|------|-------|
| All seven tables with v1 columns | As specified above |
| v1 cohort seed rows | Geography, urbanicity, road class |
| All indexes listed per table | Including partial indexes on OSM node ID fields |

### Defer until canonical mapper exists

| Item | Notes |
|------|-------|
| Canonical segment creation logic | Matching, anchor key generation, semantic signature computation |
| Derived cohort membership auto-assignment | Runs at canonical segment creation time only |
| `segment_behavior_inputs` population | Requires canonical IDs to exist |
| `segment_cohort_memberships` population | Requires canonical IDs to exist |

### Defer indefinitely

| Item | Notes |
|------|-------|
| `observed_*` columns in `segment_behavior_inputs` | Add when ingestion pipeline exists |
| Baseline data population | Data sourcing work |
| Event and route family cohort rows | Future curation |
| `segment_observations` ingestion pipeline | Future sensor work |
| Rider-facing percentile / z-score surfaces | Future UX work |
| Weighted cohort membership logic | Future analytical work |
| `canonical_segment_supersessions` table | `superseded_by_id` on `canonical_segments` is sufficient for v1 |

### v1 analysis path behavior

Route analysis writes `route_segment_instances` immediately on every analysis pass.

**Default values until the canonical mapper is introduced:**

| Field | v1 default |
|-------|------------|
| `canonical_segment_id` | `null` |
| `match_method` | `'unresolved'` |
| `match_confidence` | `null` |

The canonical mapper is introduced as an explicit admin/batch operation. **No synchronous canonical matching in v1.**

---

## 9. Resolved Decisions

### Anchor key format — resolved 2026-03-23

**Decision: hybrid coordinate/node model (Option C)**

| Aspect | Decision |
|--------|----------|
| v1 default key | Snapped coordinate string at 5dp, format `{lat},{lon}` |
| 5dp precision | ~1.1m grid — distinguishes urban intersections, survives GPS noise |
| `anchor_type` field | Records `snapped_coord` or `osm_node` |
| OSM node IDs | Nullable opportunistic enrichment; never a hard matching dependency |
| DOT / non-OSM sources | Always use `snapped_coord` |

### Mapping pass timing — resolved 2026-03-23

**Decision: deferred with explicit trigger (Option C)**

- Route analysis writes `route_segment_instances` immediately on every pass
- `canonical_segment_id = null`, `match_method = 'unresolved'` until the mapper runs
- No canonical matching in the live analysis path in v1
- The canonical mapper is introduced later as an explicit admin/batch operation
- Nothing rider-facing is blocked on canonical IDs in v1

### Backfill strategy — resolved 2026-03-23

**Decision: opportunistic on re-analysis, batch deferred (Option B)**

**Phase 1 (now):**
- When a legacy route is re-analyzed, `route_segment_instances` are written for that pass
- Rows remain unresolved until the canonical mapper exists
- `local_segment_key` and `local_geometry` are preserved for future audit and backfill

**Phase 2 (once canonical mapper is proven stable):**
- Re-analyzed routes become the natural first candidates for canonical mapping
- No forced full-database migration until the matcher is proven on fresh data

**If explicit batch backfill is pursued later:**
- Only accept legacy canonical matches at or above the near-exact reuse floor: **directed overlap ≥ 0.85**
- Lower-confidence candidates remain unresolved pending review or improved matching logic
- Consistent with ADR-033's bias: prefer duplicates over false merges

---

## Design Principle

These tables do not change what Lanterne scores.
They change what Lanterne **knows**, **compares**, and can eventually **explain**.

> Build the schema now. Populate it incrementally. Keep the Safety Score absolute throughout.
> Route analyses produce occurrences. Canonical segments accumulate knowledge.


---

## Source File: docs/02-architecture/design/ds-014-route-expedition-state-and-windowed-analysis-spec.md

# DS-014 — Route Expedition State and Windowed Analysis Spec

**Status:** Draft  
**Date:** 2026-03-24

**ADR parent:** ADR-034 (Master Route Expeditions and Windowed Long-Route Analysis)  
**Related ADRs:** ADR-026 (Canonical Route Identity), ADR-033 (Canonical Segment Identity), ADR-016 (Ride Session Data Model)  
**Related specs:** DS-005 (Canonical Route Schema), DS-010 (Slice Analysis Cache), DS-008 (Route Corridor Model)

---

## Scope

This spec defines the persistence model for expedition-grade route continuity and bounded long-route analysis.

**Covers:**
- `route_expeditions` — durable multi-day rider progress on a route
- `route_expedition_windows` — per-expedition detailed analysis windows with overlap
- `route_expedition_events` — sparse append-only audit trail for starts, resumes, pauses, and progress checkpoints

**Also defines:**
- Operational defaults for detailed-window sizing
- Resume and mismatch behavior contracts
- What is durable versus what remains transient runtime state

**Does not cover:**
- Canonical segment identity or segment-matching logic
- Full GPS tick storage
- Scoring changes
- Full offline routing of the entire route corpus
- UI pixel design
- Route geometry storage refactors in `route_history`

---

## Assumptions

- `route_history` remains the current per-user route record and is the route foreign key used here
- Route geometry already exists in the current route pipeline; this spec does not require duplicating full route geometry into expedition tables
- Browser/localStorage session continuity may continue to exist, but it is **not** the durable system of record for expedition progress

---

## Operational Policy Defaults

These values should be configurable in code. The schema captures them at expedition creation time so they can vary per expedition without requiring a code change.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `target_detail_miles` | 250 | Target window size |
| `max_detail_miles` | 400 | Hard maximum per window |
| `window_overlap_miles` | 10 | Overlap at each window boundary |
| `preload_trigger_miles` | 25 | Distance before window end to queue next window |

Windowing should be used whenever a route exceeds safe one-shot analysis budgets. The exact trigger may combine distance, point count, tile budget, wall-clock heuristics, corridor complexity, and corridor width. **Adaptive corridor width is a first-class budget lever** and may reduce the working-set cost of a window without changing expedition truth. Mileage alone is an imperfect proxy for cost.

---

## Naming Discipline

| Prefix | Meaning |
|--------|---------|
| `expedition_*` | Durable rider journey state |
| `window_*` | Bounded operational working set |
| `event_*` | Sparse recovery/audit history |

Do not use generic names like `session_state` or `chunk_data` for durable expedition persistence. Do not use window identity as a proxy for route identity.

---

## 1. `route_expeditions`

**Purpose:** The durable multi-day progress record for one rider on one route. One row represents one active or historical expedition. This is the **database source of truth** for where the rider is in the larger journey.

```sql
create table route_expeditions (
  id uuid primary key default gen_random_uuid(),

  user_id uuid not null references auth.users(id) on delete cascade,
  route_id uuid not null references route_history(id) on delete cascade,

  expedition_name text,

  expedition_status text not null default 'planned'
    check (expedition_status in ('planned', 'active', 'paused', 'completed', 'abandoned')),

  entry_mode text not null
    check (entry_mode in (
      'from_beginning',
      'resume_existing',
      'from_current_location',
      'from_custom_mile',
      'from_custom_point'
    )),

  detail_mode text not null default 'windowed'
    check (detail_mode in ('windowed', 'full')),

  -- Operational window policy captured at expedition start
  target_detail_miles numeric not null default 250,
  max_detail_miles    numeric not null default 400,
  window_overlap_miles numeric not null default 10,
  preload_trigger_miles numeric not null default 25,

  -- Route entry point
  start_route_mile     numeric not null default 0,
  start_route_fraction numeric,
  start_point_index    integer,

  -- Durable progress state
  last_confirmed_route_mile     numeric not null default 0,
  last_confirmed_route_fraction numeric,
  last_matched_point_index      integer,
  last_matched_lat              numeric,
  last_matched_lon              numeric,
  last_match_confidence         numeric,
  last_progress_source text
    check (last_progress_source in (
      'gps_match', 'manual_adjustment', 'resume_recovery', 'system'
    )),
  last_progress_at timestamptz,

  -- Active window tracking
  active_window_index integer,
  next_window_index   integer,

  -- Route metadata snapshot for convenience
  route_total_miles  numeric,
  route_point_count  integer,

  -- Bounded freeform metadata
  notes jsonb not null default '{}'::jsonb,

  started_at   timestamptz,
  paused_at    timestamptz,
  completed_at timestamptz,
  abandoned_at timestamptz,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  check (target_detail_miles > 0),
  check (max_detail_miles > 0),
  check (window_overlap_miles >= 0),
  check (preload_trigger_miles >= 0),
  check (
    last_match_confidence is null
    or (last_match_confidence >= 0 and last_match_confidence <= 1)
  )
);

-- Only one open expedition per user/route at a time
create unique index route_expeditions_one_open_per_user_route
on route_expeditions (user_id, route_id)
where expedition_status in ('planned', 'active', 'paused');

create index on route_expeditions (user_id, expedition_status);
create index on route_expeditions (route_id);
create index on route_expeditions (last_progress_at);
create index on route_expeditions (active_window_index);
```

**Notes:**
- This table is **durable expedition state**, not transient runtime session state. Turning GPS off must not delete this row
- `detail_mode = 'full'` is allowed for shorter routes that fit within safe one-shot budgets; ultra routes should use `'windowed'`
- `start_route_mile` and `last_confirmed_route_mile` are route-distance truth. Lat/lon alone is **not** sufficient for resume logic
- A rider may have many historical expeditions on the same route, but only **one open expedition** per user/route should exist at a time (enforced by partial unique index)
- Window policy fields are captured at expedition start so they can vary per expedition
- **v1 migration note:** `route_id` references `route_history(id)`, not a canonical route. When canonical routes exist, a rider uploading the same route twice will get two different `route_id` values and potentially two separate expedition records for the same journey. Future migration should reference `canonical_route_id` instead. This is acceptable for v1.

---

## 2. `route_expedition_windows`

**Purpose:** The bounded detailed-analysis windows for one expedition. These rows define what subsection of the full route receives full corridor analysis, heatmap rendering, and navigation context. Each window has a rider-visible core span and an analysis load span that includes overlap.

```sql
create table route_expedition_windows (
  id uuid primary key default gen_random_uuid(),

  expedition_id uuid not null references route_expeditions(id) on delete cascade,
  window_index  integer not null,

  -- Rider-visible planned span
  core_start_mile numeric not null,
  core_end_mile   numeric not null,

  -- Actual analysis working set including overlap
  load_start_mile numeric not null,
  load_end_mile   numeric not null,

  -- Route point boundaries for deterministic slicing
  core_start_point_index integer,
  core_end_point_index   integer,
  load_start_point_index integer,
  load_end_point_index   integer,

  window_status text not null default 'planned'
    check (window_status in (
      'planned', 'queued', 'analyzing', 'ready',
      'active', 'completed', 'failed', 'stale'
    )),

  analysis_version   integer,
  analysis_cache_key text,
  route_cache_key    text,

  ready_at     timestamptz,
  activated_at timestamptz,
  completed_at timestamptz,
  failed_at    timestamptz,
  failure_reason text,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),

  unique (expedition_id, window_index),

  check (window_index >= 0),
  check (core_start_mile >= 0),
  check (core_end_mile > core_start_mile),
  check (load_start_mile >= 0),
  check (load_end_mile > load_start_mile),
  check (load_start_mile <= core_start_mile),
  check (load_end_mile >= core_end_mile)
);

create index on route_expedition_windows (expedition_id, window_index);
create index on route_expedition_windows (expedition_id, window_status);
create index on route_expedition_windows (analysis_cache_key);
```

**Notes:**
- `core_*` fields are the **rider-facing planned span**
- `load_*` fields are the **true analysis/rendering span** and must include overlap
- The constraint `load_start_mile <= core_start_mile` and `load_end_mile >= core_end_mile` enforces that the load span always fully contains the core span
- Window identity is expedition-local. It must **not** be reused as canonical route or segment identity
- `analysis_cache_key` / `route_cache_key` are optional linkage points to existing cache artifacts for future per-window materialized analysis results
- `'failed'` status must preserve `failure_reason`; failure to analyze one window must not erase existing expedition progress

---

## 3. `route_expedition_events`

**Purpose:** Sparse append-only audit trail for important expedition transitions and recovery checkpoints.

This table is **intentionally not a full GPS log**. It exists to support auditability, resume debugging, manual recovery, and coarse historical reconstruction.

```sql
create table route_expedition_events (
  id uuid primary key default gen_random_uuid(),

  expedition_id uuid not null references route_expeditions(id) on delete cascade,

  event_type text not null
    check (event_type in (
      'started',
      'resumed',
      'paused',
      'progress_checkpoint',
      'window_queued',
      'window_ready',
      'window_activated',
      'window_failed',
      'manual_reposition',
      'completed',
      'abandoned'
    )),

  source_type text not null default 'system'
    check (source_type in ('system', 'gps', 'manual', 'admin')),

  route_mile     numeric,
  route_fraction numeric,
  point_index    integer,
  lat            numeric,
  lon            numeric,
  window_index   integer,

  payload  jsonb not null default '{}'::jsonb,
  event_at timestamptz not null default now()
);

create index on route_expedition_events (expedition_id, event_at desc);
create index on route_expedition_events (event_type);
create index on route_expedition_events (window_index);
```

**Notes:**
- Do **not** write every GPS sample here — this is a sparse checkpoint table
- Write only: start, resume, pause, meaningful progress save, window transitions, manual position changes, completion, abandonment
- `payload` is bounded metadata only; do not dump large route geometry blobs into it
- The append-only nature supports crash recovery and audit without requiring mutable history

---

## 4. Window Planning Rules

The system should create expedition windows at expedition start or when the rider changes the intended detailed stage length.

**Recommended v1 rule:**
- Clamp chosen detailed span to `max_detail_miles`
- Create sequential core windows from the chosen start mile forward
- Apply overlap to produce load windows
- Allow shorter windows when corridor-cost heuristics indicate the current working-set budget is tight
- Mark the first window `'planned'` or `'queued'`

**Example with `target_detail_miles = 400` and `overlap = 10`:**

| Window | Core span | Load span |
|--------|-----------|-----------|
| 0 | 0–400 mi | 0–410 mi |
| 1 | 400–800 mi | 390–810 mi |
| 2 | 800–1200 mi | 790–1210 mi |

This preserves one clean rider-visible progression while giving the renderer and cue system buffer around seams.

---

## 5. Resume Contract

On app boot or route reopen, resume logic must consult `route_expeditions` first. The runtime session cache is secondary.

**Expected contract:**

| Case | Condition | Behavior |
|------|-----------|---------|
| **A** | Current GPS near last confirmed route progress | Offer one-tap resume; reopen active window; start new live session under same expedition |
| **B** | Current GPS materially far from last confirmed progress | Show mismatch; require explicit rider choice; do not silently advance or rewind |
| **C** | GPS not yet available | Still restore last known route progress and active window context |

**Durable fields required for resume:**
- `last_confirmed_route_mile`
- `last_matched_point_index`
- `last_progress_at`
- `active_window_index`

Do not rely solely on raw latitude/longitude for resume.

---

## 6. What Stays Transient

The following remain runtime or client-state concerns in v1 and do **not** require durable SQL storage here:

| Transient state | Where it lives |
|-----------------|----------------|
| Second-by-second GPS samples | Memory / local storage |
| Current elapsed timer tick | Memory |
| Current screen mode and UI state | Memory |
| Immediate map camera position | Memory |
| Volatile cue-advance internals | Memory |

These are not the expedition system of record.

---

## 7. What to Build Now vs Later

### Build now

| Item | Notes |
|------|-------|
| `route_expeditions` | Full v1 columns and indexes |
| `route_expedition_windows` | Full v1 columns and indexes |
| `route_expedition_events` | Full v1 columns and indexes |
| One-open-expedition partial unique index | Enforces one active expedition per user/route |
| Update path for progress fields | `last_confirmed_route_mile`, `last_matched_point_index`, `active_window_index` |
| Sparse event writes | start / resume / pause / progress_checkpoint / window transitions |

### Defer

| Item | Notes |
|------|-------|
| Cross-device live session sync | Future capability |
| Full-route overview geometry in SQL | Only if not already covered by existing route storage |
| Per-window materialized analysis-result tables | After v1 proven stable |
| Adaptive window sizing from device telemetry | Requires measured performance data |
| Batch analytics over expedition events | Future analytics work |
| Automatic overnight stop inference | Beyond basic persisted progress |

---

## 8. Resolved Decisions

### Window trigger rule — resolved 2026-03-24

**Decision: compound trigger (any condition forces windowed mode)**

| Condition | Threshold | Why |
|-----------|-----------|-----|
| Route distance | > 400 miles | Mileage hard cap |
| GPX point count | > 8,000 points | Memory budget |
| Estimated road density | > 500 roads/mile | Reuses existing guardrail logic |

All three conditions are cheap to evaluate before any analysis runs. The first two require only route metadata. The third reuses the `MAX_ROADS_IN_MEMORY` guardrail already implemented in the analysis pipeline. If any condition is true, `detail_mode = 'windowed'` is set automatically at expedition creation.

This gives v1 a concrete executable rule without requiring measured device telemetry.

### Progress checkpoint cadence — resolved 2026-03-24

**Decision: dual-condition trigger (both must be true)**

| Condition | Value |
|-----------|-------|
| Minimum distance since last checkpoint | 2 miles |
| Minimum time since last checkpoint | 10 minutes |

Both conditions must be met before a `progress_checkpoint` event is written and the `route_expeditions` row is updated. This is a single upsert — not expensive. The 2-mile / 10-minute floor prevents database spam on a 200-mile day while giving useful recovery resolution (worst-case loss: 2 miles of progress).

These values are tunable constants in code. They are not schema-locked.

### Resume mismatch threshold — resolved 2026-03-24

**Decision: 2-mile projected distance threshold**

| GPS distance from last confirmed route mile (projected onto route geometry) | Behavior |
|-----------------------------------------------------------------------------|---------|
| ≤ 2 miles | Offer one-tap resume — no mismatch warning |
| > 2 miles | Show mismatch clearly — require explicit rider choice |

"Distance" here is the rider's current GPS position projected onto the route geometry and compared to `last_confirmed_route_mile` — not raw straight-line distance from last GPS point. This prevents false mismatches from GPS drift while correctly catching genuine position disagreements.

2 miles is a tunable constant. It is not schema-locked.

### Active-window preload policy — resolved 2026-03-24

**Decision: distance-remaining trigger for v1; connectivity/battery deferred**

The next window is queued when the rider is within `preload_trigger_miles` of the current window's `core_end_mile` (default: 25 miles). This is pure distance-based and uses the existing `preload_trigger_miles` field already in the schema.

Connectivity and battery state awareness is deferred to a future version. In v1, the preload trigger fires unconditionally when the distance threshold is crossed. If the preload fails (no connectivity, analysis error), `window_status = 'failed'` is set with `failure_reason` preserved, and the rider is warned before they reach the window boundary.

---

## Design Principle

Persist the journey. Bound the working set. Do not confuse session recovery with expedition continuity.


---

## Source File: docs/03-adrs/adr-000-README.md

# Architecture Decision Records

ADRs capture the historical decisions that shaped Lanterne's architecture.

Architecture documents describe the system as it exists today.

When conflicts appear, architecture documents reflect the current system. ADRs provide historical context.

---

## Source File: docs/03-adrs/adr-001-route-acquisition-model.md

# ADR-001 — Route Acquisition Model

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-002 (Vault Concept), ADR-026 (Canonical Route Identity)  
**No companion DS required** — this ADR is self-contained.

---

## Context

Routes enter Lanterne through several different paths. The UI must reflect a mental model that maps to how route geometry actually appears on the map, rather than mixing user intent with data source labels.

---

## Decision

Lanterne recognizes three ways a route enters the system:

| Action | Label | Meaning |
|--------|-------|---------|
| `generate` | **Route To** | Generate a route to a destination |
| `create` | **Draw** | Draw a route manually |
| `open` | **Open** | Open an existing route |

**Open** includes the following sources:

- Vault
- RWGPS
- GPX
- History

---

## Rationale

This model maps directly to how route geometry appears on the map. It avoids mixing user intent (generate vs draw) with data provenance (RWGPS vs GPX), keeping the UI coherent and predictable.

---

## Consequences

**Advantages:**
- UI labels match user mental model
- Data sources are grouped under a single "Open" action rather than cluttering the top-level UI
- Extensible — new sources (Komoot, Strava, etc.) slot cleanly under Open

**Tradeoffs:**
- "Route To" implies navigation which may need clarification as the product evolves

---

## Design Principle

Route acquisition labels should reflect what the rider is doing, not where the data came from.


---

## Source File: docs/03-adrs/adr-002-vault-concept.md

# ADR-002 — Vault Concept

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-001 (Route Acquisition Model), ADR-003 (Mode-Aware Vault Filtering)  
**No companion DS required** — this ADR is self-contained.

---

## Context

Lanterne serves riders who participate in curated events and permanent routes — RUSA brevets, gran fondos, bikepacking routes, and similar. These routes should be available natively inside the app as a first-class collection, distinct from personally imported files.

---

## Decision

Lanterne contains a **Vault** representing curated route collections.

```
Vault
 ├ RUSA 2026 Events
 ├ Randonneurs Canada
 ├ TransVA Grand Depart
 └ Endless Gravel PA
```

**Vault contains collections, not files.**

Collections are:
- Curated by Lanterne
- Mode-aware (see ADR-003)
- Native to the Lanterne experience

External ingestion (RWGPS / GPX) is **not** part of the Vault. Those live under Open in the acquisition model (ADR-001).

---

## Rationale

A Vault of curated collections gives Lanterne a native content layer that no generic file importer can replicate. It creates a reason to open Lanterne even before a rider has their own route file — and it aligns with the randonneuring community's culture of organized, named events.

---

## Consequences

**Advantages:**
- Native content layer distinct from personal imports
- Supports community-facing product positioning
- Collections can carry metadata (event dates, controls, organizer info) that raw GPX cannot

**Tradeoffs:**
- Requires curation effort to keep collections current
- Scope of Vault content needs governance as the library grows

---

## Design Principle

The Vault is Lanterne's native route library. It is curated, not crowd-sourced, and it is never just a file browser.


---

## Source File: docs/03-adrs/adr-003-mode-aware-vault-filtering.md

# ADR-003 — Mode-Aware Vault Filtering

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-002 (Vault Concept), ADR-007 (Index Families)  
**No companion DS required** — this ADR is self-contained.

---

## Context

Lanterne supports multiple riding modes — randonneuring, gravel, bikepacking, and potentially others. Each mode influences analysis weighting, graph presentation, and POI emphasis. The Vault should reflect the active mode rather than presenting all collections at once regardless of context.

---

## Decision

Vault collections may be filtered by **Mode**.

```
Randonneuring
 ├ RUSA 2026 Events
 └ Randonneurs Canada

Gravel
 └ Endless Gravel PA

Bikepacking
 └ TransVA Grand Depart
```

---

## Rationale

Mode already controls analysis weighting, graphs, and POI visibility throughout the app. Applying the same lens to Vault collections keeps the UI coherent — a rider in randonneuring mode sees RUSA events surfaced, not bikepacking routes. The Vault becomes an extension of the mode experience rather than a separate context switch.

---

## Consequences

**Advantages:**
- Vault content stays relevant to the rider's current context
- Reduces cognitive load — fewer irrelevant collections surfaced
- Consistent with mode-aware behavior elsewhere in the app

**Tradeoffs:**
- Collections must be tagged with mode(s) during curation
- A route that fits multiple modes (e.g. a gravel permanent) needs explicit multi-mode assignment

---

## Design Principle

The Vault should feel like it understands what kind of riding you are planning. Mode filtering is the mechanism that makes that possible.


---

## Source File: docs/03-adrs/adr-004-rider-field-notes-deferred.md

# ADR-004 — Rider Field Notes (Deferred)

**Status:** Deferred — explicitly post-alpha  
**Date:** 2026

**Related ADRs:** ADR-028 (Field Note Confirmation Model)  
**No companion DS required** — DS-016 will cover implementation when deferred status is lifted.

---

## Context

A Field Notes feature was discussed during early product design. The feature would allow riders to attach geolocated observations to routes and locations — hazards, water sources, dogs, construction, unofficial amenities, and similar.

The feature was explicitly deferred until post-alpha to keep the initial scope manageable.

---

## Decision

Field Notes are **deferred until post-alpha**.

When implemented, the intent is:

```
route
 └ field notes
      └ rider observations
```

**Key design principles for when implementation begins:**
- Notes are **structured**, not freeform text blobs
- **Optional mile markers** may be attached
- **No threaded comments** — this is observational, not conversational
- Notes are observational in nature, not social

---

## Rationale

Field Notes are a meaningful long-term feature for the randonneuring community, where ground-truth observations about water sources, hazards, and services have real value. However, they require a trust model, a confirmation model, and UI surface area that is premature for alpha.

The design principles are recorded here so the feature is implemented correctly when the time comes, rather than being designed from scratch under time pressure.

The confirmation and trust model is specified in ADR-028.

---

## Consequences

**When implemented:**
- Enables community-sourced route intelligence
- Ground-truth observations can upgrade confidence on OSM-derived data
- Rider reports can eventually feed the segment observation layer (DS-013)

**Tradeoff of deferral:**
- No community intelligence layer at alpha
- Some early riders may expect a notes or hazard reporting feature

---

## Design Principle

Field Notes are observations, not conversations. When the time comes, build them structured and trust-aware — not as a comment thread.


---

## Source File: docs/03-adrs/adr-005-route-analysis-model.md

# ADR-005 — Route Analysis Model

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-011 (Route Slice Model), ADR-020 (Atomic Analysis Unit)  
**No companion DS required** — see DS-001 (Route Intelligence Pipeline) and DS-007 (Route Slice Generation) for implementation.

---

## Context

Route analysis must capture how conditions change along a route — traffic, remoteness, lighting, surface quality, and similar. The choice of analysis unit determines how accurately those changes are captured.

---

## Decision

Route analysis is performed on **small internal slices of the route**, not large segments.

Display segments may later aggregate these slices for readability, but the underlying analysis always operates at slice granularity.

---

## Rationale

Large segments smooth over critical changes:

- A dangerous half-mile disappears inside a 10-mile average
- Remoteness dips near towns are invisible
- Traffic variation across an intersection is lost
- Lighting and weather changes have sub-mile timing windows that large segments cannot represent

Small slices allow indices to reflect real-world conditions along the route. The display layer can always aggregate upward for readability — but aggregation can never recover truth that was never captured.

---

## Consequences

**Advantages:**
- Accurate modeling of route variation
- Short dangerous sections are preserved rather than averaged away
- Environmental conditions can be mapped to precise arrival windows
- Future analytics (fatigue accumulation, worst-mile detection) require this granularity

**Tradeoffs:**
- Higher storage requirements than large-segment approaches
- Requires aggregation logic for display

---

## Design Principle

Analyze small. Display aggregated. Never let display granularity determine analysis granularity.


---

## Source File: docs/03-adrs/adr-006-safety-definition.md

# ADR-006 — Safety Definition

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-007 (Index Families), ADR-032 (Comparative Traffic Context and Segment Cohorts)  
**No companion DS required** — this ADR is self-contained. The safety score is implemented as part of the core scoring engine.

---

## Context

Safety is a word that means different things to different people. Without a precise definition, the Safety Score risks absorbing unrelated factors — weather discomfort, rough surfaces, fatigue — that dilute its meaning and undermine trust.

Lanterne must define safety narrowly and defend that definition as the product evolves.

---

## Decision

**Safety is defined narrowly as:**

> The likelihood of a rider being struck by a motor vehicle, and the severity of the resulting injury.

**The Safety Score is derived from:**
- Traffic Index
- Bike Support Index

**Environmental conditions are explicitly excluded from the Safety Score:**
- Wind
- Temperature
- Precipitation
- Light state
- UV
- Surface quality

These conditions belong in other index families (see ADR-007) and may be presented alongside safety, but they must never contaminate the Safety Score itself.

---

## Rationale

Keeping safety narrowly defined does three things:

1. **Preserves alignment with traffic safety research.** The core safety question for a cyclist on a public road is about motor vehicle interaction. That is what the research literature measures and what the score should reflect.

2. **Prevents score dilution.** A safety score that includes weather, surface, and fatigue becomes an averaged comfort score — not a safety score. A rider in light rain on a protected bike path should score differently from a rider in sunshine on a 55mph highway shoulder.

3. **Maintains trust.** Riders who understand what the score means will trust it. A score that seems to go up and down based on unrelated factors quickly loses credibility.

---

## Consequences

**Advantages:**
- Safety Score is meaningful and defensible
- Score is stable across weather and conditions that don't affect motor vehicle risk
- Aligns with the broader architecture principle of keeping index families separate (ADR-007)

**Tradeoffs:**
- Some riders may expect a broader "overall risk" score that includes weather
- The system must clearly communicate what the Safety Score does and does not include

---

## Design Principle

Safety means motor vehicle risk. Everything else is a different question with a different answer.


---

## Source File: docs/03-adrs/adr-007-index-families.md

# ADR-007 — Index Families

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-006 (Safety Definition), ADR-025 (Fatigue Index as Extensible Model Family)  
**No companion DS required** — this ADR is self-contained.

---

## Context

Lanterne computes many different kinds of route intelligence. Without explicit grouping, these indices risk being treated as interchangeable or averaged together in ways that destroy their meaning — particularly the Safety Score, which must remain narrow and uncontaminated.

---

## Decision

Indices are grouped into **three conceptual families**.

### Safety Family
Inputs to the core Safety Score. These reflect motor vehicle risk only (per ADR-006).

| Index | Role |
|-------|------|
| Safety Score | Rider-facing composite |
| Traffic Index | Motor vehicle exposure |
| Bike Support Index | Infrastructure protection from traffic |

### Route Reality Family
Stable route characteristics independent of time and conditions.

| Index | Role |
|-------|------|
| Remoteness Index | Distance from services, civilization, and bailout options |
| Surface Quality Index | Rideability of the surface |
| Fatigue Index | Cumulative physical burden of the route |
| Descent Risk Index | Technical descent exposure |

### Conditions Family
Time-dependent overlays computed from forecast and astronomical data.

| Index / Signal | Role |
|----------------|------|
| Wind | Speed and direction relative to rider bearing |
| Temperature | Thermal exposure along route timeline |
| Precipitation | Rain/hail/snow probability and intensity |
| Light | Daylight, twilight, night state per slice |

---

## Rationale

Separating these families prevents unrelated factors from contaminating safety metrics. A rider choosing between two routes needs to know the safety difference independently of which one has headwinds or rough gravel — those are real concerns but they are different questions.

The family structure also gives the UI a natural organization: safety first, then route reality, then conditions overlay.

---

## Consequences

**Advantages:**
- Safety Score stays clean and defensible
- Riders can evaluate safety, route character, and conditions independently
- Index families provide a natural UI organization
- New indices slot into an existing family rather than floating unanchored

**Tradeoffs:**
- Some indices could reasonably belong to more than one family (e.g. surface quality affects fatigue)
- Family boundaries must be maintained as new indices are added

---

## Design Principle

Keep index families separate. Mixing safety with comfort produces neither a safety score nor a comfort score — just noise.


---

## Source File: docs/03-adrs/adr-008-environmental-light-system.md

# ADR-008 — Environmental Light System

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-007 (Index Families), ADR-009 (Sun and Moonlight), ADR-010 (Sun Glare Detection)  
**No companion DS required** — this ADR is self-contained.

---

## Context

Cycling conditions change dramatically with light — sunrise and sunset introduce glare hazards, twilight reduces visibility, and nighttime riding on rural roads with only moonlight is a materially different experience from daytime riding. Lanterne needs a system for modeling and surfacing these conditions in a way riders intuitively understand.

---

## Decision

Lanterne models riding light conditions using a **sun/moon visual system**.

**Daytime:**
- Sun icon representing current solar position
- UV intensity halo indicating exposure level
- Sun glare detection for segments where bearing approaches the sun (see ADR-010)

**Night:**
- Moon phase icon indicating available natural illumination
- Cloud cover overlay modifying effective moonlight
- Tooltips providing simple plain-language explanations

---

## Rationale

Riders intuitively understand sun and moon conditions without needing numeric output. A sun icon with a glare warning communicates danger faster than "solar azimuth 87°, bearing delta 12°." A moon phase icon immediately conveys whether a nighttime stretch will have natural illumination.

This system sits in the Conditions family (ADR-007) — it is a contextual overlay, not part of the Safety Score.

---

## Consequences

**Advantages:**
- Intuitively legible without numeric clutter
- Sun and moon icons are universally understood
- Glare detection adds safety-relevant context for dawn/dusk riding
- Moonlight visibility is directly relevant to randonneurs riding overnight

**Tradeoffs:**
- Icon-based system requires careful design to avoid ambiguity
- Cloud cover interaction with moonlight adds modeling complexity

---

## Design Principle

Light conditions should be communicated visually and intuitively. Riders glancing at the app while planning should understand the light story without reading numbers.


---

## Source File: docs/03-adrs/adr-009-sun-and-moonlight.md

# ADR-009 — Sun and Moonlight Visualization

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-008 (Environmental Light System), ADR-010 (Sun Glare Detection), ADR-007 (Index Families)  
**No companion DS required** — this ADR is self-contained.

---

## Context

Long-distance cycling frequently involves riding at night, especially in randonneuring where overnight riding is the norm rather than the exception. The amount of natural illumination available dramatically affects the riding experience — a full moon on a clear night provides meaningful visibility; a new moon with heavy cloud cover means near-total darkness on rural roads without streetlights.

---

## Decision

Lanterne represents moonlight and sunlight conditions using a visual icon system:

**Sun:**
- Sun icon with UV intensity representation
- Solar position modeled per slice based on rider arrival time
- Feeds into sun glare detection (ADR-010)

**Moon:**
- Moon phase icon (crescent through full)
- Cloud cover overlay modifying effective illumination
- Tooltips providing plain-language context when needed

**Effective illumination model:**

| Moon Phase | Clear Sky | Partly Cloudy | Overcast |
|------------|-----------|---------------|----------|
| Full | High | Moderate | Low |
| Gibbous | Moderate | Low | Very low |
| Quarter | Low | Very low | Dark |
| Crescent | Very low | Dark | Dark |
| New | Dark | Dark | Dark |

---

## Rationale

Moonlight dramatically changes the night riding experience and is especially relevant to randonneurs completing 200–1200km events that span multiple nights. A waxing gibbous on a clear night on open farmland roads is a qualitatively different ride from a new moon under cloud cover on forested roads.

This information is free to compute (astronomical calculations are deterministic) and is genuinely useful to the target audience. No other cycling app surfaces it meaningfully.

---

## Consequences

**Advantages:**
- Directly relevant to the randonneuring audience
- Astronomical calculations are deterministic — no external data dependency for the moon phase model
- Cloud cover interaction requires forecast data but degrades gracefully if unavailable

**Tradeoffs:**
- Moon phase is a "treat" feature — high value for overnight riders, minimal value for day riders
- Icon design must communicate phase and cloud cover clearly at small sizes

---

## Design Principle

For riders spending nights on the road, the moon is infrastructure. Model it accordingly.


---

## Source File: docs/03-adrs/adr-010-sun-glare-detection.md

# ADR-010 — Sun Glare Detection

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-008 (Environmental Light System), ADR-009 (Sun and Moonlight Visualization), ADR-011 (Route Slice Model)  
**No companion DS required** — this ADR is self-contained.

---

## Context

One of the most dangerous conditions for cyclists on public roads is not rain or darkness — it is low sun glare. When the sun is near the horizon at sunrise or sunset, drivers approaching from behind or from the side may be effectively blinded. A rider traveling eastward at dawn or westward at dusk may be nearly invisible to overtaking traffic during a glare window that lasts only 20–40 minutes but represents a materially elevated collision risk.

Lanterne can predict this condition for any route given a start time and pace, because solar position is deterministic and rider bearing per slice is known.

---

## Decision

Lanterne will detect and flag **sun glare risk windows** along the route based on the intersection of:

1. **Solar position** — sun azimuth and elevation computed per slice using rider estimated arrival time and slice location
2. **Rider bearing** — direction of travel on each slice
3. **Glare threshold** — sun elevation within a defined window above the horizon where glare is most dangerous to drivers

**Glare risk is flagged when all three conditions are met:**
- Sun elevation is between approximately **0° and 6° above the horizon** (peak blind-driver zone)
- The sun azimuth is within approximately **±30° of the rider's direction of travel** (driver approaching from behind is facing the sun) or within **±30° of the opposite bearing** (oncoming traffic is sun-blinded)
- The rider is estimated to be on that slice during the relevant glare window

**Output per slice:**
- `glare_flag` boolean
- `glare_direction` — `behind` (drivers overtaking) / `oncoming` / `both`
- `glare_severity` — based on sun elevation within the window and bearing alignment tightness

---

## Rationale

This is a computable prediction that no other cycling app surfaces. The calculation requires only:
- Slice geometry (bearing)
- Rider estimated arrival time (from the timeline model)
- Astronomical solar position (deterministic, no external API needed)

The risk is real and well-documented in road safety research — dawn and dusk are disproportionately dangerous times for pedestrians and cyclists precisely because of driver glare.

For randonneurs, this is especially relevant: a 300km route starting at 6am will have riders on exposed roads during the evening glare window, and a rider starting a 200km at 3pm will hit the morning glare window the following day. Knowing which segments fall in that window before the ride is genuinely useful planning information.

---

## Glare Window Definition

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Sun elevation lower bound | 0° (horizon) | Below horizon = no glare |
| Sun elevation upper bound | ~6° | Above ~6° drivers can look away; glare diminishes |
| Bearing delta threshold | ±30° | Tighter alignment = higher severity |
| Peak danger zone | 1°–3° elevation | Sun fully in windshield zone |

These values may be tuned based on real-world validation. The 6° upper bound is a starting point derived from traffic safety research on glare-related incidents.

---

## Consequences

**Advantages:**
- Fully computable from existing data — no new external dependencies
- Surfaces a real, underappreciated risk that other apps ignore
- Particularly valuable for long-distance riders whose routes cross dawn/dusk windows
- Integrates naturally with the slice model and timeline model

**Tradeoffs:**
- Requires accurate rider arrival time estimates — error in the timeline model propagates to glare prediction
- Cloud cover modifies glare risk (overcast sky eliminates it) — integration with forecast data adds complexity
- Must degrade gracefully when forecast data is unavailable (flag potential glare, note cloud cover unknown)

---

## Design Principle

Sun glare is a driver behavior problem that creates a cyclist safety problem. Lanterne can predict it. Flag it.


---

## Source File: docs/03-adrs/adr-011-route-slice-model.md

# ADR-011 — Route Slice Model

**Status:** Accepted  
**Date:** 2026

**Related ADRs:** ADR-005 (Route Analysis Model), ADR-020 (Atomic Analysis Unit), ADR-023 (Predicted vs Observed Condition Layers)  
**Companion specs:** DS-007 (Route Slice Generation), DS-005 (Canonical Route Schema), DS-010 (Slice Analysis Cache)

> **Note:** ADR-020 supersedes and expands the slice model detail from this ADR. ADR-011 is the original decision record. ADR-020 is the authoritative technical specification of the atomic analysis unit. Both are retained for lineage.

---

## Context

Lanterne analyzes cycling routes using multiple indices:

- Traffic Index
- Bike Support Index
- Remoteness Index
- Surface Quality Index
- Fatigue Index
- Descent Risk Index
- Environmental conditions (wind, temperature, light)

These characteristics change frequently along a route:

- A quiet rural road suddenly entering a town
- A safe shoulder disappearing on a bridge
- A remote stretch interrupted by a service stop
- A sunrise glare window appearing for a short section

Traditional route analysis divides routes into large segments, which smooths away this variation. Lanterne needs a finer unit.

---

## Decision

Routes are internally divided into **fixed-length analysis slices**.

Slices are the **atomic unit of route intelligence**. All indices and environmental conditions are calculated at the slice level.

Slices are later aggregated into:
- Display segments for visualization
- Route-level rollups for summary scores

---

## Slice Characteristics

**Typical slice length:** ~200–500 meters  

**Slice boundaries are determined by:**
- Distance thresholds
- Road transitions
- Intersection boundaries
- Surface changes
- Infrastructure changes
- Environmental timing changes (lighting, glare windows)

---

## What Each Slice Stores

**Geometry and road attributes:**
- Distance, bearing, road classification
- Surface type, infrastructure attributes, hazard flags

**Derived indices:**
- Traffic Index, Bike Support Index, Remoteness Index
- Surface Quality Index, Fatigue contribution, Descent Risk

**Environmental state (when modeled):**
- Wind, temperature, precipitation, light state, sun glare risk

**Derived metadata:**
- Expected arrival time, traffic-time multiplier, confidence signals

---

## Three Distinct Units

| Unit | Purpose | Rider-visible? |
|------|---------|----------------|
| Slice | Internal analysis unit | No |
| Display segment | UI visualization | Yes |
| Route rollup | Rider-facing summary | Yes |

Slices remain invisible to riders but enable accurate modeling. The display layer aggregates slices for readability.

---

## Storage Strategy

Slices are stored in a dedicated table: `route_slices`

Each slice references: `route_id`, `slice_index`, `geometry`, `distance`, `bearing`, `road_id`

Associated analysis data lives in related tables:
- `route_slice_indices`
- `route_slice_conditions`
- `route_slice_hazards`

This prevents slice rows from becoming overly wide while allowing indices to evolve independently.

---

## Predicted vs Observed Data

Slices may contain two parallel data layers per ADR-023:

**Predicted layer** — derived from OSM, DOT data, weather forecasts, astronomical models  
**Observed layer** — derived from ride uploads, telemetry, radar observations, rider reports

Observed data does not overwrite predicted data. Both layers remain available for calibration and model refinement.

---

## Rollup Strategies

Slices are aggregated to produce rider-facing metrics. Different metrics use different strategies:

| Strategy | Used for |
|----------|---------|
| Weighted mean | Traffic Index, Bike Support Index |
| Percentile emphasis | Worst-section exposure |
| Worst-stretch weighting | Remoteness, Descent Risk |
| Cumulative | Fatigue Index |

This prevents short dangerous sections from being hidden by route averages.

---

## Rationale

Using slices preserves the true shape of the ride experience. Large segments introduce truth smoothing, environmental drift errors, rollup distortion, and limit future extensibility. Slices unlock the entire analysis model.

---

## Consequences

**Advantages:**
- High-resolution route intelligence
- Accurate environmental modeling along the ride timeline
- Future ride-time telemetry comparison
- More meaningful detour and variant scoring
- Advanced analytics such as fatigue accumulation

**Tradeoffs:**
- Increased storage volume
- Additional aggregation logic
- More complex queries

---

## Design Principle

The rider does not experience a route as a single average condition. They experience a sequence of changing conditions along the road. Route slices are the system's representation of that reality.


---

## Source File: docs/03-adrs/adr-012-predicted-vs-experienced-conditions.md

# ADR-012 — Predicted vs Experienced Conditions

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-023 (Predicted vs Observed Condition Layers), ADR-016 (Ride Session Data Model), ADR-011 (Route Slice Model)  
**No companion DS required** — see ADR-023 for the expanded decision and DS-015 (reserved) for the future schema spec.

> **Note:** ADR-023 supersedes and expands this ADR. ADR-012 is the original decision record focused on ride-time conditions. ADR-023 covers the broader predicted/observed separation across all condition layers.

---

## Context

Lanterne models ride conditions such as wind, temperature, precipitation, light state, and glare windows. These conditions are both:

1. **Predicted** before the ride — computed from forecasts, solar position, and the rider's planned start time
2. **Experienced** during the ride — recorded from actual conditions encountered

Treating these as the same data would make it impossible to accurately record the rider's experience or compare what was predicted against what actually happened.

---

## Decision

The system separates conditions into **two layers**.

### Predicted Ride Conditions

Derived from:
- Forecast weather
- Solar position calculations
- Rider start time
- Estimated arrival time along route slices

Conditions are calculated in **~15-minute time buckets** aligned to the rider's expected arrival at each slice.

Predictions may be recalculated if forecasts are updated before the ride.

### Experienced Ride Conditions

During a ride session, Lanterne records **snapshots** of actual conditions encountered.

**Snapshot contents:**

| Field | Description |
|-------|-------------|
| `timestamp` | When the snapshot was taken |
| `location` | lat/lon at time of snapshot |
| `wind` | Speed and direction |
| `temperature` | Actual temperature |
| `precipitation` | Active precipitation |
| `light_state` | Daylight / twilight / night |
| `uv` | UV index |
| `glare_flag` | Active glare condition |
| `moon_phase` | Phase at time of snapshot |
| `cloud_cover` | Cloud cover class |

**Snapshots occur:**
- Every 10–15 minutes
- When major conditions change
- When significant events occur

These snapshots form the **ride log timeline**.

---

## Rationale

This separation keeps the system honest:

- A rider planning a route needs the **best available prediction**
- A system improving over time needs the **historical observed truth**
- Comparisons between predicted and actual conditions are only meaningful if both are preserved independently

Merging them would contaminate the prediction model with outlier observations and destroy the ride history record.

---

## Consequences

**Advantages:**
- Accurate ride history
- Enables narrative ride summaries (ADR-014)
- Enables comparison between predicted and actual conditions
- Supports shareable ride experiences
- Provides data for future model calibration

**Tradeoffs:**
- Requires additional ride session storage
- Snapshot logic must handle cases where conditions don't change (avoid redundant writes)

---

## Design Principle

What was predicted before the ride and what was experienced during the ride are different kinds of truth. Never merge them.


---

## Source File: docs/03-adrs/adr-013-personalized-emergency-alerts.md

# ADR-013 — Personalized Emergency Alert Model

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-015 (Route Vulnerability Feature Model), ADR-016 (Ride Session Data Model), ADR-019 (Route Corridor & Proximity Rules)  
**No companion DS required** — this ADR is self-contained. Alert logic implementation is part of the ride session layer.

---

## Context

Severe weather alerts are often generic and location-wide. For a cyclist on a long route, the danger depends heavily on the specific route environment — not just whether a warning has been issued in the general area.

**Examples of why generic alerts fail cyclists:**
- A flash flood warning covers a county, but the rider may be on a ridge, not a flood-prone low area
- A crosswind advisory covers a region, but only the exposed bridge section of the route is actually dangerous
- A lightning warning covers the area, but the rider is 40 miles away from the exposed ridgeline and won't reach it for 3 hours

Generic alerts cause alert fatigue. Route-aware alerts are actionable.

---

## Decision

Emergency alerts will use a **personalized alert model** based on three inputs:

```
hazard signal
×
route vulnerability
×
rider ETA
→ personalized alert
```

### Hazard Signal

Live weather alerts including:
- Severe thunderstorm warning
- Tornado warning
- Lightning risk
- Hail warning
- Flash flood warning
- Extreme wind warning

### Route Vulnerability

Route features that amplify the specific hazard (see ADR-015):
- Flood-prone low areas
- Exposed bridges
- Open ridgelines
- Remote stretches with limited shelter
- Steep descents in weather-sensitive conditions

### Rider Timing

Alerts are only triggered when the rider will encounter the vulnerable segment soon:
- Current rider location
- ETA to vulnerable segment
- Hazard timing window overlap

---

## Example Alert

**Generic alert (what Lanterne does not do):**
> Flash flood warning in your area.

**Personalized alert (what Lanterne does):**
> Flash flood warning. Flood-prone low area ahead in 5 miles. You are expected to reach it in approximately 20 minutes.

---

## Consequences

**Advantages:**
- Alerts are relevant and actionable rather than generic noise
- Avoids alert fatigue from warnings that don't apply to the rider's current route segment
- Improves rider safety by surfacing real timing-specific risks

**Tradeoffs:**
- Requires route vulnerability detection (ADR-015)
- Requires periodic server-side hazard monitoring during active rides
- Alert timing accuracy depends on rider pace estimates

---

## Design Principle

An alert that doesn't tell the rider when and where the danger applies is just noise. Personalized alerts answer: *what hazard, which segment, how soon.*


---

## Source File: docs/03-adrs/adr-014-ride-narrative-event-model.md

# ADR-014 — Ride Narrative Event Model

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-012 (Predicted vs Experienced Conditions), ADR-016 (Ride Session Data Model)  
**No companion DS required** — event schema is defined in ADR-016 and the ride session data model.

---

## Context

Typical cycling apps record only telemetry: speed, heart rate, power, GPS track. This produces a data record of a ride but not a *story* of a ride.

Long-distance cycling — especially randonneuring — involves meaningful environmental and experiential moments that are worth preserving:
- A climb beginning at golden hour before darkness settles
- Moonlit riding on open rural roads at 2am
- A glare window on an exposed stretch at dawn
- A severe weather alert triggering 20 miles from a flood-prone crossing
- A control stop at a familiar diner

These moments make a brevet memorable and are exactly what riders talk about afterward. Lanterne should capture and surface them.

---

## Decision

Lanterne will store **ride events** during ride sessions.

Events represent meaningful moments along the ride timeline. They are used to generate ride narratives and summaries.

### Event Structure

```
ride_event
  ride_id
  timestamp
  distance
  lat
  lon
  event_type
  metadata
```

### Supported Event Types

| Event Type | Trigger |
|------------|---------|
| `ride_start` | Session begins |
| `ride_end` | Session ends |
| `climb_start` | Grade threshold exceeded |
| `climb_end` | Climb concludes |
| `descent_warning` | Technical descent ahead |
| `sunset` | Solar position crosses horizon |
| `sunrise` | Solar position crosses horizon |
| `moonlit_segment` | Moon phase + cloud cover produces meaningful illumination |
| `glare_window` | Sun elevation + bearing alignment triggers glare risk |
| `poi_stop` | Rider pauses at a known POI |
| `control_point` | Brevet control reached |
| `severe_weather_alert` | Personalized hazard alert triggered |

---

## Example Narrative Output

Events are assembled into a ride narrative that can be shared or reviewed:

> *You began riding under warm evening light. A long climb started at mile 28 before sunset. Darkness settled by mile 60, and the waxing moon illuminated the rural road ahead. A control stop at mile 82 before the final push to finish.*

---

## Consequences

**Advantages:**
- Enables automatic ride storytelling without manual input
- Produces compelling ride logs that reflect the actual experience
- Events are derived from stored ride data — no additional recording required
- Narratives are shareable and meaningful to the randonneuring community

**Tradeoffs:**
- Requires event detection logic running during the ride
- Narrative quality depends on event detection accuracy

---

## Design Principle

A brevet is not a data file. Capture the story of the ride, not just the movement of the bike.


---

## Source File: docs/03-adrs/adr-015-route-vulnerability-feature-model.md

# ADR-015 — Route Vulnerability Feature Model

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-013 (Personalized Emergency Alert Model), ADR-011 (Route Slice Model), ADR-019 (Route Corridor & Proximity Rules)  
**No companion DS required** — vulnerability features are stored at the slice level per the slice schema (DS-005, DS-007).

---

## Context

Lanterne needs to personalize route intelligence and emergency alerts based not only on weather or route geometry, but on what kinds of route environments make certain hazards worse.

**Examples:**
- Heavy rain is more dangerous in flood-prone low areas
- Severe crosswinds are more dangerous on exposed bridges
- Lightning is more dangerous on open ridgelines
- Hail is more dangerous on exposed descents far from shelter
- Remote stretches change the seriousness of all emergency conditions

Without route vulnerability features, the system can only issue generic warnings. Generic warnings cause alert fatigue and erode trust.

---

## Decision

Lanterne will define and store a set of **route vulnerability features** attached to analysis slices and rolled up to route-level summaries.

These features are **not top-level rider-facing scores** by default. They are internal route traits used to power:
- Personalized emergency alerts (ADR-013)
- Remoteness interpretation
- Shelter and bailout awareness
- Hazard amplification logic
- Future rerouting and decision support

---

## Initial Vulnerability Feature Set

### Water / Flooding

| Feature | Type |
|---------|------|
| `is_flood_prone` | boolean |
| `is_low_lying_area` | boolean |
| `has_underpass_flood_risk` | boolean |
| `has_low_water_crossing` | boolean |
| `near_watercourse` | boolean |

### Wind Exposure

| Feature | Type |
|---------|------|
| `is_exposed_bridge` | boolean |
| `is_exposed_ridgeline` | boolean |
| `is_open_exposed_terrain` | boolean |
| `is_high_crosswind_exposure` | boolean |

### Shelter / Support

| Feature | Type |
|---------|------|
| `has_limited_shelter_access` | boolean |
| `has_limited_bailout_access` | boolean |
| `is_remote_stretch` | boolean |

### Terrain / Control Risk

| Feature | Type |
|---------|------|
| `is_descent_sensitive` | boolean |
| `is_weather_sensitive_descent` | boolean |
| `is_visibility_sensitive_segment` | boolean |

These may be stored as booleans or low/medium/high classifications as the model matures.

---

## Spatial Model

Route vulnerability features are computed and stored at the **analysis slice level**.

They may be rolled up into:
- Route summaries
- Rider warnings
- Display layers
- Narrative events

Storing at slice level avoids over-smoothing — vulnerability can spike locally even when the rest of the route is benign.

---

## Alert Model Dependency

These features combine with live hazard signals and rider ETA to generate personalized alerts (ADR-013):

```
hazard signal
×
route vulnerability
×
ETA overlap
→ personalized alert
```

**Generic alert:**
> Heavy rain warning in your area.

**Lanterne personalized alert:**
> Flash flood warning. Flood-prone low area ahead in 4 miles. You are expected to reach it in about 18 minutes.

---

## Consequences

**Advantages:**
- Route-aware safety alerts that are actionable
- Smarter ride guidance with local context
- More useful remoteness interpretation
- Reusable feature layer across multiple systems

**Tradeoffs:**
- Requires feature extraction logic from OSM and route context
- Some features will begin as heuristics before becoming more refined

---

## Design Principle

Route vulnerability features should be specific, explainable, spatially local, and useful across multiple systems. They are not a dumping ground for every possible route fact.


---

## Source File: docs/03-adrs/adr-016-ride-session-data-model.md

# ADR-016 — Ride Session Data Model

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-012 (Predicted vs Experienced Conditions), ADR-013 (Personalized Emergency Alerts), ADR-014 (Ride Narrative Event Model), ADR-015 (Route Vulnerability Feature Model)  
**No companion DS required** — this ADR defines the core schema. A dedicated DS may be written when ride session implementation begins.

---

## Context

Lanterne analyzes routes before a ride and predicts conditions along them. During an actual ride the rider experiences real conditions that may differ from forecasts. Additionally, the system generates meaningful ride events — sunrise/sunset transitions, moonlit segments, climbs, descents, POI stops, severe weather alerts.

These moments should be preserved as part of ride history.

Traditional cycling apps focus on raw telemetry: speed, power, heart rate. Lanterne instead focuses on recording the **environmental and experiential context** of the ride.

A dedicated ride session model is required to separate:
- Route intelligence (stable, pre-computed)
- Predicted conditions (forecast-based)
- Actual ride experience (recorded during the ride)

---

## Decision

Lanterne introduces a **Ride Session layer** representing a specific traversal of a route.

Ride sessions are separate from routes and route analysis. A single route may have many ride sessions across different riders and different conditions.

---

## Core Entities

### `ride_sessions`

Represents one ride instance.

| Field | Description |
|-------|-------------|
| `id` | Unique session identifier |
| `user_id` | Rider |
| `route_id` | Route being ridden |
| `start_time` | Ride start timestamp |
| `end_time` | Ride end timestamp |
| `start_location` | Actual start lat/lon |
| `total_distance` | Recorded distance |
| `total_duration` | Total elapsed time |
| `ride_mode` | Active ride mode |

---

### `ride_condition_snapshots`

Captures environmental conditions experienced during the ride.

**Snapshots are recorded:**
- Approximately every 10–15 minutes
- When major conditions change
- When significant ride events occur

| Field | Description |
|-------|-------------|
| `ride_session_id` | Parent session |
| `timestamp` | Snapshot time |
| `lat` / `lon` | Location |
| `distance` | Distance from start |
| `wind_speed` | Actual wind speed |
| `wind_direction` | Wind direction |
| `temperature` | Actual temperature |
| `precipitation` | Precipitation state |
| `cloud_cover` | Cloud cover class |
| `uv_index` | UV index |
| `light_state` | Daylight / twilight / night |
| `glare_flag` | Active glare condition |
| `moon_phase` | Moon phase at snapshot time |

---

### `ride_events`

Represents meaningful moments detected during the ride. Forms the basis of the ride narrative (ADR-014).

| Field | Description |
|-------|-------------|
| `ride_session_id` | Parent session |
| `timestamp` | Event time |
| `lat` / `lon` | Event location |
| `distance` | Distance from start |
| `event_type` | See event type table below |
| `metadata` | Event-specific payload |

**Supported event types:**

| Event Type | Trigger |
|------------|---------|
| `ride_start` | Session begins |
| `ride_end` | Session ends |
| `climb_start` | Grade threshold exceeded |
| `climb_end` | Climb concludes |
| `descent_warning` | Technical descent ahead |
| `sunset` | Solar crossing |
| `sunrise` | Solar crossing |
| `moonlit_segment` | Meaningful lunar illumination |
| `glare_window` | Sun glare risk active |
| `poi_stop` | Rider pauses at POI |
| `control_point` | Brevet control reached |
| `severe_weather_alert` | Personalized hazard alert triggered |

---

### `alert_events`

Records emergency or advisory alerts triggered during a ride.

| Field | Description |
|-------|-------------|
| `ride_session_id` | Parent session |
| `timestamp` | Alert time |
| `lat` / `lon` | Rider location at alert |
| `alert_type` | Alert category |
| `severity` | Alert severity level |
| `message` | Rider-facing alert text |

**Example alert types:** `lightning_warning`, `flash_flood_warning`, `extreme_wind_alert`, `hail_warning`

---

## Narrative Generation

Ride narratives are generated from the combination of:
- Ride events
- Condition snapshots
- Route features
- POI interactions

**Example narrative:**
> *You began riding just before sunset. A steady climb began around mile 28 before darkness settled over the farmland. The waxing moon illuminated the next stretch of road before a stop at the diner in Centerville.*

Narratives are derived from stored ride data — they do not require additional recording during the ride.

---

## Consequences

**Advantages:**
- Enables rich ride storytelling and shareable summaries
- Captures the environmental experience of long rides, not just movement
- Supports comparison between predicted and actual conditions
- Enables future insights and analytics across rides

**Tradeoffs:**
- Requires additional ride session storage
- Event detection logic must run during rides without draining battery

---

## Design Principles

Ride sessions should record meaningful environmental context and key ride moments. They should **not** record excessive raw telemetry unless required for specific features.

The goal is to preserve the **experience** of the ride, not just the movement of the bike.


---

## Source File: docs/03-adrs/adr-017-local-osm-derived-data-strategy.md

# ADR-017 — Local OSM-Derived Data Strategy

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-018 (Server-Cached Slice Analysis Model), ADR-011 (Route Slice Model), ADR-020 (Atomic Analysis Unit)  
**Companion specs:** DS-010 (Slice Analysis Cache)

---

## Context

Live OpenStreetMap queries (Overpass / API) are too slow and unreliable for runtime route analysis. During development, route loading took 3–5 minutes and triggered IP throttling at scale.

Lanterne must analyze routes quickly and repeatedly without depending on live OSM queries during normal operation.

However, fully ingesting and maintaining a complete national or global OSM mirror at launch would add unnecessary infrastructure complexity.

The system needs a strategy that:
- Avoids repeated live OSM queries
- Allows incremental expansion of geographic coverage
- Supports reusable map intelligence
- Supports fast route analysis after first load

---

## Decision

Lanterne will maintain a **local OSM-derived feature store** combined with a **route-level derived cache**.

The system uses three layers:

```
regional feature tiles
       ↓
route corridor extraction
       ↓
route-level derived cache
```

---

## Layer 1 — Regional Feature Store

Geographic regions touched during route analysis are stored locally as reusable **feature tiles**.

Tiles contain extracted OSM features relevant to route intelligence:
- Road network geometry and classifications
- Bike infrastructure tags
- Surface types
- Bridges, tunnels, rail crossings
- POIs and services
- Water features and other hazard-relevant map features

**If a region is analyzed once, it is persisted and reused for future routes.** The tile-based system uses a fixed 0.05° grid, enabling route-independent cache sharing between different routes traversing the same tiles.

---

## Layer 2 — Route Corridor Extraction

When a route is analyzed, the system extracts a **corridor dataset** around the route from the regional feature store.

The corridor includes:
- Route geometry
- Nearby roads
- Nearby services
- Environmental context
- Hazard-relevant features

Corridor radius varies by feature type and remoteness level per ADR-019.

---

## Layer 3 — Route-Level Derived Cache

Once a route is enriched, Lanterne stores results as a derived cache:
- Canonical route geometry
- Analysis slices
- Slice-level indices
- Vulnerability features
- Route rollups
- Analysis version metadata

Subsequent loads of the same route use the cached analysis instead of repeating OSM extraction.

---

## First-Time Route Analysis

New routes touching uncached geographic regions require initial enrichment:
1. OSM features are fetched from Overpass
2. The regional feature store is populated
3. Route analysis is computed
4. Derived route data is cached

First-time analysis may take longer than subsequent loads. This is acceptable — it happens once per region.

---

## Refresh Strategy

OSM-derived data should be refreshed when:
- Routes are explicitly reanalyzed
- Underlying analysis logic changes
- Cached regional data exceeds a staleness threshold

Refreshes occur only for regions already stored locally.

---

## Consequences

**Advantages:**
- Eliminates dependency on live OSM queries during normal operation
- Fast route loading after first analysis
- Reusable geographic intelligence across routes sharing the same corridor
- Supports future multi-country expansion incrementally

**Tradeoffs:**
- Requires maintaining a local feature store
- Initial route analysis is slower for uncached regions
- Tile cache must be managed for size and staleness

---

## Design Principle

Lanterne should own its derived map intelligence rather than relying on live external queries during normal operation. Fetch once, cache, reuse.


---

## Source File: docs/03-adrs/adr-018-server-cached-slice-analysis-model.md

# ADR-018 — Server-Cached Slice Analysis Model

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-017 (Local OSM-Derived Data Strategy), ADR-011 (Route Slice Model), ADR-020 (Atomic Analysis Unit)  
**Companion specs:** DS-010 (Slice Analysis Cache), DS-005 (Canonical Route Schema)

---

## Context

Route analysis involves expensive geospatial calculations:
- Traffic exposure estimation
- Bike infrastructure detection
- Remoteness analysis
- Hazard detection
- Terrain analysis

These computations operate on analysis slices derived from route geometry.

Recomputing these on the client for every route load would cause:
- High device battery usage
- Inconsistent results across devices
- Slower route loading
- Duplicated logic across platforms

---

## Decision

Lanterne will **compute and cache slice-level analysis results on the server**.

Clients consume cached slice data and perform only lightweight presentation logic.

---

## Server Responsibilities

The server performs all heavy route analysis:
- Generating analysis slices
- Computing slice-level indices
- Computing vulnerability features
- Computing route rollups
- Storing analysis version metadata

Results are persisted in the route-level derived cache.

---

## Client Responsibilities

Clients are responsible for **presentation only**:
- Rendering route heatmaps
- Grouping slices into display segments
- Mapping ride-time conditions onto slices
- Smoothing visualization
- Generating tooltips and UI explanations

Clients do **not** recompute the core analysis model.

---

## Contextual Overlays

Ride-time conditions (wind, temperature, precipitation, solar position, glare windows) may be computed dynamically using:
- Cached slice geometry
- Forecast data
- Rider start time
- Estimated arrival time

These overlays **do not require recomputing** the underlying route analysis — they use the stable cached slice geometry as the substrate.

---

## Reanalysis

Slice-level indices may be recomputed when:
- The scoring model changes
- OSM-derived features are updated
- Route geometry changes
- Analysis version increments

Previous analysis results may be retained for comparison.

---

## Consequences

**Advantages:**
- Consistent analysis results across all devices
- Fast route loading after first computation
- Reduced client battery consumption
- Easier debugging and version control — truth lives in one place

**Tradeoffs:**
- Increased server storage requirements
- Need for analysis version management
- Server becomes a dependency for analysis quality

---

## Design Principle

**Server computes truth. Client composes experience.**

The server owns the canonical route intelligence model. Clients focus on visualization and interaction.


---

## Source File: docs/03-adrs/adr-019-route-corridor-and-proximity-rules.md

ADR-019 — Route Corridor & Proximity Rules

Status: Accepted
Date: 2026-03-17

Companion spec: DS-008 (Route Corridor Model)

⸻

Context

Lanterne must interpret not only the road itself, but also the surrounding environment.

However, "nearby" does not mean one universal radius.

Examples:
	•	a bridge hazard must be attached tightly to the route
	•	a toilet visible from the road is immediately useful
	•	a motel 8 miles away may matter in remote country but not in a city
	•	a bailout turn matters if it heads toward civilization, not just if it exists

The system therefore needs a proximity model that distinguishes:
	•	route-adjacent truths
	•	visible roadside points of interest
	•	reachable off-route support
	•	broader area context

Decision

Lanterne will use a three-level corridor and proximity model.

tight route adjacency
↓
route-side POI adjacency
↓
dynamic enrichment horizon

⸻

1. Tight route adjacency

Used for features that physically affect the ridden road segment itself.

Examples:
	•	road class
	•	bike support
	•	surface changes
	•	bridges
	•	tunnels
	•	rail crossings
	•	underpasses
	•	flood-prone dips on the route
	•	exposed bridges
	•	direct hazard geometry

Rule

These features use tight geometry-based matching with a small route buffer.

Typical tolerance:
	•	approximately 25–75 meters
	•	feature-specific tuning allowed later

Purpose

To answer:

does this feature directly affect the rider on this stretch of road?

⸻

2. Route-side POI adjacency

Used for POIs that are not literally on the route geometry but are close enough to be immediately visible or obviously useful from the road.

Examples:
	•	toilets
	•	water
	•	convenience stores
	•	cafes
	•	small shelters
	•	roadside services

Rule

A POI is considered route-side adjacent if it lies within roughly:
	•	100–200 yards of the route

Human rule of thumb:

"I can see the porto potty over there."

Purpose

To answer:

would a rider experience this as right there, without meaningfully leaving the route?

⸻

3. Dynamic enrichment horizon

Used for reachable but non-adjacent support and context.

Examples:
	•	food
	•	water
	•	toilets
	•	lodging
	•	urgent care / hospital
	•	bike shops
	•	transit stops
	•	police / fire
	•	controls / brevet-relevant services
	•	bailout options
	•	service clusters

Rule

Search horizons expand or contract per analysis slice / segment depending on context.

Searches are:
	•	feature-specific
	•	context-aware
	•	capped at a maximum distance
	•	ideally based on network-access distance

Straight-line distance may also be stored for explanatory purposes.

Purpose

To answer:

what useful support exists that the rider could realistically reach from here?

⸻

4. Dynamic search behavior

Search radius should not be fixed for the entire route.

It may grow or shrink per slice depending on:
	•	urban vs rural context
	•	current remoteness
	•	service density
	•	feature type

Examples:
	•	city slices may need only a small service search horizon
	•	remote slices may require searches extending several miles
	•	urgent care may justify a larger horizon than cafes

This allows the system to adapt to routes that pass through both dense and remote areas.

⸻

5. Cached subset vs full POI universe

Lanterne distinguishes between:

A. Cached corridor POIs

A curated subset stored locally for:
	•	route analysis
	•	remoteness
	•	alerts
	•	narratives
	•	fast route loading

B. Full POI universe

The complete live set returned when the rider explicitly opens POI browsing / POI search in the app.

This may still query public OSM or other live sources.

Design principle

Lanterne caches the POIs needed for route intelligence.
It does not assume its cached subset is the entire world.

⸻

6. POI categories

Current rider-facing categories include:
	•	water
	•	food
	•	bio
	•	rest
	•	tourism
	•	nature
	•	health
	•	help

Internally, richer subtypes may exist under these categories.

First-class cached/enriched classes for v1:
	•	toilets / bio
	•	water
	•	cafes / food
	•	convenience stores / gas stations
	•	lodging / rest
	•	urgent care / hospital / health
	•	bike shops
	•	transit stops
	•	police / fire / help
	•	controls / brevet-relevant services

Grocery / supermarket rule

Grocery is not a primary food class but may be used as a fallback food source if no stronger food options are found within the current search horizon.

⸻

7. Confidence tiers for POIs

Some useful amenities are explicit in map data; others are inferred.

Lanterne will support confidence-based POI interpretation.

Example: water

High confidence:
	•	explicit drinking water
	•	public tap
	•	fountain

Medium confidence:
	•	gas station
	•	convenience store
	•	restaurant
	•	cafe

Low confidence:
	•	cemetery
	•	fire station
	•	police station
	•	park
	•	church
	•	school

Example: toilets

High confidence:
	•	public toilet
	•	explicit restroom

Medium confidence:
	•	gas station
	•	restaurant
	•	convenience store

Low confidence:
	•	park
	•	campground
	•	church
	•	community facility

Future versions may support:
	•	community-confirmed POIs
	•	confidence upgrades based on rider reports

⸻

8. Town / civilization model

A town is not defined purely by map labels or formal boundaries.

Decision

For Lanterne purposes, civilization = service cluster

This means a slice is closer to civilization when it has access to a cluster of useful amenities rather than a single isolated POI.

Examples of amenities contributing to a service cluster:
	•	food
	•	convenience / gas
	•	water
	•	lodging
	•	health / urgent care
	•	help services
	•	bike shop
	•	transit

This concept is used in remoteness and bailout logic.

⸻

9. Bailout definition

Decision

A bailout option is:

any turn which heads toward civilization

This means bailout logic should prefer roads that plausibly improve the rider's odds of reaching:
	•	settlement
	•	services
	•	pickup access
	•	support

Routing preference

When choosing among bailout options, the system should prefer:
	•	shortest path toward civilization
	•	excluding interstates / motorways

Later versions may also exclude other non-bike-safe limited-access roads.

⸻

10. Remoteness structure

Remoteness should not be defined only by nearest POI distance.

Decision

Remoteness should prioritize:
	1.	bailout scarcity
	2.	civilization scarcity
	3.	service scarcity

This reflects rider concern more accurately than a simple nearest-amenity model.

Rider fear order

Locked priority from user research / product direction:
	1.	No escape roads nearby
	2.	No people / towns nearby
	3.	No services nearby

⸻

11. Enrichment timing

Dynamic amenity enrichment may occur after initial core scoring analysis.

Core scoring should focus first on:
	•	route safety
	•	infrastructure
	•	direct route traits

Then enrichment can add:
	•	support context
	•	service access
	•	remoteness detail
	•	narrative POIs
	•	alert personalization

This helps keep initial route analysis manageable.

⸻

12. Consequences

Advantages:
	•	proximity logic matches rider reality
	•	supports both dense and remote routes
	•	enables intelligent remoteness modeling
	•	enables route-aware emergency alerts
	•	supports narrative and POI-rich ride logs

Tradeoffs:
	•	more complex than one fixed radius
	•	requires confidence-aware POI interpretation
	•	requires careful performance tuning for dynamic searches

⸻

Design principle

Lanterne should distinguish between:
	•	what is on the road
	•	what is right beside the road
	•	what is reachable from the road
	•	what indicates civilization or escape

These are different questions and should not be forced into one radius.


---

## Source File: docs/03-adrs/adr-020-atomic-analysis-unit.md

- # ADR-020 — Atomic Route Analysis Unit

  **Status:** Accepted  
  **Date:** 2026  
  **Owner:** Lanterne Architecture

  ---

  ## Context

  Lanterne analyzes cycling routes to generate safety, environmental, and logistical insights for long-distance riding.

  The system must compute route intelligence with enough granularity to reflect real changes along the route without exploding complexity or storage. Large road segments smooth away meaningful variation. Fixed-distance chunks are too blunt and often ignore the things riders actually experience.

  The system therefore needs a stable internal analysis unit that supports:
  
  - traffic variation
  - bike facility changes
  - shoulder changes
  - surface changes
  - remoteness shifts
  - major intersection and ramp complexity
  - environmental timing transitions
  - later rolling-window and route-character analysis
  
  ---
  
  ## Decision
  
  Route analysis will operate on **bounded dynamic slices**.
  
  Slices are internal analysis units created from:
  
  - meaningful route/context changes
  - maximum length limits
  - environmental event boundaries
  
  Internal slices may be aggregated for display. The UI does not need to expose raw slice granularity.
  
  ---
  
  ## Slice boundary rules
  
  A slice boundary occurs when a meaningful change happens in:
  
  - road classification
  - traffic environment
  - bike facility presence
  - shoulder condition
  - surface type
  - settlement / service proximity
  - bailout access proximity
  - major intersections or ramps
  
  Slices may also split when time-aware conditions change, such as:
  
  - daylight → twilight → night
  - sunrise/sunset glare conditions
  - significant remoteness dips
  - future ride-timeline interruptions governed by the ride timeline model

  Even if no meaningful change occurs, slices are capped at approximately **750–1000 meters**.
  
  ---

  ## Expected slice density

  Typical routes should produce approximately:

  **200–400 slices per 200 km route**

  This provides:

  - sufficient precision for route intelligence
  - manageable computational load
  - stable support for future rolling-window analysis

  ---
  
  ## Normalization rule

  Raw source data must not be used directly in scoring logic.

  OpenStreetMap and related source inputs are translated into **normalized Lanterne variables** before index computation.
  
  The architecture therefore follows this model:
  
  ```text
  Raw source inputs
          ↓
  Normalized Lanterne variables
          ↓
  Index computations


---

## Source File: docs/03-adrs/adr-021-osm-variable-registry.md

ADR-021 — OSM Variable Registry

Status: Accepted
Date: 2026

Related ADRs: ADR-020 (Atomic Analysis Unit), ADR-022 (Phase 1 Enum Registry)
Living spec: DS-004 (OSM Variable Registry Spec)

⸻

Context

ADR-020 established that OSM tags must be translated into normalized Lanterne variables
rather than used directly in scoring logic.

This ADR governs the registry that defines those variables.

⸻

Decision

Lanterne will maintain a canonical OSM Variable Registry.

The registry defines the normalized variables stored for each route slice.

Governance rules:
	•	New variables may only be added when they answer a clear rider question
	•	Variables must be traceable to their OSM source tags
	•	Each variable must carry a confidence class
	•	The registry is versioned; analysis outputs reference the registry version used

The living variable specification — including the current variable list, allowed values,
confidence rules, and OSM tag mappings — is maintained in DS-004.

ADR-021 governs the existence and rules of the registry.
DS-004 contains the actual registry contents.

⸻

Variable categories

The registry is organized into the following categories:

A. Road Environment Variables
Support Traffic and Safety analysis.
Examples: road_class, speed_limit_value, lane_count_value, intersection_density_value

B. Bike Support Variables
Describe cycling infrastructure.
Examples: bike_facility_type, shoulder_class, shoulder_width_value

C. Surface Variables
Capture surface type and rideability.
Examples: surface_type, surface_quality_class, offroad_context_class

D. Remoteness and Support Variables
Proximity to services, bailout options, and civilization markers.
Examples: settlement_proximity_m, bailout_access_proximity_m, support_context_class

E. Geometry and Terrain Variables
Support fatigue and descent analysis.
Examples: elevation_m, grade_percent, descent_flag, curvature_class

F. Environmental Timing Variables
Capture ride conditions dependent on time of day and forecast.
Examples: light_state, wind_speed_value, temperature_value, precip_probability

G. Traceability Fields
Metadata enabling debugging and model transparency.
Examples: raw_osm_tags_json, normalized_variable_evidence_json, community_override_json

⸻

Observed speed hierarchy

Traffic speed variables follow this priority:
	1.	observed car speed (radar data with ≥3 samples)
	2.	posted speed limit
	3.	inferred speed environment

Observed speed overrides other speed values when confidence is sufficient.

⸻

Environmental variables

Environmental timing variables depend on ride start time and forecast data.

They are treated as contextual variables rather than inputs to the core safety model.

This boundary is established in ADR-023.

⸻

Consequences

Advantages:
	•	consistent OSM interpretation across the entire system
	•	transparent scoring logic
	•	robust handling of incomplete OSM data
	•	ability to incorporate future data sources

Tradeoffs:
	•	normalization logic must be maintained as OSM evolves
	•	registry versioning adds migration overhead

⸻

Design principle

OSM data must be interpreted consistently across the entire system.
The registry is the contract between raw map data and route intelligence.


---

## Source File: docs/03-adrs/adr-022-phase-1-enum-registry.md

ADR-022 — Phase 1 Enum Registry

Status: Accepted
Date: 2026

Related ADRs: ADR-020 (Atomic Analysis Unit), ADR-021 (OSM Variable Registry)
Living spec: DS-004 (OSM Variable Registry Spec)

Note: This ADR is self-contained. No separate companion DS is required.
      Enum values that evolve over time are maintained in DS-004.

⸻

Context

ADR-020 and ADR-021 established that OSM tags are translated into normalized Lanterne
variables before use in scoring logic.

This ADR defines the string-valued enum system used for those normalized variables
in Phase 1 of the registry.

⸻

Decision

Lanterne will use string-valued enums for normalized phase-1 variables.

Enums are intended to be:
	•	human-readable
	•	debuggable
	•	stable across ingestion and scoring pipelines
	•	easy to inspect in SQL, admin tooling, and evidence trails

Where exact values are known and trustworthy, they may be stored alongside enums.
Where data is weak or incomplete, the enum acts as the normalized fallback.

Traffic volume is explicitly treated as an externally sourced operational variable
rather than an OSM-derived enum.

Reason

String enums improve transparency and debugging, which is important for:
	•	model trust
	•	community review
	•	future tuning
	•	fast iteration during schema evolution

This architecture avoids fake precision while preserving the ability to use more
exact values where available.

⸻

Phase 1 Enum Definitions

1. road_class
Allowed values:
	•	path
	•	local_road
	•	collector
	•	arterial
	•	highway_adjacent
	•	service_road
	•	track

2. speed_environment_class
Allowed values:
	•	very_low
	•	low
	•	moderate
	•	high
	•	very_high

Associated exact values when available:
	•	speed_limit_value
	•	car_speed_value

3. lane_count_class
Allowed values:
	•	single_lane_each_direction
	•	multilane
	•	divided_multilane

Associated exact value when available:
	•	lane_count_value

4. shoulder_class
Allowed values:
	•	none
	•	narrow
	•	usable
	•	wide

Associated exact value when available:
	•	shoulder_width_value

5. bike_facility_type
Allowed values:
	•	none
	•	marked_bike_route
	•	painted_bike_lane
	•	protected_bike_lane
	•	bike_path
	•	multiuse_path

6. surface_type
Allowed values:
	•	paved
	•	gravel
	•	dirt
	•	trail
	•	unknown

7. offroad_context_class
Allowed values:
	•	none
	•	paved_path
	•	gravel_road
	•	forest_road
	•	trail
	•	singletrack

8. crossing_complexity_class
Allowed values:
	•	none
	•	minor
	•	caution

9. curvature_class
Allowed values:
	•	straight
	•	gentle
	•	curvy
	•	technical

10. light_state
Allowed values:
	•	daylight
	•	civil_twilight
	•	night

11. uv_class
Allowed values:
	•	low
	•	moderate
	•	high
	•	very_high

12. cloud_cover_class
Allowed values:
	•	clear
	•	partly_cloudy
	•	mostly_cloudy
	•	overcast

13. precip_intensity_class
Allowed values:
	•	none
	•	light
	•	moderate
	•	heavy

Associated parallel value:
	•	precip_probability

14. confidence_class
Allowed values:
	•	observed
	•	explicit
	•	inferred
	•	weak

15. effective_value_source
Allowed values:
	•	osm
	•	radar
	•	inferred
	•	community_unverified
	•	community_verified
	•	admin_verified

16. Operational traffic variables

Traffic volume is not primarily normalized from OSM.

It is treated as an externally sourced operational variable with fields such as:
	•	traffic_volume_value
	•	traffic_volume_unit
	•	traffic_volume_source
	•	traffic_volume_confidence

Current sources may include:
	•	HPMS
	•	state DOT APIs
	•	radar observations (future)

Traffic volume feeds Traffic Index but is distinct from the OSM normalization layer.

⸻

Design principle

String enums keep scoring logic transparent and debuggable.
Exact values supplement enums where precision is justified.
Fake precision is worse than an honest enum.


---

## Source File: docs/03-adrs/adr-023-observed-conditions-layers.md

## ADR-023 — Predicted vs Observed Condition Layers

Status: Accepted

Decision:
Lanterne will store predicted and observed ride conditions as separate data layers.

Predicted conditions are generated from:
- forecast weather
- astronomical calculations
- modeled traffic conditions
- route timing estimates

Observed conditions are generated from:
- activity uploads
- rider device telemetry
- future radar integrations
- future community-confirmed observations

Predicted layers are used for route planning.
Observed layers are used for historical truth, model validation, and future model improvement.

Observed values do not overwrite predicted values directly.
Instead, both are preserved and linked by:
- route
- slice
- ride/activity
- timestamp

Reason:
This keeps the system honest.

A rider planning a route needs the best available prediction.
A system improving over time needs the historical observed truth.

Separating the two prevents:
- forecast values from contaminating historical observations
- observed outliers from overwriting planning defaults
- confusion about what was known before the ride vs what actually happened during the ride


---

## Source File: docs/03-adrs/adr-024-ride-timeline-plans.md

ADR-024 — Ride Timeline Plans

Status: Accepted
Date: 2026

Companion spec: DS-003 (Ride Timeline Model Spec)
Related ADRs: ADR-020 (Atomic Analysis Unit), ADR-023 (Predicted vs Observed Condition Layers),
              ADR-025 (Fatigue Index as Extensible Model Family)

⸻

Context

Lanterne computes time-aware overlays per slice — weather, light conditions, traffic
patterns, glare, and similar factors. These all depend on knowing when the rider
arrives at each slice.

A naive model assumes a single uninterrupted pacing function from start to finish.

That model is insufficient for long-distance riding, where riders regularly plan:
	•	long rest stops
	•	sleep breaks (especially on overnight brevets and multi-day events)
	•	control stops of variable duration
	•	fueling stops

Without the ability to model interruptions, the arrival-time estimates for the second
half of a long overnight ride will be wrong, causing all time-aware overlays
(light, traffic, temperature, glare) to be materially incorrect.

⸻

Decision

Time-aware ride overlays must support future timeline interruptions such as sleep
and long rest stops.

Predicted arrival times must not be modeled as a single uninterrupted pacing function.

The architecture must allow future timeline plans that modify slice arrival times
through inserted stop and sleep events.

This constraint applies to:
	•	the ride timeline model
	•	slice arrival time calculations
	•	all time-aware environmental overlays
	•	fatigue modeling that incorporates rest

⸻

Implications

The ride timeline model (DS-003) must be structured to accept:
	•	a base pace assumption
	•	an ordered list of stop events with location and duration
	•	sleep events with location, start, and end time

Arrival time at each slice is computed by applying the pacing function with stop
and sleep events inserted at the appropriate positions along the route.

This does not require full implementation immediately.
It requires that the timeline data model not assume interruptions are impossible.

⸻

Consequences

Advantages:
	•	arrival time estimates remain accurate for long-distance and overnight rides
	•	all time-aware overlays remain meaningful throughout the full route
	•	fatigue modeling can incorporate rest without architectural rework

Tradeoffs:
	•	timeline model requires more structure than a simple pace × distance formula
	•	UI must eventually support stop / sleep event editing

⸻

Design principle

Conditions must be tied to when the rider actually arrives at each part of the route,
accounting for planned stops and sleep.
A pacing model that cannot be interrupted is not a model for long-distance riding.


---

## Source File: docs/03-adrs/adr-025-fatigue-index-as-extensible.md

ADR-025 — Fatigue Index as Extensible Model Family

Status: Accepted
Date: 2026

Related ADRs: ADR-020 (Atomic Analysis Unit), ADR-024 (Ride Timeline Plans)
Pending spec: DS file not yet required; fatigue model details will be specced
              when implementation moves beyond the basic route-burden model

Note: This ADR is self-contained at the decision level.

⸻

Context

Lanterne computes a Fatigue Index as one of its core route intelligence indices.

In its initial form, the Fatigue Index is a route-burden model based primarily on
accumulated elevation gain, distance, and grade.

Long-distance cycling fatigue is multidimensional. A simple route-burden model
is a reasonable starting point but will become inadequate as the product matures
toward advanced planning use cases.

The system must not be architected in a way that would require a schema rewrite
or product rename to evolve the fatigue model.

⸻

Decision

Fatigue Index is a stable user-facing concept but must support richer internal
models over time.

The system must allow fatigue to evolve from a simple route-burden model into more
complex formulations incorporating:
	•	terrain
	•	duration
	•	environmental stress
	•	stop / sleep planning
	•	fueling and hydration assumptions
	•	potentially athlete-specific data

Fatigue outputs must preserve:
	•	top-line value (rider-facing summary score)
	•	component breakdowns (for explainability and debugging)
	•	model versioning (so historical outputs remain interpretable)

⸻

Implications

The Fatigue Index storage schema must support:
	•	a versioned model identifier
	•	a top-line score
	•	a component breakdown (jsonb or structured columns)

New fatigue model versions should produce new versioned outputs rather than
overwriting existing ones.

The rider-facing name "Fatigue Index" is stable regardless of internal model evolution.

⸻

Consequences

Advantages:
	•	fatigue modeling can grow in sophistication without product disruption
	•	component breakdowns enable transparency and coaching use cases
	•	versioning preserves historical comparability

Tradeoffs:
	•	model versioning adds schema complexity
	•	richer inputs (fueling, athlete data) require additional data collection

⸻

Design principle

The rider-facing name is stable.
The internal model is extensible.
Output versioning is required.


---

## Source File: docs/03-adrs/adr-026-canonical-route-identity.md

ADR-026 — Canonical Route Identity

Status: Accepted
Date: 2026-03-18

Companion specs: DS-005 (Canonical Route Schema), DS-006 (Route Canonicalization),
                 DS-009 (Route Corridor Fingerprint)
Related ADRs: ADR-031 (Multi-Day Events as Ordered References onto Canonical Geometry)

⸻

Context

Routes can enter Lanterne through multiple ingestion paths:
	•	RWGPS import
	•	GPX upload
	•	RUSA permanent import
	•	manual route drawing
	•	future external sources

Different sources frequently represent the same real-world route with small variations:
	•	start/end moved slightly
	•	GPS sampling differences
	•	file simplification differences
	•	event controls adjusted
	•	minor detours around construction

If each import becomes a new route record, the system accumulates large numbers of
near-duplicate routes.

This creates several problems:
	•	analysis duplication
	•	slice cache fragmentation
	•	inconsistent user experience
	•	cluttered route discovery

Lanterne is designed as a route intelligence system, not a route file repository.

Therefore the system must treat routes primarily as experienced road corridors,
not as file artifacts.

⸻

Decision

Lanterne will maintain a canonical route identity for routes that represent the same
underlying road experience.

Route identity is determined primarily by geometry similarity, not by metadata.

If two routes represent the same corridor of roads with only minor variation, they
should resolve to the same canonical route.

Sources and variants are attached as provenance or versions, not as separate
canonical routes.

⸻

Core mental model

A route is the line.
Sources tell where the line came from.
Analyses tell what Lanterne thinks about the line.

⸻

Implications

A canonical route may have multiple associated inputs:

canonical_route
 ├ RWGPS import
 ├ RUSA permanent
 ├ GPX upload
 ├ user edits
 └ event variant

Analysis and slice calculations attach to the canonical route, allowing reuse
of computation across multiple sources.

Metadata such as:
	•	start location
	•	control placement
	•	naming differences
	•	source platform

do not determine route identity.

⸻

Examples

Example 1 — Start location moved

A 200 km permanent route begins at a McDonald's.
An event organizer shifts the start to a Burger King one block away.
The road corridor remains identical.
Result: same canonical route.

Example 2 — GPS sampling differences

Two GPX files represent the same route with different point densities.
Result: same canonical route.

Example 3 — Meaningful reroute

A route detours around a town and uses a different highway alignment for 15 miles.
Result: new canonical route.

⸻

Non-Goals

This ADR does not define:
	•	exact similarity algorithms
	•	geometric tolerance thresholds
	•	canonicalization pipeline details

These are specified in DS-006 (Route Canonicalization Spec).

⸻

Consequences

Advantages:
	•	prevents duplicate route explosion
	•	enables slice analysis reuse
	•	stabilizes route intelligence
	•	keeps discovery clean

Tradeoffs:
	•	canonicalization requires geometric comparison
	•	some edge cases may require manual review

⸻

Design principle

Two riders following the same road experience should land on the same canonical
route even if their files differ.
The canonical route is the experienced corridor, not the file artifact.


---

## Source File: docs/03-adrs/adr-027-lantern-screen-model.md

ADR-027 — Lantern Screen Model

Status: Accepted
Date: 2026

Companion specs: DS-011 (Ride-Time Situational Awareness Interface),
                 DS-012 (Ride Computer Tile System)
Related ADRs: ADR-029 (Ride-Time Situational Awareness Mode),
              ADR-030 (Ride Mode Display, Power, and Sensor Architecture)

⸻

Context

The Lanterne lantern interaction originally exposed a fixed set of route intelligence
layers (e.g., hazards, cues, stops, weather).

As the interaction evolved, the lantern became a mode selector driven by gesture,
displaying large center text confirmation rather than chip buttons.

Because the lantern selector no longer occupies fixed UI slots, the constraint of equal
layer counts across ride modes is no longer necessary.

Different ride contexts (e.g., randonneuring, commuting, bikepacking) require different
sets of glanceable information.

Additionally, riders are accustomed to configuring screens on cycling head units
(Garmin, Karoo), though those systems typically expose raw data fields rather than
meaningful riding views.

⸻

Decision

Lantern selector items will represent Lantern Screens, not fixed map layers.

A Lantern Screen represents a glanceable riding view optimized for a single type
of information.

Examples include:
	•	hazards
	•	cues
	•	controls
	•	stops
	•	weather
	•	remoteness
	•	surface quality
	•	traffic metrics
	•	filtered POI types (e.g., food, toilets)

Lantern Screens are organized into a Lantern Stack.

The Lantern Stack is:
	•	defined by ride mode defaults
	•	user-overridable per mode
	•	variable length
	•	ordered by importance for quick glance interaction

The lantern gesture interaction itself remains mode-agnostic:
	•	pull upward to scroll through screens
	•	center text confirms selection
	•	release activates the selected screen

⸻

Consequences

Advantages:
	•	ride contexts can surface only relevant information
	•	the lantern interaction remains consistent while content adapts
	•	the system can support future screen types without redesigning the lantern control

Tradeoffs:
	•	lantern thresholds must be computed dynamically from stack length
	•	UI must support configuration of Lantern Screens per ride mode

⸻

Future evolution

The Lantern Screen model enables future customization such as:
	•	user-configured screen ordering
	•	filtered POI screens (e.g., food vs toilets)
	•	multi-metric riding screens
	•	traffic and environmental intelligence views

These capabilities are not required for initial implementation.

⸻

Design principle

The lantern is the anchor and selector.
Screens are the content.
The interaction model is stable; the content adapts to context.


---

## Source File: docs/03-adrs/adr-028-field-note-confirmation-model.md

ADR-028 — Field Note Confirmation Model

Status: Accepted
Date: 2026

Pending spec: DS-016 (Field Note Schema and Confirmation Model) — not yet written
Related ADRs: ADR-026 (Canonical Route Identity)

Note: This ADR is self-contained at the decision level.
      DS-016 will define the schema and interaction model when field notes are implemented.

⸻

Context

Lanterne allows riders to submit Field Notes, which are geolocated observations tied
to a route or location.

Examples include:
	•	hazards
	•	dogs
	•	construction
	•	water sources
	•	food stops
	•	bathrooms
	•	unofficial amenities

Because these notes may appear in remote areas with very low rider traffic, the system
must distinguish between:
	•	an initial report
	•	independent rider confirmation

This distinction is important for trust and for long-term route intelligence.

⸻

Decision

Field Notes will use a report + confirmation model.

A Field Note consists of:
	•	an initial report
	•	optional independent confirmations from other riders

Confirmations are displayed as +1 increments.

Example:

dogs here
reported
+3

Meaning:
	•	1 rider reported the observation
	•	3 additional riders confirmed it

The original report does not automatically count as a +1 confirmation.

Confirmations must be submitted by other riders.

⸻

Reactions

Riders can interact with a Field Note using lightweight reactions.

Initial supported reactions:
	•	+1 → confirm observation
	•	cleared / gone → invalidate observation

These reactions update the note's confidence signals.

⸻

Rationale

This model preserves a clear distinction between claims and independent verification.

Automatic confirmation (e.g., the original report counting as +1) was explicitly
rejected because it creates misleading confidence signals.

The confirmation model favors accuracy and rider trust over popularity scoring.

⸻

Consequences

Advantages:
	•	preserves semantic integrity of confirmations
	•	avoids inflated confidence scores
	•	supports future hazard reliability scoring
	•	supports promotion of rider-reported POIs through independent verification

Tradeoffs:
	•	new notes will initially display with zero confirmations
	•	trust signals accumulate more slowly in low-traffic regions

⸻

Non-Goals

This decision does not define:
	•	ranking algorithms for Field Notes
	•	moderation systems
	•	promotion rules for POI candidates

Those will be defined in DS-016.

⸻

Design principle

A claim is not a confirmation.
Independent verification is the only honest basis for trust signals.


---

## Source File: docs/03-adrs/adr-029-ride-time-situational-awareness-mode.md

ADR-029 — Ride-Time Situational Awareness Mode

Status: Proposed
Date: 2026-03-22

Companion specs: DS-011 (Ride-Time Situational Awareness Interface),
                 DS-012 (Ride Computer Tile System)
Related ADRs: ADR-027 (Lantern Screen Model),
              ADR-030 (Ride Mode Display, Power, and Sensor Architecture)

Note: Status remains Proposed pending feature deployment.
      DS-011 and DS-012 are forward-looking specs written against this ADR's direction.

⸻

Context

Lanterne was originally conceived as a route intelligence system for long-distance cyclists.

The core architecture focuses on:
	•	route geometry normalization
	•	slice-level route analysis
	•	index calculation (traffic, remoteness, fatigue, etc.)
	•	environmental modeling (weather, sun, moon)

These systems allow riders to understand a route before a ride begins.

However, many of the environmental systems Lanterne computes are time-dependent and
become most meaningful during the ride itself:
	•	wind direction vs rider bearing
	•	temperature exposure
	•	precipitation
	•	sun position and glare
	•	moon phase and night visibility
	•	real-time navigation context

Traditional cycling computers display instrument data — speed, distance, heart rate,
power, navigation — but do not meaningfully communicate the environmental context
of the ride.

This creates an opportunity for Lanterne to function as a ride-time environmental
awareness system, complementing or replacing traditional bike computer displays.

⸻

Decision

Lanterne will support a Ride-Time Situational Awareness Mode.

This mode presents a minimal, glanceable interface that surfaces environmental and
navigational context relevant to the rider's current position and direction of travel.

The goal is not to replicate a cycling computer dashboard, but to communicate:
	•	what the environment is doing to the rider
	•	what is immediately ahead
	•	what conditions will affect the next stretch of the route

⸻

Interface principles

Large value-first presentation

Signals should be presented as large, legible values or icons.

Examples:
	•	speed
	•	temperature
	•	wind direction relative to route bearing
	•	distance to next turn
	•	distance remaining to destination

Adaptive layout grid

The display layout adjusts depending on the number of signals shown.

Examples:
	•	1 signal → centered on screen
	•	2 signals → vertical thirds
	•	4 signals → screen quadrants

The layout must avoid covering the map tracking position indicator.

Map-aware contrast

Text and icons must adapt to map theme:
	•	light map → charcoal
	•	dark map → white
	•	satellite → contrast sampled from background

Minimalist iconography

Where appropriate, signals may use minimalist icons instead of text.

Examples:
	•	wind direction arrow
	•	sun glare indicator
	•	moon visibility indicator
	•	precipitation icon

Icons must remain readable at large scale and low glance time.

⸻

Interaction model

Ride-time screens will be navigated through the Lantern control.

Key behaviors:
	•	screens scroll horizontally via lantern movement
	•	release of the lantern thumb selects the current screen
	•	tapping to confirm selection is not required

⸻

Mode visibility

Ride-time screens should appear only when appropriate conditions exist.

Examples:
	•	navigation screens appear only when a route is active
	•	ride instrumentation screens appear only when GPS tracking is active

Controls should not appear in the interface if the underlying capability is unavailable.

⸻

Environmental context signals

Ride-time situational awareness may include:
	•	wind direction relative to route bearing
	•	temperature exposure
	•	precipitation probability or active precipitation
	•	sun glare risk
	•	UV intensity
	•	moon phase and night illumination

These signals come from the contextual environmental modeling layer already present
in the architecture.

⸻

Strategic position

This feature positions Lanterne as both a route intelligence system and a ride-time
environmental awareness tool.

Rather than competing directly with cycling computers, Lanterne functions as:
	•	a second screen companion, or
	•	a primary display for riders who prioritize environmental awareness over instrument data

⸻

Consequences

Advantages:
	•	leverages Lanterne's environmental modeling strengths
	•	creates a differentiated product category
	•	provides value both before and during rides
	•	avoids competing directly with cycling computer hardware

Tradeoffs:
	•	requires careful UI design to maintain glanceability
	•	introduces additional interface complexity if not constrained

⸻

Design principle

Ride-time mode answers one question:

"What is the environment doing to me right now?"

Not: how many metrics can we display?


---

## Source File: docs/03-adrs/adr-030-ride-mode-power-and-sensor-architecture.md

ADR-030 — Ride Mode Display, Power, and Sensor Architecture

Status: Proposed
Date: 2026-03-22

Companion spec: DS-012 (Ride Computer Tile System)
Related ADRs: ADR-027 (Lantern Screen Model), ADR-029 (Ride-Time Situational Awareness Mode)

Note: Status remains Proposed pending implementation.
      DS-012 is the implementation spec for the ride computer tile layer.
      Power mode and sensor architecture implementation specs are not yet written.

⸻

Context

Lanterne's broader architecture already separates:
	•	route geometry and normalization
	•	slice-level route analysis
	•	route intelligence indices
	•	environmental modeling
	•	ride-time presentation

That separation allows heavy route understanding to happen before the ride while keeping
ride mode focused on fast, glanceable context during the ride.

The current direction already includes key ride-time primitives:
	•	a GPS tracking position indicator / blue dot
	•	a color-coded heatmap route overlay
	•	a navigation engine built around GPS duty cycling, corridor snapping,
	  cue detection, and off-route detection

ADR-027 establishes that the lantern selector represents Lantern Screens organized
into a variable-length Lantern Stack. ADR-029 establishes that ride-time mode is a
situational-awareness mode surfacing environmental and navigational context.

If Lanterne is to be considered a serious bike-computer replacement by committed cyclists,
ride mode must eventually support local sensor inputs such as heart rate, cadence,
power, and radar.

Battery behavior is therefore not a side concern. It is part of the product definition.

⸻

Decision

Ride mode is implemented as a lantern-centered, extensible ride-time screen framework.

The lantern is:
	•	the home anchor
	•	the screen selector
	•	the gesture control surface
	•	the entry point into the ride-time Lantern Stack

Ride mode is a variable-length Lantern Stack, not a fixed sequence of pages.

⸻

Lantern Stack

Initial screen families may include:
	•	Ride Computer
	•	Hazards
	•	Stops
	•	Weather
	•	Controls
	•	Cues
	•	Field Notes
	•	future Lanterne-defined screens
	•	future partner-defined screens

Different ride contexts surface different stacks:
	•	road navigation → hazards, cues, weather, controls, ride computer
	•	bikepacking/gravel → remoteness, water, weather, surface, ride computer
	•	future partner integrations → injected screens without changing the interaction model

⸻

Traversal model

Upward traversal

Swiping or pulling upward traverses the primary situational-awareness stack:
	•	Hazards
	•	Stops
	•	Weather
	•	Controls
	•	Cues

Downward traversal

Swiping or pulling downward traverses the ride instrumentation stack:
	•	Ride Computer
	•	(future instrumentation screens)

In-screen flipping

Individual screen elements may support lateral flip gestures to reveal alternate faces
(e.g., switching between sub-metrics on a tile).

⸻

Power modes

Ride mode supports four power modes:

Standard
Normal GPS and screen refresh cadence.

Battery Saver
Reduced GPS duty cycle and screen refresh. Appropriate for long rides with charging risk.

Ultra
Maximum battery conservation. Minimum viable GPS and display.

Dynamic
A policy engine that automatically selects among the other three modes based on:
	•	current battery level
	•	battery outlook relative to route distance remaining
	•	charging opportunities along the route
	•	rider behavior patterns
	•	defined energy horizon thresholds

Dynamic mode is the most sophisticated power mode and is a future implementation target.

⸻

Sensor architecture

Ride mode supports a local sensor architecture for serious cycling hardware.

Initial sensor targets:
	•	heart rate (BLE/ANT+)
	•	cadence (BLE/ANT+)
	•	power (BLE/ANT+)
	•	radar (Garmin Varia or equivalent)

Stable route intelligence and live ride-time telemetry remain separate domains.
Sensor data feeds ride-time display and recording. It does not alter route analysis outputs.

⸻

Map behavior in ride mode

The map becomes quieter but does not disappear.
The route heatmap remains visually dominant.
OLED-aware map styles minimize battery draw on compatible displays.

⸻

Consequences

Product consequences:
	•	moves Lanterne closer to bike-computer replacement territory
	•	combines route intelligence, ride-time awareness, navigation, and sensor support

Engineering consequences:
	•	requires a clean Lantern Stack config model
	•	requires a power-policy engine
	•	requires OLED-aware map styles
	•	requires a ride-time sensor manager
	•	requires strong compute discipline to meet battery targets

Data model consequences:
	•	stable route intelligence and live ride-time telemetry remain separate domains

⸻

Non-Goals

This ADR does not define:
	•	exact SQL schema for ride-time telemetry
	•	exact BLE implementation details
	•	exact GPS cadence per power mode
	•	exact redraw thresholds
	•	exact battery projection formulas
	•	exact radar UI shape

Those belong in implementation specs.

Required follow-on specs:
	•	DS-012 covers the ride computer tile system (written)
	•	Power mode implementation spec (not yet written)
	•	Sensor connection lifecycle spec (not yet written)

⸻

Design principle

Ride mode is not a separate product inside Lanterne.
It is the live operational layer of the wider Lanterne system.
The lantern is the anchor. Screens are the content. Power is a first-class concern.


---

## Source File: docs/03-adrs/adr-031-model-multi-day-events-as-ordered-references-onto-canonical-geometry.md

# ADR-031: Model Multi-Day Events as Ordered References onto Canonical Geometry

- **Status:** Proposed
- **Date:** 2026-03-22
- **Deciders:** Lanterne
- **Supersedes:** None
- **Related:** ADR-001 Route Acquisition Model, ADR-005 Route Analysis Model, ADR-011, ADR-020

## Context

Lanterne’s route model is built around a core architectural rule:

> **Canonical routes represent stable reusable geometry.**
> Event structure, source artifacts, and rider-facing presentation are separate concerns.

This becomes critical for multi-day events, especially **cloverleaf formats** where multiple days start and end at the same hub location (for example, the same hotel) and may partially overlap one another.

A cloverleaf exposes a structural weakness in any model that assumes a canonical route is always a single linear itinerary. In practice, a multi-day event may consist of:

- multiple loop days sharing a common hub
- multiple source route files per day
- partial corridor overlap between days
- alternate packaging of the same geometry as a 1-day, 2-day, 3-day, or full-series ride

If itinerary structure is allowed to define canonical route identity, the model breaks down fast.

That failure mode is not subtle.

## Problem

The architecture must support multi-day events without corrupting canonical identity.

The specific failure to avoid is:

> **Baking itinerary structure into canonical route identity.**

Examples of the wrong direction:

- treating a 3-day cloverleaf as a single canonical route because the event says it is one thing
- storing `Day 1`, `Day 2`, and `Day 3` semantics inside the canonical route itself
- making canonical identity depend on event packaging rather than reusable geometry
- turning the canonical model into an itinerary container, segment manifest, and network object all at once

If we do that, we create a mess:

- canonical identity stops meaning geometric truth
- deduplication and overlap logic become fragile
- the same physical roads get multiple incompatible identities
- future reuse across events, vaults, and ride formats gets harder instead of easier
- slice-based analysis becomes subordinate to human packaging decisions

That is the architecture mistake that will absolutely destroy multi-day events.

## Decision

**Multi-day events SHALL be modeled as ordered references onto canonical geometry, not as canonicals defined by itinerary structure.**

### Core rule

- **Canonical routes** represent reusable geometry.
- **Events** represent itinerary/grouping.
- **Event days or event parts** represent ordered references onto canonical routes or slice ranges within canonical routes.

### Therefore

For cloverleaf and other multi-day formats:

1. **Each day loop SHOULD normally be its own canonical route**
   when it is independently meaningful, analyzable, and reusable as geometry.

2. The overall multi-day ride SHALL be represented by an **event parent record**.

3. Each day/part of the event SHALL reference:
   - a canonical route, or
   - an ordered list of canonical route slice ranges

4. Day ordering, labels, hotel/hub semantics, overnight structure, optional participation, and packaging SHALL live in the **event layer**, not in `canonical_routes`.

5. A canonical route SHALL NOT be upgraded into a multi-geometry itinerary/network container solely to express multi-day event structure.

## Rationale

### 1. Preserves canonical identity as stable geometric truth

Canonical routes need to stay as close as possible to “this is the reusable road corridor / loop / line” rather than “this is how one organizer packaged it one year.”

That keeps matching, deduplication, enrichment, and reuse sane.

### 2. Keeps slice-based analysis clean

Lanterne’s analysis model is based on computing on small internal units and rolling those up later. That only stays clean if itinerary packaging references geometry rather than redefining it.

A day is a view onto geometry.
It is not the geometry itself.

### 3. Supports reuse across many ride formats

The same geometry may later appear as:

- a single event day
- part of a 1200k series
- a compressed 2-day variant
- an optional day for riders joining late
- a permanent
- a vault route
- a rider’s saved route

Those uses should all point to the same underlying canonical geometry where appropriate.

### 4. Avoids false complexity in the canonical layer

A multi-geometry canonical with internal day manifests sounds powerful, but right now it would force `canonical_routes` to carry too many jobs:

- geometry identity
- event packaging
- ordering semantics
- day boundaries
- network topology
- source-file composition

That is bad architecture. One object doing five jobs is how systems get “clever” and then rot.

### 5. Handles hub-based cloverleaf ambiguity correctly

In a hotel cloverleaf, several days may start and end at the same place. Shared hubs make traversal and ordering ambiguous if modeled as one giant network canonical.

That ambiguity belongs in the **itinerary definition**, where ordered day/part references can describe exactly what happens.
It does **not** belong in canonical identity.

## Chosen Option

### Option A — One canonical per loop/day plus a parent event record

This ADR adopts **Option A** as the current architecture.

For a multi-day cloverleaf:

- each loop/day is typically its own canonical route
- the event ties those loops together
- day order is stored explicitly
- overlapping roads are handled by shared geometry, slice analysis, and overlap logic
- multiple imported source files for a single day are handled in ordered event-part composition

This fits the existing architecture and does not require expanding canonical routes into a network model prematurely.

## Rejected Option

### Option B — A multi-geometry canonical with a segment manifest

This option is rejected **for now**.

Reason:

It moves itinerary semantics into the canonical layer too early and makes the route identity model carry responsibilities that belong to events and event parts.

This may become useful in the future if Lanterne intentionally introduces a first-class **network canonical** concept for complex route systems. But that is a separate architectural move and should not be smuggled in through multi-day event support.

## Data Model Guidance

This ADR does not lock exact table names, but the schema should follow this shape.

### `canonical_routes`

Represents stable reusable geometry.

Suggested fields:

- `id`
- `geometry`
- `fingerprint`
- `geometry_fingerprint`
- `distance_m`
- `start_anchor`
- `end_anchor`
- `is_loop`
- `canonical_name`

### `route_slices` (or equivalent ADR-020 slice table)

Represents atomic internal analysis units for a canonical route.

Suggested fields:

- `id`
- `canonical_route_id`
- `slice_index`
- `start_m`
- `end_m`
- `geometry`
- stable analysis outputs / indices

### `events`

Represents the human/organizational container.

Suggested fields:

- `id`
- `name`
- `series_name`
- `event_type`
- `start_date`
- `base_location`
- `source`
- `notes`

### `event_days` or `event_route_parts`

Represents ordered itinerary components.

Suggested fields:

- `id`
- `event_id`
- `part_order`
- `label`
- `canonical_route_id`
- `start_slice_index`
- `end_slice_index`
- `overnight_location`
- `returns_to_hub`
- `is_optional`

Use a full-slice-span reference when the day corresponds to a whole canonical route.

### `event_route_part_segments` (only when needed)

Used when one event day is composed of multiple source artifacts or multiple canonical references in sequence.

Suggested fields:

- `id`
- `event_route_part_id`
- `sequence`
- `canonical_route_id`
- `start_slice_index`
- `end_slice_index`

This solves the “one day, multiple RWGPS files” problem without pretending the day itself is a canonical geometry object.

## Rules

### Rule 1
A **canonical route SHALL NOT encode event day semantics**.

### Rule 2
An **event day SHALL reference canonical geometry by canonical route and optional slice range**.

### Rule 3
If an event day is composed of multiple route files or route portions, composition SHALL be modeled in the **event layer**, not by inventing a special canonical.

### Rule 4
Shared hubs, repeated starts/ends, and organizer packaging SHALL be modeled as itinerary semantics, not canonical identity.

### Rule 5
Future support for graph/network canonicals, if ever introduced, MUST be a deliberate architectural expansion with its own ADR rather than an ad hoc extension to solve cloverleaf events.

## Consequences

### Positive

- preserves canonical route identity
- keeps deduplication and overlap logic stable
- supports slice-based analysis cleanly
- allows the same geometry to be reused across multiple event formats
- handles cloverleaf hubs without canonical ambiguity
- keeps future options open

### Negative

- multi-day event assembly requires explicit event/day modeling
- some events will need additional composition tables for ordered parts
- this does not yet provide a first-class route-network abstraction

These are acceptable tradeoffs. They are the cost of not doing something stupid in the canonical layer.

## Implementation Notes

1. Continue treating `canonical_routes` as reusable geometry units.
2. Build `route_slices` as the atomic analysis layer.
3. Add an event/day referencing model that points onto canonicals by slice range.
4. For cloverleaf events, ingest each meaningful day loop as its own canonical when appropriate.
5. Where a single day consists of multiple imported route files, model that as ordered event-part composition.
6. Do not introduce multi-geometry canonicals unless and until Lanterne explicitly decides to support first-class network canonicals via a separate ADR.

## Final Statement

Lanterne will model multi-day events as **ordered itinerary references onto canonical geometry**.

The system will not allow event packaging to redefine what a canonical route is.

---

## Source File: docs/03-adrs/adr-032-comparative-traffic-context-and-segment-cohorts.md

ADR-032 — Comparative Traffic Context and Segment Cohorts

Status: Accepted
Date: 2026-03-23

Companion spec: DS-013 (Comparative Traffic Context Schema)
Related ADRs: ADR-033 (Canonical Segment Identity), ADR-020 (Atomic Analysis Unit),
              ADR-026 (Canonical Route Identity)

Context

Lanterne's core Safety Score is defined narrowly: the likelihood of a rider being struck by a motor vehicle and the severity of the likely outcome.

That definition is intentional and must remain stable.

However, safety analysis produces richer signal than a single absolute score.

Two related problems emerged:

First, traffic behavior has multiple dimensions that the current model largely collapses:
	•	pass frequency — how often vehicles interact with the rider
	•	pass intensity — how fast and forceful those interactions are
	•	driver accommodation — whether drivers slow and give space

These dimensions vary meaningfully across geographies and road contexts and deserve distinct representation.

Second, a single road segment can belong to many overlapping comparative contexts at once.

Example:

A segment on a busy city street in Honolulu may simultaneously be:
	•	a Hawaii segment
	•	a major US city segment
	•	a city-limits segment
	•	an Ironman World Championship corridor
	•	a RUSA permanent route corridor
	•	a 600K-associated segment

A single discrete classification tree cannot represent that reality cleanly.

The system therefore needs a model that distinguishes:
	•	canonical segment facts
	•	comparative baselines
	•	cohort membership


Decision

Lanterne will use a three-layer model for segment traffic context.

canonical segment facts
↓
comparative traffic baselines
↓
segment cohort memberships


⸻

1. Canonical segment facts

The source of truth for what is known or inferred about a segment.

Contains:
	•	road and infrastructure characteristics
	•	observed traffic behavior (when available)
	•	inferred traffic behavior (deterministic from known inputs)
	•	predicted traffic behavior (model outputs)
	•	confidence and provenance metadata

Rule

Canonical facts are absolute.

They describe the segment as it is, not relative to anywhere else.

Purpose

To answer:

what do we know or infer about this segment?

⸻

2. Comparative traffic baselines

Regional and contextual averages used for benchmarking, priors, and explanatory context.

Examples:
	•	country-level pass frequency
	•	state-level vehicle speed distributions
	•	metro-level driver accommodation norms
	•	road-class-level traffic intensity bands
	•	urbanicity-stratified passing behavior

Rule

Baselines are used for:
	•	initializing predicted traffic-behavior fields when direct evidence is absent
	•	generating relative context for rider-facing explanations
	•	calibrating and auditing model outputs
	•	future model training slices

Baselines are not used to rescale the headline Safety Score.

A dangerous segment does not become safer because it is average for its region.

Purpose

To answer:

how does this segment compare to similar places and riding contexts?

⸻

3. Segment cohort membership

A many-to-many model allowing a segment to belong to multiple overlapping comparative groups simultaneously.

Cohort types include:
	•	geography (country, state, metro, locality)
	•	urbanicity band
	•	event ecosystem (Ironman, Gran Fondo, etc.)
	•	route family (RUSA permanents, 600Ks, bikepacking classics)
	•	rider-usage patterns
	•	curated product collections

Rule

A segment may belong to any number of cohorts.

Cohort membership is not intrinsic to the segment fact row.
It is an analytical context layered on top of the segment.

Core, stable facts — posted speed, lane count, AADT, shoulder class, observed behavior — belong in structured columns.

Cohort membership belongs in a separate many-to-many table.

Purpose

To answer:

which comparative lenses apply to this segment?

⸻

4. Traffic behavior dimensions

Traffic behavior is not one thing.

Lanterne distinguishes three dimensions:

Exposure
How often vehicles interact with the rider.
Example inputs: predicted_passes_per_mile, predicted_passes_per_hour

Intensity
How forceful or fast those interactions are.
Example inputs: predicted_vehicle_speed_mph, predicted_speed_delta_mph

Courtesy / accommodation
Whether drivers slow and give space.
Example inputs: predicted_driver_slowdown_mph, predicted_pass_duration_sec

These dimensions may be scored independently and combined into a traffic behavior composite.

They feed the safety scoring subsystem. They do not replace it.

⸻

5. Absolute score vs relative context

The headline Safety Score remains absolute.

It is not graded on a national or regional curve.

Relative context belongs in the explanation layer, not the score itself.

Examples of valid relative context:
	•	"Passing speed is higher than typical for roads like this in Hawaii."
	•	"Driver accommodation is lower than the regional norm for this road class."
	•	"Unusually high pass frequency for a suburban arterial."

Examples of invalid score manipulation:
	•	Raising a segment's score because it is average for Brazil.
	•	Softening a danger rating because similar roads in the region are also dangerous.

Design principle

Absolute Safety Score: how risky is this segment for the rider?
Comparative context: how unusual is this segment relative to a chosen cohort?

These are different questions and must not be collapsed.

⸻

6. Naming discipline

Field naming must communicate provenance.

	•	observed_* — measured in the field
	•	inferred_* — deterministic from known segment truth
	•	predicted_* — model output
	•	baseline_* — regional or cohort prior
	•	confidence_* — evidence strength
	•	score_* — normalized Lanterne output

Examples:
	•	observed_passes_per_mile
	•	inferred_posted_speed_mph
	•	predicted_driver_slowdown_mph
	•	baseline_country_passes_per_mile
	•	confidence_traffic_behavior_model
	•	score_traffic_exposure

This naming scheme must be applied consistently across all traffic behavior tables.

⸻

7. Evidence precedence

When inputs conflict, apply this order:

	1.	observed — direct field measurement
	2.	inferred — deterministic from segment truth
	3.	predicted — model output
	4.	baseline — geographic or cohort prior

Lower-confidence inputs downweight their influence rather than silently overwrite stronger evidence.

⸻

8. Geography and road class cohorts are v1

Most cohort population can be deferred.

Geography and road class cohorts must not be deferred.

These are the baseline comparison inputs the scoring model already implicitly depends on. Without formalizing them as cohort rows, the relative context layer has no anchor and remains permanently deferred in practice.

Minimum v1 cohort scaffold:
	•	country
	•	state / province
	•	road class
	•	urbanicity band

All other cohort types may be populated later.

⸻

9. Raw observations are separate from inputs

Raw rider-contributed or sensor-imported evidence must not be written directly into the segment behavior inputs table.

A separate observations layer absorbs raw evidence and feeds the inputs layer through a quality and provenance review step.

This keeps the inputs table clean and prevents provenance from becoming ambiguous.

Note: This is where future Varia radar data belongs.

⸻

10. Consequences

Advantages:
	•	preserves trust in the absolute Safety Score
	•	supports richer traffic behavior modeling without coupling it to the headline score
	•	cleanly handles segments that belong to multiple comparative contexts
	•	keeps the fact model structured and queryable
	•	prevents premature score hard-baking
	•	enables future rider-facing relative context without score dilution

Tradeoffs:
	•	introduces a third conceptual layer beyond facts and scores
	•	requires governance of cohort definitions to prevent tag soup
	•	increases schema complexity modestly
	•	relative context UX must be kept disciplined to avoid undermining score trust

⸻

Non-Goals

This ADR does not require:
	•	building all comparative baselines immediately
	•	populating all cohort types at launch
	•	implementing rider-facing percentile or z-score displays now
	•	changing the current Safety Score definition or weights

This ADR establishes the model direction.
Implementation is phased.

⸻

Design principle

A segment has one canonical fact profile.

A segment may have many comparative contexts.

The Safety Score answers: how risky is this for the rider?
Comparative context answers: how unusual is this relative to its peers?

These are different questions and must stay that way.


---

## Source File: docs/03-adrs/adr-033-canonical-segment-identity.md

ADR-033 — Canonical Segment Identity and Route-to-Canonical Mapping

Status: Accepted
Date: 2026-03-23

Companion spec: DS-013 (Comparative Traffic Context Schema)
Related ADRs: ADR-032 (Comparative Traffic Context and Segment Cohorts),
              ADR-026 (Canonical Route Identity), ADR-020 (Atomic Analysis Unit)

Context

Lanterne needs segment-level persistence that can do two things at once:
	•	preserve route-analysis-local truth, order, and lineage for a specific analyzed route
	•	accumulate stable segment knowledge across many routes, many analyses, and future rider and device observations

Those are not the same problem.

The comparative traffic context schema (spec-032) assumes canonical per-segment facts, but leaves two critical questions open:
	•	what key identifies a segment across analyses
	•	whether the segment fact layer is route-local or canonical across routes

If those questions are deferred, ingestion code will either:
	•	write route-local identifiers into tables that are supposed to be canonical, or
	•	collapse distinct route occurrences into a fake shared identity

Both outcomes create expensive cleanup later.

OSM also drifts over time:
	•	ways get split
	•	tags change
	•	directionality changes
	•	geometry is edited

Canonical segment identity therefore cannot be defined as "whatever the latest OSM way is," and cannot rely on a brittle raw geometry hash alone.

Decision

Lanterne will introduce a canonical segment layer that is distinct from route-analysis-local segment instances.

A route analysis pass produces segment instances.

Lanterne's long-lived traffic facts, cohort memberships, and future observations attach to canonical segments.

route analysis pass
↓
route_segment_instances (route-local truth, order, lineage)
↓
canonical_segments (stable, directed, semantic, cross-route)
↓
segment_behavior_inputs / segment_cohort_memberships / segment_observations


⸻

1. What a canonical segment is

A canonical segment is the smallest directed truth segment Lanterne uses for scoring and segment-level reasoning.

It is broken at every scoring-relevant semantic boundary, not at arbitrary display boundaries.

Segmentation boundaries include:
	•	road identity changes
	•	directionality changes
	•	posted speed changes
	•	lane-count changes
	•	bike-facility changes
	•	shoulder class changes
	•	access or classification changes
	•	other rules explicitly defined by the active segmentation schema version

A canonical segment is therefore:
	•	directed
	•	semantic
	•	scoring-aligned
	•	stable by design
	•	independent of any one uploaded route

⸻

2. What does not define a canonical segment

Canonical identity will not be defined by any one of these alone:

	•	road_id + direction — too coarse; one road can contain multiple safety-relevant transitions
	•	raw OSM way id — not stable under map edits
	•	geometry hash alone — too brittle for minor edits; carries no semantic meaning
	•	route-local segment index — an analysis artifact, not global identity
	•	route-local matcher output key — same as above

⸻

3. Canonical segment identifier strategy

The primary key of a canonical segment is a surrogate UUID:

canonical_segments.id uuid primary key

That UUID is the only durable foreign key used by:
	•	segment_behavior_inputs
	•	segment_cohort_memberships
	•	segment_observations (nullable until resolved)
	•	future segment-level aggregate tables

The UUID is backed by a deterministic identity scaffold.

Each canonical segment must store enough identity material to support deterministic matching, auditing, and controlled drift handling.

Required identity scaffold fields:
	•	network_source — e.g. osm, dot, custom
	•	direction — forward / backward / undirected
	•	segmentation_schema_version — integer; which set of boundary rules produced this segment
	•	start_anchor_key — stable node or coordinate anchor at segment start
	•	end_anchor_key — stable node or coordinate anchor at segment end
	•	geometry_hash_normalized — hash of simplified directed geometry after normalization
	•	semantic_signature — deterministic hash of scoring-relevant attributes: speed bucket, lane count, bike facility class, shoulder class; captures what makes this segment distinct for safety purposes
	•	map_snapshot_version — which OSM or source snapshot this was derived from
	•	is_active — boolean; false when superseded
	•	superseded_by_id — nullable uuid; points to replacement canonical segment when applicable

The UUID is stable for joins.
The identity scaffold explains why that UUID exists and how new route analyses should match to it.

⸻

4. Route analysis output is instance data, not canonical identity

Every route analysis pass writes a route-local mapping layer: route_segment_instances.

One row per segment occurrence in a specific route analysis.

This table preserves:
	•	analysis_id
	•	route_id
	•	segment_index
	•	route order
	•	local geometry slice
	•	canonical_segment_id (the resolved mapping result)
	•	match_method (exact / near_exact / new)
	•	match_confidence

This table is the bridge between what happened in one analysis pass and what segment Lanterne believes it corresponds to globally.

A route analysis pass must never write its local segment key directly into canonical fact tables.

⸻

5. Matching flow: route-local to canonical

When a route analysis emits local truth segments, Lanterne maps each one to a canonical segment in order.

Step 1 — Exact deterministic match

Match on active segmentation schema plus directed identity scaffold:
	•	same network_source
	•	same direction
	•	same anchor pair
	•	same geometry_hash_normalized
	•	same semantic_signature
	•	compatible map snapshot or approved lineage rule

Step 2 — Near-exact reuse

If exact match fails, attempt controlled geometric reuse against active canonical segments:
	•	endpoint proximity within tolerance
	•	directed overlap ≥ 85%
	•	semantic compatibility (matching semantic_signature or explicit override)
	•	road class and access sanity checks pass

This is not a fuzzy free-for-all. It is a bounded reconciliation step with a high overlap threshold.

If directed overlap is below 85%, do not reuse. Create new.

Step 3 — Create new canonical segment

If neither exact nor near-exact reuse is valid, create a new canonical segment row.

⸻

6. Bias in ambiguous cases

When mapping is ambiguous, Lanterne will bias toward creating a new canonical segment rather than silently merging.

False merges are worse than duplicates.

A duplicate can be reconciled later with explicit lineage.

A false merge contaminates facts, baselines, cohorts, and future observations with no clean recovery path.

⸻

7. OSM drift policy

Canonical segments are stable by design.

They are not mutated in place to chase every underlying OSM edit.

When OSM changes materially affect segment identity, Lanterne will:
	•	create a new canonical segment row
	•	mark the old canonical segment inactive when appropriate
	•	connect the lineage using superseded_by_id

Examples of material change requiring a new canonical segment:
	•	a segment is split into multiple safety-relevant parts
	•	directionality changes
	•	facility class changes
	•	speed, lane, or access semantics change enough to alter segmentation
	•	geometry realignment breaks deterministic identity

This is a versioned lineage model, not an in-place mutation model.

Minor OSM edits that do not alter the semantic_signature or anchor pair do not require a new canonical segment.

⸻

8. How facts, cohorts, and observations attach

Canonical facts

segment_behavior_inputs is keyed by canonical_segment_id.

It does not store route_id.

It describes what Lanterne knows or infers about the canonical segment itself, independent of which route traversed it.

Cohort memberships

segment_cohort_memberships is keyed by canonical_segment_id.

Cohorts apply to canonical segments, not to one route pass.

Observations

segment_observations may be ingested with canonical_segment_id nullable.

This allows ingestion to happen before canonical mapping is complete, while preventing raw unresolved data from polluting canonical fact tables.

Once resolved and accepted, observations may be attached to a canonical segment.

⸻

9. Required new tables

This ADR requires:
	•	canonical_segments
	•	route_segment_instances

Deferred but anticipated:
	•	canonical_segment_supersessions as a separate lineage table if split/merge history needs to stay queryable beyond the superseded_by_id field on the main table

⸻

10. Implementation rule

No ingestion or schema work may write segment-level traffic facts or cohort memberships against a free-text route-local segment_id.

Canonical tables require canonical_segment_id uuid referencing canonical_segments.

Route-local analysis output belongs in route_segment_instances.

⸻

11. What is safe to build before this ADR is implemented

Safe to build now:
	•	cohorts table and seed rows
	•	traffic_behavior_baselines
	•	segment_observations stub with nullable canonical_segment_id

Do not build yet:
	•	segment_behavior_inputs
	•	segment_cohort_memberships
	•	any ingestion logic that treats today's route-local matcher key as a durable segment identity

⸻

12. Consequences

Advantages:
	•	prevents route-local ids from leaking into canonical tables
	•	allows many routes to accumulate knowledge on the same real-world segment
	•	gives future observations a proper long-lived join target
	•	handles OSM drift without corrupting historical lineage
	•	keeps route-order and analysis-local truth intact in route_segment_instances
	•	makes cohorting and traffic baselines structurally honest

Tradeoffs:
	•	adds at least one required mapping table
	•	requires a deterministic segment construction contract
	•	introduces reconciliation and supersession logic
	•	increases migration and ingestion complexity
	•	makes "shove the current matcher key in a text column" impossible — which is correct even when inconvenient

⸻

Non-Goals

This ADR does not require:
	•	solving global cross-provider segment identity beyond the current road network source
	•	perfect automatic reconciliation of all future map edits
	•	backfilling all legacy route analyses immediately
	•	rider-facing UI changes
	•	changes to the headline Safety Score

⸻

Design principle

Route analyses produce occurrences.
Canonical segments accumulate knowledge.
Map drift creates lineage, not silent mutation.


---

## Source File: docs/03-adrs/adr-034-master-route-expeditions-and-windowed-analysis.md

# ADR-034 — Master Route Expeditions and Windowed Long-Route Analysis

**Status:** Draft  
**Date:** 2026-03-24

**Companion spec:** DS-034 (Route Expedition State and Windowed Analysis Spec)  
**Related ADRs:** ADR-026 (Canonical Route Identity), ADR-033 (Canonical Segment Identity), ADR-016 (Ride Session Data Model), ADR-020 (Atomic Analysis Unit)

---

## Context

Lanterne's route analysis model currently assumes a bounded working set:
- One uploaded route
- One analysis pass
- One active rendered result
- One live ride session

That assumption is acceptable for ordinary rides and many brevets. It is not sufficient for ultra-distance use cases.

**Three related problems emerged.**

### Problem 1 — Route upload size and analysis size are different problems

A rider may reasonably want to load a 600K, a multi-day bikepacking route, or a 3,000-mile race route as one coherent journey. But analyzing and rendering that entire corridor as one monolithic client-side working set creates failure risk:
- Too many GPX points in memory
- Too many corridor tiles fetched or indexed
- Too many roads held in memory
- Too much wall-clock analysis time
- Too much heatmap geometry to render on a phone

### Problem 2 — A live ride session is not the same as durable route progress

Crash recovery and browser-session continuity are useful, but they do not solve the multi-day case:
- Rider stops for sleep
- Phone powers down to recharge
- GPS is intentionally turned off
- App is reopened many hours later

At that point the system must remember where the rider was in the larger journey — not merely whether one browser session can be resumed.

### Problem 3 — Chunking is a delivery strategy, not a truth model

The rider should think in terms of one route and one expedition. The system may internally analyze and preload bounded windows, but that must not fragment route identity or canonical segment identity.

---

## Decision

Lanterne will support ultra-long routes through a **four-layer model**:

```
master route
    ↓
route expedition
    ↓
active analysis window
    ↓
live ride session
```

The uploaded route remains one coherent route. Detailed analysis and heatmap rendering operate on bounded windows. Durable route progress is stored independently from live ride session state.

---

## 1. Master Route

The master route is the rider's full uploaded or created route. It may be much larger than what the client can safely analyze in one pass.

**Rules:**
- The master route remains a single route identity
- The rider is never required to manually split a GPX file into multiple uploads just to make the system work
- Route chunking must not create fake route identities or fake canonical-segment boundaries
- Full-route overview rendering may use simplified geometry and reduced visual detail

> **Purpose:** Answer the question — *what is the full journey the rider intends to undertake?*

---

## 2. Route Expedition

A route expedition is the **durable, multi-day progress record** for one rider on one master route.

It is not the same thing as a live GPS session.

**It stores:**
- Where the rider started within the route
- Where the rider last confidently progressed to along the route
- Which detailed analysis window is currently active
- Which next window should be loaded or preloaded
- Whether the expedition is active, paused, completed, or abandoned

**Rules:**
- A rider may start from mile 0, from current location, from a chosen route mile, or by resuming prior progress
- Expedition progress must survive browser close, app crash, phone restart, GPS disable, and overnight charging
- Expedition progress does not expire on a short timer
- Expedition progress is matched to route distance and route point index — not just raw latitude/longitude

> **Purpose:** Answer the question — *where is this rider in the larger route journey, even if the live session has ended?*

---

## 3. Active Analysis Window

The active analysis window is the bounded route subsection that receives full detailed treatment:
- Corridor fetch
- Road matching
- Cue generation
- Detailed heatmap
- Near-route POI and navigation context

The active analysis window is an **operational working set**. It is not the rider's whole route identity.

**Rules:**
- A route may be accepted in full while only a bounded window is analyzed in detail
- Window boundaries are based on route distance and route point index
- Windows must overlap so the rider does not experience a hard seam at handoff
- The next window may be preloaded before the rider reaches the end of the current one
- Detailed window sizing is governed by working-set budget, not mileage alone
- Adaptive corridor width is an allowed and encouraged budget lever

**Default policy:**

| Parameter | Default | Hard max |
|-----------|---------|----------|
| Target detailed window | 250 miles | 400 miles |
| Window overlap | 10 miles | — |
| Preload trigger | 25 miles before window end | — |

These defaults are operational policy, not canonical route truth. They may be revised without changing route identity. A shorter window on a dense corridor may be safer than a longer window on a sparse one.

> **Purpose:** Answer the question — *what bounded part of the route should the phone fully analyze and render right now?*

---

## 4. Live Ride Session

A live ride session is the **transient runtime state** for the current outing:
- Active GPS tracking
- Elapsed and moving time
- Current on-screen mode
- Recent position samples
- Immediate cue progression

**Rules:**
- Live ride session continuity is useful but subordinate to expedition continuity
- Ending or losing a live ride session must not erase expedition progress
- Reopening the app may create a new live ride session attached to the same expedition
- Multi-day expeditions may contain many live ride sessions

> **Purpose:** Answer the question — *what is happening in the rider's current active outing right now?*

---

## 5. Start Modes

Lanterne will support the following expedition start modes:

| Mode | When used |
|------|-----------|
| `from_beginning` | Rider starts at mile 0 |
| `resume_existing` | Rider reopens a paused expedition |
| `from_current_location` | Rider joins at their current GPS position |
| `from_custom_mile` | Rider specifies a route mile to start from |
| `from_custom_point` | Rider selects a specific point on the route |

The system must not assume that every meaningful route use begins at mile 0.

---

## 6. Resume Behavior

On app reopen, Lanterne should first recover **expedition state**, not just browser-session state.

Resume logic should prefer the rider's last confidently matched route progress using: last confirmed route mile, last confirmed route fraction, last matched route point index, and current active window. Raw GPS coordinates are supporting evidence, not the sole source of truth.

**Case A — Current location is near last known route progress:**
- Offer one-tap resume
- Reopen the same active window
- Restore route progress
- Start a new live ride session under the same expedition

**Case B — Current location is materially far from last known route progress:**
- Show the mismatch clearly
- Allow the rider to choose between resuming from prior progress or joining at current location
- Do not silently advance or rewind expedition progress

**Case C — GPS not yet available:**
- Still show the last known route progress and current active window
- Allow the rider to reopen that route context before GPS lock

---

## 7. Chunking and Overlap

Windowing exists to constrain analysis and rendering cost. It does not change the meaning of the route.

Each expedition window has:
- A **core span** — the rider-visible planned section
- A **load span** — extends beyond the core span to include overlap

**Design rule:** Window overlap must be large enough to prevent abrupt heatmap, cue, or POI discontinuities at handoff boundaries. A slightly redundant working set is preferable to a brittle seam.

---

## 8. Full-Route Overview vs Detailed Analysis

Lanterne distinguishes between:

| Mode | Rendering |
|------|-----------|
| **Full-route overview** | Simplified geometry, reduced detail, possibly greyscale |
| **Detailed active-window analysis** | Full safety heatmap, detailed interaction, navigation context |

This distinction is required for ultra-long routes. It is **not** a degraded fallback — it is the intended operating model.

---

## 9. Persistence Model

| State type | Durability |
|------------|-----------|
| Expedition state | **Durable** — must survive all session/device interruptions |
| Live ride session state | **Transient** — useful for continuity within a session |

**Implementation rule:** No expedition-critical progress may exist only in ephemeral browser memory.

**Minimum durable fields:**
- `route_id`, `user_id`
- Expedition status
- Start mode
- Last confirmed route mile
- Last confirmed route point index
- Last progress timestamp
- Active window identity

Append-only expedition events are recommended for auditability and crash recovery.

---

## 10. Non-Goals

This ADR does not require:
- Full offline analysis of an entire 3,000-mile route in one shot
- Storing every GPS tick in the database
- Forcing the rider to plan every overnight stop up front
- Replacing canonical segment identity with expedition-window identity
- Changing the definition of the Safety Score

---

## 11. Consequences

**Advantages:**
- Supports multi-day and multi-week route continuity
- Lets Lanterne accept very long routes without pretending phones have infinite memory
- Preserves one coherent rider mental model: one route, one expedition, many sessions
- Separates durable progress from ephemeral browser runtime state
- Reduces the chance that a crash or overnight stop erases operational context
- Creates a clean future foundation for expedition-grade navigation and state recovery

**Tradeoffs:**
- Introduces new persistent state beyond `route_history`
- Requires explicit resume and mismatch UX
- Adds window-planning and preload logic
- Requires care so chunking does not leak into canonical segment identity or route truth
- Creates more state transitions to test under fatigue and poor connectivity

---

## Design Principle

One route. One expedition. Many bounded windows. Many live sessions.

The rider's journey stays whole even when the analysis working set does not.

