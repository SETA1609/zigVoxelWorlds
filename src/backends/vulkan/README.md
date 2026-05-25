# `src/backends/vulkan/`

> `render_server` implementation. The only Zig code that talks to Vulkan directly (via Volk + VMA from `libs/zig-cpp-vulkan-stack-adapter/`).

Holds:

- Swapchain + frame pacing
- Renderpass / pipeline objects, descriptor caches
- Memory allocator wrapper (VMA via the stack adapter)
- Voxel chunk mesh upload + indirect draw
- Shader module loading (compiled SPIR-V from `editor/import/shader.zig`)

**Hardware target:** Intel UHD-class iGPU + i3 CPU at 50–60 FPS. No mesh-shader assumptions.

**Reference patterns:** Godot `servers/rendering/renderer_rd/renderer_compositor_rd.cpp` (forward+ + RD architecture). Unreal `NaniteStreamingManager.h:88-302` is read for the *streaming idea* only — Nanite itself is not portable to iGPU (see [`engine-references.md`](../../docs/engine-references.md) § Unreal · Nanite — conceptual only).

**Layering rule:** call into the Vulkan stack adapter's C ABI only. Never import `scene/`, `modules/`, or `editor/`.
