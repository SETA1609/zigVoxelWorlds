# `modules/rogue_tower/`

> Endless tower / dungeon-crawler progression layer. **Optional** — disabled by default; opted-in per project. Post-1.0.

## What it provides

- Procedural floor generator — voxel-chunk-shaped floors with parameterized layouts (room density, corridor topology, hazard set)
- Run state — current floor, accumulated loot, character build snapshot
- Boss / mini-boss gating — fixed-floor bosses; configurable per-project
- Run-summary export — for post-run displays + leaderboards

## Why a module, not a hard-coded feature

Same rationale as `modules/farming/` — engine-as-app. A roguelike project ships with `voxel_core` + `rogue_tower` + `combat` (TBD) + `inventory`; doesn't ship farming, dialog, or large overworld code.

## Layering

Depends on `core/`, `scene/`, `modules/voxel_core`, `modules/inventory`, `modules/skills`, `modules/perks`. MUST NOT import `editor/`.

## Spec

TBD (post-1.0). One of the four target archetypes per [`specs/gameplay.md`](../../docs/specs/gameplay.md).
