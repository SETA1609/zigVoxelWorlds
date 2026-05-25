# `modules/magic/`

> Spell effects + costs + cooldowns + **9 color schools** (Roy G. Biv + White + Black). Phase 8.

## Color schools (function-parallel to Morrowind/Daggerfall, color-named)

| Color | What it does (parallel) |
| --- | --- |
| **Red** | Destruction — offensive damage (fire/frost/shock/poison), drains |
| **Orange** | Conjuration — summoning creatures, bound weapons, energy creation |
| **Yellow** | Illusion — light, invisibility, charm, calm |
| **Green** | Restoration (mortal) — healing, buffs, cures |
| **Blue** | Alteration — water-walking, feather, lock magic, transmutation |
| **Indigo** | Thaumaturgy — levitation, force manipulation |
| **Violet** | Mysticism — soul-trap (detect/store), telekinesis, mark/recall, dispel |
| **White** | Holy / Divine — protection auras, banish undead/demons, divine-grade healing, sanctify. Opposed by Black. |
| **Black** | Necromancy / Death — raise undead, life-drain, decay, curses, soul-corruption. Opposed by White; **weaponizes** what Violet only detects/stores. |

Engine vocabulary uses color names (`red_magic`, `orange_magic`, …). ES parallels are commentary only; **no `destruction` / `conjuration` / etc. anywhere in code or TOML keys**. See [`project-magic-rainbow-schools` memory](#) and [`specs/gameplay.md`](../../docs/specs/gameplay.md) § Magic.

## What it provides

- Effect registry — atomic effects (`burn`, `restore_health`, `summon_skeleton`, `teleport`, `banish_undead`, `raise_dead`, …) defined in `<project>/assets/data/spells/effects/<color>/*.toml` (`<color>` ∈ `red|orange|yellow|green|blue|indigo|violet|white|black`)
- Spell composition — a spell is a composition of one or more effects with magnitude / duration / target shape
- Color-school affinity — each spell tagged with a primary color; the player's `<color>_magic` skill scales cost / power / chance
- Cooldowns + costs (mana, stamina, reagents)
- Cast hooks dispatched onto the event bus
- UI palette — each color drives a card tint + particle palette via RCSS variables in [`src/ui/`](../../src/ui/README.md)

## Interaction with `modules/skills/`

Each color is a Fallout-style skill (1–100) in `modules/skills/`. The skill value is **raised by spending skill points on level-up** (see [`modules/skills/`](../skills/README.md)). Casting a spell does **NOT** grant XP to its color skill. XP enters via quest / event bus only.

## Module-specific editor sub-tree

`modules/magic/editor/` (scaffolded) is the example case for module-owned editor panels. Spell composition is data-heavy and benefits from a dedicated editor more than a cross-module panel in `src/editor/panels/`.

## Layering

Depends on `core/`, `scene/ecs`, `modules/skills` (for color-skill values), `servers/audio_server` (cast SFX). MUST NOT import `editor/` or `editor/`-only subtrees outside its own `magic/editor/`.

## Spec

[`specs/gameplay.md`](../../docs/specs/gameplay.md) § Magic.
