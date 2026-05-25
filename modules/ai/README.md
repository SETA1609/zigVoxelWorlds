# `modules/ai/`

> Behavior trees + utility scoring + NavMesh pathfinding. Phase 8.

## What it provides

- Behavior tree runtime — data-driven trees from `<project>/assets/data/ai/trees/*.toml`
- Utility scoring — score-then-select for decision nodes; modders can register custom scorers via the C ABI
- NavMesh generation + A* pathfinding — via [`Recast/Detour`](https://github.com/recastnavigation/recastnavigation) from the (planned) navigation stack adapter
- Sensors — sight (cone + raycast), hearing (event-bus driven), memory (recent stimuli decay)
- Faction + disposition tables — drive whether NPCs treat the player as friend / neutral / enemy

## Reference patterns

- Recast/Detour for NavMesh — the standard open-source navigation stack
- Unreal MassEntity processors — the *idea* of running AI ticks as parallel ECS processor passes ([`engine-references.md`](../../docs/engine-references.md) § Unreal · MassEntity)
- No verbatim ports from Unreal or others

## Layering

Depends on `core/`, `scene/ecs`, `servers/physics_server` (raycasts for sight), the planned navigation stack adapter. MUST NOT import `editor/` (BT visual editor lives at `src/editor/panels/` and is a separate concern).

## Spec

[`specs/ai.md`](../../docs/specs/ai.md).
