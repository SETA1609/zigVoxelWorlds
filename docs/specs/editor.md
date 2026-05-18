# Editor Spec

> The in-engine tools when `tools_enabled` is set. Roadmap: [`ROADMAP.md` § Phase 12](../ROADMAP.md). Code-editor + debug-log subpanels: [`engine-vs-game.md` § 6 + § 7](../engine-vs-game.md). Reference patterns: [`engine-references.md` → Hazelnut + Godot + Unreal Editor](../engine-references.md).

## Scope

Editor-only code lives under `src/editor/` behind comptime `tools_enabled`. Runtime layers never import it (enforced by `build.zig` import-graph check, see [`ARCHITECTURE.md` cross-cutting](../ARCHITECTURE.md#cross-cutting-concerns)).

## Panels (ImGui)

- **Voxel brush** — paint / erase / fill / replace
- **Biome / region painter** — paint per-region biome IDs
- **Scene instance browser** — drag-and-drop dungeons / towers / houses into the world
- **Entity spawner** — live preview before commit
- **Skill + perk editor** — visual; edits TOML in `<project>/assets/data/skills/` and `perks/`
- **Crafting recipe editor** — visual; edits TOML in `<project>/assets/data/recipes/`
- **Asset browser** — file tree + thumbnails + drag-into-scene
- **Module enable/disable UI** — toggles `project.toml` `[modules]` table
- **Live hot-reload preview** — re-import + re-run on file change

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
