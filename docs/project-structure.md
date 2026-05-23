# Project Structure

> Scope: [`vision.md`](vision.md) + [`mission.md`](mission.md). Layers: [`ARCHITECTURE.md`](ARCHITECTURE.md). Stack: [`tech-stack.md`](tech-stack.md). Adapters: [`external-libs-catalog.md`](external-libs-catalog.md). Borrowed patterns: [`engine-references.md`](engine-references.md).
>
> ⚠ **Current vs target.** The layout below is the **target** for Phase 0 completion. On disk today, the repo contains only the stub layout (`src/main.zig` + `src/c/` + `src/cpp/` hello-world) inherited from the build template. Closing Phase 0 means creating `src/core/`, `src/servers/`, `src/backends/`, `src/scene/`, `src/platform/`, `modules/`, and the editor / project-manager / importers / export subtrees as documented below — even if most start empty.

The layout reflects three architectural decisions from [`ARCHITECTURE.md`](ARCHITECTURE.md):

1. **Engine-as-app** — one binary opens as Project Manager or editor depending on CLI args
2. **Modular subsystems** — each engine system lives in `modules/<name>/` with a fixed contract (`config.zig`, `register_types.zig`, `src/`, optional `editor/`)
3. **Editor/runtime split** — `editor/` and `modules/<name>/editor/` are gated by a comptime `tools_enabled` build option; runtime layers never import them

## Top-level layout

