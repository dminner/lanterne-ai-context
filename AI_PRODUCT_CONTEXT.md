# Lanterne Product Philosophy Context


---


This document contains the philosophical and product framework for Lanterne.

Sources included:

• philosophy
• rider archetypes
• UX principles
• visual language
• iconography
• product guardrails


---

## Source File: docs/01-philosophy/phi-001-lanterne_manifesto.md

# The Lanterne Manifesto

## The Road Is Not Neutral

Cyclists are told that a line on a map is a route.

It is not.

A line on a map hides everything that actually matters.

It hides traffic.
It hides isolation.
It hides dangerous descents.
It hides roads that feel safe and roads that feel wrong.

Two routes may look identical on a map and yet be completely different experiences on the ground.

One is quiet and beautiful.

The other is terrifying.

Cyclists deserve to know the difference.

------

## The Problem With Modern Cycling Software

Most cycling software was built to answer performance questions.

How fast did you ride?
How far did you go?
Who was faster?

Those are interesting questions.

But they are not the questions riders ask when they are alone on a dark road hundreds of miles from home.

Long-distance riders ask different questions.

Is this road safe?
Is there anywhere to stop ahead?
How remote am I right now?
What happens if something goes wrong here?

Existing tools rarely answer these questions.

Lanterne exists to change that.

------

## Intelligence, Not Just Navigation

Lanterne is not simply another route planner.

It is a system designed to reveal the hidden character of roads.

It studies a route and exposes the things a cyclist cannot see on a map:

• traffic exposure
• infrastructure support
• remoteness
• environmental conditions
• fatigue accumulation
• descent risk

The goal is not to overwhelm riders with data.

The goal is to help them make better decisions.

------

## Built for the Riders Who Go Far

Lanterne is designed for cyclists who travel far from home.

Riders who:

• ride through the night
• cross unfamiliar landscapes
• ride alone for hours or days
• commit to roads they have never seen

These riders depend on good judgment.

Lanterne exists to support that judgment.

------

## Calm Technology

Cyclists already face enough noise.

Traffic.
Weather.
Fatigue.
Darkness.

Technology should not add to that noise.

Lanterne is designed to feel calm and quiet.

Complex analysis happens behind the scenes.

The rider sees only what matters.

------

## Respect for the Road

Cycling is not a video game.

It is not a leaderboard.

It is a human being moving through a real landscape under real conditions.

The road deserves respect.

And so do the riders who travel it.

Lanterne was built for them.

------

## A Different Kind of Cycling Tool

Lanterne is not trying to replace existing cycling platforms.

Performance tracking, social networks, and route libraries all have their place.

But something has always been missing.

A system that helps riders understand the road itself.

That is what Lanterne is meant to be.

A lantern on the road ahead.

---

## Source File: docs/01-philosophy/phi-003-analysis_model.md

# Lanterne Analysis Model

## Purpose

This document defines the conceptual analysis model behind Lanterne.

It is intentionally written in product-aware language rather than pure math notation so it can stay useful to both technical and product decisions.

---

## 1. Core Principle

Lanterne analyzes routes in layers:

1. **Route geometry**
2. **Small internal analysis slices**
3. **Index calculations**
4. **Route rollups**
5. **Mode-aware presentation**

Lanterne does not treat a route as one giant average. It evaluates many small parts of a route, then rolls them up into a rider-facing picture.

---

## 2. Atomic Analysis Unit

**Decision:** Indices are computed on **small internal slices** of the route rather than on large visible segments.

**Why:** Large segments smooth away important truth — remoteness dips near a town, traffic variation, lighting changes, weather changes, surface transitions.

**Implication:** The thing Lanterne *calculates on* does not have to be the same thing it *shows in the UI*. Display segments may aggregate many slices for readability.

Typical slice length: 200–500 meters, capped at 750–1,000 meters. See DS-007.

---

## 3. Safety Definition

**Safety Score** is defined narrowly as:

> Likelihood of a rider being struck by a motor vehicle, and severity of the likely outcome.

### Included in Safety
- Traffic Index
- Bike Support Index

### Not included in Safety

These may matter greatly for the ride but are not part of the narrow safety definition. They are modeled separately and must never contaminate the Safety Score:

- Remoteness
- Fatigue
- Temperature
- Wind
- Precipitation
- Moonlight
- UV
- Surface quality
- Descent risk

---

## 4. Index Families

### A. Safety
Contribute to the top-line Safety Score.

