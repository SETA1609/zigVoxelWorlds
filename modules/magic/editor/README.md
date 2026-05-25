# `modules/magic/editor/`

> Module-owned editor sub-tree for `modules/magic/`. Tools-only — gated by the comptime `tools_enabled` build option.

This is the canonical example of a per-module editor sub-tree (see [`project-structure.md` § Module contract](../../../docs/project-structure.md)). Most modules don't need one; magic does because spell composition is data-heavy.

## Planned panels

- `spell_composer.zig` — drag-and-drop atomic effects onto a spell card; assign primary color school; preview cost / power scaling vs. each color skill level
- `school_palette.zig` — preview the RCSS color palette per school (red→white→black); validate accessibility (color-blind-safe contrasts)
- `effect_browser.zig` — list view of all atomic effects across the 9 colors; filter by color; jump-to-edit the underlying TOML

## Layering

`tools_enabled = true` only. May import `core/`, `scene/ecs`, the parent `modules/magic/src/`, and ImGui through the editor's adapter. MUST NOT be imported by `modules/magic/src/` or any non-editor code.

## How `build.zig` finds it

The module-walker in `build.zig` looks for an `editor/` sub-tree inside every `modules/<name>/`. When found and `tools_enabled = true`, it adds the sub-tree to the editor build. When `tools_enabled = false`, the sub-tree is elided entirely.