```text
.
├── build.zig                    # Root build — wires core + modules + editor + adapters + game
├── build.zig.zon                # Dependency manifest
│
├── docs/                        # All planning + reference docs
│   ├── vision.md                # Long-horizon vision
│   ├── mission.md               # Current operating statement
│   ├── ARCHITECTURE.md
│   ├── ROADMAP.md               # Lean — phase + milestone + refs only
│   ├── tech-stack.md
│   ├── project-structure.md     # This file
│   ├── engine-vs-game.md
│   ├── external-libs-catalog.md  # Committed deps + integration tier
│   ├── external-libs-survey.md   # Candidate landscape per gap
│   ├── engine-references.md
│   ├── licensing.md
│   ├── cpp-style.md
│   ├── guard.md
│   ├── gaps.md
│   ├── mvp.md
│   └── specs/                   # Per-subsystem detailed specs
│       ├── voxel.md
│       ├── physics.md
│       ├── ecs.md
│       ├── gameplay.md
│       ├── scene.md
│       ├── multiplayer.md
│       ├── project-manager.md
│       └── editor.md
│
├── src/
│   ├── main.zig                 # Entry. Branches: no-args → Project Manager; --project <p> → editor
│   ├── core/                    # Init level 0: allocators, math, logging, handle table, TOML parser, asset DB
│   ├── servers/                 # Init level 1: stable handle-based interfaces (the Godot "server" pattern)
│   │   ├── render_server.zig
│   │   ├── voxel_server.zig
│   │   ├── physics_server.zig
│   │   ├── audio_server.zig
│   │   └── net_server.zig
│   ├── backends/                # Concrete implementations behind the servers (the Vulkan/Jolt-touching layer)
│   │   ├── vulkan/
│   │   ├── jolt/
│   │   ├── miniaudio/
│   │   └── enet/                # OR gns/
│   ├── scene/                   # Init level 2: scene tree, ECS, handle-only world model
│   │   ├── world.zig
│   │   ├── chunk.zig
│   │   ├── entity.zig
│   │   ├── ecs/                 # Archetype ECS
│   │   └── orchestrator.zig     # orchestrator.toml loader, scene instancing
│   ├── platform/                # Window, input, file I/O, threading, process spawning (std.process.Child)
│   ├── ui/                      # Runtime UI: TOML layout + SCSS styling
│   ├── project_manager/         # editor-only; opens at startup when no --project given
│   │   ├── pm_window.zig
│   │   ├── project_list.zig     # ports Godot ProjectList::Item
│   │   └── project_create.zig
│   ├── editor/                  # editor-only; ImGui panels, gizmos, viewport, hot-reload host
│   │   ├── editor_layer.zig
│   │   ├── viewport.zig
│   │   ├── panels/
│   │   │   ├── voxel_brush.zig
│   │   │   ├── biome_painter.zig
│   │   │   ├── scene_browser.zig
│   │   │   ├── entity_spawner.zig
│   │   │   ├── skill_editor.zig
│   │   │   ├── recipe_editor.zig
│   │   │   └── asset_browser.zig
│   │   ├── code_editor/         # Embedded Neovim host (msgpack-RPC); ImGui fallback if nvim missing
│   │   ├── script_builder/      # Watches <project>/scripts/, invokes zig build-lib / zig c++ -shared
│   │   └── hot_reload/
│   ├── export/                  # editor-only; build a per-project libzvox-runtime.{so,dll} + PCK
│   │   ├── tree_shake.zig       # resolves enabled modules from project.toml
│   │   ├── link.zig             # invokes zig build-lib -dynamic
│   │   ├── pack.zig             # PCK writer (Godot-style magic + dir + blobs)
│   │   └── launcher_template/   # tiny stub that loads libzvox-runtime + PCK
│   ├── importers/               # editor-only; asset pipeline (PNG→KTX2, glTF→mesh, .vox→voxel, etc.)
│   │   ├── texture.zig
│   │   ├── mesh.zig
│   │   ├── voxel_model.zig
│   │   ├── audio.zig
│   │   ├── shader.zig
│   │   └── data.zig             # TOML game data → validated cached binary
│   ├── modding/                 # Layered loader, native plugin ABI, mod TOML reader
│   └── tools/                   # Standalone CLI: --dump-save, --validate-mod, --bake-assets
│
├── modules/                     # Init level 2/3: pluggable subsystems (Godot modules/ pattern)
│   ├── voxel_core/              # Chunk + meshing + streaming + lighting
│   │   ├── config.zig
│   │   ├── register_types.zig
│   │   └── src/
│   ├── skills/
│   ├── perks/
│   ├── magic/
│   │   ├── config.zig
│   │   ├── register_types.zig
│   │   ├── src/
│   │   └── editor/              # tools-only sub-tree, gated by build option
│   ├── crafting/
│   ├── inventory/
│   ├── ai/
│   ├── multiplayer/             # Network protocols + sync — depends on net_server + scene
│   ├── farming/                 # Stardew-style — optional
│   ├── rogue_tower/             # Endless tower — optional
│   └── steam/                   # Steamworks + Workshop, only built with -Dsteam=true
│
├── adapters/                    # Each C/C++ lib gets its own adapter sub-project (see external-libs-catalog.md)
│   ├── vulkan/                  # Volk + VMA wrapper
│   ├── glfw/
│   ├── imgui/
│   ├── jolt/
│   ├── toml/
│   ├── cgltf/
│   ├── ktx2/
│   ├── glslang/
│   ├── enet/                    # OR gamenetworkingsockets/
│   ├── steamworks/              # Conditional
│   ├── miniaudio/
│   ├── tracy/
│   ├── zstd/
│   └── flatbuffers/             # OR capnproto/
│
├── core_pack/                   # Base game content (ships with engine, distributed as a project template)
│   ├── project.toml
│   ├── assets/
│   └── data/
│
├── tests/                       # Integration tests (unit tests live next to source as `test "..." {}`)
│
├── zig-out/                     # Build artifacts (gitignored)
├── .gitignore
└── README.md
```

## Project layout (what a user's project looks like)

Projects are not stored inside the engine repo — they live wherever the user wants. The PM tracks paths in `$XDG_DATA_HOME/zvoxrealms/projects.cfg`. A project is identified by the presence of `project.toml`.

