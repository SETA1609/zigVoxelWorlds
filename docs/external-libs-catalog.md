# External Libraries — Integration Catalog

> Every third-party library zVoxRealms commits to, and **how** it's integrated. Three integration styles with a decision tree.
>
> **Companion doc:** [`external-libs-survey.md`](external-libs-survey.md) — the candidate landscape per gap, with cross-references to the four reference engines. Workflow: survey first (what are my options?), catalog second (record the commitment + integration tier).

Cross-reference: [`tech-stack.md`](tech-stack.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`engine-references.md`](engine-references.md), [`licensing.md`](licensing.md), [`cpp-style.md`](cpp-style.md), [`LICENSES.md`](../LICENSES.md).

## Decision tree

When adding a new external dependency, ask in this order:

```text
1. Is there a mature Zig-native package?
   └─ Yes → use it directly via build.zig.zon (no adapter, no @cImport)
   └─ No  → continue

2. Is the library pure C with a simple API (single-header preferred)?
   └─ Yes → @cImport directly, or addCSourceFile in the consuming module
            (no separate adapters/<name>/ sub-project)
   └─ No  → continue

3. Is the library C++ or complex C, or a vendor SDK?
   └─ Yes → standalone adapter sub-repo with its own LICENSE
            (consumed by zVoxRealms via build.zig.zon or git submodule)
```

The three styles map to the three sections below.

---

## §1. Zig-native packages — use directly

Pure Zig dependencies pulled in via `build.zig.zon`. No C, no `@cImport`, no adapter. Just standard Zig dependency graph.

| Library | Upstream | Role | Phase |
| --- | --- | --- | --- |
| **vulkan-zig** | <https://github.com/Snektron/vulkan-zig> | Comptime-generated Vulkan bindings from `vk.xml` | 1 |
| **TOML parser** (zig-toml or similar) | <https://github.com/sam701/zig-toml> | TOML parsing for project data, manifests, scenes | 2 |
| **Math** | hand-written, or zig-gamedev/math | Vectors, matrices, quaternions, SIMD helpers | 1 |
| **Containers** | `std` | Hash maps, ArrayLists, etc. | always |
| **Allocators** | `std` | Arena, fixed-buffer, GeneralPurposeAllocator | always |

Rule: if a Zig-native package exists and is well-maintained, **default to it**. Don't write an adapter for libraries that already have idiomatic Zig bindings.

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
| **miniaudio** | <https://github.com/mackron/miniaudio> | MIT-0 / public domain | Audio playback, mixing, streaming | 7.5 |
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
- The C ABI boundary is the architectural commitment for mod compatibility ([`engine-vs-game.md`](engine-vs-game.md))

