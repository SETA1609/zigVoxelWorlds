# `src/`

> The engine source tree. Layered per [`project-structure.md`](../docs/project-structure.md); `build.zig` enforces the layering with an import-graph check.

## Layering (lower → higher; lower may not import higher)

```text
core/          → foundational primitives (allocators, math, handles, asset DB, TOML)
  ↓
servers/       → opaque-handle interfaces (RenderServer, VoxelServer, …)
backends/      → concrete impls behind the servers (Vulkan, Jolt, miniaudio, ENet)
  ↓
scene/         → world model + ECS; handles only, no raw C/C++ types
  ↓
modules/<n>/   → pluggable subsystems (in the sibling `modules/` tree, not under `src/`)
  ↓
project_manager/ + editor/   → tools-only, gated by comptime `tools_enabled`
  └── editor/import/         → asset import pipeline
  └── editor/export/         → per-project libzvox-runtime + PCK build
```

Other top-level `src/` subdirs:

- [`platform/`](platform/README.md) — OS abstraction (window, input, threading, FS, process spawning)
- [`ui/`](ui/README.md) — runtime UI Zig API; RmlUi-aware wrapper lives in `libs/zig-cpp-ui-stack-adapter/` (see [`specs/ui.md`](../docs/specs/ui.md))
- [`project_manager/`](project_manager/README.md) — editor-only PM, re-exec model

## Entry point

`main.zig` (already exists). Branches on CLI args:

- No `--project` → launches Project Manager (`src/project_manager/`)
- `--project <path>` → launches the editor with the project loaded (or, in built-game launchers, runs the runtime)
- `--<cli-subcommand>` → dispatched in-process; no separate `src/tools/` tree (CLI subcommands like `--dump-save`, `--validate-mod`, `--bake-assets` are flags on the same binary)

## Transitional contents

`src/c/` and `src/cpp/` hold the hello-world template inherited from the build scaffolding. They are not part of the target layout in [`project-structure.md`](../docs/project-structure.md) and will be removed once `core/` + `servers/` + the Vulkan backend take over their role (Phase 1). They stay for now because `build.zig` still walks them.

## Layering rule (build.zig-enforced)

Build will fail if any of these imports appear:

- `core/` → anywhere else
- `servers/` / `backends/` → `scene/`, `modules/`, `editor/`, `project_manager/`, `ui/`
- `scene/` → `modules/`, `editor/`, `project_manager/`
- `modules/<x>/src/` → `editor/`, `project_manager/`, sibling `modules/<y>/src/` *unless* `<y>` is in `<x>`'s `config.zig` `deps`

Tooling subtrees (`editor/`, `project_manager/`) may import everything below them; they are themselves elided in non-`tools_enabled` builds.
