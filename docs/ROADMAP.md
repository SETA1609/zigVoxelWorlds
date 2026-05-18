# zVoxRealms Roadmap

> A roadmap, not a feature catalog. Per-phase scope lives in [`specs/`](specs/). High-level architecture lives in [`ARCHITECTURE.md`](ARCHITECTURE.md). Scope authority: [`vision.md`](vision.md), [`mission.md`](mission.md). Code-level references: [`engine-references.md`](engine-references.md). Open planning items: [`planning-gaps.md`](planning-gaps.md).

The roadmap is **vertical-slice-driven**: each phase ends with something runnable. The v1 target is a small Daggerfall-style co-op slice — one town, one dungeon, working skills/magic/crafting, 4-player co-op, mod loader live, packaged via the per-project export pipeline.

---

## Phase 0: Foundation (In Progress)

Planning, layout, decisions — no engine code beyond hello-world.

- [x] Zig + C + C++ hybrid build system (`src/main.zig` + `src/c/` + `src/cpp/` build via `build.zig`)
- [x] Planning documents in `docs/` (vision, mission, ARCHITECTURE, tech-stack, project-structure, engine-vs-game, engine-references, external-libs, licensing, cpp-style, guard, planning-gaps, ROADMAP, mvp, plus `specs/*.md`)
- [ ] On-disk project layout matches [`project-structure.md`](project-structure.md)
- [ ] Build artifact renamed from `demo` to `zvoxrealms` in `build.zig`
- [ ] Data-schema docs landed (per [`planning-gaps.md` §1](planning-gaps.md))

See [`mvp.md`](mvp.md) for the MVP definition (= Phase 1 + Phase 2).

---

## Phase 1: Window + Vulkan Basics (Next)

Open a Vulkan window, render a debug primitive. Validates the platform layer + renderer scaffolding.

**Milestone:** colored rotating cube on screen.

---

## Phase 2: Module System + Server Pattern

Architectural backbone. Module contract (`modules/<name>/{config.zig, register_types.zig, src/}` + four init levels), comptime dispatch codegen, `Handle` opaque ID, `RenderServer` scaffolding, `src/core/{profile, log_sink, metrics}.zig` no-op stubs.

Borrows: [Godot module system + server pattern](engine-references.md), [Unreal plugin descriptor](engine-references.md). Architecture: [`ARCHITECTURE.md` § Module System + § Server Pattern](ARCHITECTURE.md).

**Milestone:** boot the engine, load `modules/hello/` through all four init levels, render the rotating cube via `RenderServer` calls (scene code holds only handles).

---

## Phase 3: Voxel Core

`modules/voxel_core/` proves the module pattern on a real subsystem.

Spec: [`specs/voxel.md`](specs/voxel.md). Reference: [Luanti voxel core](engine-references.md).

**Milestone:** walk through a generated 1 km² world.

---

## Phase 4: Asset Pipeline

Source → baked-runtime transforms with GUID database, hot-reload.

