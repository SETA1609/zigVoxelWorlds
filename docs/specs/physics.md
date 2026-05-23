# Physics Spec

> What `modules/physics_jolt/` provides. Roadmap: [`ROADMAP.md` § Phase 5](../ROADMAP.md). Adapter notes: [`external-libs-catalog.md` §3](../external-libs-catalog.md). Open questions: [`gaps.md`](../gaps.md).

## Scope

Physics is built as a module behind `PhysicsServer`. The Jolt C++ library is wrapped by a standalone adapter sub-repo (`zig-jolt-adapter`, MIT) — see [`external-libs-catalog.md` § 3](../external-libs-catalog.md).

## Components

- **Jolt integration** — built as a C++ static lib via the adapter's `build.zig`
- **Thin C ABI wrapper** — `extern "C"` boundary, every function `noexcept` (see [`cpp-style.md`](../cpp-style.md))
- **Physics world setup** — gravity, broadphase, world bounds
- **Character controller** — capsule + slide-on-slope + step-up
- **Voxel-to-collider conversion** — batched static geometry from chunks
- **Destructible voxel bodies** — chunk-region detach on damage threshold
- **Ragdolls** — multi-body articulated skeletons
- **Vehicles** — wheeled + tracked (lower priority; deferred unless a target game needs it)

## Server-pattern integration

Scene code holds `BodyHandle` (u64). `PhysicsServer` is the only Jolt-aware Zig code. Physics tick runs in a dedicated thread; main thread enqueues commands via the marshalling pattern (Unreal Chaos `ChaosMarshallingManager` reference).

## Reference patterns

- Unreal `Engine/Source/Runtime/Experimental/Chaos/Public/Chaos/ChaosMarshallingManager.h` — game/physics thread marshalling (architecture only — Jolt does the actual physics)

## Tracy integration

Phase 5 also flips `src/core/profile.zig` from no-op stubs to the real Tracy backend. Gated by `-Dtracy=true`. Never linked into shipped `libzvox-runtime`.
