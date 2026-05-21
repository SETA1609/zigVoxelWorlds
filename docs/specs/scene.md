# Scene & Instancing Spec

> The persistent world + dungeon/tower/house instances, wired via `orchestrator.toml`. Roadmap: [`ROADMAP.md` § Phase 9](../ROADMAP.md). Open questions on scene format: [`gaps.md` § 3](../gaps.md).

## Scope

- **Persistent main world** — seed-deterministic baseline + saved deltas (see [save model in ARCHITECTURE](../ARCHITECTURE.md#cross-cutting-concerns))
- **Instanced scenes** — dungeons, towers, houses; load/unload on transition (door, portal, teleport)
- **`orchestrator.toml`** — links scenes to world locations + triggers
- **Hot-reload of scene definitions** — edits in the editor take effect without restart

## Scene contents

Per [`gaps.md` § 3](../gaps.md), each `scene.toml` declares:

- Entity spawn list (positions + component overrides)
- Region AABBs with [edit policies](../ARCHITECTURE.md#cross-cutting-concerns)
- Lighting setup
- Weather state (or "inherit from world")
- Time-of-day override (or inherit)
- Music / ambient audio cues
- Triggers (volumes that fire scripted events)

## Reference patterns

- Godot scene-instancing pattern (the user-facing UX, not the Node/Control hierarchy — see [`engine-references.md` → Godot](../engine-references.md))

## Milestone (from ROADMAP)

Enter a dungeon from the overworld, exit, state persists.
