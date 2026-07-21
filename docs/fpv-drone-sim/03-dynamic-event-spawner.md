# 3. Dynamic Event Spawner — Design & Blueprint Logic

The spawner turns the **living traffic sim** into endless missions. Instead of
hand-placing targets, a **director** watches the traffic stream, finds valid candidates,
and "promotes" ordinary vehicles/drones into mission targets according to **data-driven
spawn tables** and a **pacing curve**.

> Scope reminder: this is game-director logic. "Targets", "strikes", and "EW" are
> gameplay states (health components, score, FX) resolved by `UDamageResolver`. The code
> below spawns and scores in-engine actors — it is ordinary game AI, no different in kind
> from a wave director in any action game.

## 3.1 Design Model

Three layered pieces:

1. **Spawn Tables (`UEncounterTableAsset`)** — data. What can spawn, weights, constraints.
2. **Encounter Director (`UEventSpawnerSubsystem`)** — the brain. Decides *when/what/where*.
3. **Encounter Instance (`UEncounterInstance`)** — a live objective with a lifecycle.

### Encounter lifecycle (state machine)

```
        ┌─────────┐  budget & cooldown ok   ┌────────┐  candidate found   ┌─────────┐
        │  IDLE   │────────────────────────▶│ ARMING │───────────────────▶│ ACTIVE  │
        └─────────┘                          └────────┘                    └────┬────┘
             ▲                                    │ no candidate / timeout       │
             │                                    ▼                              │ objective met / failed / target lost
             │                              (back to IDLE)                       ▼
             │                                                              ┌──────────┐
             └──────────────────────────────────────────────────────────── │ RESOLVING│
                                       cleanup + score + cooldown            └──────────┘
```

## 3.2 Data Schema (design these as DataAssets / DataTables)

```cpp
// One row per encounter archetype in a UDataTable.
USTRUCT()
struct FEncounterDef
{
    FName            EncounterId;            // "PURSUIT_VIP", "RECON_INDUSTRIAL", "COUNTER_DRONE_EW"
    EEncounterType   Type;                   // Pursuit / Recon / CounterDrone
    float            Weight;                  // selection weight in the table
    FIntPoint        PlayerSkillRange;        // gate by pilot rating / progression
    float            CooldownSeconds;         // min gap before this id can repeat

    // Placement constraints
    ERoadClass       RequiredRoadClass;       // Highway / Arterial / InnerCity / Service / Indoor
    FFloatRange      SpawnDistanceFromPlayer; // e.g. 250–600 m so it's reachable but not on top of you
    float            MinLineOfSightToPlayer;  // require partial occlusion? (drives RF drama)

    // Target composition
    ETargetType      TargetType;              // VIPTransport / ContrabandTruck / HostileDrone
    int32            TargetCount;             // convoy size, drone swarm size
    EBehaviorProfile Behavior;                // Evasive / Convoy / Patrol

    // Difficulty modifiers
    float            EWIntensity;             // 0..1 electronic-warfare RF pressure
    float            TrafficDensityBias;      // thicken/thin surrounding traffic
    FEncounterFXSet  FXSet;                   // spawn/marker/resolve presentation
};
```

## 3.3 Director Logic — Pseudocode

The director runs on a **low-frequency tick** (e.g. every 0.5–1 s), not per frame.

```cpp
// UEventSpawnerSubsystem::Tick(DeltaTime)  — runs on the Game Event Bus owner
void UEventSpawnerSubsystem::EvaluateSpawns(float Dt)
{
    // 1. Respect global budget & pacing — don't overwhelm the pilot.
    if (ActiveEncounters.Num() >= PacingDirector.MaxConcurrent()) return;
    if (GlobalCooldown > 0.f) { GlobalCooldown -= Dt; return; }

    // 2. Ask the pacing director whether it's time for intensity.
    //    Pacing curve = f(session time, recent success, current tension).
    const float Desire = PacingDirector.SpawnDesire(); // 0..1
    if (FMath::FRand() > Desire) return;

    // 3. Pick a valid encounter def by weighted selection, filtered by
    //    player skill and per-id cooldown.
    FEncounterDef Def;
    if (!SelectWeightedEncounter(PlayerProfile, /*out*/ Def)) return;

    // 4. Find a placement: sample the live traffic/road graph for a candidate
    //    that satisfies Def's constraints (road class, distance, LoS, target type).
    FSpawnCandidate Cand;
    if (!TrafficDirector->FindCandidate(Def, PlayerState, /*out*/ Cand))
    {
        // No suitable traffic right now — nudge the traffic director to create some,
        // then retry next tick (ARMING state).
        TrafficDirector->RequestDensity(Def.RequiredRoadClass, Def.TrafficDensityBias);
        return;
    }

    // 5. PROMOTE an existing traffic actor (or spawn one from the pool) into a target.
    UEncounterInstance* Enc = CreateEncounter(Def, Cand);
    Enc->Promote(Cand.Actor);          // attach UTargetComponent, hitboxes, behavior
    ApplyEWField(Def.EWIntensity);     // scales RF degradation in the area (doc 1.3)

    // 6. Broadcast on the event bus: HUD marker, audio cue, mission log, scoring arm.
    EventBus->OnEncounterStarted.Broadcast(Enc);

    ActiveEncounters.Add(Enc);
    GlobalCooldown = PacingDirector.PostSpawnCooldown();
}
```

