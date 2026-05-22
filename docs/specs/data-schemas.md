# Data Schemas Spec

> The TOML manifests that every project, mod, and asset database use. Closes [`gaps.md`](../gaps.md) §3 #1, #2, #5, #40. **Phase-0-blocking** — until these are nailed down, nothing else can be implemented against them.

## Scope

Three TOML files form the manifest surface of the engine:

| File | Role | Location |
| --- | --- | --- |
| `project.toml` | Project manifest — declares whether the project is a game or a mod, what engine modules it uses, what platforms it exports to | Project root |
| `mod.toml` | Mod manifest — declares mod metadata, ABI compat, dependencies, load order | Mod root (inside `mods/<name>/` or as standalone) |
| `.assetdb.toml` | Asset database — maps GUIDs to source files + content hashes + importer settings | `<project>/assets/.assetdb.toml` |

## `project.toml` — project manifest

The single declaration that drives the editor, build system, and export pipeline.

```toml
# Project identity
[project]
name = "shadowtower"                # short id, kebab-case; ends up in exports
display_name = "Shadow Tower"       # shown in UI
version = "0.1.0"                   # semver
engine_compat = ">=0.1.0, <1.0"     # engine version range this project targets
kind = "game"                       # "game" or "mod"
description = "A voxel rogue-like dungeon crawler."
author = "@SETA1609"
homepage = "https://github.com/SETA1609/shadowtower"
icon = "branding/icon.png"          # GUID-resolved at runtime

# Only for kind = "mod" — declares which game this mod extends
[project.parent_game]
id = "shadowtower"                  # parent game's [project] name
min_version = ">=0.1.0"
modkit_path = "../shadowtower/modkit/"  # relative to project root, or absolute
                                        # (optional: omit if game is installed system-wide)

# Engine modules enabled for this project (drives per-project tree-shaken export)
[modules]
voxel_core = true
physics_jolt = true
audio = true
skills = true
perks = true
magic = true
crafting = true
inventory = true
multiplayer = false                 # singleplayer rogue-like for v1.0
steam = false                       # set true under -Dsteam=true builds

# Scripting language(s) and entry point
[scripts]
language = "zig"                    # "zig" | "cpp" | "both"
entry = "scripts/game.zig"          # relative to project root
build_options = ["-O", "ReleaseFast"]

# Core mod — for kind = "game" only
[core_mod]
id = "shadowtower-core"
path = "core_mod/"                  # relative to project root

# Per-target export options
[export.linux_x86_64]
embed_pck = false                   # ship .pck as separate file
icon = "branding/icon.png"
launcher_name = "shadowtower"
include_modkit = true               # bundle the auto-generated modkit/

[export.windows_x86_64]
embed_pck = true                    # embed .pck inside the .exe
icon = "branding/icon.ico"
launcher_name = "shadowtower.exe"
include_modkit = true

[export.android]                    # post-v1.0
embed_pck = true
package_id = "com.seta1609.shadowtower"

# Telemetry settings (only meaningful when modules.steam = true or for dedicated servers)
[telemetry]
otel_enabled = false                # OTel only for dedicated-server builds
crash_reports = "opt-in"            # "off" | "opt-in" | "opt-out"

# Project-level overrides for engine settings
[engine_options]
default_view_distance_chunks = 8
target_fps = 60
max_active_entities = 10000
```

### Field semantics

- `[project] kind` — `"game"` projects produce a runnable game + modkit; `"mod"` projects produce a single `.mod` archive against a parent game
- `[project.parent_game]` — required if `kind = "mod"`; tells the editor where to find the parent's modkit
- `[modules]` — each `true` entry pulls the module into the per-project tree-shaken `libzvox-runtime`; `false` strips it
- `[core_mod]` — only meaningful for `kind = "game"`; declares where the vanilla game's content lives (per [`engine-vs-game.md`](../engine-vs-game.md))
- `[export.<target>]` — one table per export target; each gets its own pipeline
- `[engine_options]` — runtime defaults for the shipped game; players can override via settings UI

### Validation

`zig build` validates `project.toml` against this schema at import time. Errors are surfaced with file:line in the editor's diagnostics panel.

## `mod.toml` — mod manifest

Used by all mods (core mod, DLC mods, third-party mods). Same schema regardless of source.

```toml
[mod]
id = "better-loot"                  # short id, kebab-case, unique within the parent game
display_name = "Better Loot"
version = "1.2.0"                   # semver
description = "Replaces vanilla loot tables with more variety + rarer artifacts."
author = "@somemodder"
homepage = "https://github.com/somemodder/better-loot"
icon = "icon.png"                   # bundled in the mod
tags = ["loot", "balance", "gameplay"]

# Compat — what engine version + ABI version this mod was built for
[mod.compat]
engine_version = ">=0.5, <1.0"
abi_version = "^2.0"                # semver-compatible: any 2.x

# Dependencies — other mods this mod requires / conflicts with
[mod.dependencies]
required = ["shadowtower-core >=1.0"]
optional = ["expanded-economy >=0.3"]
incompatible = ["loot-overhaul"]

# Load order hint (relative to other mods of the same priority)
[mod.load_order]
after = ["shadowtower-core"]
before = []
priority = 100                      # 0 = base game, 100 = default mod, higher = later

# Plugin — native .so/.dll if the mod ships compiled code
[mod.plugin]
path = "plugin/libbetterloot.so"    # absent = data-only mod
abi_entry = "better_loot_init"      # exported symbol name

# Content roots
[mod.content]
data = "data/"                      # CSV + TOML
assets = "assets/"                  # textures, models, sounds
scripts = "scripts/"                # Zig / C++ source if applicable
```

