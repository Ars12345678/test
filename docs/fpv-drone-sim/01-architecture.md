# 1. Core Game Architecture & Modular Systems

The architecture is a **modular, event-driven** design. Systems communicate through a
central **Game Event Bus** (a UE5 `GameInstanceSubsystem` broadcasting delegates) so
that, e.g., the RF system, the HUD, and the scoring system all react to a "signal lost"
event without hard references to each other. This keeps subsystems independently
testable and lets you disable any one of them (traffic, RF, weather) for debugging.

```
                         ┌─────────────────────────────┐
                         │      Game Event Bus          │
                         │ (GameInstanceSubsystem +     │
                         │  typed multicast delegates)  │
                         └──────────────┬──────────────┘
        ┌───────────────┬───────────────┼───────────────┬───────────────┐
        │               │               │               │               │
 ┌──────▼─────┐  ┌──────▼─────┐  ┌──────▼──────┐  ┌─────▼──────┐  ┌─────▼──────┐
 │  World &   │  │  Flight    │  │ RF / Signal │  │  Mission / │  │  Input &   │
 │ Streaming  │  │  Physics   │  │  Occlusion  │  │  Event Sub │  │  Telemetry │
 │  Subsystem │  │  (Chaos)   │  │  Subsystem  │  │  system    │  │  Subsystem │
 └────────────┘  └────────────┘  └─────────────┘  └────────────┘  └────────────┘
        │               │               │               │               │
 ┌──────▼───────────────▼───────────────▼───────────────▼───────────────▼──────┐
 │                    Actor / Component Layer (Pawns, Vehicles, Targets)        │
 └─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1.1 World & Environment Subsystem

**Goal:** stream a photorealistic Minsk at FPV speeds (0–150 km/h) with no hitches.

### Geospatial pipeline
- **Source data:** OpenStreetMap building footprints + heights; terrain from an SRTM/
  local DEM; optional **Cesium 3D Tiles** (`Cesium for Unreal` plugin) for global
  context and far-field city mass.
- **Georeferencing:** a single `CesiumGeoreference` origin anchored on central Minsk
  (~53.9°N, 27.56°E) so all local coordinates share one ENU frame.
- **Hero zones vs. context:** Cesium/photogrammetry tiles give the mid/far city; a set
  of **hand-authored "hero" districts** (Oktyabrskaya Street industrial zone, a
  constructivist block, a modern high-rise cluster) are built as bespoke Nanite assets
  for close-flight fidelity. This hybrid avoids the "melted" look of pure photogrammetry
  up close.

### Rendering
- **Nanite** for all static building/prop geometry — virtualized geometry means you
  author dense meshes and let Nanite handle LOD. Ideal for a city of thousands of
  facades.
- **Lumen** for global illumination + reflections. FPV flight whips between bright sky
  and deep concrete shadow constantly; Lumen's dynamic GI sells that transition. Use
  Hardware Ray Tracing Lumen on target platforms, Software Lumen as the floor.
- **World Partition + Data Layers:** the map is a World Partition grid. Data Layers
  toggle season/weather/time-of-day dressing. **HLODs** provide the far silhouette so
  the skyline is always present even when cells are unloaded.
- **Level streaming budget:** stream cells on a predictive frustum + velocity vector
  (load ahead along the drone's heading), because FPV closing speeds are high.

### Modules
| Module | Responsibility |
|--------|----------------|
| `UWorldStreamingSubsystem` | Predictive cell load/unload from drone velocity. |
| `AMinskGeoAnchor` | Cesium georeference + ENU origin. |
| `UTimeOfDaySubsystem` | Sun/sky, Lumen sky light, streetlight toggling. |
| `UWeatherSubsystem` | Rain/fog/wind — feeds both visuals and physics wind vector. |

---

## 1.2 Dynamic Traffic System

A **spline-driven AI traffic** system with variable density, feeding both ambience and
the mission spawner (targets *are* traffic actors — see doc 3).

- **Road graph:** import OSM road network → build a runtime **lane graph** of
  `USplineComponent` segments with metadata: `RoadClass` (highway/arterial/inner-city/
  service), `SpeedLimit`, `LaneCount`, `Density` weight.
- **Agents:** lightweight vehicle actors that follow splines with an **IDM (Intelligent
  Driver Model)** car-following rule for spacing + a gap-acceptance model at junctions.
  No full physics per car — kinematic on-spline with a Chaos physics stand-in only when
  on camera and relevant (LOD by distance).
- **Variable density:** each spline advertises a target vehicles-per-km; a
  `UTrafficDirector` maintains population per district — dense inner city (Nezavisimosti
  Ave.), sparse industrial (Oktyabrskaya), fast highway ring (MKAD analog).
- **Pooling:** vehicles are pooled and recycled around the player's interest bubble;
  off-screen traffic is simulated statistically (counts only), not as actors.

```
UTrafficDirector
  ├── FDistrictDensityProfile[]        // per-zone target density curves by time-of-day
  ├── ULaneGraph (splines + metadata)
  ├── FVehiclePool                     // recycled AVehicleAgent actors
  └── SpawnPolicy: keep N agents within interest radius, statistical beyond