| Index | Rider question |
|-------|---------------|
| Safety Score | How safe is this road from a motor vehicle collision standpoint? |
| Traffic Index | How dangerous is the motor vehicle environment? |
| Bike Support Index | How well does this road support cyclists? |

### B. Route Reality
Describe the physical and logistical character of the route.

| Index | Rider question |
|-------|---------------|
| Remoteness Index | How far am I from help, services, and bailout options? |
| Surface Quality Index | How rideable is the surface? |
| Fatigue Index | How much cumulative burden does this section contribute? |
| Descent Risk Index | How risky is this downhill if something goes wrong? |

### C. Conditions
Describe ride-time environmental conditions. Depend on start time and forecast data.

| Signal | Rider question |
|--------|---------------|
| Wind | How much will wind affect this part of the ride? |
| Temperature | Will temperature be a problem here? |
| Precipitation | How much do wet conditions worsen this section? |
| Light | What is the light state when I arrive here? |

### D. Light / Sky Signals
Condition signals rather than major route scores.

- Sun glare warning
- UV halo
- Moon phase
- Cloud overlay

---

## 5. Index Definitions

### 5.1 Traffic Index

**Rider question:** How stressful or dangerous is the motor vehicle environment?

**Typical inputs:** road classification, speed environment, lane count, shoulder absence, connector/highway proximity, intersection density, traffic proxy features, AADT where available.

**Output type:** Stable baseline route analysis.

---

### 5.2 Bike Support Index

**Rider question:** How well does this road support a person riding a bike?

**Typical inputs:** bike lane presence, protected facility presence, paved shoulder width/quality, path/greenway/trail support, continuity of bike-supportive infrastructure.

**Output type:** Stable baseline route analysis.

---

### 5.3 Remoteness Index

**Rider question:** How far am I from help, services, food, water, and bailout options?

**Typical inputs:** settlement proximity, service density, resupply access, bailout opportunities, route isolation, road network sparsity, time-to-help proxy.

**Output type:** Stable-ish route analysis.

**Important:** Remoteness should not be over-smoothed. A route passing through one access point should show a local dip, not flatten the whole region. Longest unbroken remote stretch matters as much as the average.

---

### 5.4 Surface Quality Index

**Rider question:** How rideable is the surface on this part of the route?

**Typical inputs:** paved vs unpaved, surface type, roughness/smoothness proxies, degraded pavement proxy, gravel/dirt/trail character.

**Output type:** Stable-ish route analysis. Especially important for gravel and bikepacking contexts.

---

### 5.5 Fatigue Index

**Rider question:** How much cumulative rider burden does this part of the route contribute?

**Typical inputs:** grade, cumulative climbing, accumulated route distance, repeated rollers, stop/start burden, traffic stress contribution, surface drag contribution. Future: weather burden, personal fatigue arc.

**Output type:** Stable baseline with room for future contextual adjustments.

**Important:** Fatigue is not just climbing. A flat but exposed, windy, rough, stressful, long route can still be highly fatiguing.

---

### 5.6 Descent Risk Index

**Rider question:** How risky is this downhill section if something goes wrong?

**Typical inputs:** negative grade, descent length, curvature, road width, shoulder availability, surface quality. Future: weather interaction, darkness interaction.

**Output type:** Stable baseline with future contextual modifiers.

---

### 5.7 Wind

**Rider question:** How much will wind punish or affect this part of the ride right now?

**Typical inputs:** forecast wind speed, gusts, direction, route bearing, timing estimate, terrain exposure proxy.

**Output type:** Contextual ride-time condition.

---

### 5.8 Temperature

**Rider question:** Will temperature be a problem on this part of the ride?

**Typical inputs:** air temperature, apparent temperature, humidity, timing estimate, exposure duration, extreme threshold flags.

**Output type:** Contextual ride-time condition.

---

### 5.9 Precipitation

**Rider question:** How much do rain, snow, or wet conditions worsen this part of the ride?

**Typical inputs:** precipitation probability, intensity, type, timing estimate, interaction with surface, interaction with descent.

**Output type:** Contextual ride-time condition.

---

## 6. Light and Sky Model

Lanterne models light conditions separately from route safety.

### Light State
Computed from solar altitude: Daylight / Twilight / Night.

### Sun System
Visualized using sun icon, UV halo, optional cloud context, tooltips.

**Sun glare (ADR-010):** Glare warning occurs when sun elevation is between 0–6° above the horizon AND rider bearing is within ±30° of the sun azimuth. This is a condition warning, not a safety score input.