### Field semantics

- `[mod.compat]` — engine + ABI version range; engine refuses to load mods outside the range
- `[mod.dependencies]` — `required` blocks load if missing; `optional` enables extra integrations if present; `incompatible` blocks load if also enabled
- `[mod.load_order]` — `after` / `before` are soft hints; `priority` is the tiebreaker (base game = 0, normal mods = 100, override mods = 200)
- `[mod.plugin]` — optional; mods without `plugin` are data-only (no native code)

## `.assetdb.toml` — asset database

GUID-keyed registry of every source asset in the project. Auto-managed by the editor; humans read it for debugging.

```toml
# Database metadata
[meta]
schema_version = 1
hash_algorithm = "blake3"           # "blake3" | "sha256"
created_at = "2026-05-21T14:32:00Z"

# Each asset is a [[asset]] entry
[[asset]]
guid = "018f4b3a-7c00-7000-8000-1a2b3c4d5e6f"   # UUIDv7 (time-ordered)
source = "textures/iron_albedo.png"
content_hash = "blake3:9a8f7e6d5c4b3a2918..."
imported_at = "2026-05-21T14:30:12Z"
importer = "texture"
importer_version = 1

[asset.settings]                     # importer-specific settings
format = "BC7"                       # texture compression target
mip_levels = "auto"
srgb = true

[[asset]]
guid = "018f4b3b-1200-7000-8000-aabbccddeeff"
source = "models/iron_sword.gltf"
content_hash = "blake3:..."
imported_at = "2026-05-21T14:30:45Z"
importer = "mesh"
importer_version = 1

[asset.settings]
deduplicate_verts = true
generate_lods = true
lod_count = 3
```

### Field semantics

- `guid` — **UUIDv7** (time-ordered, sortable). Generated on first import of a source file. Never regenerated even on content change. Stable across renames + moves.
- `source` — relative path from the project's `assets/` directory. Updated on rename (the GUID stays stable).
- `content_hash` — Blake3 of the source file content. Used to detect when a re-bake is needed.
- `importer` + `importer_version` — which importer baked this asset, and at what version. Re-bake if the importer version changes.
- `settings` — importer-specific TOML; schema varies per importer.

### Why UUIDv7

- Time-ordered: insertion order is preserved in sort, useful for debugging
- 128-bit collision-resistant
- Standard format: tooling-friendly
- Source: [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562) (UUIDv7 spec)

### Why Blake3 not SHA-256

- ~5–10× faster than SHA-256 on modern hardware
- 128-bit prefix is enough collision-resistance for content addressing
- Already used by tools we like (Bazel, Cargo's planned content store)
- Source: [BLAKE3 spec](https://github.com/BLAKE3-team/BLAKE3)

## Schema versioning

Each file declares a `schema_version` in its top-level table (`[project]`, `[mod]`, `[meta]`).

- v1 = the schema above
- Bumps go up by 1
- The engine ships migration functions for each version-bump: read v(N-1), write v(N)
- Migrations are run at editor-open time + cached
- Bumping schemas is a breaking change for tooling; coordinate with `[`gaps.md`](../gaps.md)` #23 (ABI versioning) on cadence

## Validation rules (lint at import time)

These should be enforced by the editor's validator before any export step:

- `[project] kind` must be `"game"` or `"mod"`
- If `kind = "mod"`, `[project.parent_game]` must exist
- If `kind = "game"`, `[core_mod]` must exist
- Module `true` entries must be modules the engine knows about
- `[scripts] entry` path must exist
- `[export.<target>]` blocks must reference at least one defined target
- `.assetdb.toml` entries must reference existing source files (or the asset is orphaned — flagged in editor diagnostics)

## Open decisions deferred to later

- Mod priority numeric ranges + reserved bands (e.g. 0-99 = base game, 100-199 = mods, 200-299 = overrides) — finalize at Phase 14
- `[engine_options]` exhaustive list — grows as engine modules add tunables
- Cross-game mod support (a mod that works for multiple parent games via duck-typing) — defer to v1.x
- Save format embedding `project.toml` hash for compatibility check — Phase 13 detail

## Closes

- [`gaps.md`](../gaps.md) #1 — `project.toml` schema
- [`gaps.md`](../gaps.md) #2 — `mod.toml` schema
- [`gaps.md`](../gaps.md) #5 — `assetdb.toml` format
- [`gaps.md`](../gaps.md) #40 — `project.toml` `kind` + `parent_game` extension
