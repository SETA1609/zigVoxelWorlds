# zVoxRealms — Vision & High-Level Architecture

> Authoritative scope lives in [`vision.md`](vision.md) + [`mission.md`](mission.md). Vision (long-term): [`vision.md`](vision.md). Mission (current operating statement): [`mission.md`](mission.md). This document describes how the engine is layered to deliver that scope.
> Concrete code references from private Hazel, luanti-custom, Godot, and UnrealEngine: [`engine-references.md`](engine-references.md). C/C++ libraries that need adapter wrappers: [`external-libs-catalog.md`](external-libs-catalog.md). Engine-vs-game split + library categorization: [`engine-vs-game.md`](engine-vs-game.md).

## Core Goal

Build a high-performance, **moddable**, **multiplayer-by-default** voxel engine in **Zig** (primary language) using **Vulkan** for rendering and **Jolt Physics** for simulation. **C/C++ is used only when it makes more sense than Zig** — typically for mature vendor libraries (Jolt, VMA, Steamworks) where rewriting in Zig would cost more than it returns. Everything else is Zig.

The engine is named **zVoxRealms**. The repo `zigVoxelWorlds` hosts its source.

## Target Games

zVoxRealms is built to support the following game styles on top of a shared engine:

- **Voxel Daggerfall-style open-world RPG** with Morrowind-style spellmaking
- **Voxel Atelier-style crafting/alchemy RPG**
- **Voxel Stardew Valley-style life/farming sim**
- **Dungeon crawlers & endless-tower rogue-likes**
- **Classless RPG systems** (Fallout/Daggerfall/Morrowind style)
- **Multiplayer co-op** (4 players default; optional 40–50 player servers)

Performance target: **50–60 FPS on low-end PCs** (i3 / Ryzen 3 + integrated graphics) in 4-player co-op, < 3.5 GB RAM, 8-chunk default view distance.

## Core Principles

