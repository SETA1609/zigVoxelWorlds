# `modules/voxel_core/`

> The engine's core voxel subsystem. Chunk lifecycle, meshing, streaming, lighting, bulk-edit API. Phase 3.

## Subsystems

- **Chunk store** — RAM-resident records of voxel blocks (`MapBlock` analog from Luanti `src/mapblock.h`)
- **Meshing** — greedy meshing + compute-shader paths; produces draw-ready mesh upload for the Vulkan backend
- **Streaming** — load/unload chunks based on player position + interest radius
- **Lighting** — propagation across chunk borders, sun + block lights
- **Bulk-edit API** — multi-voxel writes that defer lighting + meshing until the batch closes (`VoxelManipulator` pattern from Luanti `src/voxel.h:24`)
- **Hardness + mining** — Minecraft-style hardness + tool-tier + per-(player,voxel) mining-progress tracking + 10-stage break overlay. Server-authoritative in PvP. Used by every voxel-editing target. See project memory `project-voxel-hardness-mining` + [`specs/voxel.md`](../../docs/specs/voxel.md) § Mining.
- **Edit-policy compositor** — evaluates scene/region policies (`coord_range_allowlist`, `tag_allowlist`, `script`, …) before any mutation; cheap binary reject before the hardness mechanic runs

## Reference patterns (Luanti — `engine-references.md` § Luanti)

- `src/voxel.h:24` + `src/voxel.cpp` — `VoxelManipulator`
- `src/voxelalgorithms.h/cpp` — lighting propagation, `VoxelLineIterator`, `update_block_border_lighting`
- `src/mapblock.h` — chunk structure
- `src/client/meshgen/` — meshing strategies
- `src/mapgen/` — biomes, noise (seed-deterministic regeneration per [save-model](../../docs/specs/save-ux.md))

Adaptation rule per [`engine-references.md` § Legal](../../docs/engine-references.md): read the file, close it, write Zig that solves the same problem with our own data layout. **No verbatim translation.**

## Spec

[`specs/voxel.md`](../../docs/specs/voxel.md).

## Layering

Depends on `core/` + `servers/voxel_server` + `servers/render_server` (for mesh upload). MUST NOT import `editor/`. A module-specific `editor/` sub-tree can land here later for voxel-brush-related importers (not yet present).