### Moon System
Visualized using moon phase icon and cloud overlay. Communicates whether night conditions feel moonlit or dark — important for randonneurs riding overnight.

---

## 7. Rollups

Indices exist at two main levels:

- **Slice level** — most truthful, most granular
- **Route rollup level** — rider-facing summary across the whole route

**Rollup strategies:**

| Metric | Strategy |
|--------|---------|
| Traffic Index | Weighted mean + 95th percentile emphasis |
| Bike Support Index | Weighted mean |
| Remoteness Index | Longest unbroken stretch + peak isolation |
| Surface Quality | Weighted mean + worst-surface emphasis |
| Fatigue Index | Cumulative burden |
| Descent Risk | Concentrated-risk emphasis |

Rollups must preserve route character — a short dangerous section must not be washed out by many easy miles.

**Windowed rollups (ultra-distance routes):** For routes using the expedition model (ADR-034), rollups are computed per analysis window and assembled into a full-route summary. The active window receives detailed treatment; the full route may be shown in simplified overview form. This is the intended operating model for ultra routes, not a degraded fallback.

---

## 8. Mode and Context Profiles

Indices keep the same meaning across modes. What changes by mode is prominence, weighting, explanation, and UI ordering.

| Profile | Emphasis |
|---------|---------|
| Road | Safety Score, Traffic Index, Bike Support Index, Fatigue |
| Bikepacking / Gravel | Surface Quality, Remoteness, Fatigue, Temperature, Precipitation |

---

## 9. Safety Score Structure

Safety Score remains narrow: **Traffic Index + Bike Support Index**.

Everything else remains visible but separate.

A dangerous segment on a road with no bike infrastructure should score the same whether or not it has a headwind. Weather is a different question with a different answer.

---

## 10. Storage Principles

- Use real columns for core index values
- Use JSON for component breakdowns and confidence signals
- Avoid overgeneralized metric mush — do not reduce core route intelligence to one generic key/value score table
- Stable analysis and ride-time conditions live in separate tables (ADR-023)

---

## 11. Future Expansions

- Rider-specific fatigue modeling
- Light-aware descent warnings
- Stronger route-direction sensitivity
- Mode-specific route profiles
- Confidence / uncertainty scoring
- Service access detail under remoteness
- Traffic behavior dimensions (pass frequency, intensity, driver accommodation) — see ADR-032

---

## 12. Working Principle

Lanterne should not try to collapse the whole ride into one giant abstract number.

It should provide:
- One disciplined Safety Score
- A clear set of route reality indices
- A rich but elegant conditions layer

That is how the system stays both useful and trustworthy.


---

## Source File: docs/01-philosophy/phl-002-product_principles.md

# Lanterne Product Principles

## 1. Built for riders who go long

Lanterne focuses on the realities of long-distance riding:

- randonneuring
- bikepacking
- remote endurance riding

Not short urban rides or fitness tracking.

---

## 2. Route intelligence over route recording

Most cycling apps track rides.

Lanterne analyzes routes before riders leave home.

Goal: help riders understand the road ahead.

---

## 3. Safety is defined narrowly

Safety refers specifically to:

> risk of collision with motor vehicles and injury severity.

Other challenges — fatigue, weather, remoteness — are modeled separately and never contaminate the Safety Score.

---

## 4. Real-world conditions matter

The system accounts for environmental realities including:

- wind
- temperature
- precipitation
- sunlight and sun glare
- moonlight and cloud cover

These factors shape the riding experience and are surfaced as a distinct conditions layer, not mixed into safety.

---

## 5. Simplicity beats data overload

Complex models exist behind the scenes.

The interface surfaces insights through:

- simple icons
- short explanations
- visual cues

Lanterne should never feel like a data cockpit.

---

## 6. Respect the experience of riders

Features should reflect how riders actually experience the road:

- night riding under moonlight
- sun glare in the eyes at dawn
- remote stretches without services or bailout
- the difference between a safe shoulder and none at all

---

## 7. Built for curiosity

Lanterne should encourage riders to explore and understand routes more deeply before they leave home.

---

## 8. Built for the long haul

For multi-day routes and expeditions, continuity matters as much as analysis quality.

A rider who stops for sleep, powers down their phone, or opens the app twelve hours later should return to exactly where they left off in the larger journey — not lose context or have to start over.

Expedition state is durable. Live session state is transient. These are different things and must not be conflated.


---