- **Zig-first**: All game logic, engine core, and glue code in Zig
- **C/C++ only when it pays off**: Use C/C++ when a mature library (Jolt, VMA, Steamworks) would cost more to rewrite in Zig than it saves. Wrap behind a thin `extern "C"` boundary so the Zig side stays clean. Default answer is still Zig.
- **Data-driven**: TOML for hand-authored content (skills, perks, spells, recipes, scenes, UI); binary for runtime/save/asset data; JSON only where external standards or tool interop require it (glTF, import sidecars). See [`tech-stack.md` → Data Layer](tech-stack.md#data-layer-data-driven-design).
- **Data-oriented design**: Archetype ECS, cache-coherent layouts, SIMD where it pays off
- **Explicit control**: No hidden allocations or magic
- **Modding as a first-class feature**: Layered overrides (Core → Mods → Player), stable C ABI for native plugins
- **Multiplayer by default**: Authoritative client-server from day one; singleplayer is "host + one client"
- **Modular & iterative**: Build in clear milestones, ship a vertical slice early

## Distribution Model

zVoxRealms ships as **one binary** that doubles as Project Manager and editor — the same shape as the Godot binary (see [`engine-references.md` → Godot](engine-references.md)).

- Launched with no args → opens the **Project Manager** (list/create/open projects, recents, favorites)
- Launched with `--project <path>` → opens straight into the editor for that project
- Opening a project from inside the PM spawns a fresh child process via `std.process.Child` and the PM quits. Crash isolation between PM and editor for free.
- Recents/favorites stored in `$XDG_DATA_HOME/zvoxrealms/projects.cfg` (TOML)

**Export model — diverges from Godot deliberately.** Godot copies a fat pre-built template binary that contains every compiled-in module. zVoxRealms does **per-project tree-shaken dynamic-lib export**:

1. Editor reads the project's `project.toml` manifest (enabled modules + features)
2. `build.zig` resolves the transitive module set and runs `zig build-lib -dynamic` over just those modules
3. Output is a small launcher executable + `libzvox-runtime.so`/`.dll` + a PCK asset bundle
4. Shipped game contains no engine code the project doesn't use

This is only practical because Zig ships LLVM bundled. C++ engines can't do it, which is why Godot uses fat templates.

The PCK format borrows Godot's three-location magic-search (`file_access_pack.cpp:218-285`): the runtime finds assets whether they live in a separate `.pck` file, an embedded section of the executable, or appended at the end with a footer-encoded back-pointer.

## Module System

Engine subsystems are organised as **modules** — each a self-contained directory with a fixed contract. Borrowed from Godot's `modules/` pattern, simplified with Zig comptime.

Per-module layout:

```text
modules/<name>/
├── config.zig          # Manifest: name, can_build(opts), deps, init level
├── register_types.zig  # initialize(level), uninitialize(level)
├── src/                # Module sources
└── editor/             # Optional, gated by tools_enabled
```

**Four init levels**, called in order during boot:

1. **Core** — primitive types, allocators, file I/O registration
2. **Servers** — `RenderServer`, `VoxelServer`, `PhysicsServer`, `NetServer` come online
3. **Scene** — scene tree, ECS, gameplay types
4. **Editor** — only in tools builds; editor panels, importers, gizmos

`build.zig` walks enabled modules and codegens a comptime dispatch table (`register_module_types.zig`). Zig comptime replaces Godot's `#ifdef MODULE_X_ENABLED` salad with a clean `inline for (enabled_modules) |m| m.initialize(level);`.

A project's `project.toml` declares which modules it enables; the editor builds with all of them; the export filters to just the project's set.

## Server Pattern (Scene → Server → Backend)

Strict three-layer split, borrowed from Godot's `RenderingServer`:

1. **Scene layer** (`World`, `Chunk`, `Entity`, gameplay components) — stores only **opaque handles** (`Handle` = u64 ID), never raw Vulkan/Jolt pointers
2. **Server layer** (`RenderServer`, `VoxelServer`, `PhysicsServer`, `AudioServer`, `NetServer`) — the only code that touches the backend; exposes a stable handle-based API
3. **Backend layer** — Vulkan via VMA, Jolt via C ABI wrapper, miniaudio, etc.

This makes hot-reload, threading, and serialization tractable: scene data is just IDs, so it can be saved, sent over the network, or reloaded without touching GPU state. Borrows Godot's `CommandQueueMT` pattern for the optional render-thread split (main thread enqueues, render thread drains).

## High-Level Layers

1. **Platform Layer** — Window, input, Vulkan surface, file I/O, threading, process spawning
2. **Core** — Allocators, math, logging, handle table, TOML parser, asset DB
3. **Rendering Server + Backend** — Vulkan abstraction + voxel-specific pipeline (greedy meshing, compute, LOD)
4. **Voxel Server + Backend** — Chunk storage, meshing, streaming, LOD tiers, deterministic generation from seed + voxel atlas
5. **Physics Server + Backend** — Jolt integration, voxel-to-rigidbody conversion, character controllers, ragdolls, vehicles
6. **Audio Server + Backend** — miniaudio wrapper, OGG/Opus streaming
7. **Net Server** — Authoritative server, client prediction + reconciliation, interest management, LAN discovery
8. **ECS Core** — Archetype-based ECS, system scheduler, query cache
9. **Scene & Instancing** — Persistent world + instanced scenes (dungeons, towers, houses) wired via `orchestrator.toml`
10. **Gameplay Modules** — Skills, perks, magic (spellmaking), crafting/synthesis, inventory, NPC AI — each a module
11. **Modding** — Layered loader (Core → Mods → Player Overrides), TOML data mods (JSON sidecars only where needed), native plugin ABI, Steam Workshop client
12. **UI** — TOML layout + SCSS styling for in-game UI, ImGui for editor/dev tools
13. **Project Manager** — Project list, recents, favorites, project creation wizard (editor build only)
14. **Editor / Tools** — Voxel brush, biome painter, scene browser, entity spawner, skill/perk editor, recipe editor, live hot-reload (editor build only)
15. **Export Pipeline** — Reads `project.toml`, tree-shakes modules, links `libzvox-runtime.so`, packs PCK (editor build only)

## Cross-Cutting Concerns

- **Determinism** — World generation and simulation must be reproducible from seed for multiplayer and replays. The same seed always produces the same world.
- **Save model** — Saves have **two layers**, persisted differently:
  1. **Seed-deterministic baseline** (NOT saved) — static terrain, biome layout, dungeon generator output, initial NPC placements. Always regenerated from seed at load time.
  2. **Authored / runtime state** (saved fully) — split into two kinds:
     - **World deltas (binary, compact)** — voxel modifications (player builds, mined blocks, destroyed walls). Layered over the regenerated baseline at load.
     - **Game state (binary, structured)** — quest state (flags, completion, branch points), player(s) state (inventory, skills, perks, location, stats), faction reputation, NPC state (positions, relationships, dialog history), discovered locations, learned recipes/spells, time of day, weather state, mod-registered save sections.
  Loading: regenerate world from seed → replay voxel deltas over affected chunks → restore game state. This makes saves small (no static terrain duplicated), makes patches survivable for the deterministic side, and makes multiplayer sync tractable (servers ship deltas + game-state diffs, not chunks). All save binary formats must be **stable across engine patches** or shipped saves break — version every section.
- **Multiplayer authority** — Server is authoritative. Default 4-player co-op; optional 40–50 player dedicated servers. Clients predict + reconcile. The world generator is part of the runtime (in every shipped game) so clients can regenerate baseline chunks locally and only receive deltas over the wire.
- **World editability policy** — Voxel mutations are **not** automatically allowed. Each scene/region declares an **edit policy** in TOML with **three orthogonal axes + optional ownership**: `destroy` (can the player remove voxels?), `place` (can the player add voxels / drop items?), `persistence` (do edits survive save+reload / scene reload / a triggered reset — `persistent` / `session_only` / `transient`), and `ownership` (does the policy change based on game state — `purchasable` / `quest_gated` / `faction`). Policies are composable: project default → scene default → region override (innermost wins). `VoxelServer` evaluates per-axis before applying any delta. Per-region granularity, not per-chunk. See [`specs/scene.md` § Edit policy](specs/scene.md) for full schema + four worked examples (buyable city plots, story dungeons that accept dropped items, procedural dungeons that reset per run, Stardew farm + mines). This three-axis model also drives the meshify-or-not rendering decision (see [`specs/voxel.md` § Meshified static chunks](specs/voxel.md)).
- **Hot-reload** — TOML data, JSON import sidecars, gameplay scripts (Zig + C++ via `dlclose`/`dlopen`), and asset re-bakes reload without restarting the session
- **Asset pipeline** — Source files (PNG/glTF/WAV/TOML) bake into runtime formats (KTX2/binary/Opus) on import; runtime only loads baked output. See [`tech-stack.md` → Asset Pipeline](tech-stack.md#asset-pipeline-godotunreal-style)
- **Interest management** — Networking and entity update systems share a "what's near the player" budget
- **Hard budgets** — Per-chunk entity caps, view-distance caps, RAM ceilings enforced in code, not hoped for
- **Editor/runtime split** — One comptime build option `tools_enabled` differentiates editor and exported-game builds from the same source tree. Runtime layers (`core/`, `servers/`, `scene/`, `modules/*/` non-editor) never import `editor/`. Enforced by a `build.zig` import-graph check
- **Opaque handles** — Scene/gameplay code stores `Handle` (u64 ID) only; never raw Vulkan/Jolt/audio objects. Keeps the scene layer serializable, network-shippable, and hot-reloadable

## Scripting & Game Code

Game logic is written in **Zig (preferred) or C++** — both AOT-compiled. No interpreted scripting (no Lua, no GDScript, no Python). See [`engine-vs-game.md` § 5](engine-vs-game.md#5-scripting) for the full design.

- `<project>/scripts/` is compiled per project to `<project>/.import/scripts/libgame.{so,dll}`
- Compilation uses the bundled Zig toolchain (`zig build-lib -dynamic` for Zig, `zig c++ -shared` for C++) — no external toolchain on user's machine
- Scripts use the **same stable `extern "C"` ABI as native mods** — game code IS a mod, structurally
- Hot-reload via `dlclose` + recompile + `dlopen`
- ABI versioned; engine refactors must not break it or shipped games break

## In-Engine Editor Panels

The editor has two specialized panels with distinct backends. Both are engine-only — not shipped with games.

**Code editor (`src/editor/code_editor/`)** — **Neovim**, bundled with the engine. GUI grid embed via msgpack-RPC: spawns `bundled/nvim/bin/nvim --embed` via `std.process.Child`, renders Neovim's `redraw` grid notifications directly (no terminal in the path). Reference: [neovide](https://github.com/neovide/neovide), [goneovim](https://github.com/akiyosi/goneovim). User's `~/.config/nvim/` is picked up automatically; engine ships a default config under `bundled/nvim/share/nvim/sysinit.vim` for new users. Last-resort fallback: ImGui text edit if the bundled binary is missing or subprocess spawning is blocked. Bundling adds ~30–50 MB to the engine release archive per platform (Apache 2.0, attribution required).

**Runtime debug output (`src/editor/playtest_log/`)** — [libghostty](https://github.com/ghostty-org/ghostty) as the text-stream rendering surface; engine-side multiplexer on top handles structured logs, level filtering, channel tabs (Render / Physics / Network / ECS / Modding), pause/clear/save-to-file, and click-to-jump (`file:line` → routes to the Neovim panel). Crash dumps pipe to both the panel and `<project>/.import/crash_<timestamp>.log`. Wrapped behind `src/core/log_sink.zig` so the backend can swap (libghostty's embedding API is post-1.0; some refactor risk over the next year).

## Key Technical Challenges

- Efficient sparse voxel storage (chunked grids; SVO only if profiling demands it)
- Fast meshing (greedy + compute shader approaches)
- Vulkan synchronization with compute-based meshing
- Bidirectional coupling between physics and voxel world
- Memory management at scale (gigabytes of voxel data on a low-end PC)
- Deterministic multiplayer with mod-introduced systems
- Stable C ABI for native mods that survives engine refactors

## What zVoxRealms is NOT

- Not a photorealistic engine — stylized voxel only
- Not an MMO platform — co-op scale (40–50 players max)
- Not console-first — Windows/Linux desktop first, Android later, no console parity
- Not a general-purpose engine — voxel and these genres only

Next: Read [`ROADMAP.md`](ROADMAP.md), [`tech-stack.md`](tech-stack.md), [`project-structure.md`](project-structure.md), [`external-libs-catalog.md`](external-libs-catalog.md), and [`engine-references.md`](engine-references.md).
