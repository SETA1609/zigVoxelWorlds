# `src/servers/`

> Init level 1 — stable handle-only interfaces. The Godot "server" pattern.

Each `*_server.zig` is a thin, opaque-handle-typed Zig API. It hides the backend (Vulkan, Jolt, miniaudio, ENet, …) so that `scene/` and `modules/*/` never see raw C/C++ types.

Planned files:

- `render_server.zig` — meshes, materials, render targets, camera
- `voxel_server.zig` — chunk lifecycle, bulk-edit, meshing requests
- `physics_server.zig` — bodies, shapes, queries, stepping
- `audio_server.zig` — sources, buses, listeners
- `net_server.zig` — connections, channels, replication frames

**Reference patterns:** Godot `servers/rendering/rendering_server.h:64-117` (pure-virtual interface, `RID`-typed methods) and `servers/rendering/rendering_server_default.h:45-47, 80` (command-queue threading wrapper). See [`engine-references.md`](../../docs/engine-references.md) § Godot · Server pattern.

**Adapt note:** we use plain Zig structs + comptime backend selection — no vtables. One backend per build.

**Layering rule:** servers may import `core/` only. Backends live next door in `../backends/`; the wiring is done by `build.zig`, not by `servers/` calling into `backends/`.