```

---

## 1.3 RF Signal Occlusion Subsystem

The signature mechanic: **line-of-sight radio quality**. Concrete between the drone and
the "ground station" degrades the video feed. This is a **post-process + audio** effect
driven by a physics query — it does not affect the drone's control authority directly
(that would feel unfair), but it degrades what the pilot *sees*, which is the whole
point of FPV.

### Signal model (per tick, low frequency ~10–20 Hz, not every frame)
1. **LoS trace:** sphere/line trace from drone → ground station (player's notional
   antenna position). Count occluders and accumulate **material attenuation**:
   concrete ≫ brick > glass > foliage > air.
2. **Fresnel/multipath approximation:** add penalty for near-grazing building edges and
   a small random multipath jitter so the signal "breathes" like the real thing.
3. **Link budget:** `SNR = TxPower − PathLoss(distance) − Σ(occluder attenuation) + AntennaGain(orientation)`.
   Drone antenna orientation matters — a hard yaw behind a building tanks the link.
4. **Quality → feed state:** map SNR to a `SignalQuality` 0..1 driving the feed.

### Analog vs. digital feed profiles (selectable in loadout)
| Feed type | Degradation character |
|-----------|-----------------------|
| **Analog** | Graceful: static/noise bands, rolling sync tears, color loss, then full snow. Always *some* image. |
| **Digital (HD)** | Cliff-edge: crisp until threshold, then macroblocking, frame freeze, latency spike, **blue-screen loss**. |

### Presentation modules
- `URFLinkComponent` (on the drone): runs the link budget, outputs `SignalQuality`,
  `Latency`, `FeedState` and broadcasts `OnSignalDegraded/OnSignalLost` on the event bus.
- **Post-process material** on the FPV camera: parameterized by `SignalQuality` — noise,
  scanlines, chromatic tearing (analog) / block-corruption + frame-hold (digital).
- **Simulated latency:** feed is rendered to a small ring buffer of render targets and
  displayed N frames late as latency rises — the pilot literally sees the past, which is
  the true FPV penalty.
- **Audio:** RF hiss / digital artifact chirp rises with degradation.

> Design note: expose all attenuation constants and the SNR→feed curve as a
> `UDataAsset` so designers tune "how punishing is concrete" without recompiling.

---

## 1.4 Flight Physics Subsystem (Acro Mode)

The heart of the sim. Built on **Chaos physics** with a custom force/torque model on the
drone's rigid body so it behaves like a rate-controlled multirotor, not a helicopter or
a plane. Target the physics tick at a **fixed high rate (e.g. 200–400 Hz substep)**
decoupled from render for stable, deterministic feel.

### Control model — rate (Acro) mode
- Sticks command **angular rates**, not angles. There is no self-leveling. Pipeline:

```
Stick input ──▶ Rate Setpoint (via Rates curve) ──▶ [PID controller] ──▶ Torque
                       ▲                                    ▲
                RC Rates / Super Rates / Expo        Gyro feedback (current body rates)
