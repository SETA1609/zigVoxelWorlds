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

zVoxRealms uses a **two-tier mod runtime** with a **per-project trust root** (decision 2026-05-26):

| Tier | Runtime | Eligibility | Perf | When |
| --- | --- | --- | --- | --- |
| **Native plugin** | `dlopen` / `LoadLibrary` against the stable C ABI | **Only mods packaged as `.zvxmod` and Ed25519-signed by the project publisher's key** (the owner of the *game*, not of the engine) | ~native | The shipped project itself, official DLC, owner-curated extra content. NOT third-party mods, no matter who made them. |
| **WASM sandbox** | WAMR (`libs/zig-cpp-wasm-stack-adapter/`) against a curated Host API | **Everything else** — Workshop mods, sideloaded mods, any content using the public SDK to write mods, anything without a valid publisher signature | ~50–70% of native | Default + only tier for third-party content |

### Trust model: the *project* is the trust unit, not the *engine*

Each game built on zVoxRealms is its own trust domain. The engine itself doesn't ship with any baked-in publishing key; the game's **launcher binary** does. Specifically:

- **Project publisher** (the owner of the game — the developer, not the zVoxRealms maintainer) generates an Ed25519 keypair once via `zvox-keygen`. Private key stored offline / in CI secrets / on hardware token. Public key tracked at `<project>/publisher.pub`.
- **At export time** (per [`src/editor/export/`](../../src/editor/export/README.md)), the export pipeline reads `publisher.pub` and bakes it into the launcher binary as `const PROJECT_PUBKEY: [32]u8 = …;`. The launcher's mod loader uses that baked pubkey to verify signatures.
- **Each shipped game has its own pubkey.** Game A's launcher trusts only Game A's publisher; Game B trusts only Game B's. Mods crossing between games never carry trust.

### What gets signed by the project publisher

- The **base project** (the game itself — its `libzvox-runtime.so` + the official PCK)
- **Official DLCs** — paid or free expansion content
- **Owner-curated extra content** — community content the publisher has reviewed and adopted as official

### What does NOT get signed

- **SDK-built mods from anyone except the project publisher** — even if the modder is reputable, even if they use the project's mod SDK
- **Workshop mods** — by definition third-party, always WASM-sandboxed
- **Sideloaded community mods** — same

The pattern: anything **the project publisher shipped or adopted** = signed, native tier. Anything **the mod community produced** = WASM, regardless of source. The publisher's choice on what to officially adopt is the only path to native-tier trust.

**Hard rule:** the launcher has exactly ONE trust root — the project publisher's Ed25519 public key compiled into the launcher binary at export time. Binary classification: signed-by-publisher OR sandboxed. **No plaintext "trusted publishers" file, no TOFU, no per-user trust override** — any of those would be text-editable and defeat the sandbox entirely.

### WASM-sandbox tier — hard rules

1. **WASI is NOT exposed inside the sandbox.** WAMR optionally provides WASI (filesystem, env, network, process). For mod sandboxes this MUST stay disabled. The mod sees only the engine's curated Host API — a subset of the C ABI documented in [`specs/c-abi.md`](c-abi.md).
2. **Resource limits per mod** — non-negotiable:
   - **CPU budget per game tick** — default 1 ms per mod per 16 ms tick; mod exceeding budget gets paused with a "mod X is consuming too much time, disable?" prompt
   - **Memory cap** — default 64 MB per mod's linear memory; configurable per project
   - **Network access** — default DENY. Per-mod opt-in for HTTPS GET to specific allowlisted domains (e.g. asset CDN); never raw socket.
   - **Filesystem access** — default DENY. The Host API provides scoped read/write to the mod's own data directory only.
3. **No host-process exit** — `proc_exit` / abort intrinsics are intercepted; killing the mod doesn't kill the game.
4. **Deterministic by default** — for multiplayer + replays, the sandboxed mod must be deterministic. WAMR's interpreter mode is deterministic; AOT mode needs verification per platform.

