# `src/scene/ecs/`

> Archetype ECS — bitset-keyed archetype storage + processor scheduler + query API.

Planned files (per [`specs/ecs.md`](../../../docs/specs/ecs.md)):

- Component registry + bitset type
- Archetype table + chunked column storage
- Entity ↔ archetype index
- Query API (`view<A, B, !C>`)
- Processor / system runner with parallel scheduling

**Reference patterns:**

- Hazel `Hazel/src/Hazel/Scene/Components.h:16` — clean component structs (`IDComponent`, `TagComponent`, `TransformComponent`, …)
- Hazel `Hazel/src/Hazel/Scene/Entity.h` — entity wrapper around the archetype handle
- Unreal `MassEntity/Public/MassEntityTypes.h:28-33` — bitset archetype keys
- Unreal `MassEntityManager.h:95, 247, 383` + `MassProcessor.h:76, 201, 209` — batch APIs + processor dependency solver

See [`engine-references.md`](../../../docs/engine-references.md) § Hazel and § Unreal · MassEntity.

**Adapt note:** the heavy UObject / UScriptStruct welding in MassEntity does not port — Zig comptime types replace it. Target footprint: ~a few hundred lines of plain Zig.