## Source File: docs/05-product/prod-001-positioning.md

# Product Positioning

Lanterne occupies a unique position within the cycling ecosystem.

It does **not compete directly** with most existing cycling platforms.

Instead, it sits **alongside them**.

------

## Cycling Platform Landscape

| Product      | Core Value                             |
| ------------ | -------------------------------------- |
| Strava       | performance tracking + social network  |
| RideWithGPS  | route planning and sharing             |
| Garmin       | device ecosystem and ride recording    |
| Komoot       | recreational route discovery           |
| **Lanterne** | route intelligence and safety analysis |

------

## The Gap Lanterne Fills

Existing tools answer questions like:

• How fast did I ride?
• How far did I go?
• What route should I follow?

Lanterne answers a different question:

**“What kind of road am I committing to?”**

------

## The Lanterne Layer

Think of Lanterne as an **intelligence layer above the map**.

It reveals things that are otherwise invisible:

• traffic exposure
• infrastructure support
• route remoteness
• environmental context
• fatigue accumulation
• descent risk

These factors matter deeply for long-distance cyclists but are rarely modeled.

------

## The Rider Lanterne Serves

Lanterne is designed for riders who:

• travel far from home
• ride alone frequently
• ride through the night
• explore unfamiliar terrain
• plan multi-day routes

For these riders, route choice is not trivial.

It can be the difference between a great ride and a dangerous one.

---

## Source File: docs/05-product/prod-002-vision.md

# Lanterne — Product Vision

Lanterne is a **route intelligence system for long-distance cyclists**.

It helps riders understand the safety, support, remoteness, and environmental conditions of a route before and during a ride.

Lanterne exists for riders who travel **far, alone, and often at night**.

These riders must constantly answer questions like:

- How dangerous is this road?
- Is there support nearby if something goes wrong?
- How remote am I right now?
- What conditions will I encounter overnight?
- Is this route actually the best choice?

Most cycling apps answer none of these questions.

Lanterne does.

------

## What Lanterne Is

Lanterne is:

• a **route intelligence engine**
• a **decision support tool for endurance riders**
• a **safety-aware navigation companion**

It combines route geometry, infrastructure data, environmental modeling, and rider context to generate actionable insight.

------

## What Lanterne Is Not

Lanterne is **not**:

• a social fitness network
• a training analytics platform
• a leaderboard system
• a performance tracker

Those problems are already well solved.

Lanterne focuses on a different question:

**“Is this a good road to ride right now?”**

------

## Core Belief

Routes are not equal.

Some are safer.

Some are quieter.

Some are more remote.

Some are better at night.

Cyclists deserve to understand these differences before committing to hundreds of miles of road.

---

## Source File: docs/05-product/prod-003-rider_archetypes.md

# Rider Archetypes

Lanterne is optimized for a specific set of cyclists.

Understanding these riders helps guide product decisions.

------

## 1. The Randonneur

The randonneur rides extremely long distances under time limits.

Typical characteristics:

• rides of 200–1200 km
• overnight riding is normal
• self-supported
• rides through rural areas

Needs from Lanterne:

• safety awareness at night
• support availability insight
• fatigue-aware route understanding

------

## 2. The Bikepacker

The bikepacker travels for days or weeks across varied terrain.

Typical characteristics:

• remote routes
• mixed surface riding
• unpredictable conditions

Needs from Lanterne:

• remoteness awareness
• support location awareness
• environmental context

------

## 3. The Endurance Soloist

This rider pushes extreme personal challenges.

Typical characteristics:

• solo rides
• very long distances
• minimal external support

Needs from Lanterne:

• route risk awareness
• infrastructure context
• decision support during rides

------

## 4. The Nocturne Rider

Many endurance cyclists ride heavily at night.

Typical characteristics:

• significant night riding
• changing visibility conditions
• fatigue accumulation

Needs from Lanterne:

• sun/moon modeling
• glare detection
• night safety context

------

## Who Lanterne Is Not Designed For

Lanterne is not optimized for:

• casual fitness riders
• short recreational loops
• group ride coordination
• social activity tracking

Those riders are well served by existing apps.

---

## Source File: docs/05-product/prod-004-guardrails.md

# Product Guardrails

These rules protect Lanterne from drifting away from its core purpose.

They should guide product and UI decisions.

------

## 1. Map First

The map is the primary interface.

Analysis supports the map — it never replaces it.

------

## 2. Calm Interface

Lanterne should feel calm and legible while riding.

The interface must avoid:

