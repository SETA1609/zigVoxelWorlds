# Gameplay Systems Spec

> Skills, perks, magic, crafting, inventory — each its own module under `modules/`. Roadmap: [`ROADMAP.md` § Phase 8](../ROADMAP.md). All data-driven via TOML per [`tech-stack.md` § Data Layer](../tech-stack.md). Open data-model questions: [`gaps.md`](../gaps.md) #18–22.

## Skills (`modules/skills/`)

Morrowind / Fallout-style classless progression:

- Per-skill XP accumulation from use
- Level-ups derived from skill milestones (not class)
- Skill values clamp at species/perk caps
- Skill checks via roll-vs-target

Data: skill definitions in TOML (`<project>/assets/data/skills/*.toml`).

## Perks (`modules/perks/`)

Fallout-style perk / Special system:

- Perk trees or free-form perks (game-author choice via TOML)
- Each perk: prerequisites (skill levels, other perks, attributes), effect descriptor
- Effects compose into the active modifier set on a character

Data: perk definitions in TOML; visual tree layout authored in the editor.

## Magic (`modules/magic/`)

Morrowind-style spellmaking + Daggerfall-style schools:

- **Effects as data** — each spell effect is a TOML-defined building block (`damage_fire`, `restore_health`, `teleport`, etc.) with magnitude / duration / target shape
- **Spellmaking** — combine effects into a custom spell; cost derived from sum
- **Schools** — taxonomy of effects; affinity gated by player's school skill
- **Dynamic application** — effects can apply to entities and to the world (e.g. light areas, transmute voxels — see [editability policy](../ARCHITECTURE.md#cross-cutting-concerns))

Data: effects + schools in TOML; player-made spells stored in save.

## Crafting (`modules/crafting/`)

Atelier-style multi-stage synthesis:

- **Stages** — gather → process → synthesize, each gated by skill + station
- **Quality** — derived from `f(skill_level, ingredient_qualities, station_tier, rng_seed)`; exact formula in [`gaps.md` § 3](../gaps.md)
- **Recipe discovery** — locked recipes revealed by gameplay events (NPC teach, find scroll, experiment with ingredients)
- **Synthesis UI** — Atelier-style ingredient placement + slot effects

Data: recipes in TOML; stations defined as entity types.

## Inventory (`modules/inventory/`)

The quickbar (10 slots, universal across all four target games) is owned by the UI layer + lives in [`specs/ui.md`](../specs/ui.md) § Quickbar. Inventory provides the underlying item references; the quickbar slots are stable refs to inventory items.

- Slot count, weight, stack rules, container hierarchy
- Equipped vs carried (different slot sets per body type)
- Hot-bar / quick-use mapping
- NPC merchant inventories with regen schedules
- Saved per-entity in the structured-state save section ([ARCHITECTURE save model](../ARCHITECTURE.md#cross-cutting-concerns))

## Cross-cutting

- All five systems hook into the [editability policy](../ARCHITECTURE.md#cross-cutting-concerns) when actions modify the world (mining = inventory + voxel edit; spell = magic + voxel edit; crafting station = scene-fixed)
- All five expose mod-replaceable extensions via the [stable C ABI](../engine-vs-game.md)
- Quest system ([`gaps.md` § 3](../gaps.md)) consumes events from all five
