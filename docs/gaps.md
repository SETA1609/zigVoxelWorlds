# zVoxRealms — Gap Analysis

> The single source of truth for what zVoxRealms is missing relative to (a) the canonical game-engine architecture, (b) the four target games in [`vision.md`](vision.md), and (c) decisions we haven't made yet. Replaces the older `v1-gaps.md` and `planning-gaps.md`.
>
> **To resolve a gap:** see [`gap-references.md`](gap-references.md) — for each missing system, the specific reference-engine files (Hazel / Luanti / Godot / Unreal) to study and adapt from. Per [`engine-references.md` § Legal](engine-references.md): read, understand, reimplement in Zig — never copy verbatim.

## How to read this doc

Three sections:

- **§1. Canonical engine cores — alignment** — what industry-standard cores exist (per Jason Gregory, *Game Engine Architecture*) and how zVoxRealms stacks up against each
- **§2. Missing systems & features** — what we lack to ship the four target games (tier-ranked: mission-blocking → usability-blocking → post-1.0 polish)
- **§3. Open decisions** — specific data-schema / architecture / process choices that need to be made before or during named phases (43 items; 1 resolved)

**Status legend (§1):** ✅ aligned · ⚠ partial · ❌ missing
**Priority tags (§3):** 🔥 urgent (unblocks Phase 0) · ⏰ before-phase (named in entry) · 📅 later (Phase 12+ / post-v1.0)
**Tier scale (§2):** Tier 1 = no v1.0 without it · Tier 2 = works but painful · Tier 3 = polish after v1.0

Cross-refs: [`vision.md`](vision.md), [`mission.md`](mission.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`ROADMAP.md`](ROADMAP.md), [`engine-references.md`](engine-references.md).

---

## §1. Canonical engine cores — alignment

The industry-standard engine layering, from Jason Gregory's *Game Engine Architecture* (3rd ed., <https://www.gameenginebook.com/>) Chapter 1. Unreal, id Tech, Naughty Dog's ICE, Bevy, Godot, etc. all converge on roughly this set. Mapped against the current zVoxRealms doc set.