• dense dashboards
• flashing alerts
• unnecessary complexity

------

## 3. Progressive Disclosure

Information should appear gradually.

The rider should not be overwhelmed with data.

------

## 4. Safety Before Performance

Lanterne prioritizes safety context over performance metrics.

This means:

• traffic exposure matters more than speed
• infrastructure matters more than KOM segments

------

## 5. The Rider Is Often Alone

The product must assume the rider may be:

• remote
• fatigued
• riding at night
• without immediate assistance

Design decisions should reflect that reality.

------

## 6. Intelligence Should Feel Invisible

Lanterne performs complex analysis behind the scenes.

The rider should experience:

clarity, not complexity.

---

## Source File: docs/05-product/prod-005-ux_principles.md

# UX Principles

These principles guide the design of the Lanterne interface.

------

## Calm Over Clever

The interface should feel stable and trustworthy.

Avoid novelty that distracts from riding.

------

## Visual Before Numeric

Maps, color, and spatial context should communicate meaning before numbers do.

------

## Context Matters

The meaning of a route changes with:

• time of day
• weather
• rider progress

The interface should reflect this.

------

## Reduce Cognitive Load

Cyclists often check navigation while moving.

Information must be readable at a glance.

------

## Encourage Good Decisions

The goal is not to overwhelm riders with data.

The goal is to help them make **better route decisions**.

---

## Source File: docs/05-product/prod-006-anti_features.md

# Anti-Features

These are features Lanterne intentionally avoids.

This document protects the product from feature creep.

------

## No Leaderboards

Competition is not the goal of Lanterne.

------

## No Segments

Segment chasing encourages risky riding behavior.

------

## No Social Feed

Lanterne is a tool, not a social network.

------

## No Kudos

The platform does not exist to validate performance.

------

## No Training Analytics

Training metrics belong in specialized training platforms.

------

## No Gamification

Long-distance cycling is already challenging.

The product should not turn it into a game.

------

## The Guiding Rule

If a feature encourages riders to ride **faster instead of smarter**, it likely does not belong in Lanterne.

---

## Source File: docs/05-product/prod-006-branding.md

# Lanterne In-App Branding Refresh

# **Nocturne Survey**

## Design Philosophy

**Nocturne Survey** defines the visual identity of Lanterne as a field instrument for riders who travel long distances through changing light.

The system combines two visual traditions:

**Night Signal** — a dark, lantern-lit atmosphere inspired by night riding, deep road solitude, and minimal light revealing the terrain ahead.

**Cartographer’s Kit** — the precise language of maps, surveys, and field notebooks: contour lines, elevation marks, coordinate ticks, and sparse annotation.

Together these influences produce a design that feels both **technical and human**. It should evoke the quiet intelligence of a rider studying a route under a lamp — part map room, part expedition notebook.

This aesthetic must never drift toward outdoor-brand nostalgia or decorative vintage styling. Lanterne is not a camping brand or a gear catalog. It is a **navigation and route-intelligence tool** built for riders who go long.

The visual system therefore balances:

- darkness and clarity
- warmth and precision
- exploration and discipline

Light should feel directional and intentional. Amber signals reveal important elements rather than illuminating the entire interface.

The result should feel like **a cartographer’s field notebook opened under lantern light**.

------

# Core Visual Principles

### 1. Darkness is the canvas

The interface emerges from a deep charcoal or near-black background.

Darkness is not merely a theme choice — it reflects the lived reality of long-distance riding, where navigation often happens in low light and focused attention.

UI elements should appear as if revealed by a lantern beam rather than sitting on a uniformly lit surface.

Edges may soften or fade subtly into shadow.

------

### 2. Light is a signal, not decoration

Warm amber acts as the visual “lantern” of the interface.

It highlights:

- key route insights
- active navigation elements
- selected tools
- important alerts

Amber should feel purposeful and restrained. It should not dominate the interface or become a decorative accent.

------

### 3. Cartographic precision

The interface borrows subtle cues from mapping and surveying tools:

- topographic contour lines
- coordinate ticks
- elevation notations
- small grid references
- map annotation marks

These should appear sparingly and lightly, never cluttering the interface.

The goal is to evoke the language of maps without turning the UI into a literal topographic poster.

------

### 4. Field-instrument restraint

The visual system should feel like a **tool**, not an illustration.

Avoid:

- decorative wilderness imagery
- camping iconography
- gear illustrations
- distressed vintage graphics
- heavy textures

