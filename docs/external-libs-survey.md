# External Libraries — Candidate Survey

> The landscape of C/C++ libraries we could use, grouped by remaining-task bucket, cross-referenced against what the four reference engines actually ship (Hazel · Luanti · Godot · Unreal). **What we commit to** lives in [`external-libs-catalog.md`](external-libs-catalog.md); this doc is the menu we pick from.
>
> Workflow: when a new task needs a third-party lib, read this survey for the option set, decide, then add the commitment to the catalog with its integration tier (§1 / §2 / §3 / §4). The catalog is the source of truth for "what's in the build."

## Why two docs

| Doc | Answers | When to read |
| --- | --- | --- |
| **This survey** | "What are my options? What does each engine use? What's the licensing trap?" | When picking a lib for a new gap / phase |
| [`external-libs-catalog.md`](external-libs-catalog.md) | "What's actually committed? Which tier (Zig-native / direct cImport / adapter sub-repo)? Build order?" | When wiring the lib into `build.zig` / writing an adapter |

The split mirrors the [`gaps.md`](gaps.md) ↔ [`gap-references.md`](gap-references.md) split: one doc asks the question, the other points at the answer.

## License compatibility legend

Per [`licensing.md`](licensing.md) and [`feedback_reference_engine_no_verbatim`](.) memory:

- ✅ **Apache-2.0 / MIT / BSD-2 / BSD-3 / zlib / ISC / Unlicense / public domain (CC0)** — fully compatible with engine Apache 2.0
- ⚠ **MPL-2.0 / BSL-1.0** — weak copyleft, file-level only, acceptable but flag in the catalog
- ❌ **GPL / LGPL / AGPL / SSPL / Commons Clause / "non-commercial only"** — **forbidden** (would force the engine open or block commercial games)

When a lib is dual-licensed (e.g. zstd is BSD-3 OR GPL-2), **pin the permissive option** in `build.zig.zon` and record it in [`LICENSES.md`](../LICENSES.md).

## Cross-engine universal picks — the safe bets

Libraries used by **all or nearly all** of the four reference engines. These are the lowest-risk choices because everyone has battle-tested them in production:

| Lib | Job | Godot | Luanti | Unreal | Hazel |
| --- | --- | --- | --- | --- | --- |
| **FreeType** | Font rasterization | ✅ | ✅ | ✅ | (uses msdf-atlas-gen output) |
| **zstd** | Compression | ✅ | ✅ | partial | — |
| **Recast / Detour** | Navmesh + A* | ✅ | — | ✅ (Detours) | — |
| **Dear ImGui** | Editor / debug UI | (own) | (own) | (own) | ✅ |
| **glslang / shaderc** | GLSL → SPIR-V | ✅ | — | ✅ | — |
| **stb_image** _or_ libpng+libjpeg-turbo | PNG/JPG decode | ✅ (libpng/libjpeg-turbo) | ✅ | ✅ (via FreeImage) | ✅ (stb_image) |
| **Jolt Physics** | Physics solver | ✅ (4.3+) | — | — | — (Box2D 2D-only) |
| **MikkTSpace** | Tangent-space basis | implicit | — | ✅ | — |

Where everyone converges, **so do we** — no contrarian picks unless we have a specific reason.

Where the four diverge (ECS / scripting / GUI engine / network transport), the choice is a project decision and we pick what fits zVoxRealms' constraints, not what's popular.

---

## Per remaining-task bucket — candidate matrices

