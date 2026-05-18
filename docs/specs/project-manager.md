# Project Manager Spec

> The Godot-style entry screen: same engine binary opens as PM when no project is given. Roadmap: [`ROADMAP.md` § Phase 11](../ROADMAP.md). Reference pattern: [`engine-references.md` → Godot Project Manager](../engine-references.md). Architecture: [`ARCHITECTURE.md` § Distribution Model](../ARCHITECTURE.md#distribution-model).

## Scope

- Engine binary launched with **no args** → opens the Project Manager window
- Engine binary launched with **`--project <path>`** → opens the editor for that project, skipping the PM
- Project = directory containing `project.toml`
- Opening a project from inside the PM spawns a fresh child process via `std.process.Child` and the PM quits → crash isolation for free

## Project Manager window (ImGui)

- Project list (recents + favorites)
- Buttons: New / Import / Open / Remove / Settings
- New-project wizard: skeleton `assets/`, `.import/`, `project.toml` with sensible defaults

## State persistence

- Recents + favorites in `$XDG_DATA_HOME/zvoxrealms/projects.cfg` (TOML — per [`tech-stack.md`](../tech-stack.md))
- Each entry: `path`, `name`, `last_opened`, `favorite`, `tags`

## `project.toml` (per project)

The project manifest. Schema not yet specified — see [`planning-gaps.md` #1](../planning-gaps.md) and the example block in [`engine-vs-game.md` § 9](../engine-vs-game.md). Minimum fields:

- `[project]` — name, version, engine_compat
- `[modules]` — enable/disable table
- `[scripts]` — language + entry path
- `[export.<target>]` — per-platform options

## Reference patterns

- Godot `editor/project_manager/project_manager.cpp:559-608` — `_open_selected_projects` (re-exec pattern)
- Godot `editor/project_manager/project_list.cpp:1003-1011, 1627` — `projects.cfg` INI format
- Godot `main/main.cpp:217, 2130, 2210` — `project_manager` boot flag fallback

## Milestone (from ROADMAP)

Launch engine binary with no args, see PM; create a new project, open it, exit, reopen from recents.
