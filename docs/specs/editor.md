# Editor Spec

> The in-engine tools when `tools_enabled` is set. Roadmap: [`ROADMAP.md` § Phase 12](../ROADMAP.md). Code-editor + debug-log subpanels: [`engine-vs-game.md` § 6 + § 7](../engine-vs-game.md). Reference patterns: [`engine-references.md` → Hazelnut + Godot + Unreal Editor](../engine-references.md).

## Scope

Editor-only code lives under `src/editor/` behind comptime `tools_enabled`. Runtime layers never import it (enforced by `build.zig` import-graph check, see [`ARCHITECTURE.md` cross-cutting](../ARCHITECTURE.md#cross-cutting-concerns)).

## Dual-authoring principle — text + GUI both work

**Every config / data TOML in the project is editable both manually (text editor) and in-engine (GUI panel).** Decision 2026-05-26 — see project memory `project-editor-dual-authoring-toml`.

The same Godot-style philosophy: `.tscn` / `.tres` / `project.godot` can be edited as text OR through Godot's GUI. zVoxRealms's TOMLs (`project.toml`, `skills/*.toml`, `spells/effects/<color>/*.toml`, `recipes/*.toml`, `items/tools/*.toml`, `voxels/*.toml`, `mods/<mod>/mod.toml`, `orchestrator.toml`, `.assetdb.toml`) follow the same rule.

User picks per workflow:

- **Text editing** (vim / VS Code / direct file edit) — fastest for power users; full diff-ability in git; comments preserved naturally
- **GUI editing** (in-engine ImGui panels) — better for non-programmers, theme designers, content creators; immediate visual feedback

Both routes must converge on the same TOML file. The editor watches files for external changes and hot-reloads.

### Round-trip discipline (the hard part)

When the editor writes TOML back, it MUST preserve user-authored formatting + comments. Without this, every editor save destroys hand-edited annotations and the dual-authoring promise breaks.

Implementation requirement on the data-stack adapter (`libs/zig-cpp-data-stack-adapter/`): expose a **comment-and-formatting-preserving TOML API**. References:

- [`toml-edit`](https://docs.rs/toml_edit/) (Rust) — proven comment-preserving editor API
- [`tomlkit`](https://github.com/python-poetry/tomlkit) (Python) — same model
- `toml++` (C++, our likely vendor) — has AST traversal but the default emit drops comments; would need wrapping or a fork to preserve formatting

This is a hard constraint on the data-stack adapter design — it can't just use a "parse → object → emit" round-trip. It must operate on the AST/CST level. If toml++ can't do this cleanly, vendor a different lib (e.g. write thin C bindings around toml-edit) — but pick the lib WITH this in mind.

If round-trip preservation is genuinely impossible at v0.5, the editor surfaces a warning on save ("comments / formatting may be lost — confirm?") and the project owner accepts the trade-off explicitly per file. This is a regression compared to Godot's clean dual-authoring; mark it as a tracked gap.

## Dirty-buffer + explicit-save + undo/redo

Per Godot's editor model — and stated explicitly 2026-05-26 — the editor uses a **dirty-buffer + explicit-save** workflow with undo/redo on the buffer:

- Each TOML opened in the editor maintains an **in-memory model** that may diverge from the on-disk file (the "dirty" state)
- All mutations happen against the in-memory model first, **never directly to disk**
- **Undo / redo** operate on the in-memory model via a command pattern — each user action is a `Command` with `do` / `undo` / `redo` operations
- **Save** (Ctrl+S, File → Save, File → Save All) flushes the in-memory model back to disk via the comment-preserving TOML writer in the data-stack adapter
- **Dirty indicator** in tab titles — e.g. `*project.toml` while there are unsaved changes
- **Save-on-close** prompt blocks any operation that would discard unsaved buffers: editor exit, project switch, the re-exec into a built game (Godot Project Manager re-exec pattern from [`engine-references.md`](../engine-references.md))

### External-edit conflict resolution

When the file on disk changes (e.g. user edited TOML directly in vim) AND the editor has an unsaved buffer for the same file:

| Editor state | External change | Behavior |
| --- | --- | --- |
| No buffer (file not open in editor) | external write | Silent hot-reload — file appears in next refresh |
| Buffer matches on-disk | external write | Silent hot-reload — no conflict, buffer follows disk |
| Buffer **dirty** (unsaved) | external write | **Conflict dialog** blocks the editor until resolved |

Conflict dialog options:

1. **Discard editor buffer, accept on-disk version** — loses unsaved changes; ✓ for "I forgot I had this open"
2. **Discard external change, keep editor buffer** — overwrites on next save; ✓ for "the external edit was a mistake"
3. **Show 3-way diff and let me merge** — Godot's approach for `.tscn` merge conflicts; bigger UI investment, but the right answer when both sides have real work

**Default policy: dialog blocks until resolved.** Never silently lose work in either direction.

### Where undo/redo lives

- **Per-document** — each open file has its own undo stack. Switching tabs preserves stack state. Closing a file discards the stack (with save-on-close prompt first).
- **In the editor subtree only** — code lives at `src/editor/undo/`. Runtime never sees it; `tools_enabled = false` strips it.
- **Command granularity** — one `Command` per *logical* change (e.g. "set `hardness` from 1.5 to 2.0"), not per keystroke. The editor batches typing within a debounce window (~500 ms) into a single command.
- **Memory cap** — bounded undo history per file (default: 100 commands or 10 MB whichever is hit first). Saving does **not** clear the stack — undo continues to work across save boundaries until the file is closed.
- **Cross-document undo is NOT supported in v1.0** — Godot doesn't either. A change in `project.toml` and a change in `skills/foo.toml` have separate undo stacks. Bulk operations that span files (e.g. a refactor renaming a skill GUID across many files) are explicit commands the editor offers, not undo-stack composability.

### Hot-reload semantics — split into two paths

The dirty-buffer model splits hot-reload into two distinct paths:

1. **Game runtime hot-reload** — reads from **disk** only. The editor's unsaved buffer does NOT affect the running game. To "test current changes," the user must save first. Same behavior whether edits came from the GUI or from text-editing the file directly.
2. **Editor preview** — uses the **in-memory buffer** immediately for live preview. Example: changing a widget theme color in the theme editor updates the preview pane instantly, without writing the theme TOML.

This split is important: the playtest button inside the editor sees the *saved* version of files, not the buffer. The widget theme preview sees the *buffer*. Document this clearly in any panel that has preview + playtest paths.

### Reference patterns

- **Godot** — `editor/editor_node.cpp` (save/load lifecycle); `editor/editor_undo_redo_manager.h` (`EditorUndoRedoManager` per-scene undo); conflict dialog in `editor/filesystem_dock.cpp` external-change watcher
- **Unreal** — `Editor/UnrealEd/Public/Editor/TransBuffer.h` (`UTransBuffer` transaction-based undo); not directly relevant since UE uses asset-level transactions, not file-level
- **VS Code / generic editors** — dirty tab indicator, save-on-close prompt, file-watcher-based external-change conflict — all common patterns

## Panels (ImGui)

Each panel below edits its corresponding TOML file with comment-preserving round-trip. Hot-reload on external changes is the inverse direction (file changes → UI refresh).

### v1.0 panels (Phase 11 / 12)

- **Project Settings panel** — graphical front-end for `project.toml`. Tabs: General (name, version, target platforms), Modules (enable/disable per module, with dependency validation), Audio (`backend = sdl3|miniaudio`), UI (`document = true|false`), Multiplayer (`max_players`, modes allowed), Save (paths, compression). Roughly equivalent to Godot's Project → Project Settings dialog.
- **Voxel brush** — paint / erase / fill / replace
- **Biome / region painter** — paint per-region biome IDs
- **Scene instance browser** — drag-and-drop dungeons / towers / houses into the world; edits `orchestrator.toml`
- **Entity spawner** — live preview before commit
- **Skill + perk editor** — visual; edits TOML in `<project>/assets/data/skills/` and `perks/` (tagged-skill picker, point-cost preview per Fallout model)
- **Crafting recipe editor** — visual; edits TOML in `<project>/assets/data/recipes/`
- **Voxel-type editor** — `<project>/assets/data/voxels/*.toml` — hardness slider, tool-tier picker, effective-tool-class dropdown, drops table (per project memory `project-voxel-hardness-mining`)
- **Tool editor** — `<project>/assets/data/items/tools/*.toml` — mining-speed multiplier, tool-class, tier, durability
- **Spell / effect editor** — `<project>/assets/data/spells/effects/<color>/*.toml`; lives under `modules/magic/editor/` (canonical per-module editor sub-tree)
- **Asset browser** — file tree + thumbnails + drag-into-scene; edits `.assetdb.toml` GUID map
- **Live hot-reload preview** — re-import + re-run on file change

### Theme editor — widget kit + RCSS

- **Widget theme editor** — graphical color / typography / spacing picker for the engine widget kit (Layer 1 of [`specs/ui.md`](ui.md) § Two-layer architecture). Writes a TOML theme file consumed by `src/ui/widgets/theme.zig`. Preview pane shows live widgets re-themed in real-time.
- **RCSS preview pane** — for projects using the document UI layer (Layer 2 of UI spec). NOT a full WYSIWYG editor in v1.0 — RmlUi handles the rendering; the editor just provides a viewport that re-renders on file save. Full visual RCSS editing is post-v1.0.

### Module-owned panels

Some panels live under `modules/<name>/editor/` instead of `src/editor/panels/`, when they're tightly coupled to a single module's data. Examples:

- `modules/magic/editor/spell_composer.zig` — spell composition (multi-effect drag-and-drop)
- `modules/magic/editor/school_palette.zig` — per-color RCSS palette preview (the 9 rainbow schools)
- `modules/magic/editor/effect_browser.zig` — filter atomic effects across all 9 colors

The editor layer aggregates `src/editor/panels/` + every `modules/<name>/editor/` subtree at build time.

## Specialized sub-systems (own docs)

- **Code editor panel** (bundled Neovim) — see [`engine-vs-game.md` § 6](../engine-vs-game.md#6-code-editor-panel-neovim)
- **Runtime debug output panel** (libghostty) — see [`engine-vs-game.md` § 7](../engine-vs-game.md#7-runtime-debug-output-panel-libghostty)

## Reference patterns

- Hazelnut `Panels/SceneHierarchyPanel.cpp:35` — ImGui-based entity hierarchy + property editor
- Hazelnut `EditorLayer.cpp` — viewport handling, entity picking via render ID, gizmos
- Godot `editor/` layering — modular module-registered panels
- Unreal `Editor/UnrealEd/Public/EditorModeManager.h:42, 90` — modal mode-tool registry (relevant for brush / painter modes)
- Unreal `Editor/PropertyEditor/Public/IDetailCustomization.h` — per-type detail customization (relevant for skill/perk/recipe editors)

## Milestone (from ROADMAP)

Build a small dungeon end-to-end in the editor without touching code; edit a script in the bundled Neovim panel; trigger a runtime error and click the stack-trace `file:line` in the debug log panel to jump to its source in the Neovim panel.
