# `modules/inventory/`

> Item containers, stacks, slots, equipment, encumbrance. Phase 8.

## What it provides

- Item registry — data-driven via `<project>/assets/data/items/*.toml`
- Container component (ECS) — variable-sized inventory grid with slot constraints
- Equipment slots (helm, chest, weapon hand, …) — per-game configurable
- Stacking rules (max stack per item, partial stacks)
- Encumbrance — weight / volume budget; affects movement speed
- Quickbar binding — stable item references for the UI quickbar (see [`specs/ui.md` § Quickbar](../../docs/specs/ui.md))
- **Tool stats** for the voxel mining model (per project memory `project-voxel-hardness-mining`): each tool item declares `mining_speed_multiplier`, `tool_class` (pickaxe/axe/shovel/scythe/hand), `tier` (0=hand, 1=wood, …, 4=diamond), `durability`. The voxel server reads these when evaluating mining progress.

## Slot binding rules

Quickbar / hotbar slots store a **stable item reference** (per-stack identity, not a per-item handle that breaks when count changes). If the bound item is fully consumed → slot empties silently. If moved between containers → slot follows (still bound, by reference). See [`specs/ui.md` § Slot binding rules](../../docs/specs/ui.md).

## Layering

Depends on `core/`, `scene/ecs`. Used by `modules/crafting` (consumes ingredients, produces outputs), `modules/skills` (carry-weight gated by `endurance` if exposed), UI (quickbar bindings).

## Spec

[`specs/gameplay.md`](../../docs/specs/gameplay.md) § Inventory.
