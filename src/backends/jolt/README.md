# `src/backends/jolt/`

> `physics_server` implementation. Wraps Jolt Physics through `libs/zig-cpp-physics-stack-adapter/` (planned).

Holds:

- World / body / shape lifecycle
- Stepping + sub-stepping
- Queries (raycast, overlap, sweep)
- Voxel-collision integration (chunks as static compound shapes)
- Game-thread ↔ physics-thread marshalling

**Reference patterns:** interface shape inspired by Unreal Chaos `ChaosMarshallingManager.h` — the game-thread/physics-thread marshalling pattern, not the solver. See [`engine-references.md`](../../docs/engine-references.md) § Unreal · Chaos Physics — architecture only.

See [`specs/physics.md`](../../../docs/specs/physics.md) for the planned API surface.
