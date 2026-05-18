# Third-Party Licenses

This document lists all third-party software bundled with, linked against, or distributed alongside zVoxRealms. Each entry includes the upstream project, license, and the role it plays in the engine. zVoxRealms itself is licensed under Apache License 2.0 — see [`LICENSE`](LICENSE).

> Scope: this file enumerates current dependencies as documented in [`external-libs.md`](docs/external-libs.md) and [`tech-stack.md`](docs/tech-stack.md). Update when adding a new dependency. Each release archive must ship this file in the distribution root so end users have a single place to read attributions.

## How to keep this file current

When you add a new dependency:

1. Add an entry in the appropriate section below with name, upstream URL, license SPDX identifier, and a one-line role description
2. If the upstream ships its own `LICENSE` / `NOTICE` / copyright file, copy the relevant attribution text into the entry (or `vendor/<name>/LICENSE`)
3. If the dependency is GPL/AGPL/SSPL — **stop**. These are incompatible with the Apache 2.0 licensing strategy. See [`docs/licensing.md`](docs/licensing.md) for the policy.
4. If the dependency has multiple licenses (e.g. zstd is BSD-3 or GPL-2.0), pin the permissive choice explicitly in `build.zig.zon` and note the chosen license here

---

## Bundled with the engine binary (dev + editor builds)

These libraries are linked into the engine binary (statically or dynamically) at build time. They ship with every distribution of the engine.

| Library | Upstream | License (SPDX) | Role |
| --- | --- | --- | --- |
| Vulkan-Headers / volk | <https://github.com/KhronosGroup/Vulkan-Headers> · <https://github.com/zeux/volk> | Apache-2.0 / MIT | Vulkan API headers and loader |
| VMA (Vulkan Memory Allocator) | <https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator> | MIT | GPU memory management |
| GLFW | <https://www.glfw.org/> | Zlib | Window + input (transitional; replaced by pure Zig platform later) |
| Dear ImGui | <https://github.com/ocornut/imgui> | MIT | Editor / dev panels |
| Jolt Physics | <https://github.com/jrouwe/JoltPhysics> | MIT | Physics solver, character controllers, ragdolls |
| cgltf | <https://github.com/jkuhlmann/cgltf> | MIT | glTF 2.0 model parsing |
| KTX-Software / Basis Universal | <https://github.com/KhronosGroup/KTX-Software> · <https://github.com/BinomialLLC/basis_universal> | Apache-2.0 | GPU-compressed texture loading (BC7 / ASTC) |
| glslang | <https://github.com/KhronosGroup/glslang> | BSD-3-Clause + others | GLSL → SPIR-V compilation |
| ENet | <http://enet.bespin.org/> | MIT | UDP transport (candidate for Phase 10 networking) |
| GameNetworkingSockets (alternative to ENet) | <https://github.com/ValveSoftware/GameNetworkingSockets> | BSD-3-Clause | UDP transport (candidate; pick one in Phase 10) |
| miniaudio | <https://github.com/mackron/miniaudio> | MIT-0 / public domain | Audio playback, mixing, streaming |
| Tracy | <https://github.com/wolfpld/tracy> | BSD-3-Clause | Real-time CPU/GPU profiling (`-Dtracy=true` only) |
| zstd | <https://github.com/facebook/zstd> | BSD-3-Clause (chosen over GPL-2.0) | Chunk + save compression |
| FlatBuffers | <https://github.com/google/flatbuffers> | Apache-2.0 | Zero-copy structured save data (Phase 13) |
| TOML parser (toml++ or zig-toml) | <https://marzer.github.io/tomlplusplus/> · <https://github.com/sam701/zig-toml> | MIT | TOML parsing for project data, manifests, mods |
| libghostty | <https://github.com/ghostty-org/ghostty> | MIT | Editor playtest log panel surface (`tools_enabled` only) |
| Neovim (bundled binary) | <https://neovim.io/> | Apache-2.0 | Code editor panel host (`tools_enabled` only) |
| OpenTelemetry SDK (optional) | <https://github.com/open-telemetry/opentelemetry-cpp> | Apache-2.0 | Production server telemetry (only when `[modules.telemetry]` enabled) |

## Optional, conditional dependencies

| Library | Upstream | License | When included |
| --- | --- | --- | --- |
| Steamworks SDK | <https://partner.steamgames.com/> | Proprietary (Valve Steamworks SDK Agreement) | Only with `-Dsteam=true`; only in Steam builds; never in the open-source binary downloads |

## Build tooling (used at build time, not shipped)

| Tool | License | Role |
| --- | --- | --- |
| Zig (compiler + bundled LLVM/Clang) | MIT (Zig) · Apache-2.0 with LLVM exceptions (LLVM) | Compiles everything; ships as the build-time toolchain |

---

## License compatibility summary

Every dependency above is **permissively licensed** and compatible with Apache 2.0 distribution. No GPL, AGPL, or SSPL dependencies are present in the engine code. This is a hard rule — see [`docs/licensing.md`](docs/licensing.md) § Dependency Policy.

## Attribution requirements you must preserve when shipping a game

When you export a project and ship it on Steam (or anywhere else), the resulting `libzvox-runtime.{so,dll}` statically links the subset of the above libraries that the project enables. The shipped game must include:

1. A copy of this `LICENSES.md` file (or an equivalent enumeration of the linked subset) in the distribution
2. The Apache 2.0 `LICENSE` file for the engine itself
3. The Neovim attribution **only** if Neovim is bundled (editor builds only; shipped games do not bundle Neovim)
4. Any per-asset attributions for game-specific assets (textures, music, fonts) that the game's authors used

The engine's export pipeline (Phase 13) should automatically copy `LICENSE` + a filtered `LICENSES.md` (containing only the modules linked into that specific build) into the export output directory.

---

## Game scripts and mods

Game scripts under `<project>/scripts/` and third-party mods under `<project>/mods/` are **not part of the engine** and use their own licenses chosen by their authors. The engine ABI does not impose a license on consumers.

Your games (the project owner's commercial releases) are typically **proprietary** under your own EULA — see [`docs/licensing.md`](docs/licensing.md) § Game IP.

---

## Reporting a missing attribution

If a third-party dependency is added to the build without being listed here, that is a bug. File an issue or open a PR.
