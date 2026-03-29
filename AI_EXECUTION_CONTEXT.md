# Lanterne Execution Systems Context


---


This document contains operational documentation for building and running Lanterne.

Sources included:

• execution plans
• system manuals
• infrastructure projects
• architecture audits
• migration history


---

## Source File: docs/04-execution/exec-000-READ_ME.md

# exec-000-READ_ME

This file is the **front door** to the Lanterne docs.

Its job is simple:

- tell you what each top-level folder is for
- tell you where to start
- tell you where the important open paths live now
- stop you from getting lost in a pile of good docs

This is **not** the source of truth for architecture decisions.
It is the map to the source of truth.

---

# 1. How this docs repo is organized

## 01-philosophy
This is the “why” layer.

Read this when you want to remember:
- what Lanterne believes
- what kind of product it is trying to be
- what the core analysis worldview is

Start here if you feel the project getting muddy.

Main files:
- `phi-001-lanterne_manifesto.md`
- `phl-002-product_principles.md`
- `phi-003-analysis_model.md`

---

## 02-architecture
This is the “how the system is shaped” layer.

It contains:
- data model
- system architecture
- project map
- system guide
- design specs

If you want to know how the machine is supposed to work, this is where you go.

Important files:
- `arch-001-data_model.md`
- `arch-002-system_architecture.md`
- `arch-003-project_map.md`
- `arch-004-system_guide.md`
- `arch-005-recommended_schema_shape.md`

### 02-architecture/analysis
This is where scoring and index math lives.

Important files:
- `anal-001-indices_calculation.md`
- `anal-002-score_calculation.md`

### 02-architecture/design
This is where the **design specs** live.

These are concrete implementation-shape docs.

Use these when you need:
- schema definitions
- pipeline behavior
- analysis rules
- system-specific detailed build rules

Key files right now:
- `ds-005-canonical-route-schema-spec.md`
- `ds-006-route-canonicalization-spec.md`
- `ds-007-route-slice-generation-spec.md`
- `ds-008-route-corridor-model-spec.md`
- `ds-010-slice-analysis-cache-spec.md`
- `ds-013-comparative-traffic-context-schema-spec.md`
- `ds-014-route-expedition-state-and-windowed-analysis-spec.md`

---

## 03-adrs
This is the **decision log**.

If something is a durable architectural decision, it belongs here.

Use ADRs when you want to answer:
- why did we choose this?
- what tradeoff was made?
- what is locked vs still flexible?

Important recent ADRs:
- `adr-026-canonical-route-identity.md`
- `adr-031-model-multi-day-events-as-ordered-references-onto-canonical-geometry.md`
- `adr-032-comparative-traffic-context-and-segment-cohorts.md`
- `adr-033-canonical-segment-identity.md`
- `adr-034-master-route-expeditions-and-windowed-analysis.md`

---

## 04-execution
This is the **what do we actually do next** layer.

This folder matters a lot because it turns theory into action.

### Main execution files
- `exec-001-master_build_order.md` → big build sequence
- `exec-002-architecture_overview.md` → higher-level system overview
- `exec-003-current_focus.md` → what matters right now
- `exec-004-idea_dump.md` → raw idea holding area

### 04-execution/02_system_manuals
These are the operational manuals for major systems.

If you want the “explain it to me clearly” version of a system, go here.

Key files:
- `sys-001-expedition_system.md`
- `sys-002-route_ingestion_system.md`
- `sys-003-analysis_engine.md`
- `sys-005-navigation_engine.md`
- `sys-006-ride_computer.md`
- `sys-007-comparative_traffic.md`
- `sys-009-vault_system.md`

### 04-execution/03_infrastructure_projects
These are the major build tracks.

Use these when the question is:
- what infra project is active?
- what has to be completed to unlock the next layer?

Key files:
- `infra-001-canonical_schema_completion.md`
- `infra-002-rusa_corpus_ingestion.md`
- `infra-003-osm_enrichment_pipeline.md`
- `infra-004-route_analysis_backfill.md`
- `infra-005-canonical_segment_mapper.md`
- `infra-006-traffic_baseline_build.md`

---

## 05-product
This is the product framing layer.

Use this for:
- positioning
- vision
- rider archetypes
- UX guardrails
- anti-features
- brand/product boundaries

Key files:
- `prod-001-positioning.md`
- `prod-002-vision.md`
- `prod-003-rider_archetypes.md`
- `prod-004-guardrails.md`
- `prod-005-ux_principles.md`
- `prod-006-anti_features.md`

---

## assessments
This is where point-in-time architecture reviews live.

Use these to understand:
- risk
- blind spots
- debt
- where the system looked strong or shaky at a given time

Key files:
- `ass-001-architecture_audit_2026-03-08.m_.md`
- `ass-002-architecture_audit_2026-03-24.md`

---

## migrations
This is the human record of important migration and reconciliation work.

This is useful context, but it is not the main architecture home.

---

# 2. What to read first

If you are coming in cold, read in this order:

1. `01-philosophy/phl-002-product_principles.md`
2. `02-architecture/arch-004-system_guide.md`
3. `04-execution/exec-001-master_build_order.md`
4. `04-execution/exec-003-current_focus.md`

That gives you:
- product truth
- system truth
- build order
- present-tense focus

If you still feel lost after that, read:
- `02-architecture/arch-003-project_map.md`
- `04-execution/exec-002-architecture_overview.md`

---

# 3. Where the major open paths live now

This replaces the old “one giant master manual” way of thinking.

The paths still exist.
They are just now properly housed across the repo.

## Path: Expedition / multi-day continuity
What it is:
- durable route progress across days
- active windows
- resume behavior
- later handoff/preload

Read:
- `03-adrs/adr-034-master-route-expeditions-and-windowed-analysis.md`
- `02-architecture/design/ds-014-route-expedition-state-and-windowed-analysis-spec.md`
- `04-execution/02_system_manuals/sys-001-expedition_system.md`

Current truth:
- architecture is good
- first rider-facing pieces exist
- manual validation still matters before declaring trust

---

## Path: Long-route loading / chunked analysis
What it is:
- making giant routes feel like one journey
- only analyzing one bounded working set at a time
- tying window size to real corridor budget, not just miles

Read:
- `03-adrs/adr-034-master-route-expeditions-and-windowed-analysis.md`
- `02-architecture/design/ds-014-route-expedition-state-and-windowed-analysis-spec.md`
- `04-execution/02_system_manuals/sys-001-expedition_system.md`
- `04-execution/02_system_manuals/sys-003-analysis_engine.md`

Current truth:
- the architecture is right
- handoff/preload is not the first thing to build

---

## Path: Canonical segment identity
What it is:
- stable segment identity across routes and analyses
- route-local occurrences vs canonical long-lived segments

Read:
- `03-adrs/adr-033-canonical-segment-identity.md`
- `03-adrs/adr-032-comparative-traffic-context-and-segment-cohorts.md`
- `02-architecture/design/ds-013-comparative-traffic-context-schema-spec.md`
- `04-execution/03_infrastructure_projects/infra-005-canonical_segment_mapper.md`

Current truth:
- the dangerous ambiguity was fixed
- the matcher is intentionally deferred
- do not rush it into the live path

---

## Path: Comparative traffic context / cohorts
What it is:
- the model for canonical facts, baselines, and cohort memberships
- richer context without corrupting the top-line Safety Score

Read:
- `03-adrs/adr-032-comparative-traffic-context-and-segment-cohorts.md`
- `02-architecture/design/ds-013-comparative-traffic-context-schema-spec.md`
- `04-execution/02_system_manuals/sys-007-comparative_traffic.md`
- `04-execution/03_infrastructure_projects/infra-006-traffic_baseline_build.md`

Current truth:
- strong architecture
- still phased
- rider-facing richness should wait for trustworthy facts

---

## Path: Route persistence / DB discipline
What it is:
- how route writes happen
- how staging and production stay sane
- how SQL changes get promoted

Read:
- `02-architecture/arch-005-recommended_schema_shape.md`
- `04-execution/exec-003-current_focus.md`
- relevant infra docs as they are created/updated

Current truth:
- this has improved
- it still needs discipline
- do not let environment confusion creep back in

---

## Path: Core analysis engine
What it is:
- road fetch
- corridor construction
- caching
- scoring compute
- scaling risks

Read:
- `02-architecture/arch-002-system_architecture.md`
- `04-execution/02_system_manuals/sys-003-analysis_engine.md`
- `assessments/ass-002-architecture_audit_2026-03-24.md`

Current truth:
- real strength exists here
- Overpass and client-side compute are still central risks

---

## Path: Navigation / ride mode
What it is:
- actual riding product
- cues, alerts, ride surfaces, bike-computer-ish behavior

Read:
- `03-adrs/adr-029-ride-time-situational-awareness-mode.md`
- `03-adrs/adr-030-ride-mode-power-and-sensor-architecture.md`
- `02-architecture/design/ds-011-ride-time-situational-awareness-interface-spec.md`
- `02-architecture/design/ds-012-ride-computer-tile-system-spec.md`
- `04-execution/02_system_manuals/sys-005-navigation_engine.md`
- `04-execution/02_system_manuals/sys-006-ride_computer.md`

Current truth:
- promising
- must stay rider-trust-first
- do not confuse “exists” with “trustworthy deep into a ride”

---

## Path: Vault / ingestion / route corpus
What it is:
- route acquisition
- provenance
- route corpus growth
- source-aware route storage

Read:
- `03-adrs/adr-001-route-acquisition-model.md`
- `03-adrs/adr-002-vault-concept.md`
- `04-execution/02_system_manuals/sys-002-route_ingestion_system.md`
- `04-execution/02_system_manuals/sys-009-vault_system.md`
- `04-execution/03_infrastructure_projects/infra-002-rusa_corpus_ingestion.md`

Current truth:
- conceptually strong
- should compound over time
- provenance needs to stay explicit

---

## Path: Public expedition visibility / dot watchers
What it is:
- future watcher/family/public follow layer
- public expedition surface
- shared live progress

Read:
- `05-product/prod-002-vision.md`
- `04-execution/exec-004-idea_dump.md`
- later dedicated note/manual when it exists

Current truth:
- real opportunity
- not the thing to operationalize first
- do not let it pollute rider-critical work

---

# 4. What is the actual source of truth?

Use this rule:

## If the question is “what did we decide?”
Read the ADR.

## If the question is “what is the exact shape?”
Read the design spec.

## If the question is “how does this system work in English?”
Read the system manual.

## If the question is “what should happen next?”
Read the execution docs.

## If the question is “why are we even doing this?”
Read philosophy or product docs.

That is the hierarchy.

---

# 5. What is not replaced by this file

This file does **not** replace:
- ADRs
- design specs
- system manuals
- infra project docs
- current focus docs

This file is just the guide rail.

If this file and a real ADR/spec disagree, the ADR/spec wins.

---

# 6. Where to start depending on your mood

## If you want the highest-level truth
Read:
- `01-philosophy/phl-002-product_principles.md`
- `05-product/prod-002-vision.md`

## If you want to know what to build next
Read:
- `04-execution/exec-001-master_build_order.md`
- `04-execution/exec-003-current_focus.md`

## If you want to understand the machine
Read:
- `02-architecture/arch-004-system_guide.md`
- `02-architecture/arch-002-system_architecture.md`

## If you want to understand one specific system
Read the relevant file in:
- `04-execution/02_system_manuals/`

## If you want to understand one specific locked decision
Read the relevant file in:
- `03-adrs/`

---

# 7. Current reality in plain English

The docs are now much better structured than before.

That is good.

But the new risk is not chaos.
The new risk is **fragmentation**.

So the job of this file is to keep you from forgetting:

- where things live
- what to read first
- which path you are actually in
- what is architecture vs execution vs product thinking

That is all this file needs to do.

---

# 8. Short version

The old giant manual is no longer the main home.

The new homes are:

- **ADRs** for decisions
- **Design specs** for exact shapes
- **System manuals** for clear explanation
- **Infra docs** for active build tracks
- **Execution docs** for sequence and focus

Use this file when you need to orient yourself fast and then jump to the right real doc.


---

## Source File: docs/04-execution/exec-001-master_build_order.md


# Master Build Order

This document defines the correct sequence of major development phases.

## Phase 1 — Core Data Foundation

- Canonical schema finalization
- Slice engine implementation
- Segment fact model
- Cohort table scaffolding

## Phase 2 — Route Corpus Ingestion

- Ingest RUSA permanent routes
- Normalize route geometry
- Deduplicate routes
- Attach provenance

## Phase 3 — OSM Enrichment

- Slice-level OSM queries
- Infrastructure tagging
- Hazard detection
- Elevation enrichment

## Phase 4 — Baseline Route Analysis

- Traffic Index
- Bike Support Index
- Safety scoring pipeline

## Phase 5 — Extended Indices

- Remoteness Index
- Surface Quality Index
- Fatigue Index
- Descent Risk Index

## Phase 6 — Environmental Modeling

- Weather forecasts
- Solar calculations
- Moon phase modeling
- Glare detection

## Phase 7 — Route Comparison

- Route comparison UI
- Segment explanation layers

## Phase 8 — Expedition System

- Expedition state storage
- Crash recovery
- Resume functionality

## Phase 9 — Ride Computer

- Bike computer interface
- Cue sheet navigation
- Sensor integrations

## Phase 10 — Intelligence Layer

- Traffic behavior modeling
- Comparative context system
- Decision support tools


---

## Source File: docs/04-execution/exec-002-architecture_overview.md


# Lanterne Architecture Overview

This document visualizes the core architecture of Lanterne.

Lanterne is built around a layered intelligence system for long‑distance cyclists.

