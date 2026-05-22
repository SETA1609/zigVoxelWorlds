# Core Types Spec

> Foundational data layouts every other system depends on: `Handle`, coordinate system, voxel data layout, chunk size. **These four decisions are tightly coupled** — each constrains the others. Closes [`gaps.md`](../gaps.md) #7, #8, #9, #10. **Phase-0-blocking.**

## Scope

Four locked-in value decisions plus their reasoning. Once landed, every server / module / save format / network protocol depends on these. Changing later = breaking the world. Settle now.

## `Handle` — opaque entity identifier

**Decision: u64 with `(generation: u32, index: u32)` layout.**

```zig
pub const Handle = packed struct(u64) {
    index: u32,           // index into the server's storage table
    generation: u32,      // bumped each time the slot is reused
};

pub const INVALID_HANDLE = Handle{ .index = 0xFFFFFFFF, .generation = 0xFFFFFFFF };
```

### Why this layout

- **`u32` index = 4 billion entities per server**, way past what any target game needs (Daggerfall-scale ~50k entities)
- **`u32` generation = 4 billion reuses per slot** before wraparound; effectively forever
- **`u64` total fits in a single register**, copies are free, atomic loads work
- Validating a handle = compare generation against the storage table's current value at that index → O(1) and lock-free
- Borrowed pattern from Bevy ECS, Unity ECS, EnTT — proven design

### Rejected alternatives

- `(server_id: u8, generation: u24, index: u32)` — server-id-in-handle complicates routing. Cleaner to make Handles per-server and require explicit server reference for cross-server lookups.
- `u128` — overkill; u64 is more than enough and fits SIMD lanes better.
- Pointer + tag — fragile across save/load and across mod boundaries; defeats the opaque-handle discipline.

### Usage rules

- **One Handle space per server** (not global). `RenderServer.MeshHandle ≠ VoxelServer.ChunkHandle ≠ PhysicsServer.BodyHandle`. Type aliases distinguish them at compile time.
- **Handles are stable across saves**. Save format stores raw `u64` per entity; on load, the engine validates generation and either rebinds or flags as dead.
- **Handles cross the C ABI as opaque `uint64_t`**. Mods + scripts use them but never inspect the layout.

## Coordinate system

**Decision: Y-up, right-handed, units in meters, 1 voxel = 1 meter.**

```text
       +Y (up)
        |
        |
        +-----+X (right / east)
       /
      /
     +Z (forward / toward viewer / south)
```

### Why these choices

- **Y-up**: matches Vulkan default + glTF 2.0 + most contemporary game engines (Unity, Unreal, Bevy). Z-up is more common in CAD/sim/Godot, but Y-up is more common in shipped games, asset stores, and gltf-2.0 imports. Lower friction for asset pipeline.
- **Right-handed**: Vulkan's clip-space convention after the standard projection matrix. Z+ goes *toward* the viewer. (Note: some engines flip this; we don't.)
- **1 voxel = 1 meter**: matches human-scale buildings + reasonable physics. A player is ~1.8 voxels tall. This matches Minecraft + Vintage Story conventions; players will instinctively know the scale.
- **Meters as unit**: physics (Jolt) defaults to meters; matches real-world physics constants without unit conversion.

### Camera + projection

- Camera looks down `-Z` by default (right-handed convention)
- FoV measured vertically (per [`specs/camera.md`](camera.md))
- Near plane: 0.05 m (5 cm — close enough to see your hands in first-person)
- Far plane: depends on view distance; default ~1 km

### Origin rebasing

Float32 world coordinates lose precision past ~1 km from origin. Two strategies:

1. **Origin rebasing** (deferred to Phase 6): when the player crosses a threshold (e.g. 512 m from origin), shift the world's origin so the player is back near 0. Re-encode all chunk positions. Complex but uses cheap float32 everywhere.
2. **Float64 throughout**: simpler but doubles memory + slower SIMD.

Decision: **start with float32 + origin rebasing**. See [`gaps.md`](../gaps.md) #11 for the rebasing trigger threshold (open, before Phase 6).

## Voxel data layout

**Decision: 16-bit voxel ID per voxel + per-chunk palette + separate light/state channels.**

### Per-voxel struct

```zig
pub const Voxel = packed struct(u32) {
    id: u16,           // index into the chunk's palette
    light: u8,         // packed nibbles: high = sky light, low = block light (0-15 each)
    state: u8,         // game-specific state bits (e.g. rotation, growth stage, damage)
};
```

Total per voxel: **4 bytes**.

### Chunk palette

Each chunk has a **local palette** mapping `u16 id → u32 global voxel-type-id`:

```zig
pub const Chunk = struct {
    voxels: [16 * 16 * 16]Voxel,           // 16³ voxels (see § Chunk size)
    palette: []u32,                         // local id (u16) → global id (u32)
    light_dirty: bool,
    mesh_dirty: bool,
};
```

### Why this layout

