# `src/editor/panels/`

> Individual ImGui panels. Each panel is a self-contained Zig file that registers with `editor_layer.zig` and renders into its own ImGui dockspace tab.

Planned panels (from [`specs/editor.md`](../../../docs/specs/editor.md)):

- `voxel_brush.zig` — place / erase / paint voxels in the active chunk
- `biome_painter.zig` — paint biome IDs onto the heightmap layer
- `scene_browser.zig` — tree view of the world's scene-instance graph
- `entity_spawner.zig` — drag-and-drop entity templates into the viewport
- `skill_editor.zig` — TOML-driven data editor for `<project>/assets/data/skills/`
- `recipe_editor.zig` — TOML-driven data editor for `<project>/assets/data/recipes/`
- `asset_browser.zig` — file-system view of `<project>/assets/`, drag-source for the viewport

Module-specific panels (magic spell editor, perk tree, …) live in `modules/<name>/editor/`, not here. The editor layer aggregates both directories.

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