```
Route Sources
   │
   ▼
+-------------------+
|  Ingestion Layer  |
|-------------------|
| Route To          |
| Draw              |
| GPX Upload        |
| RWGPS Import      |
| RUSA Import       |
+-------------------+
   │
   ▼
+-------------------+
| Storage Layer     |
|-------------------|
| routes            |
| route_versions    |
| provenance        |
| analysis_outputs  |
+-------------------+
   │
   ▼
+-------------------+
| Analysis Engine   |
|-------------------|
| Slice Generation  |
| Traffic Index     |
| Bike Support      |
| Remoteness        |
| Fatigue           |
| Surface Quality   |
| Descent Risk      |
+-------------------+
   │
   ▼
+-------------------+
| Conditions Layer  |
|-------------------|
| Weather           |
| Wind              |
| Temperature       |
| Solar Position    |
| Moon Phase        |
| Glare Detection   |
+-------------------+
   │
   ▼
+-------------------+
| Presentation      |
|-------------------|
| Map Overlays      |
| Route Analysis    |
| Cue Sheets        |
| Comparison Views  |
+-------------------+
   │
   ▼
+-------------------+
| Decision Support  |
|-------------------|
| Route Comparison  |
| Ride Planning     |
| Expedition Mode   |
| Ride Computer     |
+-------------------+


---

## Source File: docs/04-execution/exec-003-current_focus.md

# Current Focus

## Purpose

Prevent idea thrash, keep development focused, and make it obvious what is active now versus what is intentionally deferred.

------

## Current Priorities

### 1. Canonical schema + route intelligence foundation

Lock the schema and supporting route-intelligence structure tightly enough that ingestion, analysis, and downstream scoring can stop shifting underneath the product.

​	https://chatgpt.com/g/g-p-69a5a40f6d9c81919302f52ccc4cdd32-lanterne/c/69c7cb36-6418-832f-b4aa-63876cd873d0

### 2. Safety Scoring Model Hardening: DS15 & Lovable Prompt

Deep Reserach: https://chatgpt.com/c/69c7cee2-bb58-8330-8cc5-eea6a0d6f96c
CGPT Pro (AFTER RESEARCH IS DONE: 
I want you to act as a senior product + systems reviewer and pressure test my current safety scoring approach for Lanterne.

Context:
Lanterne is a route intelligence system for long-distance cyclists. Its current architecture deliberately keeps the Safety Score narrow and separate from route-reality and conditions layers. That overall philosophy is intentional. The goal is a production-grade score riders can trust, not a giant kitchen-sink score. This aligns with the project’s current principles and analysis model. :contentReference[oaicite:0]{index=0} 

Current scoring direction:
- Safety Score = collision likelihood with motor vehicles + expected injury severity
- Core live scoring is primarily based on speed environment and traffic exposure, with infrastructure mitigation and hazard penalties
- Other route intelligence dimensions like remoteness, fatigue, surface, descent risk, and weather are intentionally separate for now :contentReference[oaicite:2]{index=2} :contentReference[oaicite:3]{index=3}

Current implemented shape:
- speed-related risk
- traffic-related risk / AADT when available
- lane / roadway context
- rail crossing contribution
- bike infrastructure mitigation
- shoulder credit
- left-turn penalty
- selected micro-hazard penalties
- route-level risk-per-mile transformed into a rider-facing score/grade :contentReference[oaicite:4]{index=4} :contentReference[oaicite:5]{index=5} :contentReference[oaicite:6]{index=6}

Attached:
1. Deep Research critique of the model
2. Relevant internal docs / scoring docs / ADRs

Your task:
Take the research critique and my current architecture direction, then recommend the best production-grade scoring approach for Lanterne Phase 0 / Phase 1.

Deliverables:
1. Verdict on whether the current scoring philosophy is fundamentally right or needs correction
2. The 5 biggest weaknesses in the current model
3. The 5 highest-value improvements, ranked
4. A blunt section called:
   “What is likely bullshit or fragile in the current score”
5. A section called:
   “What is defensible enough to ship now”
6. A section called:
   “What should stay out of the Safety Score”
7. A section called:
   “Recommended scoring architecture for Lanterne”
   Include:
   - factor families
   - whether each should be core, modifier, penalty, or separate index
   - how to avoid double-counting
8. A section called:
   “Recommended route rollup strategy”
   Include:
   - how not to wash out short dangerous stretches
   - whether to use weighted mean, percentile emphasis, worst reasonable stretch, etc.
9. A section called:
   “Recommended implementation order”
   Break into:
   - ship now
   - next version
   - later / research only
10. A final section called:
   “If I were locking the production score this month, I would do this”

Rules:
- Do not give me a tutorial version
- Do not give me an academic survey
- Do not suggest a giant complex model unless the gain is worth it
- Assume I want the strongest production approach that is still practical
- Be explicit about tradeoffs
- Call out where my product instincts are right
- Call out where I’m fooling myself
- If the research and the current product direction conflict, resolve that conflict and explain why
- Write like you are trying to save me 6 months of wrong implementation

Output format:
Use these headings exactly:
1. Overall Verdict
2. Biggest Weaknesses
3. Highest-Value Improvements
4. What Is Fragile or Likely Bullshit
5. What Is Defensible Enough to Ship
6. What Should Stay Out of the Safety Score
7. Recommended Scoring Architecture
8. Recommended Route Rollup Strategy
9. Recommended Implementation Order
10. Final Production Recommendation

### 3. Bike Crash research project (future but planning for it while subscriped to CGPT Pro)

​	https://chatgpt.com/c/69c76e27-f64c-8330-9afd-0d5e56a856eb

### 4. Front-end architecture refactor - PAUSED

Move the front end away from page-level state sprawl and toward a cleaner workflow/session model.

Current sub-focus:

- shrink the main page/god component
- separate workflow/lifecycle ownership from rendering
- plan workerization for heavy analysis so the app stays usable during loading
- reduce map monolith / surface sprawl
- make loading/analysis state part of a coherent workflow model rather than one-off UI patches

Reference thread:

- See bottom section of this doc for remaining steps
  	

------

## Active Execution Threads

These are the threads that are allowed to drive current work:

- Canonical schema finalization
- Slice engine implementation
- Front-end architecture refactor
- Expedition system state persistence
- Traffic cohort architecture

------

## Active but Secondary

These matter, but should not steal priority from the current execution threads unless directly required:

- Hazard taxonomy refinement
- Loader / analysis presentation polish
- Route review patterning / explainability
- Comparative traffic context

------

## Explicitly Deferred

Do not start these yet unless they become prerequisites for the current priorities:

- Ride computer advanced features

- Radar integrations

- UI polish outside the current front-end refactor

- LLM / voice assistant systems

- Broader expedition feature expansion beyond persistence/state hardening

- More robust loading sequence pulled from new scoring pipeline

  - Route: Port Orange - Ormond Loop Length: 100km Points: 2534 Fetching corridor from Overpass... Roads: 32986 Detecting junctions... Junction points: 678 / 2534 Crossing highway distribution: {  tertiary: 216,  residential: 524,  secondary: 51,  unclassified: 62,  tertiary_link: 4,  primary: 64,  secondary_link: 5,  primary_link: 3,  cycleway: 11 }

  - with headings

  - **Tracing route geometry**
     *2,534 route points mapped*

    **Fetching surrounding road network**
     *33,129 nearby roads analyzed*

    **Detecting intersections and turns**
     *678 plausible junctions identified*

    **Scoring maneuver complexity**
     *25 left-turn maneuvers, 13 major left turns*

------

## Success Criteria

This focus window is complete when:

- canonical schema is stable enough to stop churn
- slice engine is operational
- RUSA routes can be ingested and analyzed cleanly
- front-end workflow ownership is cleaner and less page-bound
- the app remains usable during route analysis or has a concrete workerization path underway
- map/surface state is moving toward a simpler, more governable structure

------

## Completed

### 2026-03-28

- Created deep research asset re: bike-car crash research framework res-001-bike_car_crash_data_mining.md
- Had Lovable generate a plain english explanation of the Safety Scoring Model ds-015-safety_scoring_model.md
- Confirmed turns are overwritten properly w/o reproducing the canonical route
- Moving on to hardening the canonical route hashing approach for multi-direction / out-and-back / slight variation routes so there isn't route sprawl in the future
- Tried to switch left turn analysis into a new pipeline version but failed to get it as accurate as the old route-level turn scoring so reverted to the old method since it isn't truly required to be known at the segment level at time of ingest and if one wanted to ascribe it to a segment, it could be done based on location after the fact.
- 

### 2026-03-27

- Hardened the route loader to a usable state
- Added route loading cancel behavior and cleaned up the cancel/reopen loop
- Reduced “segments worth reviewing” on the Boulder, CO route from 255 instances into 17 pattern buckets
- Classified those 17 buckets into:
  - 8 protected bike lanes scored risky
  - 8 high-speed road exposure patterns
  - 1 unusual segment break
- Added 6 hazards to the bottom drawer:
  - Bad Angles
  - Traffic
  - Pinch Points
  - Descents
  - No Shoulder
  - Grates
- Documented `prod-012` and `prod-013` and explicitly deferred them

### 2026-03-26

- Implemented new hazard tags
- Added ability to cancel a loading route
- Hardened expedition mode entry logic
- Added admin debug setting to show all metal grated bridges
- Began loader fixes as part of the broader front-end refactor

------

## Working Rules

- Do not let loader polish turn into another isolated UI rabbit hole
- Do not start advanced ride computer work while core schema / analysis / front-end structure are still unstable
- Treat front-end refactor as an architecture thread, not a cosmetics thread
- Only add new active threads if they are true prerequisites or materially block current priorities

# Front-End Work Remaining

## Status

**Completed**

- Phase 0 baseline instrumentation:
  - fixed GPX fixtures
  - smoke test definitions
  - performance markers
  - perf budget logging
  - baseline regression harness

**Not yet approved to start**

- Phase 1
- Phase 2
- Phase 3
- Phase 4

These remain blocked until slice-level scoring and OSM ingestion/enrichment stabilize the analysis contracts.

------

# Remaining Front-End Phases

## Phase 1 — Extract Bounded Hooks + Layout Reducer

Purpose: reduce `Index.tsx` from god-component status without changing behavior.

### Planned hook extractions

- `useRouteAcquisition`
- `useAnalysisSession`
- `useRoutePersistence`
- `usePoiManager`
- `useRideSession`

### Additional Phase 1 work

- introduce `resetRouteSession(reason)` as the single teardown path
- evolve `LayoutContext.tsx` from boolean bag to reducer
- introduce `RouteWorkflowState` separate from `MapMode`

### Phase 1 done criteria

- `Index.tsx` reduced to ~800–1000 lines
- typed public interfaces for all five hooks
- no cross-domain imports between hooks
- no direct state mutation across hook boundaries
- `LayoutContext` reducer replaces standalone drawer/panel booleans
- all Phase 0 smoke flows still pass

------

## Phase 2 — Worker Boundary

Purpose: move heavy matching/scoring compute off the main thread.

### Planned split

- `route-analysis-core.ts` → pure compute only
- `route-analysis-io.ts` → fetch/cache/Supabase/orchestration
- `analysis.worker.ts` → imports only core
- `analysis-protocol.ts` → typed worker messages

### Compute that moves to worker

- road matching
- forensic pipeline
- boundary refinement math
- safety scoring
- transition chain computation
- cue generation

### Work that stays on main thread

- corridor tile fetch
- cache reads/writes
- HPMS/DOT fetches
- railroad crossing detection
- heatmap-building / UI-facing orchestration

### Phase 2 done criteria

- worker serialization within budget on long-route fixture
- zero long tasks over budget during compute phase
- map remains pannable during analysis
- cancel → worker abort → UI reset stays within budget
- stale worker results blocked by `sessionId` guard
- all Phase 0 smoke flows still pass

------

## Phase 3 — RouteMap Decomposition

Purpose: kill the RouteMap monolith and replace prop sprawl with explicit layer boundaries.

### Planned work

- define `MapScene` contract before extraction
- replace 70+ prop sprawl with scene objects
- extract layers in this order:
  1. `useMapCore`
  2. `useRoutePolyline`
  3. `useHeatmapLayer`
  4. `useHazardLayer`
  5. `useCueLayer`
  6. `useGpsLayer`
  7. `usePoiLayer`
  8. `useDebugLayers`

### Phase 3 done criteria

- `RouteMap.tsx` reduced to ~300–400 lines
- each layer handles its own cleanup
- rider-facing hooks contain no admin logic
- click handlers, tooltips, zoom behavior remain unchanged
- layer mount order preserves z-order behavior
- all Phase 0 smoke flows still pass

------

## Phase 4 — Surface Governance + Resilience

Purpose: finish workflow/state cleanup after the structural refactor.

### Planned work

- wire `RouteWorkflowState` into visibility rules
- audit remaining rogue surface toggles
- add `failed` and `partial` workflow states
- add stale reopen / re-analyze recovery behavior

### Phase 4 done criteria

- zero standalone drawer/panel booleans left in `Index.tsx`
- `failed` state shows retry UI
- `partial` state shows warning + usable partial results
- stale reopen detection works
- all Phase 0 smoke flows still pass

------

## Key Front-End Risks

- Phase 1 is the highest-risk extraction because callbacks in `Index.tsx` cross multiple domains.
- `usePoiManager` has hidden coupling to route and map-bounds state and must use explicit parameters.
- worker serialization could become expensive on long routes.
- RouteMap extraction must preserve explicit Leaflet z-order.
- Layout reducer migration touches many callback props that currently toggle booleans directly.

------

## Current Decision

Front-end work is limited to **Phase 0 only** until:

- slice-level scoring model is stable
- OSM ingestion/enrichment contracts are stable
- route-analysis input/output contracts are frozen

After that, execute front-end Phases 1 → 4 in order.


---

## Source File: docs/04-execution/exec-004-idea_dump.md

# Idea Dump For Future Improvements

You can't focus on everything all at once, and not everything needs to be decided today. Let this be a holding place for big ideas so you can focus on making progress with the basics.

------

## Expedition Mode Could Become Product #1

- import past activities from Strava/Garmin
- stitch together entire multi-week/month/year journey
- watchers can follow, athletes can use as an explorers journal/log, field notes style
- would apply to other sports/activities too (ultra-running/walking/kayaking/sailing/etc where long distances are traversed of a long period of time)

------

## iOS/Android App

- Build an app for native bluetooth, would need a companion app anyway to get BT reliably into the Safari based app.  Janky solution to bookmark a PNW + install an app that has to go through AppStore anyway. 
- Opens up other possibilities beyond BT: battery mgmt, haptics (lantern wheel, etc)
- Can be minimal coding with a wrapper around the React framework and can still use Lovable 
- Can avoid re-approvals from apple if most logic is kept in web part of the app
- need a couple screens (perhaps load screen) to be iOS native (plus making use of hardware features of phone) so that Apple approves initial application
- CGPT thread context: https://chatgpt.com/s/t_69c33b5ca9ac8191a08b6918ef0c1746

- 

------

## Icons Improvement

- need to fit better with Lanterne theme
- thin lines if visible enough
- See Phosphor Thin from streamlinehq.com

##### Link Tree

- where to?

  - route to...
  - draw
  - load
    - rwgps 
    - vault
      - rusa perms
      - saved
      - (future expansion)
    - .gpx
    - history

- analyze

- cues

- dev (admin only)

- inspect (admin/superuser only)

- stops & layers

  - Stops (with gear icon to configure toggles)
  - water
    - potable water
    - natural water crossings
    - cemetaries
  - food
    - restaurants
    - supermarkets
    - Cafes/fastfood
    - convenience stores
    - gas stations
  -  bio (toilet)
    - toilets
    - Showers
  - rest
    - hotels
    - hostels
    - campgrounds
    - post offices
  - tourism
    - monuments
    - ruins
    - memorials
    - attractions
    - Info offices
  - nature
    - peaks
    - springs
    - caves
    - viewpoints
  - health
    - hospitals
    - clinic
    - pharmacies
  - help
    - fire stations
    - police stations
    - emergency phones

- map buttons

  - current bearing - pointer
  - heatmap on/off - Light bulb
  - day night - sun & crescent moon
  - OSM street and satlleite - folded map
  - Gps - crosshairs

- drawing map buttons

  - snap on button
  - '# points and miles counter button
  - need elevation counter button + icon
  - cancel button

- edit route button

- admin/superuser map buttons

  - edit route
  - show roads
  - anomalies
  - inspector on (should probably kill this and just always have inspector on for these user types)
  - truth mode (admin only)
  - hold to peek (eyeball)

- hazards

  - RR Crossing

  - RR Crossing - Dangerous Angle?

  - Left Turn (Crosses Traffic)

  - Metal Grate Bridge

  - Future Possibilities:

    - Pinch Point
    - Remote Corridor
    - Service Gap
    - Cellular Dead Zone
    - etc.

    

  

  


---

## Source File: docs/04-execution/exec-005-debugging_logs.md

# Debugging Logs

### Hazards

Enable with: localStorage.DEBUG_FLAGS = '{"HAZARD_DEBUG":true}' then reload and re-analyze.

**Console logging** (when HAZARD_DEBUG=true):

- [HAZARD-TRACE] now includes OSM node ID, nearest route coord, snap distance (meters), and route distance (miles) for both accepted AND rejected crossings
- [HAZARD-RENDER] shows raw → attached → rendered coordinates with deltas

**Map debug overlay** (when HAZARD_DEBUG=true):

- **Cyan circles** = raw OSM detection point (with permanent label showing OSM ID + coords)
- **Yellow circles** = nearest attached route point + yellow dashed line connecting raw→attached
- **Magenta circles** = rendered marker point (currently = raw, confirming no remap)
- **Red circles** = REJECTED crossings with tooltip showing rejection reason + snap distance, plus red dashed line to nearest route point



---

## Source File: docs/04-execution/exec-006-phase0-smoke-tests.md

# Phase 0 — Smoke Tests & Performance Baselines

## Fixtures

| Fixture | File | Purpose |
|---------|------|---------|
| Short urban hazard | `public/demo/fixture-urban-hazard.gpx` | ~15mi KC MO, dense crossings/bridges, stress hazard detection |
| Long rural | `public/demo/fixture-long-rural.gpx` | ~200mi KS Flint Hills, worker serialization budget, guardrails |
| History-loaded | `public/demo/fixture-history-loaded.gpx` | ~40mi Lawrence KS loop, rehydration/cache hit/re-analyze |
| Detour-edit | `public/demo/fixture-detour-edit.gpx` | ~25mi Topeka out-and-back, detour drag/save/delta-panel |

## Smoke Test Paths

### 1. GPX Upload → Analyze → Save
1. Upload `fixture-urban-hazard.gpx`
2. Wait for analysis to complete (score panel renders)
3. Open left drawer, verify score + grade visible
4. Save to history
5. **Pass:** Route appears in history list with correct name

### 2. Manual Create → Analyze → Save
1. Enter route create mode
2. Place 3+ waypoints (~5mi route)
3. Finish drawing → analysis begins
4. Wait for completion
5. Save to history
6. **Pass:** Route saved, can be re-loaded

### 3. History Load → Re-analyze
1. Load `fixture-history-loaded.gpx`, analyze, save
2. Open history, click the saved route
3. Verify polyline renders on map
4. Trigger re-analyze
5. **Pass:** New score renders, no ghost state from previous analysis

### 4. Detour Save
1. Upload `fixture-detour-edit.gpx`, analyze
2. Click on the high-traffic middle segment
3. Drag a detour waypoint to create alternate route
4. Verify delta panel shows score comparison
5. Save detour
6. **Pass:** Detour saved to history, delta panel values correct

## Hard Performance Budgets

| Metric | Budget | Module |
|--------|--------|--------|
| Cancel acknowledged | ≤ 300ms | `analysis:cancel:latency` |
| Worker serialization (long-route) | ≤ 80ms | `worker:serialization` |
| Main-thread long tasks during compute | 0 tasks > 100ms | `PerformanceObserver('longtask')` |
| Render-to-interactive after done | ≤ 500ms | `render:to-interactive` |

Budget violations emit `[PERF-BUDGET] ⚠` warnings to console.

## Instrumentation Points

All marks defined in `src/lib/refactor-perf-budgets.ts`:
- `markAnalysisStart()` — called when `analyzeRouteProgressive` loop begins
- `markAnalysisDone()` — called on `analysis` stage received
- `markAnalysisCancelRequest()` — called when cancel button tapped
- `markAnalysisCancelAck()` — called when UI teardown complete
- `markRenderToInteractiveStart()` — called at analysis done phase
- `markRenderToInteractiveEnd()` — called on first drawer open after analysis
- `startLongTaskObserver()` / `stopLongTaskObserver()` — bracketing analysis phase

## Done Criteria

- [ ] 4 fixture files committed to `public/demo/`
- [ ] 4 smoke paths documented (this file) and manually passing
- [ ] Perf marks emitting to console during analysis
- [ ] Budget violations logged as warnings
- [ ] **No code moved or restructured** — this is baseline only


---

## Source File: docs/04-execution/exec-007-turn_event_persistence_handoff.md

# Current Focus

## Purpose

Prevent idea thrash, keep development focused, and make it obvious what is active now versus what is intentionally deferred.

------

## Current Priorities

### 1. Canonical schema + route intelligence foundation

Lock the schema and supporting route-intelligence structure tightly enough that ingestion, analysis, and downstream scoring can stop shifting underneath the product.

### 2. Front-end architecture refactor

Move the front end away from page-level state sprawl and toward a cleaner workflow/session model.

Current sub-focus:

- shrink the main page/god component
- separate workflow/lifecycle ownership from rendering
- plan workerization for heavy analysis so the app stays usable during loading
- reduce map monolith / surface sprawl
- make loading/analysis state part of a coherent workflow model rather than one-off UI patches

Reference thread:

- See bottom section of this doc for remaining steps
  	

------

## Active Execution Threads

These are the threads that are allowed to drive current work:

- Canonical schema finalization
- Slice engine implementation
- Front-end architecture refactor
- Expedition system state persistence
- Traffic cohort architecture

------

## Active but Secondary

These matter, but should not steal priority from the current execution threads unless directly required:

- Hazard taxonomy refinement
- Loader / analysis presentation polish
- Route review patterning / explainability
- Comparative traffic context

------

## Explicitly Deferred

Do not start these yet unless they become prerequisites for the current priorities:

- Ride computer advanced features

- Radar integrations

- UI polish outside the current front-end refactor

- LLM / voice assistant systems

- Broader expedition feature expansion beyond persistence/state hardening

- More robust loading sequence pulled from new scoring pipeline

  - Route: Port Orange - Ormond Loop Length: 100km Points: 2534 Fetching corridor from Overpass... Roads: 32986 Detecting junctions... Junction points: 678 / 2534 Crossing highway distribution: {  tertiary: 216,  residential: 524,  secondary: 51,  unclassified: 62,  tertiary_link: 4,  primary: 64,  secondary_link: 5,  primary_link: 3,  cycleway: 11 }

  - with headings

  - **Tracing route geometry**
     *2,534 route points mapped*

    **Fetching surrounding road network**
     *33,129 nearby roads analyzed*

    **Detecting intersections and turns**
     *678 plausible junctions identified*

    **Scoring maneuver complexity**
     *25 left-turn maneuvers, 13 major left turns*

------

## Success Criteria

This focus window is complete when:

- canonical schema is stable enough to stop churn
- slice engine is operational
- RUSA routes can be ingested and analyzed cleanly
- front-end workflow ownership is cleaner and less page-bound
- the app remains usable during route analysis or has a concrete workerization path underway
- map/surface state is moving toward a simpler, more governable structure

------

## Completed

### 2026-03-28

- 

### 2026-03-27

- Hardened the route loader to a usable state
- Added route loading cancel behavior and cleaned up the cancel/reopen loop
- Reduced “segments worth reviewing” on the Boulder, CO route from 255 instances into 17 pattern buckets
- Classified those 17 buckets into:
  - 8 protected bike lanes scored risky
  - 8 high-speed road exposure patterns
  - 1 unusual segment break
- Added 6 hazards to the bottom drawer:
  - Bad Angles
  - Traffic
  - Pinch Points
  - Descents
  - No Shoulder
  - Grates
- Documented `prod-012` and `prod-013` and explicitly deferred them

### 2026-03-26

- Implemented new hazard tags
- Added ability to cancel a loading route
- Hardened expedition mode entry logic
- Added admin debug setting to show all metal grated bridges
- Began loader fixes as part of the broader front-end refactor

------

## Working Rules

- Do not let loader polish turn into another isolated UI rabbit hole
- Do not start advanced ride computer work while core schema / analysis / front-end structure are still unstable
- Treat front-end refactor as an architecture thread, not a cosmetics thread
- Only add new active threads if they are true prerequisites or materially block current priorities

# Front-End Work Remaining

## Status

**Completed**

- Phase 0 baseline instrumentation:
  - fixed GPX fixtures
  - smoke test definitions
  - performance markers
  - perf budget logging
  - baseline regression harness

**Not yet approved to start**

- Phase 1
- Phase 2
- Phase 3
- Phase 4

These remain blocked until slice-level scoring and OSM ingestion/enrichment stabilize the analysis contracts.

------

# Remaining Front-End Phases

## Phase 1 — Extract Bounded Hooks + Layout Reducer

Purpose: reduce `Index.tsx` from god-component status without changing behavior.

### Planned hook extractions

- `useRouteAcquisition`
- `useAnalysisSession`
- `useRoutePersistence`
- `usePoiManager`
- `useRideSession`

### Additional Phase 1 work

- introduce `resetRouteSession(reason)` as the single teardown path
- evolve `LayoutContext.tsx` from boolean bag to reducer
- introduce `RouteWorkflowState` separate from `MapMode`

### Phase 1 done criteria

- `Index.tsx` reduced to ~800–1000 lines
- typed public interfaces for all five hooks
- no cross-domain imports between hooks
- no direct state mutation across hook boundaries
- `LayoutContext` reducer replaces standalone drawer/panel booleans
- all Phase 0 smoke flows still pass

------

## Phase 2 — Worker Boundary

Purpose: move heavy matching/scoring compute off the main thread.

### Planned split

- `route-analysis-core.ts` → pure compute only
- `route-analysis-io.ts` → fetch/cache/Supabase/orchestration
- `analysis.worker.ts` → imports only core
- `analysis-protocol.ts` → typed worker messages

### Compute that moves to worker

- road matching
- forensic pipeline
- boundary refinement math
- safety scoring
- transition chain computation
- cue generation

### Work that stays on main thread

- corridor tile fetch
- cache reads/writes
- HPMS/DOT fetches
- railroad crossing detection
- heatmap-building / UI-facing orchestration

### Phase 2 done criteria

- worker serialization within budget on long-route fixture
- zero long tasks over budget during compute phase
- map remains pannable during analysis
- cancel → worker abort → UI reset stays within budget
- stale worker results blocked by `sessionId` guard
- all Phase 0 smoke flows still pass

------

## Phase 3 — RouteMap Decomposition

Purpose: kill the RouteMap monolith and replace prop sprawl with explicit layer boundaries.

### Planned work

- define `MapScene` contract before extraction
- replace 70+ prop sprawl with scene objects
- extract layers in this order:
  1. `useMapCore`
  2. `useRoutePolyline`
  3. `useHeatmapLayer`
  4. `useHazardLayer`
  5. `useCueLayer`
  6. `useGpsLayer`
  7. `usePoiLayer`
  8. `useDebugLayers`

### Phase 3 done criteria

- `RouteMap.tsx` reduced to ~300–400 lines
- each layer handles its own cleanup
- rider-facing hooks contain no admin logic
- click handlers, tooltips, zoom behavior remain unchanged
- layer mount order preserves z-order behavior
- all Phase 0 smoke flows still pass

------

## Phase 4 — Surface Governance + Resilience

Purpose: finish workflow/state cleanup after the structural refactor.

### Planned work

- wire `RouteWorkflowState` into visibility rules
- audit remaining rogue surface toggles
- add `failed` and `partial` workflow states
- add stale reopen / re-analyze recovery behavior

### Phase 4 done criteria

- zero standalone drawer/panel booleans left in `Index.tsx`
- `failed` state shows retry UI
- `partial` state shows warning + usable partial results
- stale reopen detection works
- all Phase 0 smoke flows still pass

------

## Key Front-End Risks

- Phase 1 is the highest-risk extraction because callbacks in `Index.tsx` cross multiple domains.
- `usePoiManager` has hidden coupling to route and map-bounds state and must use explicit parameters.
- worker serialization could become expensive on long routes.
- RouteMap extraction must preserve explicit Leaflet z-order.
- Layout reducer migration touches many callback props that currently toggle booleans directly.

------

## Current Decision

Front-end work is limited to **Phase 0 only** until:

- slice-level scoring model is stable
- OSM ingestion/enrichment contracts are stable
- route-analysis input/output contracts are frozen

After that, execute front-end Phases 1 → 4 in order.


---

## Source File: docs/04-execution/01_system_manuals/sys-001-expedition_system.md

# System Manual — Expedition System

## Purpose

The expedition system makes Lanterne work for long routes that span hours, days, sleep stops, browser restarts, and phone interruptions.

It is the system that turns Lanterne from a pre-ride analysis tool into a durable companion for ultra-distance riding.

## What this system owns

- expedition creation
- durable progress checkpoints
- analysis windowing for very large routes
- resume behavior on reopen
- mismatch handling between stored progress and current GPS

## What is already done

- expedition tables exist
- create / pause / resume plumbing exists
- checkpoint cadence is defined as **2 miles and 10 minutes**
- windowed mode exists conceptually and in schema
- resume detection on app boot exists in Phase 1 form
- mismatch card behavior exists in shipped or near-shipped form

## What is not done yet

- richer mismatch handling edge cases
- join-at-current-location logic
- better preload / queue behavior for next windows
- seam transitions between windows
- overnight heuristics and resume polish
- more resilient debugging / admin tooling

## Correct build order

1. Keep expedition durability authoritative.
2. Make resume reliable.
3. Improve window transitions.
4. Add comfort and recovery UX later.

## How the system should work

### Layer model

For ultra routes, keep this four-layer model intact:

- master route = permanent journey identity
- expedition = durable rider progress
- active analysis window = bounded working set
- live session = transient runtime state

The rider experiences one route and one expedition.
The system manages the windows quietly underneath.

## Build order

### Step 1 — Treat the database expedition row as the source of truth

The authoritative answer to “where am I in this big ride?” lives in `route_expeditions`, not in browser memory.

That means:

- live session state can be rebuilt from expedition state
- expedition state must survive closes, crashes, and charging stops

### Step 2 — Keep checkpointing sparse and intentional

Do not turn this into a GPS logger.

Keep checkpoints sparse:

- rider moved at least 2 miles
- at least 10 minutes since last checkpoint

That is enough to recover progress without bloating the table.

### Step 3 — Make resume behavior boring and reliable

On app boot:

- check for active/paused expeditions once per session
- if none exist, do nothing
- if one exists, show resume affordance cleanly

If current GPS is close to the last checkpoint, give the rider the simple path.
If it is not, show explicit choices and do not guess aggressively.

### Step 4 — Improve mismatch handling before adding cleverness

The order matters.

First make these states trustworthy:

- resume from last matched point
- join at current location
- start over
- keep expedition but do nothing now

Do not add clever auto-healing until these four paths are solid.

### Step 5 — Queue windows before the rider needs them

For long routes, next-window readiness matters.

Build toward:

- current active window
- next window queued before the seam
- enough overlap to avoid ugly route context loss

Window logic should feel invisible to the rider.

### Step 6 — Keep writes fire-and-forget but observable

Expedition writes should not block riding UX.
That part is correct.

But failures should still be inspectable later through logs or admin tooling.

## Do not skip

- the durability split between expedition and live session
- sparse checkpoint design
- explicit mismatch choices
- overlap-aware windowing

## Do not touch yet

- chatty AI riding companion behavior
- social expedition sharing
- threaded expedition commentary
- detailed session analytics that are really just ride recording in disguise

## Definition of done

This system is healthy when:

- riders can reopen after interruption and continue confidently
- the wrong part of the route does not load by accident
- window seams do not feel like route fragmentation
- expedition durability survives the messy realities of long-distance riding


---

## Source File: docs/04-execution/01_system_manuals/sys-002-route_ingestion_system.md

# System Manual — Route Ingestion System

## Purpose

The route ingestion system governs how routes enter Lanterne and become usable route records.

It is responsible for turning many entry paths into one clean internal model.

## What this system owns

- Route To / Draw / Open acquisition model
- GPX import
- RWGPS import
- RUSA route import
- ride-history re-open
- canonical resolution
- provenance preservation

## What is already done

- Lanterne already recognizes multiple ingress paths.
- The route acquisition model is conceptually clear: Route To, Draw, Open.
- Canonical route identity and imported provenance tables exist.
- RWGPS proxy and harvester now exist.
- GPX upload and manual route creation already exist operationally.

## What is not done yet

- ingestion flows are not yet fully unified around the same canonical persistence contract
- all paths do not yet share identical post-ingestion behavior
- provenance handling and user-save handling still need disciplined convergence
- large corpus imports still need route identity lock-down before scale

## Correct build order

1. Normalize every entry path.
2. Resolve canonical identity.
3. Preserve provenance.
4. Register the user's relationship to the route.
5. Kick off analysis only after the route record is clean.

## How the system should work

### The rider-facing mental model

The UI should think in terms of:

- Route To
- Draw
- Open

Where Open includes things like:

- Vault
- RWGPS
- GPX
- History

That is the correct mental model because it describes how route geometry appears, not where product politics wants it to live.

## Build order

### Step 1 — Normalize geometry first

Regardless of source, the first internal job is the same:

- clean geometry
- consistent coordinate handling
- basic route metrics
- fingerprint generation

Do not let each source invent its own geometry rules.

### Step 2 — Resolve to canonical identity

After normalization:

- try to match an existing canonical route
- create a new canonical route only if needed

The rule is simple:

source does not define route identity.
The road corridor does.

### Step 3 — Preserve provenance separately

Store where the route came from in import/provenance records.
Do not jam source metadata into the canonical route and call it done.

### Step 4 — Register the user relationship separately

A rider saving or opening a route is a separate concern from route identity.
That should remain true in code.

Examples of user-layer details:

- custom name
- uploaded_at
- saved relationship
- personal history semantics

### Step 5 — Only then move into analysis or expedition creation

Do not start long-running work until route identity and provenance are sane.

Ingestion should produce a clean route record first.
Analysis is the next step, not part of the same conceptual layer.

## Do not skip

- one normalization contract for all inputs
- canonical resolution before analysis
- provenance separation
- user relationship separation

## Do not touch yet

- blending Vault curation rules into raw ingestion
- field notes / community commentary on imported routes
- elaborate route-family social features

## Definition of done

This system is healthy when:

- all entry paths converge to the same internal model
- duplicate routes are minimized by canonical resolution
- provenance is preserved cleanly
- the rider can open the same route from multiple sources without creating architecture garbage


---

## Source File: docs/04-execution/01_system_manuals/sys-003-analysis_engine.md

# System Manual — Analysis Engine

## Purpose

The analysis engine turns route geometry and route facts into route intelligence.

It is the core mechanical brain of Lanterne.

## What this system owns

- corridor acquisition
- route matching
- boundary refinement
- stable index computation
- safety scoring
- cue generation
- analysis warnings / guardrails
- worker isolation over time

## What is already done

- client-side analysis is operational
- chunk-corridor fetch dramatically reduced Overpass pressure
- matching and forensic re-analysis exist
- safety scoring exists
- cue generation exists
- hard guardrails now exist
- density-based advisory warning exists

## What is not done yet

- analysis still primarily runs on the main client runtime path
- worker isolation is not formalized enough
- compute-once architecture is only partially in place
- route hash still needs direction normalization
- tile cache read path still needs batching improvements
- partial/match metadata persistence needs finishing

## Correct build order

1. Keep the current engine reliable.
2. Isolate heavy compute from the UI thread.
3. Finish cache hygiene.
4. Move repeatable truth into stored artifacts over time.

## The non-negotiable rule

Compute on slices.
Present on human-readable sections.
Do not backslide into giant averaged segments.

## Build order

### Step 1 — Keep current output behavior trustworthy

Before chasing major runtime changes, preserve what already works:

- chunk corridor
- matching
- guardrails
- scoring
- cue generation

If you break these while chasing architecture purity, you lose the product.

### Step 2 — Put heavy compute behind a worker boundary

This is a real execution priority.

The UI thread should own:

- rendering
- map interaction
- drawers and controls
- progress display

The analysis worker should own:

- GPX parsing
- spatial indexing
- corridor processing
- matching
- forensic passes
- scoring
- cue generation

Do not half-migrate this and leave two permanent codepaths. That way lies hell.

### Step 3 — Define worker protocol cleanly

Before moving logic:

- define request payloads
- define progress events
- define cancellation
- define timeout and partial-result behavior
- define error surfaces

The worker should never directly own React state.

### Step 4 — Finish cache hygiene

High-value follow-ups:

- direction-independent route hash
- loop-safe hash behavior
- batched tile cache reads
- persist partial + match quality metadata
- stale-while-revalidate tile strategy later

These are not sexy, but they improve performance and correctness materially.

### Step 5 — Gradually shift repeatable truth into stored artifacts

Long term, the direction is:

- server/pipeline computes canonical route facts and stored analysis artifacts
- client composes experience from those artifacts

But do not rush full server-side scoring just because it sounds architecturally pure.

## Do not skip

- worker isolation planning
- guardrails
- partial-result behavior
- cache hygiene
- parity checks if runtime boundaries change

## Do not touch yet

- moving everything server-side prematurely
- giant model rewrites while workerization is unfinished
- adding complex new indices before runtime architecture is stable

## Definition of done

This system is healthy when:

- the app stays responsive during heavy analysis
- output quality remains believable
- cache reuse is strong
- stored artifacts and runtime computation are moving toward a coherent hybrid model instead of a pile of one-off hacks


---

## Source File: docs/04-execution/01_system_manuals/sys-004-environmental_modeling.md

# System Manual — Environmental Modeling

## Purpose

Environmental modeling tells the rider what the route will feel like at a specific time.

This system exists to model **conditions**, not stable route truth.

## What this system owns

- ride-time timeline construction
- wind
- temperature
- precipitation
- light state
- sun glare
- moon phase / moonlit context

## What is already done

- the architecture cleanly distinguishes stable vs contextual analysis
- Scenario Context exists and is the right foundation
- sun/moon concepts are core to the product identity
- light and sky signals are already defined conceptually

## What is not done yet

- `ride_instance_runs` does not exist in the live schema
- `ride_instance_slice_conditions` does not exist in the live schema
- environmental results are not yet fully written as a first-class ride-instance layer
- the product still needs a full arrival-time-per-slice pipeline in stored form

## Correct build order

1. Build the ride instance container.
2. Build the timeline model.
3. Compute conditions per slice.
4. Surface them quietly in UI.

## The non-negotiable rule

Weather and light do **not** belong in the stable Safety Score.
A bad road is still a bad road in sunshine.
Conditions answer a different question.

## Build order

### Step 1 — Create the ride instance layer

The system needs a row representing:

- this route
- this rider plan
- this start time
- this pace assumption / scenario

That is `ride_instance_runs`.
Without it, conditions stay informal.

### Step 2 — Build arrival-time-per-slice logic

Every slice needs an estimated arrival time.
That is the backbone for:

- forecast lookup
- light state
- glare detection
- moon context
- time-dependent traffic multipliers

Do not skip this.
Conditions without timing are just vibes.

### Step 3 — Compute slice-level condition outputs

Write per-slice condition rows for:

- wind
- temperature
- precipitation
- light state
- glare flag
- sun azimuth if needed
- moon phase / moon context

These rows belong in `ride_instance_slice_conditions`, not in stable analysis tables.

### Step 4 — Keep wind bearing-relative

Wind should answer the rider's question, not a weather nerd's question.

That means the key output is not just compass wind direction.
The key output is wind relative to route bearing:

- headwind-ish
- tailwind-ish
- crosswind-ish

### Step 5 — Keep glare and moon systems emotionally legible

This is part of what makes Lanterne special.

Do not turn sun/moon into a giant scientific dashboard.
The UI should make riders feel:

- this part will be dark
- this part may be moonlit
- this dawn section may create glare problems

## Do not skip

- ride-instance container rows
- arrival-time-per-slice logic
- separate storage for contextual outputs
- bearing-relative wind
- emotionally legible light modeling

## Do not touch yet

- mixing weather into safety score
- hyper-detailed meteorological dashboards
- historical/replay weather work that distracts from forward decision support

## Definition of done

This system is healthy when:

- a rider can choose a route and start time and see what conditions they will hit along the route
- those conditions are computed separately from stable route truth
- sun/moon behavior feels useful instead of gimmicky


---

## Source File: docs/04-execution/01_system_manuals/sys-005-navigation_engine.md

# System Manual — Navigation Engine

## Purpose

The navigation engine is the ride-time system that helps the rider stay oriented on the chosen route without turning Lanterne into a noisy gadget circus.

## What this system owns

- cue consumption
- GPS matching during rides
- off-route detection
- route progress during a ride
- route rejoin logic
- interaction with expedition windows

## What is already done

- cue generation exists
- GPS tracking exists
- GPS look-ahead exists
- a navigation engine exists conceptually and in code
- ride mode architecture has started taking shape

## What is not done yet

- full rider-trustworthy ride-time behavior is not finished
- rejoin and off-route behaviors need tightening
- cue timing and dismissal rules need polish
- expedition window seam behavior during active navigation needs more work
- power and sensor interactions are still evolving

## Correct build order

1. Make route progress trustworthy.
2. Make cue delivery predictable.
3. Make off-route / rejoin behavior sane.
4. Add richer ride-mode behavior later.

## Build order

### Step 1 — Treat route progress as a navigation primitive

The system must know, with enough confidence:

- where on the route the rider is
- what comes next
- what was already passed

If this is noisy or unstable, everything downstream suffers.

### Step 2 — Use cues as structured outputs, not decorative UI

Cue generation already exists.
The ride engine now needs to use cues consistently.

Define clearly:

- when a cue becomes upcoming
- when it becomes active
- when it is considered passed
- when it should be dismissed

### Step 3 — Make off-route behavior calm and explicit

When a rider drifts off-route, the system should not panic.

It should be able to distinguish:

- brief GPS wobble
- actual route miss
- intentional detour / store stop
- meaningful divergence

A bike computer replacement cannot feel hysterical.

### Step 4 — Define rejoin behavior before fancy rerouting

Before smart rerouting, make these states boring and reliable:

- continue on route
- rejoin nearby route section
- hold current location and wait

Do not ship aggressive auto-rerouting that feels wrong.

### Step 5 — Respect expedition windows

On long routes, active navigation will occur inside an expedition window.
That means navigation progress must not be allowed to corrupt expedition durability.

The rules for:

- route mile
- point index
- active window index
- seam handoff

must stay consistent.

## Do not skip

- route progress confidence
- cue state rules
- calm off-route handling
- seam-safe expedition interaction

## Do not touch yet

- full “safe-ish reroute everywhere” behavior before route progress is trustworthy
- chatty spoken coaching
- giant training-metric overlays in navigation mode

## Definition of done

This system is healthy when:

- riders can actually follow a route with it
- off-route moments do not create chaos
- the engine behaves predictably across long-route windows
- cues feel trustworthy enough to support real riding, not just demoing


---

## Source File: docs/04-execution/01_system_manuals/sys-006-ride_computer.md

# System Manual — Ride Computer

## Purpose

The ride computer system is Lanterne's on-bike experience layer.

It should feel glanceable, calm, and useful enough that a rider could plausibly choose it instead of a dedicated cycling computer in some scenarios.

## What this system owns

- metric layout
- ride screen composition
- on-bike overlays
- glanceability rules
- eventual sensor/radar integration path

## What is already done

- ride computer metric registry exists
- configurable slot layout exists
- core metrics such as speed, elapsed time, distance, and progress are represented
- ride overlays exist
- the product has already started differentiating between planning mode and ride mode

## What is not done yet

- full on-bike interaction model is not finished
- keep-awake / power realities still need deliberate handling
- sensor strategy is not fully settled
- radar integration path is not complete
- the product still needs real-world riding polish before this can be trusted as a primary bike computer

## Correct build order

1. Nail the core glanceable screen.
2. Nail route progress + cues on that screen.
3. Add basic sensor support.
4. Treat radar/native work as a later layer.

## Build order

### Step 1 — Keep the first ride computer screen minimal

The rider should be able to glance and get:

- where am I?
- what is next?
- how far have I gone?
- how much route is left?
- what metric matters right now?

Do not build a cockpit.

### Step 2 — Make metric slots declarative and stable

The metric registry approach is correct.
Keep it that way.

A ride screen should be composed from:

- small set of trusted metrics
- consistent formatting
- strong unit handling

### Step 3 — Put route progress and cues at the center

This cannot be just a speed dashboard.
Lanterne's advantage is route intelligence.

The ride computer should emphasize:

- cue timing
- route progress
- nearby hazard/support context when appropriate

### Step 4 — Split sensor work into phases

Phase 1:

- browser-friendly BLE sensors where realistic
- heart rate / power / speed / cadence where practical

Phase 2:

- deeper native bridge work only if the core ride computer is already worth it

Do not let hardware ambition outrun product usefulness.

### Step 5 — Treat radar carefully

Radar matters, but it is not the first build step.

The system should first become a trustworthy ride interface.
Then evaluate:

- browser limits
- native wrapper or bridge needs
- whether radar support is worth the complexity in the current phase

## Do not skip

- glanceability
- route-first screen design
- clear sensor phasing
- realistic treatment of PWA limitations

## Do not touch yet

- giant metrics wall
- fully native bridge work before the web ride screen is proven
- vanity customization that hurts legibility

## Definition of done

This system is healthy when:

- the ride screen is calm and useful at a glance
- route progress and cues are the core experience
- basic metrics feel solid
- future sensor/native expansion has a clean path instead of a pile of hacks


---

## Source File: docs/04-execution/01_system_manuals/sys-007-comparative_traffic.md

# System Manual — Comparative Traffic

## Purpose

The comparative traffic system adds a new layer of meaning on top of absolute safety.

It exists to help Lanterne answer:

- how unusual is this road compared with similar roads?
- what traffic behavior is typical here?
- what cohort lenses apply to this segment?

## What this system owns

- traffic behavior dimensions
- canonical segment facts for behavior inputs
- comparative baselines
- cohort memberships
- future explanation-layer comparisons

## What is already done

- the architecture for comparative traffic is defined
- schema scaffolding exists
- the system distinguishes exposure, intensity, and accommodation conceptually
- naming discipline for observed / inferred / predicted / baseline fields is defined

## What is not done yet

- canonical segment mapping is incomplete
- segment behavior inputs are not populated at scale
- baseline population is sparse
- cohort memberships are largely unpopulated
- rider-facing comparison UI should still be considered deferred

## Correct build order

1. Canonicalize segments.
2. Populate behavior inputs.
3. Populate minimum baselines.
4. Attach cohort memberships.
5. Add explanation UX later.

## Non-negotiable rule

The headline Safety Score stays absolute.
Comparative traffic context must never quietly rescale it.

## Build order

### Step 1 — Keep the three behavior dimensions distinct

Do not collapse traffic behavior into one mushy number.

Keep separate:

- exposure = how often vehicles interact
- intensity = how forceful / fast those interactions are
- accommodation = whether drivers slow and give space

These can later combine into composites, but they are not the same thing.

### Step 2 — Respect evidence precedence

When data conflicts, keep this order:

1. observed
2. inferred
3. predicted
4. baseline

Do not let a weak prior silently overwrite stronger evidence.

### Step 3 — Keep cohort membership many-to-many

A segment can belong to many lenses at once.
That is the right model.

Examples:

- state
- metro
- road class
- urbanicity
- event ecosystem later

Do not force a single classification tree just because it seems simpler.

### Step 4 — Populate the minimum v1 context first

Minimum v1 should focus on:

- geography
- road class
- urbanicity

This is enough to support the early comparison layer without exploding scope.

### Step 5 — Keep rider-facing comparisons disciplined

When the data is finally good enough, comparison language should be simple.

Examples of acceptable ideas later:

- “higher passing speed than typical for roads like this”
- “lower driver accommodation than similar suburban roads”

But do not ship this early.

## Do not skip

- canonical segment identity
- evidence precedence
- many-to-many cohorts
- hard separation from Safety Score

## Do not touch yet

- percentile-heavy vanity UI
- score normalization by region
- rider observation ingestion before the canonical mapper exists

## Definition of done

This system is healthy when:

- canonical segments can hold traffic behavior truth
- behavior dimensions remain distinct
- comparison priors exist without corrupting safety
- the architecture is ready for future explanation features


---

## Source File: docs/04-execution/01_system_manuals/sys-008-route_comparison.md

# System Manual — Route Comparison

## Purpose

Route comparison is one of Lanterne's clearest product promises.

It exists so a rider can answer:

- which route is safer?
- which route is less remote?
- which route will feel harder?
- which route makes more sense for this ride plan?

## What this system owns

- comparison between two or more route candidates
- side-by-side summary logic
- decisive difference highlighting
- same-context comparison rules

## What is already done

- route-level summaries are part of the target architecture
- index families are defined
- the product promise explicitly includes helping riders choose one route over another
- score and index logic already define the ingredients comparison needs

## What is not done yet

- no mature rider-facing route comparison workflow exists yet
- stored analysis backfill must land before this becomes powerful at scale
- same-scenario comparison for conditions is not fully built
- decisive section highlighting needs a designed output model

## Correct build order

1. Compare stable route truth first.
2. Compare conditions second.
3. Highlight decisive differences, not giant data tables.
4. Keep the interface calm.

## Build order

### Step 1 — Compare like with like

A comparison is only fair if both routes are evaluated under the same assumptions.

That means for any given comparison you should control for:

- mode profile
- analysis version
- start time if conditions are included
- pacing assumptions if conditions are included

### Step 2 — Start with route-level stable comparison

Before weather and timing complexity, make stable comparison strong.

Show things like:

- Safety Score
- Traffic Index
- Bike Support Index
- Remoteness
- Fatigue
- major route character differences

### Step 3 — Highlight decisive sections, not just averages

The rider does not just need to know that Route A scored 74 and Route B scored 79.
They need to know why.

Good comparison surfaces later should emphasize:

- dangerous sections
- unsupported stretches
- ugly descents
- major shoulder / infrastructure changes

### Step 4 — Add conditions comparison only after ride-instance modeling is ready

Condition comparison is powerful, but only if it is computed honestly.
That requires:

- same ride day / time assumptions
- same scenario context
- real slice timing

Do not fake this with vague weather badges.

## Do not skip

- same-context comparison rules
- stable comparison first
- decisive difference highlighting
- calm UI language

## Do not touch yet

- giant comparison matrices
- comparing dozens of routes at once
- social voting on best route
- overconfident condition comparison before ride-instance modeling lands

## Definition of done

This system is healthy when:

- a rider can meaningfully compare route options
- the comparison explains why one route is more appealing
- the UI helps decisions instead of dumping metrics


---

## Source File: docs/04-execution/01_system_manuals/sys-009-vault_system.md

# System Manual — Vault System

## Purpose

The Vault is Lanterne's curated route collection system.

It is not just another import source.
It is the place where Lanterne presents organized route libraries that are intentionally selected, grouped, and framed.

## What this system owns

- curated route collections
- collection metadata
- browsing / opening curated routes
- mode-aware or audience-aware collection framing

## What is already done

- the concept is clear: the Vault contains collections, not loose files
- the route acquisition model already treats Vault as part of “Open”
- the product direction already sees curation as distinct from raw external ingestion

## What is not done yet

- the full Vault data model is not formalized here
- collection browsing and editorial workflows are not complete
- the relationship between Vault collections and canonical routes needs disciplined implementation

## Non-negotiable rule

Vault is curated, mode-aware, and native to Lanterne.
External ingestion sources are not themselves the Vault.

## Correct build order

1. Define the collection model.
2. Link collections to canonical routes.
3. Define browse/open UX.
4. Add editorial polish later.

## Build order

### Step 1 — Define the collection entity

The first question is not “how do we show files?”
The first question is “what is a collection?”

A collection should be able to store:

- title
- description
- mode / audience fit
- collection type
- ordering rules
- route membership

### Step 2 — Keep route membership canonical

Vault should point to canonical route records wherever possible.
That way:

- analysis is reusable
- duplicates are minimized
- collection browsing stays stable

### Step 3 — Keep curation separate from ingestion

A raw GPX file upload is not automatically a Vault item.
A RWGPS import is not automatically a Vault item.

Vault is for:

- selected routes
- meaningful groups
- editorial framing

### Step 4 — Design browse/open around discovery, not file management

The rider should feel like they are opening a thoughtfully organized library, not rummaging through attachments.

That means:

- good collection labels
- useful summaries
- clear route previews
- calm browse flow

## Do not skip

- collection-first data model
- canonical route linkage
- separation from raw ingestion
- editorial framing

## Do not touch yet

- community comments in the Vault
- threaded discussion systems
- turning the Vault into a generic file cabinet

## Definition of done

This system is healthy when:

- Vault collections feel curated and intentional
- routes open cleanly into the existing route model
- the Vault adds discovery value without becoming a dumping ground for imports


---

## Source File: docs/04-execution/01_system_manuals/sys-010-voice_and_alerts.md

# System Manual — Voice and Alerts

## Purpose

This system defines how Lanterne should speak, alert, and remain helpful during rides without becoming annoying, chatty, or creepy.

## What this system owns

- alert classes
- spoken vs silent delivery rules
- speaker toggle behavior
- text-to-speech strategy
- quiet-default product behavior

## What is already done

- the product philosophy is strongly anti-clutter and anti-chatter
- we have already agreed that Lanterne should not become a nonstop talking companion
- the ride-time product direction supports selective situational alerts, not conversational AI by default
- iOS/browser device speech is the most realistic v1 path for spoken output

## What is not done yet

- a formal alert taxonomy
- per-alert opt-in/opt-out settings
- actual text-to-speech wiring for ride-time messages
- strong rules for what should never be spoken
- a clear native-vs-web boundary for future custom voice work

## Non-negotiable rule

Lanterne should be **quiet by default**.
If it speaks, it should say something that materially helps the rider.

## Correct build order

1. Define alert classes.
2. Define default delivery rules.
3. Add a simple speaker toggle.
4. Use device TTS first.
5. Delay any “voice personality” work until the system is genuinely useful.

## Build order

### Step 1 — Define alert classes

At minimum, classify alerts into:

- navigation cues
- safety / hazard alerts
- support / stop alerts
- environmental condition alerts
- expedition / resume alerts

Only some of these should be candidates for speech.

### Step 2 — Define what deserves spoken output

Good spoken candidates:

- imminent turn cue
- meaningful off-route warning
- major hazard ahead if timing is right

Poor spoken candidates:

- constant metric updates
- every minor condition change
- pseudo-emotional filler
- long descriptive paragraphs

### Step 3 — Add a simple speaker toggle and quiet defaults

The rider should have one easy control:

- sound on
- sound off

Later you can add more granular preferences.
But v1 should stay simple.

### Step 4 — Use device speech first

For v1, the right architecture is:

- app generates structured short message
- device speech engine speaks it if enabled

That keeps cost low and implementation sane.
It also avoids making a custom voice platform before the alerts are proven useful.

### Step 5 — Keep message language short and mechanical

Alerts should sound like:

- “Right turn ahead.”
- “Off route. Rejoin in 0.2 miles.”
- “Hazard ahead.”

Not like:

- “Great job, brave rider, let’s keep going.”

There may be room for rare encouragement later, but not before utility is nailed.

### Step 6 — Delay custom voice personality work

If Lanterne ever gets a more distinctive spoken layer, that should be a later deliberate system.
Not a default dependency of ride mode.

## Do not skip

- alert taxonomy
- quiet-default rule
- short message language
- device TTS first

## Do not touch yet

- fully conversational AI companion mode
- heavy server voice generation infrastructure
- chatty motivational narration
- custom voice branding before alert usefulness is proven

## Definition of done

This system is healthy when:

- spoken alerts are optional
- the rider can silence them instantly
- the alerts that do speak are actually helpful
- the system feels calm and trustworthy rather than needy


---

## Source File: docs/04-execution/02_infrastructure_projects/infra-001-canonical_schema_completion.md

# Project Manual — Canonical Schema Completion

## Goal

Finish the schema that Lanterne needs **before** large-scale route ingestion and analysis backfill.

The job here is not to invent the final dream schema for every future feature.
The job is to make the current architecture **stable, queryable, and safe to build on**.

## Why this matters

If the schema boundaries drift now, every later system becomes harder:

- ingestion creates duplicate route identities
- slice facts get mixed with analysis outputs
- weather bleeds into stable route tables
- expedition state gets stored in the wrong place
- comparative traffic work becomes impossible to reason about

This project protects the architecture's hard separations.

## What is already done

- `canonical_routes` exists as the route identity layer.
- `imported_routes` exists for provenance.
- `route_slices`, `route_slice_osm_facts`, and `route_slice_support_facts` exist.
- `route_analysis_runs`, `route_slice_analysis`, and `route_analysis_summary` exist.
- `route_expeditions`, `route_expedition_windows`, and `route_expedition_events` exist.
- Comparative traffic tables exist in scaffold form: `canonical_segments`, `route_segment_instances`, `segment_behavior_inputs`, `traffic_behavior_baselines`, `cohorts`, `segment_cohort_memberships`, `segment_observations`.

## What is not done yet

- `ride_instance_runs` is not migrated.
- `ride_instance_slice_conditions` is not migrated.
- `route_slice_overrides` is not migrated.
- `route_slice_effective_facts` does not exist yet.
- The full multi-day event model (`events`, `event_days`, `event_route_part_segments`) is not migrated.
- `route_expeditions` still points at `route_history(id)` in v1 instead of canonical route identity.
- The unique constraint on `canonical_routes.geometry_fingerprint` needs to be added before the next major ingestion run.
- A clean owner-run migration package needs to exist for all remaining schema work.

## Correct build order

1. Lock route identity constraints.
2. Lock fact / analysis / conditions separations.
3. Build the `route_slice_effective_facts` layer.
4. Add ride-instance tables.
5. Fix expedition references to canonical route identity.
6. Leave full event expansion for later.

## Detailed steps

### Step 1 — Lock canonical route identity

Do this first.

- Add the missing unique constraint on `canonical_routes.geometry_fingerprint`.
- Confirm `canonical_routes` and `imported_routes` use the **same fingerprint formula**.
- Confirm `imported_routes.id` is the only join key used by tables that reference imported routes.
- Document any legacy columns that remain only for backward compatibility.

### Step 2 — Freeze the hard separations in SQL and docs

Before more features ship, confirm these rules in both migration comments and docs:

- stable route analysis does **not** store weather or light timing
- ride-time conditions do **not** live in stable analysis tables
- expedition durability does **not** depend on ephemeral session state
- provenance does **not** determine canonical identity
- raw OSM tags are not the only stored truth

If a table violates one of those rules, fix it now instead of carrying the mistake forward.

### Step 3 — Build `route_slice_effective_facts`

This is the bridge between raw extracted facts and scored truth.

Create a materialized view or resolved table that combines:

- `route_slice_osm_facts`
- `route_slice_support_facts`
- future approved `route_slice_overrides`

The rule is simple:

- extracted facts are raw structured truth
- effective facts are what the scoring engine should use

Do not skip this layer. It prevents scoring code from having to merge multiple fact sources ad hoc.

### Step 4 — Add ride-instance tables

Create:

- `ride_instance_runs`
- `ride_instance_slice_conditions`

These tables hold time-dependent outputs only.

Examples that belong here:

- wind
- temperature
- precipitation
- light state
- glare flag
- moon phase

Examples that do **not** belong here:

- traffic index
n- bike support index
- remoteness index

### Step 5 — Re-anchor expeditions to canonical route identity

Current v1 shipped against `route_history(id)` so the product could move quickly.
That is acceptable temporarily, but not long term.

Planned state:

- expedition belongs to a rider + canonical route
- route history is just one personal relationship layer, not the durable route identity layer

Migration notes:

- do not break currently open expeditions
- write a backfill plan before changing foreign keys
- preserve old history linkage where needed for UI continuity

### Step 6 — Package the owner-run migration sequence

Because Lovable does not control the self-managed database, make this easy on future-you.

Produce either:

- one idempotent migration script, or
- a clearly ordered set of numbered scripts with instructions

The rule is: no mystery database surgery.

## Do not skip

- the unique constraint on `geometry_fingerprint`
- the `route_slice_effective_facts` layer
- the stable vs contextual separation
- the expedition re-anchor plan
- a migration package a human can safely run

## Do not touch yet

- full rider observation ingestion
- percentiles or fancy relative traffic UX
- full event/day schema unless it is blocking immediate work
- giant schema expansions for hypothetical future community features

## Definition of done

This project is done when:

- canonical route identity is protected by constraints
- effective facts exist as a clean scoring input layer
- ride-instance condition tables exist
- expedition durability has a clear path to canonical route identity
- remaining schema changes can be applied by a single owner-run migration sequence


---

## Source File: docs/04-execution/02_infrastructure_projects/infra-002-rusa_corpus_ingestion.md

# Project Manual — RUSA Corpus Ingestion

## Goal

Ingest the RUSA permanent corpus into Lanterne as **stable local route records**, not as fragile runtime lookups.

This project is about building the route library that everything else depends on.

## Why this matters

The RUSA corpus is one of the most valuable seed datasets available to Lanterne.
It is the fastest way to go from a route-by-route tool to a meaningful route intelligence library.

But only if the corpus is ingested correctly.

Done wrong, you get:

- duplicate routes
- bad provenance
- inconsistent naming
- broken joins
- routes that cannot be analyzed reliably later

## What is already done

- The architecture now supports canonical route identity separate from provenance.
- `canonical_routes`, `imported_routes`, and `external_route_catalog` exist.
- The system already recognizes RUSA permanents as a supported route source.
- Pipeline tooling exists to create slices and OSM facts once canonical routes are in place.

## What is not done yet

- The full permanent corpus is not loaded into canonical storage.
- Deduplication rules for near-identical or revised permanent routes are not fully formalized.
- Event/day relationships are only partially represented.
- Canonical route fingerprints still need the hard uniqueness lock before major ingestion.

## Correct build order

1. Collect and normalize source records.
2. Resolve each imported route to canonical identity.
3. Preserve source provenance.
4. Validate joins and naming.
5. Only then run slice generation and enrichment.

## Detailed steps

### Step 1 — Build the source inventory

For each RUSA permanent, gather and store:

- source route artifact
- permanent identifier
- region / chapter metadata
- route title
- distance
- source URL or internal reference
- ingestion timestamp

If the source comes in multiple shapes, normalize them first. Do **not** try to score or enrich during this step.

### Step 2 — Create imported route rows

Every source artifact gets an `imported_routes` row.

Store:

- source platform = RUSA
- source route identifier
- original geometry
- geometry fingerprint
- basic metadata

Rule: the imported row preserves what you got from the outside world.
It does not define canonical identity by itself.

### Step 3 — Resolve to canonical routes

For each imported route:

- compute fingerprint using the same formula used by `canonical_routes`
- match to an existing canonical route if the corridor is the same
- create a new canonical route only when no match exists

Two source artifacts describing the same road experience should land on the same canonical route.

### Step 4 — Preserve provenance cleanly

Do not throw away source detail.

Make sure the product can later answer questions like:

- which permanent did this come from?
- which chapter published it?
- did this route exist in more than one source record?

That means provenance stays in the import layer, not the canonical layer.

### Step 5 — Validate names and display labels

Separate these ideas:

- canonical name
- imported/source name
- rider-facing display label

The canonical route should not become a junk drawer for every variant title.

### Step 6 — Validate joins before scale

Before importing thousands of rows, test the join chain carefully.

Especially verify:

- `imported_routes.id` is used where expected
- nothing joins incorrectly to `source_route_id`
- event references can resolve back to canonical routes

### Step 7 — Only after route identity is stable, move to analysis prep

Once the corpus is ingested cleanly:

- generate slices
- generate OSM facts
- prepare backfill

Do not collapse ingestion and analysis into one giant uncontrolled job.

## Do not skip

- provenance preservation
- canonical resolution
- join validation
- naming discipline
- a small pilot run before the full corpus

## Do not touch yet

- full comparative traffic population
- rider observations
- rich public route pages
- social/community features around permanents

## Definition of done

This project is done when:

- the RUSA permanent corpus exists locally in canonical + imported form
- duplicates are resolved cleanly
- provenance is preserved
- joins are trustworthy
- the corpus is ready for slicing and enrichment


---

## Source File: docs/04-execution/02_infrastructure_projects/infra-003-osm_enrichment_pipeline.md

# Project Manual — OSM Enrichment Pipeline

## Goal

Turn canonical routes into **structured route facts** by slicing them and extracting normalized OSM variables.

This project is the bridge between raw geometry and reusable route intelligence.

## Why this matters

If route intelligence stays dependent on raw runtime OSM lookups, Lanterne remains fragile.

The pipeline is what moves the system toward:

- compute once
- reuse many times
- version facts cleanly
- compare routes at scale

## What is already done

- `pipeline/` exists as a separate Node-based project.
- `slice-builder.ts` exists.
- `osm-facts.ts` exists.
- `road-class.ts` exists.
- `route_slices` and `route_slice_osm_facts` tables exist.
- Current slice builder uses distance threshold and road-class boundaries.
- OSM facts extraction already maps a large normalized variable set.

## What is not done yet

- pipeline automation is not in place
- support/proximity facts are not fully integrated into one resolved layer
- richer slice boundary triggers are reserved for later versions
- pipeline observability is weak
- failure handling and retry policy need to be formalized

## Correct build order

1. Freeze the slice builder contract.
2. Freeze the normalized OSM fact contract.
3. Run on a small seed set.
4. Validate outputs manually.
5. Then automate.

## Detailed steps

### Step 1 — Freeze slice builder rules for v1

Current boundary triggers are enough for v1 if they are stable.

For v1, keep the rules explicit:

- max slice length threshold
- road-class change boundary
- minimum slice size floor

Do not keep changing slice semantics while also trying to build scoring and backfill.

### Step 2 — Freeze the normalized variable contract

The output of OSM enrichment needs to be boring and predictable.

That means:

- one clear column for each normalized variable
- enums where enums are already defined
- JSON only for evidence, traceability, or bounded uncertainty

Never let scoring depend directly on raw ad hoc OSM tags.

### Step 3 — Validate a seed set manually

Pick a small and mixed group of routes:

- rural permanent
- urban-ish permanent
- mountainous route
- route with obvious bridge/tunnel transitions
- route with bike infrastructure changes

For each route, inspect:

- slice lengths
- road-class boundaries
- surface values
- bike facility values
- confidence signals
- weird nulls

Do not automate the whole corpus before doing this boring check.

### Step 4 — Add support/proximity fact generation into the pipeline plan

Stable route truth is not only OSM roadway tags.
It also includes support context.

Make sure the pipeline plan clearly includes:

- settlement proximity
- food / water / lodging / medical proximity
- bailout access

These belong in `route_slice_support_facts`, not in OSM facts.

### Step 5 — Define rerun rules

Before automating, decide when a route gets re-enriched.

Examples:

- OSM schema change
- normalization logic change
- slice-builder version bump
- bug fix affecting extracted values

If this is not defined, future reruns will be chaotic.

### Step 6 — Automate only after outputs are stable

Automation can be:

- queue-based worker later
- owner-run batch now
- edge-triggered pipeline later

But do not automate a moving target.

### Step 7 — Persist operational metrics

The pipeline should log and persist enough to answer:

- how long did slicing take?
- how many slices were created?
- how many routes failed?
- which routes need rerun?

Without this, large backfills become blind.

## Do not skip

- manual validation on a small seed set
- column-level normalization discipline
- slice versioning
- rerun rules
- separation between OSM facts and support facts

## Do not touch yet

- fancy machine learning features
- rider observation ingestion
- canonical segment mapping inside the same job
- full automation before facts are trusted

## Definition of done

This project is done when:

- canonical routes can be sliced consistently
- normalized OSM facts are produced reliably
- support facts have a clear place in the pipeline
- rerun/version rules are defined
- the pipeline is stable enough to feed large-scale backfill


---

## Source File: docs/04-execution/02_infrastructure_projects/infra-004-route_analysis_backfill.md

# Project Manual — Route Analysis Backfill

## Goal

Compute stable route analysis for the corpus that has already been canonically ingested and enriched.

This is the project that turns stored route facts into rider-facing route intelligence.

## Why this matters

Until backfill happens, the corpus is just geometry plus facts.
After backfill, it becomes:

- searchable route intelligence
- comparable route summaries
- reusable scoring artifacts
- a real product library instead of a pile of imported lines

## What is already done

- Stable analysis tables exist: `route_analysis_runs`, `route_slice_analysis`, `route_analysis_summary`.
- The current client-side scoring engine exists and can serve as the behavioral reference.
- Stable analysis families and index definitions are documented.
- The schema build order already identifies backfill as the next major step after facts are ready.

## What is not done yet

- canonical routes are not broadly backfilled into stable analysis tables
- server-side backfill logic is not fully packaged
- route-level rollups need manual validation on a seed set
- score parity between client behavior and future stored analysis must still be proven

## Correct build order

1. Seed a small set of canonical routes.
2. Generate analysis runs.
3. Write per-slice analysis.
4. Write route summaries.
5. Compare against trusted client results.
6. Only then run larger batches.

## Detailed steps

### Step 1 — Choose a seed cohort

Do not start with all 3,000 routes.
Start with a small, varied set that can expose obvious scoring mistakes.

Good seed set:

- one route with clean shoulders and lower risk
- one route with ugly arterial exposure
- one route with remoteness but moderate traffic risk
- one route with obvious descent sections
- one mixed-surface or bike-support edge case

### Step 2 — Create `route_analysis_runs`

For each seed route, create a run row that records:

- canonical route id
- analysis family = stable_route
- analysis version
- mode profile
- source snapshot versions
- run status

This run row is the container everything else hangs from.

### Step 3 — Write per-slice analysis rows

Compute and write:

- safety_score
- traffic_index
- bike_support_index
- remoteness_index
- surface_quality_index
- fatigue_index
- descent_risk_index
- breakdown / flags / confidence JSON

Rule: these are stable route outputs only.
No wind, no temperature, no precipitation.

### Step 4 — Write route-level rollups

Compute and write `route_analysis_summary`.

This should include:

- route-level index rollups
- worst mile / worst sections
- summary breakdown

Make sure rollup strategy matches the analysis model.
A short dangerous section must not disappear into a giant average.

### Step 5 — Compare stored outputs to trusted client behavior

Before scaling, compare the new stored analysis against routes already analyzed through the client flow.

You are checking for:

- gross score mismatches
- rollup mismatches
- weird null propagation
- slices with impossible values

If the outputs disagree, do not backfill the whole corpus yet.

### Step 6 — Define rerun policy before large batches

Backfill creates stored truth, but not eternal truth.

Define when a route needs reprocessing:

- analysis version bump
- scoring fix
- facts version change
- route canonicalization correction

### Step 7 — Scale in batches, not one giant blind run

Once the seed cohort is believable:

- batch routes in controlled groups
- persist success/failure states
- log runtime and failure reasons
- leave a clean rerun path for only failed or stale routes

## Do not skip

- seed cohort validation
- run container rows
- rollup comparison against trusted behavior
- explicit rerun policy
- logging failures in a way a human can inspect later

## Do not touch yet

- ride-instance weather backfill
- canonical segment population inside the same job
- relative traffic context UX
- giant optimization work before stored outputs are believable

## Definition of done

This project is done when:

- stable analysis exists for the target corpus
- per-slice and route-level outputs are believable
- runs can be rerun intentionally by version
- the stored analysis is good enough to support route comparison and vault browsing


---

## Source File: docs/04-execution/02_infrastructure_projects/infra-005-canonical_segment_mapper.md

# Project Manual — Canonical Segment Mapper

## Goal

Resolve route-local segment occurrences into stable canonical segment identity.

This project is how Lanterne stops thinking only in terms of “this segment on this route” and starts thinking in terms of “this road behavior unit in the world.”

## Why this matters

Without canonical segment identity, comparative traffic work stays theoretical.

You cannot reliably attach:

- observed traffic behavior
- predicted traffic behavior
- cohort memberships
- future rider observations
- cross-route analytics

until route-local segments can resolve to stable canonical segments.

## What is already done

- The schema exists: `canonical_segments`, `route_segment_instances`, `segment_behavior_inputs`, `segment_cohort_memberships`.
- The hard naming discipline for observed / inferred / predicted / baseline fields is documented.
- The comparative traffic architecture is already defined conceptually.

## What is not done yet

- the mapper itself does not exist
- most `route_segment_instances.canonical_segment_id` values remain unresolved/null
- no automated exact / near-exact / new segment resolution pipeline exists
- no supersession workflow exists for changed segment identity over time

## Correct build order

1. Freeze segment identity rules.
2. Create route-local segment instances.
3. Build exact matching.
4. Build near-exact fallback.
5. Only then allow new canonical segment creation.
6. Leave traffic facts population until mapping is trustworthy.

## Detailed steps

### Step 1 — Freeze the canonical identity scaffold

Canonical segment identity must be deterministic, boring, and durable.

Base it on the documented scaffold:

- network source
- direction
- segmentation schema version
- start anchor key
- end anchor key
- normalized geometry hash
- semantic signature

Do not keep changing this while trying to map real data.

### Step 2 — Generate route-local segment instances first

For each analyzed route, write `route_segment_instances` rows.

These rows answer:

- where did this segment occur in this route?
- what did the route-level analysis think it was?
- what local geometry did it use?

Do this before attempting canonical resolution so you always have a local audit trail.

### Step 3 — Build exact matching

The first pass should be conservative.

Exact match means:

- same direction
- same anchor scaffold
- same normalized geometry signature
- acceptable confidence

Do not get cute here. Exact means exact.

### Step 4 — Build near-exact matching separately

After exact matching works, add a second path for near-exact resolution.

This may handle:

- tiny geometry differences
- anchor enrichment differences
- segmentation schema evolution

But it must produce:

- explicit `match_method`
- explicit `match_confidence`
- an audit trail that a human can inspect

### Step 5 — Only then create new canonical segments

If neither exact nor near-exact resolution works, create a new canonical segment.

Rule:

- do not create new canonical segments casually
- only create them after conservative matching has failed

Otherwise you will pollute the canonical graph with duplicates.

### Step 6 — Define supersession and deactivation rules

Road identity can evolve.
The model already anticipates `is_active` and `superseded_by_id`.

Define when a canonical segment is:

- still active
- replaced by a better canonical identity
- merged into another segment family

### Step 7 — Only after mapping is trustworthy, move on to traffic facts

Do not populate `segment_behavior_inputs` aggressively until the mapper is stable.

Bad mapping plus real behavior inputs equals poisoned canonical truth.

## Do not skip

- a route-local segment instance layer
- exact matching before near-exact matching
- explicit match confidence
- supersession rules
- conservative duplicate prevention

## Do not touch yet

- rider observation ingestion
- baselines-driven UI explanations
- relative percentile displays
- score normalization based on cohorts

## Definition of done

This project is done when:

- route-local segment instances can be resolved to canonical segment ids reliably
- unresolved cases are explicit, not hidden
- duplicate canonical segment creation is controlled
- the system is safe enough to begin attaching traffic behavior facts and cohort memberships


---

## Source File: docs/04-execution/02_infrastructure_projects/infra-006-traffic_baseline_build.md

# Project Manual — Traffic Baseline Build

## Goal

Populate the comparison layer that lets Lanterne say what is **typical** for roads like this in places like this — without corrupting the absolute Safety Score.

## Why this matters

Comparative traffic context is one of the things that could make Lanterne truly special.

But this project can also go badly wrong if it gets ahead of the evidence.

The rule is:

- absolute safety answers “how risky is this?”
- baselines answer “how unusual is this compared with its peers?”

Those are not the same question.

## What is already done

- The schema exists for `traffic_behavior_baselines`.
- `cohorts` exists and has v1 seed rows for geography, urbanicity, and road class.
- Comparative traffic architecture is documented.
- The system already distinguishes observed, inferred, predicted, baseline, confidence, and score fields conceptually.

## What is not done yet

- baseline rows are mostly empty
- cohort memberships are not populated because the canonical mapper is not done
- segment behavior inputs are not populated at scale
- rider-facing relative traffic explanations are not ready

## Correct build order

1. Seed the cohort catalog properly.
2. Define the baseline dimensions.
3. Decide evidence sources.
4. Populate only credible priors.
5. Leave rich UX until the data is real.

## Detailed steps

### Step 1 — Freeze the minimum v1 cohort dimensions

Do not try to model every possible cohort first.

v1 needs only:

- geography
- road class
- urbanicity

That is enough to anchor comparative reasoning.

### Step 2 — Define baseline fields you are actually prepared to support

The comparative traffic model currently wants dimensions like:

- pass frequency
- pass intensity / vehicle speed
- driver accommodation / slowdown

Do not create fake precision.
If a field has no credible source yet, leave it null rather than inventing numbers.

### Step 3 — Define evidence hierarchy for population

Before writing any baseline row, decide what source class it came from.

Potential sources:

- direct observations later
- deterministic inference from known roadway truth
- model outputs later
- external benchmark sources

Store provenance metadata. Baselines without provenance are not trustworthy.

### Step 4 — Populate the easiest, least controversial priors first

Good first candidates:

- geography rows
- road class rows
- urbanicity rows

These can exist even if many values start sparse.

The objective is to establish the structure correctly before chasing density.

### Step 5 — Keep baselines out of the headline score

Repeat this rule everywhere:

A dangerous road in a dangerous region is still dangerous.

Do not use baselines to soften, curve, or normalize the absolute Safety Score.
Baselines belong in:

- priors
- explanation layer
- future model calibration

### Step 6 — Delay rider-facing comparison UX until the data has real coverage

Do not ship lines like:

- “higher than normal for roads like this”
- “better than the local norm”

until the underlying data is credible enough to support them.

Sparse baselines plus confident UI language will destroy trust.

## Do not skip

- provenance for baseline rows
- v1 focus on geography / road class / urbanicity only
- hard separation from the Safety Score
- nulls instead of fake precision

## Do not touch yet

- percentile-heavy rider UI
- score curving by region
- deep event ecosystem cohorts
- community observations as baseline truth before mapper + ingestion are ready

## Definition of done

This project is done when:

- the minimum cohort scaffold is real and queryable
- baseline rows exist for the supported dimensions
- provenance is stored
- the system can begin using baselines for priors and internal explanation work without touching the absolute Safety Score


---

## Source File: docs/assessments/ass-001-architecture_audit_2026_03_08.md

**Lanterne Architecture Audit & Scaling Design**



1. System Architecture Map

Where computation occurs: Everything runs client-side in the browser. The backend is purely storage (Supabase tables) and proxy (edge functions forwarding to Overpass/HPMS/DOT APIs). There is no server-side analysis worker — the user's device is the compute engine.

Component Breakdown

| **Component**       | **Location**                                   | **Description**                                             |
| ------------------- | ---------------------------------------------- | ----------------------------------------------------------- |
| GPX Parser          | src/lib/gpx.ts                                 | Parses XML, extracts lat/lon/ele points                     |
| Route Sampling      | src/lib/route-geometry.ts                      | Samples route at ~200m intervals for matching               |
| Route Hash          | src/lib/route-cache.ts                         | Deterministic hash from start/end/distance/5 samples        |
| Corridor Builder    | src/lib/corridor.ts                            | Divides route into 0.05° grid tiles, fetches roads per tile |
| Tile Cache          | tile_cache table                               | Stores Overpass road data per grid tile (2-year TTL)        |
| Road Matching       | src/lib/window-matcher.ts, forensic-matcher.ts | Assigns GPX samples to nearest road candidates              |
| HPMS/DOT Enrichment | src/lib/hpms.ts, dot-enrichment.ts             | Fetches AADT/speed data for matched roads                   |
| Scoring Engine      | src/lib/safety-scoring.ts                      | Computes risk score, grade, segment-level metrics           |
| Cue Generation      | src/lib/topology-cues.ts                       | Generates turn-by-turn cue sheet from road sequence         |
| POI Enrichment      | src/lib/pois/                                  | Parallel streaming fetch of nearby services                 |
| Route Cache         | route_cache table                              | Shared hash→SafetyResult cache                              |
| Route History       | route_history table                            | Per-user personal route archive                             |

2. Compute Cost Drivers

| **Step**                  | **Relative Cost** | **Bound** | **Why**                                                      |
| ------------------------- | ----------------- | --------- | ------------------------------------------------------------ |
| Corridor tile fetches     | 🔴 High            | Network   | Each uncached tile = 1 Overpass API call. A 100mi route touches ~40-60 tiles. At 8 concurrent with 300ms throttle, this is 2-8s of network wait. Overpass is the #1 bottleneck. |
| HPMS/DOT enrichment       | 🟡 Medium          | Network   | Per-state API calls through edge function proxies. Only fetched for matched roads (selective enrichment helps). |
| Road matching (forensic)  | 🟡 Medium          | CPU       | Window matcher is O(samples × candidates). Forensic re-analysis adds dense sub-sampling in suspicious zones (up to 8 zones × 30 samples). |
| POI enrichment            | 🟡 Medium          | Network   | Independent parallel stream, but each uncached category×tile = another Overpass call. |
| Scoring engine            | 🟢 Low             | CPU       | Pure arithmetic over matched road data. Fast.                |
| Cue generation            | 🟢 Low             | CPU       | Sequential walk over road sequence. Negligible.              |
| GPX parsing               | 🟢 Low             | CPU       | Simple XML parse. Even 10K-point files complete in <100ms.   |
| Route hash + cache lookup | 🟢 Low             | Network   | Single Supabase query. <50ms.                                |

Key insight: At 100K routes, the dominant cost is Overpass API calls and Supabase storage I/O, not CPU. The client-side architecture means you pay zero server compute for analysis — but you pay in API rate limits and tile cache storage growth.



3. Cache Effectiveness

Current Strategy Assessment

| **Cache Layer** | **Scope**           | **Effectiveness**                                            |
| --------------- | ------------------- | ------------------------------------------------------------ |
| tile_cache      | Global, geographic  | ⭐⭐⭐⭐⭐ Excellent. Fixed 0.05° grid means ANY route through a tile reuses cached roads. Popular cycling corridors converge quickly. |
| route_cache     | Global, route-shape | ⭐⭐⭐ Good for exact re-uploads. Weak for near-miss routes.    |
| route_history   | Per-user            | ⭐⭐⭐⭐ Personal recall only. Not a compute cache.              |

Route Hash Collision Risk

The hash uses: start(4dp)|end(4dp)|dist(1dp)|5_samples(4dp) — that's effectively 7 coordinate pairs + 1 distance. Collision risk is negligible for genuinely different routes. The real problem is the opposite:

Cache Miss Scenarios (identical routes, different hash)

| **Scenario**                                               | **Impact**                                                   |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| Same road, different GPS device (slightly different track) | Samples at 1/6, 2/6... positions may land on different 11m grid cells → miss |
| Same route uploaded as forward vs. reverse                 | Start/end swap → guaranteed miss                             |
| Same loop started at a different point                     | Start/end shift → miss                                       |
| Route trimmed by a few hundred meters                      | Distance rounds differently → miss                           |

Recommendations

1. Normalize direction — always hash with the lexicographically smaller endpoint first
2. Normalize loops — detect loops (start≈end) and rotate samples to a canonical starting point
3. The tile cache is your real savings — at scale, route_cache provides ~15-30% hit rate, but tile_cache provides 60-80% hit rate on Overpass calls. Focus optimization energy on tile_cache.



4. Compute-Once Architecture

Current vs. Ideal

CURRENT: Browser → fetch tiles → match → score → write cache → render

IDEAL:  Browser → upload GPX → worker scores → stored artifacts → UI reads

Recommended Schema Evolution



-- Canonical route identity (geometry-based dedup)

routes

 id uuid PK

 route_hash text UNIQUE    -- current hash strategy

 gpx_geometry jsonb      -- simplified polyline for re-rendering

 total_km numeric

 created_at timestamptz



-- Full analysis output (versioned)

route_analyses

 id uuid PK

 route_id uuid FK → routes

 analysis_version int     -- replaces data_version

 safety_result jsonb      -- score, grade, metrics

 cue_sheet jsonb        -- extracted for independent access

 match_quality numeric     -- % of route matched to roads

 computed_at timestamptz

 compute_ms int        -- how long analysis took



-- Per-segment detail (enables segment-level queries)

route_segments

 id uuid PK

 analysis_id uuid FK → route_analyses

 segment_index int

 road_id bigint

 road_name text

 speed_class text

 distance_m numeric

 risk_contribution numeric



-- User's personal route list (thin join table)

user_routes

 user_id uuid

 route_id uuid FK → routes

 custom_name text

 uploaded_at timestamptz

Key principle: Separate route identity (geometry) from analysis (versioned computation) from user ownership (personal list). This lets you:

- Reprocess routes without losing user associations
- Share analysis across users who upload the same route
- Query segment-level data without deserializing a giant JSONB blob

What to Store So Analysis Never Reruns

| **Artifact**                        | **Why**                            |
| ----------------------------------- | ---------------------------------- |
| Matched road sequence with IDs      | Road identity is stable            |
| Per-segment speed/shoulder/facility | Avoids re-fetching HPMS/DOT        |
| Cue sheet                           | Independent artifact for rider use |
| Score + grade + all metrics         | Final output                       |
| Match quality %                     | Confidence indicator               |

What NOT to store: Raw road geometry (re-fetchable from tile_cache), full corridor roads list (too large, caused the 33MB Cloudflare 520 you already fixed).



5. Analysis Versioning Strategy

Current State

Single data_version = 2 integer. Bumping it invalidates every cached route at once — a cliff event that forces all users to re-analyze.

Recommended Design

analysis_version: int (monotonically increasing)

min_valid_version: int (stored in app_config)

Rules:

1. New algorithm ships with analysis_version = N+1
2. min_valid_version stays at N initially — old results remain valid
3. Background reprocessing gradually upgrades popular routes (by hit_count DESC)
4. Once reprocessing reaches 90%+ coverage, bump min_valid_version to N+1
5. Old entries below min_valid_version are served but flagged as "may be outdated"

Gradual rollout pattern:

- Week 1: New uploads get v3, cached v2 still served
- Week 2-4: Background job reprocesses top 1000 routes to v3
- Week 5: Set min_valid_version = 3, remaining v2 entries lazy-upgrade on next access

This avoids the "everyone re-analyzes on the same day" thundering herd.



6. Observability and Metrics

Current State

You already have route_perf_events with timings, counts, and errors — this is solid. The gap is aggregate visibility and cost attribution.

Recommended Additions

1. Tile fetch metrics (append to existing perf events)

| **Metric**                                 | **Why It Matters**                                           |
| ------------------------------------------ | ------------------------------------------------------------ |
| tile_cache_hit_rate                        | Direct proxy for Overpass API cost                           |
| tiles_total / tiles_cached / tiles_fetched | Already computed in CorridorFetchResult but not persisted    |
| overpass_429_count                         | Rate limit pressure indicator                                |
| overpass_504_count                         | Timeout indicator — signals tile size or query complexity issues |

These are already available in fetchCorridorRoadsWithMetrics() — just pipe them into the PerfRun.

2. Cost-control dashboard metrics (query from existing data)



-- Daily Overpass API call estimate

SELECT DATE(created_at),

 SUM((counts->>'tilesFromOverpass')::int) as overpass_calls,

 AVG((counts->>'tileCacheHitRate')::numeric) as avg_cache_hit

FROM route_perf_events

WHERE created_at > now() - interval '30 days'

GROUP BY 1;

3. Most important metrics for cost control (ranked)

| **Rank** | **Metric**            | **Why**                                                      |
| -------- | --------------------- | ------------------------------------------------------------ |
| 1        | Tile cache hit rate   | Every miss = Overpass call. At 80% hit rate with 100K routes, you save ~320K API calls/month |
| 2        | Overpass 429/504 rate | Signals when you're hitting rate limits — need to throttle or add caching |
| 3        | Analysis time p95     | User experience proxy. If p95 > 30s, users abandon           |
| 4        | Route cache hit rate  | Secondary savings — full analysis skip                       |
| 5        | HPMS/DOT fetch count  | Third-party API cost. Less critical since selective enrichment already limits this |

Cost Projection at Scale

| **Resource**                                    | **10K MAU / 100K routes**               | **Estimated Monthly Cost**         |
| ----------------------------------------------- | --------------------------------------- | ---------------------------------- |
| Supabase database (tile_cache growth ~500MB/yr) | ~2GB total                              | ~$25/mo (Pro plan)                 |
| Edge function invocations (Overpass proxy)      | ~200K calls/mo (assuming 80% cache hit) | Included in Pro                    |
| Supabase bandwidth                              | ~50GB/mo (tile reads + route history)   | Included in Pro                    |
| Overpass API                                    | ~200K queries/mo                        | Free (public API) but rate-limited |
| Client compute                                  | 0                                       | $0 (runs in browser)               |

Total estimated infrastructure: ~$25-50/mo — the client-side architecture is extremely cost-efficient.

Primary Scaling Risk

Overpass rate limiting is the #1 risk. At 200K+ queries/month you'll hit public Overpass server limits. Mitigations:

1. Maximize tile cache hit rate (already at ~60-80%)
2. Consider self-hosted Overpass instance (~$50/mo VPS) if rate limits become blocking
3. Pre-warm tiles for popular cycling regions (batch job during off-peak hours)





7. Cost Estimation Model

Per-Route Analysis Cost (Cold Cache)

| **Resource**                                                | **Quantity** | **Unit Cost**       | **Total**    |
| ----------------------------------------------------------- | ------------ | ------------------- | ------------ |
| Overpass tile fetches (via edge fn)                         | ~40–60 tiles | ~$0.0002/invocation | $0.008–0.012 |
| HPMS/DOT proxy calls                                        | ~5–15        | ~$0.0002/invocation | $0.001–0.003 |
| Supabase DB reads (tile_cache lookups)                      | ~60          | Free tier / ~$0     | ~$0          |
| Supabase DB writes (tile_cache, route_cache, route_history) | ~50–70       | Free tier / ~$0     | ~$0          |
| Client CPU (browser)                                        | ~5–30s       | $0 (user's device)  | $0           |
| Total cold-cache route                                      |              |                     | ~$0.01–0.015 |

Per-Route Analysis Cost (Warm Cache)

| **Resource**                                          | **Quantity** | **Unit Cost** | **Total** |
| ----------------------------------------------------- | ------------ | ------------- | --------- |
| route_cache read                                      | 1            | ~$0           | ~$0       |
| tile_cache reads (if re-fetching roads for optimizer) | ~40          | ~$0           | ~$0       |
| Total warm-cache route                                |              |               | ~$0.001   |

Per-Route View (History Reload)

Single route_history SELECT → effectively $0.

Scaling Estimates

| **Scale**  | **Routes/mo** | **Cache hit %** | **Edge fn invocations** | **Est. monthly cost**          |
| ---------- | ------------- | --------------- | ----------------------- | ------------------------------ |
| 1,000 MAU  | ~3,000        | ~30%            | ~120K                   | $25–40 (Supabase Pro)          |
| 10,000 MAU | ~30,000       | ~50%            | ~750K                   | $50–80 (Pro + edge fn overage) |

The dominant cost is Supabase Pro ($25/mo base) — edge function invocations and DB storage are minor. Client-side compute means zero server CPU scaling cost.

Why It Stays Cheap

- No server-side compute — analysis runs in the browser
- Tile cache is shared — popular corridors amortize across all users
- Route cache is shared — popular routes (century rides, club routes) hit cache
- Edge functions are thin proxies — <50ms execution, no heavy logic



9. Failure and Safety Limits

Current State

The codebase has some guardrails but they're informal:

| **Guardrail**                  | **Current**                               | **Status**    |
| ------------------------------ | ----------------------------------------- | ------------- |
| Max corridor tiles             | Implicit (~tile grid math)                | ⚠️ No hard cap |
| Max roads scanned              | None                                      | ❌ Missing     |
| Max tile requests per analysis | MAX_BATCHES_PER_RUN = 2 (POI only)        | ⚠️ Partial     |
| Analysis timeout               | CATEGORY_TIMEOUT_BASE_MS = 20s (POI only) | ⚠️ POI only    |
| Max GPX points                 | None                                      | ❌ Missing     |
| Max route length               | None                                      | ❌ Missing     |

Recommended Guardrails

1. GPX Ingestion Gate

MAX_GPX_POINTS = 50,000    // ~500mi at 1pt/50ft

MAX_ROUTE_MILES = 500     // reject ultra-distance until proven

Fail: Toast "Route too long — max 500 miles supported" before any computation begins.

2. Corridor Size Cap

MAX_CORRIDOR_TILES = 200    // ~200 × 0.05° tiles ≈ 350mi corridor

Fail: Truncate corridor to first N tiles, warn user "Analysis covers first X miles."

3. Road Scan Budget

MAX_ROADS_IN_MEMORY = 25,000  // prevent OOM on dense urban corridors

Fail: Stop ingesting roads, use what's available — scoring degrades gracefully since unmatched segments default to "unknown."

4. Tile Fetch Budget per Analysis

MAX_TILE_FETCHES_PER_RUN = 100 // hard cap on Overpass calls

Fail: Skip remaining tiles, mark analysis as partial: true in route_cache, show "Partial analysis" badge.

5. Total Analysis Wall Clock

MAX_ANALYSIS_RUNTIME_MS = 120_000 // 2 minutes

Fail: Abort analysis, return partial results with whatever roads were matched so far. Store with match_quality < 0.9 so it doesn't pollute route_cache.

6. Per-User Rate Limit

MAX_ANALYSES_PER_HOUR = 10   // prevent abuse / bot uploads

Fail: Toast "Rate limit reached — try again in X minutes." Check via user_usage table or in-memory counter.

Graceful Degradation Pattern

try {

 analysis = await runAnalysis(gpx, { tileBudget, roadCap, timeout });

} catch (e) {

 if (e instanceof BudgetExceededError) {

  return partialResult(e.partialData, 'budget_exceeded');

 }

 throw e;

}

The key principle: never fail silently, never crash the UI — always return the best result possible within budget and tell the user what was limited.



10. Priority Improvements (Ranked)
11. Direction-Independent Route Hash

Impact: High cache hit rate improvement, zero infrastructure cost

Current computeRouteHash produces different hashes for the same road ridden forward vs. backward. Fix: always order endpoints lexicographically.

const [a, b] = [start, end].sort();

hash = `${a}|${b}|${dist}|...`

Expected cache hit improvement: +15–25% for out-and-back / club routes.



2. Hard Guardrails on Corridor Size + Analysis Timeout

Impact: Prevents tail-latency disasters, protects Overpass API budget

Add the 5 guardrails from Section 9. This is the single most important reliability improvement — one 600-mile route currently generates ~120 uncapped tile fetches with no timeout.



3. Tile Cache TTL + Stale-While-Revalidate

Impact: Eliminates redundant Overpass calls for warm regions

Current tile_cache has no expiry — tiles fetched once live forever. Roads change (construction, new bike lanes). Add:

- fetched_at check: if tile > 90 days old, serve stale but queue background refresh
- Prevents cache from becoming stale while avoiding cold-start storms



4. Batch Tile Cache Reads via RPC

Impact: Reduces Supabase round-trips from ~60 to 1 per analysis

Current code calls getCachedTiles() which does individual or small-batch reads. A single RPC like get_road_tiles(_tile_keys text[]) (similar to existing get_poi_tiles) would cut DB round-trips by ~50x per analysis.



5. Store match_quality + partial Flag in Route Cache

Impact: Prevents bad results from polluting cache, enables re-analysis

Currently route_cache stores results without quality metadata. A low-quality match (poor GPS, missing tiles) gets cached and served to future users of the same route. Add:

- match_quality float — only cache if ≥ 0.9
- is_partial boolean — flag budget-exceeded results
- Future analyses can overwrite partial results with full ones

This is already partially implemented (the 90% threshold check exists in code) but isn't persisted as metadata in the cache row.

---

## Source File: docs/assessments/ass-002-architecture_audit_2026_03_24.md

# Lanterne Architecture Audit & Scaling Design — v2

**Updated:** 2026-03-24  
**Previous version:** 2026-03-08  

---

## What Changed Since v1 (March 8 → March 24)

### ✅ Improvements Delivered (from v1 recommendations)

| v1 Recommendation | Status | Implementation |
|---|---|---|
| Hard guardrails on corridor size + analysis timeout | ✅ **Shipped** | `analysis-guardrails.ts` — MAX_GPX_POINTS (50K), MAX_ROUTE_MILES (950), MAX_CORRIDOR_TILES (200), MAX_TILE_FETCHES_PER_RUN (100), MAX_ANALYSIS_RUNTIME_MS (120s). `GuardrailCollector` class aggregates violations and marks results partial. Advisory density warning (500 roads/mi) logs but doesn't block. |
| MAX_ROADS_IN_MEMORY cap | ✅ **Shipped** (evolved) | Replaced with density-based `MAX_ROADS_PER_MILE = 500` advisory check. Original hard cap (25K) deprecated in favor of per-mile reasoning. |
| Store match_quality + partial flag in route cache | ⚠️ **Partial** | `is_partial` flag exists in guardrail output; not yet persisted as metadata in route_cache rows. Code checks `matchQuality ≥ 95%` before caching, but partial-result metadata is not stored for future overwrite. |
| Direction-independent route hash | ❌ **Not started** | `route-cache.ts` still hashes start→end without lexicographic normalization. Loop detection not implemented. |
| Batch tile cache reads via RPC | ❌ **Not started** | Still individual/small-batch reads per tile key. |
| Tile cache TTL + stale-while-revalidate | ❌ **Not started** | Tiles still have no expiry; 2-year TTL documented but not enforced. |

### 🆕 New Systems Since v1

| System | Files | Description |
|---|---|---|
| **Chunk-corridor fetch strategy** | `corridor-chunks.ts` | Replaces per-tile Overpass queries with ~25mi polyline-buffered corridor chunks. A 100mi route now needs ~4 Overpass requests instead of ~50+. Includes RDP simplification, adaptive 429 backoff (batch size reduction + delay escalation), and automatic tile cache backfill for future fast loads. |
| **Expedition model (Phase 1)** | `expedition.ts`, `useExpeditionResume.ts`, `ExpeditionResumeCard.tsx` | Multi-day ride state: create/pause/resume expeditions backed by `route_expeditions`, `route_expedition_windows`, `route_expedition_events` tables. GPS mismatch detection (2mi threshold), four-choice resume card (State A/B), fire-and-forget writes. |
| **Analysis pipeline (server-side seed)** | `pipeline/` directory: `slice-builder.ts`, `osm-facts.ts`, `road-class.ts`, `run-slice-seed.ts`, `speed-utils-node.ts`, `supabase-client.ts` | Node.js pipeline for server-side slice generation. Boundary triggers: distance threshold (500m max) + road class change. OSM facts extractor maps 35+ variables per slice. Writes to `route_slices` + `route_slice_osm_facts`. |
| **Map orchestrator** | `useMapOrchestrator.ts`, `MapModeContext.tsx` | Centralized UI state machine for map interaction modes: idle, search, route_create, route_edit, analysis_loading, analysis_review, details_inspect. Governs selection, pin lifecycle, tap-off dismissal, visibility matrix. |
| **Lantern Wheel state machine** | `wheel-machine.ts`, `wheel-layout.ts`, `wheel-physics.ts`, `wheel-render.ts`, `wheel-gesture.ts` | Full state machine: closed → open_primary → snapping / inertial_spin → closed. Derived transition table from reducer (no manual maintenance). Admin inspector, interaction matrix, slot configs. |
| **Corridor reveal animation** | `useCorridorReveal.ts` | Post-analysis directional road network reveal: progressive fill (start→finish, 2s) → hold (1.2s) → progressive fade (1.8s). Route-position assignment via sampled polyline. Capped at 150 roads. |
| **Ride computer** | `ride-computer/metric-registry.ts`, `MetricSlot.tsx`, `RideComputerScreen.tsx` | Declarative metric registry: elapsed time, speed, distance, avg speed, moving time, elevation gain, route progress. Unit-aware (imperial/metric). Configurable slot layout. |
| **Ride overlays** | `ride-overlays/` | Typography-only map overlays: HazardOverlay, StopsOverlay, WeatherOverlay, RoadInspectOverlay. |
| **Scenario context** | `ScenarioContext.tsx` | Ride scenario parameters: mode, start time, avg speed, date. Foundation for time-dependent features (weather, sun position). |
| **Heatmap manual toggle** | Recent behavior change | Heatmap visibility is now 100% user-controlled. Removed all automatic activation on zoom/route creation. Single boolean source of truth. |
| **Supabase environment hardening** | `staging-client.ts` | Decoupled from Lovable Cloud env vars. Production URL/key hardcoded. Staging instance switchable. All edge function URLs reference exported constants, not `import.meta.env`. |

### 📐 Architecture Documentation Expansion

ADR count grew from ~17 to **34** (ADR-018 through ADR-034). Key additions:
- ADR-018: Server-cached slice analysis model
- ADR-020: Atomic analysis unit (route slices)
- ADR-021: OSM variable registry
- ADR-022: Phase 1 enum registry
- ADR-026: Canonical route identity
- ADR-027: Lantern screen model
- ADR-029: Ride-time situational awareness mode
- ADR-030: Ride mode power and sensor architecture
- ADR-031: Multi-day events as ordered references
- ADR-032: Comparative traffic context and segment cohorts
- ADR-033: Canonical segment identity
- ADR-034: Master route expeditions and windowed analysis

Design specs grew from ~4 to **16** (DS-001 through DS-034).

---

## 1. System Architecture Map

### Where Computation Occurs

| Layer | Location | What |
|---|---|---|
| **Client (browser)** | React SPA | GPX parsing, route matching, safety scoring, cue generation, POI enrichment, corridor fetching, heatmap rendering, expedition state management, ride computer |
| **Server (edge functions)** | 11 Supabase edge functions | Overpass proxy, HPMS proxy, DOT proxy, DOT AADT proxy, RWGPS proxy, RWGPS harvester, Stripe checkout/portal/subscription check, admin user/manage |
| **Server (pipeline)** | `pipeline/` Node.js scripts | Slice generation, OSM fact extraction, canonical route seeding — run manually or by future worker |
| **Database** | Self-managed Supabase | Tile cache, route cache, route history, expeditions, slices, OSM facts, user state, subscriptions |

### Component Breakdown

| Component | Location | Description |
|---|---|---|
| GPX Parser | `src/lib/gpx.ts` | Parses XML, extracts lat/lon/ele points |
| Route Sampling | `src/lib/route-geometry.ts` | Samples route at ~200m intervals for matching |
| Route Hash | `src/lib/route-cache.ts` | Deterministic hash from start/end/distance/5 samples |
| Corridor Builder (tile grid) | `src/lib/corridor.ts` | Divides route into 0.05° grid tiles, fetches roads per tile |
| Corridor Builder (chunk) | `src/lib/corridor-chunks.ts` | **NEW** — ~25mi polyline-buffered Overpass queries with RDP simplification |
| Tile Cache | `osm_road_tile_cache` table | Stores Overpass road data per grid tile (2-year TTL) |
| Road Matching | `src/lib/window-matcher.ts`, `forensic-matcher.ts` | Assigns GPX samples to nearest road candidates |
| Boundary Refinement | `src/lib/boundary-refinement.ts`, `boundary-snapping.ts` | Aligns transitions to intersection reality |
| HPMS/DOT Enrichment | `src/lib/hpms.ts`, `dot-enrichment.ts` | Fetches AADT/speed data for matched roads |
| Scoring Engine | `src/lib/safety-scoring.ts` | Computes risk score, grade, segment-level metrics |
| Cue Generation | `src/lib/topology-cues.ts` | Generates turn-by-turn cue sheet from road sequence |
| POI Enrichment | `src/lib/pois/` | Parallel streaming fetch of nearby services |
| Route Cache | `route_cache` table | Shared hash→SafetyResult cache |
| Route History | `route_history` table | Per-user personal route archive |
| Analysis Guardrails | `src/lib/analysis-guardrails.ts` | **NEW** — Hard limits + GuardrailCollector |
| Hazard Detection | `src/lib/hazards.ts` | Community hazard reports, Waze-style confirmations |
| Expedition Manager | `src/lib/expedition.ts` | **NEW** — Multi-day expedition create/pause/resume |
| Slice Builder | `pipeline/src/slice-builder.ts` | **NEW** — Server-side route slicing (500m max, road-class boundary triggers) |
| OSM Facts Extractor | `pipeline/src/osm-facts.ts` | **NEW** — 35+ normalized variables per slice |
| Road Class Normalizer | `pipeline/src/road-class.ts` | **NEW** — OSM highway → 7-class enum (ADR-022) |
| Map Orchestrator | `src/hooks/useMapOrchestrator.ts` | **NEW** — UI interaction mode state machine |
| Lantern Wheel | `src/components/lantern-wheel/` | **NEW** — Tool-selector wheel with state machine |
| Ride Computer | `src/lib/ride-computer/` | **NEW** — Declarative metric registry + slot layout |
| Corridor Reveal | `src/hooks/useCorridorReveal.ts` | **NEW** — Post-analysis road network animation |

---

## 2. Compute Cost Drivers

| Step | Relative Cost | Bound | Notes |
|---|---|---|---|
| Corridor chunk fetches | 🔴 High | Network | **Improved:** 4 Overpass calls for 100mi (was ~50). Adaptive 429 backoff. Still the #1 bottleneck for cold routes. |
| Corridor tile fetches (legacy) | 🟡 Medium | Network | Still available as fallback; chunk strategy used for new analyses. |
| HPMS/DOT enrichment | 🟡 Medium | Network | Unchanged. Function proxies, selective enrichment. |
| Road matching (forensic) | 🟡 Medium | CPU | Unchanged. O(samples × candidates) with forensic re-analysis. |
| POI enrichment | 🟡 Medium | Network | Unchanged. Parallel stream per category×tile. |
| Slice generation (pipeline) | 🟡 Medium | CPU + DB | **NEW.** Server-side only. O(points × roads) for boundary detection. ~400 slices per 200km. |
| Scoring engine | 🟢 Low | CPU | Unchanged. Pure arithmetic. |
| Cue generation | 🟢 Low | CPU | Unchanged. Sequential walk. |
| GPX parsing | 🟢 Low | CPU | Unchanged. <100ms even for 10K points. |
| Route hash + cache lookup | 🟢 Low | Network | Unchanged. Single query. |

### Key Insight Update

The chunk-corridor strategy reduced Overpass API pressure by **~10–12x** for cold routes. At 100K routes/month with 80% tile cache hit rate, estimated Overpass calls drop from ~200K/month to ~20–40K/month. This substantially delays the point at which a self-hosted Overpass instance is needed.

---

## 3. Cache Effectiveness

| Cache Layer | Scope | Effectiveness | Change Since v1 |
|---|---|---|---|
| `osm_road_tile_cache` | Global, geographic | ⭐⭐⭐⭐⭐ Excellent | **Enhanced** — chunk-corridor backfills tile cache for future fast loads |
| `route_cache` | Global, route-shape | ⭐⭐⭐ Good for exact re-uploads | Unchanged — still weak for near-miss routes |
| `route_history` | Per-user | ⭐⭐⭐⭐ Personal recall | Unchanged |
| `route_slices` / `route_slice_osm_facts` | Per canonical route | ⭐⭐⭐⭐ **NEW** | Server-side computed, versioned, reusable across users |

### Route Hash — Still Not Direction-Independent

The v1 recommendation to normalize direction (lexicographic endpoint ordering) and detect loops remains unimplemented. Estimated cache hit improvement: +15–25% for out-and-back / club routes.

---

## 4. Analysis Versioning Strategy

### Current State (improved from v1)

The slice builder now writes `slice_builder_version: '1.0'` per slice and `osm_snapshot_version` per fact row. This gives versioning at the slice level.

The client-side route cache still uses a simple `data_version` integer. Bumping it still invalidates all cached routes at once.

### Recommended Next Step

Adopt the v1-recommended gradual rollout pattern:
1. New uploads get v(N+1), cached v(N) still served
2. Background reprocessing upgrades popular routes by hit_count DESC
3. Once 90%+ coverage, bump min_valid_version

This is now more feasible with the pipeline infrastructure in place — the `run-slice-seed.ts` script could be extended to batch-reprocess routes.

---

## 5. Guardrails (Shipped)

| Guardrail | Limit | Enforcement | Behavior on Exceed |
|---|---|---|---|
| MAX_GPX_POINTS | 50,000 | Pre-analysis | Toast + reject |
| MAX_ROUTE_MILES | 950 | Pre-analysis | Toast + reject |
| MAX_CORRIDOR_TILES | 200 | Mid-pipeline | Clamp corridor, warn user |
| MAX_TILE_FETCHES_PER_RUN | 100 | Mid-pipeline | Skip remaining, mark partial |
| MAX_ANALYSIS_RUNTIME_MS | 120s | Mid-pipeline | Abort, return partial |
| MAX_ROADS_PER_MILE | 500 | Advisory | Console warning, no block |

The `GuardrailCollector` class aggregates violations, distinguishes advisory vs. blocking, and produces a summary string. This was the #2 v1 recommendation and is now fully shipped.

---

## 6. Expedition Model (New)

### Tables (owner-managed, not Lovable-managed)

| Table | Purpose |
|---|---|
| `route_expeditions` | Active/paused/completed expedition state per user per route |
| `route_expedition_windows` | Analysis windows (core + overlap ranges) for large routes |
| `route_expedition_events` | Event log: started, paused, resumed, manual_reposition, abandoned, progress_checkpoint, window_activated |

### Architecture

- **Checkpoints** every 2mi AND 10min (both conditions must be true)
- **Windowing** auto-determined by distance (>400mi), point count (>8K), or road density (>500/mi)
- **Default window:** 0–250mi core with 10mi overlap
- **Resume detection:** queries for active/paused expeditions on authenticated app boot (once per session)
- **GPS mismatch:** 2mi threshold determines State A (single Resume button) vs State B (four explicit choices)
- **All writes fire-and-forget:** failures log to console, never surface to rider

### Remaining Expedition Work (per docs/#TO_DO.md)

- Mismatch handling edge cases
- Join-at-current-location matching
- Chunk preload
- Seam transition between windows
- Overnight heuristics
- Fallback states

---

## 7. Pipeline Infrastructure (New)

The `pipeline/` directory is a standalone Node.js project that operates against the same Supabase instance. It currently contains:

| File | Purpose |
|---|---|
| `slice-builder.ts` | Generates route_slices rows with distance/road-class boundary triggers |
| `osm-facts.ts` | Extracts 35+ normalized variables per slice (ADR-021/022 compliant) |
| `road-class.ts` | OSM highway → 7-class enum (path, local_road, collector, arterial, highway_adjacent, service_road, track) |
| `run-slice-seed.ts` | CLI entry point for batch slice generation |
| `speed-utils-node.ts` | Server-side port of speed classification logic |
| `supabase-client.ts` | Separate Supabase client for pipeline use |

### Slice Builder Architecture

- **Max slice length:** 500m (forced distance threshold)
- **Min slice length:** 50m (prevents hairline slices at class boundaries)
- **Boundary triggers (v1.0):** distance threshold, road class change
- **Reserved for v1.1+:** surface transitions, bridge/tunnel, grade, light timing
- **Target density:** 200–400 slices per 200km route
- **Full rebuild model:** deletes existing slices before insert (cascade to osm_facts)

---

## 8. UI State Architecture (New)

### Map Orchestrator

The `useMapOrchestrator` hook is now the single source of truth for map interaction state. It manages:

| Concept | Implementation |
|---|---|
| **MapMode** (user intent) | idle, search, route_create, route_edit, analysis_loading, analysis_review, details_inspect |
| **MapSelection** (selected entity) | search-result, road-segment, poi, etc. |
| **Visibility governance** | Mode-derived flags: topActionRail, floatingControls, selectionCard, drawControls, editControls, analysisProgress, sideHandles |
| **Temporary pin lifecycle** | Place/clear/dismiss on mode transitions |
| **Detour chaining** | Sub-state within route_edit for tap-to-add-waypoint |

Consumed via `MapModeContext` to avoid prop drilling.

### Lantern Wheel State Machine

Formal state machine with 5 durable states (closed, open_primary, snapping, inertial_spin, disabled) and 11 event types. The transition table is **derived from the reducer** — no manual maintenance needed. Includes:

- Admin inspector for live state visualization
- Interaction matrix (per-state tap/drag/release behaviors)
- Slot configuration registry (tool ID, label, icon, modes, feature flags)
- Global event bus for cross-tree observation

---

## 9. Cost Projection (Updated)

| Resource | 10K MAU / 100K routes | Estimated Monthly Cost | Change Since v1 |
|---|---|---|---|
| Supabase database (tile_cache + slices) | ~3GB total | ~$25/mo (Pro) | **+1GB** for route_slices + osm_facts + expedition tables |
| Edge function invocations | ~50K calls/mo (chunk strategy) | Included in Pro | **Down from ~200K** — chunk strategy reduces Overpass proxy calls 10x |
| Supabase bandwidth | ~50GB/mo | Included in Pro | Unchanged |
| Overpass API (public) | ~50K queries/mo | Free but rate-limited | **Down from ~200K** |
| Client compute | 0 | $0 | Unchanged |
| **Total estimated** | | **~$25–50/mo** | Unchanged — chunk strategy extends the scaling runway |

### Primary Scaling Risk (Revised)

Overpass rate limiting is **less urgent** than in v1. The chunk-corridor strategy reduced Overpass load by ~10x. At current growth, the public Overpass API should sustain the app through ~50K MAU before a self-hosted instance is needed.

The **new primary risk** is pipeline compute scaling: as the slice builder moves from manual CLI runs to automated processing, it will need a worker model (edge function cron, external worker, or queued job system).

---

## 10. Priority Improvements (Updated Ranking)

| Rank | Improvement | Impact | Status |
|---|---|---|---|
| 1 | **Direction-independent route hash** | +15–25% cache hit rate, zero cost | ❌ Not started |
| 2 | ~~Hard guardrails~~ | ~~Prevents tail-latency disasters~~ | ✅ **Done** |
| 3 | **Batch tile cache reads via RPC** | ~50x fewer DB round-trips per analysis | ❌ Not started |
| 4 | **Tile cache TTL + stale-while-revalidate** | Prevents stale road data, avoids cold-start storms | ❌ Not started |
| 5 | **Persist is_partial + match_quality in route_cache** | Prevents bad results from polluting cache | ⚠️ Partial |
| 6 | **Pipeline worker model** | Automates slice generation for new routes | 🆕 — pipeline exists, no automation |
| 7 | **Compute-once architecture** (browser → worker → stored artifacts) | Eliminates repeat client-side analysis | 🆕 — slice model is the foundation; client still computes scores |

---

## 11. Database Tables (Updated)

| Table | Purpose | New? |
|---|---|---|
| `osm_road_tile_cache` | Cached Overpass road data by 0.05° grid tile | |
| `hpms_tile_cache` | Cached HPMS traffic data by tile + state code | |
| `route_cache` | Cached full route analysis results by route hash | |
| `route_history` | Per-user analyzed route history + saved detours | |
| `route_perf_events` | Route analysis performance telemetry | |
| `user_events` | Analytics events | |
| `user_usage` | Monthly usage counters | |
| `user_roles` | Admin role assignments | |
| `subscription_grants` | Manual subscription grants | |
| `promo_codes` | Promotional codes | |
| `promo_redemptions` | Promo code redemption tracking | |
| `profiles` | User profile data | |
| `safety_model_versions` | Safety model version history | |
| `safety_model_factors` | Per-version scoring factor weights | |
| `route_hazard_detections` | Micro-hazard detection log | |
| `route_expeditions` | Expedition state (active/paused/completed) | 🆕 |
| `route_expedition_windows` | Analysis windows for windowed routes | 🆕 |
| `route_expedition_events` | Expedition event log | 🆕 |
| `route_slices` | Generated route slices (per canonical route) | 🆕 |
| `route_slice_osm_facts` | Normalized OSM facts per slice | 🆕 |
| `canonical_routes` | Canonical route geometry (geometry-based dedup) | 🆕 |

**Total: ~22 tables** (up from ~18 in v1)

---

## 12. Edge Functions (Updated)

| Function | Description | New? |
|---|---|---|
| `check-subscription` | Stripe subscription + admin grants + promo codes | |
| `create-checkout` | Stripe checkout session | |
| `customer-portal` | Stripe customer portal redirect | |
| `admin-users` | List all users with subscription status | |
| `admin-manage` | Admin grant/revoke, promo management | |
| `overpass-proxy` | Overpass API proxy (dual-server fallback) | |
| `hpms-proxy` | HPMS federal traffic data proxy | |
| `dot-proxy` | DOT data proxy | |
| `dot-aadt-proxy` | DOT AADT traffic data proxy | |
| `rwgps-proxy` | RideWithGPS API proxy | 🆕 |
| `rwgps-harvester` | RideWithGPS route ingestion | 🆕 |

**Total: 11 edge functions** (up from 9)

---

## 13. Observability (Same Gaps as v1)

The `route_perf_events` table captures timing, counts, and errors. The gaps identified in v1 remain:

- **Tile fetch metrics:** `tiles_total / tiles_cached / tiles_fetched` already computed in `CorridorFetchResult` but not persisted to perf events
- **Overpass 429/504 counters:** computed in chunk strategy backoff logic but not persisted
- **Cost-control dashboard:** no aggregate query views built yet
- **Chunk strategy metrics:** `chunkCount`, `overpassRequests`, and backoff state are logged to console but not persisted

### Recommended: Persist chunk strategy metrics

The chunk-corridor strategy produces valuable operational data (`ChunkFetchResult.overpassRequests`, `backoffActive`, `_chunk429Count`) that should flow into `route_perf_events` for cost monitoring.

---

## 14. Failure & Graceful Degradation (Updated)

| Guardrail | Current | Status |
|---|---|---|
| Max GPX points | 50,000 | ✅ Enforced |
| Max route length | 950 mi | ✅ Enforced |
| Max corridor tiles | 200 | ✅ Enforced |
| Max tile fetches | 100 | ✅ Enforced |
| Analysis timeout | 120s | ✅ Enforced |
| Road density warning | 500/mi | ✅ Advisory |
| Per-user rate limit | None | ❌ Still missing |
| Expedition write failures | Fire-and-forget | ✅ Designed for graceful failure |
| Chunk strategy 429 backoff | Adaptive | ✅ Batch size reduction + delay escalation |

The graceful degradation pattern from v1 is now partially implemented — `GuardrailCollector` accumulates violations and the pipeline returns partial results. The `BudgetExceededError` pattern from v1 is not yet implemented as a typed exception.

---

## Summary of Architecture Health

**Strengths:**
- Client-side compute model keeps infrastructure costs near-zero
- Chunk-corridor strategy dramatically reduced Overpass API pressure
- Guardrails prevent runaway analyses
- Expedition model enables multi-day ride tracking
- Pipeline infrastructure (slice builder + OSM facts) lays groundwork for compute-once architecture
- Map orchestrator centralizes previously scattered UI state
- Architecture decisions well-documented (34 ADRs, 16 design specs)

**Top Risks:**
1. Route hash still direction-dependent (cache efficiency loss)
2. No pipeline automation (slice generation is manual CLI)
3. No per-user rate limiting
4. Observability gaps — operational metrics logged to console but not persisted
5. Tile cache has no TTL enforcement — risk of stale road data

**Architecture Trajectory:**
The system is moving from a pure client-compute model toward a hybrid model where the server owns canonical route intelligence (slices, OSM facts, analysis rollups) and the client focuses on presentation and interaction. ADR-018 ("Server computes truth. Client composes experience.") describes the target state. The pipeline, slice builder, and expedition model are the first concrete steps toward this architecture.



---

## Source File: docs/assessments/ass-003-safety_logic_audit_2026_03_28.md

# Pressure test of Lanterne’s Safety Score logic against road safety research and bicycle crash-severity literature

-by ChatGPT Pro Deep Research on 2026-03-28

## Executive summary

Lanterne’s headline Safety Score is **intended to estimate the relative likelihood of a rider being struck by a motor vehicle and the expected severity of the outcome**. In broad structure, that goal is consistent with mainstream roadway safety practice—predict crash frequency (or a proxy for conflict exposure) and separately consider severity, then adjust for roadway features and countermeasures. citeturn5view0turn0search8turn0search2

**Verdict on Lanterne’s philosophy given the actual math:** directionally sound conceptually (pre‑ride, not blame; narrow scope), **but the current implemented logic has several high-impact validity risks** that would materially distort the intended “strike likelihood + severity” signal.

The most consequential issues are:

**Scope leakage via rail crossings (headline score mismatch).** The strongest rail-related bicycling evidence primarily concerns **single‑bicycle falls** (wheel–flangeway/approach-angle mechanics), not motor-vehicle strikes. citeturn2search0turn14view0turn15view0  
**Inference:** rail crossings belong in a separate hazard layer by default, with careful conditions for when (if ever) they influence the motor‑vehicle strike score.

**Event-vs-exposure unit mismatch and segmentation sensitivity.** In the current formula, rail crossings are added into a per‑mile risk term and then multiplied by segment length (making their impact depend on segment boundaries). Left-turn penalties cap per segment, so segmentation can also change route risk if turns are distributed across many short segments (cap bypass).  
**Inference:** these are structural modeling problems (not “tuning”), and they can produce discontinuities that are hard to defend.

**Washout of “short dangerous sections,” amplified by a hard floor to 100.** Because the rollup computes average risk per mile and then forces scores to 100 when risk-per-mile is below a threshold, even routes with a small dose of high-risk roadway can receive a perfect score if surrounded by enough “zero-risk” path mileage. This is not a theoretical corner case; it will appear in long-distance routing where greenways/path segments connect to short on-road gaps. citeturn0search2turn13view0  
**Inference:** you can keep an average-per-mile score, but you need a “peak risk” or “worst segment” companion so the score cannot completely hide a pinch point.

**Overlapping mitigation for shoulder (double counting) and speed interacting twice.** Shoulder risk reduction is applied multiplicatively *and* a legacy shoulder credit still runs, while “shared lane / shoulder only” also receives a risk reduction in the infrastructure multiplier (potentially encoding shoulder twice). Speed also drives risk directly *and* triggers a high-speed floor on the infrastructure multiplier. citeturn7view0turn5view0  
**Inference:** keep multiplicative mitigation (that aligns with crash modification factor practice), but remove redundant shoulder credit and clarify whether “infrastructure type” is independent from shoulder width.

Overall, the current system can be made substantially more defensible **without redesigning the product** by (a) correcting event handling, (b) tightening scope boundaries (rail, “safe path” = not automatically zero), (c) eliminating double counting, and (d) adding an anti-washout rollup.

## What Lanterne’s current Safety Score is doing

### Summary of implemented logic

The model assigns each route segment a **RawRisk** based primarily on a weighted combination of **speed environment**, **traffic**, and **rail crossings**, scaled by segment miles. Risk is then reduced multiplicatively by **bike infrastructure** and **shoulder**, and increased additively by a **left-turn penalty** scaled by lane count. Segment risk points are summed and divided by total route miles (risk per mile), then mapped through a logistic function to a 0–100 Safety Score and letter grade.

This structure resembles a “relative expected harm per mile” index, then compressed to a user-facing score.

### Logic flow

```mermaid
flowchart TD
  A[Segment attributes] --> B[RawRisk: speed + traffic + rail, scaled by segment miles]
  B --> C[InfraFactor multiplier with high-speed floor]
  C --> D[ShoulderFactor multiplier for speed > 25 mph]
  D --> E[+ LeftTurnPenalty additive, lane-count scaled]
  E --> F[Segment RiskPoints]
  F --> G[Sum RiskPoints across route]
  G --> H[RiskPerMile = TotalRisk / TotalMiles]
  H --> I[Logistic mapping to 0–100 + grade]
