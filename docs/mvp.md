# zVoxRealms MVP

> The smallest checkpoint that proves the architecture works. Scope authority: [`vision.md`](vision.md), [`mission.md`](mission.md). Phases: [`ROADMAP.md`](ROADMAP.md). Active sprint: [`sprint.md`](sprint.md).

**MVP = Phase 1 + Phase 2 delivered.**

The MVP is **not** the v1.0 game release — that's Phase 15. Conflating them creates pressure to declare "done" prematurely. The MVP validates the *architecture*; v1.0 ships the **voxel Daggerfall slice** (one target game, not all four — see [`vision.md`](vision.md) § Shipping strategy). The other three voxel games (rogue-like / Stardew / Atelier) follow as v1.1–v1.3 content-only derivatives.

## Definition of done

All criteria must pass:

### Build + tooling

- `zig build run` boots without errors on Linux and Windows (the v1.0 targets per [`mission.md`](mission.md))
- CI green on `main` — lint + matrix build + submodule fetch + cache per `.github/workflows/build.yml`
- Build artifact renamed from `demo` → `zvoxrealms`
- `build.zig` import-graph check enforces `runtime never imports editor`

### Adapter wiring (the foundation of everything else)

- [`libs/zig-cpp-platform-stack-adapter`](https://github.com/SETA1609/zig-cpp-platform-stack-adapter) consumed via `build.zig.zon`; engine code imports `@import("platform")`
- [`libs/zig-cpp-vulkan-stack-adapter`](https://github.com/SETA1609/zig-cpp-vulkan-stack-adapter) consumed via `build.zig.zon`; engine code imports `@import("vulkan_stack")`
- Both sub-repos at v0.1.0 or later (Phase-0 hello-world → first real wrapping)
- The two adapters are **fully decoupled** per [`specs/platform.md` § Rule 2](specs/platform.md) — no shared type, no cross-import. Engine bridges via `src/render/surface.zig`

### Window + render (Phase 1)

- Engine binary opens a Vulkan window via the Platform-stack adapter (GLFW v0 backend)
- Engine creates a Vulkan instance + surface via `render.createSurface(instance, window)` helper that pairs `platform.get*Handle(window)` with `vk_stack.create*Surface(instance, ...)`
- Rotating cube renders to that window with no direct Vulkan calls outside `src/backends/vulkan/` and the Vulkan-stack adapter

### Module + Server pattern (Phase 2)

- Module contract live: `modules/<name>/{config.zig, register_types.zig, src/}` with `build.zig` codegen of a comptime dispatch table
- `modules/hello/` registers across all four init levels (`.core`, `.servers`, `.scene`, `.editor`)
- `RenderServer` exists with handle-based API; scene code holds only `Handle` (u64), never raw Vulkan objects
- Rotating cube renders via `RenderServer.drawMesh(handle)` — not via direct Vulkan in scene code
- `src/core/profile.zig` + `log_sink.zig` + `metrics.zig` exist as no-op stubs (real backends wired later — Phase 5 / 10 / 12)

### Runtime

- < 200 MB RAM at runtime (way under the [`mission.md`](mission.md) < 3.5 GB ceiling for v1.0)
- Runs on a 10-year-old i3 / Ryzen 3 iGPU at 60 FPS for the hello-world cube
- Action-mapped input wired (`platform.actionPressed(.menu_pause)` quits the demo)

## What's explicitly excluded from MVP

- Multiplayer ([Phase 10](ROADMAP.md))
- Physics ([Phase 5](ROADMAP.md))
- Streaming + LOD ([Phase 6](ROADMAP.md))
- Full asset pipeline ([Phase 4](ROADMAP.md))
- ECS ([Phase 7](ROADMAP.md))
- Presentation layer — animation, audio, lighting, UI, dialog ([Phase 7.5](ROADMAP.md))
- Gameplay systems — skills/perks/magic/crafting ([Phase 8](ROADMAP.md))
- Scene instancing ([Phase 9](ROADMAP.md))
- Project Manager UX ([Phase 11](ROADMAP.md))
- Real editor panels — voxel brush, biome painter, etc. ([Phase 12](ROADMAP.md))
- Export pipeline ([Phase 13](ROADMAP.md))
- Modding loader ([Phase 14](ROADMAP.md))

These are subsequent phases, each producing its own runnable milestone.

## Why this scope

A solo developer + ambitious engine = real risk of architecture-on-paper never validating in code. The MVP answers one question:

> Does the **module system + server pattern + opaque-handle discipline + meta-package adapter pattern** actually feel right in Zig, on this hardware, with our build system?

If yes → continue. If no → redesign in week 3, not month 30.

## What this validates

- Zig + C + C++ hybrid build works end-to-end
- `build.zig` codegen of a comptime dispatch table is feasible
- Module init-level ordering doesn't have hidden circular dependencies
- Handle indirection through servers doesn't add unbearable overhead
- Import-graph enforcement catches real mistakes
- **Meta-package adapter sub-repos as `build.zig.zon` deps work cleanly** — Platform-stack + Vulkan-stack consumed as two standalone packages, engine bridges them with `src/render/surface.zig`
- **The GLFW → native swap path is real** — the v0 backend (GLFW) inside Platform-stack proves the API surface holds; the v1.x native backend swap is then a sub-repo version bump
- ImGui via the adapter sub-repo route is workable
- Action-mapped input from day one (no raw key-code leaks) holds up against actual gameplay code

## Open dependencies before MVP starts

All resolved as of late Phase 0:

- ✅ `project.toml` / `mod.toml` / `assetdb.toml` schemas — `specs/data-schemas.md`
- ✅ `Handle` layout + coordinate system + voxel data layout + chunk size — `specs/core-types.md`
- ✅ Threading model — `specs/threading.md`
- ✅ C ABI surface — `specs/c-abi.md`
- ✅ Test strategy — `specs/testing.md`
- ✅ Camera + input — `specs/camera.md` + `specs/platform.md` § Action-mapped input
- ✅ Platform adapter contract — `specs/platform.md`
- ✅ Vulkan-stack catalog row — `external-libs-catalog.md` § 3
- ✅ CI baseline — `.github/workflows/build.yml`
- ✅ Branching strategy — `CONTRIBUTING.md` § Branching strategy

**MVP is unblocked.** Active task list lives in [`sprint.md`](sprint.md).

## Next after MVP

Phase 3 (Voxel Core) takes the proven module/server/handle pattern and applies it to a real subsystem. If the MVP feels clumsy, redesign before Phase 3.
