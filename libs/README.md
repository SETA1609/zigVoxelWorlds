# `libs/`

> External C/C++ stacks, each a **standalone git submodule** with its own `build.zig`, `LICENSE`, and CI. Each adapter bundles related libraries behind a single stable C ABI surface — per-stack granularity, not one-dir-per-lib.

This is the "Adapter sub-project" tier (§3) in [`external-libs-catalog.md`](../docs/external-libs-catalog.md). See that file for the decision tree and per-stack contents.

## Current submodules

| Submodule | Stack contents | Phase | Status |
| --- | --- | --- | --- |
| `zig-cpp-platform-stack-adapter/` | GLFW (v0) → pure-Zig X11/Wayland/Win32/Android (v1.x). Window · events · input · time · file I/O · per-OS native handle getters | 1 | landed |
| `zig-cpp-vulkan-stack-adapter/` | vulkan-zig · VMA · volk · shaderc — bindings + GPU memory + loader + shader compile | 1 / 4 / 7.5 | landed |

## Planned submodules

| Submodule | Stack contents | Phase |
| --- | --- | --- |
| `zig-cpp-physics-stack-adapter/` | Jolt Physics | 5 |
| `zig-cpp-audio-stack-adapter/` | miniaudio | 5 |
| `zig-cpp-net-stack-adapter/` | ENet (v1) → GameNetworkingSockets (v1.x for Steam relay) | 10 |
| `zig-cpp-asset-stack-adapter/` | cgltf + KTX-Software (libktx) + basis_universal + zstd | 4 / 6 |
| `zig-cpp-data-stack-adapter/` | toml++ + FlatBuffers (or Cap'n Proto — TBD) | 2 |
| `zig-cpp-ui-stack-adapter/` | **RmlUi** + FreeType + (HarfBuzz later if needed) — runtime UI engine + glyph rasterization. See [`specs/ui.md`](../docs/specs/ui.md) for the two-layer split (this adapter is the wrapper; `src/ui/` is the thin Zig binding) | 7.5 / 12 |
| `zig-cpp-tracy-stack-adapter/` | Tracy profiler client + scopes | 5 (gated by `-Dtracy=true`) |
| `zig-cpp-imgui-stack-adapter/` | Dear ImGui + ImGuizmo + imnodes | 1 / 12 (editor only) |
| `zig-cpp-mesh-stack-adapter/` | meshoptimizer | 4 / 6 |
| `zig-cpp-nav-stack-adapter/` | Recast + Detour | 8 |
| `zig-cpp-crash-stack-adapter/` | Crashpad | 12 / 13 |
| `zig-cpp-wasm-stack-adapter/` | wasm-micro-runtime (sandboxed mod runtime) | 14 |

## Held — needs SDK terms review

| Submodule | Stack contents | Blocker |
| --- | --- | --- |
| `zig-cpp-steam-stack-adapter/` | Steamworks SDK — Workshop, achievements, DLC gating | [Steamworks SDK Agreement](https://partner.steamgames.com/documentation/sdk_access_agreement) redistribution terms vs. Apache 2.0 public repo |

## Why per-stack granularity

Bundling related libraries inside one adapter sub-repo enforces **atomic version coherence**. Example: the Vulkan stack contains vulkan-zig + VMA + volk + shaderc; VMA's headers embed assumptions about specific Vulkan-1.x function signatures, vulkan-zig's generated bindings come from a specific `vk.xml` snapshot, shaderc emits SPIR-V targeting a specific Vulkan version — **all three must move together** or you get cryptic runtime errors. One sub-repo's `build.zig.zon` enforces that.

Same logic applies to other stacks: RmlUi + FreeType is one stack because RmlUi depends on FreeType's atlas API; physics stack stays single-lib because Jolt has no co-traveler.

## What does NOT live here

- **Pure Zig deps** — `build.zig.zon` direct entries (`zig-network`, etc.); see [`external-libs-catalog.md` § 1](../docs/external-libs-catalog.md)
- **Pure C deps with stable API** — `@cImport` or `addCSourceFile` direct; see [`external-libs-catalog.md` § 2](../docs/external-libs-catalog.md)
- **Engine source code** — `src/` and `modules/`
- **Asset content** — `core_pack/` and per-project `<project>/assets/`

## Submodule discipline

Each submodule:

- Owns its own `LICENSE` (preserves upstream MIT/Apache obligations per file)
- Pinned by commit SHA in `.gitmodules` and `build.zig.zon`
- CI runs on its own repo; the engine's CI verifies the integration only
- Exposes only `extern "C"` symbols across the boundary
- Never includes any reference-engine source (Hazel/Luanti/Godot/Unreal stay external — see [reference-engine memory](#))
