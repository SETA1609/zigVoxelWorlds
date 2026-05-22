# zVoxRealms — Mission

> The current-state operating statement. What we are doing, why, and on what discipline. The long-term aspiration lives in [`vision.md`](vision.md); the phased plan lives in [`ROADMAP.md`](ROADMAP.md).

## Mission statement

**Build a Zig-first voxel engine that lets a solo developer ship four kinds of voxel game — Daggerfall-class RPGs, Stardew-style life sims, Atelier-style crafting games, and rogue-like dungeon crawlers — with 4-player co-op by default, host-authoritative networking, deterministic regen + delta saves, first-class modding, and per-project tree-shaken dynamic-lib export — running at 50–60 FPS on a low-end PC. All four target games are voxel-3D; no 2D fallback, no mesh-only worlds.**

That's the mission. Every commit either serves it, refactors toward it, or removes something that isn't.

---

## What we are building, right now

The engine is in **Phase 0 (Foundation)** of [`ROADMAP.md`](ROADMAP.md). The on-disk state today:

- `src/main.zig` + C/C++ hello-world stubs, building via `build.zig`
- All planning docs in `docs/` finalized
- **Not yet:** the architecture's directory layout (`src/core/`, `src/servers/`, `modules/`, `adapters/`, etc.). Phase 0 finishes when the layout matches [`project-structure.md`](project-structure.md).

The immediate MVP — what we'll have when Phases 1 + 2 land — is intentionally small:

- Engine binary opens a Vulkan window
- Module system contract working, with `build.zig` codegen of a comptime dispatch table
- `modules/hello/` registers across the four init levels (`.core`, `.servers`, `.scene`, `.editor`)
- `RenderServer` exists; scene code holds only opaque `Handle` (u64)
- Rotating cube renders via `RenderServer.drawMesh(handle)` — no direct Vulkan from scene code
- Build artifact renamed from `demo` to `zvoxrealms`

Nothing more is in the MVP. The full Daggerfall-class slice is **v1 Vertical Slice (Phase 15)**, not MVP. Conflating them was a mistake we caught early.

---

## Operating principles

Six rules that govern day-to-day decisions:

### 1. Zig-first; C/C++ only when it pays off

Default to Zig. Use C/C++ when a mature library (Jolt, VMA, Steamworks, libghostty) costs more to rewrite than to wrap. All C/C++ lives behind an `extern "C"` adapter under `adapters/<name>/`; the Zig side never sees raw C++.

### 2. Build for the four target games

Every feature is evaluated against "does this serve Daggerfall-clone, voxel Stardew, Atelier, or rogue-like?" If it serves none, it's out of scope. If it serves exactly one, it ships as an optional module that the project's `project.toml` enables.

### 3. Engine-as-app + per-project tree-shaken export

The engine is one binary that opens as Project Manager or editor. Export reads `project.toml`, runs `zig build-lib -dynamic` over the modules that project enables, and ships a thin launcher + `libzvox-runtime.{so,dll}` + PCK. No fat templates.

### 4. Modding is a first-class engineering concern

The C ABI used by mods is the same one used by game scripts. Versioned, deprecation-controlled. Engine refactors that break this ABI are not allowed to land without a migration story for already-shipped games.

### 5. Performance budget is non-negotiable

50–60 FPS on i3 / Ryzen 3 + integrated graphics, 4-player co-op, < 3.5 GB RAM. If a feature can't fit this budget, it doesn't ship in that form. Profiling is wired in early (`src/core/profile.zig` stubs Phase 2, Tracy Phase 5) so we know what we're paying for.

### 6. Decisions live in `docs/`, not in heads

Every architectural decision has a documented home. Save model, editability policy, observability strategy, engine-vs-game split, library categorization, export pipeline — each has a doc with a stable URL. If a decision isn't written down, it isn't made.

---

## What success looks like, near-term

We've succeeded in this phase of the mission when:

- **Phase 2 complete:** module system + RenderServer + handle discipline working, rotating cube rendered through handles
- **Phase 3 complete:** voxel core shipped as a module; walkable 1 km² generated world
- **Phase 5 complete:** physics + Tracy integration; can profile a frame in real Tracy
- **Phase 11 complete:** Project Manager opens at no-arg launch; creates a new project; re-execs into editor
- **Phase 13 complete:** export pipeline produces a per-project `libzvox-runtime` containing only enabled modules, verified by `nm` showing missing-module symbols are absent
- **Phase 15 complete:** v1.0 voxel Daggerfall slice — one town, 3–5 dungeons, one main quest line, classless skills + spellmaking + crafting, 4p co-op stable for a 1-hour session, mod loader live, modkit shipped. Released as an exported per-project bundle on Steam, running on a clean machine without Zig installed.
- **Post-v1.0 (v1.1–v1.3, ~10 months total):** voxel rogue-like → voxel Stardew → voxel Atelier, each shipped as a separate Steam product reusing the v1.0 engine + content-authoring tools. See [`vision.md`](vision.md) § Shipping strategy

Each phase is **vertical-slice-driven**: it ends with something runnable, not a half-built abstraction.

---

## What we are explicitly not doing

The mission is also defined by what's left out. From [`vision.md`](vision.md), restated as operating discipline:

- **Not adding features that serve only "future games".** Speculative architecture for game shapes outside the four targets is scope creep.
- **Not adding scripting languages other than Zig + C++.** No Lua, no GDScript, no Python.
- **Not shipping a no-code editor.** Visual scripting is deferred to post-1.0, possibly never.
- **Not chasing photorealism.** Stylized voxel only.
- **Not building for consoles.** Linux + Windows desktop. Android later.
- **Not supporting MMO-scale multiplayer.** 40–50 player ceiling.
- **Not optimizing for the latest GPU.** Optimize for the worst hardware in scope (i3 + iGPU). If it runs there, it runs everywhere we ship.

---

## Decision process

When a question doesn't have a documented answer, the resolution order is:

1. **Does it serve a target game?** If yes → continue. If no → out of scope.
2. **Does the answer change the engine/game split?** If yes → update [`engine-vs-game.md`](engine-vs-game.md) first, code second.
3. **Does it touch the C ABI?** If yes → version-bump policy applies; document the change in adapter notes.
4. **Does it touch the save format?** If yes → version each section; never break existing shipped saves.
5. **Does it cost frame budget?** If yes → benchmark on the target hardware before merging.
6. **Is there a reference implementation to study?** If yes — reach for [`engine-references.md`](engine-references.md) and read the source, not the docs.

When none of the above applies and a real decision needs making: write a memory note, summarize the tradeoff, and pick. Don't optimize for reversibility — most decisions are reversible if they're written down; few are reversible if they aren't.

---

## How this document changes

This mission is the operating posture for the current era of the project. It should be revised when:

- A target game gets dropped or added (changes scope authority)
- The Phase 15 vertical slice ships (the mission moves from "build the engine" to "publish the first game")
- A core principle proves wrong in practice (e.g. "Zig-first" gets reconsidered if we discover a critical reason to ship Rust core code)

Until any of those happens, this document stays as-is and `ROADMAP.md` carries the moving parts.
