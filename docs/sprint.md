# Sprint 1 — Phase 1: Window + Vulkan Basics

> Active sprint task list. Solo-dev — pace is yours. Mark items `[x]` as they land; create new commits per item per [`CONTRIBUTING.md` § Commit rules](../CONTRIBUTING.md).
>
> **Sprint goal:** colored rotating cube on screen, rendered through the Platform-stack + Vulkan-stack adapters. End-state validates the meta-package adapter pattern + the engine bridge in `src/render/surface.zig`.
>
> **Definition of done:** every item in this doc checked + [`mvp.md`](mvp.md) § "Window + render (Phase 1)" + adapter wiring criteria pass.
>
> **Not in this sprint:** the module system + RenderServer + Handle indirection — that's Sprint 2 (Phase 2). Sprint 1 is "raw working render loop"; Sprint 2 abstracts behind servers.

---

## Section A — Phase 0 cleanup (do first, ~half a day)

Two leftover Phase-0 items per [`gaps.md`](gaps.md). Clear them before adapter work.

- [ ] **A.1** Rename `build.zig` artifact from `demo` → `zvoxrealms` (3 lines: artifact `.name`, run-step description, log strings). Verify `zig build run` still works.
  - Files: `build.zig`
  - Acceptance: `zig-out/bin/zvoxrealms` exists; `zig build run` prints hello-world without error
  - Commit: `chore(build): rename demo artifact → zvoxrealms`

- [ ] **A.2** Match on-disk layout to [`project-structure.md`](project-structure.md). Create the skeleton directories the engine will populate during Sprint 1+2: `src/core/`, `src/platform/` (bridge layer over the adapter), `src/render/`, `src/servers/`, `src/backends/`, `src/scene/`, `modules/`, `assets/`. Each with a `.keep` file so git tracks the empty dirs.
  - Files: new dir + `.keep` per subdirectory
  - Acceptance: `tree -L 2 src/` matches the layout in `project-structure.md`
  - Commit: `chore(layout): scaffold src/{core,platform,render,servers,backends,scene} + modules/ + assets/`

---

## Section B — Adapter wiring (the unblock; ~1-2 days)

The two meta-package adapters become engine deps. Once this section is green, the engine can `@import("platform")` and `@import("vulkan_stack")` and call into them.

- [ ] **B.1** Bring up the **engine's** `build.zig.zon`. Currently doesn't exist (`grep '(no parent build.zig.zon yet)'`). Declare `name = .zvoxrealms`, `version = "0.0.0"`, `fingerprint`, `minimum_zig_version = "0.16.0"`, empty `.dependencies = .{}` for now.
  - Files: new `build.zig.zon`
  - Acceptance: `zig build` still works; manifest validated
  - Commit: `chore(build): add build.zig.zon package manifest for the engine`

