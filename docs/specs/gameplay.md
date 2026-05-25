# Gameplay Systems Spec

> Skills, perks, magic, crafting, inventory — each its own module under `modules/`. Roadmap: [`ROADMAP.md` § Phase 8](../ROADMAP.md). All data-driven via TOML per [`tech-stack.md` § Data Layer](../tech-stack.md). Open data-model questions: [`gaps.md`](../gaps.md) #18–22.

## Skills (`modules/skills/`)

Fallout 1/2/NV-style classless progression — **point-buy on level-up, not use-leveling**:

- **XP is event-driven**, not action-driven. Quests, combat kills, exploration milestones, and other event-bus signals grant XP. Repeatedly performing a skill action does **not** train it.
- XP accumulates against the player's overall level. On level-up the player receives a pool of **skill points** to spend on individual skills (the Fallout "skills get to be purchased" model).
- **Tagged skills** — 3 chosen at character creation — are **discounted** per the classic Fallout 1/2 rule: spending 1 skill point on a tagged skill grants **+2 skill value** (below the 100 cap); spending 1 point on an untagged skill grants **+1**. Same point pool, double yield → half the effective cost.
- Above 100 (if the per-game cap allows): cost-escalation is **not** modeled in v1 (NV's flat / F2's escalating rules are per-project additions via TOML, not first-party).
- NOT the New Vegas +15-at-start model.
- Skill values clamp at species/perk caps (Fallout convention: 1–100 internal scale).
- Skill checks via roll-vs-target — same as Fallout's percentile-vs-target rolls.

**Not Morrowind/Skyrim use-leveling.** Earlier wording in this spec called the model "use-leveling"; that wording was an error and is superseded by this section. Per [project memory](#) `project-skill-progression-fallout-style` (2026-05-25).

Data: skill definitions in TOML (`<project>/assets/data/skills/*.toml`). Per-character runtime state (current value + tagged set + points pool) is ECS-component data, persisted in the save delta.

### Target-game divergence

Daggerfall is one of the four target games and famously uses use-leveling. The first-party `modules/skills/` exposes the Fallout-style API; a Daggerfall-mod project replaces the XP grant source via TOML config (or by ships its own `modules/skills_daggerfall/`). The engine module does not implement use-leveling.

## Perks (`modules/perks/`)

Fallout-style perk / Special system:

- Perk trees or free-form perks (game-author choice via TOML)
- Each perk: prerequisites (skill levels, other perks, attributes), effect descriptor
- Effects compose into the active modifier set on a character

Data: perk definitions in TOML; visual tree layout authored in the editor.

## Magic (`modules/magic/`)

Morrowind-style spellmaking + **9 color schools** — Roy G. Biv plus White and Black — that parallel the Morrowind/Daggerfall school taxonomy in *function* but use color-coded *names*, avoiding Elder Scrolls naming entirely. See [`project-magic-rainbow-schools` memory](#) for the full mapping:

| Color school | Parallel ES function |
| --- | --- |
| Red | Destruction (offensive damage, drains) |
| Orange | Conjuration (summoning, bound weapons) |
| Yellow | Illusion (light, invisibility, charm) |
| Green | Restoration — mortal (healing, buffs, cures) |
| Blue | Alteration (transmutation, physical change) |
| Indigo | Thaumaturgy (force manipulation, levitation) |
| Violet | Mysticism (soul detection/storage, mind, esoteric) |
| **White** | Holy / Divine (protection auras, banish undead/demons, divine-grade healing, sanctify) — opposed by Black |
| **Black** | Necromancy / Death (raise undead, life-drain, curses, soul-corruption) — opposed by White; weaponizes what Violet only detects/stores |

Engine APIs and TOML keys use color names only (`red_magic`, `orange_magic`, `white_magic`, `black_magic`). The ES parallels above are design-intent commentary, not engine vocabulary.

- **Effects as data** — each spell effect is a TOML-defined building block (`damage_fire`, `restore_health`, `teleport`, etc.) with magnitude / duration / target shape
- **Spellmaking** — combine effects into a custom spell; cost derived from sum
- **Schools** — 9 color schools (table above: Roy G. Biv + White + Black); affinity gated by the player's color-magic skill (`red_magic`, `orange_magic`, …, `white_magic`, `black_magic`) which is a Fallout-style skill in `modules/skills/`
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
