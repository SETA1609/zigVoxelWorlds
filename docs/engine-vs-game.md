# Engine vs Game: What Ships Where, and Which Libs Go Where

> Scope: [`vision.md`](vision.md) + [`mission.md`](mission.md). Layers: [`ARCHITECTURE.md`](ARCHITECTURE.md). Stack: [`tech-stack.md`](tech-stack.md). Adapters catalog: [`external-libs.md`](external-libs.md). Layout: [`project-structure.md`](project-structure.md). Licensing: [`licensing.md`](licensing.md).

**Licensing summary:** the engine code (everything in `src/`, `modules/`, `adapters/`) is **Apache 2.0**. When a project is exported, the engine code shipped inside `libzvox-runtime.{so,dll}` remains Apache 2.0 (attribution required in the game's distribution). The project's scripts (`libgame.{so,dll}`) and assets (`game.pck`) are the game owner's IP under their own EULA. See [`licensing.md`](licensing.md) for the full strategy.

zVoxRealms ships in three forms:

1. **The Engine** — a single binary (`zvoxrealms`) that opens as either the Project Manager or the editor for a project. Distributed to **developers** (anyone making a game or a mod).
2. **A Game** — what a developer produces by clicking "Export" on a **game project**. Distributed to **players**. Internally: launcher + `libzvox-runtime` + PCK (with the **core mod** bundled in) + a **modkit** (free, for third-party modding).
3. **A Mod** — what a developer produces by clicking "Export" on a **mod project** (DLC made by you, or a third-party mod). Distributed to **players who own the target game**. Internally: a single `.mod` archive loaded by an existing game.

**The "everything is a mod" model.** The vanilla game itself is structurally the first mod loaded — the **core mod**. Same `mod.toml`, same ABI, same load order rules. No privileged "this is the base game" code path. Industry precedent: Bethesda's `Skyrim.esm` master file + `.esp` plugins; Luanti's subgames; id Tech's IWAD + PWADs.

This document is the source of truth for which code, which libraries, and which assets belong on which side of that line.

---

## 1. The three-layer lifecycle

```text
SOURCE (in repo / project)        ENGINE BINARY (distributed to devs)        EXPORTED GAME (distributed to players)
─────────────────────────────     ───────────────────────────────────       ──────────────────────────────────────
src/core/                  ─┐                                            ┌─ launcher (tiny stub, ~MB)
src/servers/                │                                            │
src/backends/               │── statically linked into ──────────────────┤── libzvox-runtime.{so,dll}
src/scene/                  │     zvoxrealms binary                      │   (per-project tree-shaken
src/platform/               │                                            │    dynamic library)
modules/<name>/ (all)       │                                            │
adapters/*/ (all)           │                                            │
                            │                                            │
src/project_manager/        │── statically linked into ──────────────────│   NOT shipped in game
src/editor/                 │     zvoxrealms binary (TOOLS_ENABLED only) │
src/importers/              │                                            │
src/export/                 │                                            │
                            │                                            │
<project>/assets/ (source)  │── baked at import time by editor ──────────│── <project>/.import/ → game.pck
<project>/scripts/ (Zig+C++)│── compiled per-project via zig build-lib ──│── libgame.{so,dll} dlopen'd at startup
<project>/mods/             │── auto-discovered, loaded at runtime ──────│── shipped alongside game, dlopen'd
<project>/project.toml      │── manifest read by editor + exporter ──────│   NOT shipped (replaced by linked module set)
```

The engine binary is **fat** (every module compiled in, plus editor + importers + Project Manager). The exported game is **slim** (only modules the project enables, plus its assets and scripts, no editor code).

---

## 2. What's part of the engine (and only the engine)

Code in this list is in the engine binary, used at design time, but **never** ships with a game.

| Subsystem | Location | Why engine-only |
| --- | --- | --- |
| Project Manager | `src/project_manager/` | Only relevant when authoring projects |
| Editor panels | `src/editor/panels/` | Voxel brush, recipe editor, etc. — design-time tools |
| Asset importers | `src/importers/` | Source → baked transforms run at import time |
| Export pipeline | `src/export/` | Produces the shipped game |
| Embedded Neovim host | `src/editor/code_editor/` | Code editing happens in the editor, not in the game |
| Module enable/disable UI | `src/editor/panels/modules.zig` | Author-time decision |
| Hot-reload watcher | `src/editor/hot_reload/` | Recompiles scripts + re-bakes assets during dev |
| Live preview | `src/editor/viewport.zig` | Editor viewport, gizmos |
| Asset DB writer | `src/editor/assetdb_writer.zig` | Writes `<project>/assets/.assetdb.toml` GUIDs |
| `TOOLS_ENABLED` code paths everywhere | comptime-gated | Stripped by Zig dead-code elimination in non-tools builds |

Enforcement: a `build.zig` import-graph check rejects builds where runtime modules import from `editor/`, `project_manager/`, `importers/`, or `export/`.

---

## 3. What's part of every game (shared runtime)

Code in this list is in both the engine binary (used by editor playtest) **and** every exported game (statically linked into `libzvox-runtime.so` per project, only if the project's manifest enables it).

| Subsystem | Location | Notes |
| --- | --- | --- |
| Platform layer | `src/platform/` | Window, input, file I/O, threading, process spawning |
| Core | `src/core/` | Allocators, math, logging, handle table, TOML parser |
| RenderServer + Vulkan backend | `src/servers/render_server.zig` + `src/backends/vulkan/` | Always needed |
| Scene + ECS | `src/scene/` | Archetype ECS, world, chunks, entities |
| Audio | `modules/audio/` | Optional — disabled for silent games |
| Voxel core | `modules/voxel_core/` | Optional — disabled for non-voxel projects (rare, but possible) |
| Physics | `modules/physics_jolt/` | Optional — disabled for projects that don't need physics |
| Networking | `modules/multiplayer/` | Optional — disabled for singleplayer-only games |
| Modding loader | `src/modding/` | Always shipped if mods are enabled in `project.toml` |
| Steam Workshop client | `modules/steam/` | Optional — only with `-Dsteam=true` |

The exact subset is decided at export time by reading `<project>/project.toml`'s `[modules]` table.

---

## 3b. Project types — game vs mod

Two project shapes the editor recognizes. Declared in `project.toml` via `[project] kind = ...`.

### Game project (`kind = "game"`)

Self-contained game. Contains a **core mod** (the vanilla gameplay rules + content) + scripts + assets. Exports to a runnable game.

```text
voxelrpg_demo/                       # game project root
├── project.toml                   # [project] kind = "game"
├── core_mod/                      # the vanilla game's content + scripts — structurally a mod
│   ├── mod.toml                   # declares the core mod's id, version, ABI compat
│   ├── data/                      # CSV + TOML game data (skills, recipes, voxel atlas, quests)
│   ├── scripts/                   # Zig + C++ game-logic scripts
│   └── assets/                    # textures, models, audio for the vanilla game
├── modkit/                        # generated at export — what third-party modders consume
│   ├── modkit.toml                # engine version, ABI version, registered content IDs
│   ├── headers/                   # C ABI headers (signature-only, no implementation)
│   ├── sample_mod/                # a "hello world" mod template
│   └── README.md                  # what's moddable + linking instructions
└── export/                        # one-click export output
    └── linux-x86_64/
        ├── voxelrpg_demo            # tiny launcher
        ├── libzvox-runtime.so     # per-project tree-shaken
        ├── voxelrpg_demo.pck        # baked assets (incl. core_mod content)
        ├── modkit/                # bundled OR downloadable separately from Steam
        └── mods/                  # where user-installed mods land
```

### Mod project (`kind = "mod"`)

Additive content for an existing game. References a parent game; exports a single `.mod` archive.

```text
better-loot-mod/                   # mod project root
├── project.toml                   # [project] kind = "mod"; [project.parent_game] id = "voxelrpg_demo" min_version = ">=0.1.0"
├── mod.toml                       # mod manifest (name, version, dependencies, ABI compat)
├── data/                          # additive CSV / TOML — new items, recipes, etc.
├── scripts/                       # mod-side Zig + C++ (against the published C ABI)
├── assets/                        # mod-specific textures / audio
└── export/
    └── better-loot-v1.0.mod       # the shipped archive (PCK-like)
```

When opened in the editor, a mod project's UI shows the **parent game's content read-only** (so you can browse + reference existing IDs) and **your mod's content writable** (additive on top).

### Modkit (auto-generated by game-project export)

A modkit is what third-party modders open in the engine. It's not a project itself — it's the surface area of the game that mods can extend. Generated by the game project's export step alongside the game binary.

The modkit contains:

- `modkit.toml` — engine + ABI versions, list of registered content IDs (voxel IDs, skill IDs, spell IDs, etc.), enabled engine modules
- `headers/` — the C ABI signatures the game exposes (function pointers, struct layouts) — like Skyrim Creation Kit's CK API headers
- `sample_mod/` — a "hello world" mod template the modder can copy + edit
- Optional: a README documenting the modkit version + what's exposed

Modders open a modkit in the editor's "New Mod Project" flow; the editor uses the modkit to populate autocomplete, validate references, and produce mods compatible with that game version.

**Proprietary IP preserved.** The modkit does **not** ship core-mod source scripts. It ships the *interface* (IDs, signatures, headers) — not the *implementation*. Same model as Bethesda's Creation Kit: enables compatible mods without exposing game code.

**Free distribution.** Modkit ships free with the game (Steam download includes both). Engine is already Apache 2.0 (free). Third-party modders need no commercial license to make mods.

### 📌 TODO — Steam Workshop + DLC integration research

Before Phase 14 (Modding) starts, research how mods + DLCs hook into Steam's two distinct systems. Both use Steamworks but through different APIs and Steam-side pages.

| Distribution channel | Steam-side identity | Steamworks API to study | Engine-side flow |
| --- | --- | --- | --- |
| **DLC mod** (made by you, paid) | Its own Steam AppID under the parent game's Store page | [`ISteamApps::BIsDlcInstalled(appId)`](https://partner.steamgames.com/doc/api/ISteamApps#BIsDlcInstalled) | At game launch, query each known DLC AppID; if installed → activate the corresponding bundled `.mod` archive |
| **Free DLC** (made by you, free Store item) | Steam AppID, free to acquire | Same DLC API as above | Same as paid DLC; just priced at 0 |
| **Third-party mod** (community, free) | Workshop item ID under the parent game's Workshop page | [`ISteamUGC`](https://partner.steamgames.com/doc/api/ISteamUGC) — `SubscribeItem`, `DownloadItem`, item-installed callback | Engine downloads via UGC API; mods land in `<game>/mods/`; loaded same as any local mod |
| **Local mod** (sideloaded) | none — file on disk | n/a | Engine auto-discovers in `<game>/mods/` |

Open questions to resolve before Phase 14 (add to [`gaps.md` § 3.7](gaps.md)):

- Cross-platform: how do non-Steam builds (DRM-free / itch.io / GOG) handle DLC + mods? — likely "DLC = mod folders in `dlc/`; mods = `mods/`" with no platform DRM
- DLC bundling: ship the DLC `.mod` archive bundled with the base game install + gated by `BIsDlcInstalled`? Or download on first DLC-purchase via UGC?
- Workshop item dependency declaration — UGC supports per-item tags + dependencies via metadata; design schema
- Mod-DLC interaction — can a free third-party mod require an owned paid DLC? (Yes, common pattern; handle the dependency gracefully)
- Workshop authoring from inside the engine — modders upload via the editor (`ISteamUGC::CreateItem` + `SubmitItemUpdate`) so they never leave the tool

Steamworks SDK is already a planned adapter (with `-Dsteam=true` gate, per [`external-libs.md`](external-libs.md) § 3). Research stays in scope of that adapter's build.

---

## 4. What's part of a specific game (project-only)

These ship with the game but are project-specific. Not in the engine.

| Thing | Where in source | Where in shipped game |
| --- | --- | --- |
| Hand-authored game data (skills, perks, spells, recipes, scenes) | `<project>/assets/data/*.toml` | Baked into `game.pck` as validated binary tables |
| **Edit-policy TOML** (per scene / region: full / none / voxel-type / coord-range / tag / script) | `<project>/assets/data/scenes/*.toml` + `orchestrator.toml` | Drives `VoxelServer` mutation checks. See [`ARCHITECTURE.md` → World editability policy](ARCHITECTURE.md#cross-cutting-concerns) |
| Source textures, models, audio, shaders | `<project>/assets/{textures,models,audio,shaders}/` | Baked into `game.pck` as KTX2/mesh-binary/Opus/SPIR-V |
| Asset GUID database | `<project>/assets/.assetdb.toml` | Embedded into `game.pck` so handles resolve |
| Game scripts (Zig / C++) | `<project>/scripts/` | Compiled to `<project>/.import/scripts/libgame.{so,dll}` and shipped alongside the runtime |
| Mods bundled with the game | `<project>/mods/` | Shipped as `<game>/mods/<name>/` next to the launcher |
| Game icon, splash, metadata | `<project>/project.toml` + `<project>/branding/` | Launcher resources |

---

## 5. Scripting

Game logic is written in **Zig (preferred) or C++** — both AOT-compiled. No interpreted scripting (no Lua, no GDScript, no Python).

**Why:** Zig + C++ scripting matches the "Zig-first" principle, avoids a separate interpreter + GC, and gives game code engine-tier performance. Both compile via the bundled Zig toolchain (`zig cc` for C++), so the developer needs no external toolchain.

**Mechanism — exactly the same as mods:**

- `<project>/scripts/` directory contains Zig or C++ source
- Editor watcher detects edits → invokes `zig build-lib -dynamic` (Zig) or `zig c++ -shared` (C++) → produces `<project>/.import/scripts/libgame.{so,dll}`
- Engine runtime `dlopen`s `libgame.{so,dll}` at startup
- On change in editor playtest: `dlclose` + recompile + `dlopen` = hot-reload
- Scripts use the same **stable `extern "C"` ABI** as native mods — they ARE mods structurally, just shipped with the game by default
- ABI surface is versioned; engine refactors must not break it or shipped games break

**Layout:**

```text
<project>/scripts/
├── game.zig            # Entry point, registers gameplay systems with engine
├── systems/
│   ├── quest_log.zig
│   └── weather.cpp
└── components/
    └── inventory.zig
```

The contract mirrors the engine module contract from [`project-structure.md`](project-structure.md#module-contract): a script package exposes `initialize(level)` / `uninitialize(level)` and is loaded at the appropriate init level after engine modules.

---

## 6. Code editor panel: Neovim

The engine's code-editing panel embeds **Neovim** as a child process. Chosen for speed and modal-editing snappiness.

- **Neovim ships bundled** with the engine at `bundled/nvim/{bin,share,lib}/`. Always available — no "is nvim installed?" UX papercut.
- Editor spawns `bundled/nvim/bin/nvim --embed` via `std.process.Child`, communicates over msgpack-RPC on stdin/stdout
- **Rendering**: GUI grid embed via `redraw` notifications. No terminal in the path. Reference: [neovide](https://github.com/neovide/neovide), [goneovim](https://github.com/akiyosi/goneovim), [gnvim](https://github.com/vhakulinen/gnvim).
- **User config respected**: `nvim --embed` picks up `~/.config/nvim/init.lua` automatically. Power users get their plugins; new users get a sane default config we ship at `bundled/nvim/share/nvim/sysinit.vim`.
- **Last-resort fallback**: ImGui text edit if the bundled binary is missing on disk or subprocess spawning is blocked (sandboxed environments).
- License: Neovim is Apache 2.0; bundling allowed with attribution in the engine's `LICENSES.md`.
- CI requirement: a release step fetches platform-specific Neovim tarballs (Linux x86_64, Windows x86_64, macOS) from the upstream Neovim releases and bundles them into the engine archive.
- Cross-panel integration: clicking a `file:line` in the runtime debug log (§7) routes through msgpack-RPC `:edit <file>` + `:<line>` to jump cursor here
- Engine-only (`src/editor/code_editor/`). Shipped games do not embed Neovim — they don't ship the editor at all.

## 6b. Observability — where each tool ships

Three telemetry concerns, three places they ship:

| Tool | Purpose | Ships in editor binary? | Ships in `libzvox-runtime`? |
| --- | --- | --- | --- |
| **Tracy** | Dev-time frame profiling | Yes (when `-Dtracy=true`); never in release | No — never shipped to players |
| **libghostty** + log multiplexer | Editor playtest debug-output panel (§7) | Yes, editor-only | No — shipped games log to plain stdout/stderr |
| **OpenTelemetry → Grafana stack** | Production telemetry from dedicated servers | No (editor uses Tracy + libghostty instead) | Yes, **only when project's `project.toml` enables `[modules.telemetry]`**, and only active when launched with `--otlp-endpoint <url>` |

The architectural move that makes this work: all three flow through stable abstract sinks in `src/core/` (`profile.zig`, `log_sink.zig`, `metrics.zig`) defined in Phase 2. Backends are chosen at startup from a build-time-enabled set; gameplay code calls the sinks without knowing or caring which backend is live.

This means a player client (singleplayer or 4-player co-op) ships with **none** of the three — just stdout + a local log file. A dedicated server ships with OTel-only. The editor binary ships with Tracy + libghostty + (optionally) OTel for testing the server path.

See [`tech-stack.md` → Observability](tech-stack.md#observability) for the tool-selection rationale and references.

## 7. Runtime debug output panel: libghostty

Separate panel from §6. During playtest, the editor needs to show game stdout/stderr, log messages, stack traces, errors, and warnings — the Godot Output / Unity Console / Unreal Output Log role. Embeds **libghostty** for the text-stream rendering surface, with an engine-side multiplexer on top.

- Editor panel hosts [libghostty](https://github.com/ghostty-org/ghostty) (the embeddable form of the Ghostty terminal emulator, Zig-native, GPU-accelerated)
- Engine writes structured logs (level + channel + timestamp + payload) to a log sink
- Log sink renders the filtered stream into the Ghostty surface as ANSI-colored text (errors red, warnings yellow, info default)
- ImGui chrome above the Ghostty pane: filter by level, channel tabs (Render / Physics / Network / ECS / Modding), pause / clear / save-to-file
- Click on a `file:line` in the stream → message handed to §6 to open the file at that line
- Crash dumps: piped to both Ghostty and `<project>/.import/crash_<timestamp>.log` on disk
- Engine-only (`src/editor/playtest_log/`). Shipped games log to plain stdout / stderr / log files; players don't ship Ghostty.
- Wrapped behind `src/core/log_sink.zig` so the backend can swap (libghostty's embedding API is post-1.0, some refactor risk over the next year — keeping the wrapper means swapping costs nothing)

---

## 8. Library categorization

Three categories, with clear rules.

### 8.1 Needs an adapter wrapper

Heavy C++ libraries (or complex C libs we want isolation from). Each gets its own sub-project under `adapters/<name>/` with its own `build.zig` and an `extern "C"` boundary. Zig never sees raw C++.

| Library | Purpose | Reason it needs an adapter |
| --- | --- | --- |
| **Jolt Physics** | Character controllers, ragdolls, destructible bodies, vehicles | C++ — must wrap. Heavy API surface |
| **VMA** (Vulkan Memory Allocator) | GPU memory management for Vulkan | C++ |
| **Dear ImGui** | Editor tools, dev panels | C++ — large API surface; we want to insulate the Zig side |
| **glslang / slangc** | GLSL/HLSL/Slang → SPIR-V compilation | C++ |
| **KTX2 / Basis Universal** | GPU-compressed textures (BC7 / ASTC), mipmaps | basisu is C++ |
| **GameNetworkingSockets** | Reliable UDP transport (one of two candidates) | C++ |
| **Steamworks SDK** | Steam Workshop, achievements, cloud saves (conditional) | C++, vendor SDK |
| **FlatBuffers** or **Cap'n Proto** | Zero-copy structured save data (one to be picked) | C++ |
| **Tracy** | High-precision CPU/GPU profiling | C++ — has its own client library |

Pattern: `adapters/<name>/build.zig` builds the C/C++ as a static lib; `adapters/<name>/adapter.h` exposes only `extern "C"` functions; `adapters/<name>/adapter.zig` exposes a typed Zig module on top.

### 8.2 Directly added (no adapter wrapper)

Either pure-Zig dependencies or single-header / small C libraries that are trivial to use via `@cImport` without a dedicated adapter sub-project.

| Library | Purpose | Why direct |
| --- | --- | --- |
| **vulkan-zig** (or volk via `@cImport`) | Vulkan loader + bindings | Zig-native bindings exist; loader is a thin C lib |
| **TOML parser** (`zig-toml` or similar) | Hand-authored data (skills, recipes, scenes, manifests) | Zig-native package available |
| **GLFW** | Window + input (transitional; replaced later by pure Zig platform layer per [`tech-stack.md`](tech-stack.md#windowing--input)) | Pure C, used behind a thin Zig `platform/` module |
| **cgltf** | glTF 2.0 model loading | Single-header C — `@cImport` directly |
| **miniaudio** | Audio playback / mixing / streaming | Single-header C |
| **ENet** | Reliable UDP (one of two candidates) | Plain C, small API; use via `@cImport` |
| **zstd** / **LZ4** | Chunk + save compression | Plain C libs |
| **Math** (vectors / matrices / quats) | Engine math | Pure Zig, hand-written or `zig-gamedev/math` |
| **Hash maps, ArrayList, etc.** | Containers | `std` |
| **All first-party modules** under `modules/<name>/` | Engine subsystems | Pure Zig |
| **Game scripts** | Game logic | Zig (or C++ via `zig c++`, with its own thin per-script `extern "C"` shim) |

The split between 8.1 and 8.2 is **"is this C++ or complex enough to deserve isolation?"** If yes → adapter. If no → direct.

### 8.3 Dynamically loaded at game runtime

Libraries / binaries that the **shipped game** loads dynamically, not statically.

| Thing | When loaded | Notes |
| --- | --- | --- |
| **`libzvox-runtime.{so,dll}`** | At game launch, by the launcher | The whole runtime — produced per-project by the export pipeline. Contains only modules the project enabled |
| **`libgame.{so,dll}`** (game scripts) | At game launch, by the runtime | Project-specific Zig + C++ scripts |
| **Mod plugins** (`mods/<name>/plugin/lib<name>.{so,dll}`) | At game launch + on user enabling mods | Third-party `extern "C"` plugins discovered in `mods/` |
| **Steamworks** | Lazily, only if Steam Workshop is invoked | dlopen'd on demand; non-Steam builds don't need the SDK present |

What's **not** in this list: the C/C++ adapter libs from §8.1. Those are statically linked into `libzvox-runtime.{so,dll}` itself. The runtime is the unit of dynamic loading; individual adapters are not loaded à la carte.

---

## 9. How `project.toml` drives all of this

A project's `project.toml` is the single declaration that determines what ships:

```toml
[project]
name = "voxelrpg_demo"
version = "0.1.0"
engine_version = "0.1.0"

[modules]
voxel_core   = true
physics_jolt = true
audio        = true
skills       = true
perks        = true
magic        = true
crafting     = true
inventory    = true
multiplayer  = false  # singleplayer game
steam        = false  # not shipping on Steam

[scripts]
language = "zig"      # or "cpp" or "both"
entry    = "scripts/game.zig"

[export.linux_x86_64]
embed_pck = false
icon = "branding/icon.png"

[export.windows_x86_64]
embed_pck = true
icon = "branding/icon.ico"
```

At export time, the pipeline:

1. Reads `[modules]` → resolves transitive dependencies → final module set
2. Reads `[scripts]` → compiles `<project>/scripts/` to `libgame.{so,dll}`
3. Runs `zig build-lib -dynamic` over the module object files → `libzvox-runtime.{so,dll}` (each adapter from §7.1 needed by an enabled module is statically linked into the runtime)
4. Packs `<project>/.import/` → `game.pck`
5. Copies the launcher template binary, names it after the project
6. Bundles `<project>/mods/` (if any) next to the launcher

Result on a player's disk:

```text
voxelrpg_demo/
├── voxelrpg_demo              # Launcher stub (~MB)
├── libzvox-runtime.so       # Runtime, tree-shaken — contains only physics_jolt + voxel_core + audio + skills + ...
├── libgame.so               # Compiled game scripts
├── voxelrpg_demo.pck          # All assets + data, baked
└── mods/                    # Optional, project-shipped mods
```

Nothing from `src/editor/`, `src/project_manager/`, `src/importers/`, `src/export/` is present. No Neovim. No build toolchain. No source files. The game is portable.

---

## 10. Quick reference

| Question | Answer |
| --- | --- |
| Is X part of the engine? | Is it under `src/editor/`, `src/project_manager/`, `src/importers/`, or `src/export/`? Yes → engine-only. |
| Is X part of every game? | Is it under `src/core/`, `src/servers/`, `src/backends/`, `src/scene/`, `src/platform/`, or `modules/<name>/` enabled by `project.toml`? Yes → ships. |
| Does library X need an adapter? | Is it C++ or a non-trivial C API? Yes → §8.1. |
| Does library X get loaded dynamically? | Is it the runtime, a script package, a mod plugin, or Steamworks? Yes → §8.3. Anything else → statically linked into whatever binary needs it. |
| Where do game scripts live? | `<project>/scripts/` source → `<project>/.import/scripts/libgame.{so,dll}` baked. Shipped with the game. |
| How is code edited in the engine? | Embedded Neovim panel (`src/editor/code_editor/`), GUI grid via msgpack-RPC. Fallback ImGui text edit if `nvim` not installed. |
| Where do runtime logs / errors / stack traces show during playtest? | `src/editor/playtest_log/` panel — libghostty surface + engine log multiplexer. Click `file:line` → jumps to Neovim. |