The atmosphere should be quiet and precise, closer to a navigation instrument or survey notebook than an outdoor lifestyle brand.

------

# Color System

### Base

Deep charcoal / near-black backgrounds

Examples:

```
#0B0C0E
#121417
#171A1D
```

These should provide depth while avoiding pure black, allowing subtle texture and layering.

------

### Signal (Lantern Amber)

Primary highlight tone.

Examples:

```
#F4B942
#E7A72E
#C78D1A
```

Uses:

- active navigation indicators
- selection highlights
- key UI focus states
- subtle glow effects

Amber should feel warm and slightly muted rather than bright neon.

------

### Earth Accents

Muted tones inspired by terrain and maps.

Examples:

Moss green
 Slate gray
 Muted ochre
 Soft rust / ember

These should appear sparingly in secondary UI states or map-related elements.

------

### Red Shift

The current red accent moves toward a warmer ember tone.

Example direction:

```
#A13A2A
```

This prevents harsh contrast against the darker palette.

------

# Typography

Typography must reinforce the balance between **human narrative and technical precision**.

### Primary Serif — Georgia

Georgia remains the primary typeface for Lanterne.

It provides:

- warmth
- readability
- subtle old-world character
- editorial authority

Georgia should be used for:

- headings
- explanatory text
- narrative UI elements
- brand language

Its tone aligns with the idea that Lanterne is built by riders who think deeply about the road.

------

### Technical Counterweight — JetBrains Mono

JetBrains Mono provides the technical voice of the system.

It should be used for:

- coordinates
- route metrics
- elevation values
- technical labels
- system annotations

This pairing creates a productive tension:

**Georgia → human experience of the ride**
 **JetBrains Mono → instrument precision**

------

# Texture

Texture should be extremely subtle.

Acceptable:

- faint paper grain
- soft ink-bleed edges on drawn lines
- slight topo layering

Unacceptable:

- visible parchment backgrounds
- distressed textures
- coffee-stain effects
- scrapbook-style paper elements

The UI must still feel modern and high-quality.

Texture should only be noticeable when looked for.

------

# Form Language

UI shapes and graphical marks should feel inspired by cartographic notation:

- thin linework
- small ticks
- contour arcs
- measured spacing
- survey marks

Rounded or soft shapes should remain minimal.

Precision and alignment matter more than ornament.

------

# Light Behavior

A defining element of the Nocturne Survey aesthetic is **directional light**.

Design rule:

Elements appear as if emerging from darkness.

Amber light should:

- highlight important areas
- create focus
- subtly illuminate selected objects

Avoid flat, evenly lit surfaces.

The interface should maintain depth and shadow.

## Signal Light Model

In the Nocturne Survey aesthetic, amber is treated as **light**, not simply as a color.

This distinction is critical to the visual identity.

Most dark interfaces apply accent colors as flat fills.
 Lanterne instead uses amber to behave like **lantern light emerging from darkness**.

The goal is to evoke the experience of navigating a road at night where signals appear illuminated rather than painted.

### Design Rule

Amber should appear as **a glowing signal within darkness**, not a solid block of color.

Primary signal elements may use:

- a core amber tone
- a faint outer halo
- subtle blur falloff
- gentle fade into surrounding darkness

Example structure:

Core signal

```
#F4B942
```

Outer glow

```
rgba(244,185,66,0.12)
```

The glow should remain extremely subtle and never appear decorative.

------

## Where Signal Glow Is Appropriate

Glow should only appear on elements that represent **active signals or navigation focus**.

Examples include:

- current rider position marker
- active route highlight
- selected navigation state
- warning indicators
- directional arrows
- key route insight markers

These elements represent information that should visually behave like **beacons in the dark**.

------

## Where Glow Must Not Be Used

Glow must not be applied to:

- paragraph text
- large UI surfaces
- background panels
- long data tables
- general interface chrome

Overuse of glow reduces clarity and creates visual noise.

Most interface elements should remain matte and unlit so that signal elements stand out clearly.

------

## Visual Goal

The interface should feel like **a navigation instrument operating in darkness**, where light is used carefully to reveal important information.

Amber signals should behave like:

- a bicycle tail light
- a lantern on a dark road
- a compass needle catching light

This subtle behavior reinforces the product’s central metaphor and distinguishes Lanterne from conventional dark-mode interfaces.

------

# Why this addition matters

Adding this rule does three important things:

1. **Prevents flat orange UI**
    (the biggest risk in dark themes)