```

- **Rates model:** implement Betaflight-style **RC Rate / Super Rate / Expo** so the
  curve matches what real pilots configure. Center stick = fine control, ends =
  aggressive. This is essential for authenticity.
- **PID loop:** per-axis (roll/pitch/yaw) PID converts rate error → motor torque demand.
  Ships with tunable presets ("smooth", "locked-in", "freestyle").

### Forces & fidelity
| Element | Model |
|---------|-------|
| **Thrust** | Per-motor thrust from RPM²·k; sum + differential = lift + torque. |
| **Thrust-to-weight** | Config-driven (typical 5" freestyle ≈ 3:1 to 8:1). Drives punch-outs. |
| **Inertia** | Real diagonal inertia tensor on the rigid body → authentic rotational snap & carry. |
| **Drag** | Body drag + per-prop drag; airframe drag scales with velocity². |
| **Prop wash** | Turbulence torque when descending through own downwash (the classic wobble). |
| **Battery voltage sag** | Battery model: voltage droops under current draw → less thrust at high throttle & as pack depletes. Punch-outs weaken near end of pack. |
| **Wind** | Wind vector from Weather subsystem adds a translational force; gusts near building corners (venturi zones authored as trigger volumes). |
| **Camera tilt** | Uptilt angle offsets pitch attitude vs. flight path — high tilt = faster forward flight, changed horizon. Player-configurable, feeds the FPV camera. |

### Modules
| Module | Responsibility |
|--------|----------------|
| `UDroneFlightController` | Rates → PID → torque; owns the control loop. |
| `UMotorComponent` (×4) | RPM, thrust, torque, current draw. |
| `UBatteryComponent` | Cell count, capacity, internal resistance, sag curve. |
| `UAerodynamicsComponent` | Drag, prop wash, wind coupling. |
| `FDroneTuningAsset` (DataAsset) | Rates, PID, mass, TWR, inertia — all data-driven per airframe. |

---

## 1.5 Gameplay, Targets & Destruction Subsystem

Missions are **objectives layered onto the living world**, resolved by a shared combat/
interaction model. (Spawning logic is doc 3.)

### Target model
- Targets are traffic/AI actors flagged by the spawner with a `UTargetComponent`
  carrying: `TargetType` (VIP transport / contraband truck / hostile drone), `Health`,
  `Hitboxes`, `Vulnerabilities`, `Behavior` (evasive, patrol, convoy).
- **Hitboxes:** multiple child collision shapes with per-zone multipliers (a "sensor
  mast" vs. "chassis") so precision is rewarded — a game hitbox system, identical in
  kind to any shooter.

### Interaction / effect types (all simulated FX + state changes)
| Effect | Simulation |
|--------|------------|
| **Kinetic collision** | Physics impulse on impact; Chaos destruction (Geometry Collections) fractures the target mesh; drone takes damage / disarms. |
| **EMP disablement** | Area effect that flips target's `IsPowered=false` → it coasts/stalls, lights die, hostile drones drop. No fracture. |
| **Spatial impact / proximity** | Radius-based effect with falloff for area objectives. |

### Mission event archetypes (feed the spawner)
1. **High-speed traffic pursuit (Precision):** intercept a moving convoy in flowing
   traffic; scored on precision + avoiding civilian traffic (penalty for collateral).
2. **Indoor / tight-gap reconnaissance:** thread interiors/gaps, capture "recon" points
   (camera dwell on a marker). Tests precision flying + RF loss indoors.
3. **Counter-drone interception under EW:** chase hostile drones while an **Electronic
   Warfare** zone cranks RF degradation and adds control noise — the signal system is the
   difficulty knob.

### Scoring & feedback
- `UScoringSubsystem` listens on the event bus: precision bonuses, time, collateral
  penalties, RF-under-pressure bonuses. Drives post-mission grade.

### Modules
| Module | Responsibility |
|--------|----------------|
| `UTargetComponent` | Type, health, hitboxes, vulnerabilities, behavior hooks. |
| `UDamageResolver` | Kinetic/EMP/proximity → state change + FX; single entry point. |
| `UDestructionComponent` | Chaos Geometry Collection fracture + VFX/SFX. |
| `UMissionSubsystem` | Objective lifecycle, win/lose, ties to Event Spawner. |
| `UScoringSubsystem` | Score, grade, telemetry rollup. |

---

## 1.6 Cross-Cutting: Data-Driven Everything

Every tuning surface — flight, RF, traffic density, spawn tables — lives in
`UDataAsset` / `UDataTable` assets, not code. This is the single most important
architectural decision for a "feel"-driven sim: designers iterate at runtime, and the
same assets power automated tuning tests.