| # | Core | Sub-system | Status | Where in docs |
| --- | --- | --- | --- | --- |
| 1 | Platform abstraction | window, input, file I/O, threading, time | ✅ | `src/platform/` in [`project-structure.md`](project-structure.md); Phase 1 |
| 2 | Core systems | allocators, math, containers, asserts, logging, profiling, RNG, handle table | ✅ | [`ARCHITECTURE.md`](ARCHITECTURE.md) high-level layers; Phase 2 stubs (`profile.zig`, `log_sink.zig`, `metrics.zig`); handle table in [`specs/ecs.md`](specs/ecs.md) |
| 3 | Resource manager | asset loading, caching, lifetime, async I/O, hot-reload, refcounting | ✅ | [`tech-stack.md`](tech-stack.md) § Asset Pipeline + Phase 4; GUID-based assetdb |
| 4 | Renderer — RHI | graphics API wrapper | ✅ | `RenderServer` + Vulkan backend; `ARCHITECTURE.md` § Server Pattern |
| 4 | Renderer — scene + culling | scene graph, frustum / occlusion | ✅ spec | [`specs/scene.md` § Culling](specs/scene.md) — frustum (CPU) + HZB occlusion (GPU) two-tier model |
| 4 | Renderer — materials | albedo/normal/rough/metallic descriptors per surface | ✅ spec | [`specs/materials.md`](specs/materials.md) |
| 4 | Renderer — shader management | runtime registry, `VkPipeline` cache, descriptor-set layouts | ✅ spec | [`specs/materials.md`](specs/materials.md) (covered together) |
| 4 | Renderer — particles / VFX | GPU emitters, particle update + render | ✅ spec | [`specs/particles.md`](specs/particles.md) |
| 4 | Renderer — post-processing | tonemap, bloom, FXAA, LUT, vignette | ✅ spec | [`specs/post-processing.md`](specs/post-processing.md) |
| 4 | Renderer — GUI rendering | in-game UI rasterization | ✅ spec | [`specs/ui.md`](specs/ui.md) + ImGui for editor |
| 4 | Renderer — lighting model | directional sun, point/spot lights, ambient probes | ✅ spec | [`specs/lighting.md`](specs/lighting.md) |
| 4 | Renderer — decals | gunshot marks, blood, footprints | ✅ spec | [`specs/lighting.md`](specs/lighting.md) (covered with lighting) |
| 4 | Renderer — scene + culling | scene graph, frustum / occlusion | ✅ spec | [`specs/scene.md` § Culling](specs/scene.md) — duplicate of row above, kept for §1.4 cross-reference stability |
| 5 | Animation | skeletal, morph targets, state machines, blending, IK | ✅ spec | [`specs/animation.md`](specs/animation.md) |
| 6 | Audio | playback, mixing, 3D positional, bus routing, streaming, DSP | ✅ spec | [`specs/audio.md`](specs/audio.md) |
| 7 | Physics + collision | broadphase, narrowphase, rigid body, character controller | ✅ spec | [`specs/physics.md`](specs/physics.md) + Phase 5 |
| 8 | Gameplay — ECS | object model, component storage | ✅ spec | [`specs/ecs.md`](specs/ecs.md) + Phase 7 |
| 8 | Gameplay — world | scene/instancing | ✅ spec | [`specs/scene.md`](specs/scene.md) |
| 8 | Gameplay — scripting | game logic compile + load | ✅ spec | [`engine-vs-game.md`](engine-vs-game.md) § 5 |
| 8 | Gameplay — events / messaging | pub/sub, signal/slot, observer | ✅ spec | [`specs/events.md`](specs/events.md) |
| 9 | AI | pathfinding, behavior systems (BT/GOAP/utility), perception | ✅ spec | [`specs/ai.md`](specs/ai.md) — Behavior Trees + NavMesh (Recast/Detour) + perception + AI LOD tiers |
| 10 | Networking | replication, prediction, server authority | ✅ spec | [`specs/multiplayer.md`](specs/multiplayer.md) + Phase 10 |
| 11 | VFX broader — weather | rain/snow/wind systems | ✅ spec | [`specs/lighting.md`](specs/lighting.md) (covered together) |
| 11 | VFX broader — time-of-day | day/night cycle, sun rotation | ✅ spec | [`specs/lighting.md`](specs/lighting.md) (covered together) |
| 12 | Front-end — UI | HUD, menus, dialog | ✅ spec | [`specs/ui.md`](specs/ui.md) |
| 12 | Front-end — transitions | fade-to-black, scene transitions | ✅ spec | [`specs/ui.md` § Animation / transitions](specs/ui.md) — per-property tween + scene fade + modal slide + menu cross-fade |
| 12 | Front-end — cutscenes | scripted scenes (no video files) | ✅ spec | [`specs/scene.md` § Scripted cutscenes](specs/scene.md) — in-engine only for v1.0; pre-rendered video deferred to v1.x |
| 13 | Tools / pipeline | asset conditioning, editor, debug, profiler | ✅ | [`specs/editor.md`](specs/editor.md), [`specs/project-manager.md`](specs/project-manager.md), importers, export pipeline |
| 14 | Save/load | save format, slots, autosave UI | ✅ spec | save model in [`ARCHITECTURE.md`](ARCHITECTURE.md); slot UX in [`specs/save-ux.md`](specs/save-ux.md); binary format still open in §3 #6 |
| 14 | Replay | record + playback | ✅ | ROADMAP Phase 15 milestone |
| 14 | Telemetry | metrics, traces, structured logs | ✅ | OTel three-tier observability ([`tech-stack.md`](tech-stack.md) § Observability) |
| 14 | Accessibility | colorblind, key remap, font scaling | ✅ spec | [`specs/accessibility.md`](specs/accessibility.md) |
| 14 | Camera system | first-person primary; third-person v1.x | ✅ spec | [`specs/camera.md`](specs/camera.md) — committed first-person primary |

### Summary

**Aligned + spec'd**: the entire canonical cores set now has a design doc or implementation plan. The four previously ❌ MISSING items and one ⚠ PARTIAL all landed in this turn.

**Recently resolved (this turn):**

- ✅ AI subsystem (§1.9) → [`specs/ai.md`](specs/ai.md) — Behavior Trees + NavMesh + perception + AI LOD
- ✅ Post-processing pipeline (§1.4) → [`specs/post-processing.md`](specs/post-processing.md) — ACES + FXAA + bloom + LUT + vignette, single fused pass
- ✅ Front-end transitions (§1.12) → [`specs/ui.md` § Animation / transitions](specs/ui.md) — per-property tween + scene fade + modal slide
- ✅ FMV / cutscenes (§1.12) → [`specs/scene.md` § Scripted cutscenes](specs/scene.md) — in-engine only for v1.0; pre-rendered video out
- ✅ Scene + frustum/occlusion culling (§1.4) → [`specs/scene.md` § Culling](specs/scene.md) — frustum CPU + HZB occlusion GPU

