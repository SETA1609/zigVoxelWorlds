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

## Scope reduction (realistic for solo v1.0)

A solo dev cannot ship all 6 tool families at v1.0. Prioritize:

1. **Voxel atlas editor** — needed by every game (Phase 12)
2. **Recipe/item editor** — needed by Atelier + Stardew (Phase 12)
3. **CSV bulk import** — multiplies productivity (Phase 12)
4. **Quest editor (basic)** — needed by Daggerfall + Atelier (defer to Phase 12.5 or v1.1)
5. **Dialog editor** — needed by Daggerfall (defer to v1.1)
6. **NPC generator** — defer to v1.1 alongside third-person rendering work

This implies **v1.0 ships the rogue-like target first** (smallest content surface), with Daggerfall / Stardew / Atelier scoped to v1.1+.

## Open decisions

- Per-tool UI: Custom widget panels vs generic property inspector (probably custom for voxel atlas; generic for recipes/items)
- Hot-reload granularity (per-file vs per-entry)
- Versioning content TOMLs (Git is enough; no in-engine VCS needed)
- Localization-aware editing (every string field is a key, not a literal)

## Milestone

Phase 12 (Editor & Tooling). Voxel-atlas editor lets a solo dev add a new voxel type in <30 seconds. Recipe editor lets you bulk-add 50 recipes from a CSV. Test the workflow end-to-end on the rogue-like content pack.
