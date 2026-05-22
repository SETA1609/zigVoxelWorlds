# Content Authoring Tools Spec

> Tools for bulk-creating game content (voxel atlases, recipes, NPCs, item databases) — beyond importing existing files. Gap: [`gaps.md` § 2.1.J](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.J](../gap-references.md).

## Scope

Phase 4 (asset pipeline) handles file *import*. This spec covers *creation* tools — where the content itself comes from. Critical for the four target games' content volumes (Daggerfall = thousands of NPCs; Stardew = hundreds of crops + recipes; Atelier = hundreds of items + recipes; rogue-like = procedural variety).

Without these tools the engine is just a runtime; with them, a solo dev can fill a world.

## Source formats — what content lives in what

| Content shape | Format | Why |
| --- | --- | --- |
| **Tabular** — items, weapons, armor, recipes, voxel atlas, spell effects, loot tables, NPC stat blocks | **CSV** | Spreadsheet-native (Excel/LibreOffice). Bulk-edit. Trivially git-diffable. Autofilter + formula-based generation. Established indie convention (Stardew, RimWorld, FTL). Maps 1:1 to baked binary tables at import time |
| **Tree / graph** — quests, scenes, dialog trees, mod manifests, behavior trees, `project.toml` | **TOML** | Nested structure + comments + named keys. Per-entity files (one file per quest) |
| **Translations** | **`.po`** (gettext) | See [`specs/localization.md`](localization.md) |
| **Assets** — textures, models, audio | source = PNG / glTF / WAV → baked = KTX2 / engine binary / OGG | See [`tech-stack.md`](../tech-stack.md) asset pipeline |

**SQLite is intentionally not used.** See § "Why CSV not SQLite" below.

## Tool families

### 1. Voxel atlas editor

In-editor visual tool to build the voxel atlas (voxel ID → visual + material + properties):

- Visual list of all voxel types
- Add/remove/edit per-voxel entry
- Preview render in a small viewport
- Per-voxel properties: name, texture, hardness, transparency, light-emission, sound family
- Output: `assets/data/voxel_atlas.toml`

### 2. Recipe / item database editor

For Atelier-style + Stardew-style crafting content. Bulk edit:

- Item list with name, icon (voxel-model preview), category, stack limit, weight
- Recipe list with ingredients (item refs + counts) + station + skill required + output
- CSV import for bulk-creating items + recipes from spreadsheets
- Hot-reload — changes reflect in running playtest

### 3. NPC template generator

Procedural NPC variation for Daggerfall-scale worlds:

- Base templates (human, elf, beast, etc.) with appearance parameter ranges
- Generate N variations programmatically (skin tone, face mesh, voxel-clothing combinations)
- Output: per-NPC entries that the world generator can spawn
- Hand-authored NPCs override generated ones by name

### 4. Quest editor

Visual state-machine editor (or TOML-driven, with editor visualization):

- States, transitions, conditions
- Dialog-tree binding to quest state
- Reward declarations (item grants, skill XP, faction shifts)

### 5. Dialog editor

- Tree of dialog nodes (speaker, text, response options)
- Localized string keys (links to [`specs/localization.md`](localization.md))
- Conditional branches (quest flag X = true)
- Voice-line file references (per [`specs/audio.md`](audio.md))

### 6. Bulk import flows

CSV / spreadsheet → engine data:

- "Items.csv" with rows = items, columns = properties → batch-creates item TOMLs
- Sprite-sheet → voxel atlas batch
- Glob-pattern asset rename (rename 500 `tree_*.png` → `oak_*.png` in one go)

## Why CSV not SQLite (for v1.0)

SQLite was considered as the editor's content database. Rejected for v1.0:

| Criterion | CSV | SQLite |
| --- | --- | --- |
| Solo-dev edit workflow | Excel/LibreOffice — autofilter, sort, bulk fill, formulas | Custom editor required (or a separate SQLite GUI) |
| Git diff | Line-oriented, perfect | Binary blob — unreadable diffs |
| Bulk import from spreadsheet | Already a CSV — zero conversion | Convert via script |
| Modder workflow | Open in any text editor or spreadsheet | Requires SQLite-aware tool |
| Cross-table queries | grep / awk / Python scripts ad-hoc | SQL — better at scale |
| Validation at edit time | None native (import-time validation) | Schema constraints + FK checks |
| Dependency footprint | Zero (CSV is just text) | ~700 KB lib + bindings |
| Translator-friendly | Yes (open in Excel) | No |

