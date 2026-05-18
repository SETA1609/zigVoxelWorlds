# Voxel System Spec

> What `modules/voxel_core/` provides. The roadmap entry is [`ROADMAP.md` § Phase 3](../ROADMAP.md). Reference patterns: [`engine-references.md` → Luanti](../engine-references.md). Open questions: [`planning-gaps.md`](../planning-gaps.md) §1 #5, §2 #9, #10, #11.

## Scope

The voxel core is the engine's first-tier feature — chunks, meshing, lighting, bulk edits, and deterministic generation. Built as a module (`modules/voxel_core/`) that exposes a handle-based API through `VoxelServer` so scene code holds only `Handle`s.

## Components

- **Chunk** — fixed-size voxel grid (16³ or 32³ — see [`planning-gaps.md` #10](../planning-gaps.md))
- **Sparse storage** — hash map of chunks; SVO only if profiling demands it
- **Meshing** — greedy or dual contouring, run on a compute shader where available
- **VoxelManipulator** — bulk-edit API for fast multi-voxel writes + lighting propagation (port of Luanti's `src/voxel.h:24`)
- **Lighting** — propagation, border updates, repair after edits (Luanti `src/voxelalgorithms.cpp`)
- **Generation** — deterministic from `(seed, voxel_atlas)`; produces baseline that saves layer deltas on top
- **Raycasting** — for editor brush + gameplay queries
- **`VoxelServer`** — opaque handle API; the only Vulkan-aware voxel code lives here

## Open decisions (see `planning-gaps.md`)

- Chunk size (16³ / 32³ / 64³) — affects memory, meshing batch, network packet size
- Voxel data layout (8/16/32-bit ID) — affects palette size + memory
- Origin rebasing strategy for 10 km² worlds
- Voxel atlas schema (TOML)

## Reference patterns

- Luanti `src/voxel.h:24` — VoxelManipulator
- Luanti `src/mapblock.h` — chunk structure (their "MapBlock")
- Luanti `src/client/meshgen/` — meshing strategies
- Luanti `src/voxelalgorithms.cpp` — lighting
- Luanti `src/mapgen/` — procedural framework