**All canonical cores now have specs.** Remaining work is implementation, not design. The §3 open-decisions list below still has open items (Phase 8 gameplay data models, Phase 10 multiplayer choices, save format) — those are unblocked by phase, not by missing-spec status.

---

## §2. Missing systems & features

Tier 1 = mission-blocking. Tier 2 = usability-blocking. Tier 3 = post-1.0 polish.

### Tier 1: mission-blocking — must land before v1.0

> **Status update:** as of this turn, every Tier 1 item except §2.1.H (Presentation Layer phase landing in ROADMAP) has a spec stub under `docs/specs/`. The remaining work is implementation in the relevant phase, not design. Entries below are kept for reference + as the implementation-status tracker.

#### §2.1.A — Animation system → ✅ spec'd in [`specs/animation.md`](specs/animation.md)

**Biggest single gap.** Daggerfall has thousands of animated NPCs. Stardew villagers walk schedules. Atelier characters animate during synthesis. Rogue-like monsters attack.

Missing:

- Skeletal animation (bones + skin weights)
- Animation blending (idle → walk → run)
- Animation state machines (Mecanim-style or simpler)
- Inverse kinematics (foot placement on uneven voxel terrain)
- Voxel-specific character animation — most engines optimize for triangle-mesh skeletons; voxel characters are a different problem worth solving once

Reference: Godot's `AnimationTree`, Unreal's Anim Graph.

#### §2.1.B — Particle / VFX system → ✅ spec'd in [`specs/particles.md`](specs/particles.md)

Magic effects, footstep dust, fire/smoke, rain, snow, blood, sparkles on Atelier synthesis. The magic system in [`specs/gameplay.md`](specs/gameplay.md) implicitly requires particles on screen but no system creates them.

Missing:

- GPU-side particle simulation
- Emitter authoring (TOML-driven? curve-based?)
- Particle-to-voxel-world interaction (sparks lighting things, snow piling on chunks)
- Performance budget — particles are easy to over-spend on iGPU

#### §2.1.C — Audio architecture beyond library choice → ✅ spec'd in [`specs/audio.md`](specs/audio.md)

[`tech-stack.md`](tech-stack.md) picks miniaudio. That's 10% of audio design. Missing:

- 3D positional / spatial audio
- Audio bus mixing (SFX / music / dialog / UI / ambient — per-bus volume)
- Music streaming + crossfade (combat ↔ explore transitions)
- Reverb zones (cave vs forest vs interior)
- Occlusion (sound through walls quieter)
- Voice / dialog system (subtitle sync, optional TTS fallback)

#### §2.1.D — UI layout engine + widget set → ✅ spec'd in [`specs/ui.md`](specs/ui.md)

[`tech-stack.md`](tech-stack.md) says "TOML layout + SCSS styling." That's the *format*, not the *engine*. Missing:

- Layout primitives (flexbox? CSS Grid? hand-rolled boxes?)
- Widget set (label / button / panel / list / scrollview / image / slider / text input / dropdown / tabs / modal)
- Animation / transitions for UI
- Controller-friendly focus navigation
- DPI scaling
- Font subsystem (FreeType? stb_truetype? CJK fallback chains?)

ImGui handles editor UI. In-game UI (menus / HUDs / inventory / dialog) needs its own engine.

#### §2.1.E — Event / messaging bus → ✅ spec'd in [`specs/events.md`](specs/events.md)

Gameplay foundation needs pub-sub: "player crafted X" → quest trigger + achievement trigger + sound effect + particle spawn. Without this, every system polls or hard-couples to every other.

Missing:

- Event registration / fire / observe API
- Type-safe payload (Zig comptime advantage)
- Listener priority + ordering
- Mod-extensible event types

Critical infrastructure for Phase 8 (gameplay modules). Specify before Phase 7.

#### §2.1.F — Scene lighting, decals, weather, time-of-day → ✅ spec'd in [`specs/lighting.md`](specs/lighting.md)

Currently voxel lighting is local (per-chunk propagation) per [`specs/voxel.md`](specs/voxel.md). Scene-level lighting + decals + weather + time-of-day are absent. Missing:

- Directional sun light (with shadow casting)
- Point / spot dynamic lights (torches, fire, spells)
- Ambient probes / SH for indoor lighting
- Day-night cycle subsystem (sun rotation tied to game time)
- Weather state machine (clear → rain → storm transitions)
- Particle-driven weather (rain, snow, dust)
- Decal projection on voxel surfaces (gunshot marks, blood, footprints)

#### §2.1.G — Materials + shader management → ✅ spec'd in [`specs/materials.md`](specs/materials.md)