For v1.0 (rogue-like target — hundreds of rows), CSV wins on every solo-dev-relevant axis. SQLite earns its keep only when:

- Content volume grows to thousands of rows with complex cross-table dependencies (Daggerfall scale)
- Multiple content authors collaborate + need live FK validation
- Quest dialog trees scale past a few hundred nodes (but those are TOML files, not tabular)

**Revisit at v1.x.** When SQLite is added, it sits **only inside the editor** as a working backend; CSV + TOML remain the source-of-truth (checked into git); SQLite imports CSV/TOML at editor start, exports back on save. SQLite never ships at runtime, never replaces the canonical text formats.

Source: [SQLite "When To Use"](https://www.sqlite.org/whentouse.html) — explicitly recommends *against* SQLite for small data + git-managed source workflows. Indie precedent: Stardew Valley, RimWorld, FTL all use spreadsheet-driven (CSV-equivalent) content pipelines.

## Scope per target game

| Game | Tools that matter most |
| --- | --- |
| Daggerfall clone | NPC generator, quest editor, dialog editor, voxel atlas |
| Voxel Stardew | Recipe/item editor, NPC editor (small set, hand-authored), dialog editor |
| Atelier clone | Recipe/item editor (extensive), NPC editor (hand-authored), dialog editor |
| Rogue-like | NPC generator (monsters), item editor (loot table), procedural generation tooling |

## Borrowed patterns

[`gap-references.md` § 2.1.J](../gap-references.md):

- Godot `editor/plugins/` (per-asset-type editor pattern; specifically the particle editor + tileset editor structures)
- Luanti `builtin/mainmenu/` (mod-driven content registration UX)

## Scope for v1.0 — Daggerfall slice requires all six tool families

Per the v1.0 = voxel Daggerfall slice commitment ([`vision.md`](../vision.md) § Shipping strategy), all six tool families land in v1.0. Daggerfall content production needs every one of them; deferring any pushes Daggerfall itself to v1.x, which contradicts the shipping strategy.

The v1.1–v1.3 derivative games (voxel rogue-like / voxel Stardew / voxel Atelier) reuse these tools unchanged — no new authoring tools needed for the derivatives.

| Tool | Scope | Phase |
| --- | --- | --- |
| **Voxel atlas editor** | All games — defines visual + properties per voxel type | Phase 12 |
| **Recipe / item editor** | Daggerfall + Atelier + Stardew + rogue-like loot | Phase 12 |
| **CSV bulk import** | All games — multiplies productivity | Phase 12 |
| **Quest editor** | Daggerfall (heavy), Atelier (medium), Stardew (light); rogue-like minimal | Phase 12 |
| **Dialog editor** | Daggerfall (heavy), Atelier (medium), Stardew (light) | Phase 12 |
| **NPC generator** | Daggerfall (thousands of NPCs); other targets use sparse hand-authored sets | Phase 12 |

This makes Phase 12 (Editor & Tooling) the largest phase in absolute work, but the work is necessary regardless — Daggerfall content scale demands it.

### What's deferred to v1.x

Polish features on top of v1.0 tooling, not new tools:

- **NPC generator with third-person rig output** — v1.0 NPCs animate first-person-facing only; rigged third-person bodies (for the v1.x third-person camera mode) defer until then
- **Quest editor: branching-dialog visualization** — v1.0 ships graph-of-states editing; full visual tree view is polish
- **Recipe editor: spreadsheet-grid bulk mode** — v1.0 ships per-recipe form + CSV import; an integrated spreadsheet grid is post-v1
- **Dialog editor: voice-line waveform alignment** — v1.0 ships text-only; voice acting is deferred per [`specs/audio.md`](audio.md) anyway

## Open decisions

- Per-tool UI: Custom widget panels vs generic property inspector (probably custom for voxel atlas; generic for recipes/items)
- Hot-reload granularity (per-file vs per-entry)
- Versioning content TOMLs (Git is enough; no in-engine VCS needed)
- Localization-aware editing (every string field is a key, not a literal)

## Milestone

Phase 12 (Editor & Tooling). Voxel-atlas editor lets a solo dev add a new voxel type in <30 seconds. Recipe editor lets you bulk-add 50 recipes from a CSV. Test the workflow end-to-end on the rogue-like content pack.
