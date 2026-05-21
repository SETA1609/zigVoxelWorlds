# Save UX Spec

> Save UI, autosave behavior, slot management, migration, Steam Cloud sync. The *format* is in [`ARCHITECTURE.md`](../ARCHITECTURE.md) + Phase 13; this is the *user-facing* layer. Gap: [`gaps.md` § 2.2.C + § 3 #17](../gaps.md). Reference patterns: [`gap-references.md` § 2.2.C](../gap-references.md).

## Scope

Players need a save UI that's at parity with mid-tier indie games — multiple slots, autosave indicators, save migration on patches, Steam Cloud sync, corruption recovery.

## Save slot UI

In-game save / load menu (UI per [`specs/ui.md`](ui.md)):

- Scrollable list of slots
- Per-slot card: screenshot thumbnail (auto-captured at save time), playtime, scene name, in-game date, optional player-set name
- Sort by: most recent, alphabetical, custom order
- Actions: load / overwrite / delete / rename
- Cloud-sync icon per slot (synced / pending / conflict)

## Save slots

- **Manual slots** — unlimited, user-named
- **Quicksave slot** — 1 dedicated, overwritten on F5
- **Autosave slot** — rotating ring of 3 (overwrite oldest on each autosave)
- **Crash-recovery slot** — special slot written before risky operations (mod load, network transition)

## Autosave behavior

Trigger conditions:

- Scene transition (entering a dungeon, fast-travel)
- Major gameplay milestones (quest completion, level-up, boss defeat) — emitted via [`specs/events.md`](events.md)
- Time interval (configurable: default 5 minutes; off / 1 / 5 / 15)
- Just before risky operations (mod activation, host disconnect)

Visual indicator:

- Spinning save icon in HUD corner (3 seconds, then fades)
- Toast notification on completion
- Non-blocking — game continues; save runs on a worker thread

## Save migration

Save format is versioned per-section (see [`ARCHITECTURE.md` Cross-Cutting Concerns](../ARCHITECTURE.md)). On engine update:

1. Load detects `save_version < engine_version`
2. Runs registered migration functions in version order
3. Writes the migrated save back (or to a new slot to preserve original)
4. UI shows "Save migrated from v0.4 → v0.5" notice

Migration functions are per-section; each engine release that breaks a section ships its migration.

## Steam Cloud sync

When `-Dsteam=true`:

- Each save slot syncs to Steam Cloud (per [`engine-vs-game.md`](../engine-vs-game.md))
- On launch, check Cloud vs local; if Cloud is newer → prompt to use Cloud
- Conflict UI: both saves' screenshot + timestamp; user picks
- Quota awareness: Steam Cloud has per-game quotas; warn before exceeding

## Corruption detection + recovery

- Each save section has a checksum (CRC32 or Blake3 short hash)
- On load, validate checksums
- On mismatch:
  - Try to load partial save (sections that pass checksum)
  - Show recovery dialog: "Save partially corrupted. Recover what's intact?"
  - Suggest the most recent autosave / crash-recovery slot as fallback

## Async save path

- Game state snapshot (in-memory) happens immediately (frame-blocking, ~ms-scale)
- Serialization + disk write happens on a worker thread
- Save isn't "done" until the worker finishes; UI indicator reflects this

## Borrowed patterns

[`gap-references.md` § 2.2.C](../gap-references.md):

- Unreal `SaveGame.h` + `SaveGameSystem.h` — slot-based architecture + async write
- Godot leaves save UI to game code; reference community patterns for the UI shape

## Settings exposed to players

- Autosave interval (off / 1 / 5 / 15 min)
- Autosave on quest complete (toggle)
- Crash recovery (toggle)
- Cloud sync (toggle, when Steam build)
- Max slots displayed
- Save folder path (advanced — for moving saves between machines)

## Open decisions

- Save file location — `$XDG_DATA_HOME/zvoxrealms/saves/<project_id>/` cross-platform
- Screenshot resolution (256×144 thumbnail; 1024×576 full preview?)
- Save metadata format — leading TOML header before binary payload, or all-binary with a sidecar `.meta.toml`?
- Multiple-character saves — separate per-character or per-save (the four target games handle this differently — start with per-save)

## Milestone

Phase 13 (Export pipeline + save format). UI built in Phase 12. Player can: save manually, see autosaves accumulating, load a save from a previous session, run the game after a patched engine (migration runs cleanly), recover from a corrupted save section.