```

### Evidence alignment of “predict then modify” structure

Road safety predictive practice commonly estimates expected crash frequency using models calibrated to roadway type and exposure, then applies **multiplicative adjustment factors / CMFs** to reflect design differences and countermeasures. citeturn5view0turn0search8turn7view0

**Inference:** Lanterne’s “base risk + multiplicative mitigations” is directionally consistent with that framework, but the specific factors (and their interactions) must respect scope (motor-vehicle strike) and avoid proxy stacking.

## Input-by-input evaluation of Lanterne’s implemented factors

This section classifies each Lanterne input as **well-grounded / weakly grounded / questionable / misplaced** for the *narrow* target: (1) likelihood of motor-vehicle strike and (2) expected severity if struck.

### SpeedRiskFactor and its 60% weight

**Evidence (strong for severity):** Multiple sources show that higher travel speeds substantially increase the probability of fatal and severe outcomes for vulnerable road users; rural guidance explicitly ties higher posted speeds to higher fatality likelihood, and severe outcomes escalate rapidly as impact speed rises. citeturn13view0turn0search2turn17view0  
**Evidence (moderate for crash occurrence proxies):** Bicycle crash prediction methods and safety screening tools frequently include posted speed limit as a key predictor or adjustment factor, including in pedestrian/bicycle SPF development work. citeturn6view0turn5view0

**Assessment:** **Well-grounded**, especially as a severity driver for long-distance riding where high-speed rural segments are common. citeturn13view0turn10search2

**Inference (pressure test of the implementation):**
- The piecewise mapping is plausibly attempting to capture non-linearity (severity increases sharply beyond ~30–40 mph), but its exact breakpoints and scaling are not anchored to a documented severity curve for cyclists. Treat as heuristic unless calibrated. citeturn13view0turn17view0  
- The **60% weight** implicitly makes speed dominate even when traffic is low. That can be defensible for “expected severity,” but it risks miscommunicating “likelihood of being struck” if users interpret the Safety Score as probability rather than expected harm. citeturn17view0turn13view0

### TrafficFactor / AADT and its 30% weight

**Evidence (strong for likelihood/exposure):** Motor-vehicle volume is central in pedestrian/bicycle crash prediction and screening; pedestrian/bicycle SPFs incorporate motor-vehicle traffic volume and segment length as key predictors, and exposure data limitations are repeatedly noted. citeturn5view0turn6view0turn13view0  
**Evidence (mixed for severity):** In bicycle–motor vehicle severity modeling on rural two-lane roads, higher AADT can be associated with *lower* injury severity (possibly reflecting lower operating speeds / different crash regimes), while higher speed limit increases severity. citeturn17view0turn13view0

**Assessment:** **Well-grounded** as a strike-likelihood driver; **weakly grounded** as a severity driver (direction can flip by context). citeturn17view0turn5view0

**Inference (pressure test of the implementation):**
- Lanterne’s traffic tiers and AADT curve are **plausible but underspecified** (tier thresholds and curve equation are not documented). This makes it hard to defend behavior at low vs moderate volumes and invites regional bias. Mark as unspecified where thresholds/curve are not explicit. citeturn5view0turn13view0  
- Defaulting “unknown traffic” to a *lower-than-medium* risk factor is a **directionally risky modeling choice** because missing data can become systematically optimistic unless treated as uncertainty. citeturn13view0turn5view0 (Evidence supports that non-motorized data gaps are common; inference is about missingness handling.)

### RailCrossings in the headline score (10% weight)

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["skewed railroad crossing bicycle wheel flangeway angle illustration","bicycle railroad crossing flangeway crash diagram","bicycle crossing train tracks low angle hazard"],"num_per_query":1}

**Evidence (strong for non-motor-vehicle injury risk):** Skewed rail crossings are well documented as hazardous for cyclists because narrow tires can be caught in flangeways; hazard increases as crossing angle decreases (worst around ~30° or less), and approach angle is a dominant determinant. citeturn14view0turn2search0turn15view0  
**Evidence (weak for motor-vehicle strike likelihood):** The most direct mechanisms in the literature are falls or loss-of-control events, not being struck by a motor vehicle. citeturn2search0turn14view0

**Assessment:** **Misplaced in the headline Safety Score** as currently implemented, given the narrow motor‑vehicle strike scope. citeturn2search0turn14view0

**Inference (pressure test of the implementation):**
- Lanterne treats rail crossings as if they are part of a *per‑mile* risk intensity and then multiplies them by segment miles, making their contribution depend on segmentation rather than just the number/geometry of crossings. This is not defensible as a stable estimate of either motor-vehicle strike likelihood or rail-crossing fall risk.  
- Because rail is included inside RawRisk, it is then reduced by InfraFactor and ShoulderFactor, even though bike facility type and shoulder width generally do not mitigate flangeway capture risk. citeturn14view0turn15view0

### SegmentMiles scaling in RawRisk

**Evidence:** Bicycle/pedestrian SPFs incorporate exposure through roadway segment length; longer segments generally produce more expected crashes all else equal. citeturn5view0turn6view0

**Assessment:** **Well-grounded** for continuous, per‑distance risk components (speed/traffic), **questionable** for point-event components (rail crossings). citeturn5view0turn2search0

**Inference:** Keep length scaling for “continuous exposure” contributors, but separate discrete events (rail crossings, turns) so they do not inherit segmentation artifacts.

### Infrastructure multiplier (InfraFactor), including the high-speed floor

**Evidence (benefit exists, magnitude varies):** Multiple peer-reviewed studies find lower injury risk on more separated/structured bicycle facilities compared with major streets without bicycle infrastructure, but effect size varies widely by context and facility definition. citeturn16view0turn2search1turn3view0  
**Evidence (U.S. CMF development for protected/separated lanes):** FHWA work developing CMFs for separated bicycle lanes (SBLs) versus traditional/buffered lanes estimates crash reductions, with CMFs in the approximate range of ~0.44–0.64 depending on base condition and vertical element configuration (city datasets, segment-level bicycle-involved crashes). citeturn4view0  
**Evidence (midblock vs intersection nuance):** Separated facilities can reduce midblock motor-vehicle bicycle crashes and associated serious injuries, but intersection conflict remains important and needs separate treatment. citeturn0search2turn13view0

**Assessment:** **Well-grounded as a concept**, **weakly grounded in its specific numeric multipliers across long-distance contexts**, and **questionable in its “safe paths = 0 risk” shortcut** (see below). citeturn16view0turn4view0turn0search2

**Inference (pressure test of the exact multipliers):**
- **Protected track = 0.25** implies a 75% risk reduction. This is within the broad envelope of published results for some contexts (e.g., cycle tracks with very low injury odds in a case-crossover study), but can be materially more aggressive than other observed relative risks (e.g., ~0.72 in Montreal cycle tracks). citeturn16view0turn2search1  
- The FHWA SBL CMF work suggests reductions relative to traditional/buffered lanes that are substantial but not uniformly as large as 0.25. citeturn4view0  
- Because Lanterne targets **motor-vehicle strike likelihood + severity**, the correct interpretation is closer to “expected harm reduction,” but most facility studies measure injury crash rates rather than conditioning on “motor-vehicle strike.” This gap is especially relevant for rail/track falls and other single-bicycle mechanisms. citeturn16view0turn15view0

**High-speed floor (InfraFactor ≥ 0.50 when speed ≥ 40 mph)**  
**Evidence:** Speed is a dominant driver of severe outcomes for vulnerable road users. citeturn13view0turn17view0  
**Inference (tradeoff framing):**
- **Conservative case (supports the floor):** even with separation, high-speed corridors often have higher-severity residual risk at crossings/driveways and at failure points; limiting credit prevents “false safety” labeling. citeturn0search2turn13view0  
- **Safe-system case (argues against a hard floor):** the relative protective value of separation is often greatest where motor vehicle speeds are high; a hard floor can perversely reduce credit exactly where separation should matter most (provided separation is truly continuous and crossings are controlled). citeturn0search2turn13view0turn4view0

### “Safe paths = risk 0, skip further steps”

**Evidence:** Off-street paths and cycle tracks reduce injury risk in some contexts but are not “zero risk,” and crashes can occur due to crossings, conflicts, and infrastructure hazards (including rail/track issues). citeturn16view0turn15view0turn13view0  
**Evidence (data practice):** Underreporting is more likely when crashes occur off the public right-of-way, including shared-use paths and driveways. citeturn13view0turn0search2

**Assessment:** **Questionable** for a headline score if “safe path” includes any road crossings or conflict points; **potentially acceptable** only if “safe path” is strictly access-controlled (no motor vehicles, no at-grade crossings). citeturn13view0turn16view0

**Inference:** Replace “risk = 0” with “very low baseline + explicit crossing events,” or at minimum keep “safe path” from triggering the route-level 100 floor when the route contains non-path gaps.

### Shoulder factor (multiplicative, only above 25 mph)

**Evidence (strong relevance for touring/rural):** Rural non-motorized safety guidance explicitly flags high speeds and absence of shoulders (space constraints) as common conditions where rural bicycle crashes occur, and speed is a major contributor to fatal outcomes. citeturn13view0turn10search2  
**Evidence (severity interaction):** On rural two-lane undivided roads, severity modeling finds an interaction between speed limit and shoulder width that significantly lowers severity (i.e., shoulder width matters in combination with speed). citeturn17view0  
**Evidence (prediction methods include shoulder type/width):** Ped/bike SPF development work uses paved shoulder / bicycle facility categories and recognizes shoulder width thresholds in adjustment factors. citeturn6view2turn5view0

**Assessment:** **Well-grounded** as a strike-likelihood modifier and potentially severity modifier (through avoidance space / conflict reduction), particularly for long-distance rural riding. citeturn13view0turn17view0

**Inference (pressure test of the implementation):**
- The chosen reductions (0.85 / 0.72) are **plausible but not traceable to a specific bicycle–motor vehicle CMF** in the provided documentation; treat as heuristic unless you can cite a calibration dataset. citeturn5view0turn17view0  
- Applying shoulder only above 25 mph is an understandable guardrail, but shoulders can matter below that threshold in towns and on 30–35 mph arterials; the threshold is a policy choice rather than a research-backed cutoff. citeturn5view0turn6view0

### Left-turn penalty (additive, capped, lane-count scaled)

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["motorist left turn facing bicyclist diagram FHWA crash type","bicycle left turn two-stage turn box protected intersection illustration","bicycle turning conflict intersection left turn diagram"],"num_per_query":1}

**Evidence (turning conflicts are real crash types):** FHWA crash-typing explicitly identifies “motorist left turn facing bicyclist” and other turning/merging crash types as recurring patterns, and PBCAT coding includes multiple left-turn crash types for both motorist and bicyclist. citeturn1search3turn1search7turn10search18  
**Evidence (lane count/road width and severity):** Route safety severity models and SPF work include roadway width / facility class / traffic volume and other operational/physical factors, consistent with the idea that multi-lane, high-volume contexts increase conflict complexity and potential injury severity. citeturn12view0turn6view0  
**Evidence (context dependence):** National summaries show most bicyclist fatalities occur at non-intersection locations in recent years, which implies intersection turns are important but not dominant everywhere—especially for long-distance rural routes. citeturn0search3turn13view0

**Assessment:** **Well-grounded conceptually** (turning/crossing conflicts are within a “motor-vehicle strike” scope), but **questionable as implemented** because it treats left turns as a uniform hazard largely independent of intersection control, turning volumes, speed regime, and bicycle facility intersection design. citeturn1search3turn12view0turn13view0

**Inference (implementation-specific critique):**
- The per-turn penalty (0.15) and weight (0.21) are not traceable to a known crash model coefficient or CMF; without calibration, it is a heuristic proxy. citeturn12view0turn5view0  
- The per-segment cap can be bypassed by segmentation (many short segments each below the cap sum to more than one long segment above the cap), creating instability.  
- Lane count scaling is directionally plausible but under-specified: “lane count” could mean total lanes, lanes crossed, or lanes per direction, and those distinctions matter for crossing-path exposure. Mark as unspecified. citeturn6view0turn5view0

### Logistic normalization and the “RPM floor to 100”

**Evidence:** Logistic transformations have been used in bicycle route safety rating research (e.g., to map predictors to an expected injury severity index), which shows the approach can be defensible as a *presentation layer* if the underlying risk index is valid. citeturn12view0  
**Evidence (data limits):** Underreporting and incomplete non-motorized exposure data are persistent problems, which generally favor *relative* scoring and careful interpretation of any absolute mapping. citeturn0search2turn13view0turn5view0

**Assessment:** **Weakly grounded** as currently parameterized (midpoint/steepness/floor appear arbitrary and can dominate behavior), not because logistic mapping is inherently wrong, but because the mapping meaning must be calibrated to real distributions and desired semantics. citeturn12view0turn5view0

**Inference (why the floor matters):**
- A hard rule that forces SafetyScore to 100 below RPM < 0.05 makes the output sensitive to “dilution,” enabling perfect scores even with short high-risk gaps on otherwise safe routes. That contradicts the user expectation that “a short dangerous section should still matter,” especially in touring where bridges/high-speed gaps are common pinch points. citeturn13view0turn0search2

### Legacy shoulder credit (additive, capped) alongside a multiplicative shoulder factor

**Evidence (modeling principles):** CMF guidance defines CMFs as multiplicative and warns that combining multiple countermeasure effects without accounting for overlap can over- or underestimate effects. citeturn7view0turn0search13

**Assessment:** **Questionable / likely misplaced** because it creates shoulder double counting relative to the multiplicative factor and to the infrastructure “shared lane / shoulder only” category. citeturn7view0turn5view0

**Inference:** Remove the legacy credit (or set it to zero) once the multiplicative shoulder factor remains, and add a separate “data confidence” flag if the legacy credit was compensating for missing width data.

## Missing variables and overlap in Lanterne’s implemented logic

### Missing variables that most likely matter for motor-vehicle strike likelihood or severity

Ranked by expected impact on Lanterne’s narrow target and relevance to long-distance riding, with notes on whether they belong in the headline score or a separate layer.

1. **Heavy vehicle exposure (truck route / % trucks proxy) — headline score (severity-weighted)**  
   **Evidence:** Bicycle route safety rating research includes truck routes among predictors of injury severity from motor-vehicle crashes. citeturn12view0  
   **Evidence (systemwide severity context):** In large-truck-involved fatal crashes, a meaningful share of fatalities are “nonoccupants” (including pedestrians and pedalcyclists), indicating the stakes of heavy vehicle exposure in vulnerable-road-user harm. citeturn10search0  
   **Inference:** For long-distance cyclists, freight corridors and high-truck-share highways are common; adding a truck proxy is one of the highest leverage severity improvements.

2. **Intersection exposure and control type (signal/stop/roundabout; major-road crossing count) — headline score (likelihood)**  
   **Evidence:** Turning/merging crash types (including left turns) are systematically cataloged, indicating intersection conflicts are a recurring motor-vehicle strike pathway. citeturn1search7turn10search18  
   **Evidence:** Rural guidance notes many rural non-motorized crashes are non-intersection-related, so intersection exposure needs to be modeled explicitly rather than assumed dominant everywhere. citeturn13view0turn0search3  
   **Inference:** Replace “left turn count” as a proxy with “high-risk crossing exposure,” while still keeping a left-turn component where it truly represents crossing-path risk.

3. **Access density / driveway density — headline score (likelihood)**  
   **Evidence:** Ped/bike SPF work includes segment length, total traffic volume, and number of driveways as predictors in crash occurrence models, supporting driveway density as a conflict-point driver. citeturn6view0turn5view0  
   **Inference:** Even on touring routes, “main street” segments through small towns can have concentrated driveway exposure—the exact sort of short section that average-per-mile rollups tend to wash out.

4. **Roadway width / lane width / crossing distance proxies — headline score (likelihood + severity)**  
   **Evidence:** Bicycle route safety rating research uses lane width and highway classification; SPF work and screening tools include roadway width and facility type factors. citeturn12view0turn6view0  
   **Inference:** This is partly captured by speed and volume, but width affects overtaking dynamics and crossing complexity in ways that speed/volume alone do not reliably proxy.

5. **Curvature / grade crest / visibility (geometry proxies) — headline score (likelihood), route-reality only if modeled as “difficulty”**  
   **Evidence:** On rural two-lane roads, grades and curved grades are associated with higher injury severity in bicycle–motor vehicle crashes. citeturn17view0  
   **Inference:** Only include geometry where it plausibly increases motor-vehicle strike risk (sight distance, risky overtakes), not as a generic “hard riding” penalty.

6. **Shoulder usability constraints (rumble strip accommodation; effective rideable shoulder) — headline score (likelihood)**  
   **Evidence:** FHWA rumble-strip bicycle-issues guidance emphasizes that rumble strips can create challenges for bicyclists and that agencies use design flexibilities (bike gaps, placement) to preserve rideable space. citeturn19view0  
   **Inference:** Lanterne already models shoulder width; adding “effective shoulder” would address real touring corridors where a nominal shoulder exists but is not rideable.

7. **Bicycle volume / “safety in numbers” proxy — headline score (likelihood), but data-limited**  
   **Evidence:** “Safety in numbers” literature finds per-person collision risk can decrease as walking/bicycling volumes increase (population-level relationship). citeturn9search11turn9search15  
   **Inference:** For touring, a route popularity proxy (if available) could improve strike-likelihood estimation, but beware endogeneity (people choose safer routes). This is Phase 1+.

Variables that clearly matter but should remain in separate layers under Lanterne’s constraints: **darkness/lighting, fog/weather, and other conditions**—even though severity models show darkness and fog increase severity. citeturn17view0turn13view0

### Variables likely double-counted or overlapping in Lanterne’s current logic

Ranked by how likely they create unstable or biased scores.

1. **Shoulder modeled three times:** InfraFactor category (“shared lane / shoulder only”), multiplicative ShoulderFactor, and legacy shoulder credit. citeturn5view0turn7view0  
2. **Speed modeled twice:** SpeedRiskFactor directly, and also via the high-speed floor that reduces infrastructure credit based on speed. citeturn13view0turn0search2  
3. **Rail counted as per-mile exposure, then multiplied by miles, then mitigated by infra/shoulder:** conflates discrete hazard events with continuous exposure and introduces unintended overlap with mitigation variables. citeturn2search0turn14view0  
4. **Left-turn cap bypass via segmentation + lane-count factor:** lane count is being used only for turn penalties, and without clear definitions it risks proxy overlap with road class/width and intersection type. citeturn6view0turn12view0  
5. **“Safe path = 0” plus “RPM floor to 100”:** collectively creates a structural override that can dominate and mask on-road exposure. citeturn13view0turn0search2

## Targeted design questions applied to Lanterne’s exact implementation

### Whether left turns belong in the headline score under Lanterne’s math

**Evidence:** Left-turn-related bicycle–motor vehicle crash types are well-established in FHWA crash typing and PBCAT coding. citeturn1search3turn1search7turn10search18

**Inference (verdict for Lanterne):**  
Left turns *can* belong in the headline motor-vehicle strike score, but **not as a uniform additive penalty** detached from intersection context. Under Lanterne’s current math, “left turn count × constant” is acting as a crude proxy for **crossing-path conflict exposure**, which is directionally correct, but the implementation is at high risk of:
- over-penalizing benign left turns on low-volume roads, and
- under-penalizing the few truly hazardous crossings (e.g., high-speed multi-lane arterials) because the penalty saturates and is not linked to speed/volume/control.

**Practical framing to keep it in-scope without redesign:** treat “left-turn penalty” as a **high-risk crossing event penalty**, with gating based on speed/traffic/lane environment (which Lanterne already has). This stays within the narrow motor-vehicle strike scope.

### Rail crossings and micro-hazards placement given Lanterne logic

**Evidence:** Rail/track-related hazards are strongly supported for cyclist falls/injuries, and key determinants like crossing angle and flangeway mechanics are documented. citeturn2search0turn14view0turn15view0  
**Evidence:** Underreporting is common for crashes off the main roadway system and for non-motor-vehicle crashes, meaning rail/track hazards won’t be well-calibrated using police crash files alone. citeturn13view0turn5view0

**Inference (placement for Lanterne):**
- **Headline Safety Score (motor-vehicle strike):** rail crossings should be **removed** by default because their dominant mechanisms are not motor-vehicle strikes and because Lanterne’s current mitigation math would discount them incorrectly.  
- **Hazard layer:** rail crossings should be **prominent**, ideally with severity cues (skew angle risk, surface condition risk).  
- **Both (only if you have a defensible strike pathway):** include in headline score only when the crossing geometry forces cyclists into travel lanes or creates documented merge conflicts (this requires additional data; otherwise keep it separated).

On “micro-hazards” beyond rail: Lanterne’s provided logic does not specify additional micro-hazard scoring; treat as unspecified rather than inferred.

### Whether lane count, roadway class, speed, and traffic volume should be separate or collapsed in Lanterne formulas

**Evidence:** Ped/bike predictive methods incorporate multiple correlated roadway variables (speed limit, traffic volume, roadway width, facility type, segment length, driveways), and calibration/structure matter to avoid redundancy. citeturn5view0turn6view0  
**Evidence:** A route risk severity model used highway classification and other operational/physical factors to estimate injury severity. citeturn12view0

**Inference (applied to Lanterne as-built):**
- Lanterne currently has **speed + traffic**, and uses **lane count only inside the left-turn penalty**. Roadway class is not explicitly included.  
- Given the current structure, **do not add roadway class as another additive “danger term”** (it would likely proxy speed and traffic again). Instead, use roadway class (if added later) for (a) imputing missing AADT/speed and (b) selecting different parameter regimes (rural two-lane vs multilane arterial) consistent with how safety methods stratify facility types. citeturn5view0turn13view0  
- Lane count’s most defensible role is as a **crossing distance / conflict complexity modifier**, not a general segment penalty, unless you can explicitly link it to increased strike likelihood independent of speed/volume in touring contexts. citeturn6view0turn12view0

### Whether bike infrastructure and shoulder should be multiplicative mitigation or additive credits in Lanterne’s math

**Evidence:** CMFs are defined as multiplicative factors; guidance notes CMFs can be multiplied when treatments are independent, but warns that independence assumptions can cause over/underestimation and that CMFs must match crash types/severities. citeturn7view0turn0search13  
**Evidence:** FHWA’s SBL CMF development work explicitly estimates crash reductions for facility conversions, reinforcing the “multiplier” framing. citeturn4view0

**Inference (applied to Lanterne):**
- Lanterne’s core choice—multiplicative InfraFactor and ShoulderFactor—is **directionally correct** and more defensible than additive “credits” for a nonnegative risk quantity.  
- However, mixing multiplicative mitigation with a legacy additive shoulder credit is not defensible unless it is clearly modeling a different mechanism; here it appears redundant.  
- A practical defensible approach is **one primary “separation/operating-space” multiplier** (mutually exclusive categories) plus limited additional multipliers only where mechanisms are distinct.

## Calibration and rollup philosophy applied to Lanterne’s rollup and normalization

### Absolute vs relative scoring

**Evidence:** Underestimation of bicycling activity and underreporting of nonfatal bicyclist injuries limit the ability to claim absolute probabilities from route scores without robust exposure measurement and integrated injury data. citeturn0search2turn13view0turn5view0

**Inference:** Lanterne should present the headline Safety Score as **relative expected harm** (comparative risk), not an absolute “chance you’ll be hit,” unless/until it is calibrated against validated exposure and crash outcomes.

### Route-level rollup and “short dangerous sections” washout

**Evidence:** Safety prediction practice commonly aggregates expected crashes across segments and intersections (additive in expectation), which conceptually supports summing segment risk contributions. citeturn5view0turn0search8  
**Evidence:** Rural bicyclist crashes often occur in non-intersection, high-speed contexts, meaning a small number of segments can dominate severe outcome risk. citeturn13view0turn0search2

**Inference (specific to Lanterne):**
- Lanterne’s rollup computes **average risk per mile**, not cumulative trip risk. That is fine for route comparison across distances, but it will **wash out** short pinch points.  
- The washout is substantially worsened by two design choices:  
  - “safe paths = 0 risk” and  
  - a hard floor of SafetyScore = 100 below a low RPM.  
  Together, they can produce perfect scores for routes that contain nontrivial short high-risk road gaps.

**Practical correction without redesign:** keep the average-per-mile score **but add a second rollup component** that captures peak exposure. Two defensible options:
- **Worst-km (or worst 0.5–2 km) risk**, or
- **95th percentile segment risk** (distance-weighted).

Either makes it impossible for a single dangerous connection to disappear in the headline, while still keeping the narrow scope.

### Lanterne’s logistic mapping (midpoint=2.5, steepness=1.4)

**Evidence:** Logistic transformations have precedent in route safety rating research, but the meaning of the transformed score depends on calibration and the validity of predictors. citeturn12view0turn5view0

**Inference:** In Lanterne, the logistic mapping is best treated as **a UI compression**, not a scientifically meaningful probability. The midpoint and slope should be set by (a) desired interpretability and (b) empirical distribution of RPM across a representative route corpus, and revisited once you have outcome validation.

## Best production scoring approach for Phase 0 / Phase 1 Lanterne

### Phase 0 recommendations for what to ship now

These are the highest-leverage changes that improve defensibility **without redesigning the product**.

**Fix scope and event mechanics**
- Remove **rail crossings** from the headline score and move them to a hazard layer (or at minimum, convert rail to an event penalty not multiplied by segment miles and not mitigated by infra/shoulder). citeturn2search0turn14view0  
- Replace “safe paths = 0 risk” with “near-zero baseline” unless you can guarantee no at-grade crossings; at minimum, ensure “safe paths” do not enable a route-level perfect score when non-path gaps exist. citeturn16view0turn13view0

**Eliminate double counting**
- Remove the **legacy additive shoulder credit** once the multiplicative shoulder factor is active. citeturn7view0  
- Clarify and enforce independence: if “shared lane / shoulder only” implies shoulder, set its InfraFactor to 1.0 (or redefine categories so shoulder width is only in ShoulderFactor). citeturn7view0turn5view0

**Reduce washout**
- Remove the RPM<0.05 “force to 100” rule or replace it with a softer cap; add a peak-risk companion rollup (worst-km or 95th percentile segment risk). citeturn13view0turn0search2

**Make left turns defensible**
- Keep a turn-related term but gate it using existing variables: only apply meaningful penalties when the turn occurs in high-speed/high-volume/multi-lane contexts (the known conflict-risk regime), and ensure caps cannot be bypassed by segmentation. citeturn1search3turn6view0turn13view0

### Phase 1 recommendations for what to defer until later versions

**Add missing high-impact predictors once data pipelines exist**
- Add a heavy-vehicle proxy (truck route / % trucks) as a severity modifier. citeturn12view0turn10search0  
- Add access/driveway density and intersection-type features for strike likelihood. citeturn6view0turn5view0  
- Add “effective shoulder usability” including rumble strip accommodation where data can support it. citeturn19view0  
- Consider a bicycle exposure proxy (route popularity) carefully, acknowledging safety-in-numbers relationships and endogeneity. citeturn9search11turn9search15

**Calibration approach (Phase 1)**
- Keep the score **relative** unless you have validated exposure measures and crash outcome comparisons; use calibration against known risk distributions and, if available, compare predicted high-risk segments against systematic safety screening outputs. citeturn5view0turn13view0turn0search2

### Final concise verdict

Lanterne’s direction (narrow, pre-ride, expected harm framing) is strong, but the current implementation **mixes discrete hazards with continuous exposure, includes a non-strike hazard in the headline score, double-counts shoulder mitigation, and can produce false-perfect scores through rollup/thresholding**. Fixing those items in Phase 0 will materially increase scientific defensibility while staying within the existing product architecture. citeturn7view0turn13view0turn2search0

### Actionable Phase 0 implementation checklist

- Redefine the headline Safety Score explicitly as **relative expected harm per mile** (strike likelihood × severity proxy), not a probability. citeturn0search2turn13view0  
- Remove rail crossings from headline score; implement as hazard layer events (and do not reduce them via Infra/Shoulder multipliers). citeturn2search0turn14view0  
- Delete the legacy additive shoulder credit (or set weight to zero). citeturn7view0  
- Remove/replace the RPM<0.05 “score=100” floor; add a peak-risk rollup (worst-km or 95th percentile). citeturn13view0turn0search2  
- Gate left-turn penalties by speed/traffic/lane context; enforce cap in a segmentation-invariant way (route-level cap or event-level modeling). citeturn1search3turn6view0  
- Audit category definitions: “safe path,” “protected track,” “shared lane/shoulder only,” and “lane count” must be unambiguous and mutually exclusive. citeturn5view0turn6view0

### Implementation table

| Factor | Keep / Modify / Remove / Defer | Why | Confidence | Recommended implementation timing |
|---|---|---|---|---|
| SpeedRiskFactor (piecewise) | Modify | Strong evidence that speed drives severity; mapping breakpoints/scale should be calibrated and explicitly treated as severity proxy, not purely strike likelihood. citeturn13view0turn17view0 | Medium-High | Phase 0 |
| Speed weight (0.60) | Modify | Speed dominance may be defensible for expected harm, but risks misinterpretation as strike probability; consider separating likelihood vs severity internally or reweighting after calibration. citeturn17view0turn5view0 | Medium | Phase 1 (or Phase 0 if lightweight) |
| TrafficFactor tiers / AADT curve | Modify | AADT is well-supported for likelihood; tier thresholds and curve shape are unspecified; missingness handling (“unknown” lower-than-medium) can bias risk downward. citeturn5view0turn6view0turn13view0 | Medium | Phase 0 |
| Traffic weight (0.30) | Modify | Volume relates strongly to crash opportunity, but severity can decrease with higher AADT in some rural severity models (speed regime confounding); weight should be calibrated. citeturn17view0turn13view0 | Medium | Phase 1 |
| RailCrossings in headline RawRisk | Remove | Literature supports rail/track hazards mainly via falls and wheel–flangeway mechanics, not motor-vehicle strikes; current integration creates unit/segmentation problems and inappropriate mitigation. citeturn2search0turn14view0turn15view0 | High | Phase 0 |
| RailCrossings as hazard layer | Keep | Strong mechanism evidence (angle/flangeway); high relevance for touring; best surfaced as explicit warnings rather than blended into strike score. citeturn14view0turn2search0 | High | Phase 0 |
| SegmentMiles multiplier for continuous exposure | Keep | Segment length is a standard exposure term in crash prediction; appropriate for continuous per-distance risk contributors. citeturn5view0turn6view0 | High | Phase 0 |
| SegmentMiles applied to discrete events (rail) | Modify | Discrete events should not scale with segment length; creates segmentation dependence. citeturn2search0turn5view0 | High | Phase 0 |
| InfraFactor multipliers (0.25/0.40/0.70/0.90/1.0) | Modify | Directionally supported, but magnitude varies widely across studies and contexts; long-distance conditions differ from urban CMF datasets; ensure conservative, context-appropriate mapping. citeturn16view0turn2search1turn4view0 | Medium | Phase 0 (conservative tuning), refine Phase 1 |
| High-speed infra floor (min 0.50 at ≥40 mph) | Modify | Speed strongly increases severity, but a hard floor can reduce separation credit where it may matter most; should be regime-based (crossings/access) not only speed. citeturn13view0turn0search2turn4view0 | Medium | Phase 1 (quick mitigation in Phase 0: soften or gate) |
| “Safe paths = 0 risk” shortcut | Modify | Separated/off-street facilities reduce injury risk but not to zero if crossings exist; underreporting off ROW complicates claims. citeturn16view0turn13view0 | Medium | Phase 0 |
| ShoulderFactor (0.85/0.72, speed>25) | Modify | Strong relevance on rural/high-speed roads; interaction with speed limit matters; numeric values need calibration; threshold is policy choice. citeturn17view0turn13view0 | Medium | Phase 0 (keep, document, calibrate later) |
| Legacy additive shoulder credit | Remove | Double counts shoulder mitigation versus multiplicative factor and violates “combine CMFs carefully” guidance. citeturn7view0 | High | Phase 0 |
| LeftTurnPenalty (additive, capped per segment) | Modify | Turning conflicts are real crash types, but uniform penalties ignore intersection context and segmentation can bypass cap; reframe as high-risk crossing events with gating. citeturn1search3turn1search7turn13view0 | Medium | Phase 0 |
| LaneCountFactor inside left-turn penalty | Modify | Lane/road width features appear in route safety severity models; but definition of “lane count” is unspecified and may not match lanes crossed; risk of proxy distortion. citeturn12view0turn6view0 | Medium | Phase 0 clarify; Phase 1 refine |
| Route rollup: average RiskPerMile | Modify | Average-per-mile supports comparing routes, but washes out short dangerous sections; needs peak-risk companion metric. citeturn13view0turn0search2 | High | Phase 0 |
| Logistic normalization parameters (midpoint=2.5, slope=1.4) | Modify | Logistic mapping is defensible as a presentation layer but must be calibrated to real distributions; otherwise arbitrary. citeturn12view0turn5view0 | Medium | Phase 1 (Phase 0: label as relative index) |
| RPM<0.05 ⇒ score forced to 100 | Remove | Creates false-perfect outputs via dilution and contradicts “short dangerous sections matter.” citeturn13view0turn0search2 | High | Phase 0 |
| Heavy vehicle proxy (truck route / % trucks) | Defer | Strong candidate for severity modeling; requires data pipelines. citeturn12view0turn10search0 | Medium | Phase 1 |
| Driveway/access density | Defer | Supported in ped/bike crash occurrence modeling; data availability varies. citeturn6view0turn5view0 | Medium | Phase 1 |
| Rumble strip / effective shoulder usability | Defer | FHWA guidance highlights bicyclist accommodation issues; would improve shoulder realism but needs data. citeturn19view0 | Medium | Phase 1 |
| Bicycle volume / popularity proxy | Defer | “Safety in numbers” supported but difficult to implement without bias; needs careful causal framing. citeturn9search11turn9search15 | Low-Medium | Phase 1+ |
| Weather/light/time-of-day | Keep out of headline (conditions layer) | Evidence shows darkness/fog increase severity, but these should remain in conditions indices per constraints and pre-ride planning semantics. citeturn17view0turn13view0 | High | Separate conditions layer (Phase 0) |



---

## Source File: docs/assessments/ass-004-evidence_first_based_safety_models_2026_03_28.md

# Evidence-first pressure-test plan for Lanterne’s vehicle-strike Safety Score

## Executive summary

A pre-ride Safety Score that targets **(a) the relative likelihood of a bicyclist being struck by a motor vehicle and (b) expected injury severity** can be pressure-tested against a fairly mature body of **crash prediction** practice—especially the U.S. **Highway Safety Manual**–style paradigm (SPFs + CMFs + calibration) and the newer U.S. national work that extends it specifically to pedestrians and bicyclists. citeturn3view1turn16view0turn2search11

For *turns/intersections*, the literature is more fragmented: there are strong **crash-typing** frameworks that describe turning conflicts (useful for structuring a turn-risk module), but fewer **widely adopted predictive models** that estimate turning-crash likelihood using turning counts at scale. citeturn3view3turn17search0turn24view8 The most defensible “gold standard” foundation for Lanterne’s use case (long-distance, often rural; pre-ride; vehicle-related) is to anchor on **HSM2-era pedestrian/bicycle predictive methods derived from NCHRP Project 17-84 / published as NCHRP Research Report 1064**, because it explicitly includes (i) models for **bicycle movements along road segments** and (ii) **bicycle movements through intersections**, and it also includes pathways for contexts where bicycle exposure data are missing—one of the biggest practical blockers for route-scale scoring. citeturn16view0turn2search9turn3view1turn25view0turn24view0

A practical pressure-test plan, therefore, is to evaluate Lanterne’s eventual scoring formula against:  
(1) **HSM-style structure** (SPF base + multiplicative CMFs; explicit calibration; no post-crash blame framing), citeturn2search11turn0search21  
(2) **NCHRP 1064 / 17-84 bicycle segment + intersection methods** (including the risk-based, iRAP-derived variant for data-sparse environments), citeturn3view1turn24view0turn25view1  
(3) **turn/conflict-specific evidence** (crash type taxonomies like PBCAT; the best-available turning-specific intersection studies), citeturn3view3turn24view2turn24view8  
(4) a **severity layer** anchored on empirically supported speed–injury relationships (to keep “severity” from devolving into generic “difficulty” or “stress”). citeturn10search2turn11search8

This report catalogs the most relevant models and proposes a **Phase 0 / Phase 1 reference architecture** for Lanterne that keeps the headline Safety Score narrowly vehicle-strike likelihood + severity, while pushing “conditions” and “ride reality” into separate indices as you requested.

## Authoritative turn and intersection models Lanterne should benchmark against

**Evidence: crash-typing as the backbone for “turn-based” structure (not prediction by itself).**  
The **Pedestrian and Bicycle Crash Analysis Tool** is explicitly designed for *crash typing*—creating structured descriptors of crashes between motorists and non-motorists, including maneuver/scene context. citeturn3view3turn0search38 This matters to Lanterne because “left turn risk” is not a single physical phenomenon; it’s multiple conflict families (e.g., left-turn across a bicyclist’s path; opposing-direction conflicts; multi-threat scenarios). PBCAT’s value is that it gives a defensible taxonomy for what your “turn module” is trying to approximate pre-ride. citeturn3view3turn0search30

**Evidence: intersection “movement” models that explicitly include turning conflicts exist, but are not universal standards.**  
A clear example of explicitly turn-typed modeling is **Wang & Nihan (2004)**, which classifies bicycle–motor vehicle (BMV) crashes at signalized intersections into three motor-vehicle movement types (**through**, **left-turn**, **right-turn**) and estimates risk models using negative binomial regression tied to the relevant flows. citeturn18view0turn24view8 This is important for Lanterne’s pressure test because it demonstrates a defensible framing: **turn risk should be modeled as conflict between specific flows**, not as a generic “turn penalty.” citeturn18view0turn24view8  
Limitation: it’s based on Tokyo data and an environment where bicycles often operate in channels adjacent to pedestrians, so direct parameter transfer is not guaranteed. citeturn18view0turn24view8

**Evidence: FHWA’s intersection safety index approach exposes which turn-related variables repeatedly matter.**  
FHWA’s **Hazard Index / Intersection Safety Index** work (Carter et al., 2006) provides “Bike ISI” models for **through**, **right turn**, and **left turn** movements with variables including main-road traffic volume, a high-speed indicator, the presence of turning-vehicle traffic across the bicyclist path, right-turn lanes interacting with bike lanes, signalization interacting with absence of bike lanes, and “lanes to cross” proxies. citeturn24view2turn9view0 Even though the ISI is not a pure HSM SPF, it is directly useful as: (a) an independent cross-check on which inputs are plausible, and (b) a sanity test for Lanterne’s eventual “turn module” and whether it is missing key interaction terms. citeturn24view2turn9view0

**Evidence: FHWA CMF studies provide intersection and segment treatment effect sizes (CMFs), including bicycle-relevant intersection treatments.**  
FHWA’s Evaluation of Low-Cost Safety Improvements pooled-fund research includes recent work estimating crash modification factors for **bicycle intersection treatments**. **Avelar et al. (2023)** reports statistically significant CMFs for some treatments in some datasets (e.g., separated bicycle lanes at intersections in Texas for certain crash subsets), while also showing many estimates are statistically insignificant—highlighting uncertainty and context sensitivity. citeturn24view3turn19view2turn1search14  
For segments, FHWA’s separated bike lane CMF work (e.g., Dixon et al., 2023) finds large crash reductions associated with physical vertical separation vs traditional lanes in studied corridors. citeturn24view4turn3view2turn0search13  
This matters because Lanterne will likely treat bike facility/shoulder as reducers; CMF studies are the closest thing to “plug-in multipliers” that align with HSM practice. citeturn0search21turn24view4

**Evidence: real-world crash mix makes intersections crucial but not exclusive.**  
For context on where risk happens: U.S. fatality statistics show a substantial share of bicyclist fatalities occur **away from intersections** and a substantial share **at intersections** (e.g., in 2021: 62% not at intersections; 29% at intersections; 9% other/unknown categories). citeturn24view7turn14view0  
Implication for Lanterne: a credible Safety Score needs both a **segment model** and an **intersection/turn model**, with a route rollup that does not wash out short, high-risk points. *(Inference; supported by the fatality distribution evidence.)* citeturn24view7

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["bicycle left hook crash diagram","motorist left turn facing bicyclist diagram","right hook bicycle crash diagram","bicycle intersection conflict diagram turning vehicle"],"num_per_query":1}

## Bicycle SPFs and intersection SPFs most relevant to a pre-ride route Safety Score

**Evidence: HSM2/NCHRP 1064 is the closest thing to a U.S. “gold standard” for bicycle crash prediction across facility types.**  
NCHRP Project **17-84**, published as **NCHRP Research Report 1064**, developed pedestrian and bicycle safety performance functions and related predictive methods intended for incorporation into the second edition of the Highway Safety Manual, explicitly including roadway segments and intersections and explicitly recognizing the need to incorporate pedestrian/bicycle exposure data where available. citeturn2search9turn3view1turn24view0 The HSM2 update materials indicate pedestrian and bicycle crash prediction methodology is placed in chapters for **rural two-lane**, **rural multilane**, and **urban/suburban arterials**, and explicitly distinguishes **bicycle movements along the road** and **through intersections**—exactly aligned to Lanterne’s long-distance use case (big rural share) and your narrow “vehicle strike likelihood + severity” target. citeturn16view0turn24view0

**Evidence: NCHRP 17-84/1064 includes a “risk-based” (iRAP-derived) approach for data-sparse environments that is unusually relevant to route scoring.**  
The NCHRP 17-84 final report describes three approaches, including adapting crash prediction models used by the U.S. Road Assessment Program (usRAP) and developing models for estimating crash potential when pedestrian/bicycle exposure data are missing. citeturn3view1turn15view0 For bicycle movements through intersections, the risk-based structure explicitly decomposes prediction into factors for likelihood and severity, including **motor-vehicle traffic speed factor**, **motor-vehicle traffic flow factor**, and bicycle flow factor, plus adjustment factors tied to intersection characteristics (e.g., intersection type; advance visibility; channelization). citeturn24view0turn25view0turn25view1  
This factorized form is practically important for Lanterne because it resembles a scoring model that can run on a large network with incomplete data—without pretending to be a statistically calibrated local SPF everywhere. *(Inference; grounded in the factorized model structure and the exposure-data barrier discussed in the report.)* citeturn15view0turn24view0

**Evidence: many strong bike-infrastructure safety studies measure injury risk, but are not SPFs and may include non-MV events.**  
Case-crossover and comparative risk studies (e.g., Teschke et al. 2012; Lusk et al. 2011) consistently find lower injury risk on facilities with greater separation from motor vehicles and highlight intersection/driveway complexity as an important modifier. citeturn11search3turn11search1turn13view0 However, because these studies often include broader injury mechanisms (including falls) and are typically urban, they should be used as **supporting evidence for directionality and facility classification**, not as the core quantitative engine for a vehicle-strike SPF. *(Inference; consistent with their study designs and settings.)* citeturn11search3turn11search1

## What inputs matter for turn risk and SPFs, and how strong is the evidence

This section is intended as the **pressure-test checklist** you will later apply to Lanterne’s actual inputs and math.

**Motor-vehicle volume / exposure (AADT, per-lane AADT, major/minor AADT).**  
Evidence is strong that motor-vehicle volume appears as exposure in intersection and segment crash models and in the most credible applied safety frameworks. For example, Avelar et al. model intersection crashes with terms involving major/minor ADT, and NCHRP 17-84/1064’s risk-based approach includes motor-vehicle traffic flow factors as a function of AADT ranges. citeturn19view2turn24view0turn25view1  
Strength rating for Lanterne: **Strong** (likelihood).

**Speed environment (posted or operating speed; speed factor).**  
Speed enters both (a) severity mechanisms (kinetic energy) and (b) likelihood mechanisms (stopping distance, driver workload). Carter et al.’s Bike ISI uses a “main street speed limit ≥ 35 mph” indicator in multiple movement models. citeturn24view2turn9view0 The NCHRP risk-based bicycle intersection method explicitly uses a motor-vehicle traffic speed factor and gives special handling for stop-controlled approaches (use ≤20 mph). citeturn25view1turn15view2 Severity evidence is also strong: bicycle crash severity models find speed limit is associated with higher injury severity, and impact-speed work (even if pedestrian-focused) provides empirically anchored curves that can support a severity-weighting layer in a bicycle Safety Score. citeturn11search8turn10search2  
Strength rating: **Strong** (severity) and **Moderate-to-strong** (likelihood, depending on whether you can approximate operating speed).

**Turning volumes and turn-lane geometry (right-turn lanes; left-turn protection; channelization; RTOR, protected/permitted phasing).**  
The most direct “turn risk” models tie crashes to turning-related flows: Wang & Nihan explicitly relates crash risk for left-turn/right-turn crash types to the related flows and uses movement-specific modeling. citeturn18view0turn24view8 Carter et al.’s Bike ISI includes explicit terms for turning-vehicle traffic, right-turn lanes, and bicyclist “lanes-to-cross” proxies in right/left turn movement models. citeturn24view2turn24view8 NCHRP 17-84’s data inventory explicitly includes left/right turn lane presence and right-turn channelization and right-turn operation (including RTOR permitted/prohibited). citeturn26view0turn26view1  
Strength rating: **Strong in mechanism**, **Moderate in scalable implementation**, because turning counts and signal phasing are often unavailable network-wide. *(Evidence for mechanisms; inference for scalability.)* citeturn26view1turn18view0

**Lanes-to-cross / crossing distance / number of legs / lane count.**  
Exposure time/distance through conflict zones is repeatedly used as a predictor or proxy. Carter et al. includes “number of traffic lanes for cyclists to cross” variables in turn movement models. citeturn24view2turn9view0 NCHRP 17-84’s intersection inventory includes crossing distance (curb-to-curb; and adjusted for refuge/channelizing islands) and intersection configuration (3-leg vs 4-leg; signalized vs stop-controlled). citeturn26view0turn26view1  
Strength rating: **Moderate-to-strong** (likelihood), especially when turning volumes are missing and you need geometric proxies.

**Bike infrastructure presence/type and separation quality.**  
Large bodies of evidence and multiple official analyses conclude that separation tends to reduce bicyclist crash risk, but intersection/driveway complexity can offset benefits if not addressed. NTSB explicitly notes separated bike lanes can reduce MV-involved bicycle crashes but require crossing vehicle traffic at intersections/driveways, making risk dependent on intersection/driveway frequency and facility configuration (street level vs raised; one-way vs two-way). citeturn24view6turn13view0 FHWA CMF work finds substantial crash reductions associated with vertical separation vs traditional bike lanes in studied datasets. citeturn24view4turn0search13  
Strength rating: **Strong directionally**, **Moderate quantitatively** (effect size depends on context, facility subtype, and intersection treatment quality).

**Shoulder width / paved shoulder availability.**  
Shoulder width is plausibly protective on higher-speed facilities (space, recovery area, separation), but evidence is mixed depending on whether the model targets frequency vs severity. Severity work on two-lane roads found an interaction of speed limit and shoulder width, with results varying by context. citeturn11search8 NCHRP 17-84 explicitly treats shoulder width as a collected roadway attribute and distinguishes it from bike lanes/parking in official roadway inventory contexts (important for data definitions). citeturn21view0turn22search10  
Strength rating: **Moderate** (likelihood and severity), higher on rural/high-speed segments, weaker in dense urban grids where intersections dominate. *(Inference; supported by where shoulder is meaningful as separation.)*

**Access density / driveways and intersection frequency (conflict opportunities).**  
NTSB and IIHS both point to intersection/driveway frequency and “complexity” as meaningful determinants of risk on otherwise separated facilities. citeturn24view6turn10search3 NCHRP 17-84 also treats intersection/channelization visibility and related attributes as explicit likelihood adjustment factors in the risk-based model. citeturn25view0turn25view1  
Strength rating: **Moderate-to-strong** (likelihood), with the practical challenge being proxy measurement from open data. *(Evidence for importance; inference for proxying.)* citeturn10search3turn25view0

## Candidate gold-standard model and how to map it to open-data reality

### Verdict on “gold standard” for Lanterne’s benchmarking set

**Evidence:** Among U.S.-oriented, practitioner-facing methods, **NCHRP Research Report 1064 / NCHRP 17-84** as implemented in the HSM2 ecosystem is the most credible benchmark because it (a) is designed for predictive safety estimation (not blame), (b) provides both segment and intersection models for bicyclist crashes, (c) explicitly addresses the lack of pedestrian/bicycle exposure data, and (d) spans rural and urban facility types. citeturn16view0turn3view1turn15view0turn24view0  
**Inference:** For a long-distance route-scoring product, the risk-based, factorized “RAP/iRAP-derived” variant in NCHRP 17-84 is particularly suitable as a *Phase 0* benchmark because it can be approximated with speed and AADT bins and a limited set of intersection quality factors, while retaining a defensible safety-theory foundation. citeturn3view1turn25view0turn25view1

### Plain-English step-by-step mapping of the NCHRP 17-84/1064 risk-based bicycle intersection model

Below is a “plain English” decomposition specifically for **bicycle movements through intersections** (the most turn-relevant part). (This is not yet Lanterne’s formula; it is the benchmark model you will pressure-test against.)

**Evidence (model structure):** NCHRP 17-84 decomposes bicycle intersection crash prediction into factors for likelihood and severity, including a motor-vehicle speed factor and a motor-vehicle flow factor (AADT-based), plus adjustment factors for intersection qualities such as advance visibility and channelization. citeturn24view0turn25view0turn25view1

**Step-by-step (benchmark workflow):**
1. **Classify the intersection type** (3-leg vs 4-leg; signalized vs stop control; exclusive left-turn lane presence) and identify major vs minor road. citeturn24view0turn25view1  
2. **Estimate baseline conflict exposure from motor-vehicle flow** using AADT-per-lane bins for the *side road entering the intersection* (this is the “motor-vehicle traffic flow factor”). citeturn25view1  
3. **Estimate severity contribution from speed environment** (motor-vehicle traffic speed factor), using mean/operating speed or a defensible proxy; apply stop-control handling (≤20 mph assumption on the stop-controlled road being analyzed). citeturn25view1turn15view2  
4. **Apply intersection quality likelihood adjustments** such as:
   - **Advance visibility of the intersection** (e.g., “limited” vs “substantial,” with example factor 1.20 vs 1.00). citeturn25view0turn15view3  
   - **Intersection channelization present/not present** (example factor 1.00 vs 1.20). citeturn25view1turn15view3  
5. **Apply intersection-type severity adjustment** (e.g., different severity adjustment factors by intersection type and exclusive left-turn lane presence). citeturn25view1  
6. Combine the above to produce an intersection-level expected relative risk (or predicted crashes, depending on implementation), then repeat as needed for major-road and minor-road bicycle movements. citeturn24view0turn25view1

### Required data fields vs open-data availability (OSM/HPMS/state DOT AADT)

The NCHRP 17-84 study explicitly enumerates intersection inventory fields including: turn lanes and turn operations (RTOR), channelization, lighting, bicycle facility type, parking, crosswalk control, and crossing distance. citeturn26view0turn26view1

**Availability assessment (Evidence + inference):**
- **Available (often) from OSM:** posted speed limit (`maxspeed`), lane count (`lanes`), turn lane indications (`turn:lanes`), traffic signals (`highway=traffic_signals`), stop control (`highway=stop`), cycling facility tags (`cycleway=*`), shoulder tagging in some regions (`shoulder=*`), and generic width tags in limited cases. citeturn23search0turn23search12turn23search1turn22search3turn22search5turn22search2turn23search2  
- **Available (for many U.S. road segments) from HPMS:** AADT, speed limit, shoulder widths, and turn lanes exist as defined HPMS fields. citeturn22search0turn22search14turn22search10  
- **Often unavailable or incomplete in open national sources:** intersection signal phasing details (protected/permitted lefts; RTOR prohibited), turning movement counts, and consistent intersection channelization/sight-distance quality. citeturn26view0turn26view1  
- **Often available via state/local DOT open data (patchy):** AADT layers, sometimes intersection counts/controls; but not consistently standardized across states. *(Inference; “patchy” is an implementation reality rather than a single-source fact.)*

### Pragmatic proxies when fields are unavailable (to keep Phase 0 feasible)

These proxies are not “ideal”; they are designed to keep a pre-ride Safety Score operational while remaining scientifically defensible.

- **Turning volumes absent → proxy with geometry + functional class + AADT:** Use AADT on major/minor legs plus “lanes-to-cross,” number of legs, and presence of turn lanes as partial surrogates; this is consistent with the way Carter et al.’s Bike ISI uses turning-vehicle presence and lanes-to-cross concepts. citeturn24view2turn26view0  
- **Operating speed absent → proxy with posted speed limit + control type:** NCHRP risk-based method’s stop-control handling demonstrates that control type can justify a speed proxy at intersections. citeturn25view1  
- **Signal phasing absent → proxy with intersection class + turn-lane presence + urbanicity:** If you cannot observe protected/permitted phasing, treat protected left-turn presence as “unknown” and avoid hard penalties; instead, model increased risk primarily through AADT, speed, and multi-lane crossing exposure, which are measurable. *(Inference; consistent with avoiding overconfident penalties without data.)*  
- **Bicycle exposure absent → proxy with modeled bicycle flow or omit and treat score as relative:** NCHRP 17-84 explicitly treats exposure-data scarcity as a core barrier and provides methods intended for absence of pedestrian/bicycle exposure. citeturn15view0turn3view1

## Recommended hybrid architecture for Phase 0 and Phase 1 benchmarking

### Evidence-first hybrid design principle

**Evidence:** HSM practice is structurally “base SPF × (product of CMFs) × calibration factor,” and FHWA defines CMFs as multiplicative factors intended to compute expected crash changes after implementing countermeasures. citeturn2search11turn0search21  
**Inference:** Lanterne can pressure-test whether its future scoring logic respects this core shape—even if Lanterne’s output is a normalized Safety Score rather than predicted crashes—by verifying that “risk reducers” behave more like **multiplicative mitigations** than unconditional additive credits, especially when multiple mitigations stack (e.g., shoulder + bike lane + separation). *(This is a design-test hypothesis for later, not a requirement today.)*

### Suggested benchmark hybrid (what to compare Lanterne against)

**Phase 0 benchmark hybrid (data-feasible, defensible):**
- **Base segment + intersection risk:** NCHRP 17-84/1064 risk-based bicycle methods (RAP/iRAP-derived) for:
  - bicycle movements along segments  
  - bicycle movements through intersections citeturn3view1turn24view0turn25view1  
- **Key CMF overlays (where reliably detectable):**
  - separated bike lanes / vertical separation using FHWA CMFs as a check on effect size directionality citeturn24view4turn0search13  
  - intersection bicycle treatment CMFs where applicable and detectable (recognizing many are statistically insignificant) citeturn24view3turn19view2  
- **Severity weighting:** Speed-driven severity curves (supported by severity models and impact-speed evidence) to keep the Safety Score’s “severity” component anchored in physics and empirics, not “difficulty.” citeturn11search8turn10search2

**Phase 1 benchmark hybrid (higher fidelity, data-heavier):**
- Replace or supplement risk-based factors with **jurisdiction-calibrated SPFs** where bicycle exposure data exist (e.g., ADBT or modeled bike volume), consistent with NCHRP 17-84’s negative binomial SPF development approach. citeturn3view1turn15view0turn19view2  
- Add a **turn/conflict submodule** inspired by turning crash-type models (Wang & Nihan) and movement-specific indices (Carter et al.), but only where turning volumes / lane assignment / intersection control are reliably observable. citeturn18view0turn24view2

### Mermaid diagrams: benchmark architecture and route rollup logic

```mermaid
flowchart TB
  A[Route polyline] --> B[Segmentization<br/>fixed length or topology-based]
  B --> C1[Segment risk module<br/>NCHRP 17-84/1064 risk-based]
  B --> C2[Intersection risk module<br/>bicycle movements through intersections]
  C2 --> D1[Turn/Conflict proxy layer<br/>lanes-to-cross, turn lanes, control type]
  C1 --> E[Apply CMF overlays where detectable<br/>e.g., separated facility]
  D1 --> E
  E --> F[Severity weighting<br/>speed-based]
  F --> G[Route rollup<br/>risk aggregation + hotspot protection]
  G --> H[Headline Safety Score<br/>relative likelihood × expected severity]
