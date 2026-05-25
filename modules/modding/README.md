# `modules/modding/`

> Layered mod loader + native plugin ABI + mod TOML reader. Phase 14.

This was previously placed under `src/modding/`. Moved here so the mod-loader itself is a regular module — same `config.zig` / `register_types.zig` / `src/` contract as every other engine subsystem, so its scope can be inspected and its build can be elided in builds that want a fully closed game.

## What it provides

- Layered loader — overlay sequence: `core_pack/` (base game) → `<project>/data/` → `<project>/mods/<mod>/` (last wins). Same shape as Luanti's `builtin/` + `games/` + `mods/`.
- Mod manifest (TOML) — declares dependencies, load order, supported engine version, optional native plugin path
- Native plugin loader — `dlopen`/`LoadLibrary` mod plugins against the engine's **stable C ABI** ([`specs/c-abi.md`](../../docs/specs/c-abi.md))
- Asset overlay — mod assets shadow base assets by GUID match
- Data overlay — mod TOML extends / overrides base data (recipes, spells, NPC tables, …)

## Reference patterns

- **Luanti** layered mods (`builtin/` + `games/` + `mods/`) — adapted, **without** the Lua API. Native C ABI + TOML data replace Lua entirely
- **Unreal** plugin descriptors (`IPluginManager.h:110`, `PluginDescriptor.h`) — informed the TOML manifest schema (same fields: name, modules, dependencies, load phase)

See [`engine-references.md`](../../docs/engine-references.md) § Luanti and § Unreal · Plugin descriptors, and [`engine-vs-game.md` § 8](../../docs/engine-vs-game.md).

## Layering

Depends on `core/` + `platform/` (for `dlopen`). May be depended on by *any* other module that wants to expose mod-replaceable extension points; by convention, modules declare those extension points via the stable C ABI.

## Spec

[`engine-vs-game.md` § 8](../../docs/engine-vs-game.md), [`specs/mod-manager.md`](../../docs/specs/mod-manager.md), [`specs/c-abi.md`](../../docs/specs/c-abi.md).
