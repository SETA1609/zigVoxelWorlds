# `modules/skills/`

> Classless skill progression — **Fallout 1/2/NV-style: point-buy on level-up, not use-leveling.** Phase 8.

## Progression model

- XP is granted by the **quest/event bus** — quests completed, kills, exploration milestones, achievements. Repeated use of a skill does **NOT** train it.
- On level-up the player receives a pool of **skill points** to spend on individual skills.
- **Classic Fallout 1/2 — tagged skills are discounted.** 3 tagged skills chosen at character creation. Spending 1 skill point on a tagged skill grants **+2 skill value** (below the 100 cap); spending 1 point on an untagged skill grants **+1**. Same point pool, double yield on tagged → effectively half the cost.
- NOT New Vegas's +15-at-start model. NOT F2's above-100 cost escalation in v1 (per-project TOML extension only).

This module **does not** subscribe to "perform action → grant XP" handlers. XP flows in via `core/` event bus listeners on quest / kill / milestone events only. Treat any `voxel_mine_block → mining XP` shaped code as a learning-mode flag.

## What it provides

- Skill registry (data-driven via `<project>/assets/data/skills/*.toml`)
- Per-entity skill table (ECS component): current value (1–100), tagged set, available skill points
- Level-up resolver: XP → level, level → skill-point grant
- Skill-point spend API (validates point pool, applies bonus for tagged skills)
- Skill checks: percentile-vs-target rolls

## No verbatim ports

Fallout is the *gameplay shape* reference, not a source file we read. The 1/2/NV games are proprietary; their code is not in `$REFS/`. Mechanics are reimplemented from public design knowledge — see [`engine-references.md` § Legal](../../docs/engine-references.md).

## Target-game divergence

Daggerfall (a target game) uses use-leveling. The first-party module is Fallout-style; a Daggerfall-mod project replaces the XP source via TOML config or ships its own `modules/skills_daggerfall/`. The engine does not implement use-leveling.

## Layering

Depends on `core/` (event bus), `scene/ecs`. MUST NOT import `editor/`. The skill editor panel lives at `src/editor/panels/skill_editor.zig` (cross-module editor; not under `modules/skills/editor/`).

## Spec

[`specs/gameplay.md`](../../docs/specs/gameplay.md) § Skills.