Currently the renderer is a `RenderServer` interface with a Vulkan backend. Mesh entities need material descriptors. Runtime needs shader pipeline-state caching.

Missing:

- Material descriptor (albedo / normal / roughness / metallic / emissive)
- Material editor (in the engine editor — Phase 12)
- Shader registry (compile-time + runtime lookup)
- `VkPipeline` object cache
- Descriptor-set layout management
- Push-constant range management
- Draw-call sorting (front-to-back opaque, back-to-front transparent)

#### §2.1.H — Add Phase 7.5: Presentation Layer → ✅ landed in [`ROADMAP.md`](ROADMAP.md)

Phase 7.5 is now in the ROADMAP between Phase 7 (ECS) and Phase 8 (Gameplay). Groups the previously-spec'd presentation systems (animation, particles, audio, UI, events, lighting, materials, dialog) into one coherent phase.

Original rationale (kept for context): items §2.1.A through §2.1.G ship together — they share a tick budget, a streaming model, and are driven by gameplay events. Slot before Phase 8 so gameplay modules can emit events that the presentation layer renders.

> **Phase 7.5 — Presentation Layer**
> Materials + shader registry + scene lighting + decals + post-processing. Animation state machines + skeletal/voxel animation. GPU particle / VFX system. 3D positional audio with bus mixing + reverb zones + music streaming. Weather + time-of-day. In-game UI engine (layout + widgets + controller nav). Event/messaging bus. Game logic emits events; presentation layer renders them.

Probably 3–4 months solo. Currently the most under-scoped area of the entire project.

#### §2.1.I — Threading model → ✅ spec'd in [`specs/threading.md`](specs/threading.md)

Flagged in §3 #12. **Blocks Phase 2** — you can't finalize `RenderServer` / `VoxelServer` / `PhysicsServer` shapes without knowing whether each runs on its own thread + how they communicate.

Needs spec'd before Phase 2 starts:

- Which servers are threaded
- What queue / channel type carries commands between threads
- How the main loop drains them
- Worst-case latency budget per server

Reference: Godot's `CommandQueueMT` ([`engine-references.md`](engine-references.md) → Godot § Server pattern). Create `docs/specs/threading.md`.

#### §2.1.J — Content authoring beyond importers → ✅ spec'd in [`specs/content-authoring.md`](specs/content-authoring.md)

Phase 4 handles **import**. Source content has no plan:

- A Daggerfall-clone needs ~7000 NPC variations. Procedural? Hand-authored?
- The voxel atlas needs hundreds of entries for variety. Who designs it?
- Stardew has ~100 crops + tools + recipes
- Atelier needs an item / recipe database in the hundreds

If the engine doesn't provide **content-authoring tools** (procedural NPC generator, voxel-atlas editor, recipe-bulk-import flow), then "ship a Daggerfall-clone in 3 years" is unrealistic for a solo dev. Either add tools or scope down the v1.0 target game.

### Tier 2: usability-blocking — works but painful without these

#### §2.2.A — CI / cross-compile / release automation

`build.zig` exists; baseline CI landed.

**Resolved baseline** (✅ in `.github/workflows/build.yml`):

- ✅ GitHub Actions lint job — `zig fmt --check` + `clang-format --dry-run -Werror`
- ✅ Build matrix — Linux + Windows (macOS deferred per [`mission.md`](mission.md))
- ✅ Submodule fetch via SSH→HTTPS URL rewrite trick (per `.gitmodules` using SSH URLs)
- ✅ Per-PR pipeline runs before merge — required-status candidate
- ✅ Concurrency block cancels stale runs on force-push
- ✅ Zig artifact cache keyed on (`.gitmodules`, `build.zig`, `build.zig.zon`)

**Still open** (lands as phases approach):

- ⏳ `zig build test` step — needs `build.zig` to declare a test step (Phase 1 follow-up)
- ⏳ Tagged-release auto-build + GitHub Releases upload (Phase 13 export pipeline)
- ⏳ Steam upload automation (Phase 14 Steamworks integration)
- ⏳ Versioning policy (pre-1.0 semver — `0.x.y` where x = phase number?) — decide before first tagged release
- ⏳ Update channel mechanism (dev / beta / stable) — Phase 14+
- ⏳ Android cross-compile to the matrix (post-v1.0)

#### §2.2.B — Localization (i18n / l10n) → ✅ spec'd in [`specs/localization.md`](specs/localization.md) — gettext `.po` canonical

Mentioned in §3 #15 but no roadmap phase. Even single-language v1.0 should plan:

