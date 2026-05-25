# `modules/farming/`

> Stardew-style farming layer. **Optional** — disabled by default; opted-in per project via `project.toml`. Post-1.0.

## What it provides

- Soil tile state machine — tilled / watered / planted / growing / ready
- Crop registry — data-driven via `<project>/assets/data/crops/*.toml`; growth stages, season gating, yield
- Day/week/season clock — drives growth ticks; interacts with the world calendar (TBD spec)
- Tool gating — hoe / watering can / scythe via `modules/inventory`

## Why a module, not a hard-coded feature

zVoxRealms is engine-as-app: a project that doesn't want Stardew-style farming sets `farming = false` in `project.toml` and `build.zig`'s tree-shaker elides this module from the per-project `libzvox-runtime`. A project that wants only farming (no combat, no magic) ships with just `voxel_core` + `farming` + `inventory` + `modding`.

## Layering

Depends on `core/`, `scene/ecs`, `modules/inventory`, `modules/voxel_core` (for soil-tile voxel type). MUST NOT import `editor/`.

## Spec

TBD (post-1.0). Per [`specs/gameplay.md`](../../docs/specs/gameplay.md) § Target games, Stardew Valley is one of the four target archetypes.
