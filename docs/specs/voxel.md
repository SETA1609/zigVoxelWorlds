# Voxel System Spec

> What `modules/voxel_core/` provides. The roadmap entry is [`ROADMAP.md` § Phase 3](../ROADMAP.md). Reference patterns: [`engine-references.md` → Luanti](../engine-references.md). Open questions: [`gaps.md`](../gaps.md) §1 #5, §2 #9, #10, #11.

## Scope

The voxel core is the engine's first-tier feature — chunks, meshing, lighting, bulk edits, and deterministic generation. Built as a module (`modules/voxel_core/`) that exposes a handle-based API through `VoxelServer` so scene code holds only `Handle`s.

## Components

- **Chunk** — fixed-size voxel grid (16³ or 32³ — see [`gaps.md` § 3](../gaps.md))
- **Sparse storage** — hash map of chunks; SVO only if profiling demands it
- **Meshing** — greedy or dual contouring, run on a compute shader where available
- **VoxelManipulator** — bulk-edit API for fast multi-voxel writes + lighting propagation (port of Luanti's `src/voxel.h:24`)
- **Lighting** — propagation, border updates, repair after edits (Luanti `src/voxelalgorithms.cpp`)
- **Generation** — deterministic from `(seed, voxel_atlas)`; produces baseline that saves layer deltas on top
- **Raycasting** — for editor brush + gameplay queries
- **`VoxelServer`** — opaque handle API; the only Vulkan-aware voxel code lives here

## Meshified static chunks — perf win for read-only regions

A chunk's **representation** (voxel form vs. static triangle mesh) is derived from the containing region's edit policy ([`specs/scene.md` § Edit policy](scene.md)). Read-only regions get **baked once to a static triangle mesh** at chunk-load time and never re-meshed. Editable regions stay in full voxel form.

Industry precedent: Vintage Story, Astroneer, No Man's Sky, and the announced Hytale architecture all use a "bake static chunks" pattern for mostly-static voxel worlds. Standard, not novel.

### Per-chunk representation, derived from region policy

The decision is per-chunk + dynamic:

| Region policy | Chunk representation |
| --- | --- |
| `destroy = full` OR `place = full` | **Voxel form.** Edits expected; meshify cost would be wasted |
| `destroy = none` + `place = none` | **Meshified static.** Fully baked at load; never re-meshed |
| `destroy = none` + `place = tag_allowlist:dropped_item` (story dungeon with items) | **Hybrid.** Terrain meshified; placed items tracked in a small per-chunk overlay layer |
| `destroy = voxel_type_allowlist` / `coord_range_allowlist` (partial edit) | **Voxel form.** Edits could come anywhere in the chunk; voxel form is needed |
| `ownership = purchasable` (plot pre-purchase) | **Meshified.** Flip to voxel on ownership acquisition (one-time ~5–10 ms cost; player doesn't notice) |

### Storage + perf savings

Per-chunk, when meshified:

| Cost | Voxel chunk | Meshified chunk |
| --- | --- | --- |
| Voxel data RAM | 16 KB (16³ × 4 bytes) | 0 (discarded after meshify) |
| Mesh RAM | ~5–15 KB | ~5–15 KB |
| **Total RAM** | **~21–31 KB** | **~5–15 KB (50%+ savings)** |
| Re-mesh cost on neighbor change | ~1 ms | never |
| Lighting recompute on edit | yes | never (baked into vertex colors) |
| Per-frame culling cost | similar | similar |

For Daggerfall-scale worlds where ~95% of chunks are static terrain or static interiors, the engine-wide saving is roughly **80–90% of the voxel meshing CPU budget** and ~50% of chunk RAM. Directly serves the 50–60 FPS on iGPU target in [`mission.md`](../mission.md).

### Hybrid: story-dungeon item overlay

For regions with `destroy = none + place = tag_allowlist`:

- Terrain stays meshified (huge win for the dungeon's walls/floors)
- Placed items are tracked in a **separate per-chunk overlay** (`PlacedItem { pos, item_id, owner }[]`) — tens of bytes per item
- Rendering: terrain mesh + overlay item meshes drawn separately
- The terrain itself stays static; only the overlay layer changes

This is what lets "no destroy" coexist with "can drop items here."

### Flip-back cost: meshified → voxel form

When a chunk transitions back to voxel form (player acquires a plot, a quest unlocks an area, a `script` policy returns "now editable"):

1. Discard the static mesh
2. Regenerate voxel data from `(seed, voxel_atlas)` — deterministic, ~1 ms
3. Apply any previously-stored deltas for this chunk
4. Generate a voxel mesh
5. Resume voxel-edit behavior for this chunk

Total ~5–10 ms one-time per chunk. Async on a worker thread; player doesn't notice. After the flip, the chunk stays in voxel form for the rest of the session (one-direction transition).

### Lighting in meshified chunks

Per-vertex colors bake the voxel lighting at meshify-time:

- Static sun direction → directional contribution baked
- Local light sources (torches) in the region → distance-attenuated contribution baked
- Dynamic light sources (player-held torch, spell projectiles) → handled by the per-frame dynamic-light pass (per [`specs/lighting.md`](lighting.md)) — render on top of baked colors

When the region's lighting changes significantly (time-of-day cycle), meshified chunks need to be re-baked. Two options:

- Re-bake at scheduled intervals (e.g. every 15 minutes of game time = 4 bakes per game-day) — simple
- Use per-vertex multi-channel lighting (separate channels for sun/ambient/local) and blend per-frame in the shader — more complex, but cheaper at runtime

Pick the simpler option for v1.0; revisit if profiling demands.

### Multiplayer + meshify

The mesh-vs-voxel representation is **client-side derived state**, not network-replicated. Each client computes its own representation from the region policy. Server doesn't care about render representation; it just broadcasts voxel deltas to applicable chunks (per [`specs/multiplayer.md`](multiplayer.md)).

On ownership transitions (plot purchase), the server broadcasts the new policy state; clients flip representation locally and consistently.

### Memory budget integration

Meshified chunks live in a separate pool tracked by [`gaps.md` § 2.2.H](../gaps.md) memory budget enforcement. Pool sizes:

- Voxel-form chunk pool: ~256 chunks × 21 KB = ~5.4 MB
- Mesh-form chunk pool: scales with the world's static region count; ~2,000 chunks × 12 KB = ~24 MB
- Total chunk-related memory ~30 MB on iGPU — well within the 3.5 GB budget

## Mining — hardness + tool-tier + break overlay

The engine implements a Minecraft-style mining model. Every voxel-editing target uses it (arena_modes, Stardew mines, Atelier gathering, Daggerfall mining zones, rogue_tower destructible rooms). Specified 2026-05-25; see project memory `project-voxel-hardness-mining` for the full rationale.

### Voxel data (TOML schema additions)

Each voxel type declares mining properties alongside its existing data:

```toml
[voxel.stone]
hardness = 1.5                  # base time-to-mine coefficient
tool_tier_required = 1          # 0=hand, 1=wood, 2=stone, 3=iron, 4=diamond/mythic
effective_tool_class = "pickaxe"
drops = [
  { item = "stone_block", chance = 1.0 }
]

[voxel.soil]
hardness = 0.5
tool_tier_required = 0          # hand suffices
effective_tool_class = "shovel"  # shovel gets a speed bonus, but hand still works
drops = [
  { item = "dirt_block", chance = 1.0 }
]
```

### Tool data (TOML schema additions)

Each tool item declares its mining characteristics in `<project>/assets/data/items/tools/*.toml`:

```toml
[item.wooden_pickaxe]
type = "tool"
tool_class = "pickaxe"
tier = 1
mining_speed_multiplier = 2.0
durability = 60
```

### Server-authoritative mining process

1. Client sends `beginMine(voxel)` intent to the server
2. Server validates: edit-policy passes, voxel is targetable, player has tool equipped
3. Server tracks `progress` per (player, voxel) pair:

   ```text
   progress += dt × (tool.mining_speed_multiplier × class_bonus) / voxel.hardness
   ```

   where `class_bonus = 1.5` (tunable) if `tool.tool_class == voxel.effective_tool_class`, else 1.0

4. When `progress >= 1.0`:
   - Voxel destroyed; bulk-edit + lighting + meshing path runs as for any other edit
   - If `tool.tier >= voxel.tool_tier_required`, drop the item into the player's inventory
   - Else: voxel destroyed silently, no drop
   - Tool durability decremented; tool breaks at 0
5. Client `cancelMine()` or player target change clears the per-pair state

### Replication

- Mining progress: per-(player, voxel) `break_stage` integer 0–9 (10-stage break overlay, Minecraft canon)
- Interest-managed — only clients with the chunk in view receive the stage updates
- Final destruction replicated via the standard voxel-edit channel; meshes re-bake on receipt
- Anti-cheat: clients never authoritatively advance progress — they predict + reconcile

### Composition with edit-policy

Edit-policy (binary: can edit at all?) is evaluated **before** mining. If the policy rejects the voxel, no progress is tracked. Cheap reject.

The policy compositor accepts:

- `coord_range_allowlist` — e.g. `y > ground_level` for arena_modes
- `tag_allowlist` — only voxels tagged `destructible`
- `voxel_type_allowlist` — only `ore` or `dirt` (Stardew mines)
- `script` — defer to callback (storm-boundary mutations, quest-gated rules)
- Composed (AND / OR) for complex per-scene rules

### Render: 10-stage break overlay

`backends/vulkan/` renders the standard Minecraft break-overlay quads on top of the targeted voxel face, indexed by `break_stage`. Asset: 10 textures shipping in `core_pack/assets/textures/break_stages/`.

## Open decisions (see `gaps.md`)

- Chunk size (16³ / 32³ / 64³) — affects memory, meshing batch, network packet size
- Voxel data layout (8/16/32-bit ID) — affects palette size + memory
- Origin rebasing strategy for 10 km × 10 km on X/Z plane (Y vertical) worlds
- Voxel atlas schema (TOML)
- **Meshified-chunk LOD interaction** — meshified chunks can have aggressive LOD baked at multiple distance tiers; voxel chunks generate LOD dynamically. How do these coexist when transitioning between tiers? See [`gaps.md`](../gaps.md) #46
- **Lighting re-bake cadence vs. per-vertex multi-channel** — pick one strategy for time-of-day handling on meshified chunks. See [`gaps.md`](../gaps.md) #47

## Reference patterns

- Luanti `src/voxel.h:24` — VoxelManipulator
- Luanti `src/mapblock.h` — chunk structure (their "MapBlock")
- Luanti `src/client/meshgen/` — meshing strategies
- Luanti `src/voxelalgorithms.cpp` — lighting
- Luanti `src/mapgen/` — procedural framework
