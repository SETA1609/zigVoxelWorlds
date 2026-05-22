# zVoxRealms MVP

> The smallest checkpoint that proves the architecture works. Scope authority: [`vision.md`](vision.md), [`mission.md`](mission.md). Phases: [`ROADMAP.md`](ROADMAP.md).

**MVP = Phase 1 + Phase 2 delivered.**

The MVP is **not** the v1.0 game release — that's Phase 15. Conflating them creates pressure to declare "done" prematurely. The MVP validates the *architecture*; v1.0 ships the **voxel Daggerfall slice** (one target game, not all four — see [`vision.md`](vision.md) § Shipping strategy). The other three voxel games (rogue-like / Stardew / Atelier) follow as v1.1–v1.3 content-only derivatives.

## Definition of done

All criteria must pass:

- `zig build run` boots without errors
- Engine binary opens a Vulkan window
- Module system contract live: `modules/<name>/{config.zig, register_types.zig, src/}` with `build.zig` codegen of a comptime dispatch table
- `modules/hello/` registers across all four init levels (`.core`, `.servers`, `.scene`, `.editor`)
- `RenderServer` exists with handle-based API; scene code holds only `Handle` (u64), never raw Vulkan objects
- Rotating cube renders via `RenderServer.drawMesh(handle)` — no direct Vulkan calls from scene code
- `src/core/profile.zig` + `log_sink.zig` + `metrics.zig` exist as no-op stubs (real backends wired later — Phase 5 / 12 / 10)
- `build.zig` import-graph check enforces `runtime never imports editor`
- Build artifact renamed from `demo` → `zvoxrealms`
- < 200 MB RAM at runtime

## What's explicitly excluded from MVP

- Multiplayer ([Phase 10](ROADMAP.md))
- Physics ([Phase 5](ROADMAP.md))
- Streaming + LOD ([Phase 6](ROADMAP.md))
- Full asset pipeline ([Phase 4](ROADMAP.md))
- ECS ([Phase 7](ROADMAP.md))
- Gameplay systems — skills/perks/magic/crafting ([Phase 8](ROADMAP.md))
- Scene instancing ([Phase 9](ROADMAP.md))
- Project Manager UX ([Phase 11](ROADMAP.md))
- Real editor panels — voxel brush, biome painter, etc. ([Phase 12](ROADMAP.md))
- Export pipeline ([Phase 13](ROADMAP.md))
- Modding loader ([Phase 14](ROADMAP.md))

These are subsequent phases, each producing its own runnable milestone.

## Why this scope

A solo developer + ambitious engine = real risk of architecture-on-paper never validating in code. The MVP answers one question:

> Does the **module system + server pattern + opaque-handle discipline** actually feel right in Zig, on this hardware, with our build system?

If yes → continue. If no → redesign in week 3, not month 30.

## What this validates

- Zig + C + C++ hybrid build works end-to-end
- `build.zig` codegen of comptime dispatch table is feasible
- Module init-level ordering doesn't have hidden circular dependencies
- Handle indirection through servers doesn't add unbearable overhead
- Import-graph enforcement catches real mistakes
- ImGui via the adapter sub-repo route is workable

## Open dependencies before MVP starts

From [`gaps.md`](gaps.md):

- `project.toml` schema (#1) — even a skeleton
- `mod.toml` schema (#2) — even a skeleton
- `Handle` layout (#7)
- Threading model (#12) — at least the main-thread + render-thread split
- C ABI surface skeleton (#13) — at least the module entry-point shape

These unblock writing the MVP code.

## Next after MVP

Phase 3 (Voxel Core) takes the proven module/server/handle pattern and applies it to a real subsystem. If the MVP feels clumsy, redesign before Phase 3.
