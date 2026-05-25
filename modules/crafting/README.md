# `modules/crafting/`

> Recipe resolver + station gating + multi-stage synthesis. Atelier-influenced. Phase 8.

## What it provides

- Recipe registry — data-driven via `<project>/assets/data/recipes/*.toml`
- Station gating — a recipe declares which station tier is required (workbench, alchemy lab, forge, …)
- Multi-stage synthesis — Atelier-style: gather → process → synthesize, each stage gated by skill + station
- Quality resolution — output quality derived from `f(skill_level, ingredient_qualities, station_tier, rng_seed)`; exact formula per [`gaps.md` § 3](../../docs/gaps.md)
- Recipe discovery — locked recipes revealed by gameplay events (NPC teach, find scroll, experiment with ingredients) — dispatched via the event bus

## Interaction with `modules/skills/`

Crafting skills (`smithing`, `alchemy`, `cooking`, `enchanting`, …) are Fallout-style skills in `modules/skills/`. Skill values gate recipe availability and feed into the quality formula. Crafting actions do **NOT** grant skill XP. XP enters via quests / discovery events / etc.

## Layering

Depends on `core/`, `scene/ecs`, `modules/skills`, `modules/inventory` (consumes ingredients, produces output items). MUST NOT import `editor/` (recipe editor panel lives at `src/editor/panels/recipe_editor.zig`).

## Spec

[`specs/gameplay.md`](../../docs/specs/gameplay.md) § Crafting.