- **16-bit local id = 65k voxel types per chunk** — way past any sane game's variety. (Daggerfall's content scale projects to ~500-2000 unique voxel types max.)
- **Per-chunk palette** — chunks with low variety (a uniform grass field, all id=1) compress to 1 palette entry + RLE of voxel slots. Network-friendly.
- **Global voxel-type-id = u32** — supports 4 billion total voxel types across the game (mods can register their own without collision).
- **Light packed into 8 bits** — 4 bits sky + 4 bits block matches Minecraft / Luanti precedent. Visually sufficient.
- **8 bits state** — enough for rotation (3 bits), variation (2 bits), damage (3 bits) or any per-game scheme.

### Voxel type registry (global, not in the chunk)

```zig
pub const VoxelType = struct {
    id: u32,
    name: []const u8,           // "stone", "iron_ore", "torch_lit"
    flags: VoxelTypeFlags,      // solid, transparent, light-emitter, climbable, ...
    light_emission: u4,         // 0-15; if non-zero, this voxel emits light
    texture_atlas_indices: [6]u16,  // per face (top, bottom, sides×4)
    sound_family: SoundFamily,  // for footstep / break / place SFX
    hardness: f32,              // for mining + physics
};
```

This registry is global, loaded from CSV at startup (per [`specs/content-authoring.md`](content-authoring.md)).

### Why local palette + global registry

- Network packet size: chunks with low variety pack tightly via local palette delta encoding
- Mod compat: a mod adds new global IDs without renumbering existing IDs
- Storage: per-voxel `u16` palette index is 2 bytes vs. raw `u32` global id = 4 bytes; saves 50% of voxel storage for non-pathological chunks

## Chunk size

**Decision: 16³ voxels per chunk (4096 voxels).**

### Numbers

| Metric | 16³ | 32³ | 64³ |
| --- | --- | --- | --- |
| Voxels per chunk | 4,096 | 32,768 | 262,144 |
| Storage per chunk (4 bytes/voxel) | 16 KB | 128 KB | 1 MB |
| Network packet (uncompressed) | 16 KB | 128 KB | 1 MB |
| Network packet (RLE + palette) | typically 1-4 KB | 8-32 KB | 64-256 KB |
| Mesh-gen cost per chunk | Low | Medium | High (>1 frame on iGPU) |
| Streaming granularity (8-chunk view distance) | 4 × 8 × 8 × 8 = 2048 chunks visible | 128 chunks | 32 chunks |

### Why 16³

- **Mesh-gen budget**: greedy meshing 4096 voxels = fast enough to do many chunks per frame on iGPU; meshing 32³ blocks the frame
- **Network friendliness**: 1-4 KB compressed chunks ship in single UDP packets at 1500-byte MTU after fragmentation
- **Memory granularity**: 16 KB per chunk × 2048 visible chunks = 32 MB voxel data — within the < 3.5 GB RAM budget
- **Streaming**: more, smaller chunks = finer-grain load/unload as the player moves
- **Industry precedent**: Minecraft uses 16×256×16 (sliced columns of 16³); Luanti uses 16³; Vintage Story uses 32³ but has more aggressive optimization (and targets higher-end hardware)

### Tradeoffs accepted

- More chunks = more pointer chasing in the chunk hash map. Mitigated by spatial locality: neighbor chunks are usually allocated together.
- Mesh boundary seams require neighbor-chunk awareness during meshing. Add a 1-voxel padding "border" or query neighbors on the fly.

### Vertical chunks

A "chunk" is **cubic** (16³), not a vertical column. Worlds extend in all three axes equally. World height is bounded by the chunk-coord type:

- `ChunkCoord = packed struct { x: i24, y: i24, z: i24 }` = ±8M chunks per axis = ±128M voxels per axis = ±128 km world extent
- Practical: the 10 km² goal is 80 × 80 chunks on the X/Z plane, plus ~10 chunks vertically (160m height) — trivially in scope

## Cross-system implications

These four decisions cascade:

| Decision | Affects |
| --- | --- |
| `Handle` u64 layout | C ABI surface (#13), save format (#6), network protocol (#25), every server API |
| Coordinate Y-up RH meters | Camera math, physics integration, mesh import (glTF Y-up matches), shader space conventions |
| Voxel 16-bit ID + palette | Save format chunk deltas, network packet layout, voxel atlas TOML, mod ID registration |
| Chunk size 16³ | Mesh-gen worker thread budget, streaming distance UI, memory budget calculations, network MTU |

Any future change to these is a **breaking change for shipped games + saves + mods**. Treat as permanent post-v1.0.

## Closes

- [`gaps.md`](../gaps.md) #7 — Handle layout
- [`gaps.md`](../gaps.md) #8 — Coordinate system
- [`gaps.md`](../gaps.md) #9 — Voxel data layout
- [`gaps.md`](../gaps.md) #10 — Chunk size

Touches but does not close (referenced by other open items):

- #6 (save binary format) — voxel layout + chunk size drive the on-disk chunk-delta layout
- #11 (origin rebasing strategy) — depends on float32-coordinate decision; threshold still open
- #13 (C ABI surface) — Handle u64 layout is a key ABI element
