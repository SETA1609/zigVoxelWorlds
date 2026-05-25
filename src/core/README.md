# `src/core/`

> Init level 0 — runs before everything else. No deps on any other engine layer.

Foundational primitives that the rest of the engine assumes are already there:

- Allocators (general-purpose, arena, fixed-buffer)
- Math (vec/mat/quat, voxel-grid integer math)
- Logging (sinks abstract; see [observability-three-tier memory](../../docs/specs/diagnostics.md))
- Handle table (`u64` opaque handles — the Godot `RID` pattern)
- TOML parser front-end (parses once at import time, emits cached binary)
- Asset DB (GUID ↔ source-path map, content hashes)

**Reference patterns:** Godot `core/` ([`engine-references.md`](../../docs/engine-references.md) § Godot · Server pattern); handles correspond to Godot `RID` and Unreal `FNetRefHandle`.

**Layering rule:** nothing in `core/` may import from `servers/`, `scene/`, `modules/`, `editor/`, or `project_manager/`.
