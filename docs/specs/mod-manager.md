# Mod Manager UX Spec

> In-game mod browser, enable/disable, dependency resolution, Workshop integration. Without this, modding is "edit config files by hand" — which violates the "modded as easily as Minecraft" vision. Gap: [`gaps.md` § 2.2.D + § 3 #22](../gaps.md). Reference patterns: [`gap-references.md` § 2.2.D](../gap-references.md).

## Scope

The user-facing layer on top of the mod loader. Phase 14 implements loading + ABI; this spec covers the UI players interact with.

## Mod browser UI

Standalone screen accessible from the main menu and from the Project Manager:

- List of installed mods (from `<project>/mods/` and Steam Workshop subscriptions)
- Per-mod card: name, version, author, description, icon, enabled checkbox
- Sort / filter: enabled / disabled / by author / by tag
- Detail panel on selection: full description, dependencies, conflicts, changelog
- Actions: enable / disable, update, uninstall, view in file system, view on Workshop (if subscribed)

## Dependency graph view

Visual dependency graph for the currently-enabled set:

- Each mod = node
- Required dependencies = solid edges
- Optional integrations = dashed edges
- Conflicts = red highlighted edges
- Missing dependencies = red node
- Load order shown left-to-right (or top-to-bottom)

Players can reorder mods that don't have a strict dependency between them.

## Conflict warnings

A mod conflict = two mods modify the same engine resource (voxel ID, recipe ID, etc.) and one will override the other.

UI:

- Detected at mod-load time
- Warning banner: "Mod A and Mod B both define voxel #42; B overrides A. Continue?"
- Per-conflict resolution preference saved (always-A / always-B / always-prompt)

## Per-save mod set

Mods are enabled per-save, not globally:

- On new save: snapshot the currently-enabled mod set into save metadata
- On load: validate that the same mod set is available; warn if mods missing
- Allow loading with reduced mod set; mark the save as "mod-missing" so it can't go back to the full set without explicit user confirm
- Allow loading with extra mods activated since the save; new mods initialize cleanly

## Workshop + DLC integration (when `-Dsteam=true`)

Steam exposes two distinct distribution channels — both need to plug into the same mod loader. See also [`engine-vs-game.md` § 3b TODO](../engine-vs-game.md) for the cross-system summary.

**Steam Workshop** (third-party + free mods):

- In-engine Workshop browser: search / browse / view mod page (description, screenshots, rating)
- One-click subscribe → auto-download via Steamworks UGC API ([`ISteamUGC`](https://partner.steamgames.com/doc/api/ISteamUGC) — `SubscribeItem`, `DownloadItem`, item-installed callback)
- Auto-update on game launch (configurable)
- Unsubscribe → mod removed locally
- Modders publish from inside the engine via `CreateItem` + `SubmitItemUpdate` so they never leave the tool

**Steam DLC** (your paid / free DLC mods sold through the Steam Store):

- Each DLC has its own Steam AppID under the parent game's Store page
- At game launch, query [`ISteamApps::BIsDlcInstalled(appId)`](https://partner.steamgames.com/doc/api/ISteamApps#BIsDlcInstalled) for each known DLC
- If installed → activate the corresponding bundled `.mod` archive (shipped alongside the base game install)
- Unowned DLCs stay inactive; their archive is present but not loaded
- DLC purchase Store events trigger Steam to re-check; engine re-evaluates active mods on next launch

**📌 TODO — research before Phase 14 starts:**

- Cross-platform fallback for non-Steam builds (DRM-free / itch.io / GOG) — likely "DLC = `dlc/` folder; mods = `mods/` folder" with no DRM
- DLC distribution choice — bundle the `.mod` with the base install (gated by `BIsDlcInstalled`), or download-on-purchase via UGC
- Mod-DLC dependency semantics — a free Workshop mod can require an owned paid DLC; UI handles the dependency gracefully

For non-Steam builds: a "mod folder" path that players can `git clone` or unzip mods into.

## Mod load order

Mods are loaded in dependency order (topological sort). Within the same dependency level, alphabetical by default; user can drag-reorder in the UI.

Order affects:

- Override resolution (later mod wins for the same resource ID)
- Patch order (later mod's hooks run after earlier mod's hooks for the same event)

## Mod descriptor — what the UI displays

Each mod's `mod.toml` provides:

```toml
[mod]
name = "Better Loot"
version = "1.2.0"
description = "Replaces vanilla loot tables with more variety and rarer artifacts."
author = "@somemodder"
homepage = "https://github.com/somemodder/better-loot"
icon = "icon.png"             # bundled in the mod
tags = ["loot", "balance", "gameplay"]
engine_compat = ">=0.5, <1.0"

[mod.dependencies]
required = ["base-game"]
optional = ["expanded-economy"]
incompatible = ["loot-overhaul"]
```

## Borrowed patterns

[`gap-references.md` § 2.2.D](../gap-references.md):

- **Luanti is the gold standard.** `$REFS/luanti-custom/builtin/mainmenu/dlg_contentstore.lua` + `dlg_create_world.lua` for mod browsing + per-world mod selection. Read the *UX flow* — list with checkboxes, dependency graph, Workshop-equivalent. Replace Lua-driven UI with TOML-declarative UI ([`specs/ui.md`](ui.md)).
- Godot has no built-in mod system — skip
- Unreal has no built-in mod system — skip

## Security / sandbox considerations

Mods run native code (per [`engine-vs-game.md`](../engine-vs-game.md)). UI must:

- Show clearly when a mod includes native code (vs data-only)
- Warn on first activation: "This mod includes a native plugin. Native mods can execute arbitrary code on your machine. Only enable mods you trust."
- Once accepted per-mod, don't warn again (until version change)
- Workshop subscriptions: Steam's reputation system filters most badness; show subscriber count + rating prominently

## Open decisions

- Workshop API rate limits — caching strategy
- Mod conflict resolution — automatic (last-wins) vs always-prompt
- Hot-enable / hot-disable in running game vs requires restart (probably requires restart for v1.0; hot-toggle is post-1.0)
- Mod-saving — does the save bundle a copy of the mod data, or just a reference? (Reference, with version pinning)

## Milestone

Phase 14 (Modding). Install three third-party mods via the in-engine Workshop browser → mod manager UI shows them → enable two → start a new save → save loads next session with the same two mods → unsubscribe one mod → on next launch, save shows "mod missing" warning with clean recovery path.
