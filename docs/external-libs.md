# External Libraries — Integration Catalog

> Every third-party library zVoxRealms uses, and **how** it's integrated. Three integration styles with a decision tree. Replaces the older "all libs need adapter wrappers" model from the previous draft.

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
| **miniaudio** | <https://github.com/mackron/miniaudio> | MIT-0 / public domain | Audio playback, mixing, streaming | 6+ |
| **ENet** | <http://enet.bespin.org/> | MIT | UDP transport (candidate) | 10 (Multiplayer) |
| **zstd** | <https://github.com/facebook/zstd> | BSD-3 (pin permissive) | Chunk + save compression | 6 |
| **LZ4** (alternative to zstd) | <https://github.com/lz4/lz4> | BSD-2 | Faster, lower-ratio compression option | 6 |

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
| **glslang / slangc** | <https://github.com/KhronosGroup/glslang> | BSD-3 + others | MIT | GLSL → SPIR-V | 4 |
| **KTX-Software / Basis Universal** | <https://github.com/KhronosGroup/KTX-Software> · <https://github.com/BinomialLLC/basis_universal> | Apache-2.0 | **Apache-2.0** ⚠ | GPU-compressed texture loading (BC7/ASTC) — codec patent-grant matters | 4 |
| **FlatBuffers** | <https://github.com/google/flatbuffers> | Apache-2.0 | Apache-2.0 | Zero-copy structured save data | 13 |
| **Tracy** | <https://github.com/wolfpld/tracy> | BSD-3 | MIT | Real-time CPU/GPU profiling | 5 (gated by `-Dtracy=true`) |
| **libghostty** | <https://github.com/ghostty-org/ghostty> | MIT | MIT | Editor playtest log panel surface | 12 |
| **GameNetworkingSockets** (alt to ENet) | <https://github.com/ValveSoftware/GameNetworkingSockets> | BSD-3 | MIT | UDP transport (candidate) | 10 |
| **Steamworks SDK** | <https://partner.steamgames.com/> | Proprietary | n/a (separate repo, never open) | Steam Workshop, achievements (conditional, `-Dsteam=true`) | 14 |

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

Foundation work that unlocks editor + basic world:

1. **vulkan-zig** + **VMA adapter** + **GLFW** — open a window, render
2. **ImGui adapter** — editor scaffolding
3. **TOML parser** (Zig-native) — load game data
4. **Jolt adapter** — physics (Phase 5)
5. **glslang adapter** + **cgltf** + **KTX adapter** — asset pipeline (Phase 4)
6. **Tracy adapter** — profiling backend (Phase 5)
7. **ENet** or **GameNetworkingSockets adapter** — multiplayer (Phase 10)
8. **libghostty adapter** — editor playtest log (Phase 12)
9. **FlatBuffers adapter** + **zstd** — save format (Phase 13)
10. **Steamworks adapter** — Steam build (Phase 14)

---

## §6. Forbidden by policy

These categories must never appear in the dependency tree:

- **GPL / LGPL / AGPL / SSPL / Commons Clause** — incompatible with the Apache 2.0 engine licensing strategy (see [`licensing.md`](licensing.md) § Dependency policy)
- **Custom "non-commercial only" licenses** — would block shipping games
- **License-unknown / unattributed code** — never copy snippets from Stack Overflow into the engine without a license check

When a dependency is dual-licensed (zstd is BSD-3 OR GPL-2), pin the permissive option in `build.zig.zon` and note it in `LICENSES.md`.

---

## Updating this catalog

When adding or changing a dependency:

1. Place it in the right section (§1 / §2 / §3 / §4) per the decision tree
2. Add or update its row in the table with upstream URL, license SPDX, and phase
3. Update [`LICENSES.md`](../LICENSES.md) with the attribution
4. If the wrapped lib uses patent-prone tech (codecs, etc.), set the adapter sub-repo's `LICENSE` to Apache 2.0 instead of MIT
5. Cross-check: the entry should agree with [`tech-stack.md` § Observability / § Data Layer / etc.](tech-stack.md)
