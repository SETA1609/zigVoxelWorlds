# `src/project_manager/`

> Editor-only. Opens at startup when no `--project <path>` arg is given. Lists known projects, lets the user create / import / remove them, and re-execs the same binary with `--project <path>` to enter the editor.

Planned files:

- `pm_window.zig` — the ImGui window + tab layout
- `project_list.zig` — read/write `$XDG_DATA_HOME/zvoxrealms/projects.cfg` (TOML); ports the *idea* of Godot's `ProjectList::Item`
- `project_create.zig` — new-project wizard (name, path, template selection from `core_pack/`)

## Re-exec model

The PM does **not** load the editor in-process. It builds args `--project <dir> --editor` and `std.process.Child.exec`s the same binary, then quits. Crash isolation for free.

**Reference patterns:** Godot `editor/project_manager/project_manager.cpp:559-608` `_open_selected_projects` + `editor/project_manager/project_list.cpp:1003-1011, 1627`. See [`engine-references.md`](../../docs/engine-references.md) § Godot · Project Manager.

**Spec:** [`specs/project-manager.md`](../../docs/specs/project-manager.md).

## Layering rule

`tools_enabled = true` only. May import `core/`, `platform/`, `ui/` (for the ImGui surface), and the libs adapter for ImGui. MUST NOT import `editor/`, `scene/`, `modules/`, or any gameplay code — the PM stays light and crash-isolated.
