# Gap Resolution — Reference Patterns

> For each missing system in [`gaps.md`](gaps.md), where to look in Hazel, Luanti, Godot, and UnrealEngine for proven patterns to **adapt** (never copy verbatim). All paths verified against local clones at `$REFS/{Hazel,luanti-custom,godot,UnrealEngine}/`. Pairs with [`engine-references.md`](engine-references.md) (the general catalog).

## ⚠ Legal reminder — non-negotiable

Every reference below is **read, understand, reimplement in Zig from scratch**. The rule from [`engine-references.md` § Legal](engine-references.md) and [`licensing.md` § Reference-engine policy](licensing.md) applies absolutely:

- **Unreal** = Epic Source EULA — verbatim copies = lawsuit risk
- **Luanti** = LGPLv2.1+ — verbatim copies force the engine GPL
- **Godot** + **Hazel** = permissive but adapt the pattern; verbatim copies create attribution debt

If you find yourself wanting to translate a function line-by-line, **stop**, write a paragraph describing what the function does, then implement from the paragraph without re-reading the source.

---

## Tier 1 (mission-blocking) — reference patterns

### §2.1.A Animation system

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot** ✅ best fit | `$REFS/godot/scene/3d/skeleton_3d.cpp` (bones + skin) · `$REFS/godot/scene/animation/animation_tree.cpp` + `animation_blend_tree.cpp` + `animation_blend_space_{1d,2d}.cpp` (state machines + blending) | The `AnimationTree` graph model: nodes for blends, state machines, transitions. Skeleton bone hierarchy + pose interpolation. Port the *concept*; the C++ class hierarchy doesn't map to Zig. |
| **Unreal** ⚠ overkill | `$REFS/UnrealEngine/Engine/Source/Runtime/AnimationCore/` + `$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/Animation/` (`AnimationAsset.h`, `AnimSequence.h`, `AnimMontage.h`) | The Anim Graph concept (data-flow graph of pose nodes). Don't port the UObject machinery. |
| **Luanti** ⚠ minimal | `$REFS/luanti-custom/src/client/content_cao.cpp` (entity attached animations) | Basic skinned-mesh playback only — fine reference for "what's the minimum viable" |
| **Hazel** | minimal animation support | skip |

**Voxel-character specific** — none of the references handle voxel-mesh animation well. This is novel territory. Start from Godot's skeleton + manually skin voxel chunks. Consider studying [MagicaVoxel](https://ephtracy.github.io/) export formats for rigged voxel models.

### §2.1.B Particle / VFX system

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot** ✅ best fit | `$REFS/godot/scene/3d/gpu_particles_3d.cpp` (the node) · `$REFS/godot/scene/resources/particle_process_material.cpp` (GPU material) · `$REFS/godot/servers/rendering/renderer_rd/effects/` (rendering side) | GPU-driven particle compute pass; emitter shape primitives (box / sphere / mesh); attractor + collision via voxel raycasts. |
| **Unreal — Niagara** ⚠ huge | `$REFS/UnrealEngine/Engine/Source/Runtime/Niagara/Public/` + `$REFS/UnrealEngine/Engine/Plugins/FX/Niagara/Source/Niagara/` | Niagara's stack-based emitter authoring is the gold standard but the implementation is enormous. Study the **module/emitter/system** layering; reimplement at 5% complexity. |
| **Luanti** | basic particles, low priority reference | skip |

**Adaptation note:** start with GPU compute + indirect draw. Emitter TOML schema (position, lifetime, velocity, color over life). Voxel-world interaction = particles raycast against chunk boundaries.

### §2.1.C Audio architecture

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot** ✅ best fit | `$REFS/godot/servers/audio/audio_server.cpp` (bus + mixing) · `$REFS/godot/scene/audio/audio_stream_player.cpp` + `audio_stream_player_3d.cpp` + `audio_stream_player_2d.cpp` (the playback API) · `$REFS/godot/servers/audio/effects/` (reverb, distortion, etc.) | Bus tree (SFX/Music/Dialog/UI as named buses with per-bus volume + effects chain). 3D positional source attachment. The miniaudio backend you've chosen sits *below* this layer. |
| **Unreal — Audio Mixer** | `$REFS/UnrealEngine/Engine/Source/Runtime/AudioMixer/` | Source/submix routing graph. Cue triggering. Way more complex than needed; study the bus/submix concept only. |
| **Luanti** ⚠ minimal | `$REFS/luanti-custom/src/sound/` (basic SDL_mixer wrapper) | Skip — too basic for the design needed. |