```text
my-rpg/
├── project.toml                 # Manifest. Declares enabled modules, game name, version, target platforms
├── assets/                      # Source assets
│   ├── .assetdb.toml            # GUID ↔ source path + content hash map
│   ├── textures/
│   ├── models/
│   ├── audio/
│   ├── shaders/
│   └── data/                    # Game data TOML (skills, perks, spells, recipes, scenes)
├── .import/                     # Baked output, gitignored — runtime loads from here
│   └── <guid>.<ext>
├── mods/                        # Project-local mods (auto-discovered)
│   └── <mod_name>/
│       ├── mod.toml
│       ├── data/
│       ├── assets/
│       └── plugin/              # Optional native .so/.dll against stable C ABI
└── export/                      # Output of "Export project" — gitignored
    ├── linux-x86_64/
    │   ├── my-rpg               # Launcher
    │   ├── libzvox-runtime.so   # Tree-shaken: only enabled modules linked in
    │   └── my-rpg.pck
    └── windows-x86_64/
```

## Module contract

Every directory under `modules/<name>/` is a self-contained subsystem. The contract is fixed so `build.zig` can codegen the dispatch table.

```text
modules/<name>/
├── config.zig          # pub const name = "..."; pub fn canBuild(opts: BuildOptions) bool; pub const deps = .{...};
├── register_types.zig  # pub fn initialize(level: InitLevel) void; pub fn uninitialize(level: InitLevel) void;
├── src/                # Module sources, plain Zig
└── editor/             # OPTIONAL — tools-only code (panels, gizmos, importers for this module's data)
```

`build.zig` walks `modules/`, evaluates `canBuild()` against project options, and emits `src/registered_modules.gen.zig` containing a comptime array of module pointers. `main.zig` calls `initializeAll(.core)` then `.servers`, `.scene`, and (in editor builds only) `.editor`.

## Conventions

- **Zig first.** New code is Zig unless a clear case for C/C++ exists (see [`tech-stack.md` → Language Strategy](tech-stack.md#language-strategy)).
- **Adapters are isolated.** Each C/C++ lib lives in `adapters/<name>/` with its own `build.zig`, exposing only an `extern "C"` API. Zig never sees raw C++.
- **No hidden global state.** Pass context (allocators, world handles, etc.) explicitly.
- **Errors via Zig error sets.** No discarded errors at boundaries.
- **Tests next to source.** `foo.zig` keeps its tests in the same file with `test "name" { ... }`. Integration tests live in `tests/`.
- **Hot paths don't parse text.** TOML is parsed at import time and cached as binary. Runtime loads the cached binary.
- **Layering**: `core/` → `servers/` + `backends/` → `scene/` → `modules/<name>/` → `editor/` + `project_manager/` + `export/` + `importers/`. Lower layers never depend on higher ones. `build.zig` enforces this with an import-graph check.
- **Handles only at the scene boundary.** Scene/gameplay code stores `Handle` (u64). Vulkan/Jolt/audio objects live only inside their `servers/` + `backends/` pair.
- **GUIDs over paths in assets.** Inside `<project>/assets/.assetdb.toml`, every source asset has a stable GUID. All TOML references (scenes, recipes, mod data) point at GUIDs, never paths.

## What lives where (quick guide)

| You're building... | It goes in... |
| --- | --- |
| New Vulkan rendering feature | `src/backends/vulkan/` + extend `src/servers/render_server.zig` |
| New engine subsystem (e.g. weather, traffic) | `modules/<name>/` |
| New skill or perk | `<project>/assets/data/skills/foo.toml` (data-driven) |
| New magic effect type | `modules/magic/src/` (logic) + `<project>/assets/data/spells/` (definitions) |
| New importer for a file type | `src/importers/<type>.zig` |
| New editor panel | `src/editor/panels/<panel>.zig` |
| New module-specific editor panel | `modules/<name>/editor/<panel>.zig` |
| Wrapping a new C/C++ lib | `adapters/<name>/` + entry in [`external-libs-catalog.md`](external-libs-catalog.md) |
| New native mod-side system | `<project>/mods/<mod>/plugin/` against the stable C ABI |
| Project Manager UI | `src/project_manager/` |
| Export pipeline logic | `src/export/` |

Next step: finish Phase 0 docs, then begin Phase 1 (Vulkan foundation) per [`ROADMAP.md`](ROADMAP.md).
