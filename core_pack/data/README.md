# `core_pack/data/`

> Baseline TOML game data — skills, perks, magic effects, recipes — that ships with the engine. Forms the starting set of registered types when no project overrides them.

## Planned subtrees

```text
core_pack/data/
├── skills/             # baseline skill definitions (combat, magic colors, crafting)
│   ├── red_magic.toml
│   ├── orange_magic.toml
│   ├── …               # one per color (red/orange/yellow/green/blue/indigo/violet/white/black)
│   ├── smithing.toml
│   ├── alchemy.toml
│   └── …
├── perks/              # a handful of starter perks gated on skill thresholds
├── spells/
│   └── effects/
│       ├── red/        # destruction-style effects (burn, drain_health, …)
│       ├── orange/     # conjuration-style (summon_skeleton, bound_blade, …)
│       ├── yellow/     # illusion-style (light, charm, calm, …)
│       ├── green/      # mortal restoration (heal, cure_disease, …)
│       ├── blue/       # alteration (feather, water_walking, transmute, …)
│       ├── indigo/     # thaumaturgy (levitate, push, …)
│       ├── violet/     # mysticism (soul_trap_store, telekinesis, dispel, …)
│       ├── white/      # holy (banish_undead, sanctify, divine_heal, …)
│       └── black/      # necromancy (raise_dead, life_drain, curse, …)
├── recipes/            # crafting recipe seeds
├── items/              # baseline item definitions (weapons, armor, ingredients, …)
│   └── tools/          # mining tools (pickaxe/axe/shovel/scythe tiers) with mining_speed_multiplier + tool_class + tier + durability per project memory `project-voxel-hardness-mining`
└── voxels/             # voxel-type definitions with hardness + tool_tier_required + effective_tool_class + drops (per project memory `project-voxel-hardness-mining`)
```

## Schemas

Every TOML file conforms to the schemas in [`specs/data-schemas.md`](../../docs/specs/data-schemas.md). Importer at `src/editor/import/data.zig` parses + validates + bakes into the cached binary format.

## Override model

A project's own `<project>/data/` overlays on top of these via the layered loader in `modules/modding/`. A project can:

- **Extend** — add new skills, perks, spells without touching the base set
- **Override** — define a TOML with the same GUID as a base file; the project's version wins
- **Disable** — list base GUIDs in `<project>/project.toml`'s `[disabled]` table

This is the same mechanism mods use; the project is just the topmost layer (last wins).
