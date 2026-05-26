# `src/editor/panels/`

> Individual ImGui panels. Each panel is a self-contained Zig file that registers with `editor_layer.zig` and renders into its own ImGui dockspace tab.

Planned panels (from [`specs/editor.md`](../../../docs/specs/editor.md)):

- `project_settings.zig` — graphical front-end for `project.toml` (modules / audio backend / UI document toggle / multiplayer settings)
- `voxel_brush.zig` — place / erase / paint voxels in the active chunk
- `biome_painter.zig` — paint biome IDs onto the heightmap layer
- `scene_browser.zig` — tree view of the world's scene-instance graph (edits `orchestrator.toml`)
- `entity_spawner.zig` — drag-and-drop entity templates into the viewport
- `skill_editor.zig` — TOML-driven data editor for `<project>/assets/data/skills/` (Fallout-style tagged + cost)
- `perk_editor.zig` — TOML-driven data editor for `<project>/assets/data/perks/`
- `recipe_editor.zig` — TOML-driven data editor for `<project>/assets/data/recipes/`
- `voxel_editor.zig` — TOML-driven data editor for `<project>/assets/data/voxels/` (hardness + tool_tier_required + drops per project memory `project-voxel-hardness-mining`)
- `tool_editor.zig` — TOML-driven data editor for `<project>/assets/data/items/tools/` (mining_speed_multiplier + tool_class + tier + durability)
- `theme_editor.zig` — graphical color / typography / spacing picker for the widget kit; writes a TOML theme consumed by `src/ui/widgets/theme.zig`; live preview pane
- `rcss_preview.zig` — preview pane for projects using the document UI layer (RmlUi); re-renders on file save. Full visual RCSS editing deferred post-v1.0.
- `asset_browser.zig` — file-system view of `<project>/assets/`, drag-source for the viewport

Module-specific panels (`spell_composer.zig`, `school_palette.zig`, `effect_browser.zig` for magic; perk-tree visual layouts for perks; etc.) live in `modules/<name>/editor/`, not here. The editor layer aggregates both directories.

## Dual-authoring + dirty-buffer + undo/redo contract

Every panel that edits a TOML file must comply with the dual-authoring contract from [`specs/editor.md`](../../../docs/specs/editor.md) § Dual-authoring principle + § Dirty-buffer + explicit-save + undo/redo. See also project memory `project-editor-dual-authoring-toml`.

Per-panel requirements:

1. **Read**: parse the TOML through the data-stack adapter's comment-preserving API; populate UI fields
2. **Buffer mutations**: every user action emits a `Command` to the per-document undo stack at `src/editor/undo/`. No direct disk writes.
3. **Save (Ctrl+S)**: flush the buffer to disk via the comment-preserving writer. Surface a warning if format preservation isn't possible for any value.
4. **Hot-reload + conflict**: register the open file with the editor's file watcher. On external change while buffer is clean → silent refresh. On external change while buffer is dirty → conflict dialog (discard buffer / discard external / 3-way diff).
5. **Dirty indicator**: tab title prefix `*` when there are unsaved changes.
6. **Live preview vs playtest split**: the panel may show a live preview using the buffer (e.g. theme editor preview pane re-themes immediately). The playtest run reads from disk only — user must save first to test changes.

## Reference patterns

- **Hazel** `Hazelnut/src/Panels/SceneHierarchyPanel.cpp:35` — the closest API match for ImGui property-editor layout
- **Unreal Editor** `IDetailCustomization.h` + `DetailLayoutBuilder.h` — the mental model for per-type detail panels
- **Unreal Editor** `EditorModeManager.h:42, 90` — mode-tools registry; informs how voxel brush / biome painter switch active tool

See [`engine-references.md`](../../../docs/engine-references.md) § Hazel and § Unreal · Editor mode-tools.

## Panel contract

Each panel file:

```zig
// pub const Panel = struct { … };
// pub fn register(reg: *editor.PanelRegistry) void;
// pub fn render(ctx: *editor.Ctx) void;
```

Implementation TBD by owner during Phase 12 — this is a placeholder describing the shape.
