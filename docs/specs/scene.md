# Scene & Instancing Spec

> The persistent world + dungeon/tower/house instances, wired via `orchestrator.toml`. Roadmap: [`ROADMAP.md` § Phase 9](../ROADMAP.md). Open questions on scene format: [`gaps.md` § 3](../gaps.md).

## Scope

- **Persistent main world** — seed-deterministic baseline + saved deltas (see [save model in ARCHITECTURE](../ARCHITECTURE.md#cross-cutting-concerns))
- **Instanced scenes** — dungeons, towers, houses; load/unload on transition (door, portal, teleport)
- **`orchestrator.toml`** — links scenes to world locations + triggers
- **Hot-reload of scene definitions** — edits in the editor take effect without restart

## Scene contents

Each `scene.toml` declares:

- Entity spawn list (positions + component overrides)
- Region AABBs with edit policies (see § Edit policy below)
- Lighting setup
- Weather state (or "inherit from world")
- Time-of-day override (or inherit)
- Music / ambient audio cues
- Triggers (volumes that fire scripted events)

## Edit policy — three orthogonal axes + ownership

The world is not uniformly editable. Different regions have different rules — a Daggerfall town center is read-only while wilderness is open; a Stardew farm plot is editable but the village is not; a story dungeon lets you drop items but not break walls; a procedural rogue-like dungeon is fully editable but resets per run.

The current per-policy enum (`full` / `none` / `voxel_type_allowlist` / etc.) is too one-dimensional. The expanded model uses **three orthogonal axes + optional ownership**, all per-region:

| Axis | Values | Controls |
| --- | --- | --- |
| **destroy** | `full` / `none` / `voxel_type_allowlist:<list>` / `coord_range_allowlist:<aabb>` / `tag_allowlist:<list>` / `script:<fn>` | Can the player remove voxels? |
| **place** | `full` / `none` / `tag_allowlist:<list>` / `coord_range_allowlist:<aabb>` / `script:<fn>` | Can the player add voxels / drop items? |
| **persistence** | `persistent` / `session_only` / `transient` (+ `lifetime_trigger:<event>` for `transient`) | Do edits survive save+reload / scene-reload / a triggered reset? |
| **ownership** (optional) | `none` / `purchasable:<cost>` / `quest_gated:<quest_id>` / `faction:<faction_id>` / `script:<fn>` | Does the policy *change* based on game state? |

### Granularity

Policies live **per-region**, not per-chunk. Each scene declares:

- A **scene-level default policy** (applies to all chunks in the scene)
- **Region overrides** (named AABBs within the scene with their own policy)

Per-chunk metadata is only needed for the meshify-or-not rendering decision (see [`specs/voxel.md`](voxel.md) § Meshified static chunks), and that's *derived* from the containing region's policy. Authoring stays clean.

### Policy resolution order

For any voxel modification request (destroy or place):

1. Find the most-specific containing region with an explicit policy (innermost wins)
2. Fall back to the scene-level default if no region matches
3. Fall back to the project-level default from `project.toml` if no scene-level default
4. Resolve `ownership` if present — overrides take precedence based on game state
5. Evaluate the relevant axis (`destroy` or `place`)

### Four worked examples — your four target games

#### Voxel Daggerfall — overworld + city + buyable plot

```toml
[scene.wilderness_overworld]
# Default: open wilderness — full edit
edit_policy.destroy = "full"
edit_policy.place = "full"
edit_policy.persistence = "persistent"

[[scene.wilderness_overworld.regions]]
name = "riverstone_city"
aabb = [[1000, 0, 1000], [1500, 200, 1500]]
edit_policy.destroy = "none"           # city protected
edit_policy.place = "none"
edit_policy.persistence = "persistent"

[[scene.wilderness_overworld.regions]]
name = "plot_west_quarter_3"
aabb = [[1200, 64, 1100], [1240, 130, 1140]]   # plot inside the city
# Default state before purchase
edit_policy.destroy = "none"
edit_policy.place = "none"
edit_policy.persistence = "persistent"
edit_policy.ownership = "purchasable:cost=5000g"
# Policy transition when the player buys this plot
on_ownership_acquired.edit_policy.destroy = "full"
on_ownership_acquired.edit_policy.place = "full"
# persistence stays persistent — the player's house is forever
```

Industry precedent: many open-world RPGs (housing DLCs, MMO plot systems) work exactly this way. Plot starts protected; sells; protected policy flips off for the buyer.

#### Voxel Daggerfall — story dungeon (no destroy, can place items)

```toml
[scene.royal_archive]
edit_policy.destroy = "none"
edit_policy.place = "tag_allowlist:dropped_item"   # players drop loot, items persist
edit_policy.persistence = "persistent"             # or "session_only" if you'd prefer
```

The terrain meshifies (huge perf win for the static walls/floors). Dropped items track in a per-chunk overlay layer, not voxel form.

#### Voxel rogue-like — procedural dungeon (fully editable, resets per run)

```toml
[scene.procedural_dungeon]
edit_policy.destroy = "full"
edit_policy.place = "full"
edit_policy.persistence = "transient"
edit_policy.lifetime_trigger = "dungeon_run_end"   # reset on this event
```

#### Voxel Stardew — restricted farm + open mines

```toml
[scene.farm_valley]
# Default: protected (village, paths, decoration)
edit_policy.destroy = "none"
edit_policy.place = "none"
edit_policy.persistence = "persistent"

[[scene.farm_valley.regions]]
name = "player_farm_plot"
aabb = [[200, 60, 200], [280, 130, 280]]
edit_policy.destroy = "voxel_type_allowlist:tilled_soil,crop,grass,small_rock"
edit_policy.place = "tag_allowlist:crop,fence,decoration"
edit_policy.persistence = "persistent"

[[scene.farm_valley.regions]]
name = "mountain_mines"
aabb = [[600, 0, 600], [800, 60, 800]]
edit_policy.destroy = "voxel_type_allowlist:ore,stone,dirt"
edit_policy.place = "none"                  # no building inside mines
edit_policy.persistence = "session_only"    # mine reshuffles between visits — your call
edit_policy.lifetime_trigger = "scene_exited"
```

### Persistence semantics

| Persistence | Where deltas go | When deltas reset |
| --- | --- | --- |
| `persistent` | Saved in delta section of the save file (per [`ARCHITECTURE.md`](../ARCHITECTURE.md) save model) | Never — deltas live forever |
| `session_only` | Held in RAM during the play session | On scene reload OR app exit |
| `transient` | Held in RAM during the play session | On the named `lifetime_trigger` event |

The save format ([`gaps.md`](../gaps.md) #6) needs to gracefully omit `session_only` and `transient` regions on save. Their deltas aren't serialized.

### Ownership transitions

When `ownership` changes (player buys a plot, completes a quest that unlocks an area, joins a faction), the policy can transition. Mechanics:

- The `on_ownership_acquired` block declares which fields change
- The `on_ownership_lost` block (optional) declares what happens on loss (faction kicked, quest failed, plot reclaimed)
- The transition emits an event (per [`specs/events.md`](events.md)): `region.ownership_changed { region_name, old_owner, new_owner }`
- Multiplayer: the server is authoritative on ownership; clients receive a broadcast and update their local policy state

When a region flips from `destroy = none` → `destroy = full`, all chunks in that region need to switch representation from meshified-static to voxel form. See [`specs/voxel.md`](voxel.md) § Meshified static chunks for the flip cost.

### Place vs. destroy — why the split matters

The single most-asked feature in this kind of system is "the player can leave items in the dungeon but can't tear down walls." The original one-axis policy couldn't express that. The new split makes it trivial — `destroy = "none"` + `place = "tag_allowlist:dropped_item"`.

Similarly for crafting stations, fences, decorations — any "build but don't tear down" or "tear down but don't build" use case.

## Scripted cutscenes — in-engine, no video files

Resolves [`gaps.md` § 1.12 FMV / cutscenes](../gaps.md). **Decision: skip pre-rendered video for v1.0.** All cutscenes run in-engine — camera spline + scene events + dialog modal. Reuses systems already spec'd (no new codec dependency, no Theora/AV1, no extra runtime size).

### Why no video files

- Most modern indies skip video cutscenes — the Daggerfall intro was video; today's equivalents (Caves of Qud, Dwarf Fortress Adventure Mode, Cataclysm) do in-engine + text
- Voxel aesthetic doesn't gain from pre-rendered video — the world IS the visual
- Adding a video codec is a dependency cost without payoff for the four target games
- In-engine cutscenes reuse: camera spline animation (this spec) + dialog modal ([`dialog.md`](dialog.md)) + animation events ([`animation.md`](animation.md)) + audio cues ([`audio.md`](audio.md))

Revisit at v1.x if a specific project needs pre-rendered video.

### Cutscene trigger

A cutscene is a TOML-declared sequence of steps tied to a scene trigger volume or scripted event:

```toml
[scene.royal_archive.cutscenes.opening]
trigger = "on_first_enter"             # event from the trigger system
camera_path = "splines/archive_intro"   # spline asset
duration = 6.5

[[scene.royal_archive.cutscenes.opening.steps]]
at = 0.0
action = "camera.attach_to_spline"

[[scene.royal_archive.cutscenes.opening.steps]]
at = 1.2
action = "audio.play_oneshot"
sound = "sfx/ambient/archive_echo"

[[scene.royal_archive.cutscenes.opening.steps]]
at = 2.5
action = "dialog.open"
dialog_id = "archive_warden_greeting"

[[scene.royal_archive.cutscenes.opening.steps]]
at = 6.5
action = "camera.return_to_player"
```

### Step actions (v1.0)

`camera.attach_to_spline` · `camera.return_to_player` · `camera.set_fov` · `dialog.open` · `dialog.close` · `audio.play_oneshot` · `audio.fade_music_to` · `animation.play_on_entity` · `entity.teleport_to` · `world.set_time_of_day` · `ui.fade_to_black` · `ui.fade_from_black`.

### Skippable

All cutscenes can be skipped (player presses any input). Skip jumps to the cutscene's `on_skip` state — usually `camera.return_to_player` + advance any quest flags the cutscene would have set. Accessibility per [`accessibility.md`](accessibility.md) — never trap the player in a non-skippable scene.

### Multiplayer

Cutscenes do **not** pause the world on dedicated servers ([`multiplayer.md`](multiplayer.md)). In co-op, a cutscene that triggers for one player runs locally for that player only — the camera spline + dialog modal are client-side. Other players keep playing. This is the Skyrim co-op-mod convention; trying to sync cutscenes across players in real time has shipped poorly in most games.

Single-player: world pauses during cutscene per the [`dialog.md`](dialog.md) modal convention if a dialog step is open; otherwise world keeps simulating in the background but player input is captured.

## Culling — frustum (CPU) + HZB occlusion (GPU)

Resolves [`gaps.md` § 1.4 scene + frustum/occlusion culling](../gaps.md). Critical for Daggerfall scale — a 10 km × 10 km world with thousands of chunks means most are never visible. The renderer must not waste work on them.

### Two-tier model

| Tier | Stage | Cost | What it catches |
| --- | --- | --- | --- |
| **Frustum culling** | CPU, per chunk + per entity | < 0.2 ms for thousands of items | Anything outside the camera's view cone |
| **HZB occlusion culling** | GPU, after frustum + depth pre-pass | ~0.3 ms compute | Anything inside the frustum but hidden behind closer geometry |

Both are necessary. Frustum alone leaves "draws everything inside the frustum even behind a mountain"; HZB alone wastes work on already-out-of-frustum chunks.

### Frustum culling

- Each chunk has a cached AABB (8 corners, recomputed only on edit)
- Each entity has an AABB on its transform component
- Per frame: extract 6 frustum planes from the view+projection matrix; test each AABB against all 6 planes
- AABB fully outside any plane → cull
- AABB intersecting → render (over-conservative is fine; HZB catches the rest)

Standard algorithm. Implementation in `src/render/cull/frustum.zig` during Phase 6.

### HZB occlusion culling

The voxel-world-specific perf win. Without it, Daggerfall renders every chunk in the frustum even when a mountain hides 90% of them.

**Hierarchical Z-Buffer (HZB):** build a depth pyramid from last frame's (or this frame's depth pre-pass) Z buffer. Each mip level holds the **maximum** depth of its 2×2 source texels. To test an AABB for occlusion: project to screen space, find the appropriate mip level, sample, compare AABB's nearest depth against the HZB's farthest at that pixel — if AABB is fully behind, cull it.

**Frame flow:**

1. Frustum-cull (CPU) → submit depth pre-pass for survivors
2. Generate HZB pyramid from depth pre-pass (compute, ~0.1 ms)
3. For each survivor, do HZB test → cull the hidden ones
4. Submit color pass for the un-culled remainder

The first frame after a camera jump (teleport, scene change) skips HZB since the previous frame's depth isn't relevant — falls back to frustum-only for one frame. Visible-pop is negligible.

### Per-LOD interaction

HZB tests use the LOD-selected representation. A chunk far enough to be at LOD3 gets its LOD3 AABB tested. Meshified static chunks (per [`voxel.md`](voxel.md)) test the same way — AABB is AABB; the cull is representation-agnostic.

### Cost budget

Frustum cull: < 0.2 ms at 8-chunk view distance + 200 active entities.
HZB cull: < 0.3 ms including pyramid build.

Together < 0.5 ms — well under the per-frame budget. The savings (skipping rasterization of fully-occluded chunks) pays this back many times over on dense scenes.

### Reference patterns

| Engine | Where | What to adapt |
| --- | --- | --- |
| **Godot — frustum** ✅ | `$REFS/godot/scene/3d/visual_instance_3d.cpp` + `$REFS/godot/servers/rendering/renderer_scene_cull.cpp` | Frustum plane extraction + AABB-vs-plane test |
| **Godot — portal occlusion** | `$REFS/godot/scene/3d/occluder_instance_3d.cpp` | Indoor portal-based occlusion — good for cities, less for open voxel terrain |
| **Unreal — HZB** | `$REFS/UnrealEngine/Engine/Source/Runtime/Renderer/Private/HZB.cpp` + `HZBOcclusion.cpp` (verify paths during Phase 6) | The HZB pyramid build + occlusion-query compute. Industry-leading. Study the algorithm, **close the source**, implement in Zig |
| **Luanti** | `$REFS/luanti-custom/src/client/clientmap.cpp` (chunk-distance + frustum only) | Basic — no occlusion. Useful as a "minimum viable" reference |

Adaptation rule per [`engine-references.md` § Legal](../engine-references.md): Unreal HZB code is **especially** lawsuit-risk per [`feedback_reference_engine_no_verbatim.md`](../../memory/feedback_reference_engine_no_verbatim.md). Read the paper that backs it (Niessner et al. "Real-time Rendering of Massive Unbounded Voxel Worlds" + the original HZB paper by Greene/Kass/Miller 1993) for the algorithm, then implement from first principles.

## Reference patterns

- Godot scene-instancing pattern (user-facing UX) — see [`engine-references.md` → Godot](../engine-references.md)
- Open-world RPG housing systems (multiple games have this; the player buys a plot inside a protected town and the plot becomes editable) — precedent for the ownership-transition pattern
- Minecraft worldguard / WorldEdit plugins for region-based policy precedent (community wisdom on what authors want)
- MMO housing systems for "rest of the city protected; your plot is full edit"

## Milestone (from ROADMAP)

Enter a dungeon from the overworld, exit, state persists. Buy a plot in a city, build a house, save + reload — house is still there. Walk into a procedural rogue-like dungeon, destroy walls, exit + re-enter — dungeon has reset.