- [ ] **B.2** Inside `libs/zig-cpp-platform-stack-adapter/`: extend its `build.zig` so it actually exposes a `platform` module (currently it's the hello-world template). Stub `src/root.zig` with the public API surface from [`specs/platform.md` § Public API surface (v1.0)](specs/platform.md) — all functions can `@panic("not implemented")` for now, except `Window.create` / `Window.destroy` / `nextEvent` which need to actually work via GLFW. Tag this `v0.1.0` and push.
  - Files (in sub-repo): `build.zig`, `src/root.zig`, `src/common.zig`, `src/backend/glfw.zig`, `vendor/glfw/` (submodule)
  - Acceptance: `zig build` in the sub-repo produces a static lib that exports `platform` module; sub-repo's smoke test opens + closes a GLFW window
  - Sub-repo commit: `feat: minimal platform module exposing Window + nextEvent via GLFW backend`
  - Parent commit: `chore(submodules): bump platform-stack → v0.1.0 (minimal Window/event API)`

- [ ] **B.3** Inside `libs/zig-cpp-vulkan-stack-adapter/`: extend its `build.zig` so it actually exposes a `vulkan_stack` module. Add vulkan-zig as a dependency in its `build.zig.zon`. `src/root.zig` does `pub const vk = @import("vulkan");`. Stub `src/vma.zig`, `src/volk.zig`, `src/shaderc.zig` as panic-on-call for now; only the `vk` re-export needs to work. Stub the per-OS surface creators (`createX11Surface`, `createWaylandSurface`, `createWin32Surface`) as panic-on-call too — they'll get real bodies in Section D. Tag `v0.1.0` and push.
  - Files (in sub-repo): `build.zig`, `build.zig.zon` (add vulkan-zig dep), `src/root.zig`, `src/vma.zig`, `src/volk.zig`, `src/shaderc.zig`, `src/surface.zig`
  - Acceptance: `zig build` in the sub-repo produces a static lib; consumer can `const vk = @import("vulkan_stack").vk;` and access `vk.Instance`, `vk.SurfaceKHR`, etc.
  - Sub-repo commit: `feat: vulkan-stack root re-exporting vulkan-zig + stub VMA/volk/shaderc/surface`
  - Parent commit: `chore(submodules): bump vulkan-stack → v0.1.0 (vk re-export + stubs)`

- [ ] **B.4** Engine consumes both adapters via `build.zig.zon`. Add `platform_stack_adapter` and `vulkan_stack_adapter` entries pointing at the sub-repo paths (since they're git submodules, use the `.path = "libs/zig-cpp-..."` style entry). Wire the modules in `build.zig` so `src/main.zig` can `@import("platform")` and `@import("vulkan_stack")`.
  - Files: `build.zig.zon`, `build.zig`, `src/main.zig` (add a stub `_ = @import("platform"); _ = @import("vulkan_stack");`)
  - Acceptance: `zig build run` still runs hello-world; both imports compile without errors
  - Commit: `feat(build): wire platform-stack + vulkan-stack adapters as dependencies`

---

## Section C — Engine bridge + window open (~2-3 days)

The first time engine code actually does something via the adapters. Window opens, events flow, engine bridge file exists.

- [ ] **C.1** Replace `src/main.zig` hello-world with a "open window, pump events, exit on close" loop using the Platform-stack adapter only. No Vulkan yet.
  - Files: `src/main.zig`, possibly `src/platform/init.zig` if you want a thin engine-side wrapper
  - Acceptance: `zig build run` opens a 1280×720 window titled "zVoxRealms — Sprint 1", consumes events, closes when X is clicked or `platform.actionPressed(.menu_pause)` returns true (bind ESC)
  - Commit: `feat(platform): open a window and pump events via platform-stack`

- [ ] **C.2** Action-mapped input wired. Bind a single action (`menu_pause` → Escape key) at startup; query each frame.
  - Files: `src/main.zig` or `src/input/init.zig`
  - Acceptance: ESC quits cleanly (no crash, no leaked window)
  - Commit: `feat(input): bind menu_pause action to ESC; quit on press`

- [ ] **C.3** Engine bridge helper at `src/render/surface.zig` per [`specs/platform.md` § Rule 2](specs/platform.md). Implement the comptime switch over `builtin.target.os.tag`. For now its body can return an error from each branch — we just want the file in place with the right shape.
  - Files: `src/render/surface.zig`
  - Acceptance: compiles; `render.createSurface(undefined, window)` returns an error at runtime (since vulkan-stack stubs panic), proving the call path resolves
  - Commit: `feat(render): scaffold src/render/surface.zig comptime bridge`

---

## Section D — Vulkan instance + surface + clear screen (~3-5 days)

The hard part. Engine creates a real Vulkan instance, picks a device, creates a swapchain, clears to a color each frame. Bulk of Vulkan setup code lives in `src/backends/vulkan/`.

- [ ] **D.1** Implement real `vk_stack.createX11Surface` + `createWin32Surface` in the **vulkan-stack adapter sub-repo**. These take the raw primitives the platform getters return and call the matching `vkCreate*SurfaceKHR`. Tag the sub-repo `v0.2.0`.
  - Files (in sub-repo): `src/surface.zig`
  - Acceptance: passing valid `vk.Instance` + display/window pointers from a live GLFW window produces a non-null `vk.SurfaceKHR`
  - Sub-repo commit: `feat(surface): implement per-OS createX11Surface + createWin32Surface`
  - Parent commit: `chore(submodules): bump vulkan-stack → v0.2.0 (real surface creators)`

- [ ] **D.2** Vulkan instance creation in engine: `src/backends/vulkan/instance.zig`. Enumerate layers + extensions (use `platform.requiredVulkanInstanceExtensions()`); enable `VK_LAYER_KHRONOS_validation` in debug builds; create `vk.Instance`.
  - Files: `src/backends/vulkan/instance.zig`
  - Acceptance: validation layer attaches; `zig build run` prints "Vulkan instance created" + the device count
  - Commit: `feat(vulkan): create Vulkan instance with validation layer in debug builds`

- [ ] **D.3** Surface creation: engine calls `render.createSurface(instance, window)` — the bridge from C.3 + D.1 actually works now.
  - Files: `src/main.zig` integration; no new files
  - Acceptance: surface handle is non-null; validation layer doesn't complain
  - Commit: `feat(render): create Vulkan surface via the platform↔vulkan bridge`

- [ ] **D.4** Physical device selection + logical device + queue families. `src/backends/vulkan/device.zig`.
  - Files: `src/backends/vulkan/device.zig`
  - Acceptance: graphics + present queue acquired; device name + driver version logged
  - Commit: `feat(vulkan): pick physical device + create logical device with graphics/present queues`

- [ ] **D.5** Swapchain creation + image views. `src/backends/vulkan/swapchain.zig`.
  - Files: `src/backends/vulkan/swapchain.zig`
  - Acceptance: swapchain creates with 2-3 images; recreates on window resize event
  - Commit: `feat(vulkan): create swapchain + image views; recreate on window resize`

- [ ] **D.6** Command pool + command buffers + a per-frame clear-color pass. Each frame: acquire image → record clear-color command → submit → present.
  - Files: `src/backends/vulkan/frame.zig`
  - Acceptance: window is solid blue (or whatever clear color); 60 FPS with vsync; nothing in validation layer
  - Commit: `feat(vulkan): per-frame clear pass with full acquire/submit/present cycle`

---

## Section E — Rotating cube (~3-5 days)

The actual Phase 1 milestone.

- [ ] **E.1** Vertex + index buffer creation for a cube. Vertex shader, fragment shader (use shaderc via `vk_stack.shaderc` — implement shaderc wrapper now in the vulkan-stack adapter sub-repo if not yet real, OR use precompiled SPIR-V bytecode embedded in Zig source for v1).
  - Files: `src/backends/vulkan/mesh.zig`, `assets/shaders/cube.{vert,frag}.glsl`, possibly `src/backends/vulkan/shader.zig`
  - Acceptance: static cube renders at origin facing camera
  - Commit (possibly split): `feat(vulkan): cube vertex/index buffers + shader pipeline`

- [ ] **E.2** Uniform buffer for the model-view-projection matrix. Per-frame update with a rotation around Y axis.
  - Files: `src/backends/vulkan/uniform.zig`, math added inline or in `src/core/math.zig`
  - Acceptance: cube rotates smoothly at 60 FPS on the dev machine; visible from a slight elevation
  - Commit: `feat(vulkan): uniform buffer + per-frame rotation matrix`

- [ ] **E.3** Smoke verification on second target. If primarily on Linux: cross-compile + run on a Windows VM (or test via `zig build -Dtarget=x86_64-windows-gnu` + Wine). Marks the per-target tree-shake guarantee real.
  - Files: none
  - Acceptance: cube renders on Windows target; `nm` (or equivalent) on Linux build shows no Win32 symbols
  - Commit: `test(phase1): verify cube renders on Windows target; verify per-target tree-shake`

---

## Section F — Sprint close (~half a day)

- [ ] **F.1** Update [`mvp.md`](mvp.md) Definition-of-Done checklist — mark every Phase 1 item ✅
  - Commit: `docs(mvp): mark Phase 1 items as ✅ after sprint 1 close`

- [ ] **F.2** Update [`ROADMAP.md`](ROADMAP.md) Phase 1 to "Complete" + Phase 2 to "Active"; close Phase 0 items A.1 + A.2.
  - Commit: `docs(roadmap): close Phase 1; activate Phase 2`

- [ ] **F.3** Retrospective in this file: what surprised you about Zig 0.16's build system, the adapter pattern, Vulkan setup? Note anything that pushes back on a Phase 0 design decision (we'd rather revise now than after Phase 3 ships).
  - Files: append "## Sprint 1 retrospective" section to this doc, OR archive this file as `sprint-1.md` and start a fresh `sprint.md` for Sprint 2 (Phase 2)
  - Commit: `docs(sprint): close sprint 1 retrospective; tee up sprint 2`

---

## Notes on the work

### What you write yourself vs. what AI helps with

Per [`docs/guard.md`](guard.md) + [`feedback_learning_mode_guard`](.) memory:

- **You write all .zig / .c / .cpp / .h / .hpp engine code by hand.** No AI-generated bodies for Vulkan setup, the bridge helper, mesh creation, render loop, etc.
- **AI helps with:** reviewing your code, debugging compile errors you paste, drafting `.github/`, `.clang-format`, `LICENSE`, `build.zig.zon` config when small, documentation updates, scaffolding empty files, test code (limited)
- **AI does not write:** `build.zig` logic beyond comment annotations, the bridge `surface.zig`, anything under `src/backends/vulkan/`, mesh data, shader source

If you get stuck on syntax or "what's the right Zig 0.16 idiom for X" — paste the error or the code snippet; ask. Don't paste "write me a Vulkan init function."

### Commit cadence

Per [`CONTRIBUTING.md` § Commit rules](../CONTRIBUTING.md): atomic, one concern per commit, ≤72 char subjects (≤150 hard cap), trunk-based — push to `main` little-and-often. Each `[ ]` item above maps to roughly one commit.

### CI

CI runs on every push/PR. If lint fails locally before push (`zig fmt --check build.zig src/` + `clang-format --dry-run -Werror` on src/c + src/cpp), fix before pushing — don't waste a CI cycle.

### Estimating

This whole sprint is roughly **2-4 weeks of solo evenings/weekends** depending on Zig + Vulkan familiarity. Section D is the biggest unknown — Vulkan setup is famously verbose. If a section drags, pause and reduce scope (e.g. defer E.3 cross-platform verification to a "Sprint 1.5" cleanup pass).

---

## Sprint 2 preview (next, not now)

Phase 2 = Module System + Server Pattern. Wraps the raw Vulkan code from Sprint 1 behind `RenderServer.drawMesh(handle)`. Implements the comptime dispatch codegen + 4 init levels + opaque `Handle` per [`specs/core-types.md`](specs/core-types.md). Loads `modules/hello/` as the smoke test. Same rotating cube but reached only via Server calls — scene code holds no raw Vulkan.

When Sprint 1 closes, archive this file (`sprint-1.md`) and write a fresh `sprint.md` for Sprint 2.
