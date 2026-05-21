# Engine References — What to Implement from Hazel, Luanti, Godot, Unreal

> What we are borrowing, from where, and at what depth. Action-oriented, not a reference catalog. Cross-reference: [`vision.md`](vision.md), [`mission.md`](mission.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`ROADMAP.md`](ROADMAP.md).
>
> **Per-gap resolution patterns:** see [`gap-references.md`](gap-references.md) — maps each missing system in [`gaps.md`](gaps.md) to the specific reference-engine files that solve it, with adaptation notes.

Paths use `$REFS/<engine>/` — set `REFS` to wherever you cloned the reference engines (e.g. `export REFS=~/refs` then `git clone https://github.com/godotengine/godot $REFS/godot`). The repos referenced: Hazel, luanti-custom (private fork of Luanti), godot, UnrealEngine.

Goal-fit tags: ✅ implement directly · ⚠ conceptual inspiration only · ✗ misleading reference (don't borrow).

---

## ⚠ Legal: reference engines are for understanding, not copying

**Hard rule for this entire document:** every reference below is an **adaptation or inspiration** source. We **read, understand, and reimplement from scratch in Zig**. We do **not**:

- Copy code verbatim into zVoxRealms (any reference engine)
- Translate a function line-by-line, even with structural cosmetic changes
- Copy-paste-then-modify
- Include reference-engine headers in our build
- Vendor reference-engine source files anywhere in the zVoxRealms tree

The acceptable workflow: open the reference file → understand the *pattern* (data layout, algorithm shape, API design) → close the file → write a Zig equivalent that solves the same problem with our own architecture.

### Why this rule is non-negotiable

| Engine | License | Risk of verbatim copy |
| --- | --- | --- |
| **Unreal Engine** | Epic Games [Unreal EULA](https://www.unrealengine.com/eula/source) — source-available, **not open source** | **Highest.** Direct copies into an Apache 2.0 project violate the EULA. Epic has the legal team to enforce. Anything that looks like translated Unreal source can trigger DMCA + cease-and-desist + lawsuit. zVoxRealms could be argued to "build a competing engine" using Unreal source — explicitly forbidden by the EULA. |
| **Luanti** | [LGPLv2.1+](https://github.com/luanti-org/luanti/blob/master/LICENSE.txt) — copyleft | **Engine-killing.** Verbatim copies pull LGPL obligations into zVoxRealms. Mixing LGPL with Apache 2.0 in the same binary forces the whole engine relicensed under (L)GPL. The entire Apache 2.0 strategy ([`licensing.md`](licensing.md)) breaks. |
| **Godot** | [MIT](https://github.com/godotengine/godot/blob/master/LICENSE.txt) | License-compatible — but verbatim copies still require MIT attribution preserved per file, which is administrative debt and a signal of poor engineering. Adapt the pattern, write your own. |
| **Hazel** | varies (educational/tutorial origin, mixed terms) | Same as Godot — license-compatible if attribution is preserved, but direct ports are bad form. Adapt the pattern. |

### What "adaptation" looks like in practice

✅ **Safe — adaptation:**

- "I read `NaniteStreamingManager.h:88-302` to understand the cluster-page hierarchy concept. I'm writing my own chunk-page streamer in Zig using a different data layout."
- "Luanti's `VoxelManipulator` taught me the bulk-edit pattern (acquire region → mutate → propagate lighting → release). My Zig API has different function shapes but solves the same problem."
- "MassEntity uses a bitset to key archetypes. My Zig ECS does the same conceptually, but the bitset is comptime-sized and the storage layout is different."

⚠ **Risky — derivative work, avoid:**

- "I translated `voxel.cpp:244-260` from C++ to Zig, kept the same variable names and structure."
- "I rewrote Iris's `ReplicationFragment` into Zig but kept the class shape + method names + private internals 1:1."

❌ **Forbidden — verbatim copy:**

- Pasting any reference-engine file (or substantial section) into the zVoxRealms tree
- Including reference-engine headers in `build.zig`
- Vendoring reference-engine source under `vendor/` or `libs/` (even with attribution)

### Contributor expectations

PRs that look like translated reference code will be **declined** regardless of license. The maintainer (and AI reviewers — see [`guard.md`](guard.md)) check for this. Even Apache 2.0-licensed `vulkan-zig` or MIT-licensed `cgltf` are consumed as **dependencies**, never as copied-in source.

If you find yourself wanting to copy code, stop and write a one-paragraph "pattern summary" of what the original does, then implement from that summary without re-reading the source.

---

---

## Reading order — when implementing Phase X, read these first

| Phase | What to read first |
| --- | --- |
| Phase 2 (module system + server pattern) | Godot § Module system + § Server pattern (below); [`specs/project-manager.md`](specs/project-manager.md) for the re-exec context |
| Phase 3 (voxel core) | Luanti section (below); [`specs/voxel.md`](specs/voxel.md) |
| Phase 5 (Jolt physics) | [`specs/physics.md`](specs/physics.md); Unreal Chaos `ChaosMarshallingManager.h` for the threading pattern |
| Phase 7 (ECS) | Hazel `Scene/*` + Unreal MassEntity sections (below); [`specs/ecs.md`](specs/ecs.md) |
| Phase 8 (gameplay) | [`specs/gameplay.md`](specs/gameplay.md); no single reference engine — design original |
| Phase 9 (scene + instancing) | [`specs/scene.md`](specs/scene.md); Godot scene-instance UX |
| Phase 10 (multiplayer) | Unreal Iris section (below); Luanti `servermap.cpp` / `clientmap.cpp`; [`specs/multiplayer.md`](specs/multiplayer.md) |
| Phase 11 (Project Manager) | Godot § Project Manager (below); [`specs/project-manager.md`](specs/project-manager.md) |
| Phase 12 (editor) | Hazel/Hazelnut + Unreal Editor sections (below); [`specs/editor.md`](specs/editor.md) |
| Phase 13 (export) | Godot § Export (below) — note divergence; [`engine-vs-game.md`](engine-vs-game.md) |
| Phase 14 (modding) | Luanti § layered mods + Unreal § plugin descriptors (below); [`engine-vs-game.md` § 8](engine-vs-game.md) |

---

## Implementation priority

The four engines pay off in this order:

| When | What | From | Why |
| --- | --- | --- | --- |
| Phase 2 | Module system (`modules/<name>/{config, register_types, src}` + comptime dispatch) | Godot + Unreal Projects | Backbone for everything else |
| Phase 2 | Server pattern (Scene → Server → Backend, opaque `Handle`) | Godot servers | Threading, hot-reload, serialization unlock |
| Phase 3 | Voxel core (chunk + greedy meshing + lighting + bulk-edit API) | Luanti | Voxel-first goal; this IS the engine's core |
| Phase 7 | Archetype ECS (bitset-keyed archetype, processor scheduler, query) | Hazel + Unreal MassEntity | Classless RPG + magic + NPCs |
| Phase 10 | Handle-based delta replication | Unreal Iris | 4p co-op + 40–50p dedicated |
| Phase 11 | Project Manager (re-exec, single binary) | Godot | Engine-as-app UX |
| Phase 12 | Editor mode tools + detail customization | Hazel + Godot + Unreal Editor | Voxel brush + biome painter + property panels |
| Phase 13 | PCK three-location pack loader | Godot | Asset bundle for exported games |
| Phase 14 | Layered mod loader (descriptor-driven) | Luanti + Unreal Projects | First-class modding |

---

## Hazel — Clean ECS + ImGui editor (`$REFS/Hazel/`)

**What to implement:**

- **EnTT-style component structs and entity wrapper** → port to a Zig archetype ECS with `Handle`-based entities
- **ImGui editor panel patterns** (`SceneHierarchy`, `Properties`) → reuse layout/widget idioms for our voxel brush, skill editor, recipe editor
- **Clean runtime/editor separation** (Hazel vs Hazelnut) → mirrors our `tools_enabled` split

**Source pointers:**

- `Hazel/src/Hazel/Scene/Components.h:16` — `IDComponent`, `TagComponent`, `TransformComponent`, `CameraComponent`
- `Hazel/Hazel/src/Hazel/Scene/Entity.h` — entity wraps `entt::entity` with `GetComponent`/`AddComponent`/`HasComponent`
- `Hazel/Hazel/src/Hazel/Scene/Scene.h` — registry, views, groups
- `Hazelnut/src/Panels/SceneHierarchyPanel.cpp:35` — ImGui-based component property editor (closest analog to what we want)

**Goal fit:** ✅ direct

---

## Luanti — Voxel core + multiplayer (`$REFS/luanti-custom/`)

**What to implement:**

- **VoxelManipulator-style bulk-edit API** in Zig — fast multi-voxel writes + lighting propagation
- **Chunk (`MapBlock`) structure** with embedded mesh, LOD variants, metadata
- **Mesh-gen strategies** (greedy meshing) ported to Zig + compute shaders
- **Procedural generation framework** (biomes, noise) as the seed-deterministic baseline (per [save-model](ARCHITECTURE.md#cross-cutting-concerns))
- **Active-block interest management** for multiplayer sync
- **Layered mod system** (`builtin/`+`games/`+`mods/`) — replace Lua API with native C ABI + TOML data (per [`engine-vs-game.md`](engine-vs-game.md))

**Source pointers:**

- `src/voxel.h:24` + `src/voxel.cpp` — `VoxelManipulator`
- `src/voxelalgorithms.h/cpp` — lighting propagation, `VoxelLineIterator`, `update_block_border_lighting`
- `src/mapblock.h` — chunk structure
- `src/client/meshgen/` — meshing
- `src/mapgen/` — biomes, noise
- `src/servermap.cpp` + `src/clientmap.cpp` — `emergeBlock`, interest management, network serialization

**Goal fit:** ✅ direct (Lua API ✗ — replace with our ABI)

---

## Godot — Engine-as-app shell (`$REFS/godot/`)

Five distinct patterns to lift. All paths verified.

### Project Manager — re-exec model · ✅

Same binary doubles as PM and editor. Opening a project spawns a fresh child process; the PM quits. Crash isolation for free.

- `editor/project_manager/project_manager.cpp:559-608` — `_open_selected_projects` builds `--path <dir> --editor` args, calls `OS::create_instance(args)`, then `get_tree()->quit()`
- `editor/project_manager/project_list.cpp:1003-1011, 1627` — `projects.cfg` (plain INI) at `<editor_data>/projects.cfg`
- `main/main.cpp:217, 2130, 2210` — `project_manager` boot flag; falls back to PM when no `project.godot` found

**Adapt:** branch on `--project <path>` vs no-arg; `std.process.Child` for re-exec; `$XDG_DATA_HOME/zvoxrealms/projects.cfg` in TOML; ImGui PM window (not Godot's `Control`/`Node`).

### Module system — comptime dispatch · ✅

Each `modules/<name>/` is self-contained; codegen builds a dispatch table.

- `modules/modules_builders.py:15-51` — codegens `register_module_types.gen.cpp` with `#ifdef MODULE_X_ENABLED initialize_x_module(p_level)`
- `modules/gltf/{config.py, SCsub, register_types.cpp}` — concrete module example
- `main/main.cpp:787, 797, 830, 837` — `initialize_modules(level)` called at `CORE → SERVERS → SCENE → EDITOR`

**Adapt:** `modules/<name>/{config.zig, register_types.zig, src/, editor/}`; `build.zig` codegens `register_module_types.zig` as a comptime dispatch table (`inline for (enabled_modules) |m| m.initialize(level);`). Same four init levels.

### Server pattern — opaque handles + threaded queue · ✅

Scene tree holds only `RID` handles → `RenderingServer` pure-virtual interface → backend. Optional separate render thread via `CommandQueueMT`.

- `servers/rendering/rendering_server.h:64-117` — pure-virtual interface, dozens of `= 0` methods, all `RID`-typed
- `servers/rendering/rendering_server_default.h:45-47, 80` + `servers/rendering/server_wrap_mt_common.h` — command-queue threading wrapper
- `main/main.cpp:810, 3540` — `rendering_server = memnew(RenderingServerDefault(...))`

**Adapt:** Scene stores `Handle` (u64); `RenderServer`/`VoxelServer`/`PhysicsServer` are the only Vulkan/Jolt-aware code; port the command-queue threading. Use plain Zig structs + comptime backend selection (no vtables — one backend per build).

### Export — PCK three-location loader · ✅ (but diverge from "fat template")

PCK is a flat archive with magic header; runtime finds it in three locations.

- `core/io/file_access_pack.cpp:218-285` — `try_open_pack()` searches separate `.pck`, embedded section at `OS::get_embedded_pck_offset()`, or appended footer
- `core/io/file_access_pack.h:41, 168` — magic `0x43504447` ("GDPC"), `PackedSourcePCK`

**Adapt:** keep PCK format + three-location search. **Diverge:** Godot copies a fat pre-built template (`editor/export/editor_export_platform_pc.cpp:157-208` — literal `da->copy(template_path, p_path, ...)`); zVoxRealms instead runs `zig build-lib -dynamic` over the project's enabled modules → per-project `libzvox-runtime.{so,dll}`. Better, only possible because Zig ships LLVM bundled.

### Editor/runtime split — one preprocessor define · ✅

Single build option `TOOLS_ENABLED` differentiates editor vs runtime binaries from one source tree.

- `SConstruct:533, 548-549` — `env.editor_build` → `CPPDEFINES=["TOOLS_ENABLED"]`
- `core/config/engine.h:170-187` — `is_editor_hint()` is `_FORCE_INLINE_ bool { return false; }` in non-tools builds; compiler erases editor branches

**Adapt:** `build.zig` option `tools_enabled` → comptime constant via `@import("build_options")`. Hard rule: `core/`, `servers/`, `scene/`, `modules/*/` non-editor never import `editor/`. Enforce with a `build.zig` import-graph check.

---

## Unreal — MassEntity + Iris + Plugins (`$REFS/UnrealEngine/`)

Paths verified against UE 5.6-era clone (note `UE_ENABLE_INCLUDE_ORDER_DEPRECATED_IN_5_6` markers).

### MassEntity — archetype ECS · ✅ strongest Unreal borrow

Fragments (components), bitset archetype keys, processor dependency graph, parallel execution.

- `Engine/Source/Runtime/MassEntity/Public/MassEntityTypes.h:28-33` — fragment / tag / chunk-fragment / shared-fragment bitsets via `DECLARE_STRUCTTYPEBITSET_EXPORTED`
- `MassEntityManager.h:95, 247, 383` — archetype-handle-based `CreateEntity`, `AddFragmentToEntity`, batch APIs
- `MassProcessor.h:76, 201, 209` + `MassProcessorDependencySolver.h` — `ConfigureQueries`/`Execute` + processor dependency solver

**Adapt:** Port the *idea* (bitset-keyed archetype storage + parallel processor graphs) in a few hundred lines of Zig. UE's UObject/UScriptStruct welding does not port — comptime types replace it.

### Iris replication — handle-based delta · ✅ (not legacy `Net*`)

Modern UE replication. **Strong fit** for host-authoritative delta-only multiplayer; far better match than legacy property-replication system.

- `Engine/Source/Runtime/Net/Iris/Public/Iris/ReplicationSystem/ReplicationSystem.h:69` — `UReplicationSystem` with `FNetRefHandle`
- `ObjectReplicationBridge.h` + `ReplicationFragment.h` — fragment-based delta replication

**Adapt:** `NetRefHandle`-equivalent on top of zVoxRealms's `Handle`; per-fragment delta replication aligns with our save deltas (same on-wire format).

### Plugin descriptors · ✅ closest 1:1 match to `modules/`

JSON manifest with name, modules, dependencies, load phase.

- `Engine/Source/Runtime/Projects/Public/Interfaces/IPluginManager.h:110, 273, 382` — `IPlugin`, `IPluginManager`, `FindPlugin`, `FindPluginsUnderDirectory`, `GetPluginDependencies`
- `PluginDescriptor.h` + `ModuleDescriptor.h` + `ProjectDescriptor.h` — descriptor schemas

**Adapt:** module manifest in TOML (not JSON) with the same fields: `name`, `modules`, `dependencies`, `load_phase`. Skip UE's hot-reload mechanics for v1.

### Editor mode-tools + detail customization · ⚠ pattern only

- `Engine/Source/Editor/UnrealEd/Public/EditorModeManager.h:42, 90` — `FEditorModeTools.ActivateMode(FEditorModeID, bToggle)` — modal tool registry (place / paint / sculpt)
- `Engine/Source/Editor/PropertyEditor/Public/IDetailCustomization.h` + `DetailLayoutBuilder.h` — per-type detail panel customization (the actual "details panel" API; there is no `Editor/DetailsView/` directory)
- `Engine/Source/Editor/LevelEditor/Public/LevelEditor.h:70` — module-registered Slate tabs

**Adapt:** mode-tools registry maps directly to voxel brush / biome painter / scene-region tools. Detail-customization is the right mental model for ImGui per-type property panels. Don't port Slate.

### Chaos Physics — architecture only · ⚠

Jolt already committed. Borrow the *interface shape*, not the solver.

- `Engine/Source/Runtime/Experimental/Chaos/Public/Chaos/ChaosMarshallingManager.h` — game-thread ↔ physics-thread marshalling pattern (maps to our handle-based Server → Backend boundary)
- `Engine/Source/Runtime/Experimental/Chaos/Public/PBDRigidsSolver.h:83` — solver-as-module

### Nanite — conceptual only · ⚠

Voxel chunks aren't pre-baked triangle DAGs. Original doc had **three wrong file names** — corrected to:

- `Engine/Source/Runtime/Engine/Public/Rendering/NaniteStreamingManager.h:88-302` — streaming + hierarchical LOD
- `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteCullRaster.cpp` — HZB + GPU culling
- `Engine/Source/Runtime/Engine/Public/Rendering/NaniteResources.h` — cluster page format

**Read for ideas** (cluster page streaming, HZB occlusion, GPU LOD selection). Don't port — Nanite assumes mesh shaders (often absent on integrated graphics, our target hardware).

### Landscape — edit-layer composition only · ⚠

- `Engine/Source/Runtime/Landscape/Public/LandscapeEditLayerMergeRenderContext.h` — non-destructive edit-layer composition

**Adapt only:** the layered-edit composition idea for the voxel editor's undo/redo + sculpt/paint stack. Heightmap topology doesn't apply.

---

## What to skip — combined "don't bother" list

| Engine | Skip | Why |
| --- | --- | --- |
| Hazel | nothing critical to skip | Direct fit |
| Luanti | Lua API + old C++ scaffolding | We use native C ABI + TOML |
| Godot | `Object`/`Ref<T>`/`GDCLASS`/`ClassDB` reflection, fat-template export, PCK encryption + V2/V3/V4 versioning, `Control`/`Node`-as-GUI for PM, scons, `.compat.inc` shims | Designed for GDScript binding / runtime compat we don't need |
| Unreal | Nanite implementation, Slate UI, UObject reflection, legacy `Net*` replication, Actor/Component OOP for gameplay, Blueprints | Heavy C++ tax for features we either replace (ECS, ABI) or don't need (script binding, Blueprints) |

---

## Sources by file (verification index)

If you suspect a claim above is stale, `grep` the local clone:

```bash
# Hazel
grep -rn "IDComponent" $REFS/Hazel/src

# Luanti
ls $REFS/luanti-custom/src/{voxel.h,mapblock.h,voxelalgorithms.cpp}

# Godot
grep -n "OS::create_instance" $REFS/godot/editor/project_manager/project_manager.cpp
grep -n "initialize_modules" $REFS/godot/main/main.cpp
grep -n "PACK_HEADER_MAGIC" $REFS/godot/core/io/file_access_pack.h

# Unreal
ls $REFS/UnrealEngine/Engine/Source/Runtime/MassEntity/Public/
ls $REFS/UnrealEngine/Engine/Source/Runtime/Net/Iris/Public/Iris/ReplicationSystem/
ls $REFS/UnrealEngine/Engine/Source/Runtime/Projects/Public/Interfaces/
```
