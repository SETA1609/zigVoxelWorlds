# `core_pack/assets/`

> Minimal asset placeholders for the engine's first-run + "Create new project" template.

## Planned contents

- `textures/` — a small set of placeholder textures (UV grid, voxel-block defaults, missing-texture stand-in) — PNG, will bake to KTX2 via `src/editor/import/texture.zig`
- `models/` — one or two glTF models for the starter scene (e.g. a placeholder player capsule + a chest prop)
- `audio/` — a footstep clip + a UI-confirm clip (the absolute minimum to prove the audio path)
- `shaders/` — none initially; the engine ships built-in shaders inside `src/backends/vulkan/`, not here
- `fonts/` — one font in the engine's preferred chain (TBD which open license — likely Inter or Source Sans Pro)
- `ui/` — a tiny RML + RCSS demo screen the runtime can render to prove the UI stack
- `.assetdb.toml` — GUID ↔ source-path map for everything above

## Notes

- Asset content is **owner-supplied** during Phase 0–12. No Claude-generated binary content; if a placeholder is needed for a CI test, the owner provides it.
- The `.import/` baked output is git-ignored — runtime reads from there, never from `assets/`.

See [`project-structure.md`](../../docs/project-structure.md) § Project layout for the per-project asset layout.
