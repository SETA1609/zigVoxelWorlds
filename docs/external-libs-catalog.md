# External Libraries — Integration Catalog

> Every third-party library zVoxRealms commits to, and **how** it's integrated. Three integration styles with a decision tree.
>
> **Companion doc:** [`external-libs-survey.md`](external-libs-survey.md) — the candidate landscape per gap, with cross-references to the four reference engines. Workflow: survey first (what are my options?), catalog second (record the commitment + integration tier).

Cross-reference: [`tech-stack.md`](tech-stack.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`engine-references.md`](engine-references.md), [`licensing.md`](licensing.md), [`cpp-style.md`](cpp-style.md), [`LICENSES.md`](../LICENSES.md).

## Decision tree

When adding a new external dependency, ask in this order:

```text
1. Is there a mature Zig-native package?
   └─ Yes → continue to 1a
   └─ No  → continue to 2

   1a. Is this Zig package tightly version-coupled to a C++ neighbor we
       already wrap in an adapter (e.g. vulkan-zig ↔ VMA/volk/shaderc)?
       └─ Yes → fold it INTO that adapter sub-repo's build.zig.zon and
                re-export its API from the adapter's root.zig. Engine
                gets the Zig types via the adapter; no separate top-level
                dep. (See: vulkan-stack adapter — §3 row.)
       └─ No  → use directly via top-level build.zig.zon (§1)

2. Is the library pure C with a simple API (single-header preferred)?
   └─ Yes → @cImport directly, or addCSourceFile in the consuming module
            (no separate adapters/<name>/ sub-project — §2)
   └─ No  → continue to 3

3. Is the library C++ or complex C, or a vendor SDK?
   └─ Yes → standalone adapter sub-repo with its own LICENSE (§3)
            (consumed by zVoxRealms via build.zig.zon or git submodule)
```

The three styles map to the three sections below. Step 1a is the
version-coherence escape hatch — applies rarely, but the Vulkan stack
is the canonical case.

---

## §1. Zig-native packages — use directly

Pure Zig dependencies pulled in via `build.zig.zon`. No C, no `@cImport`, no adapter. Just standard Zig dependency graph.

| Library | Upstream | Role | Phase |
| --- | --- | --- | --- |
| **TOML parser** — must support **comment-and-formatting-preserving round-trip** (read → edit → write back without destroying user-authored comments) per [`specs/editor.md`](specs/editor.md) § Dual-authoring. Candidates: zig-toml + a custom CST layer, OR vendor `toml-edit` (Rust) via cbindgen, OR fork toml++ with a preserving emit pass. Plain "parse to struct, emit struct" libraries do NOT meet this requirement. | n/a — selection deferred until editor panel work begins (Phase 11/12) | TOML parsing + editor round-trip for project data, manifests, scenes, themes | 2 (read-only) / 11–12 (round-trip editor) |
| **Math** | hand-written, or zig-gamedev/math | Vectors, matrices, quaternions, SIMD helpers | 1 |
| **Containers** | `std` | Hash maps, ArrayLists, etc. | always |
| **Allocators** | `std` | Arena, fixed-buffer, GeneralPurposeAllocator | always |

Rule: if a Zig-native package exists and is well-maintained, **default to it**. Don't write an adapter for libraries that already have idiomatic Zig bindings.

