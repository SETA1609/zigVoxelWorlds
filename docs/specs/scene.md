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

## Reference patterns

- Godot scene-instancing pattern (user-facing UX) — see [`engine-references.md` → Godot](../engine-references.md)
- Open-world RPG housing systems (multiple games have this; the player buys a plot inside a protected town and the plot becomes editable) — precedent for the ownership-transition pattern
- Minecraft worldguard / WorldEdit plugins for region-based policy precedent (community wisdom on what authors want)
- MMO housing systems for "rest of the city protected; your plot is full edit"

## Milestone (from ROADMAP)

Enter a dungeon from the overworld, exit, state persists. Buy a plot in a city, build a house, save + reload — house is still there. Walk into a procedural rogue-like dungeon, destroy walls, exit + re-enter — dungeon has reset.
