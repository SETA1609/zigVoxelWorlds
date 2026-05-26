# `modules/arena_modes/`

> Session-based arena game modes — Megabonk-inspired *combat pacing* (fast, vertical, dodgy) with Hunger Games last-standing structure on top. **All loot, progression, magic, and skills are reused from the other target games — no arena-only systems.** Optional module; opted-in per project. Phase 10+.

## Design stance

This is **not** a Vampire-Survivors / Megabonk clone with abstract stacking modifiers. It is a **Diablo-style action-RPG match wrapped in a Hunger Games shell**, sharing every gameplay system with the other target games.

The Megabonk inspiration lives in:

- Combat *pacing* — fast melee + ranged + spell exchanges, vertical movement (jumping / gliding TBD)
- Wave pressure (PvE mode) — scaling-difficulty hordes you survive against
- Match length — 20–30 minutes, not multi-hour

The Megabonk inspiration does **NOT** live in:

- Power-up trees / stacking modifiers — there are none
- Build-synergy matrix — there is none
- Match-only upgrade economy — there is none

## What it provides

- **Game mode framework** — per-match mode toggle:
  - **PvE co-op** — squad vs hordes; configurable squad size
  - **PvP free-for-all / teams** — last-standing win condition; configurable team count
- **Wave spawner** (PvE only) — scaling-difficulty mob waves, configurable density curves, boss intervals
- **Shrinking play area** (Hunger Games / battle-royale staple) — voxel-marked "storm" boundary that closes over time, dealing damage outside
- **Last-N-standing win condition** + match-end summary
- **Configurable max-players per match** — driven by `project.toml` `[multiplayer] max_players` (per project memory `project-target-games-and-save-model`)
- **Match → XP grant** — match-end, kills, and survival milestones dispatched onto the event bus; `modules/skills/` (Fallout-style XP) listens
- **Limited voxel editing — Minecraft-HG style with hardness:**
  - Hands destroy: soil, sand, trees, grass (low hardness, no tool-tier required)
  - Pickaxe destroys: tagged-destructible stones (with speed bonus per the `effective_tool_class` system)
  - All destruction uses the engine's hardness + tool-tier model in `modules/voxel_core/` — see project memory `project-voxel-hardness-mining`
  - 10-stage break overlay (Minecraft pattern) renders during mining
  - **Y-axis restricted:** only voxels at `y > ground_level` are editable. Arena floor (`y ≤ ground_level`) is read-only — **no mining downward**.
  - Players can also **place blocks** for cover / structures (destroyed blocks return to inventory)
  - Implemented via the engine's edit-policy compositor: `coord_range_allowlist(y > ground)` + `tag_allowlist(destructible)` + `script(storm_boundary)` evaluated per voxel mutation, ahead of the hardness check
  - **Server-authoritative** in PvP mode — clients cannot fake destruction, placement, or mining progress

## What it does NOT provide (delegated to other modules)

| Concern | Lives in |
| --- | --- |
| Item drops (swords, staves, potions, scrolls) | `modules/inventory/` — the same registry that Daggerfall uses |
| Spell knowledge from magic books | `modules/magic/` — the 9 color schools |
| Skill XP + level-ups + skill-point spend | `modules/skills/` — Fallout-style point-buy |
| Perk unlocks | `modules/perks/` |
| Server-authoritative validation (PvP) | `modules/multiplayer/` |
| Voxel arena geometry + storm boundary mutations | `modules/voxel_core/` |
| Particle / projectile rendering | `backends/vulkan/` (instanced particles TBD) |
| HUD widgets (health bar, hotbar, match-over modal, kill feed) | `src/ui/widgets/` — pre-built, SDL3-backed, ships in every project |

**arena_modes is the small-footprint reference configuration.** Per project memory `project-subsystem-swap-pattern`:

- `[audio] backend = "sdl3"` — SDL3 audio is enough for arena combat; no miniaudio dep
- `[ui] document = false` — widgets only; no RmlUi adapter linked

This keeps the v1.0 shipping target lean — the smallest binary surface across the five target games. Daggerfall et al. opt into miniaudio + RmlUi.

The arena mode is **thin** because everything substantive is reused. The module just orchestrates the match.

## What gets dropped in an arena match

Same item registry as Daggerfall. Examples:

- **Enchanted weapons** (sword, bow, axe — equipped, weighted, can break)
- **Magic staves** (fireball staff, etc. — consume the wielder's `red_magic` or other color-magic skill)
- **Tools** (pickaxe for tagged-destructible stones; hands suffice for trees)
- **Potions** — timed buffs (haste, fire resistance, healing-over-time)
- **Magic scrolls** — one-shot pre-cast spells; consumed on use
- **Magic books** — learn a spell permanently (carries over to the player's character, even after the match if persistence is on)
- **Placed blocks** — destroyed blocks become placeable inventory items (Minecraft-HG cover building)

## Honest design trade-off

By reusing the RPG item + progression model instead of an abstract power-up tree, this target gives up the "exponential build curve" endgame feel that defines real Megabonk (you end actual Megabonk matches as a particle storm with 17 stacked multiplicative damage modifiers). The result will play more like **Diablo + Hunger Games** than a true Megabonk clone. The reuse win across all five targets (one item registry, one skill system, one magic system) is the deliberate priority — see project memory `project-target-games-and-save-model`.

## Engine challenges this target stresses

1. **Server-authoritative PvP** — first target that exercises validated movement + combat. Pushes work into `modules/multiplayer/` that the co-op targets don't need.
2. **High player count in FFA** — breaks the previous "40–50p dedicated" assumption. Memory now: configurable, no hard cap; Phase 10 benchmarking sets realistic ceilings.
3. **Wave-density entity load** — hundreds of low-AI mobs per tick (PvE). Forces the archetype ECS in `scene/ecs/` to handle wide queries efficiently. Less extreme than real Megabonk (no projectile soup) but still meaningfully more than Daggerfall.
4. **Voxel arena geometry with `edit_policy = "none"`** — the storm boundary is the only voxel-mutating actor, and only the server can fire it.

## Relationship to `modules/rogue_tower/`

| | `modules/rogue_tower/` | `modules/arena_modes/` |
| --- | --- | --- |
| Subgenre | Traditional roguelike (NetHack / ToME lineage) | Action-RPG with HG structure |
| Map style | Procedural floors with rooms + corridors | Procedural arenas with verticality |
| Combat | Discrete actions, deep simulation | Continuous real-time |
| Players | Single-player or small co-op | Up to project-configured max (PvP scales high) |
| PvP | Not in scope | Core mode toggle |
| Loot system | Shared item registry | Shared item registry |

Both are session-based + procedural + share the RPG item / skill / perk / magic backbone. They differ in pacing + map style + PvP, not in the systems they touch.

## Layering

Depends on `core/`, `scene/`, `modules/voxel_core`, `modules/multiplayer`, `modules/inventory`, `modules/skills`, `modules/perks`, `modules/magic`. MUST NOT import `editor/`.

## Spec

TBD (Phase 10+). Tracks the `Megabonk-Survivors + Hunger Games hybrid` target in project memory `project-target-games-and-save-model`.