### UI rules for the mod manager

- Show clearly when a mod is **native plugin** vs **WASM sandbox** in the browser
- Warn on first activation of a native-plugin mod: "This mod includes a native plugin. Native mods can execute arbitrary code on your machine. Only enable mods you trust."
- Once accepted per-mod (native), don't warn again until version change
- For WASM-sandboxed mods: no warning needed. They're sandboxed by definition.
- Workshop subscriptions: Steam's reputation system filters most badness; show subscriber count + rating prominently. Default tier for Workshop mods is **WASM-sandbox** unless the publisher is signed by you.

### Why the two-tier model

- **Cross-platform mod packaging** — a `.wasm` mod ships one binary that runs on Linux/Windows/macOS/Android. Native mods need four builds.
- **Trust spectrum** — content the project publisher signs (or signs adopted community content) runs at native speed; community mods run sandboxed regardless of reputation
- **Reputation rebuilds** — even if a Workshop mod turns malicious, the worst it does is exit; can't exfil data, can't pivot to host machine, can't crash other players

### Signing pipeline (per-project)

1. **One-time keygen** — project publisher runs `zvox-keygen --out publisher` on an offline machine. Produces `publisher.pub` (commit to repo) + `publisher.priv` (NEVER commit; store on hardware token or CI secret).
2. **Publisher pubkey baked into the launcher at export.** The export tool (per [`src/editor/export/`](../../src/editor/export/README.md)) reads `publisher.pub` and inlines it as `PROJECT_PUBKEY` in the generated launcher source before compilation. Editing the launcher binary after export to change the key is technically possible but breaks code signatures on Windows / macOS / Steam, so the trust boundary holds on shipped builds.
3. **Signing DLCs / extra content** — publisher runs `zvox-sign --key publisher.priv content/ --out my-dlc.zvxmod` on their secure-key machine. Produces a `.zvxmod` archive with manifest + content + Ed25519 signature over SHA-256(manifest + content tree).
4. **Verification at install / load** in the shipped launcher — compute hash, verify signature against the baked `PROJECT_PUBKEY`, route to native tier on success or WASM tier on any failure (no signature / bad signature / modified contents). Failure is silent — the content just runs sandboxed.

### Revocation

- **Compromised publisher key** — ship a launcher update (engine + new pubkey baked in). Old-key-signed content stops being trusted.
- **Specific bad content** — launcher update can include a revocation list of signature hashes; affected content falls back to WASM tier or gets blocked entirely.
- This is "limited revocation" — no online CRL service. Acceptable for indie scope.

### Editor-time vs launcher-time

- **Editor** (developer environment): no signature verification. The developer IS the publisher; everything in their own project tree is trusted by definition. If they want to playtest the signing flow, they can run `zvox-sign` locally and pass the signed `.zvxmod` to the launcher's mod loader path.
- **Exported launcher** (player environment): signature verification enforced. The baked `PROJECT_PUBKEY` is the only trust root.

Reference precedent: Luanti's Lua sandbox is the genre standard for sandboxed modding; the per-project signing model resembles Steam's per-publisher signed-binary approach combined with WASM's cross-platform sandbox. WASM is the modern multi-language equivalent to Lua (mod authors can write Rust, C++, AssemblyScript, Zig — all compile to WASM).

## Open decisions

- Workshop API rate limits — caching strategy
- Mod conflict resolution — automatic (last-wins) vs always-prompt
- Hot-enable / hot-disable in running game vs requires restart (probably requires restart for v1.0; hot-toggle is post-1.0)
- Mod-saving — does the save bundle a copy of the mod data, or just a reference? (Reference, with version pinning)

## Milestone

Phase 14 (Modding). Install three third-party mods via the in-engine Workshop browser → mod manager UI shows them → enable two → start a new save → save loads next session with the same two mods → unsubscribe one mod → on next launch, save shows "mod missing" warning with clean recovery path.