**Distribution: each adapter is a standalone sub-repo with its own `LICENSE`.** Consumed by zVoxRealms via `build.zig.zon` or git submodule. **Not** vendored in-tree under `zigVoxelWorlds/adapters/`. (Existing precedent: `libs/zig-cpp-vulkan-adapter/`.)

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
| **Jolt Physics** | <https://github.com/jrouwe/JoltPhysics> | MIT | MIT | Physics solver, characters, ragdolls | 5 |
| **VMA (Vulkan Memory Allocator)** | <https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator> | MIT | MIT | GPU memory management | 1 |
| **Dear ImGui** | <https://github.com/ocornut/imgui> | MIT | MIT | Editor / dev panels | 1 |
| **ImGuizmo** | <https://github.com/CedricGuillemet/ImGuizmo> | MIT | MIT | 3D transform gizmos in the scene editor | 12 |
| **imnodes** | <https://github.com/Nelarius/imnodes> | MIT | MIT | Node-graph UI for BT editor + material graph | 12 |
| **shaderc (over glslang)** | <https://github.com/google/shaderc> · <https://github.com/KhronosGroup/glslang> | Apache-2.0 / BSD-3 | MIT | GLSL → SPIR-V | 4 / 7.5 |
| **HarfBuzz** | <https://github.com/harfbuzz/harfbuzz> | MIT | MIT | Complex-script text shaping (CJK, Arabic, Devanagari) | 7.5 |
| **msdfgen** | <https://github.com/Chlumsky/msdfgen> | MIT | MIT | Multi-channel SDF font atlas generation (editor-time) | 7.5 / 12 |
| **msdf-atlas-gen** | <https://github.com/Chlumsky/msdf-atlas-gen> | MIT | MIT | Atlas packing on top of msdfgen (editor-time only) | 12 |
| **meshoptimizer** | <https://github.com/zeux/meshoptimizer> | MIT | MIT | Mesh LOD + vertex-cache opt + meshlets (critical for meshified chunks per [`specs/voxel.md`](specs/voxel.md)) | 4 / 6 |
| **basis_universal** | <https://github.com/BinomialLLC/basis_universal> | Apache-2.0 | **Apache-2.0** ⚠ | Universal texture transcoder → BC/ETC/ASTC (codec patent-grant matters) | 4 |
| **KTX-Software (libktx)** | <https://github.com/KhronosGroup/KTX-Software> | Apache-2.0 | **Apache-2.0** ⚠ | KTX2 container format (de-facto modern baked-texture format) | 4 |
| **Recast / Detour** | <https://github.com/recastnavigation/recastnavigation> | zlib | MIT | NavMesh generation + A* pathfinding for AI ([`specs/ai.md`](specs/ai.md)) | 8 |
| **Crashpad** | <https://chromium.googlesource.com/crashpad/crashpad/> | Apache-2.0 | Apache-2.0 | Out-of-process crash reporter for shipped builds ([`specs/diagnostics.md`](specs/diagnostics.md)) | 12 / 13 |
| **WAMR (wasm-micro-runtime)** | <https://github.com/bytecodealliance/wasm-micro-runtime> | Apache-2.0 | Apache-2.0 | Sandboxed third-party mod runtime (tiered with native `dlopen` for signed first-party mods) | 14 |
| **Tracy** | <https://github.com/wolfpld/tracy> | BSD-3 | MIT | Real-time CPU/GPU profiling | 5 (gated by `-Dtracy=true`) |
| **libghostty** | <https://github.com/ghostty-org/ghostty> | MIT | MIT | Editor playtest log panel surface | 12 |
| **GameNetworkingSockets** (v1.x alternative to ENet) | <https://github.com/ValveSoftware/GameNetworkingSockets> | BSD-3 | MIT | UDP transport — adopt for Steam relay in v1.x | post-1.0 |
| **Steamworks SDK** | <https://partner.steamgames.com/> | Proprietary | n/a (separate repo, never open) | Steam Workshop, achievements, DLC gating (conditional, `-Dsteam=true`) | 14 |

⚠ = wrap with Apache 2.0 because of patent-prone tech in the wrapped library.

For the licensing rationale of each adapter sub-repo, see [`licensing.md`](licensing.md) § Adapter sub-repos.

---

## §4. Borderline cases — pure C but worth wrapping

A pure-C library moves from §2 to §4 when **two or more** of the following hold:

- API surface is large (~50+ functions touching many subsystems)
- Consumers have high call-site density and would benefit from idiomatic Zig types
- We want to swap the underlying lib later (a thin wrapper is the abstraction point)
- The lib has unusual lifecycle/threading requirements that deserve a custom Zig surface

| Library | License | Why §4 not §2 | Future |
| --- | --- | --- | --- |
| **GLFW** | Zlib | Large API surface (window + input + monitor + gamepad). Plus we plan to replace with a pure Zig platform layer ([`tech-stack.md`](tech-stack.md#windowing--input)); the wrapper IS that abstraction point. | Graduates to §2 when the pure-Zig replacement lands; the wrapper then re-exports the Zig impl with the same shape. |

Default to §2 unless the criteria above are clearly met. Don't pre-emptively wrap "just in case" — that's premature abstraction.

---

## §5. Order of implementation

Foundation work that unlocks editor + basic world. Each step builds on prior ones; they're not parallel.

| Order | Adoption | Phase | Purpose |
| --- | --- | --- | --- |
| 1 | **vulkan-zig** (§1) + **VMA adapter** (§3) + **GLFW** (§4) | 1 | Open a window, render |
| 2 | **ImGui adapter** (§3) | 1 | Editor scaffolding |
| 3 | **TOML parser** (§1) | 2 | Load game data + manifests |
| 4 | **stb_image** (§2) + **cgltf** (§2) + **MikkTSpace** (§2) | 4 | Asset pipeline basics — image + model + tangents |
| 5 | **shaderc adapter** (§3) + **SPIRV-Reflect** (§2) | 4 / 7.5 | GLSL → SPIR-V + runtime descriptor reflection |
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