2. **Connects the aesthetic to the product metaphor**
    lantern / night riding / navigation
3. **Creates hierarchy automatically**
    signals glow, everything else stays quiet

That’s how the interface starts feeling **instrument-grade** instead of just “dark themed.”

------

# Visual Reference Canvas

A single canvas should visually express the Nocturne Survey philosophy.

The composition should include:

- deep charcoal background
- faint topographic line structures
- warm amber highlights
- sparse coordinate or survey marks
- minimal typography
- soft edge falloff into darkness

The result should feel like an **abstract map under lantern light**.

This canvas acts as the guiding reference for all subsequent UI work.

------

# Guardrails

The design must **never drift toward**:

- REI catalog styling
- vintage camping posters
- bikepacking gear aesthetics
- rustic outdoors branding

Lanterne is not an outdoor lifestyle brand.

It is a **route intelligence instrument** for riders who travel far and often through darkness.

The interface should reflect **discipline, focus, and quiet exploration**.

---

## Source File: docs/05-product/prod-007-visual_language.md

# Lanterne Visual Language

This document defines the visual signaling rules used across the Lanterne interface.

The goal is a system that riders can interpret instantly — especially in motion, fatigue, darkness, or poor weather.

The visual system should behave more like **aviation instrumentation** than a typical mobile UI.

---

# 1. Design Philosophy

Lanterne is not a decorative interface.

It is a **decision support instrument for long-distance riders**.

Visual signals must therefore be:

• stable  
• predictable  
• interpretable at a glance  
• usable under physical stress and fatigue  

Color and motion must always encode meaning.

Decorative variation is avoided.

---

# 2. Day / Night Mode Rule

Lanterne uses a strict rule for theme inversion.

## Neutral Interface Elements

Neutral interface elements invert between day and night to maintain contrast with the map.

Examples:

• text  
• drawer handles  
• UI chrome  
• compass  
• scale bar  
• neutral icons  

Day mode:
charcoal

Night mode:
white

---

## Semantic Colors

Colors that encode meaning **never change between day and night mode**.

These colors represent real-world meaning and must remain stable to prevent cognitive confusion.

Examples:

• danger indicators  
• caution signals  
• infrastructure signals  
• environmental warnings  

Example rule:

red always means danger  
yellow always means caution  
green always means supportive conditions  

These colors remain identical in both day and night modes.

---



---

## Source File: docs/05-product/prod-008-dynamic-risk-indicators.md

# prod-007 — Dynamic Risk Indicators

## Purpose

Lanterne uses **dynamic visual indicators** to communicate route risk.

These indicators adapt to **light state (day vs night)** so riders can interpret route conditions quickly while riding.

The goal is to maintain:

- strong legibility
- intuitive meaning
- consistent semantics

without forcing riders to interpret complex numeric data.

---

## Core Principle

The **meaning of colors never changes**.

Only the **visual palette** adapts to the surrounding light environment.

Example:

| Risk Level | Meaning | Day Color | Night Color |
|-------------|--------|-----------|-------------|
| Low risk | supportive riding environment | green | green |
| Moderate | caution | amber | amber |
| High risk | elevated danger | red | red |

Hue meaning must remain stable.

What changes between day and night is:

- brightness
- saturation
- contrast tuning

---

## Light-Aware Adaptation

Lanterne models **light state** using solar altitude.

States:

- Daylight
- Twilight
- Night

Dynamic risk indicators respond to this state automatically.

### Day Mode

Palette optimized for:

- bright outdoor viewing
- sun glare conditions
- high ambient light

Characteristics:

- slightly darker base colors
- stronger saturation
- thicker line rendering
- strong contrast against map background

---

### Night Mode

Palette optimized for:

- dark adaptation
- reduced eye strain
- minimal map glare

Characteristics:

- softer saturation
- slightly luminous glow
- reduced visual harshness
- subtle contrast against dark basemap

Night palettes must never appear **neon or overly bright**.

---

## Design Goals

Dynamic risk indicators must:

1. remain readable while riding
2. remain interpretable in peripheral vision
3. avoid overpowering the base map
4. preserve rider mental models
5. adapt automatically without user configuration

---

## Non-Goals

Dynamic risk indicators are **not intended to:**

- represent different risk definitions
- change score meaning
- encode additional data dimensions

They exist solely as a **visual legibility system**.

---

## Relationship to Safety Score

Dynamic risk indicators visualize **segment risk results derived from the Safety Score pipeline**.

