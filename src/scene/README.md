# `src/scene/`

> Init level 2 — the world model. Stores `Handle` values only; the Vulkan/Jolt/audio objects live behind `servers/` + `backends/`.

Planned files:

- `world.zig` — top-level world struct, chunk grid, entities
- `chunk.zig` — chunk descriptor (the in-RAM record, not the mesh)
- `entity.zig` — entity wrapper around an archetype-ECS handle
- `orchestrator.zig` — `orchestrator.toml` loader, scene instancing
- `ecs/` — archetype ECS implementation (see [its README](ecs/README.md))

**Layering rule:** scene may import `core/` and `servers/`. It MUST NOT import any `backends/<x>/` directly — that breaks the handle abstraction.

**Reference patterns:** Hazel `Hazel/src/Hazel/Scene/Scene.h` (registry/views) — see [`engine-references.md`](../../docs/engine-references.md) § Hazel.
