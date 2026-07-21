# 2. Step-by-Step Technical Roadmap

A phased plan that de-risks the hardest, most novel systems first. The golden rule for a
feel-driven sim: **prove the flight model and the input latency before you build content
around them.** A beautiful city with bad stick feel is a failed FPV sim.

## Tech Stack Summary

| Layer | Choice |
|-------|--------|
| Engine | Unreal Engine 5.4+ (Nanite, Lumen, World Partition, Chaos) |
| Geospatial | OpenStreetMap (footprints/roads) + Cesium for Unreal (3D Tiles) + DEM terrain |
| Physics | Chaos rigid body, fixed-substep custom force model |
| Input | Enhanced Input System + RawInput plugin (USB radios) + Touch/Gyro (mobile) |
| Data | UDataAsset / UDataTable for all tuning |
| Destruction | Chaos Geometry Collections |
| Target platforms | PC (primary), then mobile (Vulkan) as a scaled profile |

---

## Phase 0 — Pre-Production & Vertical Slice Target (2–3 wks)
- Lock the **vertical slice**: one hero district (Oktyabrskaya block), one airframe, one
  mission archetype (high-speed pursuit), USB radio input.
- Define the **feel benchmark**: reference videos + a checklist real FPV pilots sign off on.
- Stand up source control (UE-friendly: Perforce or Git LFS), CI, and a
  `FDroneTuningAsset` schema.

## Phase 1 — Flight Physics & Input (the make-or-break) (4–6 wks)
1. Blank level, single drone pawn on Chaos rigid body with correct **inertia tensor + mass**.
2. Implement `UDroneFlightController`: **Rates → PID → torque** loop at fixed substep.
3. Wire **Enhanced Input + RawInput** for a USB radio; get end-to-end **input latency
   under ~2 frames**. Measure it (on-screen latency HUD).
4. Add motors, battery **voltage sag**, drag, prop wash. Tune against the feel benchmark.
5. **Gate:** real FPV pilots fly it blind and say "yes, that's Acro." Do not proceed until
   this passes.

## Phase 2 — World Pipeline (4–6 wks, parallelizable)
1. Cesium georeference on Minsk; stream 3D Tiles for city context.
2. OSM → building footprints extruded, then hero-district Nanite assets authored.
3. World Partition grid + HLOD skyline + predictive streaming from velocity.
4. Lumen lighting pass + Time-of-Day; validate no hitches at 150 km/h transit.

## Phase 3 — RF Signal Occlusion (2–3 wks)
1. `URFLinkComponent` link-budget with LoS traces + material attenuation data asset.
2. Analog & digital **post-process feed materials**; latency ring-buffer.
3. Event-bus hooks (`OnSignalDegraded/Lost`) + HUD + audio.
4. Tune the SNR→feed curve against the hero district's concrete mass.

## Phase 4 — Dynamic Traffic (3–4 wks)
1. OSM road graph → lane splines with class/density metadata.
2. IDM car-following agents + junction gap acceptance; vehicle pool.
3. `UTrafficDirector` variable density by district & time-of-day.
4. Perf: statistical off-screen sim, actor LOD, on-camera Chaos stand-ins.

## Phase 5 — Gameplay, Targets & Destruction (4–5 wks)
1. `UTargetComponent` + hitboxes on traffic actors.
2. `UDamageResolver`: kinetic / EMP / proximity effects.
3. Chaos Geometry Collection destruction + VFX/SFX.
4. `UMissionSubsystem` + `UScoringSubsystem`; build the 3 mission archetypes.

## Phase 6 — Dynamic Event Spawner (2–3 wks) — see doc 3
1. Data-driven spawn tables; director samples traffic for valid targets.
2. Encounter lifecycle (arm → active → resolve → cleanup) on the event bus.
3. Difficulty/pacing director (EW intensity, density, evasion).

## Phase 7 — Mobile Controllers & Cross-Platform (3–4 wks) — see doc 4
1. Vulkan mobile render profile: scaled Nanite/Lumen fallbacks, mobile HLOD.
2. Virtual gimbals + gyro tilt input; assist/leveled mode option for touch.
3. Bluetooth/USB-C gamepad + mobile radio (ELRS-over-USB) support.
4. Perf pass to hold target framerate on tier-1 mobile devices.

## Phase 8 — Polish, Tuning & Live Ops (ongoing)
- Automated tuning regression (replay stick inputs, assert trajectory tolerances).
- Telemetry + heatmaps of where pilots crash / lose signal → tune the city.
- Content cadence: new districts, airframes, mission modifiers via data assets.

---

## Risk Register (top risks first)

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Flight feel not authentic | Fatal | Phase-1 gate with real pilots before any content. |
| Input latency too high (esp. USB radios) | Fatal for feel | Measure early; RawInput; fixed substep; on-screen latency meter. |
| Streaming hitches at FPV speed | Immersion-breaking | Predictive velocity streaming, HLOD, aggressive pooling. |
| Photogrammetry looks melted up close | Quality | Hybrid: Cesium for context, bespoke Nanite for hero zones. |
| Mobile can't hold framerate | Scope | Separate scaled render profile from day one; mobile is a *profile*, not a port afterthought. |
| RF system feels unfair | Fun | RF degrades *vision*, never control authority; tune curves as data. |