> **Note:** [`vulkan-zig`](https://github.com/Snektron/vulkan-zig) used to be listed here but moved to §3 (folded into the Vulkan-stack adapter). It's still consumed Zig-native — the adapter just re-exports its API. See decision-tree step 1a above for when this consolidation applies.

---

## §2. Direct `@cImport` / `addCSourceFile` — pure C, simple API

Pure-C libraries (especially single-header ones) used directly from the consuming Zig module. No separate `adapters/<name>/` sub-project. The Zig consumer either:

- `@cImport` the C header and call functions directly, or
- `module.addCSourceFile(...)` to compile the lib's source alongside Zig code

A thin Zig wrapper module (e.g. `src/wrappers/miniaudio.zig`) is fine if the C API needs idiomatic Zig types — that's not the same thing as a full adapter sub-project.

| Library | Upstream | License | Role | Phase |
| --- | --- | --- | --- | --- |
| **cgltf** | <https://github.com/jkuhlmann/cgltf> | MIT | glTF 2.0 model parsing | 4 (Asset Pipeline) |
| **stb_image** | <https://github.com/nothings/stb> | public domain | PNG/JPG/BMP decode at import | 4 |
| **MikkTSpace** | <https://github.com/mmikk/MikkTSpace> | zlib | Universal tangent-space basis computation | 4 |
| **miniaudio** (opt-in audio backend) | <https://github.com/mackron/miniaudio> | MIT-0 / public domain | **Opt-in richer audio backend per [`specs/audio.md`](specs/audio.md).** Default audio is SDL3 (via the platform-stack adapter); miniaudio is added via `project.toml [audio] backend = "miniaudio"` for 3D spatial, doppler, reverb. Daggerfall opts in; arena_modes / rogue_tower / likely Atelier + Stardew use SDL3 audio. | 7.5 |
| **ENet** | <http://enet.bespin.org/> | MIT | UDP transport (candidate primary) | 10 (Multiplayer) |
| **miniupnpc** | <http://miniupnp.free.fr/> | BSD-3 | UPnP port-forward for self-hosted dedicated servers | 10 |
| **zstd** | <https://github.com/facebook/zstd> | BSD-3 (pin permissive) | Chunk + save compression | 6 / 13 |
| **LZ4** (hot-path alternative) | <https://github.com/lz4/lz4> | BSD-2 | Faster-decompress option — only adopt if zstd profiling shows a hot-path miss | 6 |
| **xxHash** | <https://github.com/Cyan4973/xxHash> | BSD-2 | Non-crypto integrity hash for save chunks + assetdb | 4 / 13 |
| **BLAKE3** | <https://github.com/BLAKE3-team/BLAKE3> | Apache-2.0 / MIT / CC0 | Content-address hashes for assetdb + mod signing | 4 / 14 |
| **SPIRV-Reflect** | <https://github.com/KhronosGroup/SPIRV-Reflect> | Apache-2.0 | Runtime descriptor-set introspection (auto-bind from shader) | 7.5 |
| **FreeType** | <https://freetype.org/> | FTL (BSD-like) | Glyph rasterization (universal across Godot/Luanti/Unreal) | 7.5 |
| **backward-cpp** | <https://github.com/bombela/backward-cpp> | MIT | Pretty stack traces in dev builds | 12 |
| **filewatch** | <https://github.com/ThomasMonkman/filewatch> | MIT | Header-only file-system watcher for hot-reload | 12 |
| **doctest** | <https://github.com/doctest/doctest> | MIT | Single-header C++ test framework for adapter test code | always (test-only) |

Reference for `@cImport`: <https://ziglang.org/documentation/0.13.0/#cImport>

---

## §3. Adapter sub-project — C++ or complex C

Libraries that need an `extern "C"` wrapper layer because:

- Zig's `@cImport` cannot translate C++ classes, templates, or name-mangled symbols
- The library needs per-lib build flags (RTTI, exceptions, `-std=c++23`)
- The library has unusual lifecycle / threading / build-system requirements that benefit from isolation

### The two C ABIs — don't confuse them

There are two distinct `extern "C"` surfaces in the project. They serve different purposes and shouldn't be conflated:

| ABI | Where it lives | Who calls it | Why C |
| --- | --- | --- | --- |
| **(1) Internal adapter bridge** (this §3) | Inside each `libs/zig-*-adapter/` sub-repo (`adapter.h` / `adapter.cpp`) | Only the adapter's own Zig wrapper (`adapter.zig`) | Mechanical: Zig `@cImport` can't translate C++; per-lib build flags |
| **(2) Public mod/script ABI** ([`specs/c-abi.md`](specs/c-abi.md)) | Engine ↔ mods/scripts boundary | Mods, DLCs, game scripts — anything outside the engine binary | Architectural: stable, versioned, append-only; callable from any language that speaks C |

Engine code (`src/`) **never calls either C ABI directly.** It calls into idiomatic Zig wrappers — the adapter's `adapter.zig` re-exports Zig-shaped types over ABI #1. ABI #2 is *implemented by* the engine, not consumed by it.

So: adapters exist for §3's three reasons above, **not** because of mod compatibility. Mod compatibility is ABI #2's concern, lives in a different doc, and is unrelated to whether any specific library gets an adapter.

**Distribution: each adapter is a standalone sub-repo with its own `LICENSE`.** Consumed by zVoxRealms via `build.zig.zon` or git submodule. **Not** vendored in-tree under `zigVoxelWorlds/adapters/`. (Existing precedent: `libs/zig-cpp-vulkan-stack-adapter/`.)

Each adapter sub-repo contains:

```text
zig-<lib>-adapter/
├── LICENSE              # MIT default; Apache 2.0 for codec/IP-prone wrappers (see licensing.md)
├── README.md
├── .clang-format        # Google preset + project tweaks (mirror of zVoxRealms root)
├── build.zig            # builds the C/C++ as a static lib
├── build.zig.zon
├── src/
│   ├── adapter.h        # extern "C" surface (the stable ABI)
│   ├── adapter.cpp      # C++ → C ABI bridge
│   └── adapter.zig      # idiomatic Zig wrapper on top of the C ABI
└── vendor/<lib>/        # upstream source as git submodule or fetched tarball
```

C++ adapter code follows [`cpp-style.md`](cpp-style.md) — Google C++ Style Guide as baseline, with project-specific deviations (exceptions allowed inside adapters, forbidden across the C boundary; RTTI per wrapped-lib needs).

| Library | Upstream | License | Adapter license | Role | Phase |
| --- | --- | --- | --- | --- | --- |
| **Vulkan-stack** (meta-package — see note below) | bundle: vulkan-zig · VMA · volk · shaderc | mixed permissive | MIT | Vulkan stack: bindings + GPU memory + loader + shader compile. Single sub-repo enforces version coherence | 1 / 4 / 7.5 |
| **Platform-stack** (meta-package — see note below) | **SDL3** (decision 2026-05-26 — supersedes the earlier GLFW v0 / pure-Zig v1.x plan) | zlib (SDL3) | MIT | Window · events · input · time · file I/O · per-OS native handle getters · **gamepad (Steam Input mapping)** · **sensor (Steam Deck gyro / mobile IMU)** · **haptic (rumble)** · **clipboard** · **filesystem paths (XDG / FOLDERID / NSDirectory / Android internal storage)** · **power info** · **IME / text input**. **No Vulkan dep and no cross-adapter dep** — engine bridges per-OS getters to vulkan-stack's per-OS creators ([`specs/platform.md`](specs/platform.md)) | 1 |
| **Jolt Physics** | <https://github.com/jrouwe/JoltPhysics> | MIT | MIT | Physics solver, characters, ragdolls | 5 |
| **Dear ImGui** | <https://github.com/ocornut/imgui> | MIT | MIT | Editor / dev panels | 1 |
| **ImGuizmo** | <https://github.com/CedricGuillemet/ImGuizmo> | MIT | MIT | 3D transform gizmos in the scene editor | 12 |
| **imnodes** | <https://github.com/Nelarius/imnodes> | MIT | MIT | Node-graph UI for BT editor + material graph | 12 |
| **RmlUi** | <https://github.com/mikke89/RmlUi> | MIT | MIT | **Document-UI layer (opt-in per project)** — HTML-subset (RML) + CSS-subset (RCSS) runtime for projects that need bespoke designed UI. Engine ships a widget kit on SDL3 primitives by default (see [`specs/ui.md`](specs/ui.md) § Two-layer architecture); RmlUi is added via `project.toml [ui] document = true`. Bundled into `libs/zig-cpp-ui-stack-adapter/` alongside FreeType. arena_modes does NOT link this; Daggerfall/Stardew/Atelier do. | 7.5 / 12 |
| **HarfBuzz** | <https://github.com/harfbuzz/harfbuzz> | MIT | MIT | Complex-script text shaping (CJK, Arabic, Devanagari) | 7.5 |
| **msdfgen** | <https://github.com/Chlumsky/msdfgen> | MIT | MIT | Multi-channel SDF font atlas generation (editor-time) | 7.5 / 12 |
| **msdf-atlas-gen** | <https://github.com/Chlumsky/msdf-atlas-gen> | MIT | MIT | Atlas packing on top of msdfgen (editor-time only) | 12 |
| **meshoptimizer** | <https://github.com/zeux/meshoptimizer> | MIT | MIT | Mesh LOD + vertex-cache opt + meshlets (critical for meshified chunks per [`specs/voxel.md`](specs/voxel.md)) | 4 / 6 |
| **basis_universal** | <https://github.com/BinomialLLC/basis_universal> | Apache-2.0 | **Apache-2.0** ⚠ | Universal texture transcoder → BC/ETC/ASTC (codec patent-grant matters) | 4 |
| **KTX-Software (libktx)** | <https://github.com/KhronosGroup/KTX-Software> | Apache-2.0 | **Apache-2.0** ⚠ | KTX2 container format (de-facto modern baked-texture format) | 4 |
| **Recast / Detour** | <https://github.com/recastnavigation/recastnavigation> | zlib | MIT | NavMesh generation + A* pathfinding for AI ([`specs/ai.md`](specs/ai.md)) | 8 |
| **Crashpad** | <https://chromium.googlesource.com/crashpad/crashpad/> | Apache-2.0 | Apache-2.0 | Out-of-process crash reporter for shipped builds ([`specs/diagnostics.md`](specs/diagnostics.md)) | 12 / 13 |
| **WAMR (wasm-micro-runtime)** | <https://github.com/bytecodealliance/wasm-micro-runtime> | Apache-2.0 | Apache-2.0 | **Sandboxed third-party mod runtime — default tier for all Workshop / unsigned mods.** Tiered with native `dlopen` for signed first-party mods (full perf, full trust). WASM tier rules: WASI NOT exposed; per-mod resource limits (1 ms/tick CPU budget, 64 MB linear memory, network-deny by default, filesystem scoped to mod data dir); `proc_exit` intercepted; deterministic for multiplayer + replays. See [`specs/mod-manager.md`](specs/mod-manager.md) § Security / sandbox considerations. | 14 |
| **Tracy** | <https://github.com/wolfpld/tracy> | BSD-3 | MIT | Real-time CPU/GPU profiling | 5 (gated by `-Dtracy=true`) |
| **libghostty** | <https://github.com/ghostty-org/ghostty> | MIT | MIT | Editor playtest log panel surface | 12 |
| **GameNetworkingSockets** (v1.x alternative to ENet) | <https://github.com/ValveSoftware/GameNetworkingSockets> | BSD-3 | MIT | UDP transport — adopt for Steam relay in v1.x | post-1.0 |
| **Steamworks SDK** | <https://partner.steamgames.com/> | Proprietary | n/a (separate repo, never open) | Steam Workshop, achievements, DLC gating (conditional, `-Dsteam=true`) | 14 |

⚠ = wrap with Apache 2.0 because of patent-prone tech in the wrapped library.

For the licensing rationale of each adapter sub-repo, see [`licensing.md`](licensing.md) § Adapter sub-repos.

### Note on the Vulkan-stack meta-package

The Vulkan-stack row is structured differently from the other §3 rows because it bundles **one Zig-native package plus three C++ libraries** in a single sub-repo. The sub-repo lives at [`libs/zig-cpp-vulkan-stack-adapter/`](../libs/zig-cpp-vulkan-stack-adapter/) and follows the project-wide naming convention `zig-cpp-<name>-stack-adapter` for meta-package adapters.

What's inside:

| Bundled lib | Role | How it's exposed |
| --- | --- | --- |
| **vulkan-zig** ([github.com/Snektron/vulkan-zig](https://github.com/Snektron/vulkan-zig)) — MIT | Zig-native Vulkan bindings generated from `vk.xml` | **Re-exported as-is**: `pub const vk = @import("vulkan");` in the adapter's `root.zig`. Engine gets idiomatic Zig types, error sets, comptime dispatch tables. No C-ABI boundary. |
| **VMA** — MIT | GPU memory allocator | C++ → extern "C" bridge → idiomatic Zig wrapper. Engine calls Zig API. |
| **volk** — MIT | Vulkan function loader | C lib; small extern "C" surface; Zig wrapper. |
| **shaderc** — Apache-2.0 (wraps glslang BSD-3) | GLSL → SPIR-V | C++ → extern "C" bridge → idiomatic Zig wrapper. |

Engine code looks like:

```zig
const vk_stack = @import("vulkan_stack");
const vk      = vk_stack.vk;       // re-exported vulkan-zig — full typed API
const vma     = vk_stack.vma;      // typed Zig wrapper over VMA
const shaderc = vk_stack.shaderc;
const volk    = vk_stack.volk;

try cb.beginRenderPass(&info, .@"inline");
const buf = try vma.createBuffer(allocator, &buf_info, &alloc_info);
```

Why bundle: VMA's headers embed assumptions about specific Vulkan-1.x function signatures; vulkan-zig's generated bindings come from a specific `vk.xml` snapshot; shaderc emits SPIR-V targeting a specific Vulkan version. **All three must move together** or you get cryptic runtime errors. One sub-repo's `build.zig.zon` enforces atomic version coherence.

What's **not** in the stack:

- **SDL3** — windowing/input/etc, orthogonal to Vulkan. Vulkan can take surfaces from any window source (X11/Wayland/Win32/Android raw handles via SDL3's property API). SDL3 lives inside the **Platform-stack adapter** (see Platform-stack note below); keeping it out of the Vulkan stack means the two adapters can move independently.
- **SPIRV-Reflect** — pure C, not Vulkan-version-coupled (it walks SPIR-V binaries against the SPIR-V spec, not against a Vulkan version). Stays in §2.
- **post-processing / material pipeline / frame graph** — these are engine code, not third-party libs. They live in `src/render/`.

### Note on the Platform-stack meta-package

Second instance of the meta-package pattern (the first is the Vulkan-stack above). The sub-repo lives at `libs/zig-cpp-platform-stack-adapter/`.

The platform adapter exposes a stable Zig API to the engine (window, events, action-mapped input, time, file I/O, per-OS native handle getters: `getX11Handle` / `getWaylandHandle` / `getWin32Handle` / `getAndroidHandle`, gamepad, sensor, haptic, clipboard, filesystem paths, power, IME). The implementation backend is internal to the adapter.

**No Vulkan dependency and no cross-adapter dependency.** Surface creation lives in the Vulkan-stack adapter via its own per-OS functions (`createX11Surface` / `createWaylandSurface` / `createWin32Surface` / `createAndroidSurface`), each taking only raw OS primitives (pointers + integers). The engine wires the two together in a small `src/render/surface.zig` helper that comptime-branches on `builtin.target.os.tag`. **No shared type crosses the boundary** — both adapters are fully standalone and reusable in isolation. Reference precedent: SDL3's `SDL_GetWindowProperties()` per-backend property keys (`SDL_PROP_WINDOW_X11_DISPLAY_POINTER`, `SDL_PROP_WINDOW_WAYLAND_DISPLAY_POINTER`, etc.) + Vulkan's own `VK_KHR_*_surface` extension pairs.

| Version | Backend | Why |
| --- | --- | --- |
| **v0.x — v0.5** | GLFW (zlib) — hello-world only | Initial scaffolding; superseded |
| **v0.6 onward** | **SDL3** (zlib) | Decision 2026-05-26: Android + Steam Deck + future Switch coverage, Steam Input gamepad mapping, gyro/IMU sensor, IME, haptic — all free with SDL3; GLFW has none of these |
| ~~**v1.x** Pure-Zig native~~ | **Withdrawn 2026-05-26** | Maintaining native X11 / Wayland / Win32 / Android backends in pure Zig was solo-team-aspirational. SDL3 covers all those platforms with shipped reliability. The optional "single library, multiple backends as files" architecture in [`specs/platform.md`](specs/platform.md) still allows a native backend to be added later if a concrete reason emerges. |

Engine code looks identical across the migration:

```zig
const platform = @import("platform");
const render   = @import("render");   // engine's own bridge module

const window = try platform.Window.create(.{
    .title = "zVoxRealms",
    .size = .{ .w = 1280, .h = 720 },
    .vulkan_compatible = true,
});

while (platform.nextEvent()) |ev| switch (ev) {
    .key      => |k| input.dispatch(k),
    .resize   => |r| renderer.handleResize(r),
    .close    => running = false,
    .gamepad  => |g| input.dispatchGamepad(g),
};

if (input.actionPressed(.jump)) player.jump();

// Surface creation lives in the engine's bridge helper, not in either
// adapter. The helper calls platform's per-OS getter (getX11Handle /
// getWin32Handle / etc.) and the matching vulkan-stack creator
// (createX11Surface / createWin32Surface / etc.). Comptime-resolved
// per target — no runtime cross-platform branching.
const surface = try render.createSurface(vk_instance, window);
```

Backend changes are sub-repo-internal; the engine never sees them. The same Zig API surface stays stable; only the SDL3 calls inside the adapter change as SDL3 itself evolves.

What's deliberately **not** in the platform stack:

- **Audio** — the default audio backend is SDL3 itself (audio is part of the SDL3 vendored set already). miniaudio is the opt-in richer-audio backend per [`specs/audio.md`](specs/audio.md); when enabled, miniaudio handles its own platform abstraction (PulseAudio/ALSA/CoreAudio/WASAPI inside miniaudio). Both backends route through `src/audio/`'s LCD Zig API.
- **Vulkan rendering** — separate Vulkan-stack adapter; only the surface-creation handoff crosses the boundary
- **High-level input mapping (UI focus graph, mod-defined actions)** — engine code in `src/input/` consumes the platform layer's raw input + action-mapped events but adds the focus graph / mod-action layers itself
- **Filesystem watcher** — `filewatch` (§2) is small and works on raw paths; doesn't need platform-adapter integration

Reference patterns: we adopt [SDL3](https://www.libsdl.org/) directly (one stable C API across decades; X11/Wayland/Cocoa/Win32/Android backends rotate underneath). The Zig-native wrap means the engine sees a Zig API, never raw SDL — same C-ABI-only-across-boundary discipline as every other adapter.

Detailed contract: [`specs/platform.md`](specs/platform.md).

---

## §4. Borderline cases — pure C but worth wrapping

A pure-C library moves from §2 to §4 when **two or more** of the following hold:

- API surface is large (~50+ functions touching many subsystems)
- Consumers have high call-site density and would benefit from idiomatic Zig types
- We want to swap the underlying lib later (a thin wrapper is the abstraction point)
- The lib has unusual lifecycle/threading requirements that deserve a custom Zig surface

| Library | License | Why §4 not §2 | Future |
| --- | --- | --- | --- |
| *(no current entries — GLFW used to be a §4 candidate; the Platform-stack adapter now uses SDL3 instead, see §3)* | | | |

Default to §2 unless the criteria above are clearly met. Don't pre-emptively wrap "just in case" — that's premature abstraction. §4 is intentionally narrow — most libs route through §2 or §3.

---

## §5. Order of implementation

Foundation work that unlocks editor + basic world. Each step builds on prior ones; they're not parallel.

| Order | Adoption | Phase | Purpose |
| --- | --- | --- | --- |
| 1 | **Vulkan-stack adapter** (§3 — bundles vulkan-zig + VMA + volk + shaderc) + **Platform-stack adapter** (§3 — SDL3 backend per the 2026-05-26 decision) | 1 | Open a window, render |
| 2 | **ImGui adapter** (§3) | 1 | Editor scaffolding |
| 3 | **TOML parser** (§1) | 2 | Load game data + manifests |
| 4 | **stb_image** (§2) + **cgltf** (§2) + **MikkTSpace** (§2) | 4 | Asset pipeline basics — image + model + tangents |
| 5 | **SPIRV-Reflect** (§2) | 4 / 7.5 | Runtime descriptor reflection (shaderc itself is in the Vulkan-stack adapter from step 1) |
| 6 | **basis_universal adapter** (§3) + **KTX-Software adapter** (§3) | 4 | Compressed texture transcode + KTX2 container |
| 7 | **meshoptimizer adapter** (§3) | 4 / 6 | Mesh LOD + vertex cache + meshlets for meshified chunks |
| 8 | **Jolt adapter** (§3) | 5 | Physics (per [`specs/physics.md`](specs/physics.md)) |
| 9 | **Tracy adapter** (§3) | 5 | Profiling backend wired behind `profile.zig` |
| 10 | **miniaudio** (§2) | 7.5 | Audio runtime |
| 11 | **FreeType** (§2) + **HarfBuzz adapter** (§3) + **msdfgen adapter** (§3) | 7.5 | UI text — rasterization + shaping + SDF atlases |
| 12 | **Recast/Detour adapter** (§3) | 8 | AI navmesh + A* per [`specs/ai.md`](specs/ai.md) |
| 13 | **ENet** (§2) + **miniupnpc** (§2) | 10 | Multiplayer transport + UPnP for self-hosted |
| 14 | **libghostty adapter** (§3) | 12 | Editor playtest log surface |
| 15 | **msdf-atlas-gen adapter** (§3) + **ImGuizmo adapter** (§3) + **imnodes adapter** (§3) + **filewatch** (§2) | 12 | Editor convenience |
| 16 | **backward-cpp** (§2) + **Crashpad adapter** (§3) | 12 / 13 | Dev stack traces + shipped-build crash reports |
| 17 | **zstd** (§2) + **xxHash** (§2) + **BLAKE3** (§2) | 13 | Save format compression + integrity + content addressing |
| 18 | **WAMR adapter** (§3) + **Steamworks adapter** (§3) | 14 | Sandboxed mods + Steam build |
| 19 | **GameNetworkingSockets adapter** (§3) | post-1.0 | Adopt for Steam relay in v1.x — replaces ENet for shipped Steam builds |

---

## §5.5. Library validation strategy — exercise adapters in a reference C++ host before engine adoption

**Rationale.** Each `libs/zig-cpp-*-adapter/` sub-repo wraps a C/C++ library behind a stable C ABI. Bugs in the C ABI shape, build wiring, or platform behavior surface late — usually only when engine code starts depending on the adapter. To catch them earlier, an adapter can be **dropped into a known-working C++ host** that already uses the same upstream library, and exercised against real workloads before the engine consumes it.

Pattern source: integrating a freshly-built `libs/<x>-stack-adapter/` into a reference engine ([`engine-references.md`](engine-references.md) catalogs which engines use which upstream libraries — Luanti is the closest match for our use cases, since it ships voxel multiplayer on Linux + Windows + Android using SDL2 + ENet).

**This is a workflow, not a build step.** Reference-engine integration work happens in a separate repo from `zigVoxelWorlds/` (the reference engine, possibly a personal fork). Code never flows from the reference engine back into zVoxRealms; only **insight**. The reverse direction — zVoxRealms adapter installed in the reference engine — is the validation path. See [`engine-references.md` § Legal](engine-references.md) for license discipline; Luanti's LGPL specifically forbids reverse code flow.

### Tier A — best fit (clean swap or augmentation in a typical voxel reference engine)

| Adapter | Why it validates well | Why it matters for zVoxRealms |
| --- | --- | --- |
| **Tracy** | Augmentation only — wrap `TracyZoneScoped` around hot loops in the reference engine's server tick + meshgen + map save. Doesn't replace anything. | Validates Zig-as-C++-build-system wiring on a real C++ host. Quick payoff: ~50 LoC integration in the reference. |
| **Net (ENet)** | Many voxel reference engines (Luanti, Veloren, custom engines) already use ENet directly. Replace `#include <enet/enet.h>` with the adapter's C ABI calls. Real multi-client traffic exercises the wrapper. | **Strongest signal** for `modules/multiplayer/`. ENet API surface is small (~30 functions); good shape match. |
| **meshoptimizer** | Add as a post-pass on the reference engine's chunk-mesh generator. Doesn't replace anything — augmentation. | Pre-validates the mesh-opt wrapper for `modules/voxel_core/` greedy-mesh output. |

### Tier B — useful but moderate effort

| Adapter | Notes |
| --- | --- |
| **Crashpad** | Replaces the reference engine's signal-handler scaffolding. Validates the crash-reporting pipeline (handler process, symbol upload, minidump) on a real shipped-binary scenario. |
| **Dear ImGui** | Add as a debug overlay in the reference engine (does not replace formspec / native UI). Validates ImGui-stack adapter against an OpenGL context — useful portability check. |

### Tier C — skip for reference-engine validation

These adapters don't map cleanly onto a typical voxel reference engine's existing architecture; integration cost outweighs signal:

- **Vulkan stack** — most voxel reference engines are OpenGL or Irrlicht-based; swapping the renderer is a full rewrite
- **Platform stack** (SDL3) — if the reference uses SDL2 via IrrlichtMt (e.g. Luanti), the SDL3 swap is real but moderate work; if it uses GLFW/native, larger
- **Physics (Jolt)** — coupled to the engine's node system; not a clean swap
- **Audio (miniaudio)** — most voxel engines use OpenAL Soft; different shape
- **UI (RmlUi)** — replacing the engine's UI is rewriting all UI code
- **WASM (WAMR)** — replacing the engine's scripting layer is core-engine work
- **NavMesh (Recast/Detour)** — most voxel engines have no NavMesh; you'd add a feature, not test the adapter against existing code

### Workflow rules

1. **One-way code flow**: zVox adapter → reference engine, never the reverse. Per [`engine-references.md` § Legal](engine-references.md): Luanti is LGPL — copying Luanti code into Apache-2.0 zVoxRealms is engine-killing.
2. **Separate sessions**: don't open zVoxRealms + the reference repo in the same editor/LLM context. Cross-pollination is the realistic contamination vector for a solo dev.
3. **Time-box each validation**: pick one concrete deliverable per adapter (e.g. "Tracy integrated around server tick + meshgen", "ENet swap behind the adapter's C ABI"). Don't open-end "modernize the reference."
4. **Engine code never depends on the reference engine.** Validation lives separately. The output of a validation pass is: confidence + an adapter README note ("validated against \<host\> at \<version\>") + bug-fix commits in the adapter sub-repo if the validation surfaced issues.

### Recommended sequence

For zVoxRealms's Phase 1+ work:

1. **Tracy first** (smallest, fast feedback on Zig-build wiring)
2. **ENet second** (highest-signal — multiplayer is Phase 10 critical path)
3. **meshoptimizer third** (Phase 4 / 6 mesh optimization landing)

Tier B adapters (Crashpad, ImGui-stack) validate when their phases approach. Tier C adapters skip the reference-host validation entirely and validate directly in zVoxRealms when engine code lands.

---

## §6. Forbidden by policy

### Categorical

These categories must never appear in the dependency tree:

- **GPL / LGPL / AGPL / SSPL / Commons Clause** — incompatible with the Apache 2.0 engine licensing strategy (see [`licensing.md`](licensing.md) § Dependency policy)
- **Custom "non-commercial only" licenses** — would block shipping games
- **License-unknown / unattributed code** — never copy snippets from Stack Overflow into the engine without a license check

When a dependency is dual-licensed (zstd is BSD-3 OR GPL-2), pin the permissive option in `build.zig.zon` and note it in `LICENSES.md`.

### Concrete named libraries we've rejected

Surfaced during the [`external-libs-survey.md`](external-libs-survey.md) analysis. Listed by name so a future PR auditor can grep-check:

| Lib | Reason | Use instead |
| --- | --- | --- |
| **SDL_mixer** | LGPL | miniaudio (§2) |
| **SDL_image** | LGPL | stb_image (§2) |
| **libnice** | LGPL | libjuice (ISC) if NAT traversal needed |
| **GMP** | LGPL (Luanti vendors it, their GPL absorbs; we can't) | std math + Zig stdlib bigint |
| **gettext libintl** | LGPL | Write a ~200-line Zig `.po` parser per [`specs/localization.md`](specs/localization.md) |
| **FBX SDK** (Autodesk) | Proprietary, royalty-bearing | ufbx (MIT) if FBX needed |
| **Bink Video** (RAD) | Proprietary, paid | No video — we ship in-engine cutscenes per [`specs/scene.md` § Scripted cutscenes](specs/scene.md) |
| **assimp** | BSD-3 but heavy + drags exception machinery | cgltf (§2) + ufbx if needed |
| **OpenSSL** (full) | Apache-2.0 but huge surface | mbedtls if TLS needed; LibreSSL also acceptable |

When evaluating a new candidate: cross-reference [`external-libs-survey.md`](external-libs-survey.md) for prior analysis before adding to either the catalog or this forbidden list.

---

## Updating this catalog

When adding or changing a dependency:

1. **Consult [`external-libs-survey.md`](external-libs-survey.md) first** — the survey records the candidate analysis (license, what each reference engine uses, recommendation). If the lib is new to both docs, add it to the survey before the catalog.
2. Place it in the right section (§1 / §2 / §3 / §4) per the decision tree at the top of this doc
3. Add or update its row in the table with upstream URL, license SPDX, and phase
4. Update [`LICENSES.md`](../LICENSES.md) with the attribution
5. If the wrapped lib uses patent-prone tech (codecs, etc.), set the adapter sub-repo's `LICENSE` to Apache 2.0 instead of MIT
6. Cross-check: the entry should agree with [`tech-stack.md` § Observability / § Data Layer / etc.](tech-stack.md)
7. If the new lib affects an existing spec doc (e.g. adding Recast/Detour updates [`specs/ai.md`](specs/ai.md)), update the spec's "Reference patterns" or "Implementation sketch" section to point at this catalog entry