Spec: [`tech-stack.md` § Asset Pipeline](tech-stack.md#asset-pipeline-godotunreal-style).

**Milestone:** drag a PNG into `assets/`, see it appear as a usable texture in-engine within seconds, with a stable GUID that survives renames.

---

## Phase 5: Jolt Physics Integration

Built as `modules/physics_jolt/` behind `PhysicsServer`, via the `zig-jolt-adapter` sub-repo. Also wires the real Tracy backend behind `profile.zig`.

Spec: [`specs/physics.md`](specs/physics.md). Adapter: [`external-libs.md` § 3](external-libs.md).

**Milestone:** player walks, falls, knocks things over; can profile a physics frame in standalone Tracy GUI.

---

## Phase 6: Streaming, LOD, Large Worlds

Chunk streaming with hybrid loading, 3+ LOD tiers, origin rebasing, background asset streaming.

Open decisions: chunk size, origin rebasing strategy ([`planning-gaps.md` #10, #11](planning-gaps.md)).

**Milestone:** 10 km² seamless world, 8-chunk view distance, target FPS on low-end hardware.

---

## Phase 7: ECS Core

Archetype-based ECS with processor scheduler.

Spec: [`specs/ecs.md`](specs/ecs.md). Reference: [Hazel + Unreal MassEntity](engine-references.md).

**Milestone:** 10k entities updating at frame budget on low-end PC.

---

## Phase 8: Gameplay Modules (Data-Driven)

Skills, perks, magic, crafting, inventory — each its own module under `modules/`. All data-driven via TOML.

Spec: [`specs/gameplay.md`](specs/gameplay.md).

**Milestone:** player casts a custom-made spell and crafts a sword whose quality reflects skill.

---

## Phase 9: Scene & Instancing

Persistent main world + instanced scenes wired via `orchestrator.toml`.

Spec: [`specs/scene.md`](specs/scene.md).

**Milestone:** enter a dungeon from the overworld, exit, state persists.

---

## Phase 10: Multiplayer

Built as `modules/multiplayer/` behind `NetServer`. Also wires the OpenTelemetry backend behind `log_sink.zig` + `metrics.zig` for dedicated-server observability.

Spec: [`specs/multiplayer.md`](specs/multiplayer.md). Reference: [Luanti multiplayer + Unreal Iris](engine-references.md).

**Milestone:** 4 players co-op the dungeon-clear loop with acceptable latency; a dedicated-server instance ships tick-time p50/p95/p99 + player count to a local Grafana dashboard.

---

## Phase 11: Project Manager

Engine-as-app entry screen. Same binary opens as PM or editor depending on CLI args; re-exec on project open.

Spec: [`specs/project-manager.md`](specs/project-manager.md). Reference: [Godot Project Manager](engine-references.md).

**Milestone:** launch engine binary with no args, see PM; create a new project, open it, exit, reopen from recents.

---

## Phase 12: Editor & Tooling

Editor panels (voxel brush, biome painter, recipe editor, asset browser, module toggles), bundled Neovim code-editor panel, libghostty playtest-log panel.

Spec: [`specs/editor.md`](specs/editor.md). Editor sub-panels: [`engine-vs-game.md` § 6 + § 7](engine-vs-game.md).

**Milestone:** build a small dungeon end-to-end in the editor without touching code; edit a script in Neovim panel; click a stack-trace `file:line` in the debug log → Neovim jumps to source.

---

## Phase 13: Export Pipeline (Per-Project Dynamic-Lib)

Where zVoxRealms diverges from Godot. Per-project tree-shaken `libzvox-runtime.{so,dll}` via `zig build-lib -dynamic` over the modules a project's `project.toml` enables; PCK three-location asset bundle; thin launcher stub.

Spec: [`ARCHITECTURE.md` § Distribution Model](ARCHITECTURE.md#distribution-model) + [`engine-vs-game.md`](engine-vs-game.md).

**Milestone:** export a project; run the standalone output on a clean machine with no Zig toolchain; `nm libzvox-runtime.so` shows only the enabled modules' symbols.

---

## Phase 14: Modding

Layered loader (Core → Mods → Player Overrides), native plugin ABI, Steam Workshop integration (with `-Dsteam=true`), dedicated server GUI.

Spec: [`engine-vs-game.md`](engine-vs-game.md) (modding ABI is the same stable C ABI as scripts). Reference: [Luanti layered mods](engine-references.md).

**Milestone:** install a third-party mod that adds a new magic school without recompiling the exported game.

---

## Phase 15: v1 Vertical Slice

Integration milestone: one town, one dungeon, the full skill/magic/crafting loop, 4-player co-op stable for a 1-hour session, mod loader live, replay system for debugging + content creation. Shipped as an exported per-project bundle on a clean machine.

**Milestone:** v1.0 vertical slice playable end-to-end.

---

## Future / Deferred

- Voxel GI (VXGI-style) once Phase 6 streaming is solid
- Android port (per-platform export preset)
- Visual scripting for designers
- WebGPU fallback
- Multiplayer debugging tools (network inspector, lag simulation)
- Embedded PCK in launcher binary (vs separate `.pck`)

---

**Current priority:** finish Phase 0 planning (data-schema docs in [`planning-gaps.md` §1](planning-gaps.md)), then begin Phase 1.

See: [`vision.md`](vision.md), [`mission.md`](mission.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`tech-stack.md`](tech-stack.md), [`project-structure.md`](project-structure.md), [`engine-vs-game.md`](engine-vs-game.md), [`external-libs.md`](external-libs.md), [`engine-references.md`](engine-references.md), [`planning-gaps.md`](planning-gaps.md), [`mvp.md`](mvp.md), [`specs/`](specs/).
