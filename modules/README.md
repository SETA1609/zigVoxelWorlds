# `modules/`

> Pluggable engine subsystems. Each module is a self-contained directory with a fixed contract (`config.zig`, `register_types.zig`, `src/`, optional `editor/`). `build.zig` walks this tree, evaluates each module's `canBuild()`, and codegens a comptime dispatch table consumed by `main.zig`.

## Module contract

```text
modules/<name>/
├── config.zig          # pub const name = "..."; pub fn canBuild(opts: BuildOptions) bool; pub const deps = .{...};
├── register_types.zig  # pub fn initialize(level: InitLevel) void; pub fn uninitialize(level: InitLevel) void;
├── src/                # Module sources, plain Zig
└── editor/             # OPTIONAL — tools-only sub-tree (panels, gizmos, importers for this module's data)
```

`build.zig` walks `modules/`, evaluates `canBuild(opts)`, and emits `src/registered_modules.gen.zig` containing a comptime array of module pointers. `main.zig` calls `initializeAll(.core)` → `.servers` → `.scene` → (editor builds only) `.editor`.

## Modules in this tree

| Module | Phase | Status | What it provides |
| --- | --- | --- | --- |
| [`voxel_core/`](voxel_core/README.md) | 3 | planned | Chunk + meshing + streaming + lighting |
| [`skills/`](skills/README.md) | 8 | planned | Classless skill progression |
| [`perks/`](perks/README.md) | 8 | planned | Perk tree gated by skill thresholds |
| [`magic/`](magic/README.md) | 8 | planned | Spell effects + costs + cooldowns |
| [`crafting/`](crafting/README.md) | 8 | planned | Recipe resolver + station gating |
| [`inventory/`](inventory/README.md) | 8 | planned | Item containers, stacks, slots |
| [`ai/`](ai/README.md) | 8 | planned | Behavior trees + utility scoring + NavMesh |
| [`multiplayer/`](multiplayer/README.md) | 10 | planned | Replication + interest management + lockstep stepping |
| [`modding/`](modding/README.md) | 14 | planned | Layered mod loader + native plugin ABI + mod TOML reader |
| [`farming/`](farming/README.md) | post-1.0 | optional | Stardew-style farming layer |
| [`rogue_tower/`](rogue_tower/README.md) | post-1.0 | optional | Endless tower mode |

## Held — not in tree yet

- **`modules/steam/`** — Steamworks + Workshop integration. Held until the [Steamworks SDK Agreement](https://partner.steamgames.com/documentation/sdk_access_agreement) redistribution terms are reviewed against this repo's Apache 2.0 licensing. Likely destination: `libs/zig-cpp-steam-stack-adapter/` (C ABI only) + gameplay-side logic in a separate private repo, since Valve's SDK is not open-source-licensed.

## Reference patterns

- **Godot** `modules/modules_builders.py:15-51` — SCons codegen of `register_module_types.gen.cpp` with `#ifdef MODULE_X_ENABLED initialize_x_module(p_level)`. We use Zig comptime instead of SCons codegen, but the four init levels (`core`/`servers`/`scene`/`editor`) and the dispatch shape are direct lifts.
- **Unreal** `Engine/Source/Runtime/Projects/Public/Interfaces/IPluginManager.h:110, 273, 382` — plugin descriptor schema (name, modules, dependencies, load phase). Our `config.zig` is the TOML equivalent of `PluginDescriptor.h` + `ModuleDescriptor.h`.

See [`engine-references.md`](../docs/engine-references.md) § Godot · Module system and § Unreal · Plugin descriptors.