```

```mermaid
flowchart LR
  A[Per-segment risk values] --> B[Identify hotspots<br/>top X% or above threshold]
  A --> C[Distance-weighted mean risk]
  B --> D[Hotspot penalty / non-linear aggregation]
  C --> E[Base route score]
  D --> E
  E --> F[Final route Safety Score]
```

*(These diagrams are a planning scaffold; the exact rollup math should be pressure-tested later against “short dangerous section washout,” but the core requirement—hotspot protection—is motivated by the fact that fatal risks occur both at intersections and non-intersection locations.)* citeturn24view7

## Implementation checklist and prioritized source set for Lanterne’s later pressure test

### Phase 0 checklist (what you can benchmark immediately once Lanterne’s formula arrives)

- Build a **benchmark input dictionary** that maps each Lanterne input to the closest corresponding variable in:
  - NCHRP 17-84/1064 risk-based factors for segments and intersections citeturn24view0turn25view1  
  - FHWA Bike ISI movement models (turn-related input cross-check) citeturn24view2  
  - PBCAT crash-typing categories (to validate “left turn” or “turn risk” mappings) citeturn3view3turn0search30  
- Validate that Lanterne’s eventual model accounts for the empirical reality that both intersection and non-intersection locations matter (to guard against intersection-only scoring). citeturn24view7  
- Decide on **data feasibility tiers** for each candidate input:
  - Tier A: OSM/HPMS-supported at scale (e.g., speed limit, lane count, AADT where available) citeturn22search0turn23search0turn23search12  
  - Tier B: partially available (bike facility type, shoulder tagging) citeturn22search2turn23search2  
  - Tier C: mostly unavailable without proprietary/local feeds (turning volumes, signal phasing) citeturn26view1  
- Establish a **severity mapping** (speed → severity weight) that is separate from difficulty/effort and is rooted in empirical severity relationships. citeturn11search8turn10search2  

### Phase 1 checklist (what to defer until you choose data pipelines)

- If you plan to include genuine turn-flow modeling, build pipelines for:
  - intersection turning counts where available  
  - signal control/phasing attributes (RTOR prohibited, protected lefts) citeturn26view1  
- Implement jurisdiction-specific calibration pathways (consistent with HSM practice) where you have crash datasets and inventory alignment. citeturn2search11turn16view0  
- Add treatment-effect overlays only where Lanterne can reliably detect facility subtype (e.g., vertical separation vs buffered) because CMFs are often treatment-definition sensitive. citeturn24view4turn24view3  

### Priority reference set to cite during the later formula pressure-test

Highest priority (benchmark backbone):
- NCHRP 17-84 / NCHRP Research Report 1064 (HSM2 bicycle segment + intersection methods; exposure-data scarcity; risk-based factors). citeturn2search9turn3view1turn24view0turn25view1  
- HSM2 update materials (where methods live; required ped/bike movement counts; outputs). citeturn16view0  

Turn/intersection structure:
- FHWA Hazard Index / Bike ISI movement models (through/right/left movement models and inputs). citeturn24view2turn9view0  
- Wang & Nihan (turn-typed intersection crash risk models using flows). citeturn18view0turn24view8  
- PBCAT 3 user guide + FHWA crash type listings (turn-conflict taxonomy). citeturn3view3turn0search30  

Treatment effect sizes (CMFs):
- FHWA CMF work on separated bike lanes (segment-level) and bicycle treatments at intersections. citeturn24view4turn24view3turn0search21  

Severity anchoring:
- Klop & Khattak injury severity modeling (speed limit, grades, lighting, etc.). citeturn11search8  
- Tefft impact speed vs severe injury/death risk (useful to ground severity weighting). citeturn10search14turn10search2  

Data feasibility:
- FHWA HPMS Field Manual (AADT, speed limit, shoulder widths as standardized fields). citeturn22search0turn22search14turn22search10  
- OSM tagging references for speed, lanes, cycleway, turn lanes, stop/signal control. citeturn23search0turn23search12turn22search2turn23search1turn22search5turn22search3  

## Model comparison table for Lanterne’s benchmark set

| Model/Study | Purpose (frequency vs severity) | Key inputs (examples) | Data needs | Strengths | Weaknesses | Reproducible with OSM/HPMS/AADT? |
|---|---|---|---|---|---|---|
| NCHRP 17-84 / published as NCHRP Research Report 1064; HSM2 ped/bike methods | Frequency + severity-oriented predictive methods for ped/bike; includes bicycle along-road and through-intersection models citeturn3view1turn16view0 | AADT/flow factors; speed factors; intersection type; adjustment factors for visibility/channelization; facility type factors citeturn24view0turn25view0turn25view1 | Roadway inventory + motor-vehicle volumes; ideally bike volumes, but provides approaches when exposure data missing citeturn15view0turn24view0 | Most “standard practice”–aligned foundation for predictive scoring; spans rural + urban; explicitly addresses exposure-data scarcity citeturn15view0turn16view0 | Some components reference iRAP-derived factors and require judgmental ratings (e.g., “advance visibility”), which are hard to observe at scale citeturn25view0turn25view1 | **Partial**: AADT via HPMS/state DOT; speed/lanes/control/bike infra partly via OSM; visibility/channelization often not directly available citeturn22search0turn23search0turn26view0turn25view0 |
| FHWA Carter et al. (2006) Hazard Index / Bike ISI movement models | Movement-specific intersection risk index (through/right/left), not an HSM SPF citeturn24view2 | Main ADT; speed≥35 indicator; turning-vehicle presence; right-turn lanes×bike lane; cross ADT; signal×no bike lane; lanes-to-cross proxies; parking citeturn24view2turn24view8 | Intersection inventory; ADT; some movement proxies; bike facility presence | Explicitly turn/movement-oriented; exposes interaction terms that are easy to unintentionally omit in scoring models citeturn24view2 | Not designed as a universally calibrated crash prediction SPF; may not transfer cleanly outside studied contexts citeturn9view0 | **Partial**: ADT needed; bike lanes/lanes/signalization often in OSM; “turning-vehicle presence” is hard without turning volumes citeturn22search2turn22search3turn24view2 |
| Wang & Nihan (2004) signalized intersection BMV risk models | Frequency by crash type (BMV-1 through, BMV-2 left, BMV-3 right) citeturn18view0turn24view8 | Related vehicle flows + bicycle flows; geometry + control variables, with type-specific variable sets citeturn18view0 | Turning-related flow data; bicycle flows; signalized intersection approach-level inventory citeturn18view0 | Best demonstration that “turn risk” should be flow-conflict-specific (left/right/through) citeturn24view8turn18view0 | Non-U.S. context and specific operational assumptions; turning/bike flow data rarely available network-wide citeturn18view0 | **Usually no (at scale)**: turning flows and bicycle flows not generally available from OSM/HPMS citeturn18view0turn22search0 |
| FHWA PBCAT 3 | Crash typing (taxonomy), not prediction citeturn3view3turn0search38 | Crash circumstances/typologies (turning conflicts, overtaking, crossing paths, etc.) citeturn0search30turn3view3 | Crash report details (post-crash) | Creates defensible mapping from “left turn” to concrete crash types; supports model validation and interpretability citeturn3view3turn0search30 | Not a predictive model; cannot output pre-ride risk without pairing to SPFs/CMFs citeturn3view3 | **N/A** (tool supports analysis, not predictive scoring) citeturn3view3 |
| FHWA-HRT-23-020 (Avelar et al., 2023) CMFs for bicycle intersection treatments | Treatment effect estimation (CMFs), incl. FI and non-weather crash subsets citeturn24view3turn19view2 | Exposure terms (e.g., log ADBT); signalization; lanes/turn lanes; bicycle treatment indicators; outputs CMFs citeturn19view2turn24view3 | Crash data + intersection inventory + bicycle exposure (counts or modeled ADBT) citeturn19view1turn19view2 | Provides real effect-size priors for “infrastructure as reducer”; shows uncertainty and context dependence (many insignificant CMFs) citeturn24view3turn19view2 | CMFs are treatment-definition specific; bicycle exposure still a major barrier; not an SPF baseline by itself citeturn19view1turn19view2 | **Partial**: treatments may be detectable from OSM; bicycle exposure generally not; intersection inventory partly available citeturn22search2turn19view2 |
| FHWA-HRT-23-078 (Dixon et al., 2023) separated bike lane CMFs (segments) | Segment treatment effect estimation (CMFs) citeturn24view4 | Facility type; separation type (vertical elements vs buffered/traditional); bicycle crash counts citeturn24view4turn3view2 | Corridor/segment inventory + crash data + (ideally) exposure | Strong quantitative evidence of reduced bicycle crashes with vertical separation in studied datasets citeturn24view4turn3view2 | City-specific samples; transferability depends on matching facility design and context citeturn3view2 | **Often partial**: facility type may be inferred from OSM; exposure/crash data not available globally citeturn22search2turn24view4 |
| NTSB SS-19/01 bicyclist safety report | Risk factors + countermeasure synthesis (not an SPF) citeturn13view0turn10search0 | Emphasizes separation, speed management; highlights intersection/driveway crossing as key modifier for separated facilities citeturn24view6turn24view5 | Literature + crash data synthesis | Strong framing for vehicle-related safety and for why “intersection complexity” matters even with separation citeturn24view6turn13view0 | Not a predictive equation; needs pairing with SPFs/CMFs citeturn13view0 | **Yes for principles; no for direct scoring** citeturn13view0 |
| Klop & Khattak (1999) bicycle crash severity model | Severity (ordered probit) citeturn11search8 | Speed limit; grade/curvature; lighting/darkness; AADT interactions; rural vs urban differences citeturn11search8 | Police crash data + roadway/environment variables | Useful to keep Severity Score anchored in measurable roadway/speed context rather than “difficulty” citeturn11search8 | Older data; severity drivers can vary by jurisdiction and vehicle fleet; not a frequency model citeturn11search8 | **Partial**: speed limit/lanes via OSM; AADT via HPMS; lighting/grade may be incomplete depending on data source citeturn22search0turn23search0turn11search8 |
| NHTSA CrashStats (bicyclist fatalities location context) | Descriptive (context distribution) citeturn24view7 | Intersection vs non-intersection shares; urban/rural shares; light condition shares citeturn24view7 | National fatality data | Sets guardrails: Safety Score needs both segment and intersection components; supports hotspot-aware rollup citeturn24view7 | Not predictive; fatality-only lens (no injury-only crashes) citeturn24view7 | **N/A** (benchmark context, not scoring equation) citeturn24view7 |
| HPMS Field Manual (FHWA) | Data specification (enabler) citeturn22search0turn22search14 | Standard fields: AADT, speed limit, shoulder widths, turn lanes citeturn22search0turn22search10 | HPMS inventory submissions | Defines at-scale U.S. data fields for key risk variables; supports feasibility triage citeturn22search0turn22search10 | Still not global; intersection-level operational details not covered well at scale citeturn22search0turn26view1 | **Yes (U.S. segments)**; intersections still partial citeturn22search0turn26view0 |
| OpenStreetMap tagging references | Data specification (enabler) citeturn23search0turn23search12turn22search2 | Speed limit tags (`maxspeed`), lanes (`lanes`), turn lanes (`turn:lanes`), signals/stops, cycleway tagging citeturn23search0turn23search1turn22search3turn22search5turn22search2 | OSM extracts | Supports global-ish, open extraction of many needed fields; crucial for Phase 0 approximations citeturn23search0turn22search2 | Completeness and consistency vary geographically; AADT generally absent citeturn22search0turn23search0 | **Partial-to-good** for geometry/control/facility tags; **weak** for volumes citeturn23search12turn22search0 |

---

## Source File: docs/migrations/2026-03-21-canonical_boostrap.md

# Canonical Route Bootstrap Migration
Date: 2026-03-22
Status: Complete

## Corpus Snapshot
- 3151 routes imported from RWGPS via external_route_catalog
- 3151 canonical routes populated with geometry and fingerprints
- 0 duplicates generated
- 0 unmatched routes
- Perfect 1:1 assignment achieved

## Geometry Fingerprint Strategy

### Algorithm
Routes are fingerprinted by:
1. Normalizing geometry direction — if ST_StartPoint < ST_EndPoint 
   lexicographically, use as-is; otherwise ST_Reverse()
2. Resampling to 100 evenly-spaced points via ST_LineInterpolatePoints(geom, 0.01)
3. Rounding each coordinate to 5 decimal places
4. Serializing as pipe-delimited lon,lat string
5. SHA-256 hashing via pgcrypto digest()

### Why Direction Normalization
RWGPS exports and GPX files frequently represent the same corridor in 
opposite directions. Without normalization, a reversed export of the 
same route produces a different fingerprint and creates a duplicate 
canonical. Direction normalization ensures the same physical corridor 
always produces the same fingerprint regardless of travel direction.

### Why 100 Resampled Points
Raw point counts vary wildly (99–4924+ points observed in corpus).
Resampling to a fixed count before hashing ensures fingerprint stability
across routes with different GPS sampling densities. Two routes 
representing the same corridor but captured at different resolutions 
will produce matching fingerprints.

### Critical Implementation Note
Both imported_routes and canonical_routes must use identical fingerprint
formula. During bootstrap, imported_routes fingerprints were initially
computed without direction normalization, causing 1970 mismatches.
All imported fingerprints were recomputed with the correct formula
before scale testing. Mismatch count confirmed at 0 before proceeding.

## Join Key Discovery

### The Trap
organization_published_routes.imported_route_id stores the UUID of the
imported_routes.id column — NOT source_route_id (the RWGPS integer ID).

### Correct Join Path
canonical_routes
    ↑ (via canonical_route_id)
organization_published_routes
    ↓ (via imported_route_id = imported_routes.id)
imported_routes

### Why This Matters
Any future migration or ingestion code that tries to join on 
source_route_id will get zero rows. Always join on the UUID id column.

## Duplicate Discoveries

### Case 1 — A Birmingham Tour
- Two canonical shells with identical names
- Same imported_route_id, same creation timestamp
- Cause: import ran twice without idempotency guard
- Resolution: kept 6dd8937e, deleted e20d389f
- Repointed organization_published_routes accordingly

### Case 2 — Tail of the Lizard
- Two canonical shells: "Tail of the Lizard" and "Tail of the Lizard 200k"
- Same imported_route_id, same creation timestamp
- Cause: RUSA has two separate permanent entries (different Route_IDs,
  incremented sequentially) pointing to the same RWGPS route file
- Resolution: kept c34df51c (200k), deleted 2aa7e043
- Both organization_published_routes rows repointed to kept shell
- This is a valid RUSA data pattern — multiple permanents sharing
  one physical corridor. Expected to recur with other orgs.

## Schema Changes Applied
- canonical_routes: added geom geometry(LineString, 4326)
- canonical_routes: added length_m numeric
- canonical_routes: added GIST index on geom
- canonical_routes: geometry_fingerprint recomputed with 
  direction normalization
- imported_routes: geometry_fingerprint recomputed with 
  direction normalization
- imported_routes: source_platform backfilled to 'rwgps'

## Canonical Matching Function
assign_canonical_route(p_imported_id uuid) created in Supabase.

### Logic
1. Exact fingerprint match (fast path) — handles identical corridors
2. Bounding box spatial candidate search
3. Length filter — candidates within 25% of imported route length
4. Start point guardrail — candidates within 25km of start point
5. Corridor overlap — ST_Buffer(geom, 80m) intersection / route length
6. Threshold — overlap >= 0.85 assigns existing canonical
7. No match — creates new canonical shell

### Test Results
- Tested on 2 routes before scale run
- First test revealed fingerprint mismatch bug (see above)
- After fix, second test confirmed fast fingerprint path working
- Scale run: 3151/3151 matched existing canonicals, 0 new created

## Architectural Consequences
- All analysis attaches to canonical_route_id, never to imported_routes.id
- All future ingestion resolves via assign_canonical_route()
- Canonical routes represent physical road corridors, not source artifacts
- Multiple organizations may reference the same canonical route
- Sources and provenance live in organization_published_routes and 
  imported_routes, never in canonical_routes
- canonical_routes.geometry is the authoritative geometry for all 
  downstream analysis including route_slices

## Recommended Constraint
ALTER TABLE canonical_routes
ADD CONSTRAINT unique_geometry_fingerprint
UNIQUE (geometry_fingerprint);

Prevents future ingestion from creating duplicate canonical corridors.
Not yet applied — add before next ingestion run.

## Next Step
Create route_slices table per RECOMMENDED_SCHEMA_SHAPE.md.
This is the atomic analysis unit from ADR-020.
All slice generation reads from canonical_routes.geometry.


---

## Source File: docs/migrations/2026-03-22-docs_reconciliation.md

# Documentation Reconciliation Note
Date: 2026-03-22
Purpose: Record every conflict between pre-Phase-1 design docs and actual migrated database state

This document exists so future sessions, contributors, and AI assistants know which docs to trust and which sections are superseded.

---

## Authoritative sources (trust these)

1. **migration-log.md** — ground truth for what was actually run and why
2. **RECOMMENDED_SCHEMA_SHAPE.md** (updated 2026-03-22) — current table/column reference
3. Live Supabase schema query — always the final authority

---

## Conflicts and resolutions by document

---

### RECOMMENDED_SCHEMA_SHAPE.md (original)

**Status: Superseded. Replaced by updated version dated 2026-03-22.**

Conflicts:
- Used `route_id` as the FK name on slices. Actual column is `canonical_route_id`.
- Did not include hardening constraints (unique on fact tables, scoped is_current index).
- `route_slices` listed `slice_boundary_reason_json` (jsonb). Actual column is `slice_boundary_reason` (text[]).
- Did not reflect that `light_state_default` was intentionally excluded from `route_slice_osm_facts`.
- `route_analysis_summary` section did not note the canonical_route_id denormalization risk.
- Build order was correct and has been preserved.

---

### ds-005-canonical-route-schema-spec.md

**Status: Partially superseded. Core principles valid. Field names conflict.**

Conflicts:
- Spec uses `canonical_geometry` and `corridor_geometry` as field names. Actual `canonical_routes` columns are `geometry` and (corridor geometry not yet stored as a column).
- Spec lists `bounding_box`, `centroid`, `canonicalization_version`, `status` on `canonical_routes`. None of these exist in the current table.
- Spec defines `route_sources` as a separate table. Actual DB uses `imported_routes` + `external_route_catalog` + `organization_published_routes` for this function.
- Spec defines `route_variants` as a separate table. Not yet migrated.
- Spec calls the slice FK table `slice_variables`. Actual table is `route_slice_osm_facts`.
- Spec calls the analysis table `slice_analysis`. Actual table is `route_slice_analysis`.
- Spec calls the rollup table `route_rollups`. Actual table is `route_analysis_summary`.
- Spec calls the user save table `user_routes`. Not yet migrated.

What is still correct:
- The relationship model (canonical → slices → variables → analysis → rollup) is correct.
- The versioning strategy (separate versions for canonicalization, slices, OSM registry, analysis, rollup) is correct and should be implemented.
- The mutation rules and review/ambiguity handling intent is still valid.

---

### ds-007-route-slice-generation-spec.md

**Status: Principles valid. Field names partially conflict.**

Conflicts:
- Spec lists `route_id` as the slice FK. Actual column is `canonical_route_id`.
- Spec lists `osm_way_ids` as a slice field. Not present in `route_slices`. OSM traceability lives in `route_slice_osm_facts.raw_osm_tags_json`.
- Spec lists `road_class`, `surface_type`, `infrastructure_tags` directly on slices. These live in `route_slice_osm_facts`, not `route_slices`. Correct separation.
- Spec lists `analysis_version` on the slice row. This lives on `route_analysis_runs`, not individual slices.

What is still correct:
- Target slice length 200–500m is correct.
- Boundary trigger list is correct and maps to `slice_boundary_reason` values.
- Stability principle is correct — `slice_builder_version` column implements this.
- Relationship to display segments is correct.

---

### adr-020-atomic-analysis-unit

**Status: Valid. No conflicts with migrated schema.**

The decision (bounded dynamic slices, ~200–400 per 200km route, event-driven breaks) is correctly reflected in `route_slices`. No changes needed.

---

### adr-021-osm-variable-registry.md

**Status: Valid as a registry definition. Field placement note needed.**

Conflict:
- Section F (Environmental Timing Variables) lists `light_state`, `glare_flag`, wind, temperature, precipitation as OSM variables. These are NOT stored in `route_slice_osm_facts`. They belong in `ride_instance_slice_conditions` (Phase 2). The registry itself can list them, but the storage location is the ride instance layer, not the OSM facts table.

What is still correct:
- All sections A–E map correctly to `route_slice_osm_facts` columns.
- Section G (traceability fields) maps correctly.
- The observed speed hierarchy (radar > posted limit > inferred) is reflected in `car_speed_value` + `car_speed_source`.

---

### adr-022-phase-1-enum-registry

**Status: Valid. All enum values reflected in route_slice_osm_facts column definitions.**

No conflicts. The allowed values in this ADR are the authoritative source for what goes into the text columns on `route_slice_osm_facts`.

---

### adr-026-canonical-route-identity

**Status: Valid. Fully implemented.**

No conflicts with current schema.

---

### adr-031-model-multi-day-events

**Status: Valid architectural decision. Schema partially implemented.**

Current state:
- `event_routes` exists and supports ordered references onto canonical routes.
- Full ADR-031 model (`events`, `event_days`, `event_route_part_segments`) is not yet migrated.
- Do not use `event_routes` for cloverleaf multi-day events until the full model is in place.

---

### 2026-03-21-canonical-bootstrap.md

**Status: Accurate historical record. One pending item remains.**

Pending item still open:
```sql
ALTER TABLE canonical_routes
ADD CONSTRAINT unique_geometry_fingerprint
UNIQUE (geometry_fingerprint);
```
Apply before the next ingestion run.

---

### ds-001-route-intelligence-pipeline-spec.md

**Status: Valid. Pipeline stages map correctly to migrated tables.**

Stage → Table mapping (current):
- Stage 1 (Acquisition) → `imported_routes`, `external_route_catalog`
- Stage 3 (Canonical Resolution) → `canonical_routes`, `assign_canonical_route()`
- Stage 5 (Slice Generation) → `route_slices`
- Stage 6 (Stable Analysis) → `route_slice_osm_facts`, `route_slice_support_facts`, `route_analysis_runs`, `route_slice_analysis`, `route_analysis_summary`
- Stage 7–8 (Timeline + Contextual) → Phase 2 (`ride_instance_runs`, `ride_instance_slice_conditions`)
- Stage 9 (Rollups) → `route_analysis_summary`

No conflicts. Pipeline spec remains valid as written.

---

### INDICES_CALCULATION.md

**Status: Valid. No conflicts.**

Notes:
- Traffic Exposure Index and Bike Support Index are the currently implemented indices. Both map to columns on `route_slice_analysis`.
- Planned indices (Remoteness, Surface Quality, Fatigue, Descent Risk) also have columns on `route_slice_analysis` — the table is ready for them.
- Environmental conditions (wind, temperature, etc.) correctly noted as "not yet integrated" — they belong in Phase 2.

---

## Fields that appear in docs but do not exist in the DB

| Doc field | Doc location | Status |
|---|---|---|
| `canonical_geometry` | ds-005 | Use `geometry` on canonical_routes |
| `corridor_geometry` | ds-005 | Not yet stored as a column |
| `bounding_box` | ds-005 | Not on canonical_routes |
| `centroid` | ds-005 | Not on canonical_routes |
| `canonicalization_version` | ds-005 | Not on canonical_routes |
| `status` | ds-005 | Not on canonical_routes |
| `osm_way_ids` | ds-007 | In raw_osm_tags_json, not a column |
| `route_variants` table | ds-005 | Not yet migrated |
| `user_routes` table | ds-005 | Not yet migrated |
| `events` table | adr-031 | Not yet migrated |
| `event_days` table | adr-031 | Not yet migrated |
| `event_route_part_segments` table | adr-031 | Not yet migrated |
| `route_slice_effective_facts` | SCHEMA_SHAPE | Not yet built |
| `ride_instance_runs` | SCHEMA_SHAPE | Phase 2 |
| `ride_instance_slice_conditions` | SCHEMA_SHAPE | Phase 2 |
| `route_slice_overrides` | SCHEMA_SHAPE | Phase 2 |

---

## Fields in the DB not prominently documented

| Table | Column | Notes |
|---|---|---|
| `canonical_routes` | `distance_km` | Stored as km here; downstream tables use distance_m. Be careful when joining. |
| `canonical_routes` | `geometry_hash` | Legacy or parallel to geometry_fingerprint — clarify which is authoritative |
| `canonical_routes` | `fingerprint` | Distinct from geometry_fingerprint — document what this hashes |
| `imported_routes` | `geometry` jsonb | Legacy field; prefer `geom` (geometry type) for spatial work |
| `route_slices` | `slice_builder_version` | Should be incremented when slice generation algorithm changes |
| `route_analysis_runs` | `weather_snapshot_version` | Nullable for stable runs; required for ride_instance family |


---

## Source File: docs/migrations/2026-03-22-slices_migration_log.md

# Lanterne Migration Log
*Last updated: 2026-03-22*

---

## Entry 1 — Canonical Route Bootstrap
Date: 2026-03-21/22
Status: Complete

### Corpus Snapshot
- 3151 routes imported from RWGPS via external_route_catalog
- 3151 canonical routes populated with geometry and fingerprints
- 0 duplicates generated
- 0 unmatched routes
- Perfect 1:1 assignment achieved

### Geometry Fingerprint Strategy

**Algorithm:**
1. Normalize geometry direction — if ST_StartPoint < ST_EndPoint lexicographically, use as-is; otherwise ST_Reverse()
2. Resample to 100 evenly-spaced points via ST_LineInterpolatePoints(geom, 0.01)
3. Round each coordinate to 5 decimal places
4. Serialize as pipe-delimited lon,lat string
5. SHA-256 hash via pgcrypto digest()

**Why direction normalization:**
RWGPS exports and GPX files frequently represent the same corridor in opposite directions. Without normalization, a reversed export produces a different fingerprint and creates a duplicate canonical.

**Why 100 resampled points:**
Raw point counts vary wildly (99–4924+ points in corpus). Resampling to a fixed count ensures fingerprint stability across routes with different GPS sampling densities.

**Critical implementation note:**
Both `imported_routes` and `canonical_routes` must use the identical fingerprint formula. During bootstrap, imported_routes fingerprints were initially computed without direction normalization, causing 1970 mismatches. All imported fingerprints were recomputed with the correct formula before scale testing.

### Join Key Discovery

**The trap:**
`organization_published_routes.imported_route_id` stores the UUID of `imported_routes.id` — NOT `source_route_id` (the RWGPS integer ID).

**Correct join path:**
```
canonical_routes
    ↑ (via canonical_route_id)
