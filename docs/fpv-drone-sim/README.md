# Project "Silhouette" — FPV Drone Simulator (Minsk)

A high-fidelity FPV (First-Person View) drone flight & mission simulator built in
**Unreal Engine 5**, set in a photorealistic reconstruction of **Minsk** — mixing
Soviet constructivism, modern high-rises, and industrial zones (Oktyabrskaya Street).

This is a **game / flight-simulation design**. Everything here describes in-engine
systems: Nanite geometry, Lumen lighting, Chaos physics, gameplay actors, and
Blueprint/C++ logic. Hitboxes, "strikes", and "EMP" are simulated game events
(particle FX, health components, score) — not real-world hardware or targeting
instructions.

## Document Index

| # | Document | Contents |
|---|----------|----------|
| 1 | [`01-architecture.md`](01-architecture.md) | Core game architecture & modular systems (world, physics, RF, AI traffic, gameplay). |
| 2 | [`02-technical-roadmap.md`](02-technical-roadmap.md) | Step-by-step execution roadmap, milestones, tech stack, risk register. |
| 3 | [`03-dynamic-event-spawner.md`](03-dynamic-event-spawner.md) | Dynamic Event Spawner: data-driven design + Blueprint/pseudocode logic. |
| 4 | [`04-input-and-mobile-controllers.md`](04-input-and-mobile-controllers.md) | USB radio controllers (RadioMaster/TBS) + mobile touch/gyro controllers. |

## The Pillars

1. **Feel first.** Acro-mode flight has to feel like a real 5" freestyle quad —
   rate-based control, inertia, prop wash, voltage sag. If the stick feel is wrong,
   nothing else matters.
2. **The city is the antagonist.** Concrete mass drives both navigation and the
   signal-degradation model. Line-of-sight is a first-class gameplay mechanic.
3. **Emergent missions.** A data-driven event spawner turns a living traffic sim
   into infinitely replayable objectives, rather than hand-placed set pieces.
4. **Accessibility of input.** Full USB radio support for enthusiasts; touch + gyro
   for mobile so a newcomer can fly on a phone.
