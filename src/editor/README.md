# `src/editor/`

> Editor-only. ImGui panels, gizmos, viewport, hot-reload host, importers, and the per-project export pipeline. The entire subtree is gated by a comptime `tools_enabled` build option ([`engine-references.md`](../../docs/engine-references.md) § Godot · `TOOLS_ENABLED`).

## Subtree

- `editor_layer.zig` — top-level editor orchestration (dock layout, panel registry, lifecycle)
- `viewport.zig` — 3D scene viewport (renders through `servers/render_server`)
- [`panels/`](panels/README.md) — individual ImGui panels (voxel brush, biome painter, scene browser, …)
- [`code_editor/`](code_editor/README.md) — embedded Neovim host (msgpack-RPC) with ImGui fallback
- [`script_builder/`](script_builder/README.md) — watches `<project>/scripts/`, drives `zig build-lib` / `zig c++ -shared`
- [`hot_reload/`](hot_reload/README.md) — dynamic-lib reload host (game DLL swap without quitting the editor)
- [`import/`](import/README.md) — asset pipeline (PNG→KTX2, glTF→mesh, .vox→voxel, audio, shader, data TOML→cached binary)
- [`export/`](export/README.md) — per-project tree-shake + link + PCK pack + launcher stub

## Build gating

`build.zig` exposes `tools_enabled: bool = true` for editor builds, `false` for exported game runtimes. The compile-time constant is consumed via `@import("build_options")`; non-editor builds elide every import path that goes through `src/editor/`. Verified by `build.zig`'s import-graph check.

## Reference patterns

- **Hazel/Hazelnut** — clean runtime/editor split; panel patterns at `Hazelnut/src/Panels/SceneHierarchyPanel.cpp:35`
- **Godot** — `TOOLS_ENABLED` define + `is_editor_hint()` inline-false at `core/config/engine.h:170-187`
- **Unreal Editor mode tools** — `EditorModeManager.h:42, 90` (modal tool registry — informs voxel brush / biome painter activation model)

## Layering rule

`tools_enabled = true` only. May import `core/`, `servers/`, `scene/`, `modules/<name>/`, and `modules/<name>/editor/`. MUST NOT be imported by anything outside the editor subtree, `project_manager/`, or `main.zig`'s editor branch.