### Candidate finding (traffic sampling)

```cpp
bool UTrafficDirector::FindCandidate(const FEncounterDef& Def,
                                     const FPlayerState& P, FSpawnCandidate& Out)
{
    for (AVehicleAgent* V : ActiveAgentsMatching(Def.RequiredRoadClass))
    {
        const float Dist = Distance(V, P.DronePos);
        if (!Def.SpawnDistanceFromPlayer.Contains(Dist)) continue;

        const float LoS = RFSubsystem->LineOfSightQuality(P.GroundStation, V->Location);
        if (LoS < Def.MinLineOfSightToPlayer) continue;   // want some concrete in the way

        if (!MatchesTargetType(V, Def.TargetType)) continue; // e.g. a truck for contraband

        Out = { V, V->CurrentSpline, Dist };
        return true;
    }
    return false; // caller will request more traffic and retry
}
```

## 3.4 Encounter Instance — lifecycle hooks

```cpp
void UEncounterInstance::Tick(float Dt)
{
    switch (State)
    {
        case EState::Active:
            UpdateBehavior(Dt);                 // evasion, convoy formation, drone patrol
            if (ObjectiveMet())   Resolve(EResult::Success);
            else if (Failed())    Resolve(EResult::Fail);    // target escaped / timer / collateral
            break;

        case EState::Resolving:
            PlayResolveFX();
            Score->Submit(BuildReport());       // precision, time, collateral, RF-under-pressure
            EventBus->OnEncounterEnded.Broadcast(this);
            ReleaseToPool();                     // return actor to traffic or despawn
            Director->NotifyResolved(Def.EncounterId); // start per-id cooldown
            break;
    }
}
```

## 3.5 The Three Archetypes, Wired to the Spawner

| Archetype | RequiredRoadClass | TargetType / Behavior | Key modifier | Win condition |
|-----------|-------------------|-----------------------|--------------|---------------|
| **High-speed pursuit (Precision Strike)** | Highway / Arterial | VIPTransport convoy, `Evasive`+`Convoy` | Low EW, thick traffic (collateral risk) | Precision hit on flagged target hitbox, minimal collateral |
| **Indoor / tight-gap recon** | Indoor / Service | Static or slow objective, `Patrol` guards | RF loss indoors (concrete mass) | Dwell-capture recon markers, exfil with feed intact |
| **Counter-drone under EW** | Any / airborne | HostileDrone swarm, `Evasive` | **High EWIntensity** → severe RF degradation + control noise | Neutralize hostiles (EMP/kinetic) before they reach objective |

## 3.6 "System Prompt"–Style Director Brief

If you drive the director (or a design-time generator) with an LLM/behavior-tree brief,
here's the operating contract to give it. It is a **game-pacing brief**, not real-world
tasking:

```
ROLE: You are the Encounter Director for an FPV drone GAME. You compose fun,
fair, replayable missions from a live traffic simulation in a fictionalized city map.

GOALS (in priority order):
  1. Fairness: never spawn an unreachable or instantly-lethal encounter.
     Respect SpawnDistanceFromPlayer and current pilot skill rating.
  2. Pacing: follow the tension curve. Alternate high-intensity (pursuit,
     counter-drone-EW) with low-intensity (recon) so the pilot breathes.
  3. Variety: honor per-id cooldowns; avoid repeating the same archetype twice
     in a row unless the table has nothing else valid.
  4. Readability: every encounter must produce a clear HUD marker, audio cue,
     and objective text on spawn via the event bus.

CONSTRAINTS:
  - Only PROMOTE actors that already satisfy the encounter's road/target/LoS
    constraints; if none exist, request traffic density and wait — never teleport
    a target into the player's face.
  - EWIntensity only scales the RF/vision degradation and control-noise SYSTEMS.
    It never removes the pilot's control authority outright.
  - Collateral (civilian traffic) is a scoring penalty; weight encounters so
    avoiding it is always possible.
  - Everything you output is an in-engine spawn/score decision. You do not produce
    real-world guidance, coordinates, or hardware instructions.

OUTPUT (per decision): { EncounterId, PlacementCandidateId, EWIntensity,
TrafficDensityBias, Rationale } — consumed by UEventSpawnerSubsystem.
```

## 3.7 Tuning Knobs (all data, no recompile)
- Spawn weights, cooldowns, concurrency cap, pacing curve shape.
- `SpawnDistanceFromPlayer` and `MinLineOfSightToPlayer` per archetype.
- EW intensity → RF-curve mapping (shared with doc 1.3's RF data asset).
- Difficulty scaling vs. pilot rating (skill-based matchmaking of encounters).