organization_published_routes
    ↓ (via imported_route_id = imported_routes.id)
imported_routes
```

Any future migration or ingestion code that tries to join on `source_route_id` will get zero rows. Always join on the UUID id column.

### Duplicate Discoveries

**Case 1 — A Birmingham Tour:**
- Two canonical shells with identical names, same imported_route_id, same creation timestamp
- Cause: import ran twice without idempotency guard
- Resolution: kept 6dd8937e, deleted e20d389f
- Repointed organization_published_routes accordingly

**Case 2 — Tail of the Lizard:**
- Two canonical shells: "Tail of the Lizard" and "Tail of the Lizard 200k"
- Same imported_route_id, same creation timestamp
- Cause: RUSA has two separate permanent entries (different Route_IDs, incremented sequentially) pointing to the same RWGPS route file
- Resolution: kept c34df51c (200k), deleted 2aa7e043
- Both organization_published_routes rows repointed to kept shell
- This is a valid RUSA data pattern — multiple permanents sharing one physical corridor. Expected to recur with other orgs.

### Schema Changes Applied
- `canonical_routes`: added geom geometry(LineString, 4326)
- `canonical_routes`: added length_m numeric
- `canonical_routes`: added GIST index on geom
- `canonical_routes`: geometry_fingerprint recomputed with direction normalization
- `imported_routes`: geometry_fingerprint recomputed with direction normalization
- `imported_routes`: source_platform backfilled to 'rwgps'

### Canonical Matching Function
`assign_canonical_route(p_imported_id uuid)` created in Supabase.

**Logic:**
1. Exact fingerprint match (fast path)
2. Bounding box spatial candidate search
3. Length filter — candidates within 25% of imported route length
4. Start point guardrail — candidates within 25km of start point
5. Corridor overlap — ST_Buffer(geom, 80m) intersection / route length
6. Threshold — overlap >= 0.85 assigns existing canonical
7. No match — creates new canonical shell

**Test results:**
- Scale run: 3151/3151 matched existing canonicals, 0 new created

### Architectural Consequences
- All analysis attaches to canonical_route_id, never to imported_routes.id
- All future ingestion resolves via assign_canonical_route()
- canonical_routes.geometry is the authoritative geometry for all downstream analysis including route_slices

### Pending Constraint (apply before next ingestion run)
```sql
ALTER TABLE canonical_routes
ADD CONSTRAINT unique_geometry_fingerprint
UNIQUE (geometry_fingerprint);
```

---

## Entry 2 — Phase 1 Route Intelligence Schema
Date: 2026-03-22
Status: Complete (tables + hardening)

### Tables Created
- `route_slices`
- `route_slice_osm_facts`
- `route_slice_support_facts`
- `route_analysis_runs`
- `route_slice_analysis`
- `route_analysis_summary`

### Hardening Constraints Applied

**Unique constraints on fact tables** (prevent silent double-writes):
```sql
UNIQUE (route_slice_id, osm_snapshot_version)     -- route_slice_osm_facts
UNIQUE (route_slice_id, support_snapshot_version)  -- route_slice_support_facts
```

**Scoped uniqueness for current runs** (prevents multiple current runs for same route/family/mode):
```sql
CREATE UNIQUE INDEX route_analysis_runs_one_current_per_scope
ON route_analysis_runs (canonical_route_id, analysis_family, mode_profile)
WHERE is_current = true;
```

This correctly allows coexistence of:
- one current `stable_route` / `road` run
- one current `stable_route` / `bikepacking_gravel` run

While preventing two current `stable_route` / `road` runs for the same canonical.

**Slice geometry sanity checks:**
```sql
CHECK (sequence > 0)
CHECK (length_m > 0)
CHECK (end_distance_m > start_distance_m)
```

**Controlled vocab on analysis_runs:**
```sql
CHECK analysis_family IN ('stable_route', 'ride_instance')
CHECK status IN ('pending', 'running', 'complete', 'failed')
CHECK mode_profile IN ('road', 'bikepacking_gravel')
```

### Key Design Decisions Recorded

**slice_boundary_reason is text[]:**
A slice can be cut for multiple simultaneous reasons (e.g. distance threshold AND road class change). Array is cleaner to query than JSON for this case.

**bridge_flag / tunnel_flag / rail_crossing_flag in route_slice_osm_facts:**
These are OSM-derived structural facts, not community-reported hazards. Community hazards live in the existing hazard_comments / hazard_confirmations tables. Separation is intentional.

**light_state not in route_slice_osm_facts:**
Light is ride-time-dependent. It belongs in ride_instance_slice_conditions (Phase 2). A geographic default could be computed but that computation belongs in the ride instance layer.

**canonical_route_id denormalized in route_analysis_summary:**
Kept for query convenience (many UI queries filter by route). The run already references the canonical, so this is technically redundant. A consistency check or trigger should be added in a future migration. Always populate both from the same source in ingestion code.

### What Is Not Yet Migrated (Phase 2)
- `ride_instance_runs`
- `ride_instance_slice_conditions`
- `route_slice_overrides`
- `route_slice_effective_facts` (view or materialized)
- Full event/day model per ADR-031 (`events`, `event_days`, `event_route_part_segments`)

### Recommended Next Steps
1. Seed one `route_analysis_runs` row for a small set of canonicals
2. Populate `route_slice_analysis` and confirm indices are believable on real data
3. Populate `route_analysis_summary` and verify rollups
4. Build `route_slice_effective_facts` view
5. Then proceed to Phase 2 ride-instance tables

