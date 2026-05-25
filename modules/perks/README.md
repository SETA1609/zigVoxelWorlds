# `modules/perks/`

> Perk trees — discrete unlocks gated by skill thresholds. Complements `modules/skills/`. Phase 8.

## What it provides

- Perk registry (data-driven via `<project>/assets/data/perks/*.toml`)
- Tree topology (perks with prerequisite perks; prerequisite skill levels)
- Per-entity owned-perks bitset (component on the ECS entity)
- Unlock validation + activation hooks

## Layering

Depends on `core/`, `scene/ecs`, `modules/skills` (for level gating). MUST NOT import `editor/`.

## Spec

[`specs/gameplay.md`](../../docs/specs/gameplay.md) — perk section TBD. The data schema follows the same TOML pattern as skills, recipes, spells (see [`specs/data-schemas.md`](../../docs/specs/data-schemas.md)).
