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

_Internal Documentation — Updated 2026-04-04 (v3.1-launch)_

---

## 1. Overview

### What Lanterne Is

Lanterne is a mobile-first Progressive Web App that provides cyclists with segment-level safety analysis of their routes. Users upload a GPX file or create a route on-map, and the platform analyzes every road segment for risk factors — speed, traffic volume, shoulders, bike infrastructure, and crossing exposure — producing a composite safety grade (A+ through F) and a color-coded heatmap overlay. Railroad crossings and bridge hazards are detected and shown as a separate hazard layer.

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
| Metal grate bridge | 3 | `bridge ∈ {yes, movable} + surface=grate/grid/metal_grid` |
| Metal plate bridge | 2 | `bridge ∈ {yes, movable} + surface=metal/steel` |
| Cattle grid | 2 | `barrier=cattle_grid` |
| Single-lane underpass | 2 | `tunnel=yes + lanes=1 or maxwidth<4` |
| Covered bridge | 1 | `bridge ∈ {yes, movable} + (covered=yes or bridge:structure=covered)` |
| No-shoulder bridge | 2 | `bridge ∈ {yes, movable} + no cycling infra + narrow` |

### Step 11 — Segment Scoring - OUT OF DATE - SEE v4 UPDATE DOC

**Purpose:** Compute per-segment and aggregate route risk.
**Input:** Enriched truth segments.
**Output:** `TotalRisk`, `RiskPerMile`, `SafetyScore` (0–100), `LetterGrade` (A+ to F).
**Code:** `safety-scoring.ts → computeRouteSafetyScore()`.

**V3.1-Launch Scoring Model:**

The core risk for each continuous segment is:

```
SliceRisk = SliceMiles × (0.60 × SpeedFactor + 0.40 × TrafficFactor) × InfraFactor × ShoulderFactor
```

Where SpeedFactor uses a piecewise-linear breakpoint table (≤20→0.50 through 55+→6.20), TrafficFactor uses a 6-step AADT-per-lane ladder with a fallback evidence chain, InfraFactor is multiplicative (protected 0.50, buffered 0.68, painted 0.82, none 1.00), and ShoulderFactor applies only when speed ≥ 30 and no bike facility exists.

**Crossing risk contribution** replaces the former left-turn penalty: `min(0.75, 0.05 × √(SpeedFactor × TrafficFactor) × Width × Control × Movement)`, capped so crossings never exceed 40% of total raw canonical risk.

**Excluded from headline score:** rail crossings, bridge hazards, time-of-day traffic, critical stretch penalties, weather, surface, fatigue. These are report-only layers.

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
| **Safety Scoring** | 0–100 score with A+–F grade. V3.1 model: speed(60%)/traffic(40%) core weights × multiplicative bike infra × shoulder factor + bounded crossing risk contribution. Rail, critical stretch, time-of-day traffic are report-only, NOT in headline score. | `safety-scoring.ts`, `safety-constants.ts` |
| **Heatmap** | Speed-colored road overlays with zoom-banded truth-to-display rendering | `heatmap/builder.ts`, `RouteMap.tsx` |
| **Corridor Analysis** | Fetches and indexes the broader road network; builds CorridorGraph for local pathfinding | `corridor.ts`, `corridor-graph.ts` |
| **Micro-Hazard Detection** | 7 hazard types from OSM tags (railroad crossings, metal grate bridges, cattle grids, covered bridges, etc.) | `hazards.ts` |
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

## Source File: docs/02-architecture/arch-004-system_guide_v4_safety_score_update.md

# Lanterne System Guide

_Internal Documentation — Updated 2026-04-04 (v4, safety-model update)_

---

## 1. Overview

### What Lanterne Is

Lanterne is a mobile-first Progressive Web App that provides cyclists with segment-level safety analysis of their routes. Users upload a GPX file or create a route on-map, and the platform analyzes every road segment for risk factors — speed, traffic, shoulder and bike operating space, bounded crossing conflicts, and route-specific hazards — producing a rider-facing grade, heatmap, cue sheet, and decision-support surfaces.

### The Problem

Cyclists lack granular, road-level safety data. Existing tools may show bike lanes or route lines, but they do not explain how dangerous a specific road segment or crossing is likely to be.

### Core Workflow

```text
GPX Upload → Corridor Fetch → Road Matching → Data Enrichment → Hazard Detection → Safety Scoring → Visualization
```

### What Makes It Unique

1. **Client-side analysis** — all computation runs in the browser; infrastructure cost stays low because the user’s device is the compute engine.  
2. **Multi-source data fusion** — OSM road geometry + federal HPMS traffic + state DOT attributes + imported route-source metadata such as RWGPS surface when available.  
3. **Shared geographic tile caching** — every user warms global caches for others in the same geography.  
4. **Truth-segment scoring** — scoring operates on canonical internal route slices rather than purely visual map segments.  
5. **Two-pass forensic matching** — coarse matching followed by targeted re-analysis in suspicious zones.  
6. **Narrow safety definition** — the canonical Safety Score is limited to motor-vehicle strike exposure and likely severity, not a kitchen-sink route score.  

---

## 2. Architecture

### Computation Model

All CPU-intensive work — GPX parsing, spatial indexing, road matching, forensic analysis, scoring, heatmap building, cue generation — runs in the browser. The backend is a data-and-proxy layer: it stores caches/user data and proxies rate-limited external APIs. No analysis runs server-side.

### Safety-model summary

The launch canonical Safety Score has two parts:

1. **Continuous segment exposure risk**  
   Built from non-linear speed environment, traffic exposure, and operating-space mitigation.

2. **Bounded crossing risk contribution**  
   Built from discrete score-bearing crossing events using speed, traffic, width, control, and movement context.

The canonical score explicitly excludes:
- weather/light conditions
- critical-stretch penalties
- time-of-day bell-curving of AADT
- rail / non-motor-vehicle hazard penalties

Those may still appear in report and explanation layers.

---

## 3. Analysis Pipeline

### Step 1 — GPX Ingestion
Parses route geometry and metadata from GPX, RWGPS, or other supported sources.

### Step 2 — Corridor / route-source enrichment
Builds the route corridor, imports route-source metadata, and preserves provenance.

### Step 3 — Road matching and truth segmentation
Routes are matched to roads and broken into small internal slices / truth runs. This keeps the score tied to real changes in road context rather than only large visual segments.

### Step 4 — Data enrichment
Enriches matched roads and slices with:
- posted speed or speed proxy
- AADT where available
- lane counts and road class
- bike facility / shoulder context
- surface context
- hazard context

### Step 5 — Safety scoring
Canonical Safety Score is computed in two layers:

#### A. Continuous segment exposure
For each internal slice:

```text
ContinuousSliceRisk
= SliceMiles
× (0.60 × SpeedFactor + 0.40 × TrafficFactor)
× InfraFactor
× ShoulderFactor
```

#### B. Crossing risk contribution
For each score-bearing crossing event:

```text
CrossingEventContribution_i
= min(E_cap,
      E0 × (SpeedFactor_raw × TrafficFactor_raw)^0.5
         × WidthFactor × ControlFactor × MovementFactor)
```

Launch policy constants:
- `E0 = 0.05 crossing risk points`
- `E_cap = 0.75 crossing risk points`

The route-level crossing contribution is bounded so crossings cannot exceed 40% of raw canonical route risk.

### Step 6 — Route rollup

```text
ContinuousRPM = TotalContinuousRisk / RouteMiles
RawCrossingRiskContributionPerMile = TotalCrossingRiskContribution / RouteMiles
EffectiveCrossingRiskContributionPerMile = min(RawCrossingRiskContributionPerMile, ContinuousRPM × 0.6667)
RawRPM = ContinuousRPM + EffectiveCrossingRiskContributionPerMile
SafetyScore = 100 / (1 + e^(1.4 × (RawRPM - 2.5)))
```

### Step 7 — Report-only interpretive layers
These may appear in the analysis drawer or route explanation but do not modify the canonical score:
- critical stretch / hotspot reporting
- time-of-day contextual traffic interpretation
- rail and other non-motor-vehicle micro-hazards
- weather / light conditions

---

## 4. Data precedence and truth rules

### Surface truth
Rider-facing surface truth should follow this precedence order:
1. RWGPS explicit surface
2. Admin verified surface truth
3. explicit mapped surface tags
4. inferred surface
5. unknown

Brevet-specific official gravel distance is a separate concept and should remain in brevet-specific logic, not in the general surface-truth pipeline.

### Traffic fallback
Traffic truth follows a descending-confidence ladder:
1. official AADT per lane
2. official AADT total + known lane count
3. official AADT total + inferred lane count
4. inferred AADT total from nearby AADT values by highway type
5. generic road-class / highway-type traffic proxy
6. unknown

### Shoulder classes
- sub-usable shoulder: < 2.0 ft / 0.6 m
- usable shoulder: 2.0 ft to < 8.0 ft / 0.6 m to < 2.4 m
- wide shoulder: ≥ 8.0 ft / 2.4 m

Sub-usable shoulder remains visible to riders but earns no safety credit.

---

## 5. Presentation rules

### Canonical score vs explanation layers
The canonical Safety Score answers:

> how much motor-vehicle danger this route exposes a cyclist to

The report / explanation layers answer additional rider questions, such as:
- where danger concentrates
- how ugly the worst crossing is
- what traffic may feel like by time of day
- what hazards are present that do not belong in the headline score

### Public-facing traffic display
Whenever traffic is shown publicly as a daily figure, the UI should also provide at least one rider-readable equivalent:
- cars per hour average
- cars per minute average

---

## 6. Current strategic priorities

- stabilize the final V3 safety model implementation
- keep stale V2 scoring language from leaking into live docs or UI
- preserve narrow score semantics
- support route-source precedence cleanly (especially RWGPS surface)
- keep guest mode and client-side compute intact

---

## 7. Guiding rules

1. Keep Safety Score narrow.  
2. Keep route explanation richer than route score.  
3. Compute on small slices, present on readable segments.  
4. Separate canonical score from contextual interpretation.  
5. Use clear precedence ladders for truth and confidence.  
6. Do not let unknown silently turn into asserted truth.  


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

## Source File: docs/02-architecture/arch-006-experience_policy_layer_index.md

# Lanterne Launch Packet — Index and Companion Guide

**Status:** Draft  
**Date:** 2026-03-31  
**Purpose:** Unify the three launch-planning documents into one coherent packet so they can be read and executed as a single system rather than as isolated specs.

---

## What this packet is

This launch packet is the working architecture and implementation stack for Lanterne’s next major build phase.

It is designed to answer three different questions that should not be collapsed into one doc:

1. **What is the architecture and program shape?**  
2. **How do we implement it in sequence?**  
3. **How should it actually behave at runtime?**

Those questions map to three companion documents.

---

## Read order

Read these in this order:

### 1. EXEC-008 v2 — Experience Runtime, Surface Architecture, and Domain Migration Program
Read first.

This is the **master architecture/program plan**.
It defines:
- the core runtime-first architecture
- the major system boundaries
- the separation of mode, audience, and structure
- the three execution programs
- the domain tracks
- the target file/module shape
- the SQL and Lovable sequencing strategy at a high level

This is the answer to:

> What are we actually building, and how is it organized?

---

### 2. EXEC-008 v2 — Master Implementation Manual
Read second.

This is the **execution manual**.
It defines:
- phase order
- dependency fences
- phase goals
- acceptance criteria
- exact SQL sequence
- copy/paste Lovable prompts
- verification checklist
- frozen assumptions that should not drift mid-build

This is the answer to:

> In what order do we build this, and what exactly should we run?

---

### 3. DS-016 — Experience Policy Layer
Read third.

This is the **behavioral operating spec**.
It defines:
- canonical axes
- runtime states and sub-states
- transition rules
- prompt/caption logic
- surface-routing policy
- input request eligibility
- mode differences
- audience-role differences
- public route page behavior
- calmness / anti-spam rules

This is the answer to:

> Once the architecture exists, how should the app behave?

---

## How the three docs relate

### EXEC-008 v2 Program Plan
Owns:
- system boundaries
- architecture shape
- domain decomposition
- overall sequencing philosophy

Does **not** own:
- exact SQL migration code
- final runtime prompt logic
- detailed launch interaction tables

### Master Implementation Manual
Owns:
- execution order
- exact SQL migration sequence
- Lovable prompts
- phase gates
- build discipline

Does **not** own:
- final product behavior policy
- visual design decisions
- score semantics beyond what other specs already define

### DS-016 Experience Policy Layer
Owns:
- runtime behavior model
- surface routing rules
- prompt and caption arbitration
- input request policy
- mode- and audience-sensitive behavior

Does **not** own:
- component implementation details
- SQL migrations
- scoring formula internals
- drawer shell physics internals

---

## Quick use guide

### If you are deciding architecture
Start with:
- EXEC-008 v2 Program Plan

### If you are about to code or migrate schema
Start with:
- Master Implementation Manual

### If you are deciding what should happen in a specific user scenario
Start with:
- DS-016 Experience Policy Layer

### If you are handing work to Lovable
Use:
- Master Implementation Manual first
- DS-016 as behavioral guardrails

### If you are reviewing whether something belongs in map, tiles, lantern, drawer, review, or public page
Use:
- DS-016 first
- then confirm architectural fit in EXEC-008 v2

---

## Launch packet principles

Across all three docs, these principles are fixed:

- active ride is map-first
- drawers are not the center of gravity during motion
- mode is separate from structure
- audience role is separate from mode
- push is first-class
- expedition is the durable parent for multi-push journeys
- canonical route identity is the center of new durable systems
- Vault is curated
- History is personal
- public route pages are first-class launch scope
- Pre-Ride Notes are constrained launch observations, not full Field Notes
- runtime truth must not be buried inside component-local conditionals

---

## Suggested header block to place at the top of each companion doc

Use this block near the top of each document so the packet stays self-reinforcing:

```md
## Launch Packet Companion Note

This document is one part of the Lanterne launch packet.

Companion documents:
1. EXEC-008 v2 — Experience Runtime, Surface Architecture, and Domain Migration Program
2. EXEC-008 v2 — Master Implementation Manual
3. DS-016 — Experience Policy Layer

Use this document for its primary job only:
- Program Plan = architecture and system boundaries
- Implementation Manual = execution order, SQL, prompts, verification
- DS-016 = runtime behavior, prompts, routing, and state policy
```

---

## Recommended next move

Use the packet like this:

1. Freeze the three docs as the current launch architecture stack.
2. Run implementation from the **Master Implementation Manual**.
3. Use **DS-016** whenever a behavior question appears during implementation.
4. Update **EXEC-008 v2** only when a genuine architecture decision changes.
5. Do not let Lovable invent new architecture outside these docs.

---

## Practical bottom line

These three docs now form one coherent launch packet:

- **Program Plan** = what the system is
- **Implementation Manual** = how to build it
- **Experience Policy Layer** = how it behaves

That is enough structure to stop building by thread vibes and start building with an actual operating manual.



---

## Source File: docs/02-architecture/arch-007a-lovable_route_rendering_architecture.md

┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1 — FILE INPUT                                          │
│  File: src/components/GpxUpload.tsx                            │
│  Function: handleFile()                                        │
│  Action: FileReader reads .gpx → raw XML string                │
│  Calls: props.onFileLoad(xmlString, fileName)                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2 — GPX PARSE                                           │
│  File: src/pages/Index.tsx                                     │
│  Function: handleGpxLoad() → runAnalysis()                     │
│  Calls: parseGpx(xml) from src/lib/gpx.ts                     │
│  Output: GpxRoute { name, points[], cuePoints[], fileType,     │
│          totalDistanceMi }                                     │
│  Also: validateRouteIngest() — guardrails on size/distance     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3 — NORMALIZATION                                       │
│  File: src/pages/Index.tsx → src/lib/normalized-route.ts       │
│  Function: normalizeFromGpxRoute(route, source)                │
│  Action: Produces NormalizedRoute — consistent dense geometry,  │
│          dwell filtering, cue binding, distance recomputation   │
│  Output: NormalizedRoute (used as canonical GpxRoute downstream)│
│  Side effects: setGpxRoute(), setGpxCuePoints(), etc.          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4 — PROGRESSIVE ANALYSIS                                │
│  File: src/pages/Index.tsx → src/lib/route-analysis.ts         │
│  Function: runProgressiveAnalysis() iterates                   │
│            analyzeRouteProgressive() async generator            │
│                                                                │
│  Yields stages in order:                                       │
│    shell      → basic geometry metrics                         │
│    corridor   → OSM road fetch (Overpass)                      │
│    enrichment → HPMS/DOT/railroad data                         │
│    forensic   → forensic zone analysis                         │
│    refinement → boundary refinement                            │
│    scoring    → left turns, risk scoring                       │
│    analysis   → final SafetyResult                             │
│                                                                │
│  MATERIALIZER (inside analysis stage):                         │
│    - truth-map construction (road matching per sample)         │
│    - truthRuns[] emission (one per road segment)               │
│    - PASS 2 boundary refinement                                │
│    - routeSpeedSegments[] (display compaction — legacy)        │
│    - truthSegments[] built from truthRuns                      │
│                                                                │
│  Output: SafetyResult { truthRuns, truthSegments,              │
│          routeSpeedSegments, score, grade, ... }               │
│  Side effect: setGpxAnalysis(result)                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5 — HEATMAP BUILD                                       │
│  File: src/pages/Index.tsx (useEffect) →                       │
│         src/lib/heatmap/builder.ts                             │
│  Function: buildHeatmapLayers({ routeLine, truthRuns })        │
│                                                                │
│  Sub-steps (when DEBUG.OVERLAY_DEBUG is OFF):                  │
│    1. Build truthSegments from truthRuns (1:1)                 │
│    2. ensureCoverage() — gap fill                              │
│    3. suppressNoiseSegments() — absorb micro-segments          │
│    4. mergeForZoom() per zoom band (low/mid/high)              │
│       uses isMeaningfulBoundary() to decide merges             │
│                                                                │
│  When DEBUG.OVERLAY_DEBUG is ON:                               │
│    → bypasses all merging, returns raw 1:1 truthRun segments   │
│                                                                │
│  Output: HeatmapBuildOutput {                                  │
│    truthSegments,                                              │
│    displaySegmentsByZoom: { low, mid, high },                  │
│    animationPath                                               │
│  }                                                             │
│  Side effect: setHeatmapOutput(output)                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 6 — SEGMENT SELECTION                                   │
│  File: src/components/RouteMap.tsx (useEffect ~line 1670)      │
│                                                                │
│  if heatmapOutput exists:                                      │
│    displaySegs = truthMode                                     │
│      ? heatmapOutput.truthSegments        (raw boundaries)     │
│      : heatmapOutput.displaySegmentsByZoom[zoomBand]  (merged) │
│    hitboxSegs = heatmapOutput.truthSegments (always raw)       │
│  else:                                                         │
│    fallback to legacy routeSpeedSegments                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 7 — LEAFLET RENDER                                      │
│  File: src/components/RouteMap.tsx                             │
│         src/lib/heatmap/leaflet-gradient-layer.ts              │
│                                                                │
│  Normal mode (gradient):                                       │
│    renderGradientLayer() — continuous color blending from       │
│    displaySegs onto routeCoords polyline                       │
│                                                                │
│  Truth/HC/debug mode (discrete):                               │
│    forEach segment → L.polyline() with speed-class color       │
│    (halo + casing + core layers)                               │
│                                                                │
│  Hitbox layer: always from truthSegments (invisible polylines  │
│  for click detection → road card)                              │
└─────────────────────────────────────────────────────────────────┘


---

## Source File: docs/02-architecture/arch-007b-cgpt_gpx_to_render_pipeline_and_drift_guide.md

# GPX → Render Pipeline and Drift Correction Guide

## Purpose

This document explains:

1. All known stages of the GPX → render pipeline
2. What issues were previously identified and fixed
3. Where visual drift originates
4. Where drift MUST be corrected to avoid repeating mistakes

---

## Pipeline Overview

The system transforms a GPX route into a rendered, scored, and colored map through several stages.

### Stage 0 — Input Acquisition
- GPX upload
- RWGPS import
- route drawing

Output:
- raw route geometry (user input)

---

### Stage 1 — Normalization

Purpose:
- standardize geometry
- remove noise
- ensure consistent spacing

Key operations:
- resampling / densification
- coordinate cleanup
- interpolation (now unified across sources)

Fixes applied:
- removed source-dependent normalization differences
- ensured Strava and RWGPS are treated consistently

---

### Stage 2 — Pass 1 Matching (Truth Generation)

Purpose:
- map route points onto OSM roads
- generate "truth runs"

Key outputs:
- matched road IDs
- segment boundaries
- per-run attributes

Known issues addressed:
- incumbent cascade instability
- direction-dependent matching
- reconciliation improvements

Remaining behavior:
- produces accurate road sequence but not display-aligned boundaries

---

### Stage 3 — Truth Runs

Definition:
- contiguous segments of the same matched road

Characteristics:
- exist in densified / analysis space
- indexed by sample points

Important:
- NOT aligned to original geometry

---

### Stage 4 — Pass 2 Boundary Reconstruction

Purpose:
- convert truth runs into display-ready boundaries

THIS IS THE CRITICAL STAGE FOR DRIFT

Responsibilities:
- map truth transitions to original route geometry
- determine exact transition locations
- prepare renderable segments

---

### Stage 5 — Rendering

Purpose:
- draw colored segments
- display road names

Consumes:
- display-aligned segments

Important:
- renderer should NOT fix geometry errors

---

## Root Cause of Drift

Drift occurs because:

> Boundaries are computed in truth space but rendered in display space

This causes:
- early transitions
- late transitions
- misaligned road names

---

## Correct Model

All visible geometry must be anchored in:

> **Display geometry (original route)**

NOT:
- densified points
- truth indices

---

## Where Drift MUST Be Fixed

### Only correct location:

> **Pass 2 Boundary Reconstruction**

NOT:
- matcher
- normalization
- renderer

---

## Correct Boundary Algorithm

For each transition:

1. Identify transition in truth space
2. Determine approximate location
3. Project onto original route geometry
4. Snap to intersection if applicable


### Projection
- use nearest-point projection onto polyline
- NOT index-based mapping

### Snapping
- if near intersection
- snap to node or shared endpoint

---

## What Was Fixed

- normalization parity across sources
- canonical identity issues (separate concern)
- token cleanup (B-lite)
- reconciliation improvements

These improved data quality but did not solve drift

---

## What Was NOT Fixed (by design)

- boundary placement precision
- display alignment

These belong to Pass 2

---

## Debugging Tools Required

1. Truth mode (densified view)
2. Display mode (rendered route)
3. Boundary markers
4. Intersection visualization

---

## Debug Workflow

1. Load known route
2. Enable truth mode
3. Compare truth vs display boundaries
4. Identify offset
5. verify projection correctness
6. verify snapping

---

## Success Criteria

- transitions align exactly with intersections
- road names switch at intersections
- no early/late drift
- consistent across all route types

---

## Anti-Patterns to Avoid

Do NOT:
- adjust boundaries in renderer
- apply constant offsets
- rely on indices
- "eyeball" fixes

---

## One Sentence Summary

The system correctly identifies roads — drift exists because boundaries are placed in the wrong coordinate space.



---

## Source File: docs/02-architecture/arch-009-osm_tag_accounting_audit.md

# Plan: OSM Tag Accounting Audit Document

## What this delivers

A single comprehensive markdown file (`docs/osm-tag-accounting.md`) documenting every OSM tag used across Roads, POIs, and Hazards domains, grounded in actual code citations.

## Summary of findings from audit

Based on thorough codebase inspection:

- **Roads**: ~30 unique OSM tags actively consumed
- **POIs**: ~18 unique OSM tags (top-level classification + metadata)
- **Hazards**: ~15 unique OSM tags (many shared with Roads)
- **Cross-domain shared tags**: `highway`, `surface`, `bridge`, `tunnel`, `lanes`, `maxspeed`, `railway`, `name`, `ref`, `cycleway`, `shoulder`, `bicycle`

## Document structure

### Section 1 — Executive summary
Counts, shared tags, dead/stale tag identification.

### Section 2 — Roads tag inventory
Table covering all tags consumed during road parsing/normalization across:
- `src/lib/overpass.ts` (parseRoads, Overpass queries)
- `src/lib/corridor.ts` (parseOverpassRoads, Overpass queries)
- `src/lib/corridor-chunks.ts` (parseOverpassRoads, Overpass queries)
- `src/lib/speed-utils.ts` (isSafePathType, parseShoulder, detectBikeFacility, parseMaxspeed, isSidewalkType)
- `src/lib/heatmap/types.ts` (surfaceToBucket)
- `src/lib/heatmap/admin-verification-overlays.ts` (classifyBikeInfra, classifyShoulder)
- `src/pages/Index.tsx` (roadToSegment)
- `pipeline/src/osm-facts.ts` (all normalization functions)
- `pipeline/src/road-class.ts` (normalizeRoadClass)
- `pipeline/src/speed-utils-node.ts` (mirror of speed-utils.ts)

Tags to document (non-exhaustive):
`highway`, `maxspeed`, `surface`, `lanes`, `cycleway`, `cycleway:right`, `cycleway:left`, `cycleway:right:buffer`, `cycleway:left:buffer`, `cycleway:buffer`, `shoulder`, `shoulder:left`, `shoulder:right`, `shoulder:width`, `shoulder:both:width`, `shoulder:right:width`, `shoulder:left:width`, `bridge`, `tunnel`, `name`, `ref`, `railway`, `man_made`, `bicycle`, `footway`, `sidewalk`, `motor_vehicle`, `motorcar`, `access`, `junction`, `tracktype`, `width`, `maxwidth`, `covered`, `bridge:structure`, `material`, `lcn`, `ncn`, `_hazard_node` (synthetic), `_bridge_outline` (synthetic)

Detailed notes subsections for:
- Bike infrastructure classification (multi-tag logic across 3 files)
- Shoulder parsing (6 tags → ShoulderInfo)
- Speed parsing/defaulting (maxspeed → mph conversion)
- Safe-path logic (highway + bicycle + footway + sidewalk + motor_vehicle/motorcar/access)
- Surface classification (surface + tracktype + highway inference)
- Road class normalization (highway → 7-class enum)

### Section 3 — POI tag inventory
Tags from `src/lib/pois/fetchPoisRaw.ts` (Overpass queries) and `src/lib/pois/parsePois.ts` (classification):

**Query tags**: `tourism`, `amenity`, `shop`, `natural`, `emergency`
**Classification values**: `hotel`, `hostel`, `camp_site`, `post_office`, `restaurant`, `supermarket`, `convenience`, `fuel`, `cafe`, `fast_food`, `bakery`, `drinking_water`, `spring`, `toilets`, `shower`, `hospital`, `clinic`, `pharmacy`, `fire_station`, `police`, `phone`
**Metadata tags**: `name`, `opening_hours`, `phone`, `contact:phone`, `website`, `contact:website`, `url`, `toilets:type`, `portable`

### Section 4 — Hazards tag inventory
Tags from `src/lib/hazards.ts` and Overpass queries:

**Detection tags**: `railway` (level_crossing, crossing, rail, light_rail, narrow_gauge), `barrier` (cattle_grid), `bridge`, `tunnel`, `surface`, `covered`, `bridge:structure`, `material`, `lanes`, `width`, `maxwidth`, `maxspeed`, `highway` (traffic_signals, stop), `cycleway`, `cycleway:left`, `cycleway:right`, `shoulder`, `bicycle`, `man_made` (bridge)

Hazard-specific multi-tag logic:
- Metal grate bridge: bridge ∈ {yes, movable} + surface ∈ {grate, grid, metal_grid, steel_grid, ...}
- Metal plate bridge: bridge ∈ {yes, movable} + surface ∈ {metal, steel, metal_plate, steel_plate}
- No-shoulder bridge: bridge ∈ {yes, movable} + NO (cycleway|shoulder|bicycle=designated) + narrow (lanes=1|width<6|maxwidth<3)
- Covered bridge: bridge ∈ {yes, movable} + (covered=yes | bridge:structure=covered)
- Single-lane underpass: tunnel=yes + (lanes=1 | maxwidth<4)
- Bridge outline inheritance: man_made=bridge ways → surface/material tag → inherits to overlapping road ways

### Section 5 — Transform/function map
Reverse index: function → tags read → output. ~20 key functions across 10 files.

### Section 6 — Gaps / inconsistencies / tech debt
Known items to call out:
1. `roadToSegment()` in Index.tsx duplicates bike facility detection with inline logic instead of calling `detectBikeFacility()` from speed-utils.ts — uses `cycleway:buffer` tag that canonical function doesn't check
2. `corridor-chunks.ts` parseOverpassRoads does NOT extract `man_made=bridge` ways (missing from way filter condition), unlike `corridor.ts` — bridge outline inheritance only works in tile-based fetching, not chunk-based
3. Pipeline `osm-facts.ts` checks `lcn` and `ncn` tags for bike route classification — browser code does not
4. `shoulder:left:width` checked in admin overlay but not in canonical `parseShoulder()`
5. Three near-identical `parseOverpassRoads` functions (overpass.ts, corridor.ts, corridor-chunks.ts) with slight tag handling differences

## Implementation approach

Single file write to `docs/osm-tag-accounting.md` containing all six sections with proper markdown tables, code citations (file:function:line), and explicit status labels for each tag.

Estimated size: ~800-1000 lines of markdown.



mkdir -p ~/lanterne/docs/02-architecture/nuremberg-osm-api
mv ~/Downloads/nuremberg-lanterne-osm-api-server.js ~/lanterne/docs/02-architecture/nuremberg-osm-api/server.js


---

## Source File: docs/02-architecture/arch-010-route_load_architecture.md

# Route Load Architecture

## Purpose

Document the current route-load architecture after the controller/telemetry refactor and the first round of route-load hardening.

This is not the final target architecture. It describes the system as it exists now so future optimization work does not have to rediscover the same boundaries.

## Current Shape

Route load is now split into four layers:

1. `route-analysis.ts`
   - owns the analysis engine
   - yields progressive stages
   - returns the final runtime safety result

2. `src/lib/route-load/`
   - owns route-load orchestration
   - translates analysis stages into telemetry, progress, and road-band state

3. `Index.tsx`
   - owns app-level lifecycle and mode transitions
   - starts route load
   - consumes controller snapshots/events
   - commits canonical analysis state to React

4. `RouteMap.tsx`
   - consumes route, score, road bands, and debug/admin overlays
   - no longer owns as much inline audit/admin rendering logic as before

## Route-Load Modules

### `RouteLoadController.ts`

Responsibilities:
- start/cancel route load
- consume progressive analysis stages
- emit snapshots and stage events
- maintain controller-owned road-band state

Important rule:
- the controller should not block route completion on non-user-critical background work

### `RouteLoadTelemetry.ts`

Responsibilities:
- convert raw analysis stages into a cumulative telemetry snapshot
- expose stable counts for UI and perf logging

Important rule:
- telemetry must remain cheap enough to sit on the hot path

### `RouteLoadProgressModel.ts`

Responsibilities:
- turn telemetry snapshots into smoothed loader presentation
- keep stage labels and percent coherent during long phases

Important rule:
- it can smooth presentation, but it must not lie about completed work

### `RoadBandController.ts`

Owns three road bands:
- `coreScoringRoads`
- `nearbyContextRoads`
- `expandedEditRoads`

Current meaning:
- `coreScoringRoads`: smallest route-relevant road universe for route review/paint
- `nearbyContextRoads`: broader nearby road universe for context/audit/edit support
- `expandedEditRoads`: wider lazy band for edit/detour operations

## Canonical vs Runtime Result

The system now distinguishes between:

- `SafetyResult`
  - canonical persisted/shared analysis shape
  - safe to cache, persist, and hold in ordinary app state

- `RuntimeSafetyResult`
  - analysis result plus ephemeral runtime-only road universes
  - includes fields like `matchedRoads` and `allFetchedRoads`

Current rule:
- canonical app state, route cache, and route persistence should prefer the stripped canonical shape
- large runtime road universes should not be treated as canonical

## Current Critical Path

Current route load still looks like this:

1. shell stage
2. corridor fetch
3. enrichment
4. analysis/scoring
5. final handoff into review UI

Recent hardening changed two important things:

- enrichment can begin earlier instead of waiting for every downstream handoff to finish
- post-pipeline reconciliation/persistence was moved off the user-critical path

## What Was Extracted From `RouteMap`

`RouteMap.tsx` still owns lifecycle and major map wiring, but several rendering slices were extracted:

- admin verification overlay
- raw nearby audit overlay
- candidate audit overlay
- turn audit overlay
- sequence audit overlay
- debug overlay diagnostics
- fragment overlay
- transition debug overlay
- route creation layer
- edit handles layer

This matters because future route-load work should keep map rendering consumption-oriented rather than pushing orchestration back into `RouteMap`.

## Paint Presentation Layer

Route paint now has an explicit controller:

- `src/lib/presentation/route-paint-controller.ts`

Current user-facing modes:
- `normal`
- `high_contrast`

This is the beginning of a route-paint presentation layer rather than one-off rendering switches inside the map.

## What Still Blocks Cold Loads

The main remaining cold-load bottleneck is infrastructure and payload shape, not just React wiring.

Current major pain points:
- corridor cold misses still depend on fragile proxy/public Overpass fallback behavior
- cold runs still build a broader road universe than first-paint route review actually needs
- context/admin/debug needs are not fully separated from first-answer route review

## Architectural Rules Going Forward

1. Route score, route paint, and hazard essentials should be available before admin/context bookkeeping.
2. Canonical persisted analysis must remain smaller than runtime diagnostic/context material.
3. `coreScoringRoads` should remain the first-paint road source.
4. Larger nearby/edit context should hydrate after first usable route review.
5. Admin/debug overlays should never silently re-enter the critical route-load path.

## Files To Read First

- [src/lib/route-analysis.ts](/Users/derekminner/lanterne/src/lib/route-analysis.ts)
- [src/lib/route-load/RouteLoadController.ts](/Users/derekminner/lanterne/src/lib/route-load/RouteLoadController.ts)
- [src/lib/route-load/RouteLoadTelemetry.ts](/Users/derekminner/lanterne/src/lib/route-load/RouteLoadTelemetry.ts)
- [src/lib/route-load/RouteLoadProgressModel.ts](/Users/derekminner/lanterne/src/lib/route-load/RouteLoadProgressModel.ts)
- [src/lib/route-load/RoadBandController.ts](/Users/derekminner/lanterne/src/lib/route-load/RoadBandController.ts)
- [src/lib/route-analysis-canonical.ts](/Users/derekminner/lanterne/src/lib/route-analysis-canonical.ts)
- [src/pages/Index.tsx](/Users/derekminner/lanterne/src/pages/Index.tsx)
- [src/components/RouteMap.tsx](/Users/derekminner/lanterne/src/components/RouteMap.tsx)


---

## Source File: docs/02-architecture/arch-011-route_load_optimization_direction.md

# Route Load Optimization Direction

## Purpose

Capture the next-step optimization model for route load so tuning work does not regress into patching symptoms around the wrong critical path.

## Core Observation

Cold-cache route load is still paying for too much too early.

The system currently answers more than the user asked for on first load:
- broad nearby road context
- edit-oriented context
- admin/debug diagnostics
- canonical identity/persistence support work

For a user opening a route, the immediate question is much smaller:

`Is this route safe, and where is it risky?`

## Target Model

Split route load into three layers.

### 1. Route Core

This is the first-answer payload and the only layer that should block first route review.

Should include:
- route geometry
- core scoring roads only
- safety score and grade
- route speed/risk segments for paint
- hazard essentials
- essential cues
- minimum traffic/speed evidence needed to support scoring

Should not include:
- broad nearby road universe
- edit/detour expansion universe
- admin audit/debug payloads

### 2. Route Context

This is background hydration for ordinary interaction after first paint.

Should include:
- nearby context roads
- expanded edit roads
- detour/edit support context
- richer nearby map context

This should start automatically after the user already has a working route and score. It should not wait for the user to press a heatmap or audit button.

### 3. Route Diagnostics

This is the heavy admin/developer layer.

Should include:
- candidate audit backing data
- sequence audit backing data
- raw nearby audit backing data
- transition diagnostics
- forensic debug
- boundary debug
- left-turn debug

This layer should be opt-in, lazy, and clearly non-critical.

## Cache Strategy Direction

The right long-term model is driftless cache-first behavior:

1. if route-level cache is warm enough, use it
2. if cache is missing, compute only route core
3. once route core is visible, hydrate context and diagnostics later
4. write each layer back into cache separately when possible

This implies a future separation between:
- route core cache
- route context cache
- route diagnostics cache

## Current Infrastructure Reality

The main cold-load bottleneck is still corridor fetch.

What was learned:
- Supabase `overpass-proxy` is a fragile cold-path dependency
- when that fails, fallback to public Overpass is slow and unstable
- browser-direct Nuremberg is not currently viable because the backend does not expose CORS

So the next real infrastructure improvement is not “try direct from browser again.”

It is one of:
- enable CORS on the direct roads backend
- or provide a thinner relay path that does not do heavy edge-function compute

## Practical Optimization Priorities

### Priority 1

Make first paint depend only on route core.

That means:
- score
- hazards
- route paint
- cue essentials

### Priority 2

Hydrate `nearbyContextRoads` in the background automatically after first review is visible.

Not:
- on heatmap button press
- not in the blocking analysis handoff

But:
- immediately after successful route-core load

### Priority 3

Keep all canonical identity reconciliation and turn-event persistence out of the first-review path.

These jobs are useful, but they are bookkeeping, not route-review prerequisites.

### Priority 4

Only load diagnostics when admin/debug features actually require them.

## Rules For Future Changes

Before adding anything to the route-load path, ask:

1. Does this change the score?
2. Does this change the immediate route paint?
3. Does this change the immediate hazard answer?

If the answer is no, it probably does not belong on the first-paint critical path.

## Intended Next Refactor

The likely next architectural module is a background context hydrator under `src/lib/route-load/`.

Likely responsibilities:
- hydrate `nearbyContextRoads` after first paint
- expand `expandedEditRoads` lazily and incrementally
- update road-band state without blocking review mode
- support cancellation if the user loads a different route

Probable name:
- `RouteContextHydrator.ts`

## What “Done Optimally For Now” Looks Like

For this moment in time, the route-load system is in a good place when:

- first review appears without waiting on broad context or diagnostics
- cold loads do not stall after `pipeline_complete`
- heatmap/edit mode usually feel ready because context hydrates in the background
- admin/dev tools remain available, but only when explicitly needed
- canonical persisted results stay much smaller than runtime analysis artifacts

## Related Files

- [src/lib/route-analysis.ts](/Users/derekminner/lanterne/src/lib/route-analysis.ts)
- [src/lib/corridor.ts](/Users/derekminner/lanterne/src/lib/corridor.ts)
- [src/lib/overpass-request.ts](/Users/derekminner/lanterne/src/lib/overpass-request.ts)
- [src/lib/route-load/RoadBandController.ts](/Users/derekminner/lanterne/src/lib/route-load/RoadBandController.ts)
- [src/lib/route-load/RouteLoadController.ts](/Users/derekminner/lanterne/src/lib/route-load/RouteLoadController.ts)
- [src/lib/route-analysis-canonical.ts](/Users/derekminner/lanterne/src/lib/route-analysis-canonical.ts)


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
- Cattle grids
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

## Source File: docs/02-architecture/analysis/anal-002-score_calculation_v2.md

# Lanterne Score Calculation — V3.1 Launch
2026-04-04

## Purpose

This document explains how rider-facing **scores** are produced from underlying analysis data.

It reflects the **V3.1-launch scoring engine** (`safety-constants.ts`, `safety-scoring.ts`).

---

## 1. Score Philosophy

Lanterne does **not** collapse all route intelligence into one number.

- A primary **Safety Score** (narrowly defined as motor-vehicle harm risk)
- Several **indices describing route reality**
- Environmental **conditions**
- A **hazard summary layer** (separate from Safety Score)

The Safety Score is the only true "headline score." Everything else is supporting analysis.

---

## 2. Safety Score Definition

> The relative expected motor-vehicle harm per mile for a bicyclist.

Normalized to a **0–100 scale** and mapped to letter grades (A+ through F).

This is NOT a kitchen-sink danger score. It **excludes** weather, surface, fatigue, rail hazards, time-of-day traffic, and all non-motor-vehicle factors.

---

## 3. Score Pipeline (V3.1-Launch)

```
Route geometry
    ↓
Segment risk modeling
  SliceMiles × (0.60 × SpeedFactor + 0.40 × TrafficFactor) × InfraFactor × ShoulderFactor
    ↓
Crossing risk contribution (bounded per-event model)
    ↓
Continuous RPM + Effective Crossing RPM (capped at 40% of total)
    ↓
Logistic normalization: 100 / (1 + e^(1.4 × (RawRPM - 2.5)))
    ↓
Final Safety Score (0–100)
    ↓
Letter grade + confidence level
```

---

## 4. Speed Factor (60% weight)

Piecewise-linear breakpoint table:

| Speed (mph) | Factor |
|-------------|--------|
| ≤ 20        | 0.50   |
| 25          | 1.00   |
| 30          | 1.60   |
| 35          | 2.30   |
| 40          | 3.10   |
| 45          | 4.00   |
| 50          | 5.00   |
| 55+         | 6.20   |

Linear interpolation between breakpoints.

---

## 5. Traffic Factor (40% weight)

AADT-per-lane tiered with interpolation:

| AADT/lane    | Factor |
|--------------|--------|
| < 2,000      | 0.60   |
| 2,000–3,999  | 1.00   |
| 4,000–7,999  | 1.50   |
| 8,000–11,999 | 2.00   |
| 12,000–15,999| 2.50   |
| 16,000+      | 3.00   |

### Traffic Fallback Ladder

1. Official AADT per lane (highest confidence)
2. Official total AADT + known lane count → compute per-lane
3. Official total AADT + inferred lane count (default: 2)
4. Inferred AADT total from nearby values by highway type
5. Generic road-class / highway-type traffic proxy
6. Unknown → factor 1.10

---

## 6. Infrastructure Mitigation (InfraFactor)

Multiplicative reduction:

| Facility         | Multiplier |
|------------------|------------|
| Protected track  | 0.50       |
| Buffered lane    | 0.68       |
| Painted lane     | 0.82       |
| Shared lane      | 1.00       |
| No facility      | 1.00       |

Sharrows do NOT count as infrastructure.

---

## 7. Shoulder Mitigation (ShoulderFactor)

Applies ONLY when no dedicated bike facility AND speed ≥ 30 mph:

| Shoulder       | Multiplier |
|----------------|------------|
| Wide (≥ 8 ft)  | 0.78       |
| Usable (2–8 ft)| 0.88       |
| Sub-usable (<2 ft) | 1.00   |
| None           | 1.00       |

---

## 8. Crossing Risk Contribution

Replaces the V2/V3.0 left-turn penalty.

```
CrossingEventContribution = min(E_cap, E0 × √(SpeedFactor × TrafficFactor) × WidthFactor × ControlFactor × MovementFactor)
```

- E0 = 0.05, E_cap = 0.75
- Width: 1.00 (1–2 lanes), 1.25 (3–4), 1.60 (5–6), 2.00 (7+)
- Control: signalized 1.00, stop 1.05, unknown 1.10
- Movement: straight 1.00, right_merge 1.05, left_across 1.20, unknown 1.10

### Crossing Eligibility Gate

A crossing is score-eligible if ANY is true:
- Speed ≥ 30 mph AND effective AADT/lane ≥ 2,000
- Lanes ≥ 3
- Left-across movement AND (speed ≥ 30 OR effective AADT/lane ≥ 2,000)

**Important:** The gate uses the full traffic fallback ladder, not just raw official AADT. This prevents under-scoring obvious crossings in data-poor areas.

### Policy Assumptions in the Crossing Gate

Two design decisions in the gate are **policy-derived**, not benchmark coefficients:

**A. `left_across` + speed ≥ 30 can make a crossing score-bearing even without strong traffic evidence.**
Rationale: A left turn across oncoming traffic on a 30+ mph road is inherently risky regardless of measured volume. This is a product safety judgment, not a literature-derived threshold.

**B. `classProxyFactor → approximateAadtPerLane` is a policy-derived approximation.**
When no official AADT data exists, the gate reverse-engineers an approximate AADT-per-lane from the highway-class traffic proxy factor. This is a best-effort estimate to avoid ignoring obvious major crossings in data-poor areas. It is NOT a benchmark coefficient — it is a policy decision to use available evidence rather than default to zero.

### Crossing Cap

Total crossing risk is capped: `EffectiveCrossingRPM ≤ ContinuousRPM × 0.6667`

This ensures crossings never exceed 40% of total raw canonical risk.

---

## 9. What is NOT in the Canonical Safety Score

- Rail crossings (hazard layer only)
- Bridge hazards (hazard layer only)
- Time-of-day traffic adjustments (contextual display only)
- Critical stretch / worst-1km penalties (report-only, does NOT cap score)
- Weather, surface, fatigue
- Left-turn penalties (replaced by crossing risk contribution)

---

## 10. Critical Stretch (Report-Only)

Worst 1km RPM is computed and reported for transparency but does **NOT** modify the canonical Safety Score.

| Worst 1km RPM | Suggested Cap |
|----------------|---------------|
| < 2.5          | no cap        |
| 2.5–3.5        | 89            |
| 3.5–4.5        | 79            |
| 4.5–5.5        | 69            |
| ≥ 5.5          | 59            |

---

## 11. Confidence Output

Each score includes confidence based on data coverage:

| Coverage | Confidence |
|----------|------------|
| ≥ 80%    | high       |
| 50–80%   | medium     |
| < 50%    | low        |

---

## 12. Hazard Layer (separate from Safety Score)

Rail crossings, bridge hazards, cattle grids, and other micro-hazards are detected and displayed but do NOT affect the headline Safety Score.

---

## 13. Safe Path Baseline

Separated paths (bike paths, multi-use trails): 0.05 risk/mile baseline (not zero).

---

## 14. Letter Grades

| Grade | Score Range |
|-------|-------------|
| A+    | 97–100      |
| A     | 93–96       |
| A-    | 90–92       |
| B+    | 87–89       |
| B     | 83–86       |
| B-    | 80–82       |
| C+    | 77–79       |
| C     | 73–76       |
| C-    | 70–72       |
| D     | 60–69       |
| F     | < 60        |

---

## 15. Model Version

`v3.1-launch` — defined in `SCORE_MODEL_VERSION` constant.

---

## 16. Design Rule

Safety Score must remain:
- Narrowly defined (motor-vehicle harm only)
- Explainable
- Grounded in traffic safety research
- Resistant to feature creep

A dangerous road in light rain should not score better than the same road in sunshine. Weather is a different question with a different answer.


---

## Source File: docs/02-architecture/analysis/anal-002-score_calculation_v3_final.md

# Lanterne Score Calculation
2026-04-04

## Purpose

This document explains how rider-facing scores are produced from underlying analysis data.

It reflects the intended launch V3 canonical Safety Score model.

The goal is a score that is:
- narrow
- explainable
- benchmark-shaped
- transparent about launch policy choices

---

## 1. Score Philosophy

Lanterne does **not** collapse all route intelligence into one number.

Instead it uses:
- a primary **Safety Score**
- route reality and condition layers outside the headline score
- report and explainer layers that help riders understand context without diluting the canonical score

The Safety Score is the only headline score today.

---

## 2. Safety Score definition

Safety Score represents:

> the relative expected harm from a bicyclist being struck by a motor vehicle

The score is normalized to a **0–100** scale and mapped to letter grades.

It is **not**:
- a crash probability
- a weather score
- a fatigue score
- a hotspot penalty score

---

## 3. Score pipeline

```text
Route geometry
    ↓
Internal truth slices
    ↓
Continuous slice risk modeling
    ↓
Crossing event contribution modeling
    ↓
Route rollup (continuous + bounded crossing contribution)
    ↓
Logistic normalization
    ↓
Safety Score (0–100)
    ↓
Letter grade
```

---

## 4. Continuous slice risk

For each internal slice:

```text
ContinuousSliceRisk
= SliceMiles
× (0.60 × SpeedFactor + 0.40 × TrafficFactor)
× InfraFactor
× ShoulderFactor
```

### 4.1 SpeedFactor
Speed is non-linear.
Launch table:
- ≤20 mph → 0.50
- 25 mph → 1.00
- 30 mph → 1.60
- 35 mph → 2.30
- 40 mph → 3.10
- 45 mph → 4.00
- 50 mph → 5.00
- 55+ mph → 6.20

This is benchmark-shaped and launch-compressed for stability.

### 4.2 TrafficFactor
Traffic uses AADT-per-lane where available.
Launch table:
- <2,000/day/lane → 0.60
- 2,000–3,999/day/lane → 1.00
- 4,000–7,999/day/lane → 1.50
- 8,000–11,999/day/lane → 2.00
- 12,000–15,999/day/lane → 2.50
- 16,000+/day/lane → 3.00

### 4.3 InfraFactor
- protected / fully separated → 0.50
- buffered lane → 0.68
- painted lane → 0.82
- no dedicated facility → 1.00

### 4.4 ShoulderFactor
Shoulder only applies when:
- no dedicated bike facility
- speed ≥ 30 mph

Shoulder classes:
- sub-usable: < 2.0 ft / 0.6 m
- usable: 2.0 ft to < 8.0 ft / 0.6 m to < 2.4 m
- wide: ≥ 8.0 ft / 2.4 m

Launch values:
- sub-usable / none → 1.00
- usable → 0.88
- wide → 0.78

---

## 5. Crossing risk contribution

A crossing event is a score-bearing conflict point where the route requires more than simply continuing along the road.

### 5.1 Variables
- **E0** = base crossing risk contribution before context is applied
- **E_cap** = maximum contribution any one crossing may add
- **SpeedFactor_raw** = raw benchmark speed ratio before launch compression
- **TrafficFactor_raw** = raw benchmark traffic ratio
- **WidthFactor** = width / lanes-crossed multiplier
- **ControlFactor** = signalized / stop-controlled / unknown modifier
- **MovementFactor** = straight / right-merge / left-across / unknown modifier

### 5.2 Formula

```text
CrossingEventContribution_i
= min(E_cap,
      E0 × (SpeedFactor_raw × TrafficFactor_raw)^0.5
         × WidthFactor × ControlFactor × MovementFactor)
```

Launch constants:
- E0 = 0.05 crossing risk points
- E_cap = 0.75 crossing risk points

### 5.3 Eligibility
A crossing event enters score math when at least one is true:
- crossed/entered road speed ≥ 30 mph and AADT per lane ≥ 2,000/day/lane
- lanes crossed ≥ 3
- left across traffic on a road that is either speed ≥ 30 mph or AADT per lane ≥ 2,000/day/lane

Signalized and stop-controlled crossings may still be counted and displayed even when not all of them are score-bearing.

### 5.4 Factor tables
#### WidthFactor
- 1–2 lanes → 1.00
- 3–4 lanes → 1.25
- 5–6 lanes → 1.60
- 7+ lanes → 2.00

#### ControlFactor
- signalized → 1.00
- stop-controlled → 1.05
- unknown → 1.10

#### MovementFactor
- straight-across → 1.00
- right / merge → 1.05
- left across traffic → 1.20
- unknown → 1.10

---

## 6. Route rollup

```text
TotalContinuousRisk = Σ ContinuousSliceRisk
ContinuousRPM = TotalContinuousRisk / RouteMiles

TotalCrossingRiskContribution = Σ CrossingEventContribution_i
RawCrossingRiskContributionPerMile = TotalCrossingRiskContribution / RouteMiles

EffectiveCrossingRiskContributionPerMile
= min(RawCrossingRiskContributionPerMile, ContinuousRPM × 0.6667)

RawRPM = ContinuousRPM + EffectiveCrossingRiskContributionPerMile

SafetyScore = 100 / (1 + e^(1.4 × (RawRPM - 2.5)))
```

This route-level crossing cap ensures crossing contribution cannot exceed 40% of raw canonical route risk at launch.

---

## 7. What is out of canonical score math

The following are not part of canonical score math:
- critical stretch / hotspot penalties
- time-of-day traffic bell-curving
- rail / grate / cattle-guard / non-motor-vehicle hazard penalties
- weather / light conditions

These may still appear in report and explanation layers.

---

## 8. Report-only layers

### 8.1 Critical stretch
Critical stretch is report-only.
It may appear in the analysis drawer and route explanation, but it does not modify the canonical score.

### 8.2 Contextual traffic
If Lanterne shows public traffic as daily counts, it should also show rider-readable equivalents:
- cars per hour average
- cars per minute average

Time-of-day contextual interpretation may appear in cue sheets and explainers, but not in canonical score math.

---

## 9. Design rule

Safety Score must remain:
- explainable
- grounded in traffic safety research
- resistant to feature creep
- honest about what is benchmark-derived vs launch policy


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
| ADR-023 | Predicted vs Observed Condition Layers | Accepted | DS-016 |
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
| DS-015 | Safety Scoring Model | ADR-041, ADR-043, DS-017 | Canonical |
| DS-016 | Predicted vs Observed Conditions Schema | ADR-023 | **Reserved — not yet written** |
| DS-017 | Truth Resolution and Propagation | ADR-042, ADR-043 | Draft |
| DS-018 | Viewport Overlay Hydration and Client Budget | ADR-027, ADR-029, ADR-030 | Draft |
| DS-022 | Speed Prior and Area-Baseline Policy | ADR-042, DS-017 | Draft |
| DS-023 | Government Feed Ingestion and Triage | ADR-042, DS-013, DS-017 | Draft |
| DS-024 | Parallel Bike Facility Capture and Corridor Ownership | ADR-019, ADR-020, ADR-026, DS-008, DS-017 | Draft |
| DS-025 | Transition Candidate, Claim, and Projection Spec | ADR-005, ADR-019, ADR-042 | Draft |
| DS-026 | Keyboard Shortcuts and Map Input | ADR-027, ADR-029, ADR-030 | Draft |
| DS-027 | POI Ingestion, Selection, and Cluster Interaction | ADR-017, ADR-019, ADR-027, DS-008, DS-018 | Draft |
| DS-028 | Hazard Ingestion, Normalization, and Presentation | ADR-017, ADR-019, ADR-020, ADR-027, DS-008 | Draft |
| DS-029 | Provenance, Precedence, Confidence, and Traceability | ADR-042, ADR-043, DS-015, DS-017, DS-022 | Draft |
| DS-030 | Route Analysis Contract | DS-017, DS-024, DS-025, DS-028, DS-029 | Draft |

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
| DS-016 — Predicted vs Observed Conditions Schema | When conditions persistence work begins |
| DS-017 — Field Note Schema | When field notes implementation begins (post-alpha) |
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

## Source File: docs/02-architecture/design/ds-001-route_intelligence_pipeline_spec.md

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

## Source File: docs/02-architecture/design/ds-002-analysis_rollup_spec.md

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

## Source File: docs/02-architecture/design/ds-003-ride_timeline_model_spec.md

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

## Source File: docs/02-architecture/design/ds-004-osm_variable_registry_spec.md

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

## Source File: docs/02-architecture/design/ds-005-canonical_route_schema_spec.md

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

## Source File: docs/02-architecture/design/ds-006-route_canonicalization_spec.md

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

## Source File: docs/02-architecture/design/ds-007-route_slice_generation_spec.md

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

## Source File: docs/02-architecture/design/ds-008-route_corridor_model_spec.md

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

## Source File: docs/02-architecture/design/ds-009-route_corridor_fingerprint_spec.md

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

## Source File: docs/02-architecture/design/ds-010-slice_analysis_cache_spec.md

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

## Source File: docs/02-architecture/design/ds-011-ride_time_situational_awareness_interface_spec.md

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

## Source File: docs/02-architecture/design/ds-012-ride_computer_tile_system_spec.md

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

## Source File: docs/02-architecture/design/ds-013-comparative_traffic_context_schema_spec.md

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

## Source File: docs/02-architecture/design/ds-014-route_expedition_state_and_windowed_analysis_spec.md

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

## Source File: docs/02-architecture/design/ds-015-safety_scoring_model-explained.md

# DS-015 - Safety Scoring Methodology, Explained

**Status:** Community-facing methodology manifesto  
**Companion to:** `ds-015-safety_scoring_model.md`  
**Updated:** 2026-04-25

This is the plain-English version of Lanterne's safety scoring methodology.

The canonical spec is where the exact contract lives. This document is the why. It is written for cyclists, route makers, randonneurs, brevet organizers, planners, skeptics, and anyone who has ever looked at a route on a map and wondered whether the line was telling the whole truth.

## Why I Built This

I did not come to this as a transportation engineer sitting safely behind a model.

I came to it as a rider.

I have been the guy in a reflective jacket on a dark road wondering if the next set of headlights was going to give me room. I have been on shoulderless 55 mph roads with semis and dump trucks moving past close enough to change the emotional temperature of the ride. I have ridden into weather I underestimated, cold I did not respect enough, crosswinds that made forward motion feel like negotiation, and rural roads where the route looked simple on paper but felt very different once I was actually there.

I have also had the kind of rides that rearrange you. The ones where a work trip turns into a 200K. The ones where a state line becomes a personal milestone. The ones where a route is not just distance, but memory, marriage, recovery, grief, fear, pride, and whatever else decides to climb into the saddle bag with you.

That is the context for Lanterne.

Not fear.

Not risk avoidance.

Not an attempt to sand the adventure out of cycling.

The point is the opposite: I want riders to keep choosing hard things, but to choose them with better information.

There is a difference between accepting risk and being surprised by it.

There is a difference between saying, "I know this road is ugly, and I am choosing it anyway," and finding out at mile 47, in the dark, that the official-looking route has dumped you onto a no-shoulder truck corridor.

Lanterne exists for that gap.

## The Core Belief

Cyclists do not need a fake promise of safety.

We need context.

We need to know where the route is asking us to share space with fast traffic, where crossings get ugly, where a bike lane actually helps, where a shoulder changes the equation, where a separated path really is separated, and where a route's average comfort is hiding one bad connector that deserves attention.

Risk is not a moral judgment.

Risk is not a scolding.

Risk is not the app saying, "Do not ride this."

Risk is information.

The rider still decides.

## What The Score Is

Lanterne's safety model is intentionally narrow.

It estimates relative motor-vehicle strike risk for a bicyclist.

That means the score is about the interaction between a person on a bike and motor vehicles. It is not trying to be a complete outdoor danger score. It does not claim to know whether you are tired, overtrained, underdressed, dehydrated, descending beyond your skill, riding at midnight, fighting a headwind, or emotionally making questionable choices because a state line is calling your name.

Those things matter. Any rider knows they matter.

But if we put every real thing into one number, the number stops meaning anything.

So the model stays disciplined:

```text
motor-vehicle strike risk, weighted by likely severity
```

That narrowness is not a weakness. It is what makes the score explainable.

## The Whole Model In One Line

The methodology is built on one idea:

```text
Risk = Likelihood x Severity
```

That is the entire philosophy.

The model does not throw speed, traffic, bike lanes, shoulders, crossings, and curves into a blender and call the smoothie "safety."

It asks two different questions:

```text
How likely is a bad interaction here?
If it happens, how severe is it likely to be?
```

Then it applies that logic in two domains:

```text
Total Risk = Road Risk + Crossing Risk
Risk / Mile = Total Risk / Route Miles
```

Road risk and crossing risk are not competing truth systems. They are two places where the same likelihood-versus-severity logic applies.

## Road Risk

Road risk is the sustained exposure part of the model.

This is the part that understands that ten miles on a fast, busy, shoulderless road is not the same as ten miles on a quiet local road, even if both are technically "roads."

For road slices:

```text
Road Risk = Road Likelihood x Road Speed Severity
```

Road likelihood is shaped by:

- traffic
- bike facility
- shoulder
- sustained curvature
- distance spent in that environment

Road severity is shaped by:

- speed

That distinction matters.

Traffic mostly answers: how many motor-vehicle interactions am I exposed to?

Speed mostly answers: if one of those interactions goes wrong, how bad is it likely to be?

A no-shoulder road at 25 mph and a no-shoulder road at 55 mph are not the same moral universe. The model should not pretend they are.

## Crossing Risk

Crossing risk is the discrete conflict-event part of the model.

A crossing is not just "more road." It is a different kind of problem. You may be crossing a stream of traffic, joining a faster road, making a left across lanes, exiting a path, or threading through a wide signalized intersection where the map only showed one innocent little node.

For crossings:

```text
Crossing Risk = Crossing Likelihood x Crossed-Road Speed Severity
```

Crossing likelihood is shaped by:

- crossed-road traffic
- lanes / width
- control
- movement

Crossing severity is shaped by:

- speed of the crossed or entered road

This is why a path can be genuinely safe for miles and still have one crossing that deserves attention. A separated trail should not accumulate fake road risk, but the place where that trail crosses a high-speed arterial still matters.

## Why Speed Carries Severity

Speed is the most defensible launch severity variable.

That does not mean speed is the only thing that matters in the real world. A distracted driver, a loaded dump truck, a bad sightline, or a bad mood can all change the story.

But for a national-scale route intelligence tool, speed has three advantages:

- It is strongly connected to outcome severity.
- It is commonly available or inferable.
- Riders understand it immediately.

When I see 55 mph with no shoulder, I do not need a PhD to know what that means in my body. The model should reflect that same common-sense truth, but in a traceable way.

## Why Traffic Cannot Be Flattened Too Early

Traffic is the exposure backbone.

If an official DOT or HPMS source says a road carries a specific AADT, Lanterne should use that number. Not a vague bucket. Not a vibes-based road class. The actual value.

A road with 18,000 vehicles per lane per day is not merely "a little busier" than a road with 3,000. For a cyclist, that difference is lived as repeated exposure.

At ordinary and busy volumes, each additional vehicle still matters. Eventually, traffic becomes more like a continuous stream: platoons, queues, fewer independent gaps, and less new marginal risk from each additional car.  So the model bends the traffic curve only after giving traffic the weight it deserves.

## Why Bike Facilities And Shoulders Matter

Bike lanes and shoulders do not make risk disappear.

They change the likelihood side of the equation by giving the cyclist more operating space.

That is why the model treats facilities and shoulders as likelihood reducers, not severity reducers. A painted bike lane does not make a 45 mph impact gentle. It may reduce the chance of the interaction happening in the first place.

The launch facility factors are intentionally bounded:

| Facility | Likelihood factor |
| --- | ---: |
| fully separated / protected on-road facility | 0.50 |
| buffered bike lane | 0.75 |
| painted bike lane | 0.80 |
| no dedicated bike facility | 1.00 |

And shoulder:

| Shoulder | Likelihood factor |
| --- | ---: |
| sub-usable / none | 1.00 |
| usable shoulder | 0.85 |
| wide shoulder | 0.80 |

This is not saying every painted lane is good or every shoulder is clean. Anyone who rides knows a shoulder can be full of glass, rumble strips, gravel, dead animals, storm grates, or whatever came loose from the last landscaping trailer.

The score is not claiming perfection. It is saying operating space matters, and when the data says that space exists, the model should acknowledge it while still preserving confidence and provenance.

## Why Paths And MUPs Are Special

A separated path or MUP should carry zero continuous-road risk.

Not "very low road risk."

Zero road risk.

If you are not riding in the motor-vehicle stream, the model should not invent motor-vehicle stream exposure.

But that does not mean the whole route is magically risk-free. Path-road crossings still count. Driveway crossings can count. Joins and exits can count. The score should treat a trail as a trail and a crossing as a crossing.

That is the whole point of having a model with domains instead of one mushy average.

## Why We Stopped Pretending A 0-100 Score Was Truth

A 0-100 score looks clean.

It also lies by implication.

It makes the result feel more exact than the evidence deserves. It tempts people to ask whether an 82 is meaningfully different from an 84. It hides the comparison universe. It turns a model output into a grade-school emotional reaction.

So the canonical outputs are simpler:

```text
Total Risk
Risk / Mile
```

Total Risk tells you how much motor-vehicle risk the route accumulates.

Risk / Mile tells you what kind of route it is, mile for mile.

A 200-mile route can accumulate more total risk than a 40-mile route and still be better mile-for-mile. Both truths matter.

## Rank And Grade Are Comparisons, Not Truth

I still believe riders need an interpretation layer.

Most people do not want to stare at raw risk points and reverse-engineer whether a route is a good idea. So Lanterne can show rank and curved grade.

But those are projections.

Rank means:

```text
How does this route compare to routes in this selected network?
```

Curved grade means:

```text
Where does this route land on an A-F curve within that selected network?
```

A route can be an A among one family of routes and a C among another. That is not a bug. That is honesty. Every route belongs to more than one universe.

## Why Provenance Matters

If the score uses a value, Lanterne needs to know where that value came from.

Was speed directly tagged?

Was traffic imported from DOT?

Was lane count inferred?

Was a shoulder observed, imported, or guessed from weak context?

Was a crossing movement geometry-derived or ambiguous?

This matters because missing data should not quietly become danger, and guessed data should not masquerade as truth.

Confidence is not garnish. It is part of the result.

A route can be risky with high confidence. It can be risky with low confidence. It can appear benign because the data is weak, and that is exactly where the system has to be honest rather than comforting.

## Why Critical Stretches Are Report-Only

Route averages can hide nasty little sections.

Any long-distance rider knows this. You can have 98 miles of pleasant riding and two miles that define the emotional memory of the day.

Lanterne should surface those sections.

It should show the worst rolling stretch. It should show the top road contributors. It should show the top crossings. It should tell the rider, "This is where the route concentrates its ugliness."

But it should not secretly mutate the score.

The canonical score should remain additive and traceable:

```text
Total Risk = Road Risk + Crossing Risk
```

Critical stretch is an explanation layer. It is a detour prompt. It is a rider-respect tool. It is not a hidden punishment.

## Why The Map Has Two Paint Systems

There are two different map questions.

The analyzed route asks:

```text
What is the canonical risk along this route?
```

The viewport asks:

```text
What kind of roads are around me?
```

Those are not the same problem.

A route has direction, continuity, turns, crossings, matched context, chosen inputs, confidence, and provenance. It can use the canonical score trace.

The random roads visible in a map viewport do not have all of that. Scoring every visible road as if it were a full route would be both semantically wrong and computationally abusive. The map would get slower, and the answer would pretend to be more precise than it is.

So Lanterne has:

- Route Risk Paint for analyzed routes.
- Road-Stress Overlay for visible off-route roads.

That is not split-brain. That is two named contracts for two different jobs.

## What This Methodology Promises

Lanterne will not promise that a route is safe.

No tool should.

The promise is narrower and more useful:

- We will show the risk we can defend.
- We will separate likelihood from severity.
- We will distinguish road exposure from crossing exposure.
- We will use exact traffic data when we have it.
- We will not pretend unknown data is certain.
- We will not bury route risk inside a fake-precise 0-100 score.
- We will show where the route concentrates its ugliness.
- We will let riders compare routes against the right universe.
- We will keep the math traceable enough that a skeptical cyclist can challenge it.

## What This Methodology Does Not Promise

It does not know your skill.

It does not know whether you are riding at noon or 2 a.m.

It does not know whether the shoulder is covered in glass today.

It does not know whether the driver coming up behind you is distracted, angry, impaired, or kind.

It does not know whether you are rested enough to make good decisions.

It does not replace judgment.

It is a flashlight, not a force field.

## The Community Ask

The cycling community has always been part data, part folklore.

We share routes. We warn each other about bridges, dogs, coal rollers, bad shoulders, sketchy towns, beautiful roads, gas stations with good food, gas stations with no bathrooms, and the one connector that looks fine on a map but feels like a coin flip in real life.

Lanterne is an attempt to make some of that wisdom structured.

Not to replace the stories.

To support them.

The best version of this tool is not one where the computer gets to be smug. It is one where riders can look at a route and say:

```text
That matches what I know.
That misses something important.
That road changed.
That shoulder is fake.
That crossing deserves a warning.
That path is safer than the model thinks.
That route is hard, but I understand the risk now.
```

That feedback loop is the point.

## The Ethic

I still want adventure.

I still want absurd logistics, state lines, night starts, gas-station calories, headwinds, bad ideas that turn into good stories, and the particular kind of aliveness that only shows up after several hours of voluntary discomfort.

I just no longer believe ignorance makes the adventure purer.

If anything, better information makes the choice more meaningful.

Ride the hard route if you want.

Ride the ugly connector if it is worth it.

Take the shoulderless road if the alternative is worse and you understand what you are accepting.

But do not let a thin blue line on a map hide the stakes from you.

Cyclists deserve better than that.

Lanterne is my attempt to give something back: a tool built from the saddle, shaped by lived risk, disciplined by evidence, and offered to riders who still want to go far without being lied to by the map.

## Rider Context

This methodology was shaped by real ride experience, including public ride writeups from the AmericanXplorer project:

- [Midwest Mulligan](https://derekminner.substack.com/p/midwest-mulligan)
- [Kentucky Roulette](https://derekminner.substack.com/p/kentucky-roulette)
- [Ride of My Life: Part 2](https://derekminner.substack.com/p/ride-of-my-life-part-2)
- [I Wanna Go-ooo To Maine](https://derekminner.substack.com/p/i-wanna-go-ooo-to-maine)
- [Pain, Poke, and Panorama](https://derekminner.substack.com/p/pain-poke-and-panorama)
- [Iron Like A (Mountain) Lion in Zion](https://derekminner.substack.com/p/iron-like-a-mountain-lion-in-zion)

Those stories are not the model. The model still has to stand on defensible methodology. But they explain why the model exists and why it needs to be honest.


---

## Source File: docs/02-architecture/design/ds-015-safety_scoring_model.md

# DS-015 - Safety Scoring Model 5.5

**Status:** Draft canonical replacement candidate  
**Date:** 2026-04-25  
**Filename:** `ds-015-safety_scoring_model.md`

## Purpose

This artifact defines Lanterne's safety scoring contract as a complete, standalone model.

The model estimates **relative motor-vehicle strike risk for a bicyclist**, weighted by the severity of the likely outcome if a strike occurs. It is built for pre-ride route intelligence, route comparison, score tracing, and cached corpus analysis. It is not a crash probability model, a weather model, a rider skill model, a fatigue model, a policing model, or a complete all-hazards route danger model.

The canonical score objects are:

```text
Total Route Risk = sum(continuous-road risk points) + sum(crossing-event risk points)
Route Risk Per Mile = Total Route Risk / Route Miles
```

Those two objects are the stable scoring layer. Rank and grade are projections on top of them, not replacements for them.

## 1. Architectural Decision

Lanterne uses one canonical route-risk model and many possible interpretation layers.

The canonical model answers:

```text
What does Lanterne believe this route accumulates in motor-vehicle strike risk?
```

The comparative layer answers:

```text
How does this route compare with a selected route network?
```

Those are different questions. The system must not collapse them into a single 0-100 shell. A 0-100 number implies false precision, hides the chosen comparison universe, and makes one display value pretend to be both absolute model output and relative rider-facing summary.

The canonical route artifact must therefore preserve:

- `totalRouteRisk`
- `routeRiskPerMile`
- `roadRiskTotal`
- `crossingRiskTotal`
- `routeMiles`
- `scoreTrace`
- `confidenceSummary`
- `provenanceSummary`
- `modelVersion`
- evidence and cache version metadata

Rank and grade may be computed later from `routeRiskPerMile`, but they are projections that must carry their own network metadata.

Projection layers may not compute alternate score truth. They may only interpret the canonical route artifact for a named network, universe, UI context, or cache version.

## 2. Model Scope

The scoring scope is narrow by design:

- Include motor-vehicle exposure along the route.
- Include crossings, joins, exits, and turn conflicts involving motor-vehicle roads.
- Include speed because it is the strongest nationally available proxy for serious outcome severity.
- Include traffic because vehicle volume is the exposure backbone.
- Include operating space because facilities and shoulders reduce interaction opportunity.
- Include horizontal curvature because it is benchmark-backed and measurable from geometry.
- Exclude factors that are real but not launch-defensible as canonical math.

Out of canonical score at launch:

- weather, wind, temperature, daylight, darkness, and seasonal surface state
- fatigue, rider skill, group size, and rider compliance
- remoteness, services, rescue access, and route difficulty
- railroad flangeway risk, metal bridges, gravel, descents, potholes, and other non-motor-vehicle hazards
- driveway density, curb activity, parking, lane width, lighting, heavy-vehicle share, and state-level crash culture until defensible data exists
- time-of-day traffic storytelling
- critical-stretch warnings as score modifiers

Those may appear in companion layers, warnings, cue context, or future models. They do not belong in this narrow launch scoring contract unless promoted through a versioned model change.

## 3. Evidence Grounding

The model is evidence-informed, not an engineering-grade crash prediction system.

The governing evidence shape is:

- Road safety practice commonly separates exposure, likelihood, context adjustments, and outcome severity rather than blending everything into one weighted soup.
- Bicycle safety benchmark families support speed and traffic as core inputs.
- Published speed-factor tables support a steep, non-linear speed curve.
- AADT-per-lane benchmark families provide defensible traffic breakpoints, but Lanterne must extend the high-volume tail because real roads exceed a simple 18,000 AADT ceiling.
- Intersection and crossing research supports width, control, and movement as meaningful factors, but not with nationally stable exact coefficients.
- U.S. fatality and severity patterns support keeping sustained road exposure central while still scoring crossing conflicts explicitly.

Source families used by this contract:

- HSM / HSM2 / NCHRP bicycle modeling family for factorized roadway and intersection structure.
- National Academies / TRB `Pedestrian and Bicycle Safety Performance Functions`, Table 156, attributed to iRAP 2013l, for the speed-factor shape.
- NCHRP 17-84 / HSM2 AADT-per-lane breakpoint family for traffic exposure shape.
- FHWA Bicycle Intersection Safety Index for movement, lanes-to-cross, and control relevance.
- FHWA bicycle treatment CMF direction for operating-space mitigation.
- NHTSA fatality location summaries and NTSB midblock-severity findings for keeping sustained exposure central.
- PBCAT-style crash typing for rider-readable movement categories.

Archived DS-015 V4 and V5 material is not active specification text. The source rationale, benchmark logic, and policy justification from those drafts are consolidated into this artifact so future work does not have to chase archived files to understand why a policy decision exists.

The model uses these sources for structure, shape, and defensible ordering. It does not claim that Lanterne's launch constants are externally published crash-probability coefficients.

## 4. Canonical Route Artifact

A completed route analysis produces one canonical score-bearing artifact.

Minimum required fields:

| Field | Meaning | Required use |
| --- | --- | --- |
| `modelVersion` | Safety model version, e.g. `ds-015-5.5` | cache key, trace, rank eligibility |
| `routeMiles` | Route length in miles | denominator for risk per mile |
| `totalRouteRisk` | Sum of road and crossing risk points | canonical accumulated score |
| `routeRiskPerMile` | `totalRouteRisk / routeMiles` | canonical comparison basis |
| `roadRiskTotal` | Sum of continuous-road risk points | receipt and diagnostics |
| `crossingRiskTotal` | Sum of crossing-event risk points | receipt and diagnostics |
| `crossingRiskShare` | `crossingRiskTotal / totalRouteRisk` when total is positive | corpus sanity diagnostic |
| `scoreTrace` | Road-slice and crossing-event records | route paint, receipts, audit, cache |
| `confidenceSummary` | Route, road, traffic, crossing, and evidence confidence | caveats and cache status |
| `provenanceSummary` | Evidence mix used by score-bearing fields | receipts and audit |
| `evidenceSnapshotId` | Version of imported / resolved evidence | cache invalidation |
| `routeGeometryHash` | Stable geometry identity | cache invalidation |

No implementation may persist only grade, rank, or a legacy score shell and call that a complete safety result.

## 5. Score Trace Contract

The score trace is not debug debris. It is the contract that lets the route line, scorecard, method card, receipt cards, admin views, and cache agree.

### 5.1 Continuous-Road Unit

Each road-slice trace record must include:

- slice id and geometry reference
- slice miles
- selected speed and its provenance / confidence
- selected AADT per lane and its provenance / confidence
- selected lane count basis if AADT was converted from total AADT
- curvature class and derivation
- facility class and derivation
- shoulder class and derivation
- all factor values used in math
- `roadLikelihood`
- `roadRisk`
- any excluded or deferred factors that were visible but not score-bearing

### 5.2 Crossing-Event Unit

Each crossing-event trace record must include:

- event id and geometry reference
- event type: crossing, join, exit, left-across, right / merge, straight-through, path-road crossing, driveway / access-road crossing
- eligibility reason
- crossed or entered road identity
- selected crossed-road speed and provenance / confidence
- selected crossed-road AADT per lane and provenance / confidence
- lanes crossed / width proxy and provenance / confidence
- control type and provenance / confidence
- movement type and provenance / confidence
- all factor values used in math
- `crossingLikelihood`
- `crossingRisk`

If a crossing is shown to riders but not score-bearing, the trace must say why.

## 6. Truth Resolution, Provenance, And Confidence

Every score-bearing input has three separate concepts:

```text
chosen value
provenance
confidence
```

The chosen value feeds score math. Provenance explains where that value came from. Confidence explains how much trust the system has in that value. Confidence must not directly rescale canonical risk.

### 6.1 Provenance Classes

Required provenance classes:

| Class | Meaning |
| --- | --- |
| `observed` | Direct field measurement or validated first-hand evidence |
| `official_imported` | Authoritative agency source such as DOT / HPMS |
| `geometry_derived` | Deterministically derived from route or road geometry |
| `relationship_inferred` | Derived from nearby or related stronger evidence through a documented relationship, such as road continuity, corridor identity, sidepath relationship, facility adjacency, or propagated evidence |
| `predicted` | Model output |
| `baseline` | Generic prior used because stronger evidence is absent |
| `unknown` | No reliable source or derivation available |

The generic label `inferred` is not canonical for DS-015 score-bearing fields. Use `relationship_inferred` when the value is inferred from another observed, official, or derived relationship, and use `predicted` when the value comes from a model.

### 6.2 Evidence Precedence

Default precedence:

1. `observed`
2. `official_imported`
3. `geometry_derived`
4. `relationship_inferred`
5. `predicted`
6. `baseline`
7. `unknown`

Precedence decides which value wins. Confidence decides how much trust the winning value deserves.

### 6.3 Propagation Rule

Resolved truth may propagate along continuous road identity when no stronger conflicting evidence exists.

Propagation follows:

- road name continuity as the primary signal
- geometric continuity as the secondary signal
- directional coherence as the tertiary signal

Propagation stops when:

- road identity changes
- strong conflicting evidence appears
- an explicit override exists

Propagation does not stop merely because OSM split a way, changed a minor surface tag, or used a different highway class on a short continuation.

When a value propagates, the propagated segment must not retain the original direct provenance label. Example:

```text
official_imported speed on source segment -> relationship_inferred speed on propagated segment
observed shoulder on source segment -> relationship_inferred shoulder on propagated segment
```

This prevents UI and receipts from overstating the evidence.

### 6.4 Unknown Handling

Unknowns must not secretly become stronger risk claims than known values.

Rules:

- Unknown control uses the arithmetic mean of known control factors.
- Unknown movement uses the arithmetic mean of known movement factors.
- Unknown traffic uses the best available local prediction or baseline and lowers confidence.
- Unknown shoulder receives no mitigation credit and lowers confidence.
- Unknown facility receives no mitigation credit unless absence is known to be unreliable in that data context; in that case, retain `unknown` provenance and lower confidence.
- Unknown speed must resolve through a road-class / regional baseline rather than becoming zero or disappearing.

Missingness is carried in confidence, receipts, and caveats. It is not hidden inside a fake precise score.

## 7. Core Math

The primary conceptual split is **likelihood versus severity**.

Road risk and crossing risk are score-bearing domains. They are not competing kinds of truth. Both are derived the same way:

```text
Risk = Likelihood * SeverityWeight
```

This naming discipline matters because it prevents the model from treating "road" and "crossing" as unrelated formulas. They are two places where the same logic is applied:

- Road risk is sustained-exposure likelihood multiplied by speed severity.
- Crossing risk is discrete-event likelihood multiplied by crossed-road speed severity.

For each continuous road slice `j`:

```text
RoadRisk_j = RoadLikelihood_j * RoadSpeedSeverityWeight_j
RoadSpeedSeverityWeight_j = SpeedSeverityWeight(speed_j)
RoadLikelihood_j =
  sliceMiles_j
  * TrafficFactor_j
  * CurvatureFactor_j
  * FacilityLikelihoodFactor_j
  * ShoulderLikelihoodFactor_j
```

For each crossing event `i`:

```text
CrossingRisk_i = CrossingLikelihood_i * CrossingSpeedSeverityWeight_i
CrossingSpeedSeverityWeight_i = SpeedSeverityWeight(crossed_or_entered_road_speed_i)
CrossingLikelihood_i =
  capped(
    BaseCrossingLikelihood
    * sqrt(CrossedRoadTrafficFactor_i)
    * WidthFactor_i
    * ControlFactor_i
    * MovementFactor_i
  )
```

Route rollup:

```text
Total Route Risk = Σ RoadRisk_j + Σ CrossingRisk_i
Route Risk Per Mile = Total Route Risk / Route Miles
```

In plain English:

- Traffic, curvature, facilities, and shoulders shape sustained road likelihood.
- Crossed-road traffic, width, control, and movement shape crossing-event likelihood.
- Speed shapes severity in both domains.
- The model computes local risk points from likelihood times severity, then sums them.
- Longer routes may accumulate more total risk even when their mile-for-mile character is benign.
- `routeRiskPerMile` is the normalized basis for comparison.

## 8. Continuous-Road Risk

### 8.1 Formula

For each road slice:

```text
RoadLikelihood_j =
  0, if the slice is path / MUP domain under Section 8.6
  sliceMiles_j * TrafficFactor_j * CurvatureFactor_j * FacilityLikelihoodFactor_j * ShoulderLikelihoodFactor_j, otherwise
```

Then:

```text
RoadRisk_j = RoadLikelihood_j * RoadSpeedSeverityWeight_j
```

Speed does not appear in `RoadLikelihood` at launch. It appears in `RoadRisk` through `RoadSpeedSeverityWeight`.

### 8.2 Traffic Factor

Traffic is the continuous-road exposure backbone.

Canonical traffic input is **AADT per lane**. When official DOT / HPMS or equivalent AADT is available, Lanterne must use the exact numeric value, convert it to AADT per lane when needed, and interpolate against the traffic factor table. Real AADT must not be flattened into buckets.

Conversion:

```text
AADT per lane = total AADT / motor-vehicle lane count
```

If lane count is not directly observed or imported, the score may still use the value, but provenance and confidence must say so.

Traffic fallback ladder:

1. official AADT per lane
2. official total AADT plus known lane count
3. official total AADT plus relationship-inferred lane count
4. relationship-inferred total AADT from nearby official values and corridor context
5. `local_area_predicted` traffic
6. generic highway-type baseline
7. unknown

Each fallback step lowers confidence.

Step 5 launch policy:

- Local-area predicted traffic requires at least 2 nearby direct numeric traffic readings of the same broad highway class within the local search radius.
- Eligible contributors must be stronger direct truth only: observed/admin-approved traffic, state DOT traffic, HPMS traffic, or equivalent authoritative imported numeric AADT.
- `relationship_inferred`, `local_area_predicted`, `class_proxy`, baseline, and unknown traffic values must not be contributors.
- A single nearby reading must not promote to score-bearing local-area predicted truth; it falls through to the generic highway-type baseline.
- 2 readings may produce low-confidence `local_area_predicted` traffic.
- 3+ readings with low spread may produce medium-confidence `local_area_predicted` traffic.
- The trace must retain contributor roads, source AADT values, radius, lane-count basis, averaging math, and fallback reasons.

#### Launch TrafficFactor Table

The table is a piecewise-linear anchor table. The implementation interpolates between anchors and clamps only at the launch saturation tail.

| AADT per lane | TrafficFactor | Why |
| ---: | ---: | --- |
| 500 | 0.35 | Very low vehicle exposure; below the baseline riding environment. |
| 1,000 | 0.50 | Quiet-ish road; still below baseline but not zero. |
| 3,000 | 1.00 | Baseline light-to-moderate exposure. |
| 6,000 | 2.00 | Roughly double baseline exposure; preserves near-independent pass opportunity. |
| 12,000 | 4.00 | Still close to exposure-proportional growth. |
| 18,000 | 5.75 | Prevents very busy roads from being flattened too early. |
| 30,000 | 8.00 | Compression begins as traffic streams become more continuous. |
| 50,000 | 10.00 | High-density traffic; marginal new-car risk starts falling. |
| 75,000 | 11.25 | Platooning / traffic-stream saturation becomes material. |
| 100,000 | 12.00 | Very high traffic remains much worse than 18k, but no longer grows linearly. |
| 150,000+ | 12.50 | Launch saturation tail; incremental vehicles add little new independent exposure. |

Rationale:

- At ordinary and busy-road volumes, each additional vehicle is still a meaningful additional interaction opportunity.
- The model should therefore remain close to exposure-proportional through the lower and middle table.
- A road with 18,000 AADT per lane cannot be treated as only three times the 3,000 baseline if the model is claiming to represent vehicle exposure.
- At very high volume, traffic behaves less like independent isolated cars and more like a continuous stream. The rider's exposure window is bounded by time, gaps, queues, and platoons. Marginal risk per additional vehicle declines and eventually rounds toward zero.
- This high-volume tail is a Lanterne calibration extension beyond the most convenient published breakpoint table. It must be validated against the corpus, not presented as a directly published coefficient.

Rider-facing traffic display may convert AADT into average intensity:

```text
cars per minute average = AADT / 1440
average seconds per vehicle = 86400 / AADT
```

Those translations are explanatory only. AADT is an annual average and does not predict a specific ride hour.

### 8.3 Speed Severity Weight

Speed is the launch severity-weight backbone because posted speed is nationally available and benchmark support for a steep speed curve is strong.

The launch table is normalized so 25 mph equals 1.00.

| Posted speed | SpeedSeverityWeight | Why |
| --- | ---: | --- |
| 0-15 mph | 0.25 | Policy extension for shared-space and trail crossing contexts below the benchmark table floor. |
| <=20 mph | 0.35 | Low-speed environment; Table 156 shape supports a low tail. |
| 25 mph | 1.00 | Baseline. |
| 30 mph | 2.10 | Clear increase above baseline. |
| 35 mph | 3.60 | Major threshold; also aligns with Bike ISI high-speed signal. |
| 40 mph | 5.70 | Fast arterial / rural road regime. |
| 45 mph | 8.50 | Very high consequence environment. |
| 50 mph | 12.00 | Highway-like severity. |
| 55+ mph | 16.30 | Extreme launch severity regime. |

Implementation should interpolate between posted-speed anchors. Posted speed is a proxy for operating speed; provenance and confidence must reflect whether the value is official, tagged, relationship-inferred, predicted, or baseline.

### 8.4 Horizontal Curvature Factor

Curvature is included because it is both benchmark-backed and measurable from matched road geometry.

| Curvature class | Launch definition | CurvatureFactor |
| --- | --- | ---: |
| straight or gently curving | advisory speed >= 60 mph or radius > 2600 ft | 1.00 |
| moderate curvature | advisory speed 45 to <60 mph or radius 1300-2600 ft | 1.81 |
| sharp curve | advisory speed 25 to <45 mph or radius 650-1300 ft | 3.51 |
| very sharp curve | advisory speed <25 mph or radius <=650 ft | 6.02 |

For launch, use geometry-derived radius because advisory-speed truth is not nationally reliable.

Measurement rule:

- Classify by the worst curvature class sustained for at least 50 meters of contiguous slice-aligned geometry.
- Do not let one noisy vertex trigger a sharper class by itself.
- Store curvature derivation and confidence in the trace.

### 8.5 Facility And Shoulder Factors

Facility and shoulder reduce interaction opportunity by improving operating space. They are likelihood factors, not severity factors.

Facility table:

| Facility class | FacilityLikelihoodFactor |
| --- | ---: |
| fully separated / protected on-road facility | 0.50 |
| buffered bike lane | 0.75 |
| painted bike lane | 0.80 |
| no dedicated bike facility | 1.00 |

Shoulder table:

| Shoulder condition | ShoulderLikelihoodFactor |
| --- | ---: |
| sub-usable / none | 1.00 |
| usable shoulder | 0.85 |
| wide shoulder | 0.80 |

Application rules:

- Shoulder credit applies only when no dedicated bike facility is already providing operating-space credit.
- Shoulder credit is most meaningful at 30 mph and above; below that, launch scoring may use 1.00 unless the model version explicitly changes.
- A path / MUP is not a protected on-road facility. It is handled by the path / MUP rule below.
- Do not stack facility and shoulder credit in a way that double-counts the same operating space.

### 8.6 Path / MUP Domain

A path, MUP, or similarly separated corridor carries **zero continuous-road risk** when the route slice is functionally outside the motor-vehicle roadway stream.

The same route may still carry **crossing-event risk** where it crosses, joins, exits, or interacts with motor-vehicle roads.

Launch operational rule:

- If a slice is path-like / MUP-like and the relevant motor-vehicle speed context is absent or 15 mph or lower, set `RoadLikelihood = 0`.
- Preserve the slice in the trace with `roadRisk = 0` and a path / MUP explanation.
- Detect path-road crossings, driveway / access-road crossings, road joins, and road exits as crossing events when they meet eligibility.

This prevents the model from inventing road exposure on separated corridors while still acknowledging the crossings that can matter.

## 9. Crossing-Event Risk

Crossings are discrete conflict events. They are not smeared into continuous-road exposure and they are not ignored.

Crossing risk uses the same likelihood-versus-severity structure as road risk:

```text
Crossing Risk = Crossing Event Likelihood * Crossed-Road Speed Severity
```

The crossing likelihood side is where the event's conflict structure lives: crossed-road traffic, lanes / width, control, and movement. The severity side is still speed, because severe crossing outcomes are governed primarily by the speed environment of the crossed or entered road.

The crossing layer first asks:

```text
How conflict-prone is this crossing or join?
```

Then the speed of the crossed or entered road supplies the severity weight.

### 9.1 Event Eligibility

A crossing event enters score math when at least one condition is true:

- crossed or entered road speed is at least 30 mph and AADT per lane is at least 2,000
- lanes crossed is at least 3
- movement is left across traffic on a road that is at least 30 mph or at least 2,000 AADT per lane
- node is signalized on a materially trafficked motor road
- route joins, exits, or crosses a motor road from path / MUP domain
- route crosses a driveway or access-road connection that is materially part of the routed line and carries motor vehicles

Non-eligible crossings may still be counted, displayed, and traced as non-score-bearing events.

### 9.2 Crossing Formula

For each score-bearing crossing:

```text
CrossingLikelihood_i =
  min(
    CrossingLikelihoodCap,
    BaseCrossingLikelihood
      * sqrt(TrafficFactor_cross_i)
      * WidthFactor_i
      * ControlFactor_i
      * MovementFactor_i
  )

CrossingRisk_i = CrossingLikelihood_i * CrossingSpeedSeverityWeight_i
```

Launch constants:

```text
BaseCrossingLikelihood = 0.150
CrossingLikelihoodCap = 0.300
```

These constants are Lanterne calibration constants. They are not copied from FHWA, NCHRP, Bike ISI, or any other external table.

Why these constants exist:

- The base constant gives an ordinary score-bearing crossing a visible but bounded risk contribution.
- At baseline traffic, baseline width, baseline control, baseline movement, and 25 mph, the event contributes `0.150` risk points, comparable to a small fraction of a baseline road mile.
- The per-event cap prevents one uncertain node from becoming unbounded just because traffic, width, control, and movement all stack.
- The cap is applied before speed severity weight, so a high-speed crossing can still be meaningfully worse than a low-speed crossing.
- The constants should be tuned against known-route behavior and the full corpus. A change to either constant is a model-calibration change.

Why the traffic term uses square root:

- Crossed-road traffic should increase crossing likelihood.
- A crossing is a single event, not riding alongside every vehicle for a mile.
- Open data usually lacks turning counts, signal phasing, yielding behavior, and exact rider timing.
- Square-root compression keeps traffic important without pretending the system knows every conflict opportunity at that node.

### 9.3 Width Factor

| Lanes crossed | WidthFactor | Why |
| ---: | ---: | --- |
| 1-2 | 1.00 | Baseline crossing width. |
| 3-4 | 1.10 | More lanes, longer exposure window, more vehicle paths. |
| 5-6 | 1.20 | Wide arterial / highway-like crossing. |
| 7+ | 1.30 | Very wide crossing, still bounded because traffic and speed already do major work. |

Lanes crossed may be tagged, imported, geometry-derived, relationship-inferred, or predicted. The trace must say which.

### 9.4 Control Factor

| Control | ControlFactor | Why |
| --- | ---: | --- |
| stop-controlled | 1.00 | Baseline controllable crossing condition. |
| signalized | 1.05 | Signals often appear where the crossing problem is larger; signal does not automatically mean safer for cyclists without phasing and turning counts. |
| unknown | 1.025 | Arithmetic mean of known launch states; missing control lowers confidence. |

Unknown control is not the same thing as uncontrolled. If uncontrolled is directly known later, it should be introduced as a separate versioned category.

### 9.5 Movement Factor

| Movement | MovementFactor | Why |
| --- | ---: | --- |
| straight-across | 1.00 | Baseline movement. |
| right / merge | 1.05 | Adds conflict complexity, but remains modest without turning counts. |
| left across traffic | 1.20 | Broadest common path conflict and strongest rider-facing concern. |
| unknown / ambiguous | 1.0833 | Arithmetic mean of known launch states; missing movement lowers confidence. |

Unknown movement should be rare. If it is common, the problem is movement classification quality, not the coefficient.

### 9.6 No Route-Level Crossing Clamp

Canonical `Total Route Risk` is an additive sum of score-bearing road and crossing units. The launch model does not apply a route-level crossing-share clamp after summing, because that would make the canonical total no longer equal the sum of its trace.

Crossing influence is controlled through:

- event eligibility
- bounded width / control / movement multipliers
- square-root crossed-road traffic compression
- per-event crossing likelihood cap
- corpus-level calibration diagnostics

Expected crossing share ranges are diagnostics, not formula caps. If ordinary long-distance routes routinely show crossing shares far outside expected operating ranges, recalibrate event constants or eligibility instead of hiding the issue with a post-hoc clamp.

## 10. Sustained Exposure And Critical Stretches

Sustained road exposure is the backbone of the model. A long stretch of high-speed, high-traffic, low-space roadway should accumulate risk because the rider spends real distance in that environment.

At the same time, `routeRiskPerMile` can hide short severe segments on long rides. Lanterne must therefore compute a report-only critical-stretch layer from the same trace.

Critical stretch may include:

- worst contiguous kilometer
- worst contiguous mile
- 95th / 99th percentile road-slice risk
- top contributing road slices
- top crossing events

Critical stretch does not modify `totalRouteRisk` in this contract. It explains concentration and detour candidates.

Recommended rider-facing labels:

| Label | Meaning |
| --- | --- |
| notable ugly section | Worse than route background, but not automatically detour-worthy. |
| meaningful pinch point | Likely to deserve attention or timing. |
| detour candidate | Concentrated risk where an alternate may be worth evaluating. |
| avoid if possible | Extreme segment or crossing concentration. |

Each label must be backed by trace evidence, not hand-written vibes.

## 11. Comparative Projections

The canonical model outputs risk. Rider-facing comparison is a projection.

### 11.1 Rank

Rank compares `routeRiskPerMile` against a selected route network.

```text
Rank = ordinal position by ascending Route Risk Per Mile within the selected network
```

The safest route in the selected network is rank `#1`.

A rank is meaningless without:

- network name
- network version
- route inclusion rules
- model version
- evidence snapshot / scoring run id
- ranking timestamp

Every route may belong to multiple networks. The same route can therefore have multiple valid ranks.

Examples:

- Randonneur Network Rank
- Regional Network Rank
- Rural Route Rank
- Urban Route Rank
- Gravel-adjacent Exclusion Rank
- User-library Rank

Only `routeRiskPerMile` is the default ranking basis. `totalRouteRisk` remains visible because long rides accumulate more total exposure, but it must not be the default rank basis unless a projection explicitly says it is ranking total accumulated risk.

### 11.2 Curved Grade

Curved Grade is a school-analogous A-F label derived from percentile standing in a selected network.

Default launch mapping:

| Percentile in selected network by `routeRiskPerMile` | Grade |
| --- | --- |
| safest 10% | A |
| next 20% | B |
| middle 40% | C |
| next 20% | D |
| riskiest 10% | F |

This is not an absolute grade. It is a curve. A route can be a `B` among endurance routes and a `D` among local recreational routes if those networks have different distributions.

Every grade must carry:

- network name and version
- model version
- grade mapping version
- percentile basis
- whether ties are dense-rank, ordinal, or percentile-smoothed

The product may display grade first for readability, but receipts and admin views must preserve `totalRouteRisk` and `routeRiskPerMile`.

## 12. Map Paint Contract

Lanterne has two map-paint contracts. They are intentionally different.

Required contract names are **Route Risk Paint** for the selected analyzed route and **Road-Stress Overlay** for off-route visible roads. Legacy UI language such as heatmap may remain as a control label only if the implementation and receipts preserve this distinction.

### 12.1 Route Risk Paint

Route Risk Paint applies to an analyzed route.

It uses:

- canonical score-bearing road slices
- canonical crossing events
- score trace factors
- selected inputs
- provenance
- confidence
- cached canonical route artifact when available

It may communicate route risk because the analyzed route has direction, continuity, matched context, turns, crossings, chosen inputs, confidence, and route-specific geometry.

Once a canonical analyzed-route artifact exists, route paint must not silently fall back to viewport proxy coloring.

### 12.2 Road-Stress Overlay

Road-Stress Overlay applies to off-route visible roads in the viewport.

It answers:

```text
What kind of road environment is visible around me?
```

It does not answer:

```text
What is the canonical risk of this route?
```

The viewport overlay may use fast proxies:

- speed band
- road class
- facility class
- traffic availability or coarse traffic prior
- shoulder availability
- simple stress / suitability heuristics

It must follow client budgets:

- degrade overlay fidelity before degrading map responsiveness
- hydrate progressively
- avoid synchronous full-route scoring for every visible road
- avoid recomputing deep risk across dense viewports on overlay toggle
- label itself as road stress, road environment, or proxy overlay unless backed by canonical score-bearing artifacts

Full off-route risk may be computed only by:

- precomputed tile or slice artifacts served from cache
- worker-backed lazy computation for a bounded subset
- explicit user-requested analysis of a candidate route

This is not split-brain architecture. It is two named contracts for two different questions.

## 13. Scorecard, Method, Receipts, And Cache Contract

### 13.1 Scorecard

The scorecard should display:

- Curved Grade for the selected network
- Rank for the selected network
- Route Risk Per Mile
- Total Route Risk
- confidence band
- short caveat when important inputs are relationship-inferred, predicted, or baseline

It must not display a legacy 0-100 canonical score.

### 13.2 Method Card

The method card should explain:

- the model estimates motor-vehicle strike risk for bicyclists
- traffic and operating context shape interaction opportunity
- speed shapes likely outcome severity
- crossings are discrete score-bearing events
- missing data affects confidence and receipts
- rank and grade are relative to the selected network

### 13.3 Receipt Cards

Receipt cards should show:

- top road-slice contributors
- top crossing contributors
- traffic source and AADT basis
- speed source
- facility / shoulder basis
- crossing width / control / movement basis
- confidence caveats
- benchmark-derived versus Lanterne-calibrated items

### 13.4 Cache

The 4,000-route corpus should be scored into canonical artifacts before user-facing use whenever possible.

Cache key requirements:

- `routeGeometryHash`
- `modelVersion`
- `evidenceSnapshotId`
- traffic data version
- route matching version
- score-trace schema version

Rank and grade caches additionally require:

- network id
- network version
- inclusion filters
- grade mapping version
- scoring run id

Route load should prefer cached canonical artifacts. If a route must be scored live, the UI should show provisional status and persist the completed artifact for reuse.

## 14. Calibration, Source, And Provenance Matrix

This matrix is the practical guardrail against future model drift. Every score-bearing item must say what role it plays, what source family or policy decision supports it, whether it is benchmark-derived or Lanterne-calibrated, and why the decision exists.

Source keys:

| Key | Source family / decision basis | Used for |
| --- | --- | --- |
| `S-HSM-NCHRP` | HSM / HSM2 / NCHRP bicycle modeling family | Factorized roadway structure, likelihood-style road factors, AADT per lane, curvature, and deferred lane width / visibility / parking / lighting concepts |
| `S-SPEED-156` | National Academies / TRB `Pedestrian and Bicycle Safety Performance Functions`, Table 156, attributed to iRAP 2013l | Non-linear speed severity shape |
| `S-AADT` | NCHRP 17-84 / HSM2 AADT-per-lane breakpoint family | Traffic exposure anchor family |
| `S-BIKE-ISI` | FHWA Bicycle Intersection Safety Index | Lanes-to-cross, control, and movement relevance for bicyclist intersection conflicts |
| `S-CMF-FACILITY` | FHWA bicycle treatment CMF direction plus HSM / HSM2 facility and shoulder concepts | Directional support for operating-space mitigation |
| `S-LOCATION` | NTSB 2019 bicyclist safety study using 2014-2016 U.S. data, including about 65% of bicycle motor-vehicle crashes at intersections and 56% of bicyclist fatalities at midblock locations; NHTSA 2021 bicyclist fatality location summary, including 62% of bicyclist fatalities at non-intersection locations | Intersection-versus-midblock split, crossing importance, and sustained-exposure severity support |
| `S-PBCAT` | PBCAT-style bicycle crash typing | Rider-readable movement categories and conflict language |
| `P-LANTERNE` | Lanterne launch calibration and route pressure-testing policy | Constants, caps, thresholds, tail behavior, and product choices not published as external coefficients |
| `P-PROVENANCE` | ADR-042, ADR-043, and DS-017 truth-resolution / provenance architecture | Evidence precedence, propagation, confidence, and missing-data behavior |
| `P-PAINT-BUDGET` | DS-018 viewport overlay hydration and client-budget architecture | Route paint versus viewport overlay separation |

| Canonical item | Role in model | Source / decision basis | Status | Why |
| --- | --- | --- | --- | --- |
| Vehicle-strike-only scope | Model boundary | `S-LOCATION`, `P-LANTERNE` | Canonical scope | Keeps the score narrow enough to defend and prevents weather, fatigue, remoteness, and surface hazards from becoming hidden score math. |
| `Risk = Likelihood x SeverityWeight` | Governing score structure | `S-HSM-NCHRP`, `S-LOCATION` | Benchmark-shaped architecture | Separates conflict opportunity from likely outcome severity, which is the clearest and most defensible launch model form. |
| `Total Route Risk = Σ road risk + Σ crossing risk` | Canonical accumulated score object | `P-LANTERNE` | Canonical formula | Preserves additive traceability and lets riders see total accumulated exposure. |
| `Route Risk Per Mile = Total Route Risk / Route Miles` | Normalized comparison object | `P-LANTERNE` | Canonical formula | Compares route character without punishing long benign routes solely for length. |
| Projections may not compute alternate score truth | Projection boundary | `P-LANTERNE` | Canonical invariant | Rank, grade, and UI summaries must interpret the canonical artifact, not become competing scoring systems. |
| Intersection versus midblock split | Architecture support | `S-LOCATION` | Supporting empirical evidence | It explains why crossings matter while sustained open-road exposure remains the backbone of severe-outcome route risk. |
| Speed severity shape `sigma(s)` | Conditional severity curve | `S-SPEED-156` | Direct benchmark-derived shape | Speed has the strongest nationally available support for a steep severe-outcome curve. |
| Posted speed as severity proxy | Practical mapping of severity curve to route truth | `S-SPEED-156`, `P-LANTERNE` | Calibration / measurement policy | Posted speed is nationally available and explainable even though it is not literal operating-speed truth. |
| 25 mph normalization baseline | Sets `sigma(25) = 1.0` | `P-LANTERNE` | Calibration choice | A clear baseline is needed to express relative severity cleanly in receipts and code. |
| Rounded launch speed table values | Implementation-friendly severity table | `S-SPEED-156`, `P-LANTERNE` | Benchmark-informed / adapted | The shape stays benchmark-based while remaining usable in product code, traces, and docs. |
| Traffic anchor family | Continuous-road likelihood backbone | `S-AADT`, `S-HSM-NCHRP` | Benchmark-informed / adapted | AADT per lane is the strongest practical launch exposure backbone. |
| Exact AADT interpolation | Continuous traffic likelihood scaling | `S-AADT`, official DOT / HPMS numeric data | Canonical implementation rule | Exact DOT / HPMS readings should not be crushed into coarse buckets. |
| Expanded high-volume traffic anchors | Realistic heavy-traffic scaling | `P-LANTERNE` | Calibration choice | Traffic risk should continue rising materially through higher-volume regimes before platoon and saturation effects bend the curve. |
| Traffic saturation tail | Marginal traffic behavior | `P-LANTERNE` | Calibration choice | At very high volumes, traffic behaves more like a continuous stream; incremental cars add less new independent exposure. |
| Traffic fallback ladder | Weak-data handling for traffic truth | `P-PROVENANCE`, `P-LANTERNE` | Calibration / provenance policy | The system must stay useful under incomplete data without pretending certainty. |
| Curvature categories and factors | Geometry-driven road likelihood | `S-HSM-NCHRP` | Direct benchmark-derived factor | Sustained curve geometry is one of the few strong, measurable benchmark-backed road-likelihood factors. |
| Radius-based curvature measurement | Practical national curvature method | `S-HSM-NCHRP`, `P-LANTERNE` | Measurement policy | Radius is nationally derivable from geometry; advisory-speed truth is not. |
| 50 m sustained-curve rule | Noisy-vertex suppression | `P-LANTERNE` | Measurement policy | A single geometry blip should not fabricate a sharp-curve penalty. |
| Facility likelihood concept | Operating-space reduction of road likelihood | `S-CMF-FACILITY`, `S-HSM-NCHRP` | Benchmark-informed / adapted | Dedicated space materially reduces conflict opportunity but does not erase the surrounding road context. |
| Facility values `0.50 / 0.75 / 0.80 / 1.00` | Protected / buffered / painted / none | `S-CMF-FACILITY`, `P-LANTERNE` | Calibrated launch constants | Values reflect stronger separation for protected, meaningful benefit for buffered, and modest benefit for painted. |
| Shoulder likelihood concept | Operating-space reduction where no facility exists | `S-CMF-FACILITY`, `S-HSM-NCHRP` | Benchmark-informed / adapted | Shoulder matters as operating space, but should remain secondary to true facility separation. |
| Shoulder values `1.00 / 0.85 / 0.80` | None / usable / wide | `S-CMF-FACILITY`, `P-LANTERNE` | Calibrated launch constants | A usable shoulder deserves real credit, while still remaining weaker than true dedicated facility separation. |
| Shoulder application gate | Applies only when no facility and speed is at least 30 mph | `P-LANTERNE` | Calibration policy | Shoulder matters most where it actually changes passing-space conditions. |
| Path / MUP zero continuous-road risk | Prevents fake road risk on separated paths | `P-LANTERNE` | Canonical rule | A true path should not accumulate road risk while still carrying crossing risk. |
| 15 mph path-like threshold | Operational proxy for path treatment | `P-LANTERNE` | Measurement policy | Launch needs a workable proxy while staying conservative about safe-path certainty. |
| Non-intersection crossing inclusion | Includes path-road and similar motor-vehicle conflict points | `S-LOCATION`, `S-PBCAT`, `P-LANTERNE` | Supporting evidence plus policy | Real motor-vehicle conflict points are not limited to formal intersections. |
| Crossing structure overall | Width / control / movement crossing logic | `S-BIKE-ISI`, `S-PBCAT` | Benchmark-informed / adapted | These are real crossing shapers, but secondary to traffic and speed. |
| `Lambda_0` | Base crossing-likelihood contribution | `P-LANTERNE` | Calibration constant | Ordinary urban and suburban conflict nodes must register materially instead of vanishing. |
| `Lambda_cap` | Per-event crossing guardrail | `P-LANTERNE` | Calibration constant | One relationship-inferred node must not dominate the route. |
| Square-root crossed-road traffic term | Sublinear crossing-traffic compression | `P-LANTERNE` | Policy regularization | Crossed-road traffic should matter strongly without runaway multiplication. |
| Crossing event eligibility rules | Decides which nodes enter score math | `S-BIKE-ISI`, `S-LOCATION`, `P-LANTERNE` | Calibration policy | Not every mapped node deserves to affect canonical score. |
| Width factor concept | More lanes crossed means more conflict complexity | `S-BIKE-ISI` | Benchmark-informed / adapted | More crossing width increases exposure time and conflict space. |
| Width values `1.00 / 1.10 / 1.20 / 1.30` | 1-2 / 3-4 / 5-6 / 7+ lanes | `S-BIKE-ISI`, `P-LANTERNE` | Calibrated launch constants | Width matters, but remains bounded and secondary. |
| Control factor concept | Stop versus signalized structure matters modestly | `S-BIKE-ISI` | Benchmark-informed / adapted | Signalization changes conflict structure but is not an automatic cyclist safety credit. |
| Control values `1.00 / 1.05 / 1.025` | Stop / signal / unknown | `S-BIKE-ISI`, `P-LANTERNE` | Calibrated launch constants | Unknown should stay neutral; signalized should proxy slightly larger conflict context. |
| Movement factor concept | Straight / right / left movement structure matters modestly | `S-BIKE-ISI`, `S-PBCAT` | Benchmark-informed / adapted | Different movements create different conflict problems, especially left-across traffic. |
| Movement values `1.00 / 1.05 / 1.20 / 1.0833` | Straight / right / left / unknown | `S-BIKE-ISI`, `P-LANTERNE` | Calibrated launch constants | Left is the strongest common conflict class; unknown stays neutral instead of worst-case. |
| Unknown arithmetic-mean fallback rule | Neutral score fallback for missing modifiers | `P-PROVENANCE`, `P-LANTERNE` | Canonical policy | Missingness should lower confidence, not secretly add danger. |
| Confidence separate from risk | Trust model | `P-PROVENANCE` | Canonical policy | A poorly known risky route must not look safer just because data is missing. |
| Provenance family vocabulary | Evidence receipt contract | `P-PROVENANCE` | Canonical vocabulary | `observed`, `official_imported`, `geometry_derived`, `relationship_inferred`, `predicted`, `baseline`, and `unknown` prevent evidence claims from collapsing into one vague confidence score. |
| Propagated truth relabeling | Provenance integrity | `P-PROVENANCE` | Canonical policy | A propagated value from a direct source must not masquerade as directly observed. |
| Crossing share diagnostic | Route-level calibration and explanation signal | `S-LOCATION`, `P-LANTERNE` | Diagnostic only | Crossing share should reveal whether calibration is behaving plausibly without clamping the additive score trace. |
| Critical stretch artifact | Preserves short dangerous sections | `P-LANTERNE` | Report-only | Route averages alone should not hide short ugly connectors, but the artifact should explain concentration rather than mutate score. |
| Rank by risk per mile | Comparative interpretation basis | `P-LANTERNE` | Projection | Comparative route character should be normalized for length and scoped to a named network. |
| Curved Grade | Relative network / universe grading | `P-LANTERNE` | Projection | Riders need school-like A-F interpretation without pretending absolute precision. |
| Route Risk Paint | Analyzed-route visualization | Canonical score trace, `P-PAINT-BUDGET` | Canonical projection | The analyzed route should show actual score-bearing route truth, including direction, continuity, crossings, and chosen inputs. |
| Road-Stress Overlay | Off-route viewport visualization | `P-PAINT-BUDGET` | Non-canonical proxy | Visible off-route roads need a fast, budgeted proxy, not full canonical route-risk scoring for every visible road. |

## 15. Implementation Invariants

The implementation is wrong if any of these are true:

- A route has grade or rank but no canonical `totalRouteRisk` and `routeRiskPerMile`.
- A 0-100 value is treated as canonical score output.
- A projection layer computes alternate score truth instead of interpreting the canonical route artifact.
- Exact DOT / HPMS AADT is flattened into traffic buckets instead of interpolated.
- A path / MUP slice accumulates continuous-road risk.
- A path / MUP crossing of a motor road disappears from crossing-event detection.
- Confidence directly rescales risk.
- A score-bearing field emits generic `inferred` provenance instead of `relationship_inferred`, `predicted`, `baseline`, or another canonical provenance class.
- Propagated evidence retains a direct provenance label.
- Route paint uses viewport proxy colors after a canonical score trace exists.
- Viewport overlay is labeled as canonical route risk without score-bearing artifacts.
- Rank or grade is shown without network and version metadata.
- Cache keys omit model version, evidence snapshot, or route geometry identity.

## 16. Launch Validation

Before freezing this as production canon:

- Run the full seed corpus through the same scorer.
- Fix route corpus bugs that distort safety inputs before using ranks or grades.
- Check the traffic tail against known high-AADT routes and rider intuition.
- Check path / MUP routes to confirm zero road risk and preserved crossing risk.
- Check rural long routes to ensure sustained exposure remains the backbone.
- Check suburban / urban crossing-heavy routes to ensure crossings can matter without uncontrolled dominance.
- Compare crossing shares as diagnostics, not caps.
- Verify scorecard, route paint, method card, receipts, admin views, and cache all read from the same canonical artifact.
- Version every calibration table and cache output.

The desired launch state is not a prettier display. It is one score-bearing contract with traceable inputs, explicit projections, and enough provenance to explain why the system believes what it believes.


---

## Source File: docs/02-architecture/design/ds-015-safety_scoring_model_5_4.md

# DS-015 — Safety Scoring Model

**Status:** Canonical  
**Updated:** 2026-04-25

This document is the single authoritative specification for Lanterne's safety model.

If code, admin audit, receipts, scorecard, inspector, analyzed route paint, heatmap, or rider-facing comparative surfaces disagree with this document, the implementation is wrong until fixed or explicitly re-specified.

## 1. Purpose

Define one coherent safety system for Lanterne:

- what safety means
- what inputs are score-bearing
- how canonical truth is resolved
- how route risk is computed
- how confidence and provenance work
- how map paint contracts differ by layer
- how downstream surfaces must project the score
- how rider-facing comparative interpretation works

## 2. Canonical Safety Definition

Lanterne's canonical safety model measures one thing only:

> expected serious motor-vehicle risk to a bicyclist

That risk is expressed as route risk points and risk per mile.

Included in canonical safety:

- motor-vehicle speed environment
- motor-vehicle traffic intensity
- operating space provided by bike facility and shoulder
- sustained roadway geometry that materially changes vehicle-conflict risk
- crossing and turning conflict exposure

Explicitly excluded from canonical safety:

- weather
- wind
- temperature
- fatigue
- remoteness
- route difficulty
- surface comfort / ride quality
- descent challenge
- lighting as a launch score term
- POI convenience
- schedule / event logistics
- scenery
- non-motor-vehicle micro-hazards as canonical score terms

Those may appear elsewhere in the product. They do not belong in canonical Safety Score.

## 3. Canonical Architecture

The system has three layers.

### 3.1 Canonical truth

One chosen truth object per score-bearing field per road slice or crossing event.

### 3.2 Canonical score artifact

One canonical score trace derived only from canonical truth.

### 3.3 Projection surfaces

All downstream surfaces are projections:

- analyzed route paint
- inspector
- scorecard
- method
- receipts
- admin audit
- rider-facing rank and curved grade

Projection surfaces may summarize, suppress, reword, or repackage. They may not compute alternate score truth.

## 4. Canonical Truth Contract

Every score-bearing road slice or crossing event must resolve one chosen truth object for:

- speed
- traffic
- bike facility
- shoulder
- sustained curvature where applicable
- lane count / crossing width where applicable
- crossing control where applicable
- crossing movement where applicable

Each chosen truth field must carry:

- `value`
- `provenance`
- `confidence`
- `is_inferred`
- trace metadata sufficient for explanation and audit

## 5. Provenance and Confidence

### 5.1 Provenance families

Every score-bearing truth field must map honestly into one of:

- `observed`
- `official_imported`
- `geometry_derived`
- `relationship_inferred`
- `predicted`
- `baseline`
- `unknown`

Implementation-specific subtypes are allowed, but they must map honestly to one of these families.

### 5.2 Overclaiming is forbidden

No score-bearing field may overclaim provenance.

Examples:

- OSM-tag-derived posted speed may not be labeled as direct real-world observation
- inferred lane count may not masquerade as exact observed lane truth
- adjacent-road-derived bike-facility semantics may not masquerade as exact segment tagging
- predicted traffic may not be labeled as official traffic

### 5.3 Confidence is separate

Confidence is distinct from provenance and distinct from score.

Confidence affects:

- projection caveats
- inspection trace
- admin diagnostics
- ranking caution
- suppression of fake exactness in rider-facing displays

Confidence does not directly soften or harden canonical score math unless this spec explicitly says so.

### 5.4 Unknown handling

Unknown values must not secretly become worst-case danger.

When the score needs a numeric fallback for an unknown modifier:

- use a bounded neutral fallback
- reduce confidence separately

Missingness should damage confidence more than it damages score.

## 6. Non-Negotiable Truth Rule

Canonical score computation must consume the chosen propagated truth artifact.

It must not compute from:

- raw cue-entry fields
- pre-resolution fallback values
- display shims
- surface-specific ad hoc inference

If scoring uses different truth than inspector, analyzed route paint, or receipts claim, the implementation is wrong.

## 7. Canonical Score Model

The canonical score object is route risk.

The governing decomposition is:

- road risk
- crossing risk

But both of those are just local forms of the same underlying model:

`risk = severity × likelihood`

The key distinction is that road slices and crossing events use different likelihood inputs.

### 7.1 Core decomposition

For continuous road slices:

`road risk = road severity × road likelihood`

At launch:

- road severity is governed by speed
- road likelihood is governed by traffic, facility, shoulder, and sustained curvature

For crossing events:

`crossing risk = crossing severity × crossing likelihood`

At launch:

- crossing severity is governed by the speed of the crossed or entered road
- crossing likelihood is governed by crossed-road traffic, width, control, and movement

So the route-level model is:

`Total Risk = sum(road risk points) + sum(crossing risk points)`

This is the clean conceptual rule that downstream surfaces should understand.

### 7.2 Notation

- `M` = total route miles
- `N` = number of continuous road slices
- `K` = number of scored crossing events
- `j` = continuous road-slice index
- `i` = crossing-event index
- `m_j` = miles in continuous road slice `j`
- `lambda_road_j` = incident-likelihood contribution of road slice `j`
- `lambda_cross_i` = incident-likelihood contribution of crossing event `i`
- `sigma(v)` = conditional severity weight at speed `v`
- `r_road_j` = local road risk points from road slice `j`
- `r_cross_i` = local crossing risk points from crossing event `i`
- `R_road` = total continuous-road risk points
- `R_cross` = total crossing risk points
- `CrossingShare` = diagnostic crossing share of total route risk
- `R_total` = total route risk points
- `R_rpm` = route risk per mile

## 8. Continuous-Road Incident Likelihood

For each continuous road slice:

```text
lambda_road_j =
  0, if the slice is path-like / MUP-like under Section 8.6
  m_j × TF_j × CurvF_j × FF_j × ShF_j, otherwise
```

Where:

- `TF_j` = traffic factor
- `CurvF_j` = horizontal-curvature factor
- `FF_j` = facility likelihood factor
- `ShF_j` = shoulder likelihood factor

This is a likelihood-only equation. Speed does not appear here at launch.

### 8.1 Backbone rule

Continuous-road exposure remains the backbone of route risk.

The system is not allowed to collapse into a crossing-only model.

### 8.2 Traffic factor

Traffic uses AADT per lane as the canonical backbone.

Exact numeric AADT values from DOT or HPMS must be used continuously when available. The system must not bucket precise traffic readings when it has enough information to map them continuously.

Let `v` be the best available AADT-per-lane value for the road slice.

```text
TF(v) = InterpTrafficAnchors(v)
```

Anchor points:

```text
(500, 0.35)
(1000, 0.50)
(3000, 1.00)
(6000, 2.00)
(12000, 4.00)
(18000, 5.75)
(30000, 8.00)
(50000, 10.00)
(75000, 11.25)
(100000, 12.00)
(150000, 12.50)
```

Launch anchor table:

| AADT per lane anchor | TrafficFactor | Why |
| ---: | ---: | --- |
| 500 | 0.35 | very low vehicle exposure; below the baseline riding environment |
| 1,000 | 0.50 | quiet-ish road; still below baseline but not zero |
| 3,000 | 1.00 | baseline light-to-moderate exposure |
| 6,000 | 2.00 | roughly double baseline exposure; preserves near-independent pass opportunity |
| 12,000 | 4.00 | still close to exposure-proportional growth |
| 18,000 | 5.75 | prevents very busy roads from being flattened too early |
| 30,000 | 8.00 | compression begins as traffic streams become more continuous |
| 50,000 | 10.00 | high-density traffic; marginal new-car risk starts falling |
| 75,000 | 11.25 | platooning / traffic-stream saturation becomes material |
| 100,000 | 12.00 | very high traffic remains much worse than 18k, but no longer grows linearly |
| 150,000+ | 12.50 | launch saturation tail; incremental vehicles add little new independent exposure |

Interpolation rule:

- if `v <= 500`, use `0.35`
- if `v >= 150000`, use `12.50`
- otherwise interpolate linearly between surrounding anchors

Why this curve is shaped this way:

- in ordinary and busy-road regimes, per-car conflict opportunity stays close to linear and should not be flattened prematurely
- a road at 18,000 AADT per lane cannot be treated as only three times the 3,000 baseline if the model is claiming to represent vehicle exposure
- only at very high volumes do traffic streams begin to behave more like a coordinated mass, reducing the incremental risk added by each additional car

Traffic fallback ladder, highest confidence to lowest:

1. official AADT per lane
2. official AADT total + known lane count
3. official AADT total + inferred lane count
4. inferred AADT total from nearby official values by road type / corridor context
5. generic road-class / highway-type traffic proxy
6. unknown

Fallback rule:

- if the system has real or derived numeric AADT-per-lane, use continuous interpolation
- if the system only has generic proxy traffic, it may use the nearest anchor instead of pretending to have fine precision
- every weaker fallback step must reduce confidence

Rider-facing display rule for traffic:

- when public surfaces show daily volume, they should also translate it into rider-readable average intensity such as cars/hour or cars/minute average

### 8.3 Horizontal-curvature factor

Horizontal curvature is canonical likelihood, not just a presentation flourish.

Launch categories:

| Horizontal curvature category | Benchmark definition | HorizontalCurvatureFactor |
| --- | --- | ---: |
| straight or gently curving | advisory speed >= 60 mph or radius > 2600 ft | 1.00 |
| moderate curvature | advisory speed 45 mph to < 60 mph or 1300 < r <= 2600 ft | 1.81 |
| sharp curve | advisory speed 25 mph to < 45 mph or 650 < r <= 1300 ft | 3.51 |
| very sharp curve | advisory speed < 25 mph or r <= 650 ft | 6.02 |

For launch, use the radius side of the benchmark because advisory-speed truth is not nationally reliable.

Metric equivalents:

- `2600 ft ≈ 792 m`
- `1300 ft ≈ 396 m`
- `650 ft ≈ 198 m`

Measurement rule:

- derive local curve radius from matched road geometry
- assign a road slice to the worst materially represented curvature category inside that slice
- a single noisy vertex must not trigger a sharper category by itself
- classify by the worst curvature category present for at least `50 m` of contiguous slice-aligned geometry
- if no sustained curve exists, use `1.00`

### 8.4 Facility likelihood factor

Facility reduces continuous-road likelihood through operating space.

Launch table:

| Facility class | FacilityLikelihoodFactor |
| --- | ---: |
| fully separated / protected on-road facility | 0.50 |
| buffered bike lane | 0.75 |
| painted bike lane | 0.80 |
| no dedicated bike facility | 1.00 |

Why:

- protected remains the strongest operating-space reduction
- buffered is materially better than painted, but not as dramatic as full separation
- painted still matters, but not enough to bury the surrounding road environment

Rules:

- buffered remains distinct from painted
- sharrows are not dedicated bike infrastructure
- facility is a bounded operating-space reducer, not a hidden primary driver

### 8.5 Shoulder likelihood factor

Shoulder is a bounded operating-space reducer.

Shoulder only applies when:

- no dedicated bike facility is present
- posted speed is at least `30 mph`

Otherwise:

```text
ShF_j = 1.00
```

Launch table:

| Shoulder condition | ShoulderLikelihoodFactor |
| --- | ---: |
| sub-usable / none | 1.00 |
| usable shoulder | 0.85 |
| wide shoulder | 0.80 |

Why:

- a usable shoulder materially improves operating space and should receive real credit
- a wide shoulder is still better than a merely usable one
- shoulder remains weaker than true dedicated facility separation

Shoulder is not a disguised bike-facility class.

### 8.6 Path / MUP rule

When a slice is functionally a bike path or MUP-like facility, the canonical score must not invent continuous road risk that is not really there.

Launch operational proxy:

- if the relevant speed limit is `15 mph` or lower, the slice carries zero continuous-road risk
- that slice still carries crossing risk where it intersects or enters motor-vehicle roads

This means path/MUP mileage contributes zero road risk, but not zero crossing risk.

### 8.7 Weak-evidence path rule

Path-like names or geometry alone do not justify safe-path certainty.

If the system only has weak path-like evidence, it must not:

- invent fake motor-road baseline speed
- invent fake safe-path certainty
- overclaim provenance

### 8.8 Non-intersection conflict events

Crossings are not limited to formal mapped intersections.

If the route:

- crosses a motor road from a path / MUP
- joins a motor road from a path / MUP
- exits a motor road onto a path / MUP
- crosses a driveway-access or access-road connection that is clearly part of the routed line

then it counts as a crossing event when it meets eligibility.

## 9. Conditional Severity

Speed is the dominant launch severity variable.

It is not the only thing that matters in reality. It is the only severity-side variable that is strong enough, measurable enough, and explainable enough to carry canonical launch severity.

Let `s` be posted speed.

```text
BenchmarkSpeedRatio(s) = Interp(Table156, s) / Table156(25 mph)
sigma(s) = BenchmarkSpeedRatio(s)
```

For continuous road slices, use the slice's posted speed.

For crossing events, use the posted speed of the crossed or entered road.

Launch table:

| Posted speed | SeverityWeight | Plain English |
| --- | ---: | --- |
| <= 20 mph | 0.35 | very low-speed outcome severity |
| 25 mph | 1.00 | baseline severity environment |
| 30 mph | 2.10 | clearly above baseline |
| 35 mph | 3.60 | meaningful severity jump |
| 40 mph | 5.70 | high-consequence environment |
| 45 mph | 8.50 | very high-consequence environment |
| 50 mph | 12.00 | extreme consequence environment |
| 55+ mph | 16.30 | maximum launch severity regime |

Rules:

- non-linear speed treatment is required
- interpolation between benchmark points is preferred to arbitrary band jumps
- posted speed is a practical severity proxy, not a claim of measured operating speed truth
- normalization and rounding choices are policy choices and must be treated as such

What stays out of launch severity:

- parking
- driveway density
- curb friction
- lighting
- state
- commercial proxies
- random road-class modifiers

Heavy-vehicle severity remains explicitly deferred pending a defensible measurement layer.

## 10. Crossing-Event Incident Likelihood

Crossings are explicit, discrete, and secondary to the continuous-road backbone.

For each score-bearing crossing event:

```text
lambda_cross_i =
  min(
    Lambda_cap,
    Lambda_0 × (TF_cross_i)^0.5 × WF_i × CF_i × MF_i
  )
```

Launch constants:

```text
Lambda_0 = 0.150
Lambda_cap = 0.300
```

Why these constants exist:

- `Lambda_0` ensures ordinary urban and suburban conflict nodes register materially instead of vanishing inside the route backbone
- `Lambda_cap` prevents any one crossing, especially one built from incomplete open data, from becoming absurdly dominant
- the square-root traffic term keeps crossed-road traffic important while preventing runaway multiplication from uncertain node detail

### 10.1 Event eligibility

A detected crossing event is not automatically a scored crossing event.

Every crossing event must be classified as one of:

- `included`
- `excluded`
- `deduped`

with an explicit reason.

A crossing event enters score math when at least one of the following is true:

- the crossed or entered road has `speed >= 30 mph` and `AADT per lane >= 2,000/day/lane`
- `lanes crossed >= 3`
- the movement is left across traffic
- the node is signalized on a materially trafficked motor road
- the route joins or exits a path / MUP onto a motor road

Signalized and stop-controlled nodes may still be detected and displayed even when excluded from score.

### 10.2 Width factor

| Lanes crossed | WidthFactor |
| --- | ---: |
| 1–2 | 1.00 |
| 3–4 | 1.10 |
| 5–6 | 1.20 |
| 7+ | 1.30 |

Why:

- more lanes means more conflict space and more time exposed in the crossing
- width matters, but remains bounded and secondary

### 10.3 Control factor

| Control | ControlFactor |
| --- | ---: |
| stop-controlled | 1.00 |
| signalized | 1.05 |
| unknown | 1.025 |

Why:

- signalized is not an automatic safety credit for cyclists
- signalized nodes often proxy larger and more conflict-rich conditions
- unknown uses the arithmetic mean of known launch options

### 10.4 Movement factor

| Movement | MovementFactor |
| --- | ---: |
| straight-across | 1.00 |
| right / merge | 1.05 |
| left across traffic | 1.20 |
| unknown / ambiguous | 1.0833 |

Why:

- straight is baseline
- right / merge is meaningfully more complex than straight, but only modestly
- left across traffic is the strongest conflict class at launch
- unknown uses the arithmetic mean of known launch options

### 10.5 Secondary-shaper rule

Width, control, and movement are:

- real
- benchmark-informed
- bounded
- secondary to traffic on the likelihood side and speed on the severity side

Width should matter more than control. Left should matter more than right. Unknown should use bounded neutral fallback plus confidence loss.

## 11. Local Risk and Route Rollup

### 11.0 Conceptual reading

The route-level equation should be read as:

`Total Risk = sum(road risk points) + sum(crossing risk points)`

Where:

- road risk points are produced by:
  - severity = speed
  - likelihood = traffic × curvature × bike facility × shoulder
- crossing risk points are produced by:
  - severity = crossed-road speed
  - likelihood = crossed-road traffic × width × control × movement

This is the clearest way to explain the model without losing the exact formulas below.

### 11.1 Local road risk

```text
r_road_j = lambda_road_j × sigma(v_j)
```

### 11.2 Local crossing risk

```text
r_cross_i = lambda_cross_i × sigma(v_cross_i)
```

### 11.3 Continuous-road route risk

```text
R_road = sum(r_road_j)
```

### 11.4 Crossing route risk

```text
R_cross = sum(r_cross_i)
```

### 11.5 Crossing share diagnostic

```text
CrossingShare = R_cross / (R_road + R_cross), when denominator > 0
```

Crossing share is a calibration and explanation diagnostic. It is not a route-level clamp. Crossing contribution is controlled through event eligibility, square-root crossed-road traffic compression, bounded width/control/movement factors, and the per-event crossing likelihood cap.

### 11.6 Total route risk

```text
R_total = R_road + R_cross
```

### 11.7 Risk per mile

```text
R_rpm = R_total / M
```

Canonical score artifact:

- `Total Risk = R_total`
- `Risk / Mile = R_rpm`

## 12. Critical-Stretch Report

Route averages alone must not hide ugly short sections.

The system must preserve a canonical critical-stretch artifact, but the artifact is report-only. It does not mutate `R_total`, `R_rpm`, rank, or curved grade.

At minimum, the score artifact must include:

- worst rolling critical stretch risk per mile
- the stretch band used for rider-facing explanation

The critical-stretch artifact exists to explain concentration, identify detour candidates, and prevent a benign average from hiding a short ugly connector. It is not a hidden score penalty or hard cap.

## 13. Canonical Outputs

The canonical route output family includes at minimum:

- total continuous-road risk
- crossing risk
- crossing share diagnostic
- total route risk
- route miles
- risk per mile
- road confidence
- route confidence
- score model version
- truth-resolution / propagation version where relevant
- included / excluded / deduped crossing counts
- contribution trace by factor family
- report-only critical-stretch artifact

There is no canonical 0–100 score.

## 14. Rider-Facing Comparative Layer

Comparative interpretation is layered on top of canonical outputs. It does not replace them.

Stable underlying outputs:

- total route risk
- risk per mile
- road confidence
- route confidence

Comparative outputs:

- rank
- curved grade

### 14.1 Comparison basis

Comparative ranking is based on risk per mile, not total route risk alone.

This preserves the distinction between:

- route character per mile
- total accumulated burden over distance

### 14.2 Curved grade bands

Default comparative bands:

| Percentile standing in chosen corpus | Curved Grade |
| --- | --- |
| top 10% | A |
| next 20% | B |
| middle 40% | C |
| next 20% | D |
| bottom 10% | F |

This is school-analogous A–F grading relative to a selected network or universe, not an absolute canonical grade.

### 14.3 Metadata is mandatory

Every comparative output must carry:

- reference network / universe name
- reference network / universe version
- canonical model version
- grade-curve version

Comparative outputs without defined corpus metadata are meaningless.

### 14.4 Public honesty rule

Comparative outputs answer:

`How does this route compare with other routes in the chosen network or universe?`

They do not answer:

`What is the model's canonical truth about this route?`

## 15. Map Paint Contracts

Map paint has two distinct contracts.

### 15.1 Analyzed route paint

The selected/analyzed route paint must be canonical risk-based from the route's scoreTrace and score-bearing units.

It answers:

`What is the canonical risk along this analyzed route?`

### 15.2 Heatmap / viewport road overlay

The heatmap is the viewport road overlay. It is a fast road-stress / suitability proxy layer for visible off-route roads.

It answers:

`What kind of roads are around me?`

It is not canonical route risk.

### 15.3 Explicit separation rule

The problem is not that there are two paint paths. The problem is whether they are named and contracted clearly enough that they do not look like competing truth systems.

Therefore:

- analyzed route paint = canonical score-bearing risk
- heatmap / viewport overlay = budgeted proxy layer

The heatmap / viewport overlay must:

- degrade fidelity before degrading map responsiveness
- remain cheap enough for interactive use
- compute deeper off-route risk only lazily, in a worker, or from precomputed cache/tile artifacts

## 16. Projection Rules

### 16.1 Inspector and analyzed route paint

Inspector and analyzed route paint must project canonical truth, not alternate truth.

### 16.2 Heatmap

Heatmap is not canonical route-risk paint.

Heatmap must project the budgeted viewport overlay contract defined in Section 15.2:

- cheap enough for interactive use
- explicitly proxy in nature
- clearly distinct from canonical analyzed-route paint
- never described as the route's canonical score trace

### 16.3 Scorecard, Method, and Receipts

These surfaces must project canonical score trace.

They may not:

- recompute competing contributions
- label percentages as points
- omit score-bearing factors that the canonical trace actually used

If curvature, crossing risk, or another factor affects score, it must remain explainable downstream.

### 16.4 Admin audit

Admin audit must prefer canonical artifacts over legacy summary fields whenever both exist.

### 16.5 Corpus hydration

Corpus hydration must persist enough canonical artifact to support later comparative and explanation surfaces honestly.

Persisting only thin summaries is acceptable for ranking snapshots, not for full receipts, method, or provenance replay.

## 17. Public Transparency Rules

Public-facing explanation should say:

- the score is a route-risk model, not a crash probability
- traffic, crossings, and operating space shape incident likelihood
- speed shapes conditional severity
- sustained curvature is score-bearing where benchmark-backed and measurable
- hazards may be modeled separately without entering the canonical safety score
- missing data weakens confidence and provenance, not just the rhetoric around the score

## 18. Rider-Facing Display Thresholds

Display thresholds are not the canonical formula, but they are still part of the contract.

### 18.1 Speed paint bands

| Rider-facing display band | Speed range |
| --- | --- |
| blue | `<= 15 mph` |
| green | `16–30 mph` |
| orange | `31–45 mph` |
| red | `> 45 mph` |

These are display-only thresholds. They do not change canonical score math.

## 19. Deferred-But-Supported Factors

The following are real but not active launch canonical score terms:

- driveway / access density
- lane width
- advance visibility of curves
- vehicle parking
- street lighting
- heavy-vehicle exposure

They must remain explicitly named as deferred rather than silently disappearing from the design.

## 20. Implementation Rules

1. Keep the canonical score narrow.
2. Score only from canonical propagated truth.
3. Keep hazards separate unless they directly describe score-bearing motor-vehicle strike pathways already represented in canonical truth.
4. Do not double count speed, traffic, facility, shoulder, curvature, or crossing context.
5. Use honest provenance and confidence instead of fake precision.
6. Preserve the distinction between benchmark-derived terms and Lanterne policy/calibration terms.
7. Keep comparative outputs additive, not substitutive.
8. Keep weak-evidence path semantics from pretending to be either road truth or safe-path truth.
9. Preserve report-only critical-stretch visibility for short dangerous stretches.
10. Make every score-bearing factor traceable enough to audit.

## 21. Provenance and Calibration Matrix

| Canonical item | Role in model | Status | Why |
| --- | --- | --- | --- |
| route risk architecture `risk = likelihood × severity` | canonical score structure | benchmark-shaped architecture | Separating conflict opportunity from outcome severity is the clearest, most defensible model form. |
| intersection vs midblock split used to justify likelihood vs severity separation | architecture support | supporting empirical evidence only | It explains why crossings matter but do not replace the open-road backbone. |
| speed severity shape `sigma(s)` | conditional severity curve | direct benchmark-derived | Speed is the strongest and clearest launch severity variable. |
| posted speed as severity proxy | practical mapping of severity curve to route truth | Lanterne calibration / policy | Posted speed is nationally available and explainable even though it is not literal operating-speed truth. |
| 25 mph normalization baseline | sets `sigma(25) = 1.0` | Lanterne calibration / policy | A clear baseline is needed to express relative severity cleanly. |
| rounded launch speed table values | implementation-friendly severity table | benchmark-informed / adapted | The shape stays benchmark-based while remaining usable in product code and docs. |
| traffic anchor family | continuous-road likelihood backbone | benchmark-informed / adapted | AADT per lane is the strongest practical launch exposure backbone. |
| midpoint interpolation across traffic anchors | continuous traffic likelihood scaling | Lanterne calibration / policy | Exact DOT/HPMS readings should not be crushed into coarse buckets. |
| expanded high-volume traffic anchors | realistic heavy-traffic scaling | Lanterne calibration / policy | Traffic risk should continue rising materially through higher-volume regimes before platoon effects bend the curve. |
| traffic fallback ladder | weak-data handling for traffic truth | Lanterne calibration / policy | The system must stay useful under incomplete data without pretending certainty. |
| curvature categories and factors | geometry-driven likelihood | direct benchmark-derived | Sustained curve geometry is one of the few strong, measurable benchmark-backed road-likelihood factors. |
| radius-based curvature measurement | practical national measurement method | Lanterne calibration / policy | Radius is derivable nationally; advisory-speed truth is not. |
| 50 m sustained-curve rule | noisy-vertex suppression | Lanterne calibration / policy | A single geometry blip should not fabricate a sharp-curve penalty. |
| facility likelihood concept | operating-space reduction of road likelihood | benchmark-informed / adapted | Dedicated space materially reduces conflict opportunity but does not erase the surrounding road context. |
| facility values `0.50 / 0.75 / 0.80 / 1.00` | protected / buffered / painted / none | Lanterne calibration / policy | These values reflect stronger separation for protected, meaningful benefit for buffered, and modest benefit for painted. |
| shoulder likelihood concept | operating-space reduction where no facility exists | benchmark-informed / adapted | Shoulder matters as operating space, but should remain secondary to true facility separation. |
| shoulder values `1.00 / 0.85 / 0.80` | none / usable / wide | Lanterne calibration / policy | A usable shoulder deserves real operating-space credit, while still remaining weaker than true dedicated facility separation. |
| shoulder application gate | only when no facility and speed >= 30 mph | Lanterne calibration / policy | Shoulder matters most where it actually changes passing-space conditions. |
| path/MUP zero continuous-risk rule | prevents fake road risk on separated paths | Lanterne calibration / policy | A true path should not accumulate road risk while still carrying crossing risk. |
| 15 mph path-like threshold | operational proxy for path treatment | Lanterne calibration / policy | Launch needs a workable proxy while staying conservative about safe-path certainty. |
| non-intersection crossing inclusion | path-road and similar conflict-point inclusion | supporting empirical evidence + policy | Real motor-vehicle conflict points are not limited to formal intersections. |
| crossing structure overall | width / control / movement crossing logic | benchmark-informed / adapted | These are real crossing shapers, but secondary to traffic and speed. |
| `Lambda_0` | base crossing-likelihood contribution | Lanterne calibration / policy | Ordinary urban/suburban conflict nodes must register materially instead of vanishing. |
| `Lambda_cap` | per-event crossing guardrail | Lanterne calibration / policy | One inferred node must not dominate the route. |
| square-root crossed-road traffic term | sublinear crossing-traffic compression | Lanterne calibration / policy | Crossed-road traffic should matter strongly without runaway multiplication. |
| crossing event eligibility rules | decides which nodes enter score math | Lanterne calibration / policy | Not every mapped node deserves to affect canonical score. |
| width factor concept | more lanes crossed means more conflict complexity | benchmark-informed / adapted | More crossing width increases exposure time and conflict space. |
| width values `1.00 / 1.10 / 1.20 / 1.30` | 1–2 / 3–4 / 5–6 / 7+ lanes | Lanterne calibration / policy | Width matters, but remains bounded and secondary. |
| control factor concept | stop vs signalized structure matters modestly | benchmark-informed / adapted | Signalization changes conflict structure but is not an automatic cyclist safety credit. |
| control values `1.00 / 1.05 / 1.025` | stop / signal / unknown | Lanterne calibration / policy | Unknown should stay neutral; signalized should proxy slightly larger conflict context. |
| movement factor concept | straight / right / left structure matters modestly | benchmark-informed / adapted | Different movements create different conflict problems, especially left-across traffic. |
| movement values `1.00 / 1.05 / 1.20 / 1.0833` | straight / right / left / unknown | Lanterne calibration / policy | Left is the strongest conflict class; unknown stays neutral instead of worst-case. |
| unknown = arithmetic-mean fallback rule | neutral score fallback for missing modifiers | Lanterne calibration / policy | Missingness should hurt confidence, not secretly add danger. |
| crossing share diagnostic | route-level calibration and explanation signal | Lanterne calibration / policy | Crossing share should reveal whether calibration is behaving plausibly without clamping the additive score trace. |
| report-only critical-stretch artifact | preserves short dangerous sections | Lanterne calibration / policy | Route averages alone should not hide short ugly connectors, but the artifact should explain concentration rather than mutate score. |
| rank by risk per mile | comparative interpretation basis | Lanterne calibration / policy | Comparative route character should be normalized for length. |
| curved grade bands | relative network/universe grading | Lanterne calibration / policy | Riders need school-like A–F interpretation without pretending absolute precision. |
| analyzed route paint = canonical trace | route paint contract | Lanterne calibration / policy | The analyzed route should show actual score-bearing risk truth. |
| heatmap = viewport overlay proxy layer | off-route paint contract | Lanterne calibration / policy | Visible off-route roads need a fast, budgeted proxy, not full canonical route-risk scoring. |

## 22. Design Rule

Estimate conflict opportunity with the variables that generate incidents. Estimate consequence with the variable that most clearly governs injury severity. Resolve truth honestly. Compute one canonical score artifact. Project outward from that artifact. Layer comparison on top without replacing truth.


---

## Source File: docs/02-architecture/design/ds-016-experience_policy_layer.md

# Experience Policy layer #

2026-03-31

## Launch Packet Companion Note

This document is one part of the Lanterne launch packet.

Companion documents:
1. EXEC-008 v2 — Experience Runtime, Surface Architecture, and Domain Migration Program
2. EXEC-008 v2 — Master Implementation Manual
3. DS-016 — Experience Policy Layer
4. Lanterne Launch Packet — Index and Companion Guide

Use this document for its primary job only:
- Program Plan = architecture and system boundaries
- Implementation Manual = execution order, SQL, prompts, verification
- DS-016 = runtime behavior, prompts, routing, and state policy
- Launch Packet Index = reading order, ownership, and packet framing
------

# 1. Executive summary

The **Experience Policy Layer** is Lanterne’s behavioral operating system.

It answers, at every moment:

- what state the rider/app is in
- what information deserves attention now
- what surface gets first claim on that information
- what is interruptive vs passive
- what is durable truth vs session fluff
- what can be dismissed, snoozed, retried, or blocked

It should sit **above** component state and **below** rendering.

It should **not** own:

- route scoring
- GPS matching internals
- pixel layout
- route geometry storage
- the actual drawer components

It **should** own:

- runtime context classification
- prompt arbitration
- surface routing
- suppression/calmness
- resume/recovery behavior
- input eligibility rules
- mode and audience presentation defaults

The right model is **layered**, not monolithic:

1. **Domain policy layer**
    Durable enums, contracts, identities, push/expedition/window rules.
2. **Runtime orchestration layer**
    Computes the current experience state from route, ride, GPS, analysis, connectivity, and expedition state.
3. **Prompt/caption arbitration layer**
    Chooses whether the rider gets a blocking prompt, elevated prompt, passive caption, chip, or nothing.
4. **Surface adapter layer**
    Translates policy decisions into:
   - map emphasis
   - lantern state
   - tile order/emphasis
   - drawer defaults
   - review surface defaults
   - public page badges/actions

That gets you the goal from the brief: roughly 80% of launch behavior leaves ad hoc component conditionals and becomes durable policy.

A concrete runtime output should look like this:

```
type ExperienceDecision = {
  runtimeContext: RuntimeContext;
  primarySurfaces: SurfaceId[];
  secondarySurfaces: SurfaceId[];
  activePrompt: PromptDecision | null;
  activeCaption: CaptionDecision | null;
  lanternState: LanternAttentionState;
  tileProfile: TileProfileDecision;
  defaultDrawer: DrawerId | null;
  reviewEntryPoint: ReviewEntryPoint | null;
  blockedActions: ActionId[];
  badges: BadgeDecision[];
};
```

------

# 2. Canonical axes

## 2.1 Core axes

| Axis                           | Values                                                       | Rule                                                         |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `mode`                         | `rando`, `ultra_endurance`, `road`                           | Presentation/defaults only. Never decides structure.         |
| `structure`                    | `single_push`, `expedition`                                  | Journey form. Never inferred from mode.                      |
| `audience_role`                | `user`, `power_user`, `admin_debug`                          | Access depth and diagnostics only. Never changes score truth. |
| `runtime_context`              | `no_route`, `planning`, `analyzing`, `ready`, `active_ride`, `paused_break`, `review`, `public_page` | Top-level behavior context.                                  |
| `ride_state`                   | `not_started`, `armed`, `active_on_route`, `active_off_route`, `paused_short`, `paused_break`, `resume_pending`, `completed`, `abandoned` | Rider execution state.                                       |
| `analysis_state`               | `none`, `analyzing`, `ready_fresh`, `ready_stale`, `partial`, `failed` | Actionability and freshness of analysis.                     |
| `expedition_state`             | `none`, `planned`, `active`, `paused`, `completed`, `abandoned` | Durable multi-push truth.                                    |
| `window_state`                 | `none`, `planned`, `queued`, `analyzing`, `ready`, `active`, `completed`, `failed`, `stale` | Active detailed-analysis window state.                       |
| `connectivity_state`           | `online`, `degraded`, `offline`                              | Controls retries, queueing, and warning style.               |
| `gps_state`                    | `denied`, `unavailable`, `acquiring`, `locked_weak`, `locked_good` | Controls start/resume eligibility and confidence.            |
| `resume_state`                 | `not_applicable`, `eligible`, `mismatch`, `gps_pending`, `blocked` | Recovery/resume behavior.                                    |
| `review_depth`                 | `summary`, `standard`, `deep`, `diagnostic`                  | Controls detail exposure.                                    |
| `note_input_eligibility_state` | `ineligible`, `planning_only`, `stop_only`, `break_only`, `review_only`, `manual_only`, `queued` | Controls structured input access.                            |
| `guidance_state`               | `none`, `on_plan`, `drifting_plan`, `drifting_constraint`, `recovery_required` | Push-intelligence severity state.                            |

## 2.2 Illegal couplings

These are banned:

- `mode` must not select `single_push` vs `expedition`
- `mode` must not alter Safety Score semantics
- `audience_role` must not alter route truth or scoring
- `analysis_state` must not be inferred from drawer visibility
- `resume_state` must not be inferred from UI memory alone
- `public_page` must not expose live ride state by default
- `window_state` must not be reused as route identity
- `route_history_id` must not be the long-term center of new durable policy logic when `canonical_route_id` exists

## 2.3 Durable vs transient ownership

### Durable

Lives in domain/persistence:

- canonical route identity
- push plan
- expedition record
- expedition window records
- last confirmed route mile / point index
- structured observation records
- analysis freshness/completeness metadata
- public page publish state

### Transient

Lives in runtime/session:

- current camera position
- current open drawer
- current second-by-second timer tick
- ephemeral caption visibility
- gesture routing
- momentary lantern animation
- non-durable tile ordering state

That matches the docs: expedition state is durable, live session state is transient; stable analysis and contextual ride-time analysis are separate; and the product should not turn into a data cockpit.

------

# 3. Surface inventory

| Surface                  | Primary in                | Role                                                         | Auto-open allowed?    | Policy owner        |
| ------------------------ | ------------------------- | ------------------------------------------------------------ | --------------------- | ------------------- |
| **Map**                  | active ride, planning     | Spatial truth, progress, hazards, route shape, position, corridor/window context | Yes                   | map adapter         |
| **Lantern stack**        | active ride               | Single synthesized attention object                          | Yes                   | lantern adapter     |
| **Ride computer tiles**  | active ride               | Glanceable execution metrics and immediate guidance          | Yes                   | tile adapter        |
| **Drawers**              | planning, break, review   | Deep dive, explanation, edits, route detail, score explanation | Yes outside motion    | drawer adapter      |
| **Review surfaces**      | review                    | Structured summary, comparison, reconciliation, confidence context | Yes                   | review adapter      |
| **Public route page**    | public page               | Durable shareable route view                                 | N/A                   | public-page adapter |
| **Prompt/caption layer** | all contexts              | Interruptive choice, elevated heads-up, passive context      | Yes                   | prompt arbiter      |
| **Modal/overlay**        | recovery/destructive-only | Blocking decisions only                                      | Yes, strictly limited | prompt arbiter      |

## Surface primacy by runtime context

| Runtime context | Primary surfaces           | Secondary surfaces                   |
| --------------- | -------------------------- | ------------------------------------ |
| `no_route`      | map, acquisition controls  | none                                 |
| `planning`      | map + drawer               | caption, chips                       |
| `analyzing`     | map + analysis progress    | lantern pulse, caption               |
| `ready`         | map + drawer               | review CTA, chips                    |
| `active_ride`   | map + lantern + ride tiles | drawers only by user or stop context |
| `paused_break`  | drawer + map               | lantern, tiles                       |
| `review`        | review surface + map       | drawers                              |
| `public_page`   | public route page          | embedded map                         |

### Hard rule

During `active_ride`, drawers are never the center of gravity unless:

- rider is stopped or in break context
- a blocking recovery decision is required
- rider explicitly opens the drawer

That comes straight from the brief and should not be negotiated away later.

------

# 4. Top-level state model

This should be implemented as **four cooperating policy machines**, not one god machine.

## 4.1 Route context machine

- `no_route_loaded`
- `route_loaded_planning`
- `route_loaded_analyzing`
- `route_loaded_ready`
- `public_route_page`

## 4.2 Ride execution machine

- `not_started`
- `armed`
- `active`
- `paused_short`
- `paused_break`
- `resume_pending`
- `completed`
- `abandoned`

## 4.3 Analysis machine

- `none`
- `analyzing`
- `ready_fresh`
- `ready_stale`
- `partial`
- `failed`

## 4.4 Expedition/window machine

- `no_expedition`
- `expedition_planned`
- `expedition_active`
- `expedition_paused`
- `expedition_completed`
- `window_planned`
- `window_queued`
- `window_analyzing`
- `window_ready`
- `window_active`
- `window_failed`
- `window_stale`

## 4.5 Combined user-visible states

### A. No route loaded

Sub-states:

- home idle
- acquisition open (`Route To`, `Draw`, `Open`)
- public page entry

### B. Route loaded / planning

Sub-states:

- unanalyzed route
- analyzed editable route
- push planning
- expedition planning
- pre-ride conditions review
- share prep

### C. Analyzing

Sub-states:

- full-route analysis
- window analysis
- degraded/retrying
- partial-ready pending completion

### D. Analyzed / ready

Sub-states:

- ready fresh
- ready stale
- ready partial
- ready with publish/share available
- ready with low confidence badge

### E. Active ride

Sub-states:

- on route / stable
- drifting vs plan
- drifting vs hard constraint
- off-route
- approaching stop
- approaching window boundary
- recovery required

### F. Paused / break

Sub-states:

- short pause
- planned stop
- unplanned break
- overnight / long stop
- resume check

### G. Review

Sub-states:

- segment review
- route review
- push review
- expedition review
- end-of-push reconciliation
- post-share review

### H. Failed / partial / stale

Sub-states:

- analysis failed
- analysis partial
- window failed
- stale route analysis
- stale window analysis
- degraded connectivity

------

# 5. Transition model

## Transition principles

1. **Route truth first**
    Route/execution transitions never depend on drawer state.
2. **Resume truth first**
    Resume always checks durable expedition state before session memory.
3. **Actionability first**
    `partial` and `stale` can still be usable; `failed` is not.
4. **Interrupt only for decisions**
    Informational changes become chips/captions unless the rider must choose.
5. **Motion demotes detail**
    While moving, explanation slides down from drawer -> tile/chip/caption.

## Primary transitions

| From                        | Trigger                           | Guard                                              | Side effects                                                 | To                                                  | Surfaces affected                     |
| --------------------------- | --------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------- | ------------------------------------- |
| `no_route_loaded`           | route opened/imported/drawn       | route parse succeeds                               | create route context, clear stale prompts                    | `route_loaded_planning`                             | map, top controls                     |
| `route_loaded_planning`     | analyze requested                 | route valid, budgets acceptable                    | start analysis progress, clear share CTA                     | `route_loaded_analyzing`                            | map, analysis progress, lantern pulse |
| `route_loaded_analyzing`    | analysis success                  | complete enough for use                            | persist analysis metadata, compute badges                    | `route_loaded_ready`                                | map, drawer, chips                    |
| `route_loaded_analyzing`    | analysis partial                  | partial usable                                     | persist partial state, attach confidence warning             | `route_loaded_ready(partial)`                       | map, chip, review                     |
| `route_loaded_analyzing`    | analysis fail                     | no usable result                                   | persist failure state                                        | `failed`                                            | map, drawer, retry CTA                |
| `route_loaded_ready`        | push plan saved                   | route exists                                       | persist push/expedition config                               | `armed`                                             | planner/review                        |
| `armed`                     | start ride                        | route usable; if expedition, expedition row exists | start live session, write `started/resumed`, initialize tiles | `active_ride`                                       | map, lantern, tiles                   |
| `active_ride`               | manual pause or stop criteria met | none                                               | write sparse checkpoint; recalc stop state                   | `paused_break` or `paused_short`                    | drawer may become primary             |
| `paused_break`              | resume                            | GPS acceptable or manual override                  | write resume event, restore primary ride surfaces            | `active_ride`                                       | map, lantern, tiles                   |
| any expedition state        | app reopen                        | open expedition exists                             | consult durable expedition state first                       | `resume_pending` / `eligible` / `mismatch`          | resume card/prompt                    |
| `resume_pending`            | GPS mismatch detected             | materially far from saved progress                 | block silent resume                                          | `resume_mismatch`                                   | blocking prompt                       |
| `active_ride`               | preload threshold crossed         | `detail_mode='windowed'`                           | queue next window                                            | `window_queued`                                     | passive chip/caption                  |
| `active_ride`               | next window ready                 | queued window complete                             | mark ready                                                   | `window_ready`                                      | chip only                             |
| `active_ride`               | window boundary reached           | next window ready                                  | activate next window, write event                            | `window_active(next)`                               | passive caption, map continuity       |
| `active_ride`               | window boundary reached           | next window not ready                              | degrade confidently, surface recovery state                  | `window_failed/stale`                               | lantern + tile + chip/prompt          |
| `active_ride`               | push completed                    | route/push end met                                 | write completion event                                       | `review(push)`                                      | review surface                        |
| `review(push)`              | reconciliation saved              | expedition exists and incomplete                   | update next push assumptions                                 | `route_loaded_ready` or `expedition_active_planned` | review + planner                      |
| `route_loaded_ready/review` | publish/share                     | owner + shareable analysis                         | create/update public page state                              | `public_route_page(owner)`                          | public page                           |
| `public_route_page`         | owner opens planner               | owner rights                                       | restore private planning context                             | `route_loaded_planning`                             | map + drawer                          |

## Recovery-specific transitions

DS-014 already gives you the hard resume contract:

- GPS near saved progress -> one-tap resume
- GPS materially far -> mismatch prompt, no silent resume
- GPS unavailable -> still restore context, but don’t invent certainty

That should be treated as immutable behavior, not optional polish.

------

# 6. Surface routing rules

This is where random local decisions go to die.

## 6.1 Routing matrix

| Info / need                         | First surface                         | Second surface        | Never first during motion |
| ----------------------------------- | ------------------------------------- | --------------------- | ------------------------- |
| live position / off-route state     | map                                   | lantern               | drawer                    |
| push pace / cutoff / required speed | ride tile                             | lantern               | drawer                    |
| synthesized ride attention state    | lantern                               | tile                  | modal                     |
| upcoming cue / turn                 | map + tile                            | caption               | drawer                    |
| hazard ahead                        | map highlight + lantern               | tile                  | modal unless blocking     |
| score explanation                   | drawer                                | review surface        | lantern                   |
| score confidence / missing-data     | chip/tile badge                       | review drawer         | blocking modal            |
| stale analysis / stale window       | chip -> prompt only if action blocked | drawer                | repeated toast            |
| stop planning / stop editing        | drawer                                | map                   | modal                     |
| stop skipped                        | tile/caption                          | drawer                | full-page review          |
| resume mismatch                     | blocking overlay/modal                | map                   | passive chip              |
| approaching window boundary         | caption or tile chip                  | drawer                | modal                     |
| public share invitation             | review surface / public page          | drawer                | active ride prompt        |
| structured note request             | caption -> sheet at stop              | review surface        | forced modal in motion    |
| diagnostics                         | debug drawer                          | review diagnostic tab | rider-facing lantern      |

## 6.2 Surface routing laws

1. **Map owns spatial truth**
    Location, route progress, segment highlights, hazards, off-route, and window boundaries start on the map.
2. **Lantern owns synthesis**
    One sentence of rider-state meaning. Not data soup.
3. **Tiles own execution**
    Pace, constraint gap, next stop/window/cue, immediate route-intelligence snippets.
4. **Drawers own explanation and edit**
    If the rider needs to understand *why* or *change something*, that belongs in a drawer or review surface.
5. **Captions own calm context**
    Useful, non-blocking, low-friction context.
6. **Modals own blocked decisions only**
    Mismatch, destructive actions, impossible-to-continue ambiguity.
7. **Public page owns durable sharing**
    Not live ride state, not private stop plan, not debug.

------

# 7. Prompt and caption policy

## 7.1 Prompt tiers

| Tier       | Meaning                                                      | Examples                                     |
| ---------- | ------------------------------------------------------------ | -------------------------------------------- |
| `blocking` | rider must choose before the system can proceed safely/coherently | resume mismatch                              |
| `urgent`   | action materially affects execution now                      | severe cutoff drift, unusable next window    |
| `elevated` | rider should act soon, but ride can continue                 | stop skipped, approaching boundary with risk |
| `passive`  | informational, calm                                          | share CTA, confidence chip, heads-up caption |

Only one `blocking` prompt may exist at a time.

## 7.2 Launch prompt table

| Prompt / caption                    | Fires when                                                   | Must not fire when                                           | Tone                           | Urgency                                | Suppress / repeat                                 | Dismissal                                   | Follow-up                                  |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------ | -------------------------------------- | ------------------------------------------------- | ------------------------------------------- | ------------------------------------------ |
| **Resume mismatch**                 | open expedition + current GPS materially far from last confirmed route progress | GPS unavailable; no open expedition                          | direct, factual                | blocking                               | persists until resolved; no stack                 | no blind dismiss; choose path               | resume saved point / reposition / planning |
| **Stop skipped**                    | planned stop/control passed beyond tolerance without confirmation | stop optional; rider in complex high-attention nav moment; mismatch unresolved | calm, matter-of-fact           | elevated                               | once per stop unless rider changes status         | `mark skipped`, `still stopping`, `later`   | update stop plan and remaining stop budget |
| **Push drift / behind plan**        | projected finish or key checkpoint slips beyond configured plan threshold | no rider plan exists; within first warm-up window; drift unchanged since recent snooze | prescriptive, transparent      | passive -> urgent by severity          | repeat only on escalation tier or cooldown expiry | snooze to next stop/break or timed cooldown | open push guidance drawer                  |
| **Approaching window boundary**     | `detail_mode='windowed'` and boundary/preload threshold crossed | full mode; next boundary already acknowledged and status unchanged | heads-up                       | passive if next ready; elevated if not | once per boundary unless readiness changes        | dismiss until state changes                 | open window status chip/drawer             |
| **Stale route/window recovery**     | active or viewed analysis is stale/failed/needs refresh      | rider already accepted same stale state this session and nothing changed | plain, confidence-aware        | passive unless action blocked          | sticky chip, not spammy                           | `use as-is`, `refresh`, `later`             | queue refresh or continue degraded         |
| **Score confidence / missing-data** | low confidence, missing key inputs, partial analysis         | already visible as same chip and user hasn’t changed context | transparent, not apologetic    | passive                                | persistent badge; no repeated toast               | session-hide only                           | open “why confidence is lower”             |
| **Speed limit confirmation**        | structured input enabled + recent traversed segment has conflicting/weak speed data + rider in eligible context | rider moving; same corridor recently asked; no route context | brief, observational           | passive                                | corridor + time cooldown                          | `yes`, `no`, `unsure`, `later`              | queue structured observation               |
| **Shoulder confirmation**           | same pattern as above for shoulder uncertainty               | same exclusions                                              | brief, observational           | passive                                | same corridor + time cooldown                     | same                                        | queue structured observation               |
| **Caution marker flow**             | rider manually invokes or accepts optional stop-context suggestion | rider moving                                                 | structured, minimal            | passive                                | no auto-repeat                                    | complete / cancel                           | write structured caution marker            |
| **Public route share prompt**       | owner has fresh-enough route and no active publish flow; usually after ready/review | active motion; stale/failed analysis; already dismissed for this route version | invitational, not growth-hacky | passive                                | once per meaningful route version                 | `not now`                                   | open publish sheet/page                    |

## 7.3 Prompt threshold defaults

These should live in config, not components.

Recommended launch defaults:

- `resume_mismatch_distance_miles`: configurable, conservative
- `drift_plan_mild_minutes`: 15
- `drift_plan_severe_minutes`: 30
- `constraint_drift_urgent_minutes`: 10
- `stop_skipped_tolerance_miles`: 0.5 or stop-window-based
- `note_request_cooldown_hours`: 24
- `note_request_cooldown_miles_same_corridor`: 10
- `boundary_preload_trigger_miles`: use expedition default (`25`)
- `stale_chip_only_age_days`: configurable
- `stale_blocking_age_days_for_share`: configurable

------

# 8. Input request policy

Launch rule: **structured inputs only**. No open-ended Field Notes at launch.

## 8.1 Policy table

| Input                         | Eligibility                                                  | Safety constraints                           | Mode sensitivity                                             | Context sensitivity            | Opt-in only?              | During motion?                               | Initiator surface                              | Store / queue / retry                                        |
| ----------------------------- | ------------------------------------------------------------ | -------------------------------------------- | ------------------------------------------------------------ | ------------------------------ | ------------------------- | -------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| **Speed limit confirmation**  | recent traversed segment with low/conflicting speed confidence | never ask while moving                       | strongest value in `rando` / `road`; valid everywhere        | planning, stop, break, review  | yes                       | no                                           | caption -> sheet, review drawer, admin tool    | write local queue immediately; sync with route context, route mile, lat/lon, point index |
| **Shoulder confirmation**     | same as above for shoulder uncertainty                       | never ask while moving                       | strongest value in `rando` / `road`; valid everywhere        | planning, stop, break, review  | yes                       | no                                           | caption -> sheet, review drawer                | same queue model                                             |
| **Structured caution marker** | manual rider action only at launch                           | never allow while moving                     | strongest value in `ultra_endurance` / `rando`; valid everywhere | stop, break, review, planning  | effectively yes           | no                                           | map long-press sheet, break drawer, review CTA | queue structured marker with bounded type/severity/position  |
| **Stop confirmation**         | active planned stop/control reached within tolerance         | must prefer stopped / near-stopped contexts  | strongest in `rando`; still valid in `ultra_endurance`       | active ride, break             | no if planned stops exist | low-speed or stopped only                    | tile CTA, stop sheet                           | write immediate session state + durable stop event if expedition/push exists |
| **Stop skipped handling**     | planned stop/control passed without confirmation             | no blocking modal while rider is moving fast | strongest in `rando`; useful in `ultra_endurance`            | active ride -> review fallback | no                        | no blocking; passive only until slow/stopped | tile/caption -> drawer/review                  | update stop status and guidance budgets                      |

## 8.2 Storage contract

Each launch input should capture:

- `canonical_route_id` when available
   fallback: current route record id
- `push_id` / `expedition_id` if present
- `window_index` if windowed
- `route_mile`
- `point_index` if known
- `lat`, `lon`
- `source_context` (`planning`, `stop`, `break`, `review`)
- `created_at`
- `sync_state` (`pending`, `synced`, `failed`)
- `idempotency_key`

## 8.3 Retry model

- Write locally first
- Sync in background if online
- If sync fails, keep pending
- Surface pending state only in review/debug or subtle owner UI
- Never hammer rider mid-ride about sync failure unless it blocks a safety-critical transition

------

# 9. Mode-specific policy differences

Shared law first: **same route truth, same Safety Score semantics, same stable/contextual split across all modes.** Only prominence, copy, defaults, and review framing change.

## 9.1 Launch differences

| Dimension           | `rando`                                                 | `ultra_endurance`                                           | `road`                                               |
| ------------------- | ------------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------- |
| planner framing     | official start/close and control logic are prominent    | push continuity and carry-forward state are prominent       | route analysis and route comparison are prominent    |
| tile emphasis       | pace vs close, next control/stop, remaining stop budget | push drift, remoteness/light, next window/break, continuity | safety ahead, cue, effort/fatigue, simplified pace   |
| lantern emphasis    | “on plan / behind plan / close-risk”                    | “stable / drifting / recovery required / boundary risk”     | “clear / caution / hazard ahead”                     |
| copy style          | brevet-aware, constraint-aware                          | push-aware, continuity-aware                                | route-quality and risk-aware                         |
| stop logic          | stronger stop/control awareness                         | stronger optional break/sleep reconciliation                | lighter stop behavior unless rider enabled push plan |
| review emphasis     | stop behavior, cutoff behavior, control timing          | push reconciliation, continuity, drift carry-forward        | route comparison, risk explanation, hazard summary   |
| public page framing | route + eventish utility if owner wants it              | route + expedition-capable credibility, but no live state   | route-analysis-forward                               |
| POI default bias*   | resupply/control-first                                  | resupply/sleep/continuity-first                             | lighter convenience/café/water-first                 |

\* Exact POI ordering should be finalized against `PROD-010`, which was not retrieved here.

## 9.2 Hard no-fake-differences rule

Do **not** invent analytical mode differences that change truth.
 Examples of things mode must **not** do:

- change Safety Score meaning
- reinterpret the same shoulder differently
- make stale data acceptable in one mode but not another
- silently disable expedition logic because the mode is `road`

------

# 10. Audience-role policy differences

| Dimension                   | `user`                             | `power_user`                            | `admin_debug`                                       |
| --------------------------- | ---------------------------------- | --------------------------------------- | --------------------------------------------------- |
| default review depth        | `summary` / `standard`             | `standard` / `deep`                     | `diagnostic` available                              |
| score explanation           | plain-English, bounded             | richer breakdown, “why” depth           | raw diagnostics, contradictions, cache/version data |
| hazard visibility           | actionable only                    | more surrounding context                | full raw hazard/debug views                         |
| confidence display          | simple badge and short explanation | badge + why + freshness/version details | raw contributing reasons, thresholds, cache lineage |
| stale/partial details       | concise explanation                | explicit stale/partial reasons          | full failure reasons and retry state                |
| prompt logs                 | hidden                             | optional recent prompt history          | full suppression/arbitration log                    |
| window/expedition internals | mostly hidden                      | status visible                          | full event stream and window states                 |
| public page tools           | owner-only simple controls         | richer publish controls                 | diagnostics overlay, moderation/admin actions       |

## Audience rule

`admin_debug` is a **privileged overlay**, not a different rider experience.
 Nothing in admin/debug may leak into normal rider-facing or public-facing defaults.

------

# 11. Public route page policy

Public route pages are launch scope, but they are **durable route pages**, not live ride dashboards. 

## 11.1 Public page states

- `private_unpublished`
- `owner_preview`
- `public_fresh`
- `public_stale`
- `public_partial`
- `public_unavailable`

## 11.2 What each viewer sees

### Visitor

Sees:

- route title / description
- canonical route map
- stable analysis summary
- major hazard / caution summary
- analysis version badge
- freshness badge
- confidence badge if not high
- owner-approved route metadata

Does **not** see:

- live GPS
- current expedition progress
- private stop plan
- push drift
- unresolved owner prompts
- admin/debug truth

### Owner

Sees visitor view plus:

- publish/unpublish
- refresh analysis CTA
- share link controls
- “open in planner”
- owner-only stale/partial warnings
- optional preview of what is public vs private

### Power user / admin

Sees owner view plus:

- version lineage
- confidence explanation
- partial reasons
- diagnostics overlay (gated)

## 11.3 Shareable vs private

| Data                                      | Default                                                      |
| ----------------------------------------- | ------------------------------------------------------------ |
| route geometry                            | shareable                                                    |
| stable route analysis                     | shareable                                                    |
| stable hazards summary                    | shareable                                                    |
| analysis version / freshness / confidence | shareable                                                    |
| route conditions snapshot                 | shareable only if owner explicitly publishes a dated snapshot |
| live ride position                        | private                                                      |
| active push state                         | private                                                      |
| actual stop behavior                      | private                                                      |
| admin/debug diagnostics                   | private                                                      |

## 11.4 Public page captions and prompts

- `public_stale`: “Analysis may be outdated.”
   Owner gets refresh CTA. Visitor gets informational badge only.
- `public_partial`: “Some sections were analyzed with lower confidence.”
   Visitor sees badge + short explainer; owner sees refresh CTA.
- `share_ready`: owner-only passive CTA in planning/review/public preview.
   Never during active motion.

## 11.5 Owner actions

- publish
- unpublish
- refresh analysis
- open planner
- regenerate share snapshot
- choose stable public summary fields
- view share preview

No public crowd-notes workflow at launch.

------

# 12. Anti-spam / calmness rules

1. **One interruptive prompt at a time.**
2. **Resume mismatch beats everything.**
3. **During motion, prefer tile/chip/caption over drawer/modal.**
4. **Low-value prompts never interrupt active ride.**
5. **Confidence and freshness warnings default to chips, not popups.**
6. **Same corridor should not get repeated speed/shoulder asks aggressively.**
7. **Dismissed prompts only return on material state change or cooldown expiry.**
8. **Prompt escalation requires actual severity escalation, not just time passing.**
9. **Public share prompts never fire in motion.**
10. **Do not show rider-facing debug truth.**
11. **If a prompt is missed in motion, convert it into break/review work, not repeated harassment.**
12. **Every prompt must answer three things fast:** why now, what choice exists, what changes next.
13. **No stacked drawers + modals + captions circus.** One foreground attention channel.
14. **Calm beats clever.** If a rule is technically smart but rider-noisy, demote it.

------

# 13. State tables

## 13.1 Runtime behavior matrix

| State                 | Trigger                        | Guard                       | Action                                     | Surface                         | Next state                          |
| --------------------- | ------------------------------ | --------------------------- | ------------------------------------------ | ------------------------------- | ----------------------------------- |
| `no_route_loaded`     | user opens/imports/draws route | route valid                 | create route context                       | map + acquisition controls      | `planning.unanalyzed`               |
| `planning.unanalyzed` | analyze tapped                 | route valid                 | start analysis, clear stale prompts        | map + analysis progress         | `analyzing`                         |
| `analyzing`           | analysis complete              | fresh enough                | persist metadata, compute badges           | map + drawer                    | `ready.fresh`                       |
| `analyzing`           | analysis complete              | usable but incomplete       | persist partial + confidence               | map + chip + review             | `ready.partial`                     |
| `analyzing`           | analysis failed                | none                        | surface retry/recovery                     | map + drawer                    | `failed.analysis`                   |
| `ready.*`             | push plan saved                | none                        | persist plan/expedition                    | planner / review                | `armed`                             |
| `armed`               | ride started                   | route usable                | write start event, initialize ride profile | map + lantern + tiles           | `active_ride`                       |
| `active_ride`         | drift threshold crossed        | plan exists                 | update guidance state                      | tile + lantern + caption/prompt | `active_ride.drifting`              |
| `active_ride`         | planned stop reached           | stop exists                 | expose stop confirmation CTA               | tile + stop sheet               | `active_ride.stop_context`          |
| `active_ride`         | pause detected/manual pause    | none                        | write sparse checkpoint                    | drawer + map                    | `paused_break` or `paused_short`    |
| `paused_break`        | resume tapped                  | GPS okay or manual override | write resume event                         | map + lantern + tiles           | `active_ride`                       |
| `resume_pending`      | GPS near saved progress        | open expedition             | offer one-tap resume                       | resume card                     | `resume_eligible`                   |
| `resume_pending`      | GPS far from saved progress    | open expedition             | block silent resume                        | mismatch overlay                | `resume_mismatch`                   |
| `active_ride`         | preload trigger crossed        | windowed mode               | queue next window                          | tile/caption                    | `window.queued`                     |
| `window.queued`       | next window analyzed           | none                        | mark ready                                 | chip                            | `window.ready`                      |
| `window.ready`        | boundary crossed               | next window ready           | activate next window                       | map + caption                   | `window.active(next)`               |
| `active_ride`         | boundary crossed               | next window not ready       | degrade and recover                        | lantern + tile + prompt/chip    | `window.failed_or_stale`            |
| `active_ride`         | push completed                 | push end reached            | write completion, freeze guidance state    | review surface                  | `review.push`                       |
| `review.push`         | reconciliation saved           | expedition continues        | update next assumptions                    | review + planner                | `ready` / `expedition_planned_next` |
| `ready/review`        | owner publishes                | shareable analysis exists   | update public page state                   | public page preview             | `public_page.owner`                 |

## 13.2 Prompt arbitration matrix

| State               | Trigger                 | Guard                      | Action                           | Surface                 | Next state                           |
| ------------------- | ----------------------- | -------------------------- | -------------------------------- | ----------------------- | ------------------------------------ |
| `resume_pending`    | mismatch                | GPS materially far         | show blocking mismatch prompt    | overlay/modal           | `resume_mismatch`                    |
| `active_ride`       | stop passed unconfirmed | stop exists                | show stop-skipped prompt/caption | tile/caption            | `active_ride.stop_resolution_needed` |
| `active_ride`       | plan drift mild         | plan exists                | passive drift caption            | tile + lantern          | `active_ride.drifting_plan`          |
| `active_ride`       | constraint drift severe | hard constraint exists     | urgent drift prompt              | lantern + tile + prompt | `active_ride.recovery_required`      |
| `active_ride`       | boundary near           | windowed mode              | heads-up caption                 | tile/caption            | `active_ride.boundary_heads_up`      |
| `ready/review`      | low confidence          | confidence below threshold | show sticky chip                 | chip + drawer detail    | `ready.low_confidence`               |
| `planning/review`   | share eligible          | owner + fresh enough       | passive share CTA                | review/public page      | state unchanged                      |
| `stop/break/review` | input eligible          | request cooldown clear     | offer structured observation     | caption -> sheet        | state unchanged                      |

------

# 14. Open questions / recommended deferrals

| Item                                                     | Launch call                              | Why                                                          | Blocks MVP?                                    |
| -------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| full Field Notes                                         | defer                                    | launch brief already constrains inputs to structured observations | no                                             |
| public crowd contributions on route pages                | defer                                    | spam/governance mess on day one                              | no                                             |
| exact ride computer tile roster from `DS-012`            | finalize before UI build                 | policy can define categories now, but not final tile IDs/order blindly | yes for final Lovable UI, no for policy engine |
| exact POI category ordering from `PROD-010`              | finalize before mode polish              | mode defaults need taxonomy truth                            | no for core engine                             |
| exact review surface structure from `PROD-012`           | finalize before review UI build          | policy can define entry points now                           | no for core engine                             |
| exact push-intelligence copy from `ADR-036` / `PROD-014` | validate before copy freeze              | behavior spec can stand without final copy deck              | no                                             |
| automatic caution suggestion                             | defer                                    | too noisy before you have good confidence logic              | no                                             |
| cross-device live session sync                           | defer                                    | DS-014 explicitly keeps this out of v1                       | no                                             |
| power-based required watts guidance                      | keep hook, hide unless real model exists | don’t fake physiology                                        | no                                             |
| public live ride sharing                                 | defer                                    | violates calmness/privacy and complicates public-page contract | no                                             |
| exact drift thresholds                                   | config, not code constants               | needs field tuning                                           | no                                             |
| exact mismatch tolerance                                 | config, tuned after dogfooding           | route geometry/GPS reality will vary                         | no                                             |

------

# Recommended file/module breakdown

Do this as a clean runtime layer, not as drawer logic with a fancy name.

## Domain / contracts

- `src/lib/experience-policy/axes.ts`
  - all enums and illegal-coupling rules
- `src/lib/experience-policy/types.ts`
  - `ExperienceSnapshot`, `ExperienceDecision`, `PromptDecision`, `SurfaceDecision`
- `src/lib/experience-policy/config.ts`
  - thresholds, cooldowns, gating flags
- `src/lib/experience-policy/mode-profiles.ts`
  - `rando`, `ultra_endurance`, `road` defaults
- `src/lib/experience-policy/audience-profiles.ts`
  - `user`, `power_user`, `admin_debug`

## Runtime orchestration

- `src/lib/experience-policy/runtime-snapshot.ts`
  - normalize current app/domain state into one snapshot
- `src/lib/experience-policy/runtime-machine.ts`
  - compute `runtime_context`, `ride_state`, `analysis_state`, `resume_state`, `guidance_state`
- `src/lib/experience-policy/resume-policy.ts`
  - DS-014 resume contract
  - mismatch detection
  - one-tap resume vs manual reposition
- `src/lib/experience-policy/window-policy.ts`
  - window preload, boundary behavior, stale/failed window handling

## Prompt / caption layer

- `src/lib/experience-policy/prompt-policy.ts`
  - per-prompt fire/guard/suppress rules
- `src/lib/experience-policy/prompt-arbiter.ts`
  - single foreground prompt selection
  - severity resolution
  - cooldown handling
- `src/lib/experience-policy/prompt-history.ts`
  - suppression ledger / snooze ledger

## Surface routing

- `src/lib/experience-policy/surface-routing.ts`
  - information-to-surface mapping
- `src/lib/experience-policy/review-routing.ts`
  - review entry point selection
- `src/lib/experience-policy/public-page-policy.ts`
  - visitor/owner/admin public page badges, prompts, allowed actions

## Input policy

- `src/lib/experience-policy/input-policy.ts`
  - eligibility rules for speed/shoulder/caution/stop requests
- `src/lib/experience-policy/input-queue.ts`
  - local-first queue and sync retry

## UI adapters

- `src/adapters/experience/map-adapter.ts`
- `src/adapters/experience/lantern-adapter.ts`
- `src/adapters/experience/tile-adapter.ts`
- `src/adapters/experience/drawer-adapter.ts`
- `src/adapters/experience/public-page-adapter.ts`

These adapters should read `ExperienceDecision` and push state into existing UI systems:

- `LayoutContext`
- `LanternState`
- `Orb Control Router`
- `Map Card Store`

They should **not** recompute policy.

## Hook

- `src/hooks/useExperienceRuntime.ts`
  - one hook to compute snapshot + decision + dispatch adapter outputs

------

# Lovable implementation plan

## First

Build the **policy skeleton** only.

That means:

- enums
- runtime snapshot
- runtime machine
- prompt arbiter
- surface routing
- adapter interfaces

Do **not** start with UI polish. Give the app one brain first.

## Second

Wire **expedition/window/resume** behavior to DS-014.

That means:

- durable expedition reads
- resume eligibility
- mismatch flow
- window preload / boundary / stale handling

This is the hardest launch behavior and the one most likely to rot if left local.

## Third

Wire **launch prompts, structured input requests, and public page behavior**.

That means:

- drift
- stop skipped
- stale/confidence chips
- speed/shoulder/caution input eligibility
- owner share prompts
- audience-depth differences

Then start deleting local conditional logic from `RouteMap`, drawers, and ride overlays.

That’s the right order. Not because it’s pretty. Because it keeps you from building a haunted house of conditionals and calling it architecture.


---

## Source File: docs/02-architecture/design/ds-017-truth_resolution_and_propagation_spec.md

# DS-017 — Truth Resolution & Propagation Specification

Status: Draft (authoritative once implemented)
Date: 2026-04-XX

------

## 1. Purpose

This document defines **exactly how Lanterne determines and propagates truth** for:

- speed
- shoulder
- bike infrastructure
- traffic behavior

This is not conceptual. This is operational.

If ADR-032 defines the ingredients, and ADR-042 defines the rules, this document defines **how the system actually behaves in code**.

------

## 2. Core Principle

The system maintains:

> **one canonical truth per segment, derived deterministically from evidence**

Truth must be:

- explainable
- reproducible
- stable under identical inputs

User contributions do not directly modify truth.

------

## 3. Evidence Sources (Ordered by Decreasing Trust)

The following evidence sources are used to determine canonical truth.

They are listed in **strict order of precedence**, from most trusted to least trusted.

Higher-priority sources always override lower-priority ones.

------

### 0. measured (sensor-derived)

Measured data represents direct, instrumented observations of the road.

Examples:

- Varia radar pass data
- DOT speed sensors
- traffic loop detectors

Characteristics:

- objective
- repeatable
- time-aware
- often high-volume

Notes:

- this is the closest thing to ground truth
- values are typically aggregated (median / trimmed mean), not single samples
- propagation requires sufficient sample density

------

### 1. observed (admin / validated)

Observed values are manually confirmed by a trusted source.

Examples:

- founder/admin overrides
- validated field observations

Characteristics:

- high confidence
- sparse
- static snapshot

Notes:

- used when we explicitly know something is true
- overrides all posted and inferred data

------

### 2. authoritative_posted (DOT / HPMS)

Official posted values from government or authoritative datasets.

Examples:

- DOT speed limits
- HPMS-derived posted values

Characteristics:

- structured
- reliable
- occasionally outdated or generalized

Notes:

- considered stronger than OSM
- may not reflect real-world driving behavior

------

### 3. osm_posted

Posted values derived from OpenStreetMap tags.

Examples:

- `maxspeed=35`
- `maxspeed=25 mph`

Characteristics:

- community-maintained
- widely available
- variable accuracy

Notes:

- often correct, but not guaranteed
- can lag real-world changes

------

### 4. observation_inferred

Values propagated from nearby **observed or measured segments**.

Examples:

- a confirmed 45 mph segment extends across a continuous road

Characteristics:

- derived from high-confidence evidence
- not directly observed on the segment itself

Notes:

- carries strong signal, but weaker than direct observation
- must be clearly labeled as inferred

------

### 5. authoritative_inferred

Values inferred from authoritative datasets.

Examples:

- DOT functional class implying typical speed
- HPMS-based estimates

Characteristics:

- structured inference
- consistent but generalized

Notes:

- stronger than OSM inference
- weaker than direct posted values

------

### 6. osm_inferred

Values inferred from OSM attributes.

Examples:

- highway type → estimated speed
- lane count → inferred traffic behavior

Characteristics:

- heuristic-based
- widely applicable
- often inaccurate in edge cases

Notes:

- fallback when no better data exists

------

### 7. regional_prior

Regionally derived baseline values.

Examples:

- typical residential speed in New Jersey suburbs
- state-level speed norms

Characteristics:

- aggregated from known data
- improves baseline realism

Notes:

- stronger than generic baseline
- adapts to geography
- detailed admission and limiter rules for speed priors live in [DS-022](./ds-022-speed_prior_and_area_baseline_policy_spec.md)

------

### 8. highway_area_baseline

Baseline derived from:

- highway classification
- urbanicity (rural / suburban / urban)

Examples:

- suburban arterial → 35 mph
- rural secondary → 45 mph

Characteristics:

- structured but generic
- context-aware

Notes:

- better than pure highway baseline
- still a fallback
- detailed admission and limiter rules for speed priors live in [DS-022](./ds-022-speed_prior_and_area_baseline_policy_spec.md)

------

### 9. highway_baseline

Pure highway-type-based default.

Examples:

- residential → 25 mph
- primary → 45 mph

Characteristics:

- simple
- universal
- lowest fidelity

Notes:

- used only when no better information exists
- considered last-resort input

------

## Summary

The system always selects the highest-confidence available evidence.

Lower-confidence sources are only used when stronger evidence is absent.

This ordering ensures:

- deterministic behavior
- explainable results
- progressive improvement as better data becomes available

------

## 3.5 Measured Evidence (Highest Priority)

Measured evidence represents direct sensor-derived data.

Examples:

- Varia radar pass frequency and speed
- DOT speed sensors
- traffic loop detectors

------

### Characteristics

Measured data is:

- objective
- repeatable
- timestamped
- potentially dense (multiple samples over time)

------

### Priority

Measured evidence is the highest-priority source.

It overrides:

- observed
- posted
- inferred
- baseline

------

### Aggregation

Measured data is not used as a single value.

Instead, it is aggregated into:

- median or trimmed mean
- distribution-aware metrics (future)

------

### Propagation

Measured values propagate along the road similarly to observed values but:

- decay more cautiously
- require sufficient sample density to extend influence

------

### Future Use

Measured data may support:

- traffic exposure modeling
- pass frequency estimation
- driver behavior metrics

This enables a transition from inferred traffic behavior to directly measured reality.

------

### Design Principle

Measured data represents what actually happens on the road.

It is the closest approximation to ground truth available to the system.

------

## 3.6 Measured Evidence — Aggregation & Promotion

Measured data is the highest-trust input in the system.

However, a single measurement is not truth.

The system must distinguish between:

- raw measured events
- aggregated measured truth

------

### Raw Measured Events

A measured event represents a single real-world interaction.

Examples:

- one Varia radar pass
- one DOT sensor reading
- one speed capture

These are stored individually and must NOT directly influence canonical truth.

------

### Aggregated Measured Truth

Measured truth is derived from a sufficient number of measured events.

It represents the **typical real-world behavior** on a segment or road.

------

### Promotion Criteria (v1)

Measured data may be promoted to canonical truth only when ALL conditions are met:

- at least 10 measured events
- at least 3 distinct ride sessions
- (if multi-user data exists) at least 2 unique users
- data is not highly unstable (see spread rules below)

------

### Aggregation Method

Measured values are aggregated using:

- median (not mean)
- rounded to nearest 5 mph

Example:

Raw values:
28, 30, 31, 29, 60

→ median = 30
→ canonical = 30 mph

------

### Spread / Stability Guardrail

Measured data must be reasonably consistent.

Do NOT promote if:

- p75 - p25 > 15 mph

This prevents:

- mixed-context contamination
- outlier-driven distortion
- poor segment grouping

------

### Propagation Behavior

Once promoted:

- measured truth behaves like observed truth
- it propagates along the road per standard rules
- downstream segments are labeled as measured_inferred

------

### Naming & Storage

Measured data should be stored as:

- raw events (event-level)
- aggregated value (segment/road-level)

Suggested fields:

- measured_vehicle_speed_mph
- measured_pass_count
- measured_confidence

------

### Important Distinction

Measured vehicle speed is NOT the same as posted speed.

It represents:

> how vehicles actually behave on this road

This may differ significantly from:

- DOT posted limits
- OSM maxspeed tags

------

### Interaction with Other Evidence

Measured truth:

- overrides observed
- overrides posted
- overrides inferred
- overrides baseline

Measured truth is only superseded by:

- higher-quality measured data
- explicit admin override

------

### Design Principle

One measurement is an event.

Enough measurements become reality.

------

## 4. Persistence Model

For each segment field (speed, shoulder, bikeInfra, traffic), we persist:

- value
- source_type
- confidence (implicit via source_type)
- segment_id

We do NOT persist:

- propagation chains
- inferred relationships between segments

Propagation is recomputed deterministically.

------

## 5. Road Continuity Definition

Propagation operates along **continuous road identity**, not raw OSM segmentation.

Continuity is defined by:

- road name continuity (primary signal)
- geometric continuity (secondary)
- directional coherence (tertiary)

------

### Continuity is broken when:

- road name changes
- strong conflicting evidence appears
- explicit override exists

------

### Continuity is NOT broken by:

- highway type changes alone
- minor OSM segmentation splits
- surface tag differences

------

## 6. Truth Dominance Model

When a strong value exists on a road:

> that value becomes the working truth along that road until contradicted

This is not optional. This is the system behaving like the real world.

------

Example:

- one confirmed 45 mph segment on a rural road
  → assume 45 mph until proven otherwise

## 6.5 Source Labeling During Propagation

Propagation carries value, not source.

When a value propagates from a segment with direct evidence:

- the originating segment retains its source (e.g. observed, DOT posted, OSM posted)
- downstream segments are labeled as inferred from that source

Examples:

- observed → observation_inferred
- authoritative_posted → authoritative_inferred
- osm_posted → osm_inferred

This ensures provenance remains accurate and prevents misleading UI.

------

## Rule

No propagated segment may retain a direct evidence label unless it is directly supported by that evidence.

------

## 7. Propagation by Dimension

Each dimension behaves differently. This is intentional.

------

### 7.1 Speed

**Behavior: highly stable**

- propagates long distances
- assumed constant across named road segments
- only breaks on:
  - road name change
  - stronger evidence (DOT/OSM posted)
  - explicit override

------

### 7.2 Shoulder

**Behavior: context-dependent**

- rural:
  - propagates long distances
- suburban:
  - propagates moderately
- urban:
  - decays quickly

------

Rule of thumb:

- shoulder is assumed continuous until contradicted
- but is more fragile than speed

------

### 7.3 Bike Infrastructure

**Behavior: discontinuous**

- decays aggressively
- does not propagate far
- resets frequently

------

Example:

- bike lane ends at intersection → do not carry forward

------

### 7.4 Traffic Behavior

**Behavior: semi-stable**

- influenced by:
  - road class
  - geography
  - corridor structure

------

Rules:

- carries across similar road segments
- decays faster in:
  - urban grids
  - high intersection density areas
- reinforced by regional priors

------

## 8. Urbanicity Adjustment

Propagation distance depends on context.

We define three bands:

- rural
- suburban
- urban

------

### Rough heuristics:

- rural:
  - low intersection density
  - long continuous roads
  - long propagation allowed
- suburban:
  - moderate intersections
  - moderate decay
- urban:
  - frequent intersections
  - aggressive decay

------

Urbanicity is inferred from:

- intersection density
- road network density
- classification mix

------

## 9. Regional Priors

Regional priors exist to improve baseline inference.

Examples:

- typical speeds by state
- road-class norms
- traffic patterns

------

Rules:

- stronger than baseline
- weaker than direct evidence
- used when no better signal exists

------

## 9.5 Regional Speed Inference Model

Regional priors improve baseline speed estimation.

------

### Model Structure

For each:

- state
- urbanicity band (rural / suburban / urban)
- highway type

Maintain:

- rolling mean speed
- rounded to nearest 5 mph

------

### Update Rule

Each new observed or promoted value updates:

- rolling mean using weighted averaging
- outliers beyond ±15 mph ignored

------

### Minimum Data Threshold

- fewer than 3 samples → use baseline
- 3+ samples → use regional prior

------

### Priority

Regional prior is:

- stronger than highway baseline
- weaker than inferred or observed values

------

### Purpose

Replace naive highway-type-only inference with geographically realistic values.

------



## 10. Conflict Resolution

When multiple values exist:

1. choose highest-priority evidence
2. if equal:
   - prefer closer segment
   - prefer consistent propagation chain

------

Conflicts do NOT blend.

There is always a single resolved truth.

------

## 11. User Observations

User observations:

- are stored separately
- are session-applied immediately
- do NOT modify canonical truth
- do NOT affect scoring

------

They may:

- override display locally
- be aggregated later

------

## 11.5 Observation Promotion to Canonical Truth

User observations may be promoted to canonical truth through aggregation.

------

### Promotion Criteria (v1)

An observation set may be promoted when:

- at least N authenticated users (initially N = 3)
- values agree within tolerance:
  - speed: ±5 mph
  - categorical values: exact match

------

### Aggregation Scope

Observations are aggregated across:

- continuous road identity
- not individual segments

This ensures propagation-consistent counting.

------

### Promotion Result

When promoted:

- source_type becomes "observed"
- value becomes canonical truth
- propagation rules apply from that point

------

### Constraints

- anonymous observations do not count toward promotion
- conflicting observations delay promotion
- promotion must be deterministic

------

### Design Principle

User input becomes truth only after sufficient agreement.

Truth is earned, not assumed.



## 12. Presentation Contract

The UI must clearly distinguish:

- canonical value
- source (OSM / DOT / inferred / baseline)
- user observation (if present)

------

Example:

- “35 mph (OSM)”
- “45 mph (your report)”

------

## 13. Scoring Contract

Only canonical truth feeds:

- safety score
- heatmap coloring
- route metrics

------

User observations must NOT influence:

- score
- color
- ranking

------

## 14. Non-Goals

This spec does NOT define:

- observation aggregation
- ML prediction
- cohort-based comparison logic
- rider-specific personalization

------

## 15. Test Scenarios (Minimum Set)

The system must behave correctly under:

- single observed speed on long rural road
- conflicting OSM vs DOT speeds
- suburban shoulder appearing/disappearing
- urban bike lane discontinuities
- highway type change with same road name
- road name change with same highway type
- user observation present (no canonical change)

------

## 16. Design Principle

The system should behave like a rider who knows how roads actually work:

- speed doesn’t randomly change every 200 meters
- shoulders usually continue… until they don’t
- bike lanes disappear abruptly
- traffic patterns follow geography, not arbitrary segmentation

------

## 17. Final Rule

> Truth is not averaged.
> Truth is not guessed blindly.
> Truth is derived, propagated, and corrected over time.

------

#### END


---

## Source File: docs/02-architecture/design/ds-018-viewport_overlay_hydration_and_client_budget_spec.md

# DS-018 — Viewport Overlay Hydration and Client Budget Spec

Status: Draft  
Date: 2026-04-11  
ADR Parent(s): ADR-027, ADR-029, ADR-030

## Purpose

This document defines the client-side hydration model for explore-mode and ride-mode road overlays.

The goal is to make overlays:
- responsive on mobile hardware
- truthful about what is currently loaded
- progressively useful after map jumps
- stable under panning and ride-mode movement

This spec exists because the previous hydration model could functionally work while still overwhelming the phone:
- too many retained roads
- too many merged/display candidates
- sluggish touch interaction
- visible lag after major map jumps

## 1. Scope

This spec governs viewport-driven overlay hydration for:
- speed overlay
- bike facilities overlay
- future shoulder overlay
- future traffic overlay

It does not govern:
- route analysis scoring
- canonical route load
- route paint truth generation
- admin/debug payload contracts

## 2. Core Principle

Overlay hydration is a separate client concern from route analysis.

It must be treated as:
- viewport-first
- mode-aware
- budgeted
- progressively hydrating

It must not assume:
- a route already exists
- previously loaded roads can accumulate forever
- the phone can hold an arbitrarily large road universe

## 3. Universal Hydration Behaviors

All major viewport relocations must use the same policy, regardless of trigger source.

Examples:
- search result jump
- GPS recenter jump
- future jump-to-segment actions

These are the same class of event:
- the active viewport world changed materially
- stale prior coverage should no longer dominate
- first visible overlay value must appear quickly

### 3.1 Major jump policy

On a major viewport jump:

1. invalidate the local coverage window
2. keep any route-seeded analysis state separate
3. fetch a small landing zone first
4. hydrate the visible area progressively
5. expand outward afterward

This behavior must be universal.

Search and GPS must not diverge into separate hydration logic.

## 4. Moving Local Window

The client must treat hydrated overlay roads as a moving local window, not an endlessly cumulative history.

### 4.1 Retention rule

Retain only:
- the current viewport
- a near-ring around the viewport
- a very small recent trailing footprint if needed

Evict:
- distant prior windows
- historical overlay roads no longer relevant to the current local view

### 4.2 Why

Without explicit eviction, the client accumulates:
- raw road count
- merged fragment count
- render pressure
- touch/input lag

This is unacceptable on phone hardware.

## 5. Client Budget Policy

The client must operate under explicit budgets.

The budgets are separate for:
- hydrated raw roads
- eligible roads for the current overlay
- merged display roads
- rendered roads

### 5.1 Budget classes

The controller should track at least:

- `hydratedRawRoadCount`
- `eligibleOverlayRoadCount`
- `mergedDisplayRoadCount`
- `renderedRoadCount`
- `evictedRoadCount`

### 5.2 Mobile-first rule

Mobile budgets must be stricter than desktop budgets.

Mobile should use:
- smaller retained radius
- smaller landing-zone fetch
- smaller green-road chunk size
- earlier eviction
- lower maximum rendered road count

### 5.3 Hard rule

Overlay fidelity must degrade before map responsiveness degrades.

That means:
- trim overlay scope first
- drop lower-priority road classes next
- evict distant windows before the app becomes touch-sluggish

## 6. Progressive Hydration Model

The client must not wait for one large viewport fetch to finish before painting.

Hydration should occur incrementally as usable road groups become available.

### 6.1 Landing zone first

After a major jump, the first hydration should prioritize:
- the visible center of the viewport
- the smallest meaningful on-screen coverage

The objective is time-to-first-useful-overlay, not total coverage.

### 6.2 Expansion second

After landing-zone hydration:
- expand outward into the near-ring
- bias toward the movement direction during panning / ride mode

### 6.3 Progressive merge

As road groups arrive:
- merge them immediately into the hydrated cache
- let overlays re-render incrementally
- avoid one giant all-at-once reveal

## 7. Priority-Class Hydration

Hydration should prioritize road classes by rider planning value, not only by geography.

### 7.1 Priority order

Current desired order:

1. `blue`
2. `orange`
3. `red`
4. `green`
5. `unknown`

### 7.2 Why this order

`blue`
- highest route-building value
- safest or most infrastructure-rich roads

`orange` and `red`
- smaller sets
- useful quickly as avoid/constraint structure

`green`
- largest set
- main performance hog
- must be chunked aggressively

### 7.3 Green-road policy

`green` roads must be treated as the bulk class:
- chunked more aggressively
- retained more locally
- evicted sooner than higher-value classes

Outside the immediate viewport neighborhood, the controller may retain:
- blue
- orange
- red

while pruning far more green.

## 8. Geographic Chunking Policy

Chunking should be geographically local.

It should not use large slabs of geography.

Preferred model:
- center-visible roads first
- tiny nearby areas next
- directionally biased continuation afterward

The UX target is:
- smooth trickle from the user’s actual screen area
- not county-scale or neighborhood-scale delayed reveal

## 9. Mode Awareness

### 9.1 Explore mode

Explore mode should:
- always hydrate from current viewport when overlays are active or prewarming is allowed
- support major jump reset
- continue hydration during pan

### 9.2 Ride mode

Ride mode should use:
- larger retained window
- stronger forward bias
- more lateral padding for heading changes

Ride mode must assume:
- the viewport moves continuously
- map rotation exposes corners
- an undersized coverage window will be visually obvious

### 9.3 Route creation mode

Route creation must be allowed to suspend background overlay hydration when necessary.

Reason:
- route creation analysis should not compete with non-critical overlay prewarm right before analysis begins

## 10. Separation of Duties

Overlay hydration must remain distinct from:
- loader presentation
- route analysis readiness
- route analysis paint

This means:

### 10.1 Loader

Loader owns:
- progress UI
- narrative
- telemetry presentation

Loader does not own:
- overlay hydration
- route paint visibility

### 10.2 Route analysis paint

Route analysis paint owns:
- analyzed on-route truth display
- route hazards tied to analysis

It does not own:
- off-route speed overlay
- bike facilities explore overlay

### 10.3 Overlay hydration controller

Overlay hydration owns:
- viewport-driven road fetching
- jump reset behavior
- chunk ordering
- progressive merge
- eviction/budget enforcement

## 11. Instrumentation Requirements

The controller must log, behind gated diagnostics:

- reset reason
- landing-zone fetch start
- expansion fetch start
- chunk priority
- chunk size
- hydrated raw road count
- rendered road count
- eviction count
- budget trims

These logs should make it obvious when:
- a jump did not reset
- the client is retaining too much
- green-road accumulation is driving lag

## 12. Current Known Limits

As of this spec:

- the system can now hydrate after jumps more correctly than before
- but first-jump time-to-visible-overlay is still too slow in dense areas
- pan-follow still degrades once too many roads are retained
- blue classification in speed overlay still needs stricter filtering before it deserves premium priority treatment

## 13. Implementation Direction

The next implementation passes should prioritize:

1. explicit client budget enforcement
2. moving-window eviction
3. tighter mobile caps
4. further improvement of first visible landing-zone latency
5. stricter blue-road filtering in speed overlay

## 14. Done For Now

This system is in a good state for the current phase when:

- major jumps hydrate the new area without needing a route hack
- overlays begin appearing quickly in the new area
- panning continues to follow without retaining huge stale universes
- mobile interaction remains responsive
- route analysis paint remains isolated from overlay hydration state

## 15. Route Paint vs Viewport Overlay Risk Semantics

This spec intentionally allows route analysis paint and viewport road overlays to use different computation contracts.

That is not architectural split-brain if the boundary is explicit.

### 15.1 Analyzed route paint

The selected / analyzed route should be grounded in canonical route-analysis truth.

This means:
- the route line uses the route's score-bearing units, trace, rollup, or cached canonical analysis artifact
- route paint is allowed to communicate route risk because the route has direction, continuity, matched context, crossings, chosen inputs, confidence, and provenance
- route paint should not silently fall back to a cheaper viewport overlay proxy once a canonical analyzed-route artifact exists

### 15.2 Viewport road overlays

Viewport overlays are not canonical route scoring.

They answer a different question:

> What kind of road environment is visible around the rider or planner right now?

They may therefore use fast proxy classifications such as speed band, facility class, traffic availability, or road-stress heuristics instead of computing full route risk for every visible road.

Reason:
- full route risk is expensive to compute across every visible road in a dense viewport
- off-route roads do not yet have route direction, maneuver context, route-specific crossing events, continuity, or selected ride intent
- computing canonical-style risk for the entire viewport can make the map unusable

The performance rule is:

> Viewport overlay fidelity must degrade before map responsiveness degrades.

### 15.3 Naming rule

The product and code should avoid calling the viewport proxy a canonical risk heatmap unless it is backed by canonical score-bearing artifacts.

Preferred distinction:
- **route risk paint**: analyzed route, canonical score-bearing truth
- **road-stress overlay** or **road environment overlay**: viewport proxy, budgeted and progressively hydrated

### 15.4 Future direction

If deeper off-route risk is needed later, it should be one of:
- precomputed tile / slice artifacts served from cache
- worker-backed lazy computation for a bounded subset of roads
- explicit user-requested analysis of a candidate alternate route

It should not be recomputed synchronously for every visible road on every overlay activation.


---

## Source File: docs/02-architecture/design/ds-019-score_tracing.md

# DS-019 — Score Tracing

**Status:** Accepted as architectural context / future implementation spec
**Date:** 2026-04-12
**Filename:** `ds-019-score_tracing.md`

## Purpose

This document defines the future **Score Tracing** system for Lanterne.

Score Tracing is not a cosmetic explainer. It is the system by which Lanterne proves:

- what inputs the score used

- where those inputs came from

- how those inputs were transformed

- what segment, crossing, and route-level contributions were produced

- what confidence and fallback assumptions shaped the result

  Score Tracing exists to support:

- reliability

- debugging

- founder auditability

- calibration review

- future change summaries

- rider trust for those who want deeper visibility

  This feature is **not planned for the current implementation pass**. It is included now because the scoring engine should be built with tracing in mind rather than retrofitted later.

## Core principle

Lanterne’s Safety Score must never become a black box.

If a route receives a score, the system should be able to answer:

> **How was that score made?**

And not with a marketing paraphrase. With the actual derivation.

## What score tracing is

Score Tracing is a **structured derivation artifact** attached to a route analysis.

It records:

1. **input facts**
2. **input provenance**
3. **input confidence**
4. **intermediate transforms**
5. **segment math**
6. **crossing-event math**
7. **route rollup math**
8. **final score normalization**
9. **warnings, fallbacks, and caveats**

## What score tracing is not

Score Tracing is **not**:

- a simplified rider explainer only
- a screenshot of the score drawer
- a reconstruction generated later from partial output
- a heatmap legend
- a JSON dumpster for arbitrary debug scraps

## Design decision

Score Tracing should be generated from the **canonical scoring pipeline** at analysis time.

It should not require rerunning full analysis when a rider opens the trace view.

### Why

Post-hoc reconstruction is brittle:

- chosen inputs may be lost

- fallback decisions may be unrecoverable

- display segments may not match scoring slices

- later code changes may make an older score hard to fully explain

  The scoring engine should therefore emit trace data while the full derivation context is still in hand.

## Trace scope

Score tracing attaches to the **canonical scored analysis artifact**, not just the current UI session.

That means it belongs conceptually to:

- route version

- analysis version

- scoring model version

  not merely “what is on screen right now.”

## Trace layers

### 1. Route header

Every trace begins with route-level metadata:

- route identifier
- route version
- analysis version
- scoring model version
- computed timestamp
- route miles
- match quality
- route confidence summary
- whether the result is partial, provisional, or complete

### 2. Input summary

A compact summary of what the score was built from:

- miles with official traffic truth
- miles with inferred traffic truth
- miles using baseline traffic proxy
- miles with tagged or official speed truth
- miles with inferred speed truth
- miles with facility truth
- miles with shoulder truth
- count of scored crossings
- count of low-confidence crossings

### 3. Continuous segment trace

For each canonical score-bearing slice:

- segment index
- start / end distance
- slice length
- chosen speed input
- chosen traffic input
- infrastructure state
- shoulder state
- provenance for each
- confidence for each
- speed factor
- traffic factor
- infrastructure factor
- shoulder factor
- resulting continuous slice risk

### 4. Crossing trace

For each score-bearing crossing event:

- event index
- route distance
- crossed or entered road id / name if known
- speed context
- traffic context
- lanes crossed
- control type
- movement type
- provenance for each
- confidence for each
- event formula inputs
- event contribution before cap
- event contribution after cap
- any uncertainty notes

### 5. Route rollup trace

The full route-level derivation:

- total continuous risk
- continuous risk per mile
- raw crossing burden per mile
- effective crossing contribution per mile after saturation
- raw route risk per mile
- logistic midpoint and steepness
- final Safety Score
- letter grade
- route-level caveats

### 6. Warnings and caveats

An explicit list of what weakened interpretability:

- low match quality
- inferred traffic on important sections
- missing control truth at major crossings
- estimated lanes crossed
- baseline traffic proxy usage
- partial analysis
- stale external data if relevant

## Canonical trace rules

### Rule 1 — Trace canonical scoring slices, not display segments

The trace must anchor to the **actual scoring slices / canonical score-bearing units**, not the merged display segments used for map rendering.

Why:

- display segments may merge multiple truths
- display logic may change by zoom or mode
- score math must remain explainable independent of presentation paint

### Rule 2 — Trace the chosen input, not every discarded candidate

The trace should record:

- the winning chosen value

- provenance

- confidence

- optional fallback note

  It should not dump every discarded candidate unless a deeper debug variant explicitly asks for that.

### Rule 3 — Do not hide fallback usage

If a value was inferred, predicted, baseline-derived, or neutral-filled, the trace must say so.

### Rule 4 — Unknowns must be explicit

Unknown should appear as:

- the score-handling rule used

- provenance class

- confidence effect

  never as a silent substitution only.

## Future UI surfaces

### Rider-level entry point

The analysis drawer may eventually include an entry such as:

- **Score Trace**

- **Scoring Trace**

- **View Score Derivation**

  This should remain clearly secondary to the main ride-planning UI.

### Rider-facing trace view

The rider-facing trace should default to:

- formatted sections
- collapsible groups
- plain-English labels
- visible assumptions
- expandable raw math

### Founder / admin / debug variant

A deeper variant should support:

- raw formulas
- exact constants
- field provenance
- field confidence
- unresolved assumptions
- segment and crossing tables
- exportable JSON

## Trace persistence strategy

### Phase 1

In-memory trace for the currently analyzed route.

### Phase 2

Persist route-level trace summary plus structured trace metadata.

### Phase 3

Persist full trace artifact or bounded trace JSON for route analyses where needed.

The exact storage model may evolve, but the trace schema should be designed now so the scoring engine can emit it consistently.

## Relationship to confidence and provenance

Score Tracing depends on the confidence and provenance model.

Trace must not merely show:

- value used

  It must also show:

- **where it came from**

- **how much the system trusted it**

- **whether it was fallback-derived**

  Without provenance and confidence, the trace is incomplete theater.

## Relationship to future change summaries

Score tracing is the raw material for future **analysis change summaries**.

Later systems may compare:

- prior analysis trace

- new analysis trace

  to produce:

- score delta reason

- traffic coverage improvement notes

- new hotspot notes

- changed crossing interpretation notes

  That comparison layer is future work. Score tracing is the prerequisite.

## Output contract

At minimum, a complete future score trace must answer:

### For continuous exposure

- what speed was used?
- what traffic figure was used?
- where did those come from?
- how were they transformed?
- what slice risk resulted?

### For crossings

- why was this node counted?
- what kind of movement was it?
- what lanes, control, speed, and traffic assumptions were used?
- what event contribution resulted before and after the per-event cap?

### For route rollup

- how much of the total came from continuous exposure?
- how much came from crossings after route-level saturation?
- what final raw route burden fed the logistic curve?
- what score and grade came out?

## Non-goals

This spec does **not** require:

- implementing the trace UI now
- persisting full raw trace JSON in the current pass
- exposing every debug detail to general riders
- freezing the final storage schema today
- turning the score drawer into a cockpit

## Acceptance test for future implementation

A score tracing implementation is acceptable only if:

1. it can explain a score without rerunning full analysis
2. it uses canonical scoring slices rather than display segments
3. it shows provenance and confidence for every load-bearing input
4. it makes fallback usage explicit
5. it shows crossing contribution before and after route-level saturation
6. it shows the exact rollup and final normalization steps
7. it can explain why a score changed between analysis versions once change summaries are added

## Design principle

**If Lanterne cannot show how the sausage was made, it has not earned trust.**

Score Tracing is how the product proves that reliability is not branding.
It is architecture.

---

## Source File: docs/02-architecture/design/ds-020-confidence_model_and_tracing.md

# DS-020 — Confidence Model and Tracing

**Status:** Draft for review  
**Date:** April 14, 2026  
**Filename:** `ds-020-confidence_model_and_tracing.md`

---

## 1. Purpose

This document defines the canonical **confidence model** for Lanterne’s narrow Safety Score.

It answers a different question than the score itself.

- **Safety Score** asks:  
  > how risky is this route, given the canonical values the system chose?

- **Confidence** asks:  
  > how confident are we that those chosen values, and therefore the resulting score, are trustworthy?

This is not optional polish.

Confidence must become a first-class modeled system because:

- the score now depends on multiple truth inputs with different provenance quality
- those inputs do not contribute equally to route risk
- route-level trust cannot be inferred honestly from flat field counting
- score trace without confidence trace is only half an explanation

This spec is written to prevent the confidence system from becoming a second-rate heuristic sidecar.

---

## 2. Core principle

Confidence must use the **same canonical variable graph** as score.

That means:

- no second heuristic tree
- no independent “confidence-only” variable universe
- no flat counting of unknowns detached from raw risk contribution

Confidence and score are different outputs, but they are derived from the same underlying route truth.

If score uses:

- speed
- traffic
- facility
- shoulder
- crossing inputs

then confidence must evaluate trust using those same inputs, their chosen provenance, and their actual role in local risk.

### Design rule

> **Confidence must follow the same canonical slices, crossing events, chosen inputs, and propagation rules as score.**

If the score is calculated slice-by-slice and event-by-event, confidence must be calculated slice-by-slice and event-by-event too.

---

## 3. Scope

This spec governs confidence for the canonical narrow Safety Score only.

It applies to the first five canonical safety input families:

1. **speed**
2. **traffic**
3. **facility**
4. **shoulder**
5. **crossing inputs**
   - crossed-road speed
   - crossed-road traffic
   - crossing width / lanes crossed
   - control
   - movement

It does **not** yet govern confidence for:

- hazards
- weather / conditions
- remoteness
- fatigue
- route difficulty
- comparative / cohort context
- user reports as route-level modifiers

Those may later gain their own confidence handling, but they are outside this launch spec.

---

## 4. What confidence is and is not

### Confidence is

- a modeled estimate of how trustworthy the score-driving chosen inputs are
- sensitive to **provenance**
- sensitive to **propagation / inference depth**
- sensitive to **raw risk contribution**
- computed at:
  - field level
  - slice level
  - crossing-event level
  - route level

### Confidence is not

- a hidden score modifier
- a moral judgment about OSM or any other data source
- a flat count of “unknown fields”
- a claim that the score is exactly right in the scientific sense
- a generic quality score for the whole route independent of what actually drove risk

### Important product interpretation

Riders often experience confidence as:

> **How confident are we that this route is not riskier than the score suggests?**

That is not identical to:

> **How confident are we that the exact numeric score is perfect?**

This spec still models confidence mathematically, but it recognizes that route-level confidence should be most sensitive to uncertainty in the **inputs that materially drove route risk**, not to incidental uncertainty in low-impact fields.

---

## 5. Output representation

Confidence is represented in both numeric and banded form.

At launch, two route-level confidence outputs are canonical:

1. `road_confidence`
2. `route_confidence`

### 5.1 Numeric output

All confidence outputs use a **0–100** scale.

- **0** = no meaningful trust
- **100** = highest launch confidence

### 5.2 Band output

Launch uses five 20-point bands:

| Confidence score | Band |
|---|---|
| 0–19 | very low |
| 20–39 | low |
| 40–59 | medium |
| 60–79 | high |
| 80–100 | very high |

These bands are product-facing summaries of the numeric confidence values, not separate logic systems.

### 5.3 Route-level output split

#### `road_confidence`

This answers:

> how confident are we in the road-domain slices and crossings that materially shaped the score?

This is the audit-side confidence value.

It stays tightly coupled to:

- road-domain continuous slices
- scored crossings
- actual score participation
- actual local risk

#### `route_confidence`

This answers:

> how confident are we in our overall characterization of the route as a whole?

This is the route-understanding confidence value.

It is allowed to reflect:

- confidently identified benign/path-domain mileage
- broad route-domain understanding
- the fact that a route may be mostly well understood even when a few connectors are weaker

---

## 6. The two-part confidence model

Confidence is composed of two parts:

1. **relative source confidence**
2. **raw risk contribution**

This is the backbone of the whole spec.

### 6.1 Relative source confidence

This answers:

> how trustworthy is the chosen value source?

This part comes from provenance and propagation quality.

### 6.2 Actual score impact

This answers:

> how much did this variable actually matter to the local score on this slice or crossing?

This part comes from the same canonical score graph as the score itself.

For launch confidence weighting, this means **actual score participation**, not “deviation from a neutral baseline.”

If a `25 mph` road slice has:

- `localLikelihood > 0`
- `localSeverity = 1.0`
- `localRisk > 0`

then speed is still participating in actual local risk and may not be assigned zero confidence weight simply because `25 mph` is the severity baseline.

### 6.3 Combined idea

Launch confidence should be driven by the simple principle:

> **confidence effect = source confidence × raw risk contribution**

Equivalently, uncertainty burden is:

> **uncertainty burden = (1 - source confidence) × raw risk contribution**

That means:

- a weak source on a variable that barely mattered should barely matter
- a weak source on a variable that materially drove local score should matter a lot
- a non-operative variable with zero score impact should contribute zero confidence penalty

That is the whole point of the model.

---

## 7. Relative source confidence ladder

This section defines the launch source-confidence ladder for score-driving chosen values.

It is aligned to the DS-017 truth resolution and propagation hierarchy, but translated into explicit launch confidence values.

### 7.1 Launch candidate ladder

| Provenance class | Launch confidence |
|---|---:|
| observed | 100 |
| authoritative_posted | 100 |
| osm_posted | 95 |
| observation_inferred | 90 |
| authoritative_inferred | 90 |
| osm_inferred | 85 |
| regional_prior | calculated per 8.3 + 15 |
| highway_area_baseline | calculated per 8.3 + 10 |
| highway_baseline | calculated per 8.3 |

For dominant variables such as **speed** and **traffic**, a true terminal `unknown` state may still exist when the system genuinely has no defensible chosen value. In launch logic, that state should carry **10% confidence** for that field.

For secondary variables such as **facility** and **shoulder**, non-operative or zero-impact unknowns should not automatically create a major confidence penalty. Their burden should depend on raw risk contribution.

### 7.2 Why OSM posted remains high

OSM posted speed is not treated as weak by default.

If:
- a rider or mapper took the time to verify a posted speed
- contributed it
- and it survived community review into OSM

then that is generally strong evidence.

It is weaker than direct authoritative posted truth, but only slightly.

### 7.3 Why inferred classes still matter

Inferred is not the same as unknown.

A value inferred from:
- nearby observed truth
- nearby authoritative truth
- stable road continuity

is clearly better than a last-resort baseline.

This spec therefore preserves meaningful separation between:

- inferred from strong evidence
- inferred from weak heuristics
- generic baseline fallback

---

## 8. Dynamic baseline confidence for chosen speed

This is one of the key launch design rules.

A flat confidence value for `highway_baseline` or similar fallback speed is too dumb.

Why?

Because the confidence impact of a chosen speed baseline depends on how wrong it could be **in score terms**, not just raw mph terms.

### 8.1 The problem with flat baseline confidence

A chosen speed of:

- **55 mph** inside a **45–65 mph** plausible highway range

is not the same confidence story as:

- **25 mph** inside a **15–35 mph** plausible range

Even if both are centered in a 20 mph band.

The reason is that the severity curve is nonlinear. The risk-impact spread at higher speeds is much larger.

### 8.2 Rule

When a chosen speed comes from a baseline class, confidence should be based on the **risk-impact spread around the chosenSpeed**, not just the raw mph width of the class.

### 8.3 Launch model for baseline-derived speed confidence

For any speed value chosen from a baseline class:

1. identify the plausible low/high speed bounds for that class
2. use the actual **chosenSpeed**, not an abstract midpoint label
3. evaluate the risk-impact spread implied by those bounds around the chosenSpeed
4. convert that spread into a relative source-confidence value for `highway_baseline`
5. derive:
   - `highway_area_baseline = highway_baseline + 10`
   - `regional_prior = highway_baseline + 15`

bounded to the 0–100 scale.

This means baseline confidence is tied to **how wrong the score could be**, not just to raw mph spread.

A practical launch implementation may use the severity function itself as the comparison surface. For example:

- compute `σ(chosenSpeed)`
- compute `σ(lowerBound)` and `σ(upperBound)`
- measure the larger of the two risk-impact deviations
- map larger deviation to lower baseline confidence

### 8.4 Consequence

Two baseline-derived speeds with the same raw class width may receive different confidence if one sits in a much steeper risk-impact regime.

This is intentional, and it is one of the main reasons confidence must follow score logic rather than a flat source ladder.

---

## 9. Variable importance weighting

This section defines how much each score-driving variable family should matter **if it is uncertain**.

This is not a second scoring model. It is the confidence-side weighting of uncertainty burden.

### 9.1 Governing principle

A variable’s confidence weight should come from its actual ability to affect the score.

That means:

- speed should generally carry the strongest confidence weight
- traffic should generally carry the next strongest confidence weight
- facility and shoulder should be secondary
- crossing inputs should be weighted according to their actual contribution in crossing score

### 9.2 Launch variable families

#### Speed
- dominant on the severity side
- usually the most load-bearing confidence variable on risk-driving slices

#### Traffic
- dominant on the likelihood side
- especially important when official DOT/HPMS truth exists

#### Facility
- secondary but real
- meaningful when it materially changes localLikelihood

#### Shoulder
- secondary but real
- meaningful when it materially changes localLikelihood

#### Crossing inputs
- should use the same logic as crossing score:
  - crossed-road speed
  - crossed-road traffic
  - width
  - control
  - movement

### 9.2.1 Launch weighting direction

The launch weighting hierarchy should strongly prefer the variables that dominate score-driving risk in practice.

As a launch rule of thumb:

- **speed** should usually carry the strongest confidence weight on continuous slices
- **traffic** should usually carry the second-strongest confidence weight on continuous slices
- **facility** and **shoulder** should be secondary
- for crossings, **crossed-road speed** and **crossed-road traffic** should usually dominate crossing confidence weighting
- **control**, **movement**, and **width** should matter, but usually less than crossed-road speed/traffic

These are not permission to use a flat static weighting table detached from route context. They are launch directionality rules for the dynamic impact system.

The actual confidence effect of each variable must still be computed from its real route-specific risk contribution.

### 9.3 No flat counting

The model may not treat:
- speed unknown
- bike-lane unknown
- shoulder unknown

as if they have equal confidence importance just because they are all fields.

That is explicitly forbidden.

---

## 10. Fully dynamic impact weighting

Confidence weighting is **fully dynamic** at the slice and crossing level.

That means the confidence effect from a variable is based on the actual route circumstances in which it appears and on the raw risk contribution of that variable there.

### 10.1 Slice intuition example

If bike lanes are only relevant on 5 miles of a 125-mile route, and unknown there, their uncertainty should only damage route confidence in proportion to those relevant miles and their raw risk contribution.

If the remaining 120 miles are path-domain and bike-lane availability is irrelevant there, they should contribute **zero** confidence penalty from bike-lane uncertainty.

### 10.2 Path-domain rule

If a slice has:

- `domainType = path`
- `localLikelihood = 0`
- `localRisk = 0`

then that slice contributes **zero confidence penalty** to route confidence, regardless of incidental uncertainty in road-only inputs.

That rule is critical.

---

## 11. Field-level confidence

Each chosen input should carry a confidence record.

Field-level confidence answers:

- what value was chosen
- where it came from
- how trustworthy that source is before route-specific risk weighting

### 11.1 Canonical shape

Suggested launch shape:

```ts
ConfidenceInput<T> = {
  value: T | null
  provenance: ProvenanceClass
  relativeConfidence: number   // 0–100 from source ladder / baseline logic
  confidenceBand: ConfidenceBand
  fallbackNote?: string
  sourceRef?: string
}
```

### 11.2 Required launch fields

For continuous slices:
- chosenSpeed
- chosenTraffic
- chosenFacility
- chosenShoulder
- chosenCurvature

For crossing events:
- chosenCrossedSpeed
- chosenCrossedTraffic
- chosenWidth
- chosenControl
- chosenMovement

---

## 12. Slice-level confidence math

### 12.1 Continuous slices

A continuous slice should compute confidence from the same variables used in its score.

The score side is:

- `localLikelihood = λ_road`
- `localSeverity = σ(speed)`
- `localRisk = λ_road × σ(speed)`

The confidence side should use the same chosen inputs and ask:

- how confident are we in each chosen value source?
- how much did that variable actually contribute to local risk on this slice?

### 12.2 Launch slice-confidence shape

For each score-driving variable on a slice:

- `effectiveConfidence_i = confidenceRate_i × impact_i`

where:

- `confidenceRate_i` is the field’s relative source confidence on a 0–1 scale
- `impact_i` is that variable’s actual risk-contribution share on the slice, also on a 0–1 scale

Equivalently:

- `uncertaintyBurden_i = (1 - confidenceRate_i) × impact_i`

Then:

- `sliceConfidence = 100 × (1 - Σ uncertaintyBurden_i)`

The key point is that `impact_i` is not a static global constant. It is slice-specific and derived from the same canonical score graph.

### 12.3 Important rule

If a variable had negligible raw risk contribution on that slice, its uncertainty burden should also be negligible.

This is why zero-risk path slices contribute zero confidence penalty even if some road-only fields are unknown there.

---

## 13. Crossing-level confidence math

Crossing confidence should mirror crossing score.

### 13.1 Crossing score structure

- `localLikelihood = λ_cross`
- `localSeverity = σ(crossedSpeed)`
- `localRisk = λ_cross × σ(crossedSpeed)`

### 13.2 Crossing confidence structure

Crossing confidence should use the same ingredients:

- crossed-road speed
- crossed-road traffic
- width
- control
- movement

using the same launch confidence shape:

- `confidence_contribution_i = confidence_pct_i × risk_contribution_i`

### 13.3 Weighting intuition

In most cases, crossing confidence should be driven more by:

- crossed-road speed
- crossed-road traffic

than by:
- control
- movement
- width

But all of them should still be part of the same canonical graph.

---

## 14. Route-level confidence aggregation

This is the most important rollup rule.

### 14.1 Governing principle

Route confidence must be driven by **risk-driving uncertainty burden**, not by raw counts of unknown fields or unknown miles.

### 14.2 Consequence

A route may still earn **high** or **very high** confidence even if some minor fields are unknown broadly, so long as the risk-driving slices and events rely on strong truth.

### 14.3 Launch aggregation shape

Launch uses two different route-level rollups:

1. `road_confidence`
2. `route_confidence`

They use the same field/slice/crossing confidence machinery, but answer different questions.

### 14.4 `road_confidence`

`road_confidence` is the rollup of road-domain slices and scored crossings, weighted by their actual local risk.

Conceptually:

- `road_confidence = weighted rollup of road-domain slice confidences and crossing confidences`
- weights are the units' actual `localRisk`

This should answer:

> how confident are we in the score-driving road slices and crossings that produced the route risk?

This is the canonical audit rollup.

### 14.5 `route_confidence`

`route_confidence` is the rollup of overall route characterization confidence.

It should:

- include road-domain slices
- include path-domain slices
- allow confidently identified benign/path-domain mileage to count positively
- avoid letting a few weaker connectors fully overwrite the route-wide characterization when the route is otherwise strongly understood

Conceptually:

- `route_confidence = weighted rollup of route-characterization confidence across the whole route`

At launch:

- road-domain slices may be weighted by miles, with visibility from local risk
- path-domain slices with `localRisk = 0` may still contribute positively through route-characterization confidence
- scored crossings may contribute a small additive characterization weight or may be reported separately in receipts, as long as the implementation choice is explicit

### 14.6 Governing interpretation

`road_confidence` should answer:

> how well do we trust the risk-driving details?

`route_confidence` should answer:

> how well do we trust our overall understanding of the route?

The exact rollup implementation may evolve, but the governing rules are fixed:

- risk-driving slices/events with strong truth should dominate `road_confidence` upward
- low-impact uncertainty should have limited effect
- zero-risk slices should contribute zero penalty
- a route must not be dragged to low confidence merely because many non-score-driving fields are messy
- confidently known benign/path-domain mileage may raise `route_confidence` without softening `road_confidence`

---

## 15. Confidence bands at slice, crossing, and route level

All three levels use the same five launch bands:

| Score | Band |
|---|---|
| 0–19 | very low |
| 20–39 | low |
| 40–59 | medium |
| 60–79 | high |
| 80–100 | very high |

That keeps the system coherent.

---

## 16. Confidence trace contract

Confidence trace is required.

It should parallel score trace.

### 16.1 Confidence trace must show

- chosen input
- provenance class
- relative source confidence
- fallback path / propagation class
- risk_contribution
- confidence_contribution or uncertainty_contribution
- slice or crossing confidence rollup
- route confidence rollup

### 16.2 Why this matters

Without trace, confidence becomes just another opaque number.

The purpose of confidence trace is to let the system explain:

- what value was chosen
- where that value came from
- how confident the system was in that value
- how much that uncertainty mattered to the slice, crossing, and route

In other words, confidence trace is the trust-side equivalent of score trace.

---

## 17. Reporting and presentation contract

Confidence should appear anywhere the system is making meaningful score claims.

### 17.1 analysisReceipt
Should show:
- route confidence score
- route confidence band
- road confidence score
- road confidence band
- short explanation
- fallback summary
- warnings only when uncertainty is materially score-driving

### 17.2 evidenceSummary
Should show:
- route confidence score + band
- road confidence score + band
- miles by provenance class
- crossing counts by provenance class
- fallback counts
- low-confidence slice/event counts
- reasons for lowered confidence

### 17.3 scoreTrace
Should show:
- slice/event-level confidence alongside score derivation
- confidence-aware decomposition language

### 17.4 Warning rule

Use warnings when:
- low-confidence risk-driving slices are materially large
- major crossings are low-confidence
- official traffic coverage is sparse on score-driving segments

Do **not** use warnings for minor incidental uncertainty with negligible route effect.

---


## 18. Worked examples

These examples are not decorative.

They exist to prove that the confidence model behaves the way the score model intends:

- confidence follows the same canonical variable graph as score
- confidence burden is weighted by raw risk contribution
- zero-risk slices contribute zero confidence penalty
- dominant variables such as speed and traffic matter much more than secondary variables when they actually drive route risk
- route-level confidence is driven by risk-driving uncertainty, not flat field ugliness

These examples are behavior checks, not final implementation test fixtures.

### 18.0 Worked-example convention

For the examples below:

- confidence rates are shown on a **0–100** scale
- risk contributions are also shown on a **0–100** scale
- the operative variable risk contributions for a slice or crossing sum to **100**
- `confidence contribution = confidence rate × risk contribution`
- `slice/crossing confidence = sum of confidence contributions`
- equivalently:
  - `uncertainty burden = (100 - confidence rate) × risk contribution`
  - `confidence = 100 - total burden`

For route examples:
- `road_confidence` is the weighted rollup of the score-driving road-slice/crossing confidences
- `route_confidence` is the weighted rollup of whole-route characterization confidence
- zero-risk path-domain slices contribute **0 confidence penalty** to `road_confidence`
- zero-risk path-domain slices may still contribute positively to `route_confidence`

### 18.1 Field-level example — posted speed vs baseline speed

Two otherwise identical risk-driving road slices each give speed an **80%** risk contribution.

#### Slice A
- chosenSpeed = **45 mph**
- source = **authoritative_posted**
- confidence_pct = **100%**
- risk_contribution = **80**

Effective speed confidence contribution:

- `1.00 × 80 = 80.0`

#### Slice B
- chosenSpeed = **45 mph**
- source = **highway_baseline**
- plausible class range = **35–55 mph**
- risk_contribution = **80**

Severity values from the launch score model:
- `σ(35) = 3.6`
- `σ(45) = 8.5`
- `σ(55) = 16.3`

Compute the risk-impact spread around the chosen speed:

- lower-side deviation = `8.5 - 3.6 = 4.9`
- upper-side deviation = `16.3 - 8.5 = 7.8`
- total severity span across the class = `16.3 - 3.6 = 12.7`

Normalized uncertainty ratio:

- `max(4.9, 7.8) / 12.7 = 7.8 / 12.7 = 0.614`

Launch-calculated baseline confidence for this example:

- `1 - 0.614 = 0.386`
- rounded launch `confidence_pct = 40%`

Confidence contribution:

- `0.40 × 80 = 32.0`

#### Interpretation

Both slices use the same chosen speed value, but the baseline-derived slice gets far less confidence credit because the chosen value is much less trustworthy in risk terms.

The difference is:

- `80.0 - 32.0 = 48.0 confidence points`

That is the intended behavior.

### 18.2 Field-level example — official traffic vs weak fallback traffic

Two otherwise identical road slices each give traffic a **20%** risk contribution.

#### Slice A
- chosenTraffic = **official_per_lane**
- confidence_pct = **100%**
- risk_contribution = **20**

Effective traffic confidence contribution:

- `1.00 × 20 = 20.0`

#### Slice B
- chosenTraffic = **class_proxy**
- plausible class-proxy range for this example = **3,000–8,640 AADT/lane**
- chosenTraffic = **6,000 AADT/lane**
- risk_contribution = **20**

TrafficFactor values from the stronger three-zone launch shape:
- `TF(3,000) = 1.2`
- `TF(6,000) = 2.0`
- `TF(8,640) = 3.0`

Compute the risk-impact spread around the chosen traffic value:

- lower-side deviation = `2.0 - 1.2 = 0.8`
- upper-side deviation = `3.0 - 2.0 = 1.0`
- total TrafficFactor span across the class = `3.0 - 1.2 = 1.8`

Normalized uncertainty ratio:

- `max(0.8, 1.0) / 1.8 = 1.0 / 1.8 = 0.556`

Launch-calculated baseline confidence for this example:

- `1 - 0.556 = 0.444`
- rounded launch `confidence_pct = 45%`

Confidence contribution:

- `0.45 × 20 = 9.0`

#### True terminal-unknown traffic case
- chosenTraffic = **unknown**
- confidence_pct = **10%**
- risk_contribution = **20**

Effective traffic confidence contribution:

- `0.10 × 20 = 2.0`

#### Interpretation

Traffic is a dominant likelihood variable. Official traffic truth should therefore contribute much more confidence than weak fallback traffic, and terminal traffic unknown should contribute very little.

### 18.3 Slice-level example — low-impact bike-lane uncertainty

A 125-mile route contains:

- **120 miles** of path-domain slices
- **5 miles** of road-domain slices

Assume bike-lane uncertainty is only relevant on those 5 road miles and that, at the route level, bike-lane uncertainty could affect only **4%** of total route risk.

Let bike-lane confidence on those 5 miles be **0** for this example.

Then:

- confidence contribution from bike-lane = `0.00 × 4 = 0.0`
- uncertainty contribution from bike-lane = `(1.00 - 0.00) × 4 = 4.0`

So the route still lands at:

- route confidence = `(96 / 100) × 100 = 96% confidence`
- band = **very high**

#### Interpretation

This is the intended launch behavior.

Bike-lane uncertainty on a small score-relevant fraction of the route may reduce confidence slightly, but it may not poison the whole route.

### 18.4 Slice-level example — baseline speed on a score-driving road slice

A high-speed road-domain slice has the following launch risk contributions:

- speed = **80%**
- traffic = **15%**
- facility = **3%**
- shoulder = **2%**

Chosen input confidence rates:

- chosenSpeed = **45 mph from highway_baseline** → **45**
- chosenTraffic = **official_per_lane** → **100**
- chosenFacility = **osm_posted / known** → **95**
- chosenShoulder = **observed** → **90**

Compute each confidence contribution:

- speed: `45 × 0.80 = 36.00`
- traffic: `100 × 0.15 = 15.00`
- facility: `95 × 0.03 = 2.85`
- shoulder: `90 × 0.02 = 1.80`

Slice confidence:

- `36.00 + 15.00 + 2.85 + 1.80 = 55.65`

Band:

- **medium**

#### Interpretation

This slice is not low-confidence because every field is bad.
It is medium-confidence because a dominant variable — speed — is only baseline-derived in a score-driving context.

That is the intended behavior.

### 18.5 Slice-level example — shoulder non-stacking

A road-domain slice has:

- painted bike lane present
- shoulder also present
- score uses the bike lane as the operative facility signal

Assume facility carries **12%** of this slice’s score impact and shoulder would otherwise have carried **8%** if it were operative.

Chosen input confidence rates:

- facility confidence = **95**
- shoulder confidence = **unknown**, but shoulder is non-operative

Then:

- facility confidence contribution = `95 × 0.12 = 11.40`
- shoulder confidence contribution = `0 × 0.00 = 0.00`

#### Interpretation

Under the launch anti-double-counting rule:

- shoulder contributes **no additional score effect**
- shoulder contributes **no additional confidence burden**
- the bike lane carries the operating-space role for both score and confidence

### 18.6 Crossing example — strong crossed-road truth, weak control truth

A crossing event uses these launch risk contributions:

- crossed-road speed risk contribution = **45**
- crossed-road traffic risk contribution = **35**
- width risk contribution = **10**
- control risk contribution = **5**
- movement risk contribution = **5**

Chosen input confidence rates:

- crossed-road speed = **authoritative_posted** → **100**
- crossed-road traffic = **official_per_lane** → **100**
- width = **osm_inferred** → **85**
- control = **unknown** → **10**
- movement = **authoritative_inferred** → **90**

Compute each confidence contribution:

- crossed-road speed: `1.00 × 45 = 45.00`
- crossed-road traffic: `1.00 × 35 = 35.00`
- width: `0.85 × 10 = 8.50`
- control: `0.10 × 5 = 0.50`
- movement: `0.90 × 5 = 4.50`

Total crossing risk contribution:

- `45 + 35 + 10 + 5 + 5 = 100`

Crossing confidence:

- `(45.00 + 35.00 + 8.50 + 0.50 + 4.50) / 100 = 0.935`
- `crossing_confidence_pct = 93.50%`

Band:

- **very high**

#### Interpretation

Unknown control does reduce confidence, but it does not overwhelm the crossing because the dominant crossing variables — crossed-road speed and crossed-road traffic — are very strong.

That is the intended launch behavior.

### 18.7 Route-level example — Rockmart-style route

A route contains:

- ~120 path-domain miles with zero local risk
- 5 road-domain score-driving slice groups
- observed/posted speed truth on all risk-driving road miles
- mixed or weaker facility certainty

Assume the 5 score-driving road groups carry the following route-risk shares and slice confidences:

| Slice group | Route-risk share | Slice confidence |
|---|---:|---:|
| 1 | 53% | 94 |
| 2 | 41% | 92 |
| 3 | 2% | 78 |
| 4 | 2% | 76 |
| 5 | 2% | 80 |

All path-domain zero-risk miles contribute **zero confidence penalty** to `road_confidence`.

Road confidence:

- `(0.53 × 94) + (0.41 × 92) + (0.02 × 78) + (0.02 × 76) + (0.02 × 80)`
- `= 49.82 + 37.72 + 1.56 + 1.52 + 1.60`
- `= 92.22%`

Band:

- **very high**

#### Interpretation

This is one of the core launch expectations of the confidence model.

A route should not be dragged to low confidence just because many path-domain miles contain low-impact unknowns if the actual risk-driving road miles are strongly supported.

Route confidence may legitimately land even higher if the route is overwhelmingly confidently identified as benign path-domain mileage.

### 18.8 Route-level example — genuinely low-confidence route

A route contains:

- many score-driving road-domain miles
- speed mostly from highway_baseline
- traffic mostly from weak fallback classes
- several major crossings with inferred or unknown control
- little official or posted truth on the dominant variables

Assume route-risk shares and local confidences simplify to:

| Component | Route-risk share | Local confidence |
|---|---:|---:|
| continuous road slices | 80% | 25 |
| major crossings | 20% | 20 |

Route confidence:

- `(0.80 × 25) + (0.20 × 20)`
- `= 20 + 4`
- `= 24%`

Band:

- **low**

#### Interpretation

This route is low-confidence for the right reason:

the uncertainty sits directly on the variables that materially drive the route’s score.

### 18.9 Design lesson from the examples

These examples all point to the same design rule:

> Confidence does not care how many fields are ugly.  
> Confidence cares how uncertain the risk-driving truths are.

That is why:

- zero-risk path slices contribute zero confidence penalty
- observed/posted speed on risk-driving miles pulls confidence up strongly
- bike-lane uncertainty matters only when bike-lane truth materially affects the score
- crossing confidence mirrors crossing score structure
- road confidence is built from weighted uncertainty contribution, not flat counts
- route confidence separately reflects whole-route characterization


## 18. Non-goals

This spec does not yet define:

- final user-facing confidence UI for all app surfaces
- confidence for non-safety index families
- crowd report confidence
- ML-generated uncertainty
- hazard confidence beyond the canonical safety inputs
- comparative-context confidence

---

## 19. Design rules

1. **Confidence must use the same canonical variable graph as score.**
2. **Confidence must be risk-aware.**
3. **Zero-risk slices contribute zero confidence penalty.**
4. **Inferred is not the same as unknown.**
5. **Direct posted truth for dominant variables should pull confidence upward strongly.**
6. **Flat field counting is forbidden.**
7. **Confidence trace is required.**
8. **Reporting must explain not just what the model thinks, but how well it knows it.**

---

## 20. Launch philosophy

Confidence should be strict enough to mean something, but not so harsh that almost every route becomes mediocre by definition.

If:
- score-driving speed truth is strong
- traffic truth is strong enough
- and the remaining uncertainty is mostly secondary

then high route confidence should be achievable.

That is a requirement, not a nice-to-have.

---

## 21. Final rule

> **Confidence is not a sidecar. It is the trust model for the same canonical route truths that produce the score.**


---

## Source File: docs/02-architecture/design/ds-021-profile_based_routing_and_alternate_route_policy_spec.md

# DS-021 — Profile-Based Routing and Alternate Route Policy Spec

**Status:** Accepted  
**Date:** 2026-04-18  
**Related:** [ADR-044](../../../docs/03-adrs/adr-044-profile_based_routing_and_alternate_route_policies.md), [EXEC-013](../../../docs/04-execution/exec-013-profile_based_routing_implementation_plan.md), [ASS-010](../../../docs/assessments/ass-010-phase0_routing_audit.md)

---

## 1. Purpose

This spec defines the Lanterne-owned contract for profile-based routing without prescribing a self-developed routing engine.

Lanterne owns:

- visible route-profile semantics
- edge-cost policy semantics
- alternate comparison and suppression rules
- rider-facing result contract

Lanterne does **not** need to own:

- the low-level routing engine implementation
- preprocessing artifacts
- shortest-path internals

The preferred implementation direction is an external routing engine integration, with GraphHopper as the leading candidate.

---

## 2. Core rule

There may be only **one routing-engine integration contract** for rider-facing route generation.

That contract must serve:

- `Route To`
- draw-leg recompute
- future detour replacement workflows

Visible route differences come from policy and comparison logic, not from multiple unrelated routing stacks.

---

## 3. Launch-visible profiles

Launch rider-facing profiles remain:

- **Direct**
- **Safer**
- **Lower Traffic**
- **Bike Support**

`Balanced` may exist internally for orchestration or experimentation, but it is not a visible launch option.

---

## 4. Integration boundary

Whatever routing engine sits underneath must expose enough stable edge/path attributes for Lanterne to apply:

- profile semantics
- comparison rules
- suppression rules
- “no meaningful alternate” logic

The integration boundary should normalize concepts such as:

- distance
- estimated travel time
- speed burden
- traffic burden
- bike-support burden
- shoulder burden
- hazard burden
- turn / crossing / intersection burden

The boundary must be stable enough that Lanterne can swap routing engines without rewriting product semantics.

---

## 5. Non-goals

This spec does not approve:

- a custom self-hosted routing graph stack as a long-term architecture
- separate engines for Route To vs Draw
- rider-editable policy weights
- giant routing-settings surfaces
- fake alternates created to appear intelligent

---

## 6. Budget and honesty rules

Alternate discovery must be bounded and honest.

The integration must support:

- explicit search/runtime budgets
- explicit extra-distance / extra-time tolerances
- route-size-aware limits
- explicit no-result / no-meaningful-alternate outcomes

If an alternate cannot be found within budget, the product must say so rather than pretending otherwise.

---

## 7. Safety-score separation rule

The headline Safety Score remains canonical route analysis output.

Routing profiles influence:

- route selection
- alternate comparison

Routing profiles do **not** redefine:

- score semantics
- score curve meaning
- route-analysis truth

---

## 8. Implementation guidance

The implementation should prefer:

- external engine integration
- one adapter layer
- one policy/comparison layer
- one rider-facing contract

GraphHopper is the preferred future direction, but this spec is written so the policy layer stays valid even if the underlying engine changes.


---

## Source File: docs/02-architecture/design/ds-022-speed_prior_and_area_baseline_policy_spec.md

# DS-022 — Speed Prior and Area-Baseline Policy Specification

**Status:** Draft for review  
**Date:** 2026-04-19  
**Related:** [ADR-042](../../../docs/03-adrs/adr-042-evidence_resolution_and_truth_propagation_model.md), [DS-017](./ds-017-truth_resolution_and_propagation_spec.md), [DS-020](./ds-020-confidence_model_and_tracing.md), [DS-029](./ds-029-provenance_precedence_confidence_and_traceability_spec.md), [ASS-011](../../assessments/ass-011-speed_truth_feed_audit_2026_04_19.md)

---

## 1. Purpose

This spec defines the policy contract for predicted and baseline speed evidence layers below direct/corridor truth:

1. `local_area_predicted`
2. `regional_prior`
3. `highway_area_baseline`

It exists to prevent the speed system from collapsing into blunt class defaults such as:

- `trunk => ~55`
- `primary => ~45`
- `secondary => ~40`

without enough local or contextual evidence.

This document is intentionally prescriptive. It is meant to reduce implementation drift and prevent ad hoc “reasonable guesses” from becoming rider-facing truth.

---

## 2. Core rule

`local_area_predicted`, `regional_prior`, and `highway_area_baseline` are **guarded fallback layers**, not truth shortcuts.

They may influence speed only when all stronger sources are absent:

1. `observed`
2. `authoritative_posted`
3. `osm_posted`
4. `observation_inferred`
5. `authoritative_inferred`
6. `osm_inferred`

They must not override:

- posted speed
- same-road propagated truth
- nearby same-corridor propagated truth
- authoritative segment-level government speed

### No-freelance rule

Implementations must not invent additional prior logic outside this document.

If a caller wants to use a prior dimension not specified here, it must be added to this DS first.

---

## 3. Non-goals

This spec does not:

- define posted-speed parsing
- define same-road propagation mechanics
- define confidence scoring math
- approve rider-facing use of raw statutory maximums as if they were posted truth
- treat research workbook rows as automatically production-safe

---

## 4. Definitions

### 4.1 `local_area_predicted`

A local prediction derived from nearby direct posted or authoritative speed contributors in the route/corridor search space.

It is stronger than `regional_prior` and baseline layers because it uses real nearby speeds, but weaker than direct posted speed and same-corridor propagation because it is not the speed for this exact segment.

Rules:

- It may run only when no direct or corridor-inferred speed won.
- Contributors must be direct `observed`, `authoritative_posted`, or `osm_posted` speeds.
- Inferred, predicted, regional-prior, and baseline contributors are not eligible.
- Contributors must be within `5 mi` of the target in the local route/corridor search space.
- At least 2 eligible contributors are required.
- The target may use equal-or-lower through-road classes: trunk may use trunk/primary/secondary/tertiary; primary may use primary/secondary/tertiary; secondary may use secondary/tertiary; tertiary may use tertiary only.
- Service, residential, unclassified, path, cycleway, footway, private, and safe-path segments are not eligible targets or contributors.
- The exact average must be retained in trace.
- The displayed/resolved mph must round to the nearest 5 mph.
- Trace must retain contributor road names, route refs, source speeds, source types, class relation, distance/radius, aggregation math, rounding, and fallback reasons.

Rider-facing label:

- `Local Speed Prediction · Predicted`

### 4.2 `regional_prior`

A state- and context-sensitive prior derived from official/default rules or validated regional datasets.

It is stronger than a generic area baseline because it reflects real jurisdictional behavior.

Example:

- “New Jersey suburban business/residence district local street defaults are typically around 25–35 mph”

### 4.3 `highway_area_baseline`

A structured but more generic fallback derived from:

- road context
- area context
- facility form

It is weaker than `regional_prior`, but stronger than pure highway class.

Example:

- “rural divided non-interstate arterial roads are generally faster than urban undivided arterials”

### 4.4 `highway_baseline`

The last-resort generic class fallback.

Example:

- `trunk => generic class prior`

This layer must remain the weakest available non-null speed input.

---

## 5. Required decision dimensions

The following dimensions are allowed to influence prior selection.

No other dimensions may change prior selection unless this spec is updated.

### 5.1 State / jurisdiction

Required for `regional_prior`.

State must be carried whenever available from:

- route geography
- cue geography
- segment state lookup
- future authoritative feed coverage

If state is unknown:

- `regional_prior` must not be used
- evaluation falls through to `highway_area_baseline`

### 5.2 Urbanicity

Allowed values:

- `urban`
- `suburban`
- `urban_suburban`
- `rural`
- `unknown`

Urbanicity is a **hard limiter** for prior selection.

#### Rule

Urban/suburban contexts must not inherit rural statutory highs by default.

Specifically:

- a rural `primary` or `secondary` prior must not be reused for an urban/suburban segment
- a rural `trunk`/expressway prior must not be reused for a suburban retail corridor
- a statewide maximum must not be treated as an urban/suburban prior unless the source explicitly says so

If urbanicity is unknown:

- do not upgrade to a rural prior
- prefer a lower-confidence baseline instead

### 5.3 Municipality context

Allowed values:

- `inside_municipality`
- `outside_municipality`
- `unknown`

Municipality status is a first-class limiter because many states structure defaults around:

- inside municipality
- outside municipality

rather than pure land-use-style urbanicity.

#### Rule

If a source is municipality-based and municipality status is unknown:

- the row must not be treated as `regional_prior`
- it may only survive as `highway_area_baseline` if it is otherwise strongly limited and non-risky

### 5.4 District context

Allowed values:

- `business_district`
- `residence_district`
- `school_context`
- `open_highway`
- `general`
- `unknown`

District context outranks generic urbanicity where the source explicitly uses district language.

#### Rule

If a statute explicitly defines:

- `business district`
- `residence district`

those contexts must be preserved as their own prior selectors and must not be flattened into generic “urban”.

### 5.5 Access control

Allowed values:

- `full_control`
- `partial_control`
- `none`
- `unknown`

This dimension is mandatory for higher-order roads.

#### Rule

Higher-order suburban roads must not inherit freeway-like priors unless access control supports it.

Examples:

- `trunk` with `full_control` may justify much higher priors
- `trunk` with `none` in a retail corridor must not inherit freeway-style speeds

### 5.6 Dividedness

Allowed values:

- `divided`
- `undivided`
- `unknown`

Dividedness may influence prior selection, but only after urbanicity / municipality / access control filters pass.

It must not by itself justify high-speed rider-facing assumptions.

### 5.7 Facility class / mapping confidence

Allowed operational groupings:

- `local`
- `collector / tertiary-like`
- `secondary arterial`
- `primary arterial`
- `expressway / trunk-like`
- `freeway / motorway-like`

These mappings must be provenance-preserving and carry a confidence level.

Low-confidence mappings must not feed production priors.

---

## 6. Prescriptive policy by layer

## 6.1 `regional_prior`

### Allowed use

`regional_prior` is allowed only when:

1. state is known
2. source provenance is strong enough for operational use
3. context is sufficiently specific
4. the row does not behave like a broad statutory ceiling

### Typical valid uses

- business/residence district local defaults
- municipal/local street defaults
- clearly scoped suburban or urban local-street defaults
- clearly scoped state-specific rural local-road defaults when the mapping is narrow and operationally trustworthy

### Invalid uses

Do not use `regional_prior` for:

- broad statewide “other roads” or “general highway” rules
- urban/suburban higher-order roads where the source context is too broad
- mixed-class rows like `motorway; trunk` unless explicitly approved
- low-confidence functional-class mapping
- rows marked “review before operational use”

### Urban/suburban rule

Urban/suburban `regional_prior` should generally remain limited to:

- `residential`
- `unclassified`
- `living_street`
- very carefully approved municipal collector contexts

It should **not** be the default path for:

- `tertiary`
- `secondary`
- `primary`
- `trunk`

in urban/suburban areas unless explicit jurisdictional evidence is narrow and strong.

## 6.2 `highway_area_baseline`

### Allowed use

`highway_area_baseline` is allowed when:

1. stronger truth is absent
2. state-specific prior is unavailable, too broad, or not trustworthy enough
3. enough form/context is known to outperform pure highway class

### Typical valid uses

- rural two-lane state/US-numbered roads
- rural county-road defaults
- rural divided non-interstate multilane roads
- freeway/motorway defaults where the source is clearly freeway-specific

### Invalid uses

Do not use `highway_area_baseline` for:

- broad urban/suburban higher-order rows with unclear facility form
- rows whose main support is a statewide statutory maximum
- mixed mappings with low confidence
- contexts that require municipality or district data we do not have

---

## 7. Higher-order road guardrails

This section is the main protection against overstatement.

## 7.1 `trunk` / expressway-like roads

`trunk` must not imply “about 55” by default.

Before a `trunk` prior is allowed to influence speed, the system must check:

1. urbanicity
2. municipality status
3. access control
4. dividedness
5. corridor context if available

### Allowed high-speed prior cases

High-speed `trunk` priors are only acceptable when most of the following are true:

- rural or outside municipality
- access controlled or freeway-like
- divided multilane facility
- state/corridor support exists
- source mapping confidence is high

### Disallowed high-speed prior cases

Do not allow rural/high-speed `trunk` priors to influence:

- suburban retail corridors
- urban state routes
- municipal arterials
- undivided signalized trunk-like roads
- trunk rows whose source is really just a statewide “other highways” ceiling

### Rider-facing caution

Even when a `trunk` prior exists, rider-facing `~speed` should still prefer:

1. propagated same-road truth
2. nearby same-corridor truth
3. authoritative segment speed
4. `local_area_predicted`
5. `regional_prior`
6. `highway_area_baseline`
7. class baseline last

## 7.2 `primary` and `secondary`

Urban/suburban `primary` and `secondary` roads are especially prone to overstatement if rural defaults leak in.

#### Rule

Rural two-lane arterial defaults may only feed:

- rural `highway_area_baseline`

They must not be reused for urban/suburban `primary`/`secondary` approximation unless a stronger local/state rule explicitly supports it.

## 7.3 `tertiary`

`tertiary` is often too heterogeneous for aggressive prioring.

#### Rule

When `tertiary` lacks strong local or corridor evidence:

- prefer lower-confidence, lower-impact fallback
- do not “upgrade” tertiary to arterial-like speed just because a statute has a broad default

---

## 8. Selection algorithm

When stronger speed evidence is absent, prior resolution must follow this order:

1. determine whether same-road or same-corridor propagated truth exists
2. determine whether authoritative segment-level inferred/posting exists
3. if at least 2 eligible direct nearby contributors exist within 5 mi, evaluate `local_area_predicted`
4. if state and constrained context are known, evaluate `regional_prior`
5. if `regional_prior` is absent or blocked, evaluate `highway_area_baseline`
6. if all fail, fall through to `highway_baseline`

### 8.1 `regional_prior` admission checks

A row may be used as `regional_prior` only if all pass:

- `primary_official` provenance or equivalent approved source
- production-use approval exists
- mapping confidence is not low
- required context dimensions for that row are known
- row is not flagged for manual validation
- row does not represent a broad statewide ceiling without enough form/context

### 8.2 `highway_area_baseline` admission checks

A row may be used as `highway_area_baseline` only if all pass:

- mapping confidence is not low
- form/context is specific enough to outperform pure class baseline
- row is not flagged for manual validation
- row does not depend on missing municipality/district context

---

## 9. Provenance requirements

Every prior row used operationally must preserve:

- source title
- source URL
- citation / section
- jurisdiction
- officialness
- confidence
- mapping confidence
- operational recommendation
- production-use classification

The runtime system must retain enough provenance to explain:

- why the row qualified
- which gating dimensions were satisfied
- why stronger sources were absent

---

## 10. Future government-feed integration

This system must assume future speed feeds will arrive separately from AADT.

Therefore:

- prior rows must be replaceable without schema confusion
- feed semantics must distinguish:
  - `posted_speed`
  - `speed_zone`
  - `average_speed`
  - `mixed_or_unclear`
- segment-level authoritative feeds must supersede priors cleanly

The existence of a prior must never make it harder to adopt better government data later.

---

## 11. Launch guidance

For launch-quality behavior:

- be conservative in urban/suburban higher-order contexts
- prefer omission over overstatement
- treat statutory highs as weak rider-facing approximations
- use motorway/freeway priors only where facility form is explicit
- keep `highway_baseline` as the true last resort

If uncertain, the system should choose the lower-confidence, less aggressive fallback rather than displaying an overconfident high speed.

---

## 12. Companion implementation notes

Implementation should be split across:

- shared speed policy tables and helpers
- evidence-layer admission logic
- provenance-preserving prior datasets

But the behavioral rules above are the source of truth.

No implementation convenience may weaken the guardrails in this spec without an explicit DS update.

---

## 13. Rider-facing provenance tooltip contract

The speed source label may remain short:

- `OSM posted`
- `Government posted`
- `Regional prior`
- `Area baseline`

But rider-facing surfaces may expose one additional level of detail via hover or click.

That second layer must be structured, not freeform.

### 13.1 Tooltip purpose

The tooltip exists to answer:

1. what source layer produced this speed
2. what real-world source or citation backed that layer
3. why that layer applied here
4. what caveat or limitation still matters

It must increase trust without overstating certainty.

### 13.2 Required fields

When available, the runtime provenance payload should support:

- `sourceLabel`
  - short rider-facing label already shown inline
- `descriptor`
  - short secondary phrase such as:
    - `Arizona statutory default`
    - `HPMS posted speed`
    - `OSM maxspeed tag`
- `title`
  - canonical source/family title
- `citation`
  - exact statute section, rule section, or feed identifier
- `explanation`
  - one concise sentence explaining why this layer applied
- `caveat`
  - one concise sentence explaining any limitation
- `jurisdiction`
  - state / agency / county when relevant
- `sourceUrl`
  - only when available
- `referenceType`
  - one of:
    - `official`
    - `aggregator`
    - `secondary`

### 13.3 Presentation rules

- Do not show long prose inline on the card.
- Do not show a citation if we do not actually have one.
- Do not imply posted truth when the value is inferred, prior-based, or baseline-based.
- Do not hide caveats for priors or baseline layers.
- Do not show ambiguous verification dates in rider-facing provenance.
- If the visible link is not an official government source, label it honestly as an aggregator or secondary reference.

Public-facing provenance must not include fields such as:

- `lastVerifiedDate`
- `verified on`
- `verified by`

because those imply a legal or editorial verification claim that may not actually exist.

Internal records may preserve ingestion, snapshot, or access dates separately, but those are not rider-facing provenance fields.

### 13.4 Layer-specific guidance

#### `authoritative_posted`

Expected tooltip shape:

- label: `Government posted`
- descriptor: `HPMS posted speed` or `State DOT speed feed`
- citation: feed or statute identifier if available
- caveat: none or minimal

#### `osm_posted`

Expected tooltip shape:

- label: `OSM posted`
- descriptor: `Matched OSM maxspeed tag`
- citation: way id may appear later, but no fake legal citation
- caveat: `OSM tag quality depends on mapper accuracy.`

#### `authoritative_inferred`

Expected tooltip shape:

- label: `Government inferred`
- descriptor: `Government feed or class-derived estimate`
- caveat: `Grounded in government data, but not a directly posted segment limit.`

#### `osm_inferred`

Expected tooltip shape:

- label: `OSM inferred`
- descriptor: `Derived from matched OSM road attributes`
- caveat: `Estimated from road tags because no posted speed was available.`

#### `local_area_predicted`

Expected tooltip shape:

- label: `Local Speed Prediction`
- descriptor: `Predicted from nearby posted speeds`
- explanation: `Used because no posted or corridor-propagated speed was available for this segment.`
- caveat: contributor math, for example `FM 726 = 50 mph; CR 12 = 55 mph; target = Avg(50, 55) = 52.5 mph; displayed as 55 mph`

#### `regional_prior`

Expected tooltip shape:

- label: `Regional prior`
- descriptor: `${state} statutory/default prior`
- citation: exact statute or administrative citation when available
- explanation: `Used because no posted, propagated, or segment-level government speed was available.`
- caveat: `This is a contextual prior, not a posted segment speed.`

#### `highway_area_baseline`

Expected tooltip shape:

- label: `Area baseline`
- descriptor: `Road-form and area baseline`
- explanation: `Used because stronger local/state prior evidence was unavailable.`
- caveat: `This is a generic fallback, not a posted segment speed.`

#### `highway_baseline`

Expected tooltip shape:

- label: `Class baseline`
- descriptor: `Highway-class fallback`
- explanation: `Used only because no stronger speed source was available.`
- caveat: `Lowest-fidelity speed source.`

### 13.5 Example

Example tooltip for a statutory prior:

- source label: `Regional prior`
- descriptor: `Arizona statutory default`
- title: `Arizona Revised Statutes`
- citation: `§28-701(B)(2)`
- explanation: `Used because no posted or propagated segment speed was available.`
- caveat: `Prima facie rule; this is not the same as a posted segment limit.`

This level of detail is desirable when we truly have the citation. It must not be fabricated when we do not.


---

## Source File: docs/02-architecture/design/ds-023-government_feed_ingestion_and_triage_spec.md

# DS-023 — Government Feed Ingestion and Triage Specification

**Status:** Draft for review  
**Date:** 2026-04-19  
**Related:** [ADR-042](../../../docs/03-adrs/adr-042-evidence_resolution_and_truth_propagation_model.md), [DS-013](./ds-013-comparative_traffic_context_schema_spec.md), [DS-017](./ds-017-truth_resolution_and_propagation_spec.md), [DS-022](./ds-022-speed_prior_and_area_baseline_policy_spec.md), [ASS-011](../../assessments/ass-011-speed_truth_feed_audit_2026_04_19.md)

---

## 1. Purpose

This spec defines how Lanterne evaluates, ingests, and operationalizes government transportation feeds.

It applies to datasets such as:

- speed limits
- speed zones
- AADT / traffic volumes
- lane counts
- bike lanes / bikeways
- shoulder attributes
- curves / curvature warnings
- access control
- route-class or inventory files

This is a triage and operationalization spec, not a source-by-source registry.

---

## 2. Core rule

Government feed presence is not enough for operational use.

Every candidate feed must be classified before ingestion into one of four states:

1. `production_ready`
2. `adapter_required`
3. `inventory_only`
4. `rejected`

No feed may be allowed to affect rider-facing truth or score until it has passed this triage.

---

## 3. Why this exists

State feeds are messy.

The same state may publish separate files for:

- traffic
- posted speed
- speed zones
- bike lanes
- roadway curves
- roadway inventory

Those files may differ in:

- geometry availability
- update cadence
- keying scheme
- semantics
- legal authority
- operational usefulness

This spec exists so feed ingestion is deliberate instead of opportunistic.

---

## 4. Feed families

The system should classify each feed into one or more of these families:

### 4.1 Segment-truth families

- `posted_speed`
- `speed_zone`
- `aadt`
- `lane_count`
- `shoulder`
- `bike_facility`
- `access_control`

These may directly influence canonical route truth once normalized.

### 4.2 Structural / context families

- `curve_inventory`
- `functional_class`
- `road_inventory`
- `route_log`
- `municipal_boundary_reference`

These usually do not act as direct rider-facing truth by themselves, but may improve:

- contextual priors
- hazard inference
- corridor matching
- routing cost models

### 4.3 Reference-only families

- legal speed-order PDFs
- statute digests
- tabular summaries without geometry or stable join keys

These may be useful for provenance or research but are not operational truth feeds by default.

---

## 5. Triage dimensions

Every feed must be scored across the following dimensions.

### 5.1 Source authority

Allowed values:

- `primary_official`
- `official_but_interpreted`
- `secondary_only`

Only `primary_official` feeds may become `production_ready`.

### 5.2 Data semantics

The feed must clearly declare what it contains.

Allowed speed semantics:

- `posted_speed`
- `speed_zone`
- `average_speed`
- `advisory_speed`
- `mixed_or_unclear`

Allowed traffic semantics:

- `aadt`
- `aadt_directional`
- `peak_hour_volume`
- `classification_count`
- `mixed_or_unclear`

If semantics are mixed or unclear, the feed cannot be `production_ready`.

### 5.3 Geometry shape

Allowed values:

- `segment_geometry`
- `point_geometry`
- `route_milepost_table`
- `tabular_no_geometry`

### 5.4 Joinability

Allowed values:

- `direct_segment_join`
- `stable_route_milepost_join`
- `requires_custom_linear_referencing`
- `manual_only`

### 5.5 Update confidence

Allowed values:

- `clear_refresh_cycle`
- `irregular_but_usable`
- `stale_or_unknown`

### 5.6 Operational specificity

Allowed values:

- `segment_specific`
- `corridor_specific`
- `class_or_region_specific`
- `too_broad`

This is critical.

Broad statewide ceilings must not impersonate segment truth.

---

## 6. Triage outcomes

## 6.1 `production_ready`

A feed may be `production_ready` only if:

- source authority is `primary_official`
- semantics are explicit
- geometry or join path is operationally usable
- operational specificity is at least corridor-specific
- provenance can be preserved cleanly

Typical examples:

- state GIS linework with posted segment speed
- joined HPMS-like traffic segments
- official bike-facility geometry

## 6.2 `adapter_required`

Use when the source is valuable but not directly ingestible yet.

Typical examples:

- speed-order tables keyed by route + milepost
- roadway inventory tables needing a custom join
- curve inventories requiring route-log alignment

These feeds are good candidates for future ingestion, but must not silently act as truth yet.

## 6.3 `inventory_only`

Use when the feed is useful to know about but not yet fit for operational use.

Typical examples:

- reference PDFs
- mixed-semantics exports
- weakly keyed tables
- jurisdictional datasets whose geometry cannot yet be aligned

## 6.4 `rejected`

Use when the feed is too weak, unclear, stale, or risky to rely on.

Typical examples:

- non-official mirrors
- ambiguous summary exports
- feeds whose semantics cannot be trusted

---

## 7. Field-family-specific rules

## 7.1 Speed

Segment-level posted speed geometry is the highest-value target.

Priority within government speed ingestion:

1. posted segment speed geometry
2. speed-zone geometry with clear legal semantics
3. route/milepost speed orders that can be normalized reliably
4. statewide class/default tables only as prior inputs

Do not treat:

- average-speed feeds
- advisory-speed feeds
- broad statutory maxima

as if they were posted segment speed.

## 7.2 Traffic

AADT is already operationally useful even when not perfectly segment-specific, but must still preserve:

- source family
- directionality
- lane-count assumptions

Traffic feeds may become `production_ready` sooner than speed feeds if the semantics are clearer.

## 7.3 Bike facilities

Bike-lane and bikeway geometry can be highly valuable, but only if:

- geometry is accurate enough
- facility semantics are explicit
- the feed is not just a planning wishlist or project layer

Planned/future facilities must never be merged into current truth.

## 7.4 Curves

Curve inventories should usually begin as:

- `adapter_required`

unless they already expose clean geometry or stable route-log keys.

Curve data is best treated first as:

- hazard/context enrichment

not as speed truth.

## 7.5 Shoulder / roadway inventory

Roadway inventory fields can be high value, but only if:

- the semantics are current-condition attributes
- the geometry/join is reliable

Inventory files often make strong `adapter_required` candidates.

---

## 8. Provenance rules

Every ingested or inventoried feed must preserve:

- source title
- source owner
- source URL
- reference type
  - `official`
  - `aggregator`
  - `secondary`
- field family
- semantics classification
- geometry type
- join strategy
- triage outcome
- justification

Public-facing provenance must not imply legal verification beyond what the source actually provides.

---

## 9. Implementation guidance

The system should keep three separate layers:

1. `feed_inventory`
   - every discovered feed, even if not operational
2. `feed_triage`
   - readiness classification and justification
3. `feed_adapters`
   - actual normalization logic for production-ready or adapter-required feeds

This prevents “we know the URL exists” from being confused with “the feed is live in scoring.”

---

## 10. Launch guidance

For launch and near-term implementation:

- prefer high-value, high-clarity segment feeds first
- prefer geometry-backed truth over broad defaults
- use priors only where direct feeds are absent
- do not ingest broad, ambiguous state exports just to claim coverage

Coverage is less important than trustworthy semantics.

---

## 11. Practical sequence

Recommended sequence for each new state or feed family:

1. inventory the feed
2. classify semantics
3. classify geometry and joinability
4. assign triage outcome
5. write or reject adapter
6. preserve provenance and source family
7. only then allow it into truth/scoring

This sequence applies equally to:

- speed
- traffic
- bike support
- shoulder
- curves
- other roadway inventory files


---

## Source File: docs/02-architecture/design/ds-024-parallel_bike_facility_capture_and_corridor_ownership_spec.md

# DS-024 — Parallel Bike Facility Capture & Corridor Ownership

Status: Draft
Date: 2026-04-20

Related:
- ADR-019 — Route Corridor & Proximity Rules
- ADR-020 — Atomic Route Analysis Unit & OSM Variable Architecture
- ADR-026 — Canonical Route Identity
- DS-008 — Route Corridor Model
- DS-017 — Truth Resolution & Propagation Specification

---

## 1. Purpose

This document defines how Lanterne should handle the critical case where:

- the GPX geometry rides near a motor road corridor
- a real designated bike facility runs parallel beside that road
- the raw matcher keeps preferring the motor road centerline
- but rider truth should attach to the bike facility

This is the canonical spec for fixing:

- sidecar cycleways that hug arterials
- separated paths that track the same corridor at grade
- curbside or frontage-adjacent bike facilities that are offset from the route line

This document defines two layers:

1. `Parallel Bike Facility Capture`
   Immediate targeted fix. Must ship first.

2. `Corridor Ownership`
   Fast follow. More authoritative corridor-level policy.

The intention is:

- ship `Parallel Bike Facility Capture` now
- ship `Corridor Ownership` next
- keep `Parallel Bike Facility Capture` as a fallback even after `Corridor Ownership` exists

---

## 2. Problem Statement

The current matcher is still fundamentally road-centered.

In corridors where:

- `highway=secondary|primary|trunk`
- and a parallel `highway=cycleway` or designated path runs beside it

the system can:

- see the sidepath in fetched OSM context
- display the blue path line in roads-visible mode
- yet still attach truth runs to the arterial for most of the corridor

This produces rider-visible errors:

- wrong speed truth
- wrong road identity
- wrong bike facility interpretation
- receipts and review surfaces that imply the route is riding the road instead of the sidepath

The failure is not “missing data.”

The failure is:

- route-carrier selection
- corridor ownership
- and over-trusting centerline proximity

---

## 3. Non-Goals

This spec does not attempt to:

- solve all sidewalk/path/road ambiguities globally
- replace the general road matcher
- infer protected facilities from weak tags
- promote `cycleway=lane` on a motor road into separated infrastructure
- reclassify parking aisles, driveways, or service roads as bike facilities

This spec is only for:

- real parallel bike facilities
- that plausibly carry the route
- but are being lost to an adjacent motor-road carrier

---

## 4. Definitions

### 4.1 Parallel Bike Facility

A nearby way is a `parallel bike facility` if all of the following are true:

- it is not the same way as the current motor-road carrier
- it is bike-eligible and non-sidewalk
- it runs in the same corridor direction as the matched road
- it persists for more than a trivial fragment

### 4.2 Designated Facility

A nearby way is `designated` if it satisfies one of:

- `highway=cycleway`
- `bicycle=designated`
- `highway=path` with explicit bike allowance and not sidewalk
- `highway=track` with explicit bike allowance and not sidewalk

### 4.3 Sidewalk-Like

A way is `sidewalk-like` if it matches existing sidewalk exclusion policy, including:

- `footway=sidewalk`
- explicit sidewalk tagging
- connector/crossing-only pedestrian fragments that should not own corridor truth

### 4.4 Corridor Ownership

`Corridor ownership` means:

- for a sustained corridor span,
- the bike facility, not the motor road,
- is treated as the canonical carrier of route truth

This affects:

- truth run identity
- speed source choice
- bike facility class
- receipts and inspect surfaces

---

## 5. Immediate Layer: Parallel Bike Facility Capture

## 5.1 Intent

`Parallel Bike Facility Capture` is the immediate tactical layer.

It must:

- run after raw sample-to-road matching
- run before truth snapshot and truth-run materialization
- be able to override a motor-road carrier with a nearby real bike facility
- be corridor-aware, not single-sample-only

It exists specifically because raw matching alone is not enough.

---

## 5.2 Placement in Pipeline

This pass must execute:

1. after raw road matching and transition stabilization
2. before truth snapshot
3. before truth-run sanitizers that collapse small fragments

Reason:

- it needs the sample-level road map
- it must be able to rewrite that map before truth is frozen

---

## 5.3 Candidate Eligibility

For each matched sample currently attached to a motor road, search the nearby-context road universe for override candidates.

A candidate is eligible only if:

- `road.isSafePath === true`
- highway is one of:
  - `cycleway`
  - `path`
  - `track`
- candidate is not sidewalk-like
- candidate is not private-only access junk
- candidate is not a tiny crossing connector

Higher-confidence eligibility:

- `highway=cycleway`
- or `bicycle=designated`

These should be preferred over generic path/track candidates.

---

## 5.4 Sample-Level Gating Rules

A candidate may override a motor-road sample only if:

- it is within a corridor-local offset threshold
- it is directionally aligned with the route
- it is sufficiently parallel to the current motor-road carrier

Required sample gates:

- maximum offset from route sample
- maximum route-heading difference
- maximum motor-road-to-path bearing difference

Design principle:

- designated cycleways get looser distance tolerance
- generic path/track candidates get stricter tolerance
- sidewalks never qualify

### Target Production Thresholds

The intended production thresholds for this layer are:

- designated `highway=cycleway`
  - soft offset: `<= 18 m`
  - hard maximum offset: `<= 30 m`
- designated `path/track`
  - soft offset: `<= 12 m`
  - hard maximum offset: `<= 20 m`
- route-heading difference:
  - target cap: `<= 30°`
- motor-road-to-facility parallelism:
  - target cap: `<= 20°`

Meaning:

- within the soft offset, the facility is plausibly sidecar and eligible
- beyond the hard maximum offset, it must never be force-promoted

The hard cap exists specifically to prevent:

- nearby unrelated trails
- frontage/internal circulation paths
- off-corridor greenways
from being stolen by this rule.

---

## 5.5 Sustained Run Requirement

No single sample may be promoted on its own.

Promotion only applies if the override candidate sustains as a run:

- minimum sample count threshold
  or
- minimum mileage threshold

This is required to prevent:

- one-point blue blips
- lane-divider noise
- driveway/path adjacency false positives

Immediate launch rule:

- a promotion run must persist for at least `2` samples
  or
- at least `0.02 mi`

This threshold can be tuned, but the sustained-run rule is mandatory.

### Target Production Thresholds

The intended production thresholds for this layer are:

- minimum sustained mileage: `>= 0.08 mi`
- minimum sustained samples: `>= 4`

If we need to be stricter in suburban arterial contexts, `0.10 mi` is acceptable.

Rationale:

- `0.02 mi` is too small and invites chatter
- the floor should be closer to the length of a city block
- the rule must prefer sustained corridor truth over tiny tactical fragments

---

## 5.6 Winner Selection Policy

If multiple eligible bike facilities compete:

prefer in this order:

1. `highway=cycleway`
2. `bicycle=designated`
3. lower offset from route
4. better heading alignment
5. longer sustained continuity

Do not break ties using road name alone.

Bike facility truth must not depend on whether the sidepath happens to share the arterial’s name.

---

## 5.7 Output of Parallel Bike Facility Capture

When a sustained override run is accepted:

- rewrite the affected sample-road map entries to the bike facility
- preserve the original motor-road match only in debug/audit traces
- allow later truth layers to treat the bike facility as the substrate

This means:

- the corridor becomes blue/safe-path where appropriate
- speed truth falls to path/domain logic instead of arterial speed
- receipts reflect the bike facility as the carrier

---

## 5.8 Failure Modes This Layer Must Avoid

It must not:

- steal routes onto sidewalks
- steal routes onto parking-lot perimeter paths
- steal routes onto random nearby trails that only briefly approach the corridor
- steal routes onto crosswalk or connector fragments
- turn `cycleway=lane` on a motor road into a separated path

If uncertain, fail closed and keep the road.

---

## 5.9 Acceptance Criteria

`Parallel Bike Facility Capture` is acceptable only if it fixes:

- sustained sidecar cycleway corridors beside arterials
- without creating obvious sidewalk theft
- without increasing tiny blue fragment noise

Required test corridors:

- suburban arterial with hugging `highway=cycleway`
- path beside a major road but offset from centerline
- sidewalk beside a road that must not win
- parking-lot/service adjacency that must not win
- short crossing connector that must not win

---

## 6. Fast Follow Layer: Corridor Ownership

## 6.1 Intent

`Corridor Ownership` is the authoritative version of the same idea.

Instead of promoting sample-by-sample, it determines:

- which substrate owns a corridor span
- road or bike facility

This is the desired long-term architecture.

`Parallel Bike Facility Capture` remains as fallback for edge cases and transitional safety.

---

## 6.2 Corridor Ownership Problem

Raw matching is inherently local.

But rider truth for this use case is corridor-level:

- “the route is on the sidepath beside Volunteer Boulevard”

not:

- “some samples are on the road, some on the path, depending on centerline geometry”

So corridor ownership must be decided across a span, not per point.

---

## 6.3 Corridor Ownership Candidate Set

For each motor-road corridor run, identify candidate parallel facilities from nearby-context roads.

Candidate families:

- designated cycleway
- designated path
- designated track

Reject:

- sidewalks
- crossing-only connectors
- private access junk
- service/parking fragments

---

## 6.4 Ownership Signals

Ownership should be decided from corridor signals, not one score.

Signals include:

- percentage of samples with eligible parallel facility nearby
- mean and max offset
- mean and max heading deviation
- continuity of the facility along the run
- facility designation strength:
  - cycleway > designated path/track > weaker path
- contradiction signals:
  - route obviously departs onto road
  - facility disappears
  - large separation or heading mismatch

---

## 6.5 Ownership Decision Rule

A corridor span should transfer from motor road to bike facility only if:

- the facility is present for a high share of the corridor span
- it remains geometrically parallel
- it is designated strongly enough
- contradiction signals stay below threshold

Ownership is binary at the span level:

- either the bike facility owns the span
- or the motor road owns the span

Do not alternate ownership every few samples.

If evidence is mixed, split into explicit subspans.

---

## 6.6 Relationship to Parallel Bike Facility Capture

Once corridor ownership exists:

- it becomes the primary authoritative layer
- `Parallel Bike Facility Capture` stays enabled as fallback

Fallback should only act when:

- corridor ownership did not fire
- but a local sustained designated facility is still clearly the right carrier

This avoids indefinite dependence on the tactical layer.

---

## 6.7 Why Keep the Tactical Layer

We should keep `Parallel Bike Facility Capture` even after ownership ships because:

- local geometry can still fail corridor grouping
- short but real bike-facility spans still exist
- some facilities may be too fragmented for full ownership logic

So the architecture should be:

- corridor ownership first
- targeted capture second
- road-centered truth last

---

## 7. Implementation Sequence

## Phase 1

Ship `Parallel Bike Facility Capture`.

Requirements:

- sample-map override pass
- nearby-context search
- designated cycleway preference
- sustained-run requirement
- strong sidewalk exclusion
- debug logs for accepted promotions

### Phase 1 Implementation Note

Phase 1 should intentionally begin with a broad-brush rule set.

The goal of Phase 1 is:

- prove the seam is correct
- verify that the system can capture obvious hugging cycleways
- observe the real failure modes before hardening

So Phase 1 should not begin with every target threshold above fully enforced.

Recommended Phase 1 posture:

- keep candidate families narrow
  - `cycleway`
  - designated `path/track`
- keep sidewalk exclusion hard
- keep sustained-run logic present
- keep the implementation explainable
- avoid introducing too many coupled thresholds in the first pass

Then, once the seam is proven:

- add soft offset caps
- add hard maximum offset caps
- tighten sustained-run thresholds toward the target production values
- add more explicit contradiction handling

Reason:

- too many variables in step 1 will create reverse-debugging of our own complexity
- the first job is to prove that the layer works at all
- the second job is to harden it

## Phase 2

Ship `Corridor Ownership`.

Requirements:

- corridor-span grouping
- candidate facility aggregation
- ownership signals
- explicit transfer decision
- subspan splitting when mixed

## Phase 3

Demote `Parallel Bike Facility Capture` to fallback.

Requirements:

- ownership first
- capture second
- explicit audit logs indicating which layer won

---

## 8. Debug / Audit Requirements

Both layers must emit structured diagnostics.

For capture:

- candidate road id
- highway type
- designation strength
- offset
- heading diff
- sustained run length
- accepted / rejected reason

For ownership:

- corridor span
- candidate facility ids
- ownership signals
- final owner
- contradiction reason when rejected

These diagnostics are mandatory because this class of bug is otherwise visually obvious but hard to localize in code.

---

## 9. Product Outcome

When this policy is working:

- routes that truly ride a sidecar cycleway will show as riding that facility
- arterial speed and road identity will stop leaking into those spans
- receipts and inspect will tell the same story as the visible bike path
- short tactical fragments will no longer be the only correctly-blue portion of a long sidepath corridor

That is the acceptance bar.


---

## Source File: docs/02-architecture/design/ds-025-transition_candidate_claim_and_projection_spec.md

# DS-025 — Transition Candidate, Claim, and Projection Spec

**Status:** Draft  
**Date:** 2026-04-21  
**ADR Parent(s):** [ADR-005](../../03-adrs/adr-005-route_analysis_model.md), [ADR-019](../../03-adrs/adr-019-route_corridor_and_proximity_rules.md), [ADR-042](../../03-adrs/adr-042-evidence_resolution_and_truth_propagation_model.md)  
**Related:** [DS-017](./ds-017-truth_resolution_and_propagation_spec.md), [EXEC-005](../../04-execution/exec-005-debugging_logs.md), `src/lib/route-boundary-geometry.ts`

---

## 1. Purpose

This document defines the architectural replacement for Lanterne’s current seam ownership model.

It exists because the current route pipeline has repeatedly allowed multiple modules to act as if they independently own the answer to:

> “Where does this route transition actually happen?”

That led to recurring drift and contradiction between:

- truth / road-name transitions
- color / display transitions
- cue / transition anchors
- debug markers

The replacement model separates three things that were previously conflated:

1. **Candidate**
   - a possible physical locus on the route
2. **Claim**
   - a subsystem asserting that something important changes near that locus
3. **Zone**
   - the resolver-owned container that clusters nearby claims and candidates into one bounded decision surface
4. **Transition**
   - the canonical route event resolved from a zone

Consumers then project seams from transitions. They do not own transitions.

---

## 2. Core rule

**Burn down seam ownership, not corner detection.**

The shared geometry-first resolver is the right backbone, but it is not the final model.

The final model must be:

- geometry-first for candidate discovery
- claim-based for subsystem inputs
- zone-based for clustering and ambiguity handling
- centrally resolved for canonical transitions
- projection-based for truth, display, cues, and debug

No downstream consumer may invent or relocate physical transition ownership.

---

## 3. Why the old model failed

Different modules were re-deciding the route seam:

- `route-analysis.ts`
  - moved truth boundaries
- `heatmap/builder.ts`
  - suppressed / merged display boundaries
- `display-continuity.ts`
  - suppressed / merged presentation boundaries and later re-snapped them
- `transition-chain.ts`
  - derived corrected cue/debug boundaries again
- debug overlays
  - reconstructed alternative seam interpretations

That layering was the core bug.

The issue was not only bad point selection.

The issue was that the system had multiple seam owners.

---

## 4. Empirical baseline

The best debugging baseline so far has been the pure geometry overlay:

- white dots with black outlines
- one at every meaningful route corner
- derived from GPX geometry only
- no truth logic
- no road logic
- no color logic

The empirical result, especially in Vegas, was:

- white dots consistently caught almost every meaningful corner
- green/cyan evidence was useful secondary corroboration
- geometry appeared to be the right backbone for physical transition loci

That finding governs this design.

---

## 5. First-principles constraints

The model must preserve these truths:

1. A strong geometric corner is **not always** a true transition.
2. A true transition is **not always** at a geometric corner.
3. Different consumers may project the same physical event differently, but they should not default to independent transition ownership.
4. Presentation continuity / zoom suppression is not physical transition discovery.

---

## 6. Canonical objects

## 6.1 `TransitionCandidate`

A `TransitionCandidate` is a possible physical locus on the route.

It is usually geometry-derived, but the type system must allow future non-corner candidate classes.

```ts
export interface TransitionCandidate {
  candidateId: string;
  idx: number;
  measureM: number;
  lat: number;
  lon: number;
  supportStartIdx: number;
  supportEndIdx: number;
  kind: 'geometry_corner' | 'mid_edge_exception';
  angleDeg?: number;
  strength: number;
  reasonCode: CandidateReasonCode;
  nearbyCandidateIds: string[];
}
```

### Rules

- A candidate is not a transition.
- A candidate may be explicitly dismissed by the resolver.
- The white-dot layer is the canonical initial source of `geometry_corner` candidates.
- A candidate exposes a support window, not a zone.

---

## 6.2 `TransitionClaim`

A `TransitionClaim` is a subsystem saying:

> “I believe something materially changes near this route location.”

Claims do not move seams directly.

They must include both **where** the subsystem thinks something happens and **what changed**.

```ts
export interface TransitionStateSnapshot {
  identity?: {
    roadId?: number;
    roadName?: string;
    matchedEdgeIds?: string[];
    corridorRunId?: string;
    functionalClass?: string;
  };
  semantic?: {
    infraBucket?: string;
    speedClass?: 'low' | 'medium' | 'high' | 'unknown' | 'safepath';
    mode?: string;
    accessClass?: string;
  };
  display?: {
    colorToken?: string;
    styleToken?: string;
    widthToken?: string;
  };
  cue?: {
    cueContext?: string;
    maneuverType?: string;
    exitNumber?: string;
  };
}

export interface TransitionDelta {
  domainsAffected: TransitionDomain[];
  before: TransitionStateSnapshot;
  after: TransitionStateSnapshot;
}

export interface TransitionClaim {
  claimId: string;
  source:
    | 'truth_name'
    | 'display_color'
    | 'cue'
    | 'matcher'
    | 'hazard';
  nominalIdx: number;
  measureM: number;
  captureStartIdx: number;
  captureEndIdx: number;
  confidence: number;
  anchorPreference: 'prefer_candidate' | 'allow_mid_edge';
  reasonCode: ClaimReasonCode;
  delta: TransitionDelta;
  auditPayload?: Record<string, unknown>;
}
```

### Rules

- Claims may be emitted by upstream systems.
- Claims may not mutate truth runs, display runs, or cues directly.
- Claims must include explicit delta state.

### Delta field presence semantics

This contract must be interpreted strictly:

- omitted field
  - means **not asserted by this claim**
- explicit `unknown`
  - means the source asserts the field exists but does not know the value
- explicit `not_applicable`
  - means the field does not apply to this transition domain
- unchanged
  - must be represented by identical explicit before/after values when the
    source is asserting that no change occurred in that field

The claim normalizer is responsible for converting source-specific ambiguity
into these explicit semantics. The resolver and projectors may not infer their
own meaning from missing fields.

That explicitly answers the requirement that the model consider more than geometry alone:

- existing road
- future road
- prior state
- next state

must all be visible to the resolver.

---

## 6.3 `TransitionZone`

A `TransitionZone` is the resolver-owned clustering container produced by the
zone builder before canonical transition resolution begins.

It is the bounded surface within which claims are grouped, candidates are attached,
and ambiguity is preserved for later resolution.

```ts
export interface TransitionZone {
  zoneId: string;
  zoneStartIdx: number;
  zoneEndIdx: number;
  startMeasureM: number;
  endMeasureM: number;
  candidateIds: string[];
  claimIds: string[];
  classification: 'simple' | 'complex' | 'dismissed';
  reasonCodes: ResolutionReasonCode[];
}
```

### Rules

- Zones are owned by the resolver, not by candidates or claims.
- Zones exist before canonical transitions.
- Zones do not carry transition ids or post-resolution status.

---

## 6.4 `ResolvedTransitionZone`

A `ResolvedTransitionZone` is the post-resolution record tying one zone to its
resolution outcome.

```ts
export interface ResolvedTransitionZone {
  zoneId: string;
  resolutionStatus: 'resolved' | 'split' | 'ambiguous' | 'dismissed';
  transitionIds: string[];
  dismissedClaimIds: string[];
  dismissedCandidateIds: string[];
  reasonCodes: ResolutionReasonCode[];
}
```

### Rules

- The resolver emits this object after transition resolution.
- A zone may resolve to zero, one, or many canonical transitions.
- A complex intersection is modeled as a complex zone, not as a transition kind.

---

## 6.5 `CanonicalTransition`

A `CanonicalTransition` is the resolved route event.

It is the single object every consumer should read.

```ts
export interface CanonicalTransition {
  transitionId: string;
  zoneId: string;
  anchorIdx: number;
  anchorMeasureM: number;
  anchorLat: number;
  anchorLon: number;
  candidateId?: string;
  kind:
    | 'shared_event'
    | 'identity_change'
    | 'display_only'
    | 'non_corner_transition';
  domainsAffected: TransitionDomain[];
  resolutionStatus: 'resolved' | 'split' | 'ambiguous';
  ambiguityLevel?: 'low' | 'medium' | 'high';
  confidence: number;
  delta: TransitionDelta;
  claimIds: string[];
  supportingSources: TransitionClaimSource[];
  reasonCodes: ResolutionReasonCode[];
  projectionConstraints: {
    baseLocus: 'anchor' | 'zone';
    zoneReference?: 'start' | 'center' | 'end';
    projectionAnchorMeasureM?: number;
    allowDisplaySuppression: boolean;
    allowCueOffset: boolean;
    maxCueOffsetM?: number;
  };
}
```

### Rules

- A transition is the canonical route event.
- Consumers may project differently from it.
- Consumers may not invent a second physical transition by default.
- A transition belongs to exactly one zone.
- The projector-facing canonical model may not embed raw claim objects.

---

## 7. Pipeline

## 7.1 Candidate discovery

One module discovers route candidates.

Primary source:

- route geometry corners (`white dots`)

Later extensions may add:

- junction complexes
- explicit mid-edge exceptions

Candidate discovery must not depend on display or cue logic.

---

## 7.2 Claim emission

Each subsystem emits claims, not seams.

Initial claim sources:

- truth/name analysis
- display/color analysis
- cue analysis
- matcher / road identity evidence

Debug systems may emit claims only in shadow mode for evaluation.

They are not canonical inputs in production logic.

---

## 7.3 Zone construction

The resolver first constructs zones.

Its job is to:

1. cluster nearby claims into bounded decision surfaces
2. attach candidate neighborhoods to those surfaces
3. preserve complex or ambiguous areas without prematurely collapsing them into one point

Zone construction must happen before canonical transition resolution.

---

## 7.4 Transition resolution

The resolver is the only place where clustering and ownership happen.

Its job is to:

1. resolve zones into zero, one, or many canonical transitions
2. associate transitions to candidates where appropriate
3. preserve non-corner transitions with explicit reason codes
4. produce ordered `CanonicalTransition` objects

This is the architectural center of gravity.

---

## 7.5 Projection

Projections are derived from canonical transitions:

- truth/name projection
- display/color projection
- cue projection
- debug projection

These are consumer-specific views.

They must not redetermine physical transition ownership.

### Projection rule

Projectors may not choose a location inside a zone using local heuristics.

They may only use:

- the canonical transition anchor
- the resolver-provided `zoneReference`
- or an explicitly bounded offset allowed by `projectionConstraints`

---

## 7.6 Presentation suppression

Presentation suppression belongs **after** transition resolution.

It may:

- hide seams visually
- merge adjacent visible segments at certain zooms
- simplify map readability

It may not:

- create a new physical transition
- move a physical transition
- silently erase a canonical transition from the debug substrate

---

## 8. Clustering model

The next deep cut should cluster **claims into transition zones**, not collapse truth seams directly.

That is the key architectural change.

## 8.1 Why claims, not seams

If truth clusters its seams first and display clusters its seams later, the system still has multiple seam owners.

That is the old disease in a cleaner wrapper.

Clustering must happen centrally on claims into zones.

---

## 8.2 Clustering invariants

1. **Monotonic order**
   - transitions remain strictly ordered along the route
2. **No silent deletion of meaningful intermediate state**
   - clustering may not erase a material short intermediate run without explicit reason
3. **No clustering across a stronger intervening candidate**
   - if two claims straddle a stronger distinct candidate, they likely belong to different zones
4. **Provenance must survive**
   - every canonical transition retains the full list of attached claims
5. **Non-corner transitions require explicit reason**
   - no silent fallback
6. **Complex intersections may require zones, not points**
   - roundabouts, ramp bundles, and stacked turns may contain multiple subevents inside one complex zone
7. **Every claim ends attached or dismissed**
   - no silent orphaned claims
8. **Projection may not exit zone unless explicitly allowed**
   - cue offsets and other projected deviations must stay bounded and justified

---

## 9. Truth vs display authority

Truth/name and display/color should usually share the same canonical transition object.

They should not be forced to share the same projected seam point in every case.

### Default

- one canonical transition object
- multiple projections from that object

### Legitimate divergence cases

- straight rename with no corner
- display attribute change mid-edge
- cue timing offset from physical seam
- unresolved ambiguity inside a complex zone

### Rule

If truth and display differ, they still belong to the same transition object unless an explicit reason says otherwise.

Accidental drift is not a valid reason.

---

## 10. Initial module layout

The recommended new module family is:

- `src/lib/transitions/transition-types.ts`
  - shared interfaces and enums
- `src/lib/transitions/transition-candidates.ts`
  - geometry-first candidate builder
- `src/lib/transitions/transition-claim-adapters/`
  - source-specific claim emitters
- `src/lib/transitions/transition-claim-normalizer.ts`
  - maps source-specific claims into normalized claim semantics
- `src/lib/transitions/transition-zone-builder.ts`
  - central zone construction from claims + candidate neighborhoods
- `src/lib/transitions/transition-resolver.ts`
  - canonical transition resolution from zones
- `src/lib/transitions/projectors/truth.ts`
- `src/lib/transitions/projectors/display.ts`
- `src/lib/transitions/projectors/cue.ts`
- `src/lib/transitions/transition-audit.ts`
  - machine-readable traces and review artifacts

The current `route-boundary-geometry.ts` should become the seed of `transition-candidates.ts`.

---

## 11. Migration posture

This system should be introduced in **shadow mode first**.

That means:

- current truth/display/cue outputs still run
- the new transition system runs alongside them
- logs and overlays compare:
  - candidate
  - claim
  - canonical transition
  - projected seam
  - legacy seam

Only after shadow mode is stable should truth, display, and cue ownership move fully.

---

## 12. Tests and invariants

The new abstraction should add tests for:

- every rendered seam maps to exactly one canonical transition id
- every canonical transition belongs to exactly one zone id
- every resolved transition zone maps back to exactly one pre-resolution zone id
- every canonical transition is visible in debug
- every non-corner transition has an explicit reason
- every strong corner is either claimed or explicitly dismissed
- every claim is either attached or dismissed with a reason
- every dismissed strong candidate has a reason
- route reversal preserves transition structure within tolerance
- GPX resampling does not move anchors beyond tolerance
- presentation suppression does not mutate physical transition ownership
- projectors cannot read raw claims or raw candidates directly
- projectors cannot choose an in-zone seam location independently

Golden corpus cases must include:

- same-road bend with no transition
- straight rename with no corner
- color-only change
- one true corner with shared truth/display/cue event
- two legitimate transitions in one complex intersection
- roundabout
- ramp merge/diverge
- tiny spurious intermediate segment
- out-and-back overlapping geometry

---

## 13. Do not reintroduce

Do not:

- add another display-only seam forcing rule
- cluster based only on proximity without state change awareness
- let presentation continuity remain a physical seam owner
- treat “nearest white dot” as the whole model
- allow fallback paths to stay silent
- reconstruct canonical seam logic inside debug overlays
- implement `complex_zone` as a transition kind
- let audit payloads become new normative logic inputs
- compare only seam indices in shadow mode; compare candidates, claims, zones, transitions, and projections

---

## 14. Immediate design conclusion

The geometry-first deep cut was still worthwhile.

It should be understood as:

- **Pass 1**
  - prove that geometry-first corner discovery is the right backbone

This document defines:

- **Pass 2**
  - centralize transition ownership using candidates, claims, and canonical transitions

That is the path out of recurring seam drift.


---

## Source File: docs/02-architecture/design/ds-026-keyboard_shortcuts_and_map_input_spec.md

# DS-026 — Keyboard Shortcuts and Map Input Spec

**Status:** Draft  
**ADR Parent(s):** ADR-027, ADR-029, ADR-030  
**Last Updated:** 2026-04-21

---

## Purpose

Define the canonical keyboard shortcut contract for route planning and map navigation.

This spec exists to prevent keyboard behavior from turning into scattered `keydown`
handlers across drawers, map layers, and edit surfaces.

The core rule is:

- `Index.tsx` owns shortcut intent and mode gating
- `RouteMap.tsx` owns the live Leaflet map instance
- top-drawer workflows remain the source of truth for search/load actions

Keyboard shortcuts must trigger the same underlying actions as visible UI controls.

---

## Shortcut Scope

Shortcuts are active only when:

- no modifier keys are held (`meta`, `ctrl`, `alt`)
- focus is not inside:
  - `input`
  - `textarea`
  - `select`
  - `contenteditable`

This prevents map/navigation shortcuts from stealing keystrokes from search boxes,
URL fields, or future text-entry surfaces.

---

## Canonical Shortcut Map

### Global Map Navigation

- `ArrowUp` → pan map north
- `ArrowDown` → pan map south
- `ArrowLeft` → pan map west
- `ArrowRight` → pan map east
- `F` → zoom out
- `G` → zoom in

### Primary Creation / Acquisition

- `D` → start route drawing
- `S` → open top drawer to `where to?` / search
- `U` → open top drawer to GPX upload
- `R` → open top drawer to RideWithGPS URL entry

`D` must route through the same guarded create-route path as the visible draw control.
If a route is already loaded, the existing create-route guard remains authoritative.

These acquisition shortcuts must route through the same top-drawer actions as the visible UI.
They must not create alternate ingestion paths.

### Route Editing / Creation

- `E` → enter route edit mode when a route is currently being viewed
- `C` → invoke the currently active workflow cancel action
- `Backspace`
  - in route creation mode: undo last waypoint
  - in detour edit mode: undo last detour history step
- `Enter`
  - in route creation mode: `Done — Analyze`
  - in detour edit mode with active detours: save detour route

The bottom red cancel surface is the canonical cancel affordance for:

- analysis loading
- route drawing
- route edit sessions

Mode-specific mini cancel buttons must not proliferate when a shared cancel surface exists.

---

## Ownership Rules

### `Index.tsx`

Responsible for:

- mode-aware shortcut dispatch
- deciding whether a shortcut is allowed in current route state
- bridging shortcuts into:
  - orchestrator mode changes
  - route creation actions
  - detour history actions
  - top-drawer shortcut requests
  - map imperative actions

### `RouteMap.tsx`

Responsible for:

- exposing imperative map controls through `mapActionsRef`
- pan / zoom execution against the live Leaflet instance

It must not own global keyboard policy.

### `TopActionDrawer.tsx`

Responsible for:

- interpreting shortcut requests such as:
  - `search`
  - `gpx`
  - `rwgps`
- opening the correct subpanel using the same internal state transitions as button clicks

It must not register global shortcut listeners itself.

---

## Behavioral Invariants

1. Shortcuts must never fire while the user is typing into a text field.
2. Keyboard actions must call the same action paths as the visible UI.
3. Map panning/zooming must target the live Leaflet map, not stale center state.
4. Top-drawer shortcuts must open the drawer into a deterministic subpanel.
5. Route editing shortcuts must respect current mode and available history.
6. No shortcut may silently create a second implementation path for upload/search/edit.
7. `C` must always target the same currently active cancel action shown by the bottom red cancel surface.

---

## Non-Goals

This spec does **not** yet define:

- configurable user remapping
- accessibility keymaps distinct from the default map
- platform-specific alternatives
- Vim-style continuous camera movement

Those can be added later, but only as extensions to this contract.

---

## Future Extensions

Possible future additions:

- `?` to open a shortcut cheat sheet
- mode badges showing currently active shortcut family
- command palette integration

Any future shortcut must be added here before implementation.


---

## Source File: docs/02-architecture/design/ds-027-poi_ingestion_selection_and_cluster_interaction_spec.md

# DS-027 — POI Ingestion, Selection, and Cluster Interaction Spec

Status: Draft
Date: 2026-04-22

Related:
- ADR-017 — Local OSM-Derived Data Strategy
- ADR-019 — Route Corridor & Proximity Rules
- ADR-027 — Lantern Screen Model
- DS-008 — Route Corridor Model
- DS-018 — Viewport Overlay Hydration and Client Budget

---

## 1. Purpose

This document defines the canonical architecture for POIs in Lanterne.

It exists because the current POI stack still reflects the original prototype:

- raw fetch and cache semantics are mixed with category inference
- route-distributed selection is partially present but undone later by global pruning
- cluster counts and cluster-open behavior do not share one truth model
- map rendering, cluster interaction, and selection budgeting are coupled too tightly

The goal of this spec is to make POIs follow the same architecture pattern already used successfully for speed and traffic:

1. multiple raw inputs
2. one canonical normalized POI object
3. one canonical selection and budgeting layer
4. one canonical cluster/reveal interaction model
5. many subscribers

---

## 2. Current Failure Modes

Observed prototype failures:

- POIs concentrated in one or two dense pockets on long routes
- bubble counts that do not match what becomes visible after opening a cluster
- cluster click behaving like incremental zoom rather than reveal
- custom OMS responses causing category or name loss
- cached tiles preserving stale `unknown` category/type data

These are not separate bugs. They are consequences of missing ownership boundaries.

---

## 3. Canonical Layers

### 3.1 Raw Source Layer

Inputs may include:

- Overpass / custom OMS tile responses
- cached tile records
- future government or commercial service feeds

Raw inputs must not be treated as UI-ready POIs.

### 3.2 Canonical POI Object

Every raw record must normalize into one canonical object:

```ts
interface CanonicalPoi {
  id: string;
  sourceType: 'osm_node' | 'osm_way' | 'osm_relation' | 'feed';
  sourceId: string;
  lat: number;
  lon: number;
  canonicalType: PoiType;
  displayName?: string;
  hours?: string;
  phone?: string;
  website?: string;
  category: PoiCategory;
  provenance: {
    queryType?: string;
    rawTags?: Record<string, string>;
    tileKey?: string;
    cacheAgeMs?: number;
  };
}
```

Rules:

- raw query intent may be used as a fallback when tag-based type inference fails
- name fallback chain must be explicit
- `unknown` is allowed internally but should be rare and observable

### 3.3 Route Selection Layer

Route selection must own:

- corridor filtering
- route-distance relevance
- route-distributed budgeting
- per-category balancing

It must not globally rank by nearest-to-route and then cap from the top.

The selector must allocate along the route by chunks / mile bands so a 100+ mile loop cannot collapse into one town.

### 3.4 Cluster Interaction Layer

Cluster bubbles and revealed leaves must come from the same POI universe.

Required rule:

- if a bubble says `N`, a single cluster-open action must be able to reveal all `N` members at the current zoom

Allowed implementations:

- spiderfy / fan-out reveal at current zoom
- local expanded-cluster rendering state

Disallowed default:

- “click cluster, zoom a little, maybe reveal some”

### 3.5 Presentation Layer

Subscribers must consume canonical POI presentation data:

- label
- category
- icon
- marker color
- route context fields

Map popups, cards, and future POI lists must not each invent their own fallback naming logic.

---

## 4. Non-Negotiable Invariants

1. Category truth must survive cache reads.
2. Cluster counts must match revealable membership.
3. Route distribution beats dense-pocket nearest sorting.
4. Rendering caps may limit work, but they may not lie about cluster membership.
5. Query intent may rescue malformed raw tags, but provenance must record that fallback.

---

## 5. Immediate Implementation Direction

The next implementation pass must:

1. keep canonical type fallback from query intent
2. normalize cached `unknown` category/type records on read
3. replace global nearest-to-route pruning with route-band allocation
4. replace cluster click incremental zoom with reveal-all-at-current-zoom behavior
5. move POI label/category/icon formatting into a shared presentation helper

---

## 6. Deferred Work

- POI trust levels and provenance badges
- route-time aware POI availability filtering by opening hours
- bidirectional route-phase aware service relevance
- ride-mode POI presentation surfaces

---

## 7. Success Criteria

A long loop should show:

- POIs spread around the route
- cluster counts that match revealable items
- meaningful names instead of `Unknown`
- one-click cluster reveal semantics

If any one of those fails, the subsystem is still behaving like the prototype.


---

## Source File: docs/02-architecture/design/ds-028-hazard_ingestion_normalization_and_presentation_spec.md

# DS-028 — Hazard Ingestion, Normalization, and Presentation Spec

Status: Draft
Date: 2026-04-22

Related:
- ADR-017 — Local OSM-Derived Data Strategy
- ADR-019 — Route Corridor & Proximity Rules
- ADR-020 — Atomic Route Analysis Unit & OSM Variable Architecture
- ADR-027 — Lantern Screen Model
- DS-008 — Route Corridor Model

---

## 1. Purpose

This document defines the canonical hazard architecture for Lanterne.

It exists because the hazard stack still reflects the original prototype:

- different fetch/parser paths do not ingest the same hazard-supporting OSM elements
- hazard normalization is mixed into corridor parsing and route-analysis export
- labels and marker semantics are duplicated in subscribers
- some hazards are route-truth facts, others are rider-facing presentation artifacts, and the system does not cleanly separate those roles

This spec moves hazards toward the same model already used for speed and traffic:

1. multiple raw inputs
2. one canonical normalized hazard object
3. one presentation helper
4. many subscribers

---

## 2. Hazard Families

Current supported families:

- railroad crossings
- controlled crossings
  - signals
  - stop-controlled nodes
- bridge and span hazards
  - metal grate bridge
  - metal plate bridge
  - dismount bridge
  - covered bridge
  - no-shoulder bridge
  - narrow underpass
- node hazards
  - cattle grid

Future hazard families may include rider reports, agency feeds, and surface incidents, but they must normalize into the same canonical contract.

### 2.1 Clear Taxonomy

For implementation and debugging purposes, hazards should be understood in four groups:

1. traffic hazards
2. structural hazards
3. weather / water hazards
4. surface hazards

The important distinction is:

- God Filter answers whether a raw OSM object exists
- rider-facing hazards answer whether that object affects the route

The same raw source may feed both, but rider-facing hazards still pass route-association and presentation policy.

### 2.2 Current Implemented Hazard Tags

The table below reflects the current code, not a wish list.

| Family | Hazard | Current coded tags / rules | Current source path |
|------|-------|-----------------------------|---------------------|
| Traffic | Railroad crossing | `railway=level_crossing` or `railway=crossing` explicit node; geometric fallback from `railway=rail` intersecting a `highway=*` road | self-hosted roads corridor payload |
| Traffic | Signal crossing | `highway=traffic_signals`, excluding pedestrian-only / crosswalk-signal patterns such as `crossing=traffic_signals`, `crossing:signals=yes`, `traffic_signals:sound=yes`, `button_operated=yes`, `highway=crossing`, `crossing:island=yes`, `foot=designated`, `bicycle=dismount` | self-hosted roads corridor payload |
| Traffic | Stop-controlled crossing | `highway=stop` | self-hosted roads corridor payload |
| Structural | Cattle grid | `barrier=cattle_grid` | self-hosted roads first, supplemented by raw Overpass-compatible source for freshness |
| Structural | Metal grate bridge | `bridge=yes|movable` plus `surface` in `{grate, grid, metal_grid, steel_grid, grating, steel_grate, metal_grating, steel_grating, open_grid, metal_grid_deck}` | self-hosted roads corridor payload |
| Structural | Metal plate bridge | `bridge=yes|movable` plus `surface` in `{metal, steel, metal_plate, steel_plate}` | self-hosted roads corridor payload |
| Structural | Dismount bridge | `bridge=yes|movable` plus `bicycle=dismount` | self-hosted roads corridor payload |
| Structural | Covered bridge | `bridge=yes|movable` plus `bridge:structure=covered` | self-hosted roads corridor payload |
| Structural | No-shoulder bridge | `bridge=yes|movable` plus explicit `shoulder=no|none` and explicit narrowness evidence: bidirectional `lanes=1`, or `width<6`, or `maxwidth<3` | self-hosted roads corridor payload |
| Structural | Narrow underpass | `tunnel=yes` plus explicit narrowness evidence: bidirectional `lanes=1`, or `width<3`, or `maxwidth<3`; excluded on `highway` in `{service, parking_aisle, driveway, alley}` | self-hosted roads corridor payload |

### 2.3 Candidate Hazard Tags Not Yet Canonical

These are plausible next additions, but they are not yet part of the canonical rider-facing hazard contract unless implemented and tested.

| Family | Candidate | Likely tags / source hints | Notes |
|------|-----------|----------------------------|-------|
| Weather / water | Ford / stream crossing | `ford=yes`, `highway=ford`, possibly `ford=*` on nodes or ways | Particularly relevant in places like Maui; should likely be weather-conditional rather than always severe |
| Surface | Pothole / pavement distress | no stable universal OSM tag today; possible rider reports or agency feeds | Better handled as reports/feed-backed hazards than assumed from generic OSM tags |
| Surface | Cracked / failed pavement | agency pavement feeds, rider reports, or future normalized surface distress feed | Not currently canonical from OSM corridor fetch |
| Structural / control | Gate / bollard | `barrier=gate`, `barrier=bollard` | Likely useful for debug and raw inspection first; rider-facing policy should be narrow |

### 2.4 What “Exact Tag” Means Here

When this spec says “exact tag,” it means:

- the OSM key/value or explicit rule that currently triggers normalization
- not a fuzzy UI label
- not a broad nearby-object heuristic

Examples:

- `barrier=cattle_grid` is a canonical cattle-guard trigger
- `bridge:structure=covered` is a canonical covered-bridge trigger
- `highway=traffic_signals` is only canonical after pedestrian-signal exclusions are applied
- `railway=rail` alone is not a rider-facing hazard; it only participates in railroad-crossing geometric fallback

---

## 3. Canonical Layers

### 3.1 Raw Ingestion Layer

Hazard-relevant OSM elements may arrive from:

- corridor tile fetch
- legacy overpass fetch
- future observation/feed systems

All live fetch paths must request the same minimum hazard-supporting elements:

- `railway=level_crossing|crossing`
- `barrier=cattle_grid`
- `highway=traffic_signals`
- `highway=stop`
- `railway=rail`
- `man_made=bridge`
- road ways carrying hazard-relevant bridge/tunnel/surface tags

For clarity:

- road-shaped hazard context should come from the owned/self-hosted roads server first
- raw object hazards may be supplemented by a raw Overpass-compatible source where freshness matters
- rider-facing hazard parity requires both sources to normalize into one canonical hazard object model

Any parser that omits one of these is non-canonical.

### 3.2 Canonical Hazard Object

Every raw hazard candidate must normalize into one canonical object:

```ts
interface CanonicalHazard {
  id: string;
  kind: HazardKind;
  severity: 1 | 2 | 3;
  lat: number;
  lon: number;
  spanEntry?: [number, number];
  spanExit?: [number, number];
  sourceType: 'osm_node' | 'osm_way' | 'bridge_outline' | 'feed';
  osmWayId?: number;
  osmNodeId?: number;
  routeSnap?: {
    distM: number;
    routeMile: number;
  };
  provenance: {
    rawTags?: Record<string, string>;
    inheritedFromOutline?: boolean;
    detectionMode?: 'explicit_node' | 'line_intersection' | 'way_tags';
  };
}
```

Rules:

- ingestion and normalization own hazard truth
- downstream consumers may project or filter hazards, but may not reinterpret kind or label semantics

### 3.3 Route Attachment Layer

Route-analysis owns:

- route snapping
- attachment to route distance
- on-route filtering thresholds
- export to canonical analysis payload

This layer must not invent new hazard kinds.

### 3.4 Presentation Layer

One shared hazard presentation helper must own:

- rider-facing label
- emoji / icon semantics
- marker family
- severity-facing styling hooks

Route map, ride overlay, and future cards must subscribe to that helper instead of duplicating strings or marker choices.

---

## 4. Current Failure Modes

Prototype-shaped failures already observed:

- some hazard-supporting node types requested in one fetch path but not another
- parser drift between corridor and legacy overpass code
- duplicate label dictionaries inside map rendering
- hazard disappearance caused by ingestion mismatch rather than detector logic
- debug/user divergence where God Filter can see a raw object but rider-facing hazards cannot, because they are not using the same raw object semantics before route gating

These failures are architectural, not just bug-level.

---

## 5. Non-Negotiable Invariants

1. All analysis fetch paths must ingest the same hazard-supporting OSM primitives.
2. Hazard kind normalization happens once.
3. Presentation labels happen once.
4. Subscribers may choose visibility policy, but not invent new hazard meaning.
5. Route snapping thresholds must be explicit and auditable.

---

## 6. Immediate Implementation Direction

The next implementation pass must:

1. unify corridor and legacy fetch hazard-node coverage
2. keep `detectHazardsFromRoads()` as the canonical raw detector
3. centralize rider-facing label/icon decisions in a hazard presentation helper
4. remove duplicated hazard label dictionaries from subscriber surfaces
5. preserve provenance about explicit-node vs geometry-fallback detection
6. preserve source lineage on hazard objects:
   - `self_hosted_roads`
   - `raw_overpass`
   - `merged`
7. keep route association separate from source existence:
   - source answers “does the object exist?”
   - route policy answers “does it affect this route?”

### 6.1 Current Source Policy

Current intended source routing:

- self-hosted roads server is primary for road-shaped hazards and road-context tags
- raw Overpass-compatible fetch is supplement/fallback for raw object hazards and fresh edits
- cattle grids are the first narrow hazard family using this split explicitly

This means:

- God Filter may surface raw objects without rider gating
- rider-facing hazard layers may only surface those objects after route association and hazard-family policy

---

## 7. Deferred Work

- hazard confidence/trust display
- hazard provenance receipts
- rider-report hazard integration
- hazard-specific inspect surfaces
- canonical hazard feed persistence

---

## 8. Success Criteria

The hazard subsystem is considered healthy when:

- the same route yields the same hazard families regardless of fetch path
- controlled crossings, railroad crossings, and bridge hazards all survive corridor ingestion
- map and overlay surfaces agree on labels and marker meaning
- hazard regressions are diagnosable at the canonical object layer, not by chasing duplicated UI logic


---

## Source File: docs/02-architecture/design/ds-029-provenance_precedence_confidence_and_traceability_spec.md

# DS-029 - Provenance, Precedence, Confidence, and Traceability Specification

**Status:** Draft, partially implemented  
**Date:** 2026-05-05  
**Filename:** `ds-029-provenance_precedence_confidence_and_traceability_spec.md`  
**Related:** [DS-015](./ds-015-safety_scoring_model.md), [DS-017](./ds-017-truth_resolution_and_propagation_spec.md), [DS-020](./ds-020-confidence_model_and_tracing.md), [DS-022](./ds-022-speed_prior_and_area_baseline_policy_spec.md), [DS-028](./ds-028-hazard_ingestion_normalization_and_presentation_spec.md), [DS-030](./ds-030-route_analysis_contract.md)

## 1. Purpose

This document is the single archival reference for how Lanterne must handle:

- provenance families
- concrete source types
- precedence
- field-level confidence anchors
- confidence traceability
- traceability requirements
- fallback math for predicted and baseline values

It exists because the current system is split across several partially overlapping documents:

- `DS-015` defines the canonical provenance families.
- `DS-020` defines launch confidence behavior, but does so partly at the source-type layer.
- `DS-022` defines speed prior policy, but not the full cross-field provenance contract.

This document resolves that ambiguity by defining one hierarchy:

```text
DS-015 provenance family
  -> concrete source type
    -> source-specific math / eligibility
      -> required trace payload
        -> rider-facing explanation
```

It also defines the symmetry rule that older documents and implementations have
blurred:

```text
every score-driving truth input must have a matching confidence treatment
every provenance rule must apply symmetrically to traffic and speed unless this spec explicitly says otherwise
every inspectable explanation must exist at route, road, and input levels
```

## 2. Architectural Decision

Lanterne must use one canonical provenance architecture for both traffic and speed.

Traffic and speed may differ in:

- direct source availability
- field-specific math
- fallback datasets

They may not differ in:

- provenance family semantics
- precedence semantics
- confidence downgrade logic
- confidence traceability expectations
- traceability expectations
- parity between analysis-time truth and inspect-time truth

Risk and confidence are peer systems.

This means:

- anything that materially affects canonical risk must materially affect canonical confidence
- confidence must be computed from the same chosen inputs, provenance, and fallback structure as risk
- no field may have a richer provenance model for risk than it has for confidence
- no surface may explain risk in more detail than it can explain the confidence implications of the same chosen inputs

The canonical ordering is:

1. family
2. concrete source type
3. derivation and trace payload
4. presentation text

No presentation surface may invent provenance semantics independently.

## 2.1 Symmetry invariant

The following symmetry invariant is mandatory:

```text
if traffic and speed both participate in risk,
then traffic and speed must both participate in:
  provenance
  precedence
  confidence
  receipts
  inspectability
  score trace export
```

No implementation may justify asymmetry merely because one field historically
had richer treatment than the other.

## 3. Canonical Provenance Families

These are the only canonical DS-015 provenance families for score-bearing truth:

| Family | Meaning |
| --- | --- |
| `observed` | Direct field measurement or validated first-hand evidence |
| `official_imported` | Authoritative agency source such as DOT, HPMS, or equivalent |
| `geometry_derived` | Deterministic derivation from route or road geometry / OSM explicit tag truth |
| `relationship_inferred` | Inferred from stronger nearby or related evidence through continuity or other documented relationship |
| `predicted` | Model output from local or contextual evidence |
| `baseline` | Generic fallback prior used because stronger evidence is absent |
| `unknown` | No reliable source or derivation available |

## 4. Canonical Source-Type Table

The following table is authoritative for source-type semantics, precedence, confidence anchors, and trace requirements.

### 4.1 Score-bearing source types

| Precedence | Source type | Family | Launch source-confidence anchor | Canonical confidence band | Trace requirements |
| ---: | --- | --- | ---: | --- | --- |
| 1 | `observed` | `observed` | 100 | `very_high` | observation id, observer class, timestamp or audit ref, anchor segment, source links when available |
| 2 | `admin_approved` | `observed` | 100 | `very_high` | admin identity, timestamp, approval ref, supporting verification links; for speed, include the Street View link used to verify the sign |
| 3 | `authoritative_posted` | `official_imported` | 100 | `very_high` | agency, feed id, record id or statute ref, public source URL when available |
| 4 | `osm_posted` | `geometry_derived` | 95 | `very_high` | OSM way id, tag key/value, matched road identity, OSM or lat/lon link |
| 5 | `observation_inferred` | `relationship_inferred` | 90 | `high` | anchor source identity, propagation path, continuity reason, decay / distance context, source links when available |
| 6 | `authoritative_inferred` | `relationship_inferred` | 90 | `high` | agency/feed id, propagation path, continuity reason, decay / distance context, source links when available |
| 7 | `osm_inferred` | `relationship_inferred` | 85 | `high` | matched road identity or anchor source identity, propagation / derivation path, continuity or inference reason, decay / distance context, OSM or lat/lon link |
| 8 | `local_area_predicted` | `predicted` | 80 | `high` | source trace, contributors, aggregation math, rejection reasons, fallback reasons, source links for contributors when available |
| 9 | `regional_prior` | `baseline` | 65 | `medium` | agency, dataset row id or lookup key, jurisdiction, context selectors, default value, source document link |
| 10 | `highway_area_baseline` | `baseline` | 55 | `low_medium` | agency or governing table identity, lookup dimensions, table/version key, exact row or generic context rule, source document link |
| 11 | `highway_baseline` | `baseline` | 40 | `low` | agency or governing table identity, highway class lookup key, table/version key, source document link |
| 12 | `unknown` | `unknown` | 20 | `low` | explicit missingness reason where possible |

### 4.2 Non-score-bearing or reserved source types

These may exist operationally, but they are not currently canonical score-bearing truth:

| Source type | Status | Notes |
| --- | --- | --- |
| `measured` | reserved | May later promote above `observed`, but is not yet canonical truth |
| `community_reported` | reserved / pending policy | If later promoted, it must receive its own family mapping, precedence, confidence anchor, and trace requirements rather than being folded into `geometry_derived` |
| `user_observation` | presentation-only / session-only | Must not silently enter canonical score truth without a future DS/ADR change |

### 4.2.1 `admin_approved` scope

`admin_approved` is a canonical observed-family source type, but it is not a
free-form escape hatch.

It must represent one of these audited cases:

1. manual validation of a directly observed real-world condition
2. manual confirmation of a clearly visible mapped or source-backed condition
3. explicit admin override of the chosen value where other sources are judged
   wrong or stale

Trace requirements must distinguish:

- internal admin identifier
- public display label if shown in rider-facing surfaces
- approval timestamp
- supporting verification artifact
- whether the admin action confirmed an existing source or replaced it

`admin_approved` may override value, provenance, and confidence treatment, but
the overridden source should remain visible in audit trace when available.

### 4.2.2 Operational source lineage vs DS-015 source type

This spec now distinguishes two different things that earlier drafts blurred:

1. DS-015 provenance family / concrete source type
2. operational source lineage

The first is canonical for score-bearing truth. The second is canonical for
transport, freshness, and hazard/object traceability.

Examples:

- `authoritative_posted`
- `osm_posted`
- `local_area_predicted`
- `highway_area_baseline`

are concrete source types.

By contrast:

- `self_hosted_roads`
- `raw_overpass`
- `merged`

are operational lineage values. They answer:

- where the object or hazard record came from
- whether it came from the owned normalized roads service, a raw OSM object
  supplement, or both

They do **not** replace DS-015 provenance families for score-bearing truth.

Current rule:

- score-bearing traffic, speed, facility, shoulder, and crossing truth must
  still resolve through the DS-015 family/source-type architecture
- object-level hazard and debug/God Filter records must preserve operational
  lineage in addition to any provenance-facing semantics they carry

### 4.2.3 Current live operational lineage values

The live system currently uses these operational lineage values where object- or
hazard-level source routing matters:

| Lineage | Meaning |
| --- | --- |
| `self_hosted_roads` | came from the owned normalized roads/corridor service first |
| `raw_overpass` | came from raw OSM object semantics via raw Overpass-compatible query path |
| `merged` | same logical object/hazard was observed from both owned normalized and raw object sources |

These lineage values are currently most important for:

- raw hazard supplements such as `barrier=cattle_grid` and `ford=*`
- God Filter object inspection
- rider-facing hazard parity debugging

They are traceability fields, not score precedence rungs.

### 4.3 Score-driving field matrix

This matrix defines which score-driving fields must participate in the full
provenance, precedence, confidence, and traceability contract.

| Field | Route-level trace | Road-level trace | Input-level trace | Predicted allowed? | Baseline allowed? | Precision retention requirement |
| --- | --- | --- | --- | --- | --- | --- |
| Speed | yes | yes | yes | yes | yes | preserve exact mph when known |
| Traffic | yes | yes | yes | yes | yes | preserve exact AADT total, AADT/lane, lane count, factor inputs when known |
| Bike lane / facility | yes | yes | yes | no | no | preserve exact facility class and side context when known |
| Shoulder | yes | yes | yes | no | no | preserve exact width basis, side, and bucket thresholds when known |
| Crossing control | yes | yes | yes | no | no | preserve exact control type, not just generic risk label |
| Crossing width | yes | yes | yes | no | no | preserve exact lane count / width basis when known |
| Crossing movement | yes | yes | yes | no | no | preserve exact movement classification when known |

Rule of thumb:

- if we know the number, show the number in trace and receipts
- if we know the selector, keep the selector
- summaries may compress presentation, but canonical trace may not throw out precision

### 4.4 Non-score-bearing hazard and object traceability

Not every route artifact participates in DS-015 scoring, but non-score-bearing
objects still require canonical traceability.

Current live examples include:

- bridge surface hazards
- cattle grids
- low-water crossings / fords
- God Filter object inspection results

For these non-score-bearing objects, the canonical trace contract is:

| Field | Requirement |
| --- | --- |
| hazard/object kind | must be explicit and stable |
| location | must preserve canonical coordinate |
| `osmNodeId` / `osmWayId` | preserve when known |
| raw tags | preserve when useful for audit/debug |
| source lineage | preserve `self_hosted_roads`, `raw_overpass`, or `merged` |
| route association outcome | preserve whether the object merely exists or was accepted as route-affecting |

Important distinction:

- God Filter answers: "does the raw object exist in source?"
- rider-facing hazard rendering answers: "does the object affect this route?"

Those are not the same question, even when they share the same raw source.

## 5. Confidence Model Rules

### 5.1 Field-level confidence anchors

This spec distinguishes:

- provenance family
- source-confidence anchor
- route-level confidence rollup

The table in Section 4 defines the field-level source-confidence anchors. Those anchors are the canonical starting points for `DS-020` style confidence math.

### 5.1.1 Confidence is a first-class canonical output

Confidence is not a presentation afterthought and not a rider-facing caveat
layer bolted on after risk.

Confidence must be modeled with the same seriousness as risk:

- route-level confidence is canonical
- road-level confidence is canonical
- input-level confidence is canonical

The confidence graph must consume the same:

- chosen inputs
- source types
- provenance families
- fallback steps
- propagation outcomes
- missingness states

used by risk.

At route and road levels, confidence must be weighted by actual score
participation rather than flat source averaging.

Conceptually:

```text
confidence burden of an input
= source uncertainty * risk contribution of that input
```

This preserves the original DS-020 intuition that rough approximations on
higher-severity, higher-risk roads should hurt confidence more, without
distorting provenance semantics.

### 5.1.2 Canonical confidence bands

Unless a future DS explicitly revises them, canonical confidence bands are:

| Confidence anchor / rollup | Canonical band |
| ---: | --- |
| 90-100 | `very_high` |
| 75-89 | `high` |
| 60-74 | `medium` |
| 40-59 | `low_medium` |
| 0-39 | `low` |

Bands are presentation projections of numeric confidence. They do not alter
precedence and do not replace exact numeric confidence in trace.

### 5.1.3 Route and road confidence rollup

The canonical route/road confidence model must be executable, not merely
conceptual.

Definitions:

```text
sourceUncertainty = 1 - (sourceConfidenceAnchor / 100)
normalizedRiskShare_i = riskContribution_i / totalRelevantRisk
uncertaintyBurden_i = sourceUncertainty_i * normalizedRiskShare_i
confidencePenalty = sum(uncertaintyBurden_i)
confidenceScore = 100 * (1 - confidencePenalty)
```

Rules:

- `riskContribution_i` is the canonical local score contribution of the chosen
  input, slice, or crossing input under the same score graph used by risk
- `totalRelevantRisk` is the sum of the relevant road and crossing
  contributions in the scope being scored
- if `totalRelevantRisk <= 0`, confidence falls back to the chosen-value mix
  summary rather than dividing by zero
- zero-risk or non-operative inputs contribute zero confidence burden
- route confidence includes both road and crossing domains
- road-domain confidence may be computed separately from route confidence for
  audit and UX needs
- unknown or missing score-driving inputs contribute using their `unknown`
  anchor and the same risk-weighting rule

This formula is canonical for phase 1 unless superseded by a future DS revision.

### 5.2 Confidence downgrade rule

Confidence must step down monotonically as provenance weakens:

```text
observed / official_imported > geometry_derived > relationship_inferred > predicted > baseline > unknown
```

Within a family, more specific or less lossy sources must score higher than more generic ones.

### 5.3 Confidence is not precedence

Precedence decides which value wins.

Confidence explains how trustworthy the winning value is.

No implementation may use confidence alone to overrule the precedence ladder.

### 5.4 Confidence symmetry rule

Every source-type row in Section 4 carries both:

- a risk-facing provenance meaning
- a confidence-facing trust anchor

No source type is complete unless both are specified.

This is especially important for:

- `local_area_predicted`
- `regional_prior`
- `highway_area_baseline`
- `highway_baseline`

because these are exactly the sources most likely to accumulate asymmetry
between risk treatment and confidence treatment.

## 6. Field-Specific Precedence Ladders

This section is organized by DS-015 family first and concrete source type
second. If a field has no rung in a family, the ladder proceeds to the next
family rather than inventing symmetry where none exists.

### 6.1 Speed

- `observed`
  - `observed`
  - `admin_approved`
- `official_imported`
  - `authoritative_posted`
- `geometry_derived`
  - `osm_posted`
- `relationship_inferred`
  - `observation_inferred`
  - `authoritative_inferred`
  - `osm_inferred`
- `predicted`
  - `local_area_predicted`
- `baseline`
  - `regional_prior`
  - `highway_area_baseline`
  - `highway_baseline`
- `unknown`

### 6.2 Traffic

- `observed`
  - `observed`
  - `admin_approved`
- `official_imported`
  - direct official AADT per lane
  - official total AADT plus known lane count
- `relationship_inferred`
  - official total AADT plus relationship-inferred lane count
  - relationship-inferred total AADT from nearby stronger truth through corridor continuity
- `predicted`
  - `local_area_predicted`
- `baseline`
  - `highway_area_baseline` via class proxy / generic class traffic fallback
- `unknown`

Implementation note:

- the traffic truth object may still carry internal confidence labels such as `official_per_lane` or `class_proxy`
- those are not rider-facing semantic families
- they must map through this spec before presentation, receipts, or score trace export
- `regional_prior` and `highway_baseline` are not active traffic ladder rungs
  in the current canonical model even though they remain global source types in
  the master source table

### 6.3 Bike Lane / Facility

- `observed`
  - `observed`
  - `admin_approved`
- `official_imported`
  - authoritative imported facility truth when such a feed exists
- `geometry_derived`
  - explicit mapped facility truth from OSM or equivalent geometry-linked tags
- `relationship_inferred`
  - continuity-based carried facility truth where allowed by DS-017
- `unknown`

No `predicted` or `baseline` ladder exists for bike facility at this time.

### 6.4 Shoulder

- `observed`
  - `observed`
  - `admin_approved`
- `official_imported`
  - authoritative imported shoulder truth when such a feed exists
- `geometry_derived`
  - explicit mapped shoulder width / presence truth from OSM or geometry-linked tags
- `relationship_inferred`
  - continuity-based carried shoulder truth where allowed by DS-017
- `unknown`

No `predicted` or `baseline` ladder exists for shoulder at this time.

### 6.5 Crossing Control

- `observed`
  - `observed`
  - `admin_approved`
- `official_imported`
  - authoritative imported control truth
- `geometry_derived`
  - explicit mapped control truth from OSM or geometry-linked network tags
- `unknown`

### 6.6 Crossing Width

- `observed`
  - `observed`
  - `admin_approved`
- `official_imported`
  - authoritative imported lane / width truth
- `geometry_derived`
  - lane count, crossed width, or width proxy derived from mapped geometry
- `unknown`

### 6.7 Crossing Movement

- `geometry_derived`
  - route geometry / topology movement classification
- `unknown`

Crossing movement is included for parity, but it is currently geometry-derived
only.

### 6.9 Current live source-routing policy

The current live implementation uses a split source-routing model:

1. self-hosted normalized roads data is primary for road-shaped truth
2. raw Overpass-compatible object fetch is supplement/fallback for raw object
   semantics and freshness

This applies today as follows:

| Domain | Primary | Supplement / fallback | Notes |
| --- | --- | --- | --- |
| speed / traffic / facility / shoulder / route matching | `self_hosted_roads` | none by default | canonical road-shaped truth |
| rider-facing cattle grids | `self_hosted_roads` | `raw_overpass` | same raw semantics as God Filter, then route-gated |
| rider-facing fords | `self_hosted_roads` | `raw_overpass` | same raw semantics as God Filter, then route-gated |
| God Filter road-shaped queries | `self_hosted_roads` first when plausible | `raw_overpass` fallback when needed | object inspection path, not score truth |
| God Filter raw object queries | query-classified; may go raw first or fall back raw | `raw_overpass` | uses owned proxy entrypoint, not browser-direct public Overpass |

Additional live rule:

- operational source routing should prefer the owned/self-hosted roads service
  first where the owned normalized payload can answer the question
- raw object fetch should be used where object-level semantics or edit freshness
  are required
- direct browser dependency on public Overpass is non-canonical; raw fetches
  should go through an owned proxy entrypoint

### 6.8 Master field/source matrix

This table is the compact execution reference for score-driving fields. It is
redundant with the ladders above on purpose. The ladders explain the model. This
table makes implementation and audit faster.

| Field | Family rung | Concrete source types / truth forms | Predicted allowed? | Baseline allowed? | Resolved options / classification | Precision and trace expectations |
| --- | --- | --- | --- | --- | --- | --- |
| Speed | `observed` | `observed`, `admin_approved` | yes | yes | exact posted or observed mph | retain exact mph; trace source link; for admin-approved speed, retain Street View verification link |
| Speed | `official_imported` | `authoritative_posted` | yes | yes | exact posted mph | retain exact mph plus agency/feed/statute link |
| Speed | `geometry_derived` | `osm_posted` | yes | yes | exact mapped maxspeed mph | retain exact mph plus OSM way/tag and OSM or lat/lon link |
| Speed | `relationship_inferred` | `observation_inferred`, `authoritative_inferred`, `osm_inferred` | yes | yes | exact carried or inferred mph | retain exact mph plus propagation/derivation path, continuity reason, decay/distance context |
| Speed | `predicted` | `local_area_predicted` | yes | yes | exact predicted mph, rounded to nearest 5 only where the method requires it | retain exact mph, contributors, radius, aggregation math, rounding, fallback reasons |
| Speed | `baseline` | `regional_prior`, `highway_area_baseline`, `highway_baseline` | yes | yes | exact default mph from the selected row/table | retain exact lookup key, table/version, default mph, source document link, concrete lookup path |
| Traffic | `observed` | `observed`, `admin_approved` | yes | yes | exact AADT total and AADT/lane when known | retain exact AADT, lane basis, factor math, and verification/source links |
| Traffic | `official_imported` | direct official AADT/lane; official total AADT + known lane count | yes | yes | exact AADT/lane or exact AADT total + lane count | retain exact AADT total, AADT/lane, lane count, factor math, agency/feed link |
| Traffic | `relationship_inferred` | official total AADT + relationship-inferred lane count; corridor-carried AADT | yes | yes | exact carried AADT total and derived AADT/lane | retain exact AADT inputs, lane inference basis, propagation path, factor math |
| Traffic | `predicted` | `local_area_predicted` | yes | yes | exact predicted AADT total and derived AADT/lane | retain exact predicted AADT total, derived AADT/lane, contributor set, radius, aggregation math |
| Traffic | `baseline` | class proxy via `highway_area_baseline` | yes | yes | exact class-proxy AADT/lane and resulting factor | retain class-proxy AADT/lane, factor math, governing table/document link |
| Bike lane / facility | `observed` | `observed`, `admin_approved` | no | no | `path / MUP`, `protected`, `buffered`, `painted`, `shared`, `none` | retain exact facility class, side context if relevant, source or Street View verification link |
| Bike lane / facility | `official_imported` | authoritative imported facility truth | no | no | `path / MUP`, `protected`, `buffered`, `painted`, `shared`, `none` | retain exact facility class and source link |
| Bike lane / facility | `geometry_derived` | explicit mapped OSM / geometry-linked facility truth | no | no | `path / MUP`, `protected`, `buffered`, `painted`, `shared`, `none` | retain exact facility class, tag basis, OSM or lat/lon link |
| Bike lane / facility | `relationship_inferred` | continuity-carried facility truth | no | no | `path / MUP`, `protected`, `buffered`, `painted`, `shared`, `none` | retain facility class plus propagation path and continuity reason |
| Shoulder | `observed` | `observed`, `admin_approved` | no | no | `none`, `narrow`, `regular`, `wide` | retain exact width basis, side, threshold bucket, source or Street View verification link |
| Shoulder | `official_imported` | authoritative imported shoulder truth | no | no | `none`, `narrow`, `regular`, `wide` | retain exact width / shoulder truth and source link |
| Shoulder | `geometry_derived` | explicit mapped width/presence truth | no | no | `none`, `narrow`, `regular`, `wide` | retain exact width basis or mapped bucket plus OSM/lat-lon link |
| Shoulder | `relationship_inferred` | continuity-carried shoulder truth | no | no | `none`, `narrow`, `regular`, `wide` | retain width/bucket plus propagation path and continuity reason |
| Crossing control | `observed` | `observed`, `admin_approved` | no | no | `none`, `signed`, `signaled` | retain exact control type and visual/source verification link |
| Crossing control | `official_imported` | authoritative imported control truth | no | no | `none`, `signed`, `signaled` | retain exact control type and source link |
| Crossing control | `geometry_derived` | explicit mapped control truth | no | no | `none`, `signed`, `signaled` | retain exact control type, tag basis, OSM/lat-lon link |
| Crossing width | `observed` | `observed`, `admin_approved` | no | no | exact crossed lane count and exact width basis when known | retain exact lane count / width and visual/source verification link |
| Crossing width | `official_imported` | authoritative imported lane/width truth | no | no | exact crossed lane count and exact width basis when known | retain exact lane count / width and source link |
| Crossing width | `geometry_derived` | mapped lane count / width proxy / crossed width derivation | no | no | exact crossed lane count and exact width basis when known | retain exact lane count, width proxy, and derivation math |
| Crossing movement | `geometry_derived` | route geometry / topology movement classification | no | no | `straight`, `left_across`, `right_merge`, `join`, `exit`, `path_crossing`, `unknown` | retain exact movement classification and derivation basis |

## 7. Explicit Math Contracts

This section is intentionally prescriptive. It exists to replace ambiguous prose with explicit calculation contracts.

### 7.1 `local_area_predicted` speed

Eligibility:

- no stronger speed source won
- target is a motor-road through class: `trunk`, `primary`, `secondary`, or `tertiary`
- eligible contributors are equal-or-lower through-road classes:
  - trunk target may use trunk, primary, secondary, tertiary
  - primary target may use primary, secondary, tertiary
  - secondary target may use secondary, tertiary
  - tertiary target may use tertiary only
- same urbanicity when available and urban/rural context must not cross-match
- within local search radius and within corridor-bounded search space
- minimum 2 eligible contributors
- contributors must be direct stronger truth only:
  - `observed`
  - `admin_approved`
  - `authoritative_posted`
  - `osm_posted`

Search radius:

- `5 mi` for all urbanicity bands

The route-analysis implementation only has a route/corridor-local contributor
pool today, but the explicit cap remains `5 mi` so predicted speed does not
become a broad regional prior in disguise.

Computation:

```text
broadClass = highwayType with `_link` stripped
contributors = eligible nearby speed readings within radius
rawMean = average(contributor speeds)
valueFinal = round(rawMean to nearest 5 mph)
```

Confidence anchor:

- `80`

Required stored trace:

- radius used
- class context
- area / urbanicity context
- contributor count
- capped contributor identities and values
- aggregation method
- raw result
- rounding rule
- reasons stronger sources did not win
- source links for each contributor when available
- source document / Street View links when applicable

### 7.2 `local_area_predicted` traffic

Eligibility:

- no stronger traffic source won
- same broad highway class
- same urbanicity when available and urban/rural context must not cross-match
- within local search radius and within corridor-bounded search space
- minimum 2 eligible contributors
- contributors must be direct stronger truth only:
  - `observed`
  - `admin_approved`
  - authoritative imported numeric traffic truth

Concrete contributor rule:

- eligible traffic contributors are direct numeric traffic truths whose resolved
  family/source maps to `observed`, `admin_approved`, or `official_imported`
- `relationship_inferred`, `local_area_predicted`, and `baseline` traffic
  truths are not eligible contributors

Search radius:

- `5 mi`

Computation:

```text
broadClass = highwayType with `_link` stripped
contributors = eligible nearby total AADT readings within radius
expectedAADTTotal = round(average(contributor AADT totals))
lanes = known lane count if available, else inferred lane count
aadtPerLane = expectedAADTTotal / lanes
trafficFactor = ds015TrafficFactorFromAADTPerLane(aadtPerLane)
```

Confidence anchor:

- `80`

Required trace:

- same as speed, plus the lane-count basis used to convert total AADT into per-lane truth

### 7.3 `regional_prior`

This is a dataset or jurisdiction-backed contextual default, not a local estimate.

Speed math:

```text
lookupKey = (state, urbanicity, highwayType)
valueFinal = dataset defaultMph for best matching row
```

Traffic math:

`regional_prior` is not currently a separate canonical traffic fallback in the active DS-015 ladder. If later introduced for traffic, it must be specified here first and may not appear ad hoc in code.

Required trace:

- source document link when one exists
- dataset name/version
- agency
- row id or effective lookup key
- jurisdiction
- dimensions matched
- default value

### 7.4 `highway_area_baseline`

This is a structured generic fallback driven by road form and surrounding area context. It is weaker than `regional_prior` because it is less jurisdiction-specific and weaker than `local_area_predicted` because it is not built from nearby empirical contributors.

Speed math:

```text
if state-specific area-baseline row exists for (state, urbanicity, highwayType):
  valueFinal = row.defaultMph
else:
  valueFinal = generic area-context table(highwayType, urbanicity, metricRegion)
```

Traffic math:

```text
classProxyAadtPerLane = DS015 highway-type proxy table(highwayType)
trafficFactor = ds015TrafficFactorFromAADTPerLane(classProxyAadtPerLane)
```

Required trace:

- source document link
- agency or governing table identity
- lookup dimensions
- table/version key
- whether the result came from a state-specific row or generic context row
- exact baseline value used
- what concrete lookup produced the chosen value

### 7.5 `highway_baseline`

This is the final non-null generic fallback.

Speed math:

```text
valueFinal = generic highway-class baseline(highwayType)
```

Traffic math:

`highway_baseline` does not currently have a distinct active traffic ladder step. Traffic falls from class proxy to `unknown` under the present DS-015 contract. If a separate highway-baseline traffic layer is introduced later, it must be specified here first.

Required trace:

- source document link
- agency or governing table identity
- highway class
- table/version key
- baseline value
- what concrete lookup produced the chosen value

## 8. Canonical Traceability Contract

Every score-bearing chosen value must be inspectable. The system must be able to answer:

1. what value won
2. what source type won
3. what provenance family it belongs to
4. what stronger sources were absent or ineligible
5. what concrete math or lookup produced the chosen value
6. what exact inputs contributed if the value was derived

Traceability is required at three levels, not one.

### 8.0 Traceability levels

| Level | Canonical question | Required surfaces |
| --- | --- | --- |
| Route trace | Why did this route score and land in this confidence band the way it did overall? | scorecard, route summary, route receipt, admin audit |
| Road trace | Why did this stretch get this road risk, local confidence, and displayed receipt math? | inspect panel, segment receipt, road card expansion |
| Input trace | Why did this field resolve to this chosen value and confidence anchor? | confidence dropdowns, audit panels, expanded receipts |

These levels must link to each other:

- route trace must summarize road traces
- road trace must summarize input traces
- input trace must explain the exact source or derivation

### 8.1 Required trace depth by source type

| Source type | Minimum stored trace |
| --- | --- |
| `observed` | conditions, anchor observation ref, who/what observed it, where it applies, source links when available |
| `admin_approved` | conditions, admin identity, timestamp, approval ref, supporting verification links; for speed, include the Street View verification link |
| `authoritative_posted` | conditions, agency/feed/statute ref, matched segment or record id, source URL if public |
| `osm_posted` | conditions, OSM way id plus tag key/value, OSM or lat/lon link |
| `observation_inferred` | conditions, anchor source identity, propagation path, continuity reason, decay / distance context |
| `authoritative_inferred` | conditions, agency/feed id, propagation path, continuity reason, decay / distance context, source links when available |
| `osm_inferred` | conditions, matched road or anchor source identity, propagation / derivation path, continuity or inference reason, decay / distance context, OSM or lat/lon link |
| `local_area_predicted` | conditions, contributor list, radius, aggregation, rounding, rejection reasons, fallback reasons, source links for contributors |
| `regional_prior` | conditions, dataset row / lookup key, agency, matched context, default value, source document link |
| `highway_area_baseline` | conditions, governing table / context row, matched dimensions, exact lookup used, source document link |
| `highway_baseline` | conditions, highway-class baseline key, exact lookup used, source document link |
| `unknown` | explicit missingness reason when available |

### 8.1.4 Non-score-bearing hazard trace contract

For rider-facing but non-score-bearing hazards, expanded trace must still be
able to answer:

1. what hazard kind this is
2. where it is on the route
3. what route mile it occurs near
4. whether it came from `self_hosted_roads`, `raw_overpass`, or `merged`
5. which OSM object it came from when known
6. whether it merely exists in source or passed rider-facing route association

Current rider-facing popup contract is consistent with this:

- concise rider popup body
- route mile shown where available
- richer admin/debug detail may remain available separately

### 8.1.1 Route-level trace contract

The route-level trace must be able to explain:

- total route risk
- route risk per mile
- route confidence
- road-domain confidence
- provenance-family miles by contribution
- provenance-family risk contribution
- provenance-family confidence burden
- fallback burden summaries for speed, traffic, facility, and shoulder
- top contributing road and crossing records
- what concrete math or lookup produced the chosen value for the route-level top contributors
- exact resolved values when known rather than collapsed labels

Route-level trace must preserve precision. Examples:

- show exact AADT or AADT-per-lane when known, not only `Moderate`
- show lane counts when known
- show exact movement and width descriptors such as `Left turn across 6 lanes`
- show exact control types such as `Traffic light`

### 8.1.2 Road-level trace contract

The road-level trace must be able to explain:

- chosen speed, traffic, facility, shoulder, and curvature
- chosen provenance family and source type for each field
- factor math
- local likelihood
- local severity
- local risk
- local confidence contribution
- direct links or embedded summaries of the underlying input traces
- what concrete math or lookup produced the chosen value for each chosen input

Road-level trace must also preserve precision. Examples:

- `25,829 AADT total, 7 lanes, 3,690 AADT/lane`
- `55 mph`
- `Traffic light`
- `Left turn across 6 lanes`
- `Straight across 4 lanes`

Road-level trace is the canonical basis for inspector math and receipts. Those
two surfaces may differ in layout, but they may not differ in truth.

### 8.1.3 Input-level trace contract

The input-level trace must be able to explain:

- conditions that made the source eligible
- what concrete math or lookup produced the chosen value
- exact resolved values and selectors
- source links
- visual verification links when the field is visually inspectable

### 8.2 Source trace schema

`SourceTrace` is the common structured trace payload family for score-bearing
chosen inputs.

- `DerivationTrace` is the `SourceTrace` subtype for modeled or computed values
  such as `local_area_predicted`
- lookup-backed baseline sources may use a narrower lookup-style `SourceTrace`
  without pretending to be contributor-derived predictions

For `local_area_predicted`, the system must store a compact machine-readable
`DerivationTrace` payload rather than only prose.

Recommended canonical shape:

```text
SourceTrace {
  version
  field
  family
  sourceType
  method
  valueRaw
  valueFinal
  units
  radiusMi
  classContext
  areaContext
  candidateCount
  contributors[]
  aggregation
  rejectedStrongerEvidence[]
  fallbackReason[]
}
```

Contributor payload:

```text
Contributor {
  roadId
  roadName
  value
  units
  distanceMi
  sourceType
  family
  highwayType
}
```

Aggregation payload:

```text
Aggregation {
  op
  inputCount
  rawResult
  rounding { mode, increment, result }
}
```

### 8.3 Storage rules

Store:

- compact machine fields
- capped contributors
- symbolic rejection and fallback reasons
- route-level provenance and confidence rollups
- road-level chosen-input and local-confidence summaries

Do not store:

- full candidate geometry
- unbounded rejected candidate lists
- repeated giant prose strings

Generate prose at presentation time.

## 9. Presentation Contract

Rider-facing surfaces must not improvise provenance semantics.

All of the following must render from the same canonical truth and trace payload:

- inspector confidence tab
- field dropdowns
- road cards
- overlays
- receipts
- scorecard
- admin audit views

### 9.1 Summary vs expanded trace

Compact surfaces may show:

- value
- source label
- family label
- confidence band

Expanded surfaces must be able to show:

- why this source won
- what it was derived from
- what math was applied
- why stronger sources did not apply
- how the chosen input affected both risk and confidence

Visible inspectable fields that benefit from visual verification should expose a
Google Street View link with the proper bearing where feasible. This applies in
particular to:

- bike lane / facility
- shoulder
- crossing width
- crossing control
- admin-approved speed verification

Rider-facing non-score hazards should also expose concise route-local context
without forcing admin/debug detail into the default popup. Current live
expectation:

- hazard type
- route mile
- object/road name where available
- Street View / OSM inspect affordances where supported

Source-backed fields should also expose the underlying source link in the field
confidence dropdown itself when one exists, including OSM-backed fields.

### 9.2 Risk-confidence explanation parity

If a surface explains:

- why a road is risky

it must also be able to explain:

- why the system is confident or not confident in that risk explanation

This applies at:

- route level
- road level
- input level

## 10. Performance and Payload Guardrails

Full inspectability is required. Full artifact bloat is not.

Required guardrails:

- compute local-area predictions once per analysis-time pass, not per UI consumer
- cap stored contributors to 3-5
- store candidate counts separately from displayed contributor list
- store ids and values, not geometry
- dedupe identical trace payloads route-wide if repeated often
- render human-readable prose lazily from stored machine trace
- do not create separate risk-only and confidence-only trace universes

Phase-1 rule:

- `local_area_predicted` must be fully inspectable
- all other source types must at least meet the minimum trace depth in Section 8.1
- route-, road-, and input-level confidence traces must remain aligned to the
  same chosen truths used by risk

## 11. Reconciliation Rules for DS-015, DS-020, and DS-022

### 11.1 `DS-015`

`DS-015` remains the canonical family hierarchy.

### 11.2 `DS-020`

`DS-020` must be treated as a confidence projection of `DS-015`, not as an independent semantic truth model.

That means:

- `DS-020` should not define provenance meaning directly at the concrete source-type layer without first mapping through `DS-015`
- `highway_area_baseline` and `regional_prior` are concrete source types, not semantic families
- `local_area_predicted` must live in `predicted`, not baseline
- any earlier highway-confidence intuition about rough approximations hurting more on dangerous roads should be preserved through risk-weighted confidence burden, not through ad hoc provenance distortion

### 11.3 `DS-022`

`DS-022` governs speed-specific prior policy, but it must defer to this document for:

- family assignment
- precedence semantics
- confidence anchors
- traceability requirements

## 12. Implementation Scope

This section defines the intended execution scope for the next implementation pass.

### 12.1 Canonical truth and mapping layer

- [src/lib/evidence/types.ts](/Users/derekminner/lanterne/src/lib/evidence/types.ts)
- [src/lib/evidence/resolver.ts](/Users/derekminner/lanterne/src/lib/evidence/resolver.ts)
- [src/lib/evidence/local-area-predicted.ts](/Users/derekminner/lanterne/src/lib/evidence/local-area-predicted.ts)
- [src/shared/scoring/ds015-contract.ts](/Users/derekminner/lanterne/src/shared/scoring/ds015-contract.ts)

Required outcomes:

- traffic and speed align to the same family/source model
- local-area traffic maps to `local_area_predicted`
- speed gains the same local-area predicted layer
- precedence and confidence stay monotonic
- traffic and speed remain symmetrical in both risk and confidence treatment

### 12.2 Route-analysis and artifact layer

- [src/lib/route-analysis.ts](/Users/derekminner/lanterne/src/lib/route-analysis.ts)
- [src/lib/canonical-analysis-artifact.ts](/Users/derekminner/lanterne/src/lib/canonical-analysis-artifact.ts)
- `heatmap` and route-cache types that serialize chosen truth

Required outcomes:

- exported truth and inspector truth agree
- chosen input provenance and trace payload survive serialization
- receipts can render from stored truth rather than re-inventing derivations
- scorecard, road receipt, and confidence surfaces consume the same trace stack

### 12.3 Presentation layer

- [src/lib/presentation/provenance-labels.ts](/Users/derekminner/lanterne/src/lib/presentation/provenance-labels.ts)
- `semantic-tokens`, inspectable field presentation, speed presentation controller
- inspector, road card, overlay, receipt, and admin audit consumers

Required outcomes:

- predicted does not disappear into baseline wording
- compact surfaces stay compact
- expanded surfaces become auditable
- any risk explanation can be paired with its corresponding confidence explanation

### 12.4 Tests

Required test families:

- resolver precedence
- traffic reclassification
- speed local-area predicted ordering
- route-analysis vs inspector parity
- receipt / inspector / road-card label parity
- artifact serialization

## Appendix A. Field vocabularies

This appendix lists the canonical resolved option sets for the score-driving
fields. Thresholds, tag mappings, and derivation rules belong in Appendix C and
the field-specific DS references.

### A.1 Bike lane / facility

- `path / MUP`
- `protected`
- `buffered`
- `painted`
- `shared`
- `none`

`path / MUP` belongs to the facility truth vocabulary but still triggers the
separate path-domain scoring rule under `DS-015`.

### A.2 Shoulder

- `none`
- `narrow`
- `regular`
- `wide`

### A.3 Crossing control

- `none`
- `signed`
- `signaled`

### A.4 Crossing width

- exact crossed lane count when known
- exact crossed width basis when known
- width proxy only when direct width is unavailable

### A.5 Crossing movement

- `straight`
- `left_across`
- `right_merge`
- `join`
- `exit`
- `path_crossing`
- `unknown`

## Appendix B. Source-link and Street View policy

### B.1 Source links

Whenever a stable source URL or document link exists, the field-confidence
dropdown should expose it directly.

This includes:

- authoritative feeds and statutes
- regional-prior dataset or policy rows
- baseline source documents
- OSM-backed source types through an OSM object or lat/lon link

### B.2 Street View links

Where a field is visually verifiable from roadway imagery, the field-confidence
dropdown should expose a Street View link with the best available bearing.

This especially applies to:

- admin-approved speed verification
- bike lane / facility
- shoulder
- crossing control
- crossing width

## Appendix C. Classification and predicate mapping

This appendix is the bridge from archival policy to code predicates. The first
pass is intentionally limited to the highest-ambiguity categories.

### C.1 Path / MUP

Canonical intent:

- classify as `path / MUP` when the resolved facility is a separated riding
  space rather than an ordinary on-road lane treatment

Implementation mapping:

- explicit safe-path or MUP classification from the existing path-domain logic
- explicit mapped path/cycleway truth from OSM or equivalent source
- where geometry rules are used, preserve the exact predicate or distance test
  in code comments and trace payloads

### C.2 Bike lane / facility classes

Canonical intent:

- `protected`, `buffered`, `painted`, `shared`, `none`

Implementation mapping:

- preserve the exact OSM tags, side-specific tags, and continuity predicates
  used to reach the resolved class
- if multiple tags collapse into the same rider-facing class, keep the exact
  tags in input trace

### C.3 Shoulder classes

Canonical intent:

- `none`, `narrow`, `regular`, `wide`

Implementation mapping:

- preserve the exact width basis and threshold rule used to assign the bucket
- if width was mapped rather than measured, keep the exact tags or geometry
  basis in input trace

### C.4 Crossing control

Canonical intent:

- `none`, `signed`, `signaled`

Implementation mapping:

- derive from observed truth, authoritative truth, or mapped control geometry
- do not use a `relationship_inferred` rung for crossing control
- preserve the exact tags or control predicates used in trace

### C.5 Crossing width

Canonical intent:

- exact crossed lane count when known
- exact width basis when known
- width proxy only when direct width is unavailable

Implementation mapping:

- derive from observed truth, authoritative truth, or crossed-road geometry
- do not use a `relationship_inferred` rung for crossing width
- preserve the exact lane tags, width proxy rule, and concrete derivation math
  in trace

## Appendix D. Canonical source-type matrix by field

This appendix records whether a source type is active, inactive, or not allowed
for a given field in the current model.

| Field | Source type / truth form | Status | Notes |
| --- | --- | --- | --- |
| Speed | `observed`, `admin_approved`, `authoritative_posted`, `osm_posted`, `observation_inferred`, `authoritative_inferred`, `osm_inferred`, `local_area_predicted`, `regional_prior`, `highway_area_baseline`, `highway_baseline`, `unknown` | active | full ladder |
| Traffic | `observed`, `admin_approved`, official AADT variants, corridor-carried AADT, `local_area_predicted`, `highway_area_baseline`, `unknown` | active | canonical traffic ladder |
| Traffic | `regional_prior`, `highway_baseline` | inactive | retained globally but not active traffic rungs |
| Bike lane / facility | `observed`, `admin_approved`, authoritative imported facility, geometry-derived mapped facility, continuity-carried facility, `unknown` | active | no predicted/baseline |
| Shoulder | `observed`, `admin_approved`, authoritative imported shoulder, geometry-derived shoulder, continuity-carried shoulder, `unknown` | active | no predicted/baseline |
| Crossing control | `observed`, `admin_approved`, authoritative imported control, geometry-derived control, `unknown` | active | no relationship_inferred |
| Crossing width | `observed`, `admin_approved`, authoritative imported width, geometry-derived width, `unknown` | active | no relationship_inferred |
| Crossing movement | geometry-derived movement, `unknown` | active | geometry/topology derived only |

## Appendix E. Derivation trace schema and enums

### E.1 Canonical `DerivationTrace` enums

Allowed values in phase 1:

- `field`: `speed_mph`, `aadt_total`, `aadt_per_lane`, `traffic_factor`
- `method`: `local_area_predicted`, `regional_prior_lookup`, `highway_area_baseline_lookup`, `highway_baseline_lookup`
- `units`: `mph`, `vehicles_per_day`, `vehicles_per_day_per_lane`, `factor`
- `aggregation.op`: `mean`
- `rejectedStrongerEvidence.reason`: `missing`, `ineligible`, `too_far`, `class_mismatch`, `urbanicity_mismatch`, `outside_corridor`, `not_direct_truth`
- `fallbackReason`: `no_direct_truth`, `no_relationship_truth`, `no_local_candidates`, `fell_to_baseline`

### E.2 Required fields by source type

| Source type | Required derivation fields |
| --- | --- |
| `local_area_predicted` | full `DerivationTrace`, contributors, aggregation, radius, classContext, areaContext |
| `regional_prior` | method, lookup key, jurisdiction, valueFinal, source document link |
| `highway_area_baseline` | method, lookup key, table/version, valueFinal, source document link |
| `highway_baseline` | method, lookup key, table/version, valueFinal, source document link |

## Appendix F. Confidence rollup notes

The confidence rollup formula in Section 5.1.3 is canonical, but implementations
should also follow these notes:

- cap confidence score to `[0, 100]`
- preserve exact numeric confidence in trace and artifact
- bands are derived after numeric confidence is computed
- route summaries may highlight only top contributors by confidence burden
- the same materiality threshold used for risk trace surfacing may also cap
  route-level confidence detail surfaces

## Appendix G. Local-area lookup policy

### G.1 Broad road class

For phase 1, `BroadRoadClass` is defined as:

- strip `_link` suffix when present
- treat the remaining highway type as the broad class key
- exclude rail, path, sidewalk, crossing-connector-only, and safe-path-only
  objects from local-area speed and traffic contributors unless a future DS
  explicitly promotes them

This intentionally avoids broader class-family grouping in phase 1.

### G.2 Corridor-bounded search

For phase 1, corridor-bounded search means:

- candidate contributors must already belong to the fetched route corridor
  evidence universe
- candidate distance is then evaluated from the target segment within that
  corridor-bounded universe

This is narrower than an unconstrained radial search and broader than strict
same-road continuity.

### G.3 Urbanicity context

For phase 1:

- use per-segment urbanicity
- allowed values are `urban`, `suburban`, `rural`
- if urbanicity is unknown on the target segment, do not promote to a broader
  cross-context match
- urban/rural mismatches are not eligible contributors

## Appendix H. Baseline lookup policy

### H.1 State-specific vs generic area baseline order

For `highway_area_baseline` speed:

1. if state, urbanicity, and highway type are known and a state-specific row
   exists, use that row
2. otherwise use the generic area-context table keyed by highway type and
   urbanicity
3. if no area-context value exists, fall to `highway_baseline`

### H.2 Traffic class proxy order

For traffic baseline:

1. use the DS-015 highway-type class proxy AADT/lane table
2. derive factor from canonical DS-015 traffic-factor math
3. if no class proxy exists, fall to `unknown`

All baseline lookups must carry table identity, version, and source document
link.

## Appendix I. Source-link policy

| Source type | Link policy |
| --- | --- |
| `observed` | optional when a stable external reference exists |
| `admin_approved` | required supporting verification link when one exists; audit-only refs may remain internal |
| `authoritative_posted` / `authoritative_inferred` | required when a stable public or internal authoritative link exists |
| `osm_posted` / `osm_inferred` | generated OSM object link or lat/lon link |
| `local_area_predicted` | contributor links when available |
| `regional_prior` / `highway_area_baseline` / `highway_baseline` | source document or governing table link required |

## Appendix J. Street View policy

| Field | Street View policy | Notes |
| --- | --- | --- |
| admin-approved speed | required when sign-visible verification exists | preserve verification bearing where feasible |
| bike lane / facility | optional but preferred | degrade gracefully if no pano exists |
| shoulder | optional but preferred | degrade gracefully if not visible |
| crossing control | optional but preferred | bearing should face the relevant approach |
| crossing width | optional but preferred | bearing should face the crossing approach |

Street View is an audit/verification capability. Exact UI placement may vary by
surface.

## Appendix K. Artifact serialization and rollout phases

### K.1 Phase table

| Item | Phase 1 | Phase 2+ |
| --- | --- | --- |
| `local_area_predicted` trace | fully required | remains required |
| other source-type minimum trace identity | required | remains required |
| full route/road/input trace architecture | required schema | remains required |
| full derivation trace for non-predicted sources | schema reserved | complete as source-specific phases land |
| broad audit UX parity across all surfaces | partial rollout acceptable | required completion path |

Phase 1 is sequencing, not permission to leave the rest asymmetrical.

### K.2 Serialization shape

Canonical artifacts should support either:

- inline per-field trace payloads, or
- route-level trace table plus per-field `traceId` references

Required properties:

- trace payload versioning
- compatibility with legacy artifacts that lack full trace
- capped contributor storage
- no full geometry duplication inside trace payloads

## 13. Canonical Rule of Interpretation

If any older doc, implementation comment, or UI string conflicts with this document on provenance family, precedence, confidence anchor, or traceability requirement:

1. this document governs
2. `DS-015` governs family semantics where this document references it
3. implementation must be updated to match before adding new exceptions

This document is intended to eliminate half-measures in provenance handling. Any future source type or fallback layer must be added here first, with:

- family assignment
- precedence
- confidence anchor
- explicit math
- traceability requirements


---

## Source File: docs/02-architecture/design/ds-030-route_analysis_contract.md

# DS-030 - Route Analysis Contract

**Status:** Draft for implementation  
**Date:** 2026-04-27  
**Filename:** `ds-030-route_analysis_contract.md`  
**Related:** [DS-017](./ds-017-truth_resolution_and_propagation_spec.md), [DS-024](./ds-024-parallel_bike_facility_capture_and_corridor_ownership_spec.md), [DS-025](./ds-025-transition_candidate_claim_and_projection_spec.md), [DS-028](./ds-028-hazard_ingestion_normalization_and_presentation_spec.md), [DS-029](./ds-029-provenance_precedence_confidence_and_traceability_spec.md)



## 0. Governing Principle

Lanterne must prefer “unknown” over “incorrectly certain.” No pipeline stage may invent score-bearing identity or downgrade evidence to produce a complete-looking result. “Invent” means assigning identity, speed, traffic, or confidence without the minimum candidate, geometry, topology, or provenance support required by the invariant that owns that field.

## 1. Purpose

This document defines the canonical route analysis invariants discovered across
the DC, TX-LA, Cape May, and T&H diagnostic audits.

It is a contract for route-analysis correctness. It does not introduce product
design, fallback heuristics, or implementation patches.

The contract separates ownership across these branches:

- Ghost truth cleanup
- Evidence propagation
- DS-029 provenance
- Presentation parity
- Hazard

## 2. Invariants

| Invariant | Owning pipeline stage | Branch bucket |
| --- | --- | --- |
| Detached path/cycleway identity must remain path identity unless adjacent motor-road inheritance is locally proven by close, parallel geometry across multiple samples. Parent-road inheritance requires:<br/>\- bounded offset distance using an explicit, tested threshold<br/>\- bounded angular difference using an explicit, tested threshold<br/>\- consecutive sample support, never a single-sample claim<br/>\- topology-valid adjacency, never crossing, underpass, or grade-separated proximity<br/>The threshold values may live in implementation config, but they must be named, testable, and traceable in diagnostics. | Matcher / parallel cycleway reclassification | Ghost truth cleanup |
| Passing under, crossing, diverging by angle, or separating beyond threshold must break parent-road inheritance at the first unsupported sample or the first projected boundary generated from that unsupported sample. | Matcher / boundary refinement | Ghost truth cleanup |
| Traffic signals, stop nodes, crossings, and other control nodes may affect hazard/crossing evidence only; they must never become `roadName`, `highwayType`, speed source, traffic source, or clickable segment identity. | Candidate classification / truth materialization | Ghost truth cleanup |
| Broad nearby-road retrieval must not override local path/cycleway candidates. “Local” means the candidate set inside the primary matcher radius before fallback expansion. Long-distance fallback identity requires explicit local geometry support: bounded distance, bounded angular agreement, and consecutive sample support at the chosen identity. | Nearby candidate retrieval / matcher | Ghost truth cleanup |
| Truth boundaries must land at the physical transition. Boundary placement precedence is: route departure point when the route leaves one geometry for another; otherwise intersection apex when a true intersection exists; otherwise closest approach when the transition is supported by sampled route geometry. Boundaries must not lag through a turn or fire early before the rider leaves the current road. | Boundary refinement / transition projection | Ghost truth cleanup |
| Same-path -> road -> same-path crossings are invalid unless the route geometry is aligned to the motor road and motor-road candidates dominate path candidates for consecutive samples between two distinct path segments. A single crossing, underpass, bridge, or nearest-road sample does not prove that the rider rode the motor road. | Truth materialization | Ghost truth cleanup |
| Named road continuity must survive short OSM way splits unless there is positive evidence of a real identity change. “Short” must be defined by an explicit distance or sample-count threshold. Positive evidence requires more than a way ID change: it requires supported name/ref/highway change, route-geometry alignment, or topology evidence. Matcher owns identity assignment. Materializer may collapse runs but must not modify or replace identity fields (roadId, roadName, highwayType, scoreDomainType). | Matcher / truth materialization | Ghost truth cleanup |
| Named -> unnamed -> same named road requires proof of a true unnamed segment; otherwise preserve continuity or mark uncertain, not low-confidence named truth. Proof requires stable unnamed way identity, consecutive sample support, absence of a stronger same-name continuation candidate, and no topology evidence that the unnamed run is only a connector or artifact. If named continuity is supported, preserve named continuity; if unnamed truth is supported, keep unnamed truth; if neither is supported, mark unresolved. | Truth materialization / sanitizer | Ghost truth cleanup |
| Unresolved/off-network GPX mileage must remain unknown or uncertain. Unresolved means no score-bearing road identity, no baseline speed, no baseline traffic, and exclusion from matched-mile totals. It must not become unnamed 15mph/25mph area-estimate road truth. | Matcher null-fill / truth materialization | Ghost truth cleanup |
| Match quality must measure pre-fill candidate support, not post-fill availability of any road-like assignment. The measured state is the matcher candidate/support state before null-fill, sanitizer, materializer collapse, or evidence fallback. | Matcher diagnostics / route quality rollup | Ghost truth cleanup |
| Official speed or traffic evidence must not be lost merely because a truth segment boundary or OSM way split occurs. | Evidence resolution / propagation | Evidence propagation branch |
| Relationship-inferred official evidence outranks baseline fallback when geometry/name continuity is intact. Evidence resolution must consider both local truth-run evidence and adjacent corridor evidence. “Continuity” must be represented by explicit distance, sample-count, name/ref, and geometry-alignment criteria. Short gaps in local evidence must not trigger fallback while those criteria remain satisfied. Baseline is eligible only after relationship continuity fails or stronger propagated evidence is explicitly rejected. | Evidence resolver | Evidence propagation branch |
| Speed and traffic must use symmetric DS-029 provenance, precedence, confidence, receipts, and inspect traces. | Evidence resolver / confidence graph | DS-029 provenance branch |
| Resolver recomputation per truth segment is allowed only if it preserves upstream evidence lineage or records a specific rejection reason. Minimum lineage is: source type, selected value, source identifier when available, segment span, propagation reason when accepted, and rejection reason when rejected. | Evidence resolver | DS-029 provenance branch |
| Baseline, regional prior, and local-area estimates may fill only after stronger official/posted/OSM/relationship evidence has been rejected with traceable cause. Rejection causes must be explicit categories such as missing field, spatial mismatch, identity mismatch, stale/conflicting source, failed continuity, or below-threshold support. | Evidence resolver | DS-029 provenance branch |
| Provenance resets at segment boundaries are invalid; the trace must show whether evidence was direct, propagated, rejected, or unavailable. | Provenance trace export | DS-029 provenance branch |
| Risk scoring must not use provenance/confidence as a risk multiplier. Provenance affects confidence and explanation, not hazard/risk magnitude. The dependency is one-way: chosen score-driving inputs feed confidence burden; confidence and provenance do not feed risk magnitude. | Safety scoring | DS-029 provenance branch |
| Confidence must be computed from the same chosen score-driving inputs as risk. No surface may explain risk with richer source detail than confidence can trace. | Confidence graph / artifact export | DS-029 provenance branch |
| Route-level truth, inspect truth, receipts, clickable segments, and debug panels must agree on chosen speed, traffic, source type, confidence, and rejection path after accounting for aggregation or summarization. They may differ in display text, but not in canonical chosen inputs or provenance lineage. | Export / inspect surfaces | DS-029 provenance branch |
| Paint colors must project canonical truth; paint modes must not create, merge, or reinterpret truth identity. Debug modes may display non-canonical intermediate states only when the state is labeled as non-canonical and cannot be mistaken for canonical paint or truth. | Paint projection | Presentation parity branch |
| Traffic badge severity and total paint severity must be explicitly distinguishable when traffic is low but total risk is medium/high due to speed, shoulder, or infrastructure. The inspect payload must separately represent per-factor traffic severity and total-risk severity. | Presentation / semantic tokens | Presentation parity branch |
| Risk bucket thresholds must be consistent between truth export, route paint, inspect panels, and receipts. Debug-only alternate thresholds are allowed only when labeled non-canonical and not used for rider-facing or score-bearing truth. | Presentation / scoring export | Presentation parity branch |
| Hazard detection must be independent of road-name truth success when raw corridor geometry/tag evidence exists. Detection may not require a successful `roadName`, but attachment still requires route-geometry proximity, intersection, or span support. | Hazard detection / attachment | Hazard branch |
| Metal grate bridges must fire when corridor data contains bridge plus grate/grid/metal surface tags and that qualifying evidence intersects or spans the route geometry. | Hazard detection | Hazard branch |
| Railroad hazards must fire when corridor data contains explicit crossing nodes or rail-road geometric intersections and that qualifying evidence intersects the route geometry. | Hazard detection | Hazard branch |
| Left-turn debug must be derived from route movement geometry at transitions, not from final road-name materialization alone. Turn classification must use an explicit, tested bearing-delta threshold owned by transition debug config. | Transition debug / hazard-debug presentation | Hazard branch |
| Debug/inspect surfaces must distinguish real truth bugs from projection artifacts: geometry truth, source truth, confidence trace, canonical paint, and labeled non-canonical debug states are separate layers. | Debug tooling / admin surfaces | Presentation parity branch |

## 4. Must Nots

The system must not:
\- assign road identity to control nodes (traffic_signal, stop_sign, etc.)
\- treat crossings as adjacency
\- convert unresolved segments into baseline roads
\- drop authoritative evidence due to segment boundaries or OSM way splits
\- recompute evidence per segment without lineage
\- use provenance/confidence to alter risk magnitude

## 5. Implementation Order

1. Ghost truth cleanup should go first. It owns the substrate invariants: path collapse, boundary lag, unnamed/off-network truth, candidate overreach, and false road continuity.
2. Evidence propagation and DS-029 should follow once truth identity is stable. Otherwise official speed/traffic propagation will continue to be debugged against moving or false truth segments.
3. Presentation parity and hazard work should stay separate. They expose real problems, but they should not be used to compensate for matcher/materializer drift.


---

## Source File: docs/03-adrs/adr-000-README.md

# Architecture Decision Records

ADRs capture the historical decisions that shaped Lanterne's architecture.

Architecture documents describe the system as it exists today.

When conflicts appear, architecture documents reflect the current system. ADRs provide historical context.

---

## Source File: docs/03-adrs/adr-001-route_acquisition_model.md

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

## Source File: docs/03-adrs/adr-002-vault_concept.md

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

## Source File: docs/03-adrs/adr-003-mode_aware_vault_filtering.md

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

## Source File: docs/03-adrs/adr-004-rider_field_notes_deferred.md

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

## Source File: docs/03-adrs/adr-005-route_analysis_model.md

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

## Source File: docs/03-adrs/adr-006-safety_definition.md

# ADR-006 — Safety Definition

**Status:** Accepted (updated for v3.1-launch)  
**Date:** 2026  
**Updated:** 2026-04-04

**Related ADRs:** ADR-007 (Index Families), ADR-032 (Comparative Traffic Context), ADR-039 (Bounded Crossing Risk)  

---

## Context

Safety is a word that means different things to different people. Without a precise definition, the Safety Score risks absorbing unrelated factors — weather discomfort, rough surfaces, fatigue — that dilute its meaning and undermine trust.

Lanterne must define safety narrowly and defend that definition as the product evolves.

---

## Decision

**Safety is defined narrowly as:**

> The likelihood of a rider being struck by a motor vehicle, and the severity of the resulting injury.

**The Safety Score (v3.1-launch) is derived from:**

- **Continuous road risk**: 60% Speed + 40% Traffic (AADT), modified by InfraFactor and ShoulderFactor
- **Crossing risk contribution**: bounded per-event model (E0=0.05, E_cap=0.75), capped at 40% of route risk

**Explicitly excluded from the canonical Safety Score:**

- Wind, temperature, precipitation
- Light state, UV
- Surface quality
- Rail crossings, cattle grids, metal grates (hazard overlay only)
- Time-of-day traffic adjustments (contextual layer only)
- Critical stretch / hotspot penalties (report-only)
- Fatigue, navigation difficulty

These belong in other index families (see ADR-007) and may be presented alongside safety, but they must never contaminate the Safety Score itself.

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

## Source File: docs/03-adrs/adr-007-index_families.md

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

## Source File: docs/03-adrs/adr-008-environmental_light_system.md

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

## Source File: docs/03-adrs/adr-009-sun_and_moonlight.md

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

## Source File: docs/03-adrs/adr-010-sun_glare_detection.md

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

## Source File: docs/03-adrs/adr-011-route_slice_model.md

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

## Source File: docs/03-adrs/adr-012-predicted_vs_experienced_conditions.md

# ADR-012 — Predicted vs Experienced Conditions

**Status:** Accepted  
**Date:** 2026-03-17

**Related ADRs:** ADR-023 (Predicted vs Observed Condition Layers), ADR-016 (Ride Session Data Model), ADR-011 (Route Slice Model)  
**No companion DS required** — see ADR-023 for the expanded decision and DS-015 for the canonical safety model spec.

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

## Source File: docs/03-adrs/adr-013-personalized_emergency_alerts.md

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

## Source File: docs/03-adrs/adr-014-ride_narrative_event_model.md

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

## Source File: docs/03-adrs/adr-015-route_vulnerability_feature_model.md

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

## Source File: docs/03-adrs/adr-016-ride_session_data_model.md

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

## Source File: docs/03-adrs/adr-017-local_osm_derived_data_strategy.md

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

## Source File: docs/03-adrs/adr-018-server_cached_slice_analysis_model.md

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

## Source File: docs/03-adrs/adr-019-route_corridor_and_proximity_rules.md

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

## Source File: docs/03-adrs/adr-020-atomic_analysis_unit.md

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

## Source File: docs/03-adrs/adr-021-osm_variable_registry.md

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

## Source File: docs/03-adrs/adr-022-phase_1_enum_registry.md

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

## Source File: docs/03-adrs/adr-023-observed_conditions_layers.md

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

## Source File: docs/03-adrs/adr-024-ride_timeline_plans.md

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

## Source File: docs/03-adrs/adr-025-fatigue_index_as_extensible.md

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

## Source File: docs/03-adrs/adr-026-canonical_route_identity.md

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

## Source File: docs/03-adrs/adr-027-lantern_screen_model.md

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

## Source File: docs/03-adrs/adr-028-field_note_confirmation_model.md

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

## Source File: docs/03-adrs/adr-029-ride_time_situational_awareness_mode.md

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

## Source File: docs/03-adrs/adr-030-ride_mode_power_and_sensor_architecture.md

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

## Source File: docs/03-adrs/adr-031-model_multi_day_events_as_ordered_references_onto_canonical_geometry.md

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

## Source File: docs/03-adrs/adr-032-comparative_traffic_context_and_segment_cohorts.md

ADR-032 — Comparative Traffic Context and Segment Cohorts



See ADR-042 for how canonical truth is derived from these evidence layers.

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

## Source File: docs/03-adrs/adr-033-canonical_segment_identity.md

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

## Source File: docs/03-adrs/adr-034-master_route_expeditions_and_windowed_analysis.md

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


---

## Source File: docs/03-adrs/adr-035-turn_event_canonical_vs_history_linking.md

# **Turn Event Persistence — Canonical vs Route History Linking**

**Status:** Accepted (Phase 0 / Builder Mode)
 **Date:** 2026-03-28
 **Owner:** Lanterne Core

------

## **1. Context**

Turn event persistence has been successfully implemented with the following flow:

- Route analysis runs client-side
- Canonical route is resolved via:
  - imported route match
  - canonical route match (geometry hash)
  - name fallback
- If no match exists → canonical route row is auto-created
- Turn events are written to `route_turn_events` using `canonical_route_id`

Example confirmed behavior:

- Canonical route auto-created
- Prior events deleted
- 14 turn events written successfully

------

## **2. Problem**

Each turn event row includes:

- `canonical_route_id` ✅ (required, working)
- `route_history_id` ⚠️ (currently `null`)

Question:

> Should turn events be tied to a specific user route (`route_history_id`), or only to the canonical route?

------

## **3. Decision**

### **Primary Relationship**

Turn events are **anchored to canonical routes**, not user route history.

```
route_turn_events
  → canonical_route_id (required)
  → route_history_id (optional / nullable)
```

### **Rule**

- `canonical_route_id` = **required**
- `route_history_id` = **nullable, optional provenance only**

------

## **4. Rationale**

### 4.1 Canonical route is the true identity

A route’s geometry defines its identity.

Turn events are:

- derived from geometry
- stable across users
- reusable across analyses

Therefore:

> Turn events belong to the canonical route, not to a specific user.

This aligns with the broader system principle:

- route identity (geometry)
- analysis outputs
- user ownership

are **separate concerns** 

------

### 4.2 Avoid premature coupling

Making `route_history_id` required would:

- force persistence to wait for `saveRoute()`
- break anonymous / pre-save analysis
- tightly couple analysis to user flows
- increase failure points

This violates the current architecture:

> Analysis should run independently of user persistence 

------

### 4.3 Turn events are analysis artifacts

Turn events are:

- deterministic outputs of route analysis
- comparable across runs
- tied to analysis version, not user

They behave more like:

- segment data
- scoring outputs

than user-owned records

------

## **5. Current Implementation (Accepted)**

### Behavior

- Resolve or create `canonical_routes` row
- Delete prior turn events for that canonical route
- Insert new turn events
- Write:

```
{
  "canonical_route_id": "...",
  "route_history_id": null,
  ...
}
```

### Guarantees

- Canonical route always exists before insert
- Turn events always attach to canonical route
- No dependency on user save flow

------

## **6. Future Enhancement (Not Required Now)**

### Goal

Allow optional linkage to the specific route instance that produced the analysis.

### Required Change

Update `saveRoute()`:

```
return { id: routeHistoryId, ... }
```

Then pass through:

```
persistTurnEvents(canonicalRouteId, turnEvents, routeHistoryId)
```

### Result

```
canonical_route_id → always present
route_history_id   → present when available, null otherwise
```

------

## **7. When to Implement Route History Linking**

Only implement when one of the following becomes necessary:

- UI: “open this saved route and show its exact turn events”
- Debugging: trace which analysis run produced which events
- Versioning: compare multiple analyses of same route
- User-specific derived routes need distinct event sets

Until then:

> Do not prioritize this work.

------

## **8. Anti-Patterns (Do Not Do)**

❌ Require `route_history_id` for insert
 ❌ Block turn persistence on `saveRoute()`
 ❌ Treat turn events as user-owned data
 ❌ Duplicate turn events per user instead of per route

------

## **9. Architectural Position**

This decision reinforces:

- Canonical-first architecture
- Shared analysis artifacts
- Separation of identity vs ownership vs computation

It is consistent with:

- route identity separation
- analysis reuse across users
- future versioned analysis model 

------

## **10. Current Status**

✅ Turn persistence working
 ✅ Canonical route auto-create working
 ✅ Canonical linkage correct
 ⚠️ Route history linkage intentionally deferred

------

## **11. Next Step**

Do **not** touch this further right now.

Move to:

- canonical route dedup validation
- rerun idempotency check
- slice-level scoring / OSM ingestion work

------

## **One-line summary**

> Turn events belong to the route, not the user.
>  User linkage is optional metadata, not a requirement.

---

## Source File: docs/03-adrs/adr-036-push_based_ride_intelligence.md

# ADR-036 — Push-Based Ride Intelligence

Status: Proposed
Date: 2026-03-30

Companion ADRs:
ADR-029 (Ride-Time Situational Awareness Mode)
ADR-030 (Ride Mode Display, Power, and Sensor Architecture)

Related specs:
DS-001 (Route Intelligence Pipeline)
DS-012 (Ride Computer Tile System)

------

## Context

Lanterne currently provides **route intelligence**.

It answers:

- What is this route?
- How dangerous is it?
- How remote is it?
- What conditions will shape it?

However, once a rider begins a ride, a different class of questions emerges:

- Am I still on track?
- What happens if I keep riding like this?
- How much time do I have left before I miss the cutoff?
- What do I need to do right now to stay within my goal?

Existing cycling computers provide instrumentation:

- speed
- distance
- power
- heart rate

They do not provide **forward-looking execution intelligence**.

This creates a gap between:

- understanding a route
- executing a ride

------

## Decision

Lanterne will introduce a **Push-Based Ride Intelligence system**.

This system models a ride as a sequence of **pushes**, where:

A push is:

> a bounded execution block of riding, typically up to ~1200 km, defined by a start point, end point, and timing constraints.

The push becomes the primary unit for:

- pacing
- stop planning
- execution tracking
- forward projection
- decision support

------

## Core Model

Each push consists of four distinct layers:

### 1. Official Constraints

These represent reality that does not change:

- official start time (when applicable)
- official close times or time limits
- route distance
- route terrain
- rule-based adjustments (brevets, ultra events, etc.)

These constraints are **immutable**.

They do not adjust based on rider behavior.

------

### 2. Rider Plan

This represents intent:

- planned moving behavior (speed and/or effort)
- planned stops (location + duration)
- optional rider overrides
- optional terrain-aware assumptions

The plan is:

- editable before and during the ride
- versioned
- never silently mutated by the system

------

### 3. Actual Ride State

This represents what has actually happened:

- current route position
- moving time
- stopped time
- recent performance (default: last 60 minutes)
- confirmed stops
- skipped stops

This layer is:

- event-driven
- durable across sessions
- derived from sparse updates (not continuous logging)

------

### 4. Guidance Layer

This represents forward-looking intelligence:

- projected finish time
- projected cue/control arrival times
- gap to plan
- gap to cutoff
- required moving speed
- required average watts (when power data exists)
- remaining stop budget

This layer is:

- continuously recomputed
- explicitly derived from the other three layers
- never treated as a single opaque value

------

## Separation Principle

These four layers must remain separate.

The system must never collapse them into:

> a single “smart ETA”

Every output must be explainable as:

- official reality
- planned intent
- actual behavior
- forward requirement

------

## Rider Model

The system introduces a **rider model**.

The rider model estimates:

- how terrain impacts moving performance
- optional effort (watts) required for a given outcome

Rev 1 characteristics:

- uses rider-provided sample rides (distance, time, elevation, optional watts)
- derives a continuous terrain-performance relationship
- supports both:
  - speed-based riders
  - power-based riders

Important constraints:

- no fake precision
- no inferred watts without actual data
- all outputs must include confidence

------

## Stop Model

Stops are first-class objects.

Each stop includes:

- location (mile, cue, control, POI)
- planned duration
- actual duration (when completed)

System behavior:

- detects or confirms stop start and end
- updates projections after each stop
- prompts when planned stops are skipped
- allows stop relocation or removal

Stops influence:

- elapsed time
- future projections
- stop budget calculations

------

## Projection Engine

The projection engine simulates forward from current state.

It:

- uses terrain-aware rider model
- blends with recent observed performance
- incorporates planned or actual stops
- computes arrival times and finish projections
- evaluates against official constraints

The engine operates on:

- route slices
- cumulative distance
- event-driven updates

It does not require:

- continuous GPS tracking
- second-by-second recomputation

------

## Live Execution Model

Ride mode uses:

- intermittent GPS updates (~15–30 seconds)
- event-driven recalculation

Recalculation triggers:

- ride start
- GPS update
- stop start/end
- planned stop passed
- user adjustments
- push completion

Default live basis:

> last 60 minutes of moving performance

------

## Ride Mode Integration

Push intelligence integrates into the Lantern Stack.

It does not replace the ride computer.

Instead, it provides **data objects** that can be rendered as tiles.

Examples of push-derived signals:

- finish cushion
- required pace
- required watts
- stop budget
- next control cushion

These are treated as:

> first-class ride-time signals, alongside environmental and navigation signals

This aligns with:

- ADR-029 (situational awareness)
- ADR-030 (tile-based ride computer architecture)

------

## Expedition Model

An expedition is a sequence of pushes.

Pushes are:

- independently executable
- linked through shared state

At the end of a push, the system enters a reconciliation moment.

The rider is presented with:

- actual vs planned outcome
- impact on overall goal

Options include:

- maintain original goal (tighten future pushes)
- shift goal
- defer decision

The system must:

- never auto-adjust silently
- allow optional future push detail levels
- preserve rider control

------

## Design Principle

Push Intelligence answers:

> “What happens if I keep going like this, and what should I do next?”

Not:

> “What metrics can we display?”

------

## Consequences

Advantages:

- introduces execution intelligence into cycling
- differentiates from traditional bike computers
- builds naturally on Lanterne’s existing architecture
- supports both brevets and expedition riding
- enables prescriptive guidance

Tradeoffs:

- increases system complexity
- requires careful UX discipline
- requires strong separation of concerns to maintain trust

------

## Non-Goals

This ADR does not include:

- full physiological modeling
- training recommendations
- calorie or fueling systems
- full Strava dependency
- perfect prediction accuracy

------

## Summary

Push-Based Ride Intelligence is the time-domain extension of Lanterne.

Route intelligence answers:

> what the road is

Push intelligence answers:

> what the ride becomes


---

## Source File: docs/03-adrs/adr-036b-execution_moments_and_interaction_model.md

## Ride State and Performance Model

Push Intelligence models the rider as a **changing system over time**, not a fixed average.

Performance is not constant across a push.
It evolves based on:
- duration
- terrain
- stops
- fueling
- recovery
- rider-specific behavior

The system represents this using two layers:

1. **Fatigue curve** (continuous)
2. **Ride state model** (discrete)

These are internal models that drive projections and guidance.
They are not directly exposed as raw values.

---

### Fatigue Curve

The fatigue curve represents how performance changes across the push.

It is:
- anchored to a baseline (plan, rider model, or default)
- shaped across the duration of the push
- adjusted by recovery events (stops, rest)

Key characteristics:

- Early push: near-baseline performance
- Mid push: gradual decline
- Late push: increased variability and degradation risk

The curve is:
- simple
- explainable
- designed to be replaced by observed data as quickly as possible

It is never treated as precise physiology.

---

### Ride State Model

The ride state model represents the rider’s **current operational condition**.

States:

- Fresh
- Settling In
- Settled
- Fatigued
- Degraded

These states are:

- dynamic (can move left or right)
- based on observed behavior vs expected behavior
- influenced by stops, recovery, and recent performance

They are not strictly distance-based.

---

### State Transitions

The system updates ride state based on:

- sustained deviation from expected pace
- stop behavior (frequency, duration, recovery quality)
- recent performance relative to model
- recovery signals (improved pace after rest or fueling)

Transitions must:

- require sustained evidence (no rapid oscillation)
- support recovery (movement back toward earlier states)
- avoid false precision

---

### Recovery

Recovery is explicitly modeled.

After meaningful rest or fueling, the system allows:

- partial return toward earlier states
- improvement in projected performance
- narrowing of negative drift

Recovery is:

- inferred from observed behavior
- optionally reinforced by rider input (future phase)

---

### Instability Detection

The system detects when a rider is no longer behaving like a stable continuation of the push.

This includes:

- sustained underperformance relative to expected pace
- worsening projections across multiple updates
- ineffective recovery after stops

When instability is detected:

- projections widen
- guidance becomes more conditional
- confidence is reduced

---

### Stop Behavior Model

Stops are part of performance, not noise.

The system models stops as:

- planned (explicit)
- inferred (observed)
- prior-based (default when unknown)

Stop behavior influences:

- elapsed time
- recovery
- projection accuracy

When no plan exists, the system uses distance-based priors and replaces them with observed behavior as quickly as possible.

---

### Integration with Projection Engine

At any point in the push:

Projected performance is a function of:

- baseline (plan, rider model, or default)
- fatigue curve position
- ride state
- recent observed performance
- stop behavior

The projection engine uses these to compute:

- arrival times
- finish time
- required pace
- required effort

---

### Design Principle

The system models:

> how the rider is actually evolving over the ride

Not:

> what the rider “should” be doing in theory



## Execution Moments and Interaction Model (Draft)

Push Intelligence is not continuously conversational.

It does not behave like an assistant that is always speaking.
It behaves like a system that understands when something meaningful has happened.

Interaction is reserved for **execution moments**.

### Definition

An execution moment is a point in the ride where:

- the rider’s trajectory meaningfully changes, or
- the rider’s plan diverges from reality, or
- a decision has material downstream impact

These are the only times Lanterne should:
- interrupt
- prompt
- recommend
- acknowledge

Outside of these moments, the system should remain quiet and observational.

### Categories of Execution Moments

Execution moments fall into three broad categories:

#### 1. Orientation

Moments where the rider is establishing or recalibrating context.

Examples:
- push start
- early ride (first meaningful performance signal)

Goal:
- ground the rider in reality
- establish expectations

#### 2. Drift and Correction

Moments where the rider’s behavior diverges from plan or expectation.

Examples:
- first meaningful stop
- skipped planned stop
- sustained pace drift
- transition into fatigued or degraded state
- recovery after rest or fueling

Goal:
- make divergence visible
- preserve optionality
- enable correction without forcing it

#### 3. Consequence

Moments where outcomes are now materially affected.

Examples:
- approaching cutoff thresholds
- loss of meaningful buffer
- end-of-push reconciliation
- expedition carry-forward impact

Goal:
- clearly show consequences
- present decisions explicitly
- never silently adjust outcomes

### Interaction Rules

1. **No constant chatter**

   The system must not provide continuous commentary.
   It only speaks when something meaningful changes.

2. **No silent mutation**

   The system must never adjust plans, stops, or projections without:
   - surfacing the change
   - allowing the rider to confirm or override

3. **Facts first, guidance second**

   All interactions must:
   - present reality (what changed)
   - then optionally present guidance (what to do)

4. **Consent-based guidance**

   Prescriptive behavior (recommendations or encouragement) must be optional.

   Riders may enable or disable:
   - in-ride recommendations
   - encouragement

   The system must function fully with both disabled.

5. **Explain why, not just what**

   Any meaningful change in projection should include a brief explanation of cause.

   Examples:
   - pace drift
   - stop duration
   - skipped stop
   - recovery

### Design Principle

Push Intelligence answers:

> “What just changed, and what does it mean for the rest of this push?”

Not:

> “What can we say right now?”

---

## Source File: docs/03-adrs/adr-037-canonical_route_identity_vs_ride_envelope_containment.md

# ADR: Canonical Route Identity vs Ride Envelope Containment

## Status
Proposed

## Date
2026-04-01

## Context

Lanterne currently tries to use canonical route matching as the main mechanism for deciding whether an uploaded ride is "the same route." That has proven too brittle for real long-distance cycling behavior.

Recent debugging and instrumentation established several important facts:

- Exact or near-exact matched OSM way sequence is too strict for the equivalence class we care about.
- Reverse-order uploads, source variance (for example RWGPS vs Strava), midpoint starts, and route-expression differences can all produce different substrates even when a rider would reasonably consider the route the same.
- Dual-pass reconciliation, endpoint trimming, source normalization parity, and B-lite token cleanup improved the substrate but did not make equality-based canonical matching robust enough for real-world usage.
- Pass 2 is the wrong seam for this problem. Pass 2 is boundary refinement, not canonical identity or ride association.

At the same time, the actual domain behavior of randonneurs is broader than "upload equals route":

- Riders may start at home and ride 20, 30, or 50 miles to the route.
- Riders may enter a route at any point.
- Riders may ride the route in either direction when allowed.
- Riders may leave course briefly for food, rest, lodging, or other practical reasons.
- A long uploaded ride may contain the perm without being equal to the perm.

This means the product must serve two truths at once:

1. **Canonical route truth**: the perm itself must remain stable, strict, and geometry-first.
2. **Rider truth**: a rider can meaningfully have ridden the perm without uploading a route that is geometrically equal to it.

## Problem

The existing equality-based model is trying to answer multiple different questions with one mechanism:

- What is the canonical route?
- What exactly did the rider upload?
- Did the uploaded ride include the canonical route?
- Is this part of a larger expedition?

Those are different questions and should not be collapsed into a single canonical-equality test.

## Decision

Lanterne will adopt a layered model:

### 1. Canonical Route
The canonical route remains a strict, stable, geometry-first identity object.

This object represents the route itself, not the full variability of how riders may approach, leave, or embed it inside larger rides.

Canonical route identity should remain:

- strict
- versioned
- source-independent
- geometry/corridor-based
- separate from ride uploads and rider ownership

### 2. Ride Envelope
An uploaded ride is treated as a **ride envelope**.

A ride envelope represents what the rider actually did that day. It may include:

- ride-to-start miles
- the canonical route
- ride-home miles
- small detours for food, lodging, or practical needs
- route-expression differences that do not change the core route intent

A ride envelope is not assumed to be a canonical route candidate by equality.

### 3. Containment Association
Association between a ride envelope and a canonical route is based on **containment**, not equality.

The core question becomes:

**Does this ride envelope contain this canonical route as an ordered subpath of the rider-experienced corridor?**

Containment must support:

- entry at any point on the route
- traversal in either direction
- extra prefix and suffix miles without penalty
- tolerance for small local deviations and connector noise
- rejection of rides that merely share segments without actually containing the route

### 4. Expedition
An expedition is a durable higher-order journey layer above ride envelopes and canonical routes.

An expedition may be represented as an ordered series of ride envelopes, each of which may contain one or more canonical routes.

Expedition logic must not replace canonical identity.

## Rationale

This model cleanly separates concerns:

- **Canonical route** answers: what is the route?
- **Ride envelope** answers: what did the rider actually do?
- **Containment association** answers: did this ride include that route?
- **Expedition** answers: how do multiple rides roll up into a durable journey?

This matches both:

- the real rules and behavior of randonneurs
- Lanterne's broader architectural principle that source, identity, analysis, ride-time context, and rider-facing state should be separate concerns

## Containment Resolver Direction

The containment resolver should not rely on exact token equality.

The intended matching representation is:

- normalized route spine
- buffered corridor geometry
- sparse ordered anchor signature

### Why not exact matched way-id sequence?
Because it is too brittle across:

- reverse direction
- midpoint entry
- source normalization differences
- tiny connector differences
- fragmented OSM way splits
- harmless ride-expression changes

### Why not pure overlap only?
Because overlap alone can over-merge routes that share large common stretches.

### Why normalized spine + corridor + ordered anchors?
Because together they provide:

- subpath matching
- tolerance for noise
- order sensitivity
- protection against false positives on shared corridors

### Controls
Controls may serve as high-value anchors, especially for perm-oriented workflows, but controls alone are not sufficient for route identity or containment. They should be treated as important anchor features, not the only rule.

## Data Model Direction

No new top-level shadow object is introduced at this stage.

Instead, the minimum production addition is a first-class association relation between ride envelopes and canonical routes.

### Proposed association model
A future table or equivalent relation should capture:

- uploaded ride identifier
- canonical route identifier
- association type (`exact`, `contains`, `partial_overlap`, `ambiguous`)
- direction (`forward`, `reverse`)
- entry fraction
- exit fraction
- canonical coverage
- distance/offset metrics
- unmatched interior metrics
- anchor hit metrics
- confidence
- resolver version
- review/ambiguity flags

This preserves strict canonical identity while supporting rider-realistic containment behavior.

## Alternatives Considered

### A. Keep equality-based canonical matching as the main model
Rejected.

Reason:
It is too brittle for long-distance cycling behavior and failed repeated real-world equivalence tests.

### B. Loosen canonical identity itself
Rejected.

Reason:
This would weaken the route object and blur the distinction between the route and the rider's actual uploaded ride.

### C. Push remaining fixes into Pass 2
Rejected.

Reason:
Pass 2 is boundary refinement. This problem belongs in identity, containment, and resolver architecture, not boundary snapping.

### D. Introduce a new shadow canonical object immediately
Deferred.

Reason:
A shadow object may still be warranted later, but it risks moving fuzziness into a new matching layer before the containment model has been properly defined. The current decision is to first separate canonical identity from ride containment without inventing a second top-level route identity object.

## Consequences

### Positive

- Preserves strict perm identity.
- Matches real rider behavior.
- Handles home-to-route and route-to-home scenarios cleanly.
- Supports midpoint entry and reverse traversal.
- Creates a natural foundation for expedition logic.
- Avoids forcing all uploaded rides into fake equality tests.

### Negative / Risks

- Containment detection is more complex than equality.
- Shared-corridor false positives must be controlled.
- Thresholds and confidence need careful tuning.
- Additional schema and resolver complexity will be introduced.

## Out of Scope

This ADR does not define the final containment algorithm in detail.

It does not finalize:

- exact containment thresholds
- anchor selection algorithm
- final candidate shortlist strategy
- review/ambiguity workflow
- expedition table design

Those will follow in implementation-focused ADRs or design notes.

## Immediate Next Steps

1. Design the minimum containment resolver.
2. Define the ride-to-route association schema.
3. Validate the containment model on a representative rando test set:
   - exact perm
   - reverse traversal
   - midpoint start
   - home -> perm -> home
   - small off-course food detour
   - overlapping but different route
4. Keep canonical route identity strict.
5. Keep Pass 2 out of this work.

## Decision Summary

Lanterne will stop treating uploaded ride equality as the primary mechanism for recognizing that a rider rode a route.

Instead:

- **Canonical route identity remains strict.**
- **Uploaded rides are modeled as ride envelopes.**
- **Route recognition for uploaded rides is based on containment.**
- **Expedition is a higher-order layer above ride envelopes and canonical routes.**



---

## Source File: docs/03-adrs/adr-038-route_defined_vs_constraint_defined_activities.md

# ADR-0398: Route-Defined vs Constraint-Defined Activities

## Status
Proposed

## Date
2026-04-01

---

## Context

Lanterne supports multiple types of cycling experiences, including:
- Randonneur permanents (perms)
- Brevets
- Ultra-distance events
- User-created routes and GPX uploads
- Multi-day journeys (expeditions)

Initial implementations implicitly treated all activities as variations of a single concept: "a route defined by geometry." However, real-world usage revealed that this assumption does not hold.

There are fundamentally different types of activities:

1. Activities defined by a **specific route line**
2. Activities defined by **constraints (rules), not a fixed route**

Trying to force both into the same model leads to:
- brittle canonical matching
- incorrect identity assumptions
- poor representation of rider behavior
- confusion in downstream logic (containment, scoring, expedition tracking)

---

## Problem

The system currently lacks a clear distinction between:

- Activities where the **route itself is the identity**
- Activities where the **rules of the journey define the identity**

Examples:

### Route-defined
- Perms
- Fixed GPX routes
- Curated route libraries

### Constraint-defined
- Point-to-point ultras
- Checkpoint-based events
- Start/finish + mandatory controls
- Self-routed challenges

These require different logic for:
- identity
- matching
- validation
- rider association

---

## Decision

Introduce a first-class distinction between activity types:

### 1. Route-Defined Activities

Definition:
Activities where the route geometry is the primary identity.

Examples:
- Perms
- Fixed brevet routes
- Curated routes

Properties:
- Canonical route exists
- Geometry-first identity
- Matching via containment or equality
- Variants represent meaningful alternate geometries

Core question:
> Did the rider follow or contain this route?

---

### 2. Constraint-Defined Activities

Definition:
Activities where the rules define the experience, not the exact route line.

Examples:
- Ultra races with checkpoints
- Start-to-finish challenges
- Self-routed events

Properties:
- No single canonical geometry
- Defined by:
  - start/end
  - controls/checkpoints
  - time constraints
  - optional route hints
- Multiple valid geometries can satisfy the same activity

Core question:
> Did the rider satisfy the constraints?

---

### 3. Shared Concepts

Both activity types use:

- Ride Envelope (what the rider actually did)
- Association layer (relationship between ride and activity)
- Expedition layer (multi-ride journeys)

---

## Rationale

This separation allows:

- Clean canonical route identity for perms
- Flexible representation of ultras
- Correct containment logic
- Avoiding forcing arbitrary rides into incorrect equality models
- Alignment with real-world cycling behavior

It also prevents architectural confusion such as:
- treating constraint-defined events as canonical routes
- polluting canonical route identity with non-route semantics

---

## Alternatives Considered

### A. Single unified route model
Rejected

Reason:
Cannot accurately represent both route-defined and constraint-defined activities

---

### B. Encode constraints inside canonical route
Rejected

Reason:
Violates separation of concerns and corrupts route identity

---

### C. Separate “ultra object” entirely
Deferred

Reason:
Constraint-defined activities can be modeled as a subtype rather than a separate system

---

## Consequences

### Positive

- Cleaner domain model
- Better alignment with real-world use
- Enables correct containment and validation logic
- Supports future features like expeditions

### Negative / Risks

- Increased conceptual complexity
- Requires branching logic in resolver and UI
- Needs careful schema design

---

## Implementation Notes

Introduce field:

```text
course_model = route_defined | constraint_defined
```

Used to route logic paths for:
- canonical matching
- containment
- validation
- UI behavior

---

## Decision Summary

Lanterne will explicitly support two activity models:

- Route-defined: identity = geometry
- Constraint-defined: identity = rules

All downstream systems will respect this distinction.



---

## Source File: docs/03-adrs/adr-039-bounded_crossing_risk_and_report_only_critical_stretch.md

# ADR-039: Bounded Crossing Risk Contribution + Report-Only Critical Stretch

**Status:** Accepted  
**Date:** 2026-04-04  
**Model Version:** v3.1-launch

## Context

The V3.0 crossing-conflict penalty model used a flat base penalty (0.12 risk points) with speed/traffic/width gate multipliers. This produced unbounded penalties for crossing-dense urban routes and did not account for crossing control type or movement type.

The V3.0 model also applied critical-stretch caps to the canonical Safety Score, which conflated route-average risk with localized risk exposure.

## Decision

### Crossing Risk Contribution

Replace the V3.0 crossing-conflict penalty with a bounded crossing risk contribution model:

```
CrossingEventContribution = min(E_cap,
    E0 × sqrt(SpeedFactor × TrafficFactor)
       × WidthFactor × ControlFactor × MovementFactor)
```

**Launch constants:**
- **E0** = 0.05 — base crossing risk contribution in risk points. This is a policy-derived constant representing the baseline risk of a single motor-vehicle conflict crossing, calibrated against the continuous risk scale where ~1.0 RPM represents a residential road.
- **E_cap** = 0.75 — maximum risk contribution any single crossing can produce, preventing outlier crossings from dominating the score.

**WidthFactor** (lanes crossed):
| Lanes | Factor |
|-------|--------|
| 1–2   | 1.00   |
| 3–4   | 1.25   |
| 5–6   | 1.60   |
| 7+    | 2.00   |

**ControlFactor**:
| Type | Factor |
|------|--------|
| Signalized | 1.00 |
| Stop-controlled | 1.05 |
| Unknown | 1.10 |

**MovementFactor**:
| Movement | Factor |
|----------|--------|
| Straight | 1.00 |
| Right/merge | 1.05 |
| Left across traffic | 1.20 |
| Unknown | 1.10 |

**Crossing scoring eligibility** (at least one must be true):
- Crossed road speed ≥ 30 mph AND AADT per lane ≥ 2,000/day/lane
- Lanes crossed ≥ 3
- Left across traffic on a road with speed ≥ 30 mph or AADT per lane ≥ 2,000/day/lane

**Route-level crossing cap:**
```
EffectiveCrossingRPM = min(RawCrossingRPM, ContinuousRPM × 0.6667)
```
This ensures crossings cannot exceed 40% of raw canonical route risk.

### Critical Stretch: Report-Only

The critical-stretch mechanism (worst-1km RPM → score cap) is retained for transparency and report-only purposes but does **NOT** modify the canonical Safety Score.

The canonical Safety Score is derived solely from:
```
RawRPM = ContinuousRPM + EffectiveCrossingRPM
SafetyScore = 100 / (1 + e^(1.4 × (RawRPM - 2.5)))
```

## Consequences

1. Crossing risk is bounded per-event and as fraction of route risk
2. Canonical score is purely RPM-derived — no hotspot overrides
3. Critical stretch remains useful for rider awareness
4. Urban crossing-dense routes are no longer disproportionately penalized
5. Control type and movement type are captured for future refinement

## Explicit Exclusions from Canonical Score

- Critical stretch / hotspot penalties
- Time-of-day traffic adjustments
- Rail / grate / cattle-guard hazard penalties
- Weather / light conditions
- Signalized/stop-controlled crossing counts alone


---

## Source File: docs/03-adrs/adr-039-segment-level-analysis-storage-model.md

# ADR-039 — Segment-Level Analysis Storage Model

Status: Accepted
Date: 2026-04-03

------

## Context

Lanterne performs detailed route analysis by evaluating many small internal slices of a route.

These slices capture real-world variation in:

- traffic exposure
- speed environment
- shoulder presence
- bike infrastructure
- hazards

This slice-level computation is essential to avoid smoothing over critical changes along a route.

However, a key architectural question emerged:

> Should segment-level analysis be persisted in the database?

------

## Decision

Lanterne will **NOT persist segment-level analysis in the database**.

Segment-level data is:

- computed client-side
- stored only in memory during analysis
- optionally cached inside `route_cache.safety_result` as JSON
- never stored in structured database tables

------

## Rationale

### 1. Segment data is derived, not canonical

Segment-level risk is:

- computed from inputs
- dependent on scoring model version
- subject to change

It is not a stable fact.

Storing it would incorrectly elevate derived data to canonical status.

------

### 2. The system is designed for client-side computation

Lanterne’s architecture is intentionally:

- browser-first
- compute-on-demand
- server-light

> All analysis runs in the user's device.

Persisting segment-level data would introduce unnecessary backend coupling.

------

### 3. Storage adds complexity without rider value

Persisting segment-level data would require:

- schema design
- migrations
- versioning
- invalidation logic
- synchronization

These add engineering cost but do not improve the rider’s experience for the core use case.

------

### 4. Caching already solves performance

The system already uses:

- tile-level caching (OSM roads)
- route-level caching (analysis results)

These eliminate redundant computation without requiring persistent segment storage.

------

### 5. Product focus is route-level decision support

Lanterne’s core promise is:

> Help riders understand a route before they ride it.

It is not intended to be:

- a global segment analytics platform
- a shared road intelligence database

------

## Implications

### What is stored

- route geometry
- route-level analysis results
- cached tiles
- user data

------

### What is NOT stored

- segment-level risk
- segment-level scoring
- segment-level presentation tokens

------

### Where segment data exists

Segment-level data exists only as:

```txt
in-memory during analysis
+
JSON inside route_cache.safety_result
```

------

### System behavior

- segment truth is recomputed when needed
- updates to scoring models automatically apply
- no backfill or migration is required

------

## Consequences

### Advantages

- simpler architecture
- faster iteration
- no schema drift
- no invalidation complexity
- lower infrastructure cost
- consistent with client-side compute model

------

### Tradeoffs

- no cross-route segment querying
- no persistent segment analytics
- no server-side reuse of segment outputs

These are acceptable given current product goals.

------

## Future Considerations

This decision may be revisited if:

- server-side compute becomes necessary
- cross-route analytics becomes a core product feature
- segment-level community data requires structured storage

Until then, segment-level persistence is explicitly out of scope.

------

## Design Principle

> Segment-level analysis is derived, not stored.

------

## Final Rule

Lanterne computes truth on demand.

It does not store derived segment data.

## 

---

## Source File: docs/03-adrs/adr-040-user-observations-as-overlay-evidence-not-canonical-truth.md

# ADR-040 — User Observations as Overlay Evidence, Not Canonical Truth

Status: Accepted
Date: 2026-04-03

## Context

Lanterne now supports rider-submitted observations for safety-relevant facts such as:

- speed
- shoulder
- bike infrastructure

These observations may be submitted without authentication.

The product intent is to let riders contribute what they see on the road while preserving the integrity of Lanterne’s deterministic safety model.

Recent implementation work introduced a session observation layer, a fire-and-forget persistence path, and a UI submission flow. During that work, user observations were temporarily being injected into canonical truth resolution.

That behavior was architecturally incorrect.

It violated several core system principles:

- canonical truth must remain deterministic
- Safety Score must remain explainable from model inputs
- user observations are evidence, not truth
- the heatmap must not become crowd-voted or silently user-modified

This ADR formalizes the correction.

## Problem

If user observations are allowed to enter the canonical truth resolver directly, then:

- scoring provenance becomes ambiguous
- heatmap colors may change due to unreviewed user input
- deterministic truth becomes contaminated by ad hoc evidence
- guest submissions could silently influence the model
- future moderation and override workflows become harder to design cleanly

In other words, the system begins to collapse the distinction between:

- canonical truth
- rider evidence
- reviewed overrides

Those are different layers and must remain separate.

## Decision

Lanterne will treat user observations as a **presentation-layer evidence overlay**, not as a canonical truth input.

### Final rule

**User observations do not participate in canonical truth resolution.**

They do not appear in:

- `collectSpeedEvidence`
- `collectShoulderEvidence`
- `collectBikeInfraEvidence`
- or any equivalent canonical resolver input path

They do not affect:

- Safety Score
- heatmap colors
- risk calculations
- canonical inspector truth
- any other deterministic scoring or truth output

Instead, user observations are applied only as:

- immediate session/UI overlays
- persisted evidence records for future review

## Architectural Model

### 1. Canonical path

Deterministic sources only:

- OSM
- HPMS / DOT
- deterministic inference

These feed:

- canonical truth resolver
- scoring engine
- heatmap builder
- canonical truth display

### 2. Observation path

User-submitted observations feed:

- session-local immediate UI feedback
- persisted evidence storage
- presentation-layer overlays

They do **not** feed:

- canonical truth resolver
- scoring
- heatmap logic
- risk calculations

### 3. Current UI behavior

User observations are rendered in `TruthSection` as display-only overlays, labeled explicitly with:

- `"(your observation)"`
- `user_observation` source badge

The underlying canonical truth remains visible and deterministic.

## Rationale

### 1. Trust in the score depends on deterministic provenance

Lanterne’s free safety promise only works if riders can trust that the model is grounded in deterministic inputs rather than raw crowd edits.

### 2. Evidence and truth are not the same thing

A rider-submitted observation may be useful, timely, and directionally correct. That still does not make it canonical truth.

Observations are evidence. Truth is the result of governed resolution.

### 3. This preserves a clean future moderation path

By keeping observations out of the resolver now, Lanterne retains the ability to later build:

- moderation workflows
- corroboration rules
- trusted-source weighting
- reviewed override layers

without contaminating the live scoring path.

### 4. It protects guest-mode contributions without weakening the model

Unauthenticated riders can still contribute useful field intelligence. Those contributions can be acknowledged immediately in the UI and stored for later use, without granting them direct influence over canonical truth.

### 5. It prevents the heatmap from becoming a crowd-edited surface

The heatmap must remain the output of the deterministic model. It must not become a blended artifact of live user submissions.

## Consequences

### Positive

- canonical truth remains deterministic
- Safety Score provenance remains explainable
- heatmap trust is preserved
- guest contributions remain possible
- UI can acknowledge rider input immediately
- future moderation / override workflows stay cleanly separable

### Tradeoffs

- user observations do not immediately improve the score
- riders may see a mismatch between their observation and canonical truth
- additional future work is required if approved evidence is later allowed to influence model inputs

These tradeoffs are acceptable and intentional.

## Non-Goals

This ADR does **not** define:

- moderation workflow
- corroboration thresholds
- auth-weighted evidence classes
- reviewed override governance
- aggregation logic that upgrades observations into stronger factual signals

Those may come later.

This ADR defines only the current architectural boundary.

## Future Direction

A future governed system may allow reviewed evidence to produce Lanterne-specific overrides that supersede upstream source values.

If that happens, the architecture must remain:

raw observation
→ review / moderation
→ approved override
→ canonical resolver input

Not:

raw observation
→ canonical resolver input

That distinction is mandatory.

## Implementation Notes

The canonical truth resolver has already been corrected so that user observations no longer participate in:

- `collectSpeedEvidence`
- `collectShoulderEvidence`
- `collectBikeInfraEvidence`

Current behavior:

- scoring, heatmap colors, and risk calculations are unaffected by user input
- observations render only as display overlays in `TruthSection`
- overlays are explicitly marked as `user_observation`

## Design Principle

**User observations are evidence overlays, not canonical truth inputs.**

## Decision Summary

Lanterne accepts rider observations as useful field evidence.

But those observations do not alter canonical truth, scoring, or heatmap behavior.

Canonical safety outputs remain fully explainable from deterministic model inputs only.

---

## Source File: docs/03-adrs/adr-041-crossing_risk_contribution_and_critical_stretch.md

# ADR-041 — Bounded Crossing Risk Contribution and Report-Only Critical Stretch

Status: Accepted  
Date: 2026-04-04

## Context

Lanterne’s headline Safety Score is intentionally narrow:

> relative expected harm from a bicyclist being struck by a motor vehicle

That definition remains stable.

Recent research and implementation pressure-testing established several important truths:

1. **Continuous segment exposure from speed and motor-vehicle volume over distance must remain the backbone of the score.** National bicycle fatality patterns and bicycle safety modeling frameworks do not support treating intersections as the dominant story by default. Most bicyclist fatalities occur away from intersections, and midblock motor-vehicle bicycle crashes are more likely to be fatal or serious than intersection motor-vehicle bicycle crashes. [1]

2. **Crossing and turn conflicts matter, but they are structurally different from riding along a segment.** They should be modeled as bounded discrete events, not smeared into segment exposure and not allowed to overwhelm long-route segment risk. [1][2]

3. **Open data supports some crossing inputs well, some partially, and some not at all.** Nationally feasible launch inputs include posted speed, some AADT coverage, lane count, turn direction from route geometry, and some signal/stop control tagging. Nationally infeasible launch inputs include turning counts, signal phasing, and reliable yielding behavior everywhere. [1][2]

4. **Critical stretch is useful as an explanation layer but not defensible enough to remain in canonical score math at launch.** It belongs in the analysis drawer and explainer surfaces, not in the narrow canonical score.

5. **Time-of-day contextual traffic interpretation is useful, but not canonical.** AADT bell-curving and ride-time traffic storytelling belong in cue sheet context, explainers, and future push intelligence—not in the canonical score.

## Decision

### 1. Canonical Safety Score structure
The canonical Safety Score is built from:

- continuous segment exposure risk
- bounded crossing risk contribution

### 2. Continuous segment exposure remains the backbone
Continuous segment exposure is computed on small internal slices using:

- a non-linear speed factor
- a traffic factor
- infrastructure mitigation
- shoulder mitigation

### 3. Crossing conflict is modeled as a bounded event contribution
Crossing and turn conflicts are modeled as discrete events using only launch-feasible inputs:

- speed environment of crossed/entered road
- traffic intensity of crossed/entered road
- crossing width / lanes crossed
- control type when available
- movement type when derivable

These events are added as a bounded secondary contribution rather than replacing the segment backbone.

### 4. Scored crossing-event eligibility
A crossing event enters score math when at least one of the following is true:

- the crossed or entered road has **speed ≥ 30 mph** **and** **AADT per lane ≥ 2,000/day/lane**
- the crossing requires **3 or more lanes** to be crossed
- the movement is **left across traffic** on a road that is either **≥ 30 mph** or **≥ 2,000 AADT/day/lane**

Signalized and stop-controlled crossings may still be counted and reported even when not every one is score-bearing.

### 5. Route-level crossing share is capped
At launch, effective crossing risk contribution may not exceed **40% of total raw canonical route risk**.

This is an explicit product-policy safeguard informed by the evidence base that speed and volume over distance remain the stronger backbone of severe-harm risk for long endurance routes. [1][2]

### 6. Critical stretch is report-only
Critical stretch remains a report and explanation layer only.

It may appear in:
- analysis drawer
- route explanation surfaces
- cue / push intelligence context

It does not modify the canonical Safety Score.

### 7. Time-of-day traffic context is report-only
AADT bell-curving / time-of-day traffic contextualization is not part of canonical score math.

It may appear in:
- cue sheet context
- score explanation surfaces
- future push-intelligence surfaces

### 8. Comparative / endurance-route context is separate
Canonical Safety Score remains absolute within the model.

Corpus-relative percentile context for endurance rides may be shown separately, but must not rescale the canonical score.

## Rationale

This keeps the score:
- narrow
- explainable
- benchmark-shaped
- honest about data limitations

It also separates **benchmark-derived structure** from **launch policy safeguards**:

- benchmark-derived: speed non-linearity, traffic flow relevance, factorized intersection structure, lanes-to-cross relevance
- policy-derived: crossing event cap, route-level crossing share cap, compression exponents, launch calibration constants

## Consequences

### Positive
- better alignment with benchmarked safety logic
- fewer black-box intersection claims
- better suitability for long endurance routes
- cleaner separation between canonical score and interpretation layers

### Tradeoffs
- some launch constants remain explicit policy choices
- crossing math is intentionally simplified relative to engineering-grade intersection prediction
- critical stretch and time-of-day traffic move out of the headline score

These tradeoffs are accepted.

## Design principles

1. Continuous exposure is the backbone.  
2. Crossings are bounded event contributions.  
3. Critical stretch explains the route; it does not define the canonical score.  
4. Public docs must distinguish benchmark-derived structure from launch policy values.  
5. Unknown values must remain unknown rather than being silently promoted to truth.

## References

[1] `ass-005-lanterne_safety_model_pressure_test.md` — production-oriented pressure test of Lanterne’s intersection crossing risk, bounded contributions, hotspot logic, and public transparency.  
[2] `ass-006-defensible_math_for_crossings_and_speed.md` — defensible production math shapes for Lanterne crossings and speed.


---

## Source File: docs/03-adrs/adr-042-evidence_resolution_and_truth_propagation_model.md

# ADR-042 — Evidence Resolution & Truth Propagation Model

This ADR operates on the evidence layers defined in ADR-032.

Status: Accepted
Date: 2026-04-XX

------

## 1. Purpose

Define how Lanterne determines **canonical segment truth** for:

- speed
- shoulder
- bike infrastructure
- traffic behavior

This ADR governs:

- evidence precedence
- propagation behavior
- decay rules
- separation of observations vs canonical inputs

------

## 2. Core Principle

A segment has:

- one canonical truth
- multiple possible evidence sources

Truth is derived deterministically from evidence.

User-submitted observations are **not canonical truth**.

------

## 3. Evidence Precedence

When multiple sources exist:  

1. observed (founder/admin)
2. authoritative_posted (DOT / HPMS)
3. osm_posted
4. observation_inferred
5. authoritative_inferred
6. osm_inferred
7. regional_prior
8. highway_area_baseline
9. highway_baseline

Lower-confidence sources must not overwrite higher-confidence ones.
**Measured** evidence is the highest-priority input and supersedes all other evidence types.

------

## 4. Truth Dominance Along a Road

If a reliable value is known for a road:

That value becomes the **dominant truth** along that road.

It propagates forward and backward until contradicted.

------

### Propagation continues until:

- road name changes
- strong conflicting evidence appears
- explicit override exists

------

### Propagation does NOT stop on:

- highway type changes alone
- minor OSM segmentation differences

------

## 5. Propagation Model

Propagation operates along continuous road identity.

Segments inherit upstream truth when:

- no stronger evidence exists
- continuity is maintained

------

## 6. Decay Rules by Dimension

### Speed

- highly stable
- propagates long distances
- primary driver of safety

------

### Shoulder

- context-dependent
- rural: long propagation
- suburban/urban: short propagation

------

### Bike Infrastructure

- discontinuous
- strict decay
- resets frequently

------

### Traffic Behavior

- semi-stable
- influenced by regional context
- decays moderately

------

## 7. Regional Inference

Regional priors improve baseline inference.

Examples:

- state-level speed norms
- road-class-specific behavior

Regional priors:

- stronger than baseline
- weaker than direct evidence

------

## 8. User Observations

User observations:

- are stored in a separate observations layer
- do not directly modify canonical truth
- may be applied as session-level overlays
- may be aggregated in future

------

## 9. Separation of Concerns

Canonical truth is used for:

- scoring
- heatmap
- analysis

Observations are used for:

- UI overlays
- aggregation pipelines
- future inference upgrades

------

## 10. Determinism Requirement

Given identical inputs:

- resolver must produce identical outputs

User observations must not affect deterministic resolution.

------

## 11. Design Principle

Truth is not crowdsourced in real time.

Truth is derived from evidence.

Evidence accumulates and is validated before influencing the model.

------

END

---

## Source File: docs/03-adrs/adr-043-confidence_and_provenance_model.md

# ADR-043 — Confidence and Provenance Model

**Status:** Accepted
**Date:** April 12, 2026
**Filename:** `ADR-043-confidence_and_provenance.md`

## Context

Lanterne’s canonical Safety Score is intentionally narrow:

> **relative expected harm from a bicyclist being struck by a motor vehicle**

That definition must remain stable.

At the same time, the system relies on a mix of:

- directly observed data

- official imported data

- geometry-derived truth

- inferred values

- predicted values

- generic baselines and fallbacks

- incomplete or unknown tags

  Those inputs do **not** deserve equal trust.

  If Lanterne collapses them into one flat pool of “truth,” the product becomes less explainable, less auditable, and less trustworthy. A route can appear precise when it is actually standing on weak assumptions. For a product built on rider trust, that is not acceptable.

  This is not only a scoring concern. Confidence and provenance affect:

- route matching quality

- speed and traffic truth

- facility and shoulder truth

- crossing control and movement truth

- fallback handling

- route-level trustworthiness

- UI caveats and diagnostics

- future score tracing and change summaries

  They therefore need first-class architectural treatment.

## Decision

Lanterne will use a three-part trust model:

1. **Provenance**  
   Where a fact came from.

2. **Confidence**  
   How strongly the system believes that fact is correct.

3. **Evidence precedence**  
   Which fact wins when multiple candidate values disagree.

   These signals will be modeled throughout the system, but they will **not** be folded directly into the canonical Safety Score formula.

   The canonical score answers:

> **How risky is this route or segment, given the best available modeled inputs?**

Confidence answers:

> **How much should the rider trust that those inputs are grounded in strong evidence rather than weak inference or fallback?**

Provenance answers:

> **Why does the system believe what it believes?**

Those questions must remain separate.

## Provenance model

Every load-bearing scoring input should carry a provenance class.

### Required provenance classes

- **observed**  
  Direct field measurement or equivalent first-hand truth.

- **official_imported**  
  Imported from an authoritative public or agency dataset.

- **geometry_derived**  
  Deterministically derived from route or map geometry.

- **inferred**  
  Deterministically inferred from nearby or related known truths.

- **predicted**  
  Model output.

- **baseline**  
  Generic prior used when stronger evidence is absent.

- **unknown**  
  No reliable source or derivation available.

### Design rule

A field’s provenance must never be hidden. If the score uses a value, the system must know **what class of evidence produced it**.

## Confidence model

Confidence is a separate field family from value and provenance.

Confidence is not “good” or “bad.” It is a bounded signal describing evidence strength.

### Allowed confidence representations

Internally, Lanterne may use either:

- numeric confidence, typically `0.0–1.0`, or

- discrete bands such as:

  - **high**

  - **medium**

  - **low**

  - **unknown**

    Numeric storage is preferred. Presentation may simplify it into bands later.

## Required confidence layers

Confidence must exist at multiple layers.

### 1. Field-level confidence

Examples:

- posted speed confidence
- AADT confidence
- facility confidence
- shoulder confidence
- movement classification confidence
- control classification confidence

### 2. Segment-level confidence

A segment’s confidence should reflect the quality of the facts feeding it.

### 3. Crossing-event confidence

Crossing events must carry their own confidence, especially where:

- movement is ambiguous
- control truth is missing
- lanes crossed are estimated
- speed or traffic truth is inferred rather than directly imported

### 4. Route-level confidence

A route-level confidence summary should reflect:

- proportion of route scored from strong evidence
- proportion using inference, prediction, or baseline fallback
- route matching quality
- number and importance of low-confidence hotspots

## Evidence precedence

When candidate facts conflict, Lanterne will apply this default order:

1. **observed**
2. **official_imported**
3. **geometry_derived**
4. **inferred**
5. **predicted**
6. **baseline**
7. **unknown**

### Important distinction

Precedence decides **which value wins**.

Confidence decides **how much trust that winning value deserves**.

Those are related, but not identical.

## Canonical score rule

Confidence must **not** be folded directly into canonical Safety Score math.

### Why

If confidence is fused into the score:

- a dangerous route can look safer just because the system knows less
- missing data can silently soften danger
- riders cannot distinguish “low risk” from “poorly known”
- the score becomes harder to explain and easier to distrust

### Therefore

Canonical score math should use:

- the **best available chosen input**

- with **provenance and confidence tracked separately**

  Confidence should instead affect:

- UI caveats

- trace output

- ranking caution

- diagnostics

- future route-comparison warnings

- whether some exact-looking displays should be suppressed

## Unknown handling rule

Unknown values should not secretly become stronger danger claims than all known values.

When a score modifier needs a value and only unknown is available:

- the canonical score should use a **bounded neutral fallback**
- uncertainty should be reflected through **confidence loss**
- provenance must explicitly say **unknown**

### Example

If known control states are:

- stop-controlled = `1.00`

- signalized = `1.05`

  then unknown control may use the arithmetic mean:

$$
\text{UnknownControlFactor} = \frac{1.00 + 1.05}{2} = 1.025
$$

The score remains bounded and neutral. The real penalty is carried in confidence and trace metadata.

## What confidence should influence

Confidence should influence:

- route-level confidence badges or caveats
- trace and inspector displays
- whether a route is marked mixed-confidence or limited-confidence
- future comparison and ranking warnings
- whether partial or low-confidence results are cached as provisional
- whether exact-looking public numbers are suppressed when they overstate certainty

## What confidence should not influence

Confidence should not:

- directly rescale the canonical Safety Score
- silently change grade thresholds
- hide missingness
- overwrite stronger provenance with weaker provenance
- exist only as cosmetic UI garnish

## Output requirements

Every score-bearing artifact should eventually preserve enough provenance and confidence to explain itself.

### Segment-level

At minimum:

- chosen values
- provenance for each chosen value
- confidence for each chosen value
- segment confidence summary

### Crossing-level

At minimum:

- movement, control, lanes crossed, speed, traffic
- provenance for each
- confidence for each
- event confidence summary

### Route-level

At minimum:

- match quality
- observed / official / inferred / predicted / baseline proportions
- route confidence band
- count of low-confidence contributors
- fallback summary

## UI implications

Confidence must be visible, but not noisy.

### Rider-facing principle

Prefer concise, plain-English cues such as:

- **High confidence**

- **Mixed confidence**

- **Limited confidence**

  Optional helper copy may include:

- “Most of this route is based on direct or official data.”

- “Parts of this route rely on estimated traffic or missing crossing detail.”

- “This result is usable, but several important inputs are inferred.”

### Inspector / trace principle

Detailed provenance and confidence belong in deeper surfaces:

- score tracing
- inspector
- diagnostics
- admin tuning tools

## Storage implications

Confidence and provenance are not temporary debug junk. They are part of the canonical analysis record.

Core route-level and segment-level trust signals should be stored in structured form where practical. Detailed breakdowns may live in bounded JSON artifacts.

At minimum, route-level analysis outputs should preserve:

- route confidence summary
- fallback counts
- evidence mix
- match quality

## Relationship to ADR-031

ADR-031 already establishes naming discipline and evidence precedence for future traffic-behavior facts. ADR-043 generalizes that same discipline across the broader scoring system.

Examples of aligned field names include:

- `observed_*`

- `official_*` or `official_imported_*`

- `inferred_*`

- `predicted_*`

- `baseline_*`

- `confidence_*`

- `score_*`

  ADR-043 does not replace ADR-031. It extends the same trust logic to canonical scoring inputs and outputs.

## Non-goals

This ADR does **not** require:

- immediate implementation of a full confidence UI

- changing the canonical Safety Score formula

- blocking launch until every input has perfect confidence scoring

- defining every numeric confidence formula today

- building all provenance tables before V4 scoring lands

  This ADR establishes the governing rule set. Implementation can phase in.

## Consequences

### Advantages

- strengthens trust without diluting the score
- keeps uncertainty explicit
- prevents missingness from masquerading as safety truth
- creates a stable foundation for score tracing
- supports future observed / predicted traffic-behavior work

### Tradeoffs

- increases schema and implementation complexity
- requires cross-system discipline
- demands consistency between scoring, UI, and diagnostics
- removes a number of tempting but misleading shortcuts

## Design principle

**A dangerous route should not look safer because the system knows less.**

The score answers **risk**.  
Confidence answers **trust**.  
Provenance answers **why**.

Those must remain separate.



---

## Source File: docs/03-adrs/adr-044-profile_based_routing_and_alternate_route_policies.md

# ADR-044 — Profile-Based Routing and Alternate Route Policies

**Status:** Accepted  
**Date:** 2026-04-17  
**Related:** ADR-001, ADR-002, ADR-005, ADR-032, [DS-021](../02-architecture/design/ds-021-profile_based_routing_and_alternate_route_policy_spec.md), [EXEC-013](../04-execution/exec-013-profile_based_routing_implementation_plan.md), [ASS-010](../assessments/ass-010-phase0_routing_audit.md)

---

## 1. Context

Lanterne’s current pre-ride contract is meaningful but still narrow:

- upload or create a route
- analyze it
- inspect risk and truth
- manually adjust the route if needed

That is useful, but it does not fully answer the practical rider question:

> “I know where I want to go. Can Lanterne offer a better way to get there?”

The existing Route To and draw/edit surfaces already prove that routing is within reach. The problem is not capability in the abstract. The problem is that the current routing-related code is fragmented across multiple overlapping systems:

- Route To / route creation via OSRM waypoint routing
- live detour editing via `src/lib/detour-routing.ts`
- hidden optimizer concepts and preview scoring via `src/lib/routing.ts` and `RouteOptimizer.tsx`
- separate local detour exploration via `realtime-detour.ts`

This fragmentation creates architectural risk:

1. **Score drift**
   - Safety Score must remain narrow and absolute.
   - Routing preferences may not redefine score semantics.

2. **RouteMap bloat**
   - routing policy may not accumulate inside `RouteMap.tsx`

3. **Split-brain routing behavior**
   - Route To, draw-leg recompute, detours, and any future optimizer work may not each invent a separate policy model

4. **Fake alternatives**
   - nearly identical routes may not be shown as distinct rider-facing choices just to create the appearance of intelligence

5. **Unbounded search**
   - alternate discovery must be budgeted and honest about failure/no-result outcomes

6. **Loss of niche value**
   - old optimizer ideas such as brevet-safe distance constraints may still matter, even if the implementation does not survive

---

## 2. Decision

Lanterne will implement a **shared profile-based routing preference engine**.

This engine will:

- use one underlying routing-engine contract with one normalized edge / cost model at the Lanterne boundary
- support multiple routing policies through different edge-cost functions
- be reusable across:
  - `Route To`
  - draw-mode leg recomputation

Launch rider-facing route choices are:

- **Direct**
- **Safer**
- **Lower Traffic**
- **Bike Support**

`Direct` is the default contract.  
The other three are alternates.

`Balanced` may exist internally if it is useful for engineering orchestration, but it is **not** a visible launch option.

---

## 3. Product contract

### 3.1 Route To

When a rider searches for a destination:

- compute **Direct** first
- treat Direct as the comparison baseline
- offer **Safer**, **Lower Traffic**, and **Bike Support** only when they are meaningfully different

### 3.2 Draw

When a rider has created a route with anchor points:

- the same routing-preference engine must be usable to recompute an individual leg between anchors
- this is not a second routing architecture
- this is the same policy engine applied to a narrower working scope

### 3.3 Non-goals

This decision does not introduce:

- freeform rider preference sliders
- rider-editable route-policy weights
- giant routing-settings panels
- “AI optimizer” positioning
- a second policy system just for Draw mode

Launch emphasis is:

- understandable choices
- honest compute limits
- real route tradeoffs
- clear architectural ownership

---

## 4. Core routing principle

### 4.1 One graph, many policies

All visible route profiles must use:

- the same underlying routing engine for a given request
- the same normalized edge attributes at the Lanterne integration boundary
- the same route-comparison and suppression rules

Visible route differences come from **policy**, not from separate hidden routing stacks.

Implementation note:

- this ADR does **not** require Lanterne to own or self-develop the underlying routing engine
- the preferred long-term shape is an external engine integration, with GraphHopper as the leading candidate
- Lanterne owns profile semantics, comparison rules, suppression rules, and presentation contract

### 4.2 Shared edge attributes

The policy engine should reason over normalized edge attributes such as:

- distance
- estimated travel time
- speed burden
- traffic burden
- bike-support burden
- shoulder burden
- hazard burden
- turn / crossing / intersection burden

### 4.3 Shared route comparison rules

All alternates must be compared against Direct using a common contract.
Alternates that are too similar, too weakly differentiated, or too costly for the gain must be suppressed.

---

## 5. Safety Score separation rule

The headline Safety Score remains:

> the absolute risk to the rider from motor-vehicle collision likelihood and resulting severity.

That meaning is fixed.

Routing profiles influence:

- pathfinding preference
- route selection

Routing profiles do **not** influence:

- the semantic meaning of Safety Score
- canonical risk interpretation
- the score curve

Examples:

- a dangerous road does not become “safe” because it is common locally
- a route chosen under `bike_support` still receives the same canonical score logic as any other route

---

## 6. Visible launch profiles

### 6.1 Direct

The most direct reasonable route.

Use when:

- the rider primarily wants straightforward travel
- alternates are comparisons against a familiar baseline

### 6.2 Safer

A route willing to spend modest extra distance or time to reduce absolute rider risk.

### 6.3 Lower Traffic

A route more strongly biased away from stressful motor-vehicle context, even when overall risk and bike-support considerations do not fully overlap.

### 6.4 Bike Support

A route that more strongly prefers bike-supportive roads where the system has meaningful truth.

This includes known infrastructure / shoulder context where available. It does **not** promise continuous bike lanes.

---

## 7. Brevet policy preservation

The old optimizer’s brevet-aware concept is strategically valuable and should not be discarded as generic legacy clutter.

Lanterne will preserve **brevet-safe routing constraints** as a specialized policy / constraint capability.

### 7.1 Launch posture

Brevet is **not** a default visible Route To button at launch.

### 7.2 Preservation rule

Brevet remains a specialized policy / constraint family for:

- brevet-aware route optimization
- route-adjustment workflows where shortening the route could invalidate the rider’s effort
- future event/control-aware routing work

### 7.3 Constraint rule

A brevet-aware mode may not shorten a qualifying route below the applicable route-distance floor unless explicitly allowed by the rider or by the surrounding workflow contract.

The old implementation does not need to survive for this rule to survive.

Preserve the concept.  
Refactor or reimplement the mechanics if needed.

---

## 8. Search-budget rule

Alternate-route discovery must be **bounded**.

The routing engine may not grind indefinitely in pursuit of tiny marginal improvements.

The system must support:

- explicit search budgets
- explicit extra-distance / extra-time tolerances
- route-size-aware limits
- profile-aware early-stop logic
- honest “no better route found within current limits” behavior

This is a launch requirement, not an optional optimization.

---

## 9. Display corridor vs routing-search corridor

The display corridor and the routing search horizon are not the same thing.

Lanterne may continue using display-oriented constraints for:

- map responsiveness
- heatmap rendering
- bounded overlay cost

But alternate-route discovery may not be silently crippled by those same limits.

A distinct routing-search horizon / budget policy is required.

---

## 10. Legacy routing audit rule

Existing routing, detour, and optimizer code must be audited before implementation continues.

For each major legacy path, classify it as:

- `reuse_directly`
- `reuse_with_refactor`
- `do_not_reuse`
- `remove`

The required Phase 0 audit is captured in [ASS-010](../assessments/ass-010-phase0_routing_audit.md).

---

## 11. Repo-specific audit notes

The current repo audit does **not** change the product contract above, but it sharpens implementation posture:

- `src/lib/detour-routing.ts` may be reusable with refactor for diverge/merge/splice concepts
- `useDetourHistory` may survive as an integration surface
- root-level `detour-routing.ts` is a stale duplicate and is a removal candidate
- `RouteOptimizer.tsx` should not survive as the launch routing architecture
- `realtime-detour.ts` and `useRealtimeDetour.ts` are not the launch profile-routing foundation
- preview scorers and old optimizer modes may not become canonical routing truth

---

## 12. Consequences

### Positive

- clearer rider-facing route choices
- one routing policy family instead of multiple hidden ones
- easier comparison and suppression of weak alternates
- cleaner reuse between Route To and Draw
- stronger basis for future specialized policies such as brevet constraints

### Costs

- legacy routing code will need quarantine or removal
- search budgets and stop rules must be implemented explicitly
- a new routing-policy layer must be introduced outside `RouteMap.tsx`

### Explicit non-consequence

This ADR does **not** redefine Safety Score semantics.