They do not replace the Safety Score and do not change how it is calculated.

---

## Implementation Rule

All UI surfaces that display segment risk must use the **same dynamic palette logic**, including:

- route heatmap
- segment highlighting
- cue sheet risk markers
- map overlays

This ensures consistent visual interpretation across the product.

---

## Design Rule

```
Hue = meaning  
Brightness = legibility
```

The hue assigned to a risk level must **never change between day and night**.

Only brightness, contrast, and saturation may adapt to light conditions.

This preserves rider intuition and prevents confusion during long rides.

---

## Future Extensions

Possible future improvements include:

- twilight palette interpolation
- color-blind accessibility palettes
- mode-aware emphasis (road vs gravel contexts)
- adaptive contrast based on ambient brightness

These enhancements must maintain the core rule:

**risk color meaning remains constant.**


---

## Source File: docs/05-product/prod-009-iconography.md

Weight rules
Map POIs: thin
Toolbar / controls / menus: regular
Selected state: fill
Hazards: regular icon inside a colored badge/halo
Admin-only tools: regular, sometimes bold for truly internal/debug stuff

That keeps the map quiet while letting controls and hazards stay legible.

Ship set I’d use
Link tree / main actions
where to → signpost
route to → navigation-arrow
draw → pencil-simple-line
load → tray-arrow-down
vault → vault
saved → bookmark-simple
history → clock-counter-clockwise
analyze → chart-line-up
cues → traffic-sign
stops & layers → stack
dev → wrench
inspect → magnifying-glass
Map buttons
current bearing → navigation-arrow
heatmap on/off → lightbulb
day / night → sun / moon
street / satellite → map-trifold
GPS → gps or crosshair

I’d use gps for state and crosshair for “center / locate me now.”

Drawing / editing
snap on → magnet
points + miles → ruler
elevation → mountains
cancel → x-circle
edit route → pencil-simple-line
hold to peek → eye

POIs: **water** / food / **rest** / **health** / **help**
potable water → drop
natural water crossings / water presence → waves
toilets → toilet
showers → shower
restaurants / food → fork-knife
cafes → coffee
convenience / resupply → storefront
supermarket / grocery → shopping-cart
gas station → gas-pump
lodging → bed
campgrounds → tent
post office → mailbox
hospital → hospital
clinic / medical help → first-aid-kit
pharmacy → pill or prescription
police → shield or police-car
fire station → fire-truck
emergency phone → phone-call

Blunt cut:

use pill, not prescription for map POIs. Faster read.
use storefront, not ten different retail metaphors.
use fork-knife for generic food and keep coffee as the only food sub-type worth separate visual treatment.
**Nature** / tourism
peaks → mountains
viewpoints → binoculars
springs → drop
park / green / shade → tree or park
monuments / memorials / ruins → columns if you insist, but I’d demote this whole bucket
Connectivity / ride computer / future continuity
bluetooth → bluetooth
bluetooth connected → bluetooth-connected
bluetooth unavailable → bluetooth-slash or bluetooth-x
radio / transmission → broadcast or radio
cell service → cell-tower
no cell / weak cell → cell-signal-slash, cell-signal-x
wifi / offline → wifi-slash, wifi-x
speaker on/off → speaker-high, speaker-slash
microphone on/off → microphone, microphone-slash
battery states → battery-high, battery-medium, battery-low, battery-warning
charging → battery-charging
download / offline save → download or cloud-arrow-down

This is where Phosphor really earns its keep. The family gives you continuity across the whole product instead of icon soup. That matches your own product principle of using simple icons and cues rather than turning Lanterne into a data cockpit.

Hazards

Here’s the important part: don’t use raw thin icons for hazards. Lanterne already treats hazards as a distinct rider-facing signal layer, and those need faster recognition than ordinary POIs.

Use these as the base glyphs, but wrap them:

railroad crossing → train
traffic conflict / turning conflict → traffic-sign
closure / obstruction / work / impassable vibe → barricade
remote corridor → road-horizon
cellular dead zone → cell-tower or cell-signal-x
service gap → storefront with custom slash/badge treatment
generic warning tier → warning-circle / warning-diamond / warning-octagon

For the custom ones:

dangerous-angle rail crossing
left turn across traffic
metal grate bridge
pinch point

…use Phosphor as the base language, but compose a Lanterne hazard symbol, not just a naked stock icon. That means regular weight, colored disc/halo, maybe a tiny overlay arrow or grate pattern. Raw stock icons alone will be too polite.

