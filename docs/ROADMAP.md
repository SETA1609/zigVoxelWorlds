# zVoxRealms Roadmap

> A roadmap, not a feature catalog. Per-phase scope lives in [`specs/`](specs/). High-level architecture lives in [`ARCHITECTURE.md`](ARCHITECTURE.md). Scope authority: [`vision.md`](vision.md), [`mission.md`](mission.md). Code-level references: [`engine-references.md`](engine-references.md). Open planning items: [`gaps.md`](gaps.md). **Missing systems / v1 gap analysis:** [`gaps.md`](gaps.md).

The roadmap is **vertical-slice-driven**: each phase ends with something runnable. The v1 target is a small Daggerfall-style co-op slice — one town, one dungeon, working skills/magic/crafting, 4-player co-op, mod loader live, packaged via the per-project export pipeline.

---

## Phase 0: Foundation (Closing)

Planning, layout, decisions — no engine code beyond hello-world.

- [x] Zig + C + C++ hybrid build system (`src/main.zig` + `src/c/` + `src/cpp/` build via `build.zig`)
- [x] Planning documents in `docs/` (vision, mission, ARCHITECTURE, tech-stack, project-structure, engine-vs-game, engine-references, external-libs-catalog, external-libs-survey, licensing, cpp-style, guard, gaps, ROADMAP, mvp, plus `specs/*.md`)
- [x] Data-schema docs landed (per [`gaps.md` § 3](gaps.md)) — `specs/data-schemas.md`, `specs/core-types.md`, `specs/c-abi.md`, `specs/threading.md`, `specs/testing.md`
- [x] Meta-package adapter sub-repos set up — [`zig-cpp-vulkan-stack-adapter`](https://github.com/SETA1609/zig-cpp-vulkan-stack-adapter) + [`zig-cpp-platform-stack-adapter`](https://github.com/SETA1609/zig-cpp-platform-stack-adapter) with LICENSE + README + build.zig.zon at scaffolding parity
- [x] CI baseline — `.github/workflows/build.yml` (lint + matrix build + submodule fetch + cache)
- [x] Branching strategy decided — trunk-based with build flags per [`CONTRIBUTING.md`](../CONTRIBUTING.md)
- [ ] On-disk project layout matches [`project-structure.md`](project-structure.md) — landing in Phase 1 sprint
- [ ] Build artifact renamed from `demo` to `zvoxrealms` in `build.zig` — landing in Phase 1 sprint

See [`mvp.md`](mvp.md) for the MVP definition (= Phase 1 + Phase 2). Active sprint plan: [`sprint.md`](sprint.md).

---

## Phase 1: Window + Vulkan Basics (Active)

Open a Vulkan window, render a debug primitive. Validates the platform layer + renderer scaffolding + meta-package adapter consumption.

Wires up:

- [`libs/zig-cpp-platform-stack-adapter`](https://github.com/SETA1609/zig-cpp-platform-stack-adapter) as `build.zig.zon` dep — engine imports `@import("platform")`; v0 backend is GLFW
- [`libs/zig-cpp-vulkan-stack-adapter`](https://github.com/SETA1609/zig-cpp-vulkan-stack-adapter) as `build.zig.zon` dep — engine imports `@import("vulkan_stack")`; re-exports vulkan-zig + wraps VMA/volk/shaderc
- `src/render/surface.zig` bridge helper per [`specs/platform.md` § Rule 2](specs/platform.md) — pairs `platform.get*Handle(window)` with `vk_stack.create*Surface(instance, ...)` at comptime per target OS

**Milestone:** colored rotating cube on screen via the GLFW v0 backend + the Vulkan-stack adapter. Action-mapped input (`platform.actionPressed(.menu_pause)`) quits the demo.

---

## Phase 2: Module System + Server Pattern

Architectural backbone. Module contract (`modules/<name>/{config.zig, register_types.zig, src/}` + four init levels), comptime dispatch codegen, `Handle` opaque ID, `RenderServer` scaffolding, `src/core/{profile, log_sink, metrics}.zig` no-op stubs.

Borrows: [Godot module system + server pattern](engine-references.md), [Unreal plugin descriptor](engine-references.md). Architecture: [`ARCHITECTURE.md` § Module System + § Server Pattern](ARCHITECTURE.md). Handle layout + core types: [`specs/core-types.md`](specs/core-types.md). Threading: [`specs/threading.md`](specs/threading.md). C ABI: [`specs/c-abi.md`](specs/c-abi.md).

**Milestone:** boot the engine, load `modules/hello/` through all four init levels, render the rotating cube via `RenderServer` calls — scene code holds only handles, never raw Vulkan objects.

---

## Phase 3: Voxel Core

`modules/voxel_core/` proves the module pattern on a real subsystem.

Spec: [`specs/voxel.md`](specs/voxel.md). Reference: [Luanti voxel core](engine-references.md).

**Milestone:** walk through a generated 1 km × 1 km world (X/Z plane), 250 m vertical (Y).

---

## Phase 4: Asset Pipeline

Source → baked-runtime transforms with GUID database, hot-reload.

Spec: [`tech-stack.md` § Asset Pipeline](tech-stack.md#asset-pipeline-godotunreal-style).

**Milestone:** drag a PNG into `assets/`, see it appear as a usable texture in-engine within seconds, with a stable GUID that survives renames.

---

## Phase 5: Jolt Physics Integration

Built as `modules/physics_jolt/` behind `PhysicsServer`, via the `zig-jolt-adapter` sub-repo. Also wires the real Tracy backend behind `profile.zig`.

Spec: [`specs/physics.md`](specs/physics.md). Adapter: [`external-libs-catalog.md` § 3](external-libs-catalog.md).

**Milestone:** player walks, falls, knocks things over; can profile a physics frame in standalone Tracy GUI.

---

## Phase 6: Streaming, LOD, Large Worlds

Chunk streaming with hybrid loading, 3+ LOD tiers, origin rebasing, background asset streaming.

Open decisions: chunk size, origin rebasing strategy ([`gaps.md` § 3](gaps.md)).

**Milestone:** 10 km × 10 km seamless world (X/Z plane), 2 km vertical (Y) — 4 km vertical ambitious. 8-chunk view distance, target FPS on low-end hardware.

---

## Phase 7: ECS Core

Archetype-based ECS with processor scheduler.

Spec: [`specs/ecs.md`](specs/ecs.md). Reference: [Hazel + Unreal MassEntity](engine-references.md).

**Milestone:** 10k entities updating at frame budget on low-end PC.

---

## Phase 7.5: Presentation Layer

Animation + VFX + audio + UI + events bus + scene lighting all ship together — they share a tick budget, a streaming model, and are driven by gameplay events. Slot before Phase 8 so gameplay modules can emit events that the presentation layer renders.

Specs already drafted, awaiting implementation:

- [`specs/animation.md`](specs/animation.md) — skeletal animation + state machines + IK
- [`specs/particles.md`](specs/particles.md) — GPU particle simulation + emitter authoring
- [`specs/audio.md`](specs/audio.md) — bus tree + 3D positional + reverb zones + music streaming
- [`specs/ui.md`](specs/ui.md) — in-game UI engine (anchor layout, widgets, controller nav, tweens, transitions)
- [`specs/dialog.md`](specs/dialog.md) — Morrowind/Daggerfall-style modal dialog + trade + services (world pauses), gamepad-first
- [`specs/events.md`](specs/events.md) — pub/sub messaging bus
- [`specs/lighting.md`](specs/lighting.md) — scene lighting + decals + weather + time-of-day
- [`specs/materials.md`](specs/materials.md) — PBR materials + shader pipeline cache
- [`specs/post-processing.md`](specs/post-processing.md) — ACES tonemap + FXAA + bloom + LUT + vignette

**Milestone:** Player walks through a torch-lit cave (lighting); torch flame flickers (particles + lighting); footsteps echo (3D positional audio with reverb zone); rain starts outside (weather + particles); approach an NPC, dialog opens (UI + dialog) and the world pauses (Morrowind-style modal); cast a fire spell, fireball particles bounce off voxels (collision), sparks light voxel surfaces; gameplay event fires (`spell.cast`), achievement listener catches it.

## Phase 8: Gameplay Modules (Data-Driven)

Skills, perks, magic, crafting, inventory, and AI — each its own module under `modules/`. All data-driven via TOML.

Specs: [`specs/gameplay.md`](specs/gameplay.md) + [`specs/ai.md`](specs/ai.md) (Behavior Trees + NavMesh via Recast/Detour + perception + AI LOD tiers). Quest / magic / crafting / inventory data models open per [`gaps.md` § 3 #17, #19, #20, #21](gaps.md) — to be spec'd before this phase starts.

**Milestone:** player casts a custom-made spell and crafts a sword whose quality reflects skill; a Daggerfall-style NPC with a daily schedule lives in a 200-NPC town with AI LOD tiers active.

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

## Phase 15: v1.0 — Voxel Daggerfall Slice

**v1.0 ships one target: the voxel Daggerfall-clone slice.** Not all four target games. The other three (voxel rogue-like / voxel Stardew / voxel Atelier) are v1.1–v1.3 derivatives — content-only reskins on top of the same engine + modkit. See [`vision.md`](vision.md) § Shipping strategy.

Integration milestone: one town, 3–5 dungeons, one main quest line, classless skills + spellmaking + crafting, 4-player co-op stable for a 1-hour session, mod loader live, modkit shipped for third-party mods, replay system for debugging + content creation. Shipped as an exported per-project bundle on a clean machine. Released commercially on Steam.

**Milestone:** v1.0 voxel Daggerfall slice playable end-to-end on a clean machine without Zig installed; modkit available for third-party modders.

---

## Future / Deferred

- Voxel GI (VXGI-style) once Phase 6 streaming is solid
- Android port (per-platform export preset)
- Visual scripting for designers
- WebGPU fallback
- Multiplayer debugging tools (network inspector, lag simulation)
- Embedded PCK in launcher binary (vs separate `.pck`)

---

**Current priority:** Phase 1 (Window + Vulkan Basics) — active sprint plan in [`sprint.md`](sprint.md). Phase 0 closes alongside the first sprint commits (binary rename + project layout).

See: [`vision.md`](vision.md), [`mission.md`](mission.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`tech-stack.md`](tech-stack.md), [`project-structure.md`](project-structure.md), [`engine-vs-game.md`](engine-vs-game.md), [`external-libs-catalog.md`](external-libs-catalog.md), [`external-libs-survey.md`](external-libs-survey.md), [`engine-references.md`](engine-references.md), [`gaps.md`](gaps.md), [`mvp.md`](mvp.md), [`sprint.md`](sprint.md), [`specs/`](specs/).