Cross-reference: [`gaps.md` § Remaining`](gaps.md) for which gap each bucket maps to.

### A. Save format + compression (§3 #6 — Phase 13)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **zstd** | ✅ BSD-3 | Save + asset compression — Pareto-optimal speed/ratio | Godot · Luanti | **Adopt** (already in catalog) |
| **LZ4** | ✅ BSD-2 | Faster-decompress alternative for hot paths (chunk streaming) | Godot · Unreal | **Hold** — only if zstd profiling shows a hot-path miss |
| **xxHash** | ✅ BSD-2 | Non-crypto integrity hash for save chunks | Godot · Unreal | **Adopt** for save-chunk checksums |
| **BLAKE3** | ✅ Apache-2.0 / MIT / CC0 | Content-address hashes — assetdb GUIDs, mod signing | Unreal | **Adopt** — single C file, trivial cImport |
| **flatbuffers** | ✅ Apache-2.0 | IDL-driven binary; zero-copy reads; schema evolution | Unreal | **Skip for v1.0** — we own all readers; IDL adds compile-time deps without payoff. Revisit if modders need to author save data |
| **Cap'n Proto** | ✅ MIT | Strict-schema alternative to flatbuffers | — | Skip — same reasoning as flatbuffers |

**Verdict:** zstd + xxHash + BLAKE3 + hand-rolled section layout.

### B. Network transport + NAT (§3 #25–#28 — Phase 10)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **ENet** | ✅ MIT | Reliable UDP — small, proven, well-understood | Godot · Luanti | **Adopt** for v1.0 |
| **GameNetworkingSockets** | ✅ BSD-3 | Valve stack; Steam relay; built-in encryption | Unreal (via Steam plugin) | **Defer to v1.x** — its key value is Steam relay, only matters once shipping |
| **libdatachannel** | ⚠ MPL-2.0 | WebRTC for browser + NAT | — | Skip unless WebGPU port forces it |
| **libjuice** | ✅ ISC | ICE/STUN/TURN client only | — | Hold — only if we need P2P traversal without Steam |
| **miniupnpc** | ✅ BSD-3 | UPnP port-forward for self-hosted dedicated servers | Godot | **Adopt** — pairs naturally with ENet |
| ~~libnice~~ | ❌ LGPL | NAT traversal | — | **Forbidden** |

**Verdict:** ENet + miniupnpc.

### C. AI runtime (`specs/ai.md` — Phase 8)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **Recast / Detour** | ✅ zlib | NavMesh build + A* + tile rebuild (the only realistic choice) | Godot · Unreal | **Adopt** as adapter sub-repo |
| **BehaviorTree.CPP** | ✅ MIT | Reference BT engine | — | **Read-only reference, do not link** — implement BT in Zig (per [`specs/ai.md`](specs/ai.md)) |

**Verdict:** Recast/Detour adapter (`libs/zig-recast-adapter`, MIT). BT lives in Zig.

### D. Asset pipeline + textures (Phase 4)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **stb_image** | ✅ public domain | PNG/JPG/BMP decode | Hazel | **Adopt** — single header, trivial |
| **libpng + libjpeg-turbo** | ✅ libpng / BSD | SIMD-tuned decode | Godot · Luanti · Unreal | Hold — only if stb_image's perf becomes a problem |
| **basis_universal** | ✅ Apache-2.0 | Universal texture transcoder → BC/ETC/ASTC at load | Godot · Unreal | **Adopt** as Apache-2.0 adapter (codec patent grant) |
| **KTX-Software (libktx)** | ✅ Apache-2.0 | KTX2 container — the modern baked-texture format | Godot | **Adopt** as Apache-2.0 adapter (already planned) |
| **meshoptimizer** | ✅ MIT | Mesh LOD + vertex-cache opt + meshlets — critical for meshified chunks per [`specs/voxel.md`](specs/voxel.md) | Godot | **Adopt** as adapter — its C++ underneath is a §3 case |
| **ufbx** | ✅ MIT | FBX importer without Autodesk SDK | Godot | **Adopt** if any project needs FBX; defer if glTF is enough |
| **cgltf** | ✅ MIT | Single-header glTF reader | — | **Adopt** (already in catalog) |
| **tiniergltf** | ✅ MIT | Tiny glTF reader (Luanti's pick) | Luanti | Alternative to cgltf; cgltf is more mature |
| **astcenc** / **etcpak** | ✅ Apache/BSD | ASTC/ETC compression at bake time | Godot | **Adopt** if/when we need mobile texture formats |
| **MikkTSpace** | ✅ zlib | Universal "compute tangents" algorithm | Unreal | **Adopt** — small single file |
| ~~FBX SDK (Autodesk)~~ | ❌ proprietary, royalty-bearing | FBX import | — | **Forbidden** — use ufbx |
| ~~assimp~~ | ✅ BSD-3 but heavy | Multi-format model loader | Hazel | Skip — too heavy; cgltf+ufbx covers our needs |

**Verdict:** stb_image + cgltf + basis_universal + libktx + meshoptimizer + MikkTSpace. Each as the integration tier the catalog assigns.

### E. Shaders (Phase 7.5 — materials + post-processing)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **glslang** | ✅ BSD-3 (Khronos) | GLSL → SPIR-V (the reference compiler) | Godot · Unreal | Indirect — via shaderc |
| **shaderc** | ✅ Apache-2.0 | Google's friendly wrapper around glslang | — | **Adopt** as adapter (already planned as "glslang/slangc" in the catalog) |
| **SPIRV-Cross** | ✅ Apache-2.0 | SPIR-V → HLSL/MSL/GLSL transpilation | Godot | Defer to v1.x cross-platform / Metal |
| **SPIRV-Reflect** | ✅ Apache-2.0 | Runtime descriptor-set introspection — auto-bind | — | **Adopt** as direct cImport (small C surface) |
| **slang** (NVIDIA) | ✅ Apache-2.0 + LLVM Exception | Modern shading language — modules, generics, autodiff | — | **Hold** — promising but non-standard shader source; defer to v1.x |

**Verdict:** shaderc adapter + SPIRV-Reflect direct.

### F. UI text + fonts (Phase 7.5)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **FreeType** | ✅ FTL (BSD-like) | Glyph rasterization | Godot · Luanti · Unreal | **Adopt** — universal pick |
| **HarfBuzz** | ✅ MIT | Complex-script text shaping (Arabic, Devanagari, CJK ligatures) | Godot | **Adopt** as adapter (C++ underneath, C API surface) |
| **ICU** | ✅ Unicode | Locale text iteration, BiDi, line-break | Godot · Unreal | **Defer** — its ~30 MB data footprint hurts tiny-game export; revisit when CJK becomes critical |
| **msdfgen** | ✅ MIT | Multi-channel SDF font atlas — sharp at any zoom | Godot | **Adopt** as adapter (editor-time) |
| **msdf-atlas-gen** | ✅ MIT | Tool around msdfgen — packs atlases | Hazel | **Adopt** as adapter (editor-time only — not shipped with games) |
| **libunibreak** | ✅ zlib (dual w/ LGPL — pin zlib) | UAX#14 line-break | — | Hold — only if HarfBuzz's break logic is insufficient |
| ~~gettext libintl~~ | ❌ LGPL | .po loader at runtime | Luanti | **Forbidden** — write a ~200-line Zig .po parser instead |

**Verdict:** FreeType + HarfBuzz + msdfgen + msdf-atlas-gen.

### G. Crash reporting + diagnostics (`specs/diagnostics.md`, §3 #37)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **Breakpad** | ✅ BSD-3 | Cross-platform minidump on signal | Unreal | Alternative to Crashpad |
| **Crashpad** | ✅ Apache-2.0 | Chromium's replacement for Breakpad — out-of-process | — | **Adopt** as adapter — out-of-process is the more reliable pattern |
| **backward-cpp** | ✅ MIT | Single-header stack-trace pretty-printer | — | **Adopt** as direct cImport — dev builds only |
| **Sentry-native** | ✅ MIT | Hosted aggregation + symbol upload | — | Optional — only if we pay for hosted Sentry post-v1 |
| **PLCrashReporter** | ✅ MIT | Apple-platform crash reporter | Unreal | Defer — macOS/iOS is post-v1 |

**Verdict:** Crashpad adapter (shipped builds) + backward-cpp direct (dev).

### H. Memory allocators + budget (§2.2.H)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **mimalloc** | ✅ MIT | Drop-in allocator with per-heap accounting | Unreal | **Hold** — only adopt if measured |
| **jemalloc** | ✅ BSD-2 | Per-arena stats; mature profiling | Unreal | Hold — same as mimalloc |
| **Tracy** | ✅ BSD-3 | Per-frame allocation events | (already adapter) | (already committed) |

**Verdict:** stick with Zig's `Allocator` interface — per-subsystem attribution is the whole point of the type. Tracy hooks for in-flight visibility. **Do not link mimalloc/jemalloc** until profiling shows a problem.

### I. Animation runtime (`specs/animation.md`, Phase 7.5)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **ozz-animation** | ✅ MIT | Guerrilla-Games-origin runtime; data-oriented; indie-darling | — | **Skip for voxel-character** — its triangle-mesh assumptions don't fit voxel rigs. Implement skeletal animation in Zig (Unreal `AnimSequence` taxonomy as the *concept* reference per [`gap-references.md` § 2.1.A](gap-references.md)) |
| **ACL** (Animation Compression Library) | ✅ MIT (Nicholas Frechette) | Best-in-class clip compression | Unreal (4.x+) | **Hold for v1.x** — adopt if save size becomes a problem |

**Verdict:** zero animation libs linked at v1.0. Pure Zig.

### J. Modding sandbox (§3 #22 + #24 — Phase 14)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **WAMR** (wasm-micro-runtime) | ✅ Apache-2.0 | Small WASM runtime — interpreter + AOT, embedded-friendly | — | **Adopt** as adapter — small footprint, sandboxed third-party mods |
| **wasmtime** | ✅ Apache-2.0 + LLVM Exception | Bytecode Alliance runtime — secure, fast | — | Alternative; WAMR is smaller |
| **wasmer** | ✅ MIT | Alternative WASM runtime | — | Alternative |

**Verdict:** **Tier the mod loader.** Signed first-party mods → native `dlopen` (existing plan). Third-party mods → **WAMR adapter** (sandboxed). This is the biggest architectural decision still open before Phase 14.

### K. Testing (§2.2.G — §3 #32)

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **doctest** | ✅ MIT | Single-header C++ test framework | Godot | **Adopt** as direct cImport — for C++ adapter test code |
| **Catch2** | ⚠ BSL-1.0 | C++ test framework | Luanti · Unreal | Skip — doctest is faster + simpler |
| **Google Test** | ✅ BSD-3 | C++ test framework | — | Skip — heavier than needed |

**Verdict:** Zig's built-in test runner for Zig code (per [`feedback_learning_mode_guard`](.) — user writes tests by hand). doctest direct cImport for C++ adapter tests.

### L. Editor convenience (Phase 12)

Already have ImGui via the catalog. Worth adding:

| Lib | License | Purpose | Used by | Recommendation |
| --- | --- | --- | --- | --- |
| **ImGuizmo** | ✅ MIT | 3D transform gizmos in the scene editor | Hazel | **Adopt** as adapter (ImGui companion) |
| **imnodes** (or **ImNodeFlow**) | ✅ MIT | Node-graph UI for BT editor + material graph | — | **Adopt** as adapter — depends on BT visual editor priority |
| **filewatch** | ✅ MIT | Hot-reload file watcher | Hazel | **Adopt** as direct cImport — header-only |
| **spdlog** | ✅ MIT | Structured logging | Hazel | **Skip** — Tracy + libghostty cover our needs |

**Verdict:** ImGuizmo + imnodes + filewatch.

---

## How the survey connects to the catalog

For each library marked **Adopt** above, the catalog gets a row:

| Decision in survey | Catalog tier | Catalog action |
| --- | --- | --- |
| "Adopt — single-header C" | §2 (direct cImport / addCSourceFile) | Add row to §2 table |
| "Adopt as adapter" | §3 (adapter sub-repo) | Add row to §3 table + create `libs/zig-<lib>-adapter/` sub-repo |
| "Adopt as Apache-2.0 adapter" | §3 with Apache-2.0 LICENSE (codec patent grant) | Same as above, adapter LICENSE = Apache-2.0 |
| "Hold" / "Skip" | not in catalog | No action; revisit when measured |
| "Forbidden" | §6 (forbidden) | Add to §6 explicitly |

The catalog is **append-mostly** — items rarely leave it. If a "Hold" turns into "Adopt" later, the survey decision is revised here, then the catalog gets a new row.

---

## Forbidden — quick-reference (full list in catalog §6)

| Lib | Why |
| --- | --- |
| **SDL_mixer / SDL_image** | LGPL — even dynamic linking creates compliance overhead for shipped games |
| **libnice** | LGPL |
| **GMP** | LGPL — Luanti vendors it, their GPL absorbs; we can't |
| **gettext libintl** | LGPL — write a Zig .po parser instead |
| **FBX SDK** (Autodesk) | Proprietary, royalty-bearing — use ufbx |
| **Bink Video** (RAD) | Proprietary, paid — and we ship no video per [`specs/scene.md` § Scripted cutscenes](specs/scene.md) |

See [`external-libs-catalog.md` § 6](external-libs-catalog.md) for the authoritative forbidden list.

---

## Audit trail — where these recommendations come from

Each lib's "used by" claim is verified against the actual third-party tree of the corresponding `$REFS/` clone:

- `$REFS/godot/thirdparty/` — Godot's vendored libs
- `$REFS/luanti-custom/lib/` + `find_package` calls in `src/CMakeLists.txt` — Luanti's deps
- `$REFS/UnrealEngine/Engine/Source/ThirdParty/` — Unreal's licensed third-party tree
- `$REFS/Hazel/Hazel/vendor/` — Hazel's vendor tree

When updating this survey, re-run the inventory commands documented in the conversation that produced this doc, not from memory.

---

## Updating this survey

When the landscape shifts (new lib appears, an old one becomes unmaintained, license changes):

1. Update the relevant bucket's matrix in this doc
2. If the recommendation changes, update [`external-libs-catalog.md`](external-libs-catalog.md) to match
3. If a forbidden lib appears in a dependency tree, raise it before merging — per [`CONTRIBUTING.md`](../CONTRIBUTING.md)

This doc and the catalog are paired — never let them drift.
