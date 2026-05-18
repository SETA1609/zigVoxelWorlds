# ECS Spec

> Archetype-based ECS in pure Zig. Roadmap: [`ROADMAP.md` § Phase 7](../ROADMAP.md). Reference patterns: [`engine-references.md` → Hazel + Unreal MassEntity](../engine-references.md).

## Scope

`src/scene/ecs/` — archetype-based ECS in pure Zig. No third-party dependency; comptime + bitset archetype keys + processor scheduler.

## Components

- **Archetype storage** — bitset-keyed; entities of identical component-sets are stored together for cache coherency
- **System scheduler** — declarative dependency graph between processors; parallel execution where dependencies allow
- **Query caching** — translate Hazel's view/group patterns to Zig comptime queries
- **Threaded entity updates** — workers pull from the scheduler's runnable queue

## Reference patterns

- Hazel `src/Hazel/Scene/Components.h:16` — clean component struct shapes
- Hazel `src/Hazel/Scene/Entity.h` — entity-wrapping-handle pattern
- Hazel `src/Hazel/Scene/Scene.h` — registry / views / groups
- Unreal `Engine/Source/Runtime/MassEntity/Public/MassEntityTypes.h:28-33` — bitset archetype keys
- Unreal `MassProcessor.h` + `MassProcessorDependencySolver.h` — processor scheduler

## Open decisions

- `Handle` layout — u64 `(generation, index)` vs `(server_id, generation, index)` ([`planning-gaps.md` #7](../planning-gaps.md))
- Component size limits + per-archetype memory budgets
- Lifecycle: when do entities get GC'd, deferred-destroy queue semantics

## Milestone (from ROADMAP)

10k entities updating at frame budget on low-end PC.