- String table format (TOML — id → translation)
- Code idiom: `t("ui.menu.start")` style
- Format-string substitution (number, gender, plurals)
- Font fallback for non-Latin glyphs (CJK is the hard case)
- Hot-reload of translations in editor

Retrofitting localization is painful. Bake the API in from Phase 2 even if shipping English-only at v1.0.

#### §2.2.C — Save UX beyond the binary format → ✅ spec'd in [`specs/save-ux.md`](specs/save-ux.md)

[Phase 13](ROADMAP.md) handles save *format*. UX is missing:

- Save / load menu UI (multiple slots, screenshots, playtime)
- Autosave cadence + visual indicator
- Save migration on engine update (read old → write new)
- Steam Cloud sync + conflict resolution UI
- Save corruption detection (checksums, recovery dialog)

#### §2.2.D — Mod manager UX → ✅ spec'd in [`specs/mod-manager.md`](specs/mod-manager.md)

Phase 14 has discovery + loading. User-facing UX is missing:

- In-game mod browser (list installed, enable/disable per save)
- Steam Workshop browser inside the engine
- Mod dependency-graph display
- Conflict warnings ("mod A and mod B both replace voxel #42")
- Mod load order UI

Without this, modding is "edit a config file by hand" — violates the "modded as easily as Minecraft" vision.

#### §2.2.E — Diagnostics + crash dump pipeline → ✅ spec'd in [`specs/diagnostics.md`](specs/diagnostics.md)

[Phase 12](ROADMAP.md) has libghostty for editor playtest logs. Shipped games need:

- Crash dump generation on segfault
- Symbol upload from CI for symbolication
- Player-facing "your game crashed, upload report?" dialog
- Save-game recovery on crash

#### §2.2.F — Engine docs for game developers using zVoxRealms

Current `docs/` are internal. A solo dev who wants to *use* zVoxRealms needs different docs:

- "How to make your first project" walkthrough
- API reference (auto-generated from Zig comptime? hand-curated?)
- Tutorial: "build a 1-room dungeon crawler in 30 minutes"
- Module catalog (what does each `modules/<name>/` give you?)
- Cookbook ("how do I add a custom voxel type", "how do I script a quest")

Without this, the 3-year-horizon "small community of developers" in [`vision.md`](vision.md) doesn't happen. Create `docs/user-guide/` as a placeholder.

#### §2.2.G — Test strategy beyond unit tests

Flagged in §3 #33 but undecided. Need:

- Integration tests (full scenario: spawn world → make edit → save → reload → verify state)
- Visual regression (golden-image diffs)
- Save-load round-trip test for every save-section version
- Network simulation (lag/loss/jitter, verify reconciliation)
- Mod-compatibility suite (test mods that should never break)
- Performance regression suite (per-PR FPS budget)

#### §2.2.H — Memory budget enforcement

[`mission.md`](mission.md) says < 3.5 GB RAM. No mechanism to enforce:

- Per-subsystem allocator instrumentation
- Per-frame allocation tracking
- GPU memory accounting (VMA exposes; needs surfacing)
- Hard budgets that fail builds if exceeded

#### §2.2.I — Steam Cloud + achievements schema design

Steamworks adapter is planned. Need:

- Achievement schema (TOML-driven? defined in `project.toml`?)
- Trigger surfaces (gameplay event → achievement check)
- Non-Steam fallback (GOG Galaxy, EOS, disabled)
- Cloud save quota awareness (Steam Cloud has per-game quotas)

#### §2.2.J — Accessibility baseline → ✅ spec'd in [`specs/accessibility.md`](specs/accessibility.md)

Currently absent in docs. v1.0-blocking minimum:

- Colorblind modes / colorblind-safe defaults
- Key + controller remapping (full)
- Font scaling
- Subtitle controls
- Motion-sickness reduction (FoV slider, head-bob toggle)
- High-contrast UI mode

A meaningful accessibility baseline is also a Steam-page selling point, not just a moral one.

#### §2.2.K — Camera system → ✅ spec'd in [`specs/camera.md`](specs/camera.md) — first-person primary committed

Decision committed in [`specs/camera.md`](specs/camera.md): **first-person primary for all four target games**, with third-person added in v1.x. This dramatically simplifies the camera system relative to a multi-mode engine. The earlier per-game framing (Daggerfall FP / Stardew top-down / Atelier third-person / rogue-like top-down) was the assumed-different-cameras model — replaced by the unified first-person commitment.

### Tier 3: post-1.0 polish

Confirmed: these don't block v1.0. Listed to keep them tracked.

