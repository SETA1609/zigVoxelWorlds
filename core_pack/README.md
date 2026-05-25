# `core_pack/`

> The base game / project template that ships with the engine. A real project per the project-layout convention — has a `project.toml`, `assets/`, and `data/` — but it lives inside the engine repo so the engine has something to load when started without `--project`.

## Two roles

1. **First-run experience.** A user who downloads the engine binary and opens it with no project should see something — `core_pack/` is what `main.zig` loads as a fallback target. Eventually this may host the small demo level for promo + onboarding.
2. **Project template.** When the Project Manager's "Create new project" wizard runs, it copies the `core_pack/` tree as the starting skeleton, then renames + edits the new `project.toml`. The user can opt into or out of including specific data subtrees (skills, recipes, …).

## What lives here

- [`project.toml`](project.toml) — the project manifest (TBD; modules enabled, version, target platforms)
- [`assets/`](assets/README.md) — minimal asset placeholders (a few textures, one glTF, a glyph atlas seed)
- [`data/`](data/README.md) — baseline TOML data (a few skills, a starter perk, the 9 color-magic effect roots, recipe set)

## What does NOT live here

- Engine source code (lives under `src/` and `modules/`)
- Adapter / library code (lives under `libs/`)
- Per-user project files (those live wherever the user creates them; tracked in `$XDG_DATA_HOME/zvoxrealms/projects.cfg`)

## Why not just an empty placeholder

The structural alternative — ship the engine with no template, force the user to construct a project from scratch — is what Godot does, and it produces a chilly first-run. zVoxRealms ships a minimal-but-real `core_pack` so:

- `main.zig`'s no-arg fallback can hand the user *into* a world rather than into the Project Manager
- The "Create new project" wizard has a known-good source tree to copy from
- CI can exercise the full load-a-project path against the same content the user sees

The pack stays minimal — it is not the engine's "demo game", just enough content to prove the loader works end-to-end.