**Adaptation:** miniaudio handles the platform/decode/mix. Build the **bus tree + spatializer + occlusion** as a thin Zig layer on top. Reverb zones triggered by entering scene volumes.

### §2.1.D UI layout engine + widget set

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot** ✅ best fit | `$REFS/godot/scene/gui/control.cpp` (the base Control + anchor system) · `$REFS/godot/scene/gui/` directory (every widget: `button.cpp`, `label.cpp`, `panel.cpp`, `scroll_container.cpp`, `text_edit.cpp`, etc.) | Anchor + offset layout (Control's `anchor_*` + `offset_*`). Theme system (every widget gets style from a theme resource). Focus-graph navigation for controllers. |
| **Unreal — UMG/Slate** ⚠ Slate is heavy | `$REFS/UnrealEngine/Engine/Source/Runtime/UMG/` (game-side) · `$REFS/UnrealEngine/Engine/Source/Runtime/Slate/` (the underlying immediate-mode framework) | Concept of "slot-based layout" + invalidation panels. Don't port Slate's template machinery. |
| **Luanti — formspecs** | `$REFS/luanti-custom/src/gui/` (`guiButton.cpp`, `guiAnimatedImage.cpp`, `guiBox.cpp`, `guiInventoryList.cpp`) + `$REFS/luanti-custom/src/gui/guiFormSpecMenu.cpp` (the formspec parser) | The **formspec string DSL** maps surprisingly well to your TOML+SCSS plan. Study the parser as a data → widget tree mapping. |
| **Hazel** | ImGui only — skip (we already use ImGui for editor) | skip |

**Adaptation:** anchor system from Godot, declarative TOML from Luanti's formspec philosophy, styling via SCSS-parsed-to-theme. Focus graph for controller nav is critical.

### §2.1.E Event / messaging bus

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — signals** ✅ best fit | `$REFS/godot/core/object/object.cpp` + `$REFS/godot/core/object/object.h` (look for `_emit_signal`, `connect`, `disconnect`, `Signal`, `Callable`) | The Signal+Callable pattern: declare signals on a node, anyone can connect a callable, emit fires all listeners. Zig comptime makes this type-safe; Godot uses Variant. |
| **Unreal — delegates** | `$REFS/UnrealEngine/Engine/Source/Runtime/Core/Public/Delegates/Delegate.h` + sibling `DelegateBase.h`, `DelegateCombinations.h` | Multi-cast delegates with `DECLARE_DYNAMIC_MULTICAST_DELEGATE_*` macros. The runtime-bound version (DynamicMulticast) is closest to what you want; Static delegates are templates. |
| **Luanti — registered callbacks** | `$REFS/luanti-custom/src/script/lua_api/` (Lua API callback registration) | Hook-based — `minetest.register_on_player_join`, etc. Pattern: typed event names + handler list. Replace Lua with Zig comptime. |

**Adaptation:** Zig comptime gives you type-safe events without Variant tax. Define events as comptime structs; listeners register at module-init time; emit fires the listener list. No reflection needed.

### §2.1.F Scene lighting + decals + weather + time-of-day

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — lighting** ✅ | `$REFS/godot/scene/3d/light_3d.cpp` (directional/omni/spot bases) · `$REFS/godot/scene/3d/world_environment.cpp` (sky, ambient, fog) · `$REFS/godot/servers/rendering/renderer_scene_render.cpp` (the actual pipeline) | Three light types as data, environment as a scene resource, renderer batches by light type. |
| **Godot — decals** ✅ | `$REFS/godot/scene/3d/decal.cpp` | Decal as an AABB projector with texture + albedo/normal/orm masks. Voxel-surface projection is the implementation challenge. |
| **Unreal — sky/weather** | `$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/Components/SkyAtmosphereComponent.h` (volumetric sky) | The atmosphere LUT approach. Probably overkill; a simple skybox + sun direction is enough for v1.0. |
| **Unreal — decals** | `$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/Components/DecalComponent.h` | Decal projection box; same idea as Godot. |
| **Luanti — day-night** ✅ | `$REFS/luanti-custom/src/server.cpp` + `$REFS/luanti-custom/src/serverenvironment.cpp` (search for `time_of_day`, `m_time_of_day`) | Time-of-day as a single float in [0,1), client interpolates lighting from it. Multiplayer-safe (server-authoritative). |

**Adaptation:** start with Luanti's time-of-day model (single float, server-owned, replicated). Add Godot-style light types as scene nodes. Decals as the next phase — project from AABB onto voxel surface normals. Weather is a particle-system consumer (snow/rain emitters tied to weather state).

### §2.1.G Materials + shader management

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — materials** ✅ | `$REFS/godot/scene/resources/material.cpp` (StandardMaterial3D, ShaderMaterial, parameter system) · `$REFS/godot/servers/rendering/storage/material_storage.cpp` (runtime material storage + pipeline cache) | Material as a typed resource referencing a shader + parameters. PBR parameter set (albedo, normal, roughness, metallic, emissive, AO). Pipeline-state cache keyed by shader + render-target format. |
| **Unreal — materials** | `$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/Materials/` (`Material.h`, `MaterialInterface.h`, `MaterialInstance.h`) | Material *Instance* hierarchy (base material → parameterized instance) is a strong pattern. Skip the visual material-graph editor for v1.0. |

**Adaptation:** PBR material struct as data (TOML-authored), shader registry keyed by feature flags (has_normal_map, has_emissive, etc.) → Vulkan pipeline cache. Pipeline-state derived from material + mesh vertex layout + render pass. Hot-reload on shader-file change re-bakes pipelines.

### §2.1.I Threading model — `CommandQueueMT` pattern

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot** ✅ | `$REFS/godot/servers/rendering/server_wrap_mt_common.h` + `$REFS/godot/core/templates/command_queue_mt.h` | Main thread enqueues; render thread drains. SPSC ring buffer. Variable-size commands. Sync barriers when main thread needs a value back. |
| **Unreal — Chaos** | `$REFS/UnrealEngine/Engine/Source/Runtime/Experimental/Chaos/Public/Chaos/ChaosMarshallingManager.h` | Game-thread ↔ physics-thread marshalling. Per-frame double-buffered commands. Apply to all servers, not just physics. |

**Adaptation:** Zig has good concurrency primitives. Build a per-server SPSC ring (one for render, one for physics, one for voxel-gen, etc.). Main thread fire-and-forget for state changes; sync barrier when querying.

### §2.1.J Content authoring tools

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — editor tools** ✅ | `$REFS/godot/editor/plugins/` (every editor plugin: particle editor, mesh editor, etc.) | Per-asset-type editor pattern. Procedural mesh tools (cylinder/sphere/torus generators). Theme editor pattern applies to your voxel-atlas editor. |
| **Luanti — mod-driven content** | `$REFS/luanti-custom/builtin/mainmenu/` | Mod-as-content-pack pattern; voxel/recipe registration via mod scripts. Replace Lua with TOML + script ABI. |

**Adaptation:** Bulk-import flows are the killer feature. CSV → recipe TOML, sprite-sheet → voxel-atlas, etc. Procedural NPC generator is its own project — defer the *visual* NPCs to v1.1; ship v1.0 with simpler placeholder rendering.

---

## Tier 2 (usability-blocking) — reference patterns

### §2.2.B Localization (i18n / l10n)

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — translations** ✅ | `$REFS/godot/core/string/translation.cpp` (TranslationServer + locale lookup) · `$REFS/godot/core/string/translation_po.cpp` (gettext .po reader) | String-table architecture: key → translated string per locale. `tr("ui.menu.start")` style. gettext .po format for editor compat (translators use existing tools). |
| **Unreal — internationalization** | `$REFS/UnrealEngine/Engine/Source/Runtime/Core/Public/Internationalization/` (`Culture.h`, `FastDecimalFormat.h`, `BreakIterator.h`) | Culture (locale + region). Number/date formatting per culture. Plural rules. CLDR-style data. |

**Adaptation:** Use TOML for string tables (one file per locale, e.g. `assets/locales/en.toml`, `de.toml`). Compile to a fast-lookup binary at import time. Plural rules from CLDR; embed the rule table.

### §2.2.C Save UX (slots, autosave, migration)

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Unreal — SaveGame** ✅ | `$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/GameFramework/SaveGame.h` + `SaveGameSystem.h` | `USaveGame` subclass per-game; serialize via Archive. Slot-based save naming. Async write to disk. |
| **Godot — no built-in** | demos: `$REFS/godot/modules/gdscript/tests/scripts/` may have examples | Godot leaves save UI to game code. Look at common community patterns: ConfigFile for settings, JSON or binary for game state. |

**Adaptation:** Slot UI is just an editor panel — file list with screenshot thumbnails + metadata (playtime, level). Autosave on milestone events (scene change, save point). Steam Cloud sync via Steamworks adapter.

### §2.2.D Mod manager UX

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Luanti — mainmenu** ✅ best fit | `$REFS/luanti-custom/builtin/mainmenu/dlg_create_world.lua` + `dlg_contentstore.lua` + `dlg_clients_list.lua` | Mod list with enable/disable toggles, dependency display, conflict warnings, Workshop-equivalent (contentstore). Built into a "main menu" UI layer. |
| **Godot — no built-in** | N/A | Skip |
| **Unreal — no built-in** | N/A | Skip |

**Adaptation:** Luanti is the gold standard here despite the Lua. Read the *UX flow* (not the Lua code). Mod browser with sortable list, per-mod dependency graph, Workshop integration via Steamworks.

### §2.2.E Diagnostics + crash dumps

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — crash handler** | `$REFS/godot/core/os/crash_handler*.cpp` + `$REFS/godot/editor/editor_crash_handler.cpp` | Platform signal handlers (SIGSEGV etc.), stack-trace capture, crash log file. |
| **Unreal — crash reporter** | `$REFS/UnrealEngine/Engine/Source/Programs/CrashReportClient/` | Separate process for crash report UI; out-of-process capture is more reliable. |

**Adaptation:** Out-of-process crash reporter (spawned at startup, parent's mini-dump on signal). Symbol upload from CI to Sentry-equivalent. Player dialog for opt-in upload.

### §2.2.J Accessibility

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — accessibility** | `$REFS/godot/scene/gui/control.cpp` (focus mode, focus neighbors) | Focus-graph for controller nav. |
| **Unreal — accessibility** | Limited engine-level support; game-side feature | Skip engine references; just bake from day one. |

**Adaptation:** No strong reference engines for game-side a11y. Bake in from the UI engine design (§2.1.D): colorblind-safe theme defaults, high-contrast mode, font scaling slider, full key+controller remap, subtitle controls, FoV slider, head-bob toggle.

### §2.2.K Camera system

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Godot — camera** ✅ best fit | `$REFS/godot/scene/3d/camera_3d.cpp` (Camera3D base, projection, frustum) | Projection setup (perspective/ortho), frustum culling, view matrix. |
| **Unreal — camera** | `$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/Camera/CameraComponent.h` | Component-on-actor pattern. Useful for "camera follows character" rig. |

**Adaptation:** Single `Camera` struct in the scene layer; mode (first-person / third-person orbit / top-down) is a behavior, not a class hierarchy. Camera holds a `Handle` to the controlled entity. Per-game camera mode selected via `project.toml`.

---

## What's intentionally not mapped

Some gaps don't have meaningful reference-engine answers — they're original engineering or product decisions:

- **§2.1.B (particle-voxel interaction)** — voxel-specific; no engine has prior art
- **§2.2.A (CI/release automation)** — engine-agnostic; use standard GitHub Actions patterns
- **§2.2.F (engine docs for game devs)** — write originally; reference Godot's docs site for *structure* (tutorials + class reference + howtos)
- **§2.2.G (test strategy)** — engine-agnostic; standard test-pyramid practices
- **§2.2.H (memory budget enforcement)** — original Zig allocator instrumentation
- **§2.2.I (Steam achievements)** — define your own schema; Steamworks SDK handles delivery
- **§3 decisions** — those are value choices (chunk size, voxel-ID width), not pattern adoptions

---

## How to use this doc

When picking up a gap to work on:

1. Read the gap entry in [`gaps.md`](gaps.md)
2. Find the same gap here for reference patterns
3. Open the cited reference files in `$REFS/`
4. **Write a one-paragraph "pattern summary"** of what the reference does
5. **Close the reference file**
6. Implement from your summary in Zig — never re-open the reference while writing

This workflow is the legal + intellectual safeguard. See [`engine-references.md` § Legal](engine-references.md) for why this is non-negotiable.