- Visual scripting (deferred in [`mission.md`](mission.md))
- Console ports (deferred in [`vision.md`](vision.md))
- WebGPU fallback (deferred in ROADMAP)
- Voice chat / Steam P2P advanced features
- Replay export to MP4 / video capture
- In-engine asset-store browser
- LSP integration inside bundled Neovim (jump-to-definition for engine APIs)
- Multiplayer debug tools (network inspector, lag simulation)
- macOS / iOS ports
- Multi-language scripting beyond Zig + C++
- AI-assisted content generation (procedural NPCs via LLM)
- Screenshot mode + free camera (for trailers)
- FMV / cutscene playback
- Frame-rate-locked rendering for video capture
- **TTS screen reader** (post-v1 accessibility upgrade) — Piper-based offline neural TTS, voice packs distributed as free DLC per platform. Particularly impactful because all four target games are text-only / mute. See [`specs/accessibility.md`](specs/accessibility.md) § Post-v1: TTS Screen Reader

---

## §3. Open decisions

43 items, of which 1 is **resolved** (#38 — camera, decided as first-person primary). Numbering is for citation, not priority. Use the tags for sequencing. The "→ ✅ decided" suffix marks resolved entries; the original number is preserved so prior references in other docs / commits keep working.

### §3.1 — Data schemas (the most urgent gap)

You've referenced `project.toml`, `mod.toml`, `scene.toml`, `orchestrator.toml`, and `assetdb.toml` throughout the docs without specifying their fields. Without these, Phase 0 cannot finish.

1. **🔥 `project.toml` schema** — exact fields per section. `[project]` name/version/engine_compat, `[modules]` enable-disable table, `[scripts]` language+entry, `[export.<target>]` per-platform options
2. **🔥 `mod.toml` schema** — manifest fields: name, version, engine ABI compat range, dependencies, load order, plugin path
3. **⏰ before Phase 9** **Scene definition format** — entity spawn list, region AABBs, edit-policy reference, lighting, weather, time-of-day overrides
4. **⏰ before Phase 9** **`orchestrator.toml`** — how it links scenes to world coordinates and triggers
5. **🔥 `assetdb.toml`** — exact GUID format (UUIDv7 recommended), content-hash algorithm (Blake3 vs SHA-256), per-entry fields
6. **⏰ before Phase 13** **Save file binary format** — exact layout, magic header, version per section, endianness, alignment

### §3.2 — Architecture — type/layout decisions

7. **🔥 `Handle` layout** — u64 `(generation, index)` vs `(server_id, generation, index)`
8. **🔥 Coordinate system** — Y-up vs Z-up; left- vs right-handed; units; block size
9. **🔥 Voxel data layout** — bits per voxel (8/16/32); palette per chunk vs global; voxel-ID semantics
10. **🔥 Chunk size** — 16³ vs 32³ vs 64³ (memory, meshing batch, network packet size)
11. **⏰ before Phase 6** **Origin rebasing strategy** — float64 vs rebase trigger
12. **🔥 Threading model** — see §2.1.I
13. **🔥 C ABI surface design** — concrete `extern "C"` functions for mods/scripts; versioned

### §3.3 — Gameplay shapes — what your four games actually look like at the engine level

14. ~~**⏰ before Phase 1** (input layer) **Input mapping**~~ → ✅ **decided**: action-mapped input with `bindAction`/`unbindAction`/`actionPressed`; bindings load from TOML, per-save rebindings in save format. See [`specs/platform.md` § Action-mapped input](specs/platform.md)
15. **⏰ before Phase 12** **UI scaling** — DPI awareness, controller-friendly nav, font subsystem
16. **⏰ before Phase 13** **Save slot UX** — see §2.2.C
17. **⏰ before Phase 8** **Quest system data model** — flags, state machines, branching dialog, scripted
18. ~~**⏰ before Phase 8** **NPC AI architecture**~~ → ✅ **decided**: Behavior Trees + NavMesh (Recast/Detour) + perception + distance-tiered AI LOD (Hot/Warm/Cold/Frozen) for Daggerfall scale. See [`specs/ai.md`](specs/ai.md)
19. **⏰ before Phase 8** **Magic system data model** — effect composition, mana/cost, cooldowns, target acquisition
20. **⏰ before Phase 8** **Crafting math** — quality formula, time-cost, success curves, station types
21. **⏰ before Phase 8** **Inventory model** — slots, weight, stack rules, container hierarchy, equipped vs carried

### §3.4 — Modding & scripting

22. **⏰ before Phase 14** **Modding security model** — disk/network access, sandbox, threat model
23. **🔥 ABI versioning policy** — semver, compatibility window across engine releases
24. **⏰ before Phase 14** **Script package contract** — entry-function signature, lifecycle hooks

### §3.5 — Multiplayer

25. **⏰ before Phase 10** **Network transport choice** — ENet vs GameNetworkingSockets vs custom UDP
26. **⏰ before Phase 10** **Edit authority model** — host-decides vs client-prediction-with-rollback
27. **⏰ before Phase 10** **Server discovery** — LAN broadcast vs lobby service vs Steam Networking
28. **⏰ before Phase 10** **NAT traversal** — direct-only vs hole-punching service

### §3.6 — Process — how you actually work

29. ~~**⏰ before commit #2** **Branching strategy**~~ → ✅ **decided**: trunk-based with build flags. Single `main`, feature branches merged little-and-often, release tags as snapshot points. See [`CONTRIBUTING.md` § Branching strategy](../CONTRIBUTING.md)
30. **📅 when it goes public** **Issue tracker** — GitHub Issues + labels? Linear?
31. **📅 post Phase 11** **Release cadence** — weekly dev / monthly stable / yearly major
32. **🔥 Test strategy** — see §2.2.G
33. **⏰ before Phase 5** **Performance regression detection** — benchmark suite + per-PR perf budget + CI hardware specs

### §3.7 — Business / publishing

34. **📅 before any Steam page** **Trademark filing** — defensive "zVoxRealms" trademark (USPTO + EUIPO)
35. **📅 before Phase 15** **Steam page / wishlist plan**
36. **⏰ before Phase 10 OTel rollout** **Telemetry consent UX** — opt-in vs opt-out (GDPR)
37. **⏰ before Phase 15** **Crash reporting** — local-only vs opt-in upload vs player-controlled (see §2.2.E)
38. ~~**⏰ before Phase 14** **Camera system spec**~~ → ✅ **decided**: first-person primary for all four target games; third-person added in v1.x. See [`specs/camera.md`](specs/camera.md)
39. **⏰ before Phase 14** **Steam Workshop + DLC integration hooks** — Workshop = `ISteamUGC` (subscribe/download/upload); DLC = `ISteamApps::BIsDlcInstalled` per DLC AppID. Cross-platform fallback for non-Steam builds. DLC distribution (bundled-and-gated vs download-on-purchase). Mod-DLC dependency semantics. See [`engine-vs-game.md` § 3b TODO](engine-vs-game.md) and [`specs/mod-manager.md`](specs/mod-manager.md) § Workshop+DLC
40. **🔥 `project.toml` `kind` + `parent_game` extension** — refinement of #1; needs `kind = "game" | "mod"` field and `[project.parent_game]` block for mod projects. Driven by the new "everything is a mod" model in [`engine-vs-game.md` § 3b](engine-vs-game.md)
41. **⏰ before Phase 13** **Modkit format spec** — what the auto-generated `modkit/` directory contains: `modkit.toml` schema (engine + ABI versions, registered content IDs, enabled engine modules), `headers/` C ABI signature files, `sample_mod/` template, README contract. Outputs as part of game-project export. See [`engine-vs-game.md` § 3b](engine-vs-game.md)
42. **⏰ before Phase 14** **Mod-project workflow in editor** — UI flow: open modkit → "New Mod Project" → editor shows parent game content read-only + new mod content writable. Needs design in [`specs/project-manager.md`](specs/project-manager.md) + [`specs/editor.md`](specs/editor.md)
43. ~~**🔥 Phase 7.5 (Presentation Layer)**~~ → ✅ **landed**: Phase 7.5 added to [`ROADMAP.md`](ROADMAP.md) between Phase 7 (ECS) and Phase 8 (Gameplay Modules)
44. **⏰ before Phase 9** **Edit-policy ownership transitions in multiplayer** — server is authoritative on ownership (plot purchases, quest unlocks, faction membership). Client policy state must update on a server broadcast. Race conditions during simultaneous ownership-change + voxel-edit attempts need defined semantics. See [`specs/scene.md` § Ownership transitions](specs/scene.md)
45. **⏰ before Phase 13** **`session_only` + `transient` delta serialization rules** — save format must gracefully omit non-persistent region deltas. Detection criteria + recovery on partial-save corruption. See [`specs/scene.md` § Persistence semantics](specs/scene.md)
46. **⏰ before Phase 6** **Meshified-chunk LOD interaction** — meshified chunks can bake LOD at multiple distance tiers; voxel chunks generate LOD dynamically. Define coexistence + transition behavior. See [`specs/voxel.md` § Meshified static chunks](specs/voxel.md)
47. **⏰ before Phase 6** **Lighting re-bake cadence for meshified chunks** — pick between (a) periodic re-bake on time-of-day change vs. (b) per-vertex multi-channel lighting blended in the shader. See [`specs/voxel.md` § Lighting in meshified chunks](specs/voxel.md)

---

## Suggested resolution order

> **Status: most of this list is now landed.** Items below show ✅ where a spec exists. The remaining open work is grouped at the bottom.

### Week 1–2 — data-schema docs → ✅ all landed in [`specs/data-schemas.md`](specs/data-schemas.md)

1. ✅ `project.toml` (#1) — landed in `specs/data-schemas.md`
2. ✅ `mod.toml` (#2) — landed in `specs/data-schemas.md`
3. ✅ `assetdb.toml` (#5) — landed in `specs/data-schemas.md`
4. ⏳ Save format header + section versioning (#6, partial — full spec deferred to Phase 13)

### Week 2–3 — architecture micro-decisions → ✅ all landed in [`specs/core-types.md`](specs/core-types.md), [`specs/threading.md`](specs/threading.md), [`specs/c-abi.md`](specs/c-abi.md), [`specs/testing.md`](specs/testing.md)

5. ✅ `Handle` layout (#7) — landed in `specs/core-types.md`
6. ✅ Coordinate system (#8) — landed in `specs/core-types.md`
7. ✅ Voxel data layout (#9) — landed in `specs/core-types.md`
8. ✅ Chunk size (#10) — landed in `specs/core-types.md`
9. ✅ Threading model (#12, §2.1.I) — landed in `specs/threading.md`
10. ✅ C ABI surface skeleton (#13) — landed in `specs/c-abi.md`
11. ✅ Test strategy (#32) — landed in `specs/testing.md`
12. ✅ Branching strategy (#29) — landed in `CONTRIBUTING.md` § Branching strategy (trunk-based with build flags)
13. ✅ ABI versioning policy (#23) — landed in `specs/c-abi.md`

### Pre-coding for each phase

14. ✅ Camera model (before Phase 1, §2.2.K) — landed in `specs/camera.md` (first-person primary committed)
15. ✅ Input mapping (#14) — landed in `specs/platform.md` § Action-mapped input (context stack + synthetic injection + axis modifiers)
16. ⏳ Origin rebasing strategy (before Phase 6, #11) — still open
17. ⏳ Quest / magic / crafting / inventory data models (before Phase 8, #17, #19, #20, #21) — open per game-system design. AI (#18) ✅ landed in `specs/ai.md`
18. ⏳ Network transport + authority (before Phase 10, #25–28) — still open
19. ⏳ Performance regression CI (before Phase 5, #33) — still open
20. ✅ Save UX + slot management (before Phase 13, #16) — landed in `specs/save-ux.md`
21. ⏳ Crash reporting + telemetry consent (before Phase 13–14, #36, #37) — design in `specs/diagnostics.md`; consent UX still open

### Eventual

22. ⏳ Trademark filing (#34) — before public Steam page
23. ⏳ Steam page assets (#35) — before Phase 15
24. ⏳ Release cadence + automation (#31) — before Phase 11

### Concurrent — Phase 7.5: Presentation Layer → ✅ landed in [`ROADMAP.md`](ROADMAP.md)

Phase 7.5 now in ROADMAP between Phase 7 (ECS) and Phase 8 (Gameplay). All Tier 1 presentation specs (animation, particles, audio, UI, events, lighting, materials, dialog) exist as specs ready for implementation.

### Net status summary

- **~13 items landed as specs** since this list was written
- **~11 items still open**, most pre-phase blockers concentrated around Phase 8 (gameplay data models) and Phase 10 (multiplayer)
- The "Save format binary format" (#6) is the biggest remaining Phase-0-blocking item with no spec yet — partial coverage in `specs/save-ux.md` but the binary layout itself isn't pinned down

---

## How this doc gets updated

When an item is decided or a system lands:

1. The decision/system goes into the relevant doc (`ARCHITECTURE.md`, `tech-stack.md`, `engine-vs-game.md`, or a new spec under `docs/specs/`)
2. Cross out the corresponding item here, or replace with a one-line "→ landed in Phase X / commit Y" / "→ decided in [link]"
3. When a tier/category empties, collapse the section
4. When v1.0 ships, this doc closes; rename to `pre-v1-gaps.md` for historical context

This file should shrink over time. When it's empty, you're done.
