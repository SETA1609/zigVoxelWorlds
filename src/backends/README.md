# `src/backends/`

> Init level 1 (paired with `servers/`) — the only Zig code that touches Vulkan, Jolt, miniaudio, ENet, etc. directly through their `extern "C"` ABIs (exposed by `libs/zig-cpp-*-adapter/`).

One subdirectory per backend. Each implements the contract declared by the paired `servers/<name>_server.zig`. The backend chosen at compile time is selected by `build.zig` flags (e.g. `-Dnet=enet` vs `-Dnet=gns`).

| Subdir | Servers it implements | Upstream lib (via `libs/`) |
| --- | --- | --- |
| `vulkan/` | `render_server` | `libs/zig-cpp-vulkan-stack-adapter/` |
| `jolt/` | `physics_server` | `libs/zig-cpp-physics-stack-adapter/` (planned) |
| `miniaudio/` | `audio_server` | `libs/zig-cpp-audio-stack-adapter/` (planned) |
| `enet/` | `net_server` | `libs/zig-cpp-net-stack-adapter/` (planned) |

**Reference patterns:** Godot `servers/rendering/renderer_rd/` and `drivers/vulkan/` are the closest analogs — interface in `servers/`, implementation behind it. See [`engine-references.md`](../../docs/engine-references.md) § Godot · Server pattern.

**Layering rule:** backends may import `core/` and call into the C ABI of the matching `libs/<stack>` adapter. They never import `scene/`, `modules/`, or `editor/`.
