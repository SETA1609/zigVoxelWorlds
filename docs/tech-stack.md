# Technology Stack Decisions

> Scope authority: [`vision.md`](vision.md) + [`mission.md`](mission.md). Layering: [`ARCHITECTURE.md`](ARCHITECTURE.md).
> Adapter libraries: [`external-libs-catalog.md`](external-libs-catalog.md).

## Language Strategy

- **Zig (0.16+)** is the default for **everything**: engine core, gameplay, tools, build system, mod ABI host.
- **C / C++** is only used when it makes more sense than Zig:
  - A mature, battle-tested library exists (Jolt, VMA, Steamworks) and rewriting in Zig would cost more than it returns.
  - A vendor SDK is only shipped as C/C++ (Steamworks).
  - A standard ABI is required for stability (the mod plugin boundary uses `extern "C"`).
- All C/C++ lives in `src/c/` or `src/cpp/` behind a thin `extern "C"` wrapper. The Zig side never sees raw C++.

If a Zig-native option exists and is roughly comparable, prefer Zig.

## Graphics

- **Vulkan** — explicit GPU control, compute shaders for meshing and (later) GI.
- **VMA** (Vulkan Memory Allocator) — used via C++; Zig port considered later if it matures.
- **volk** or manual Vulkan loader (start with whichever is fastest to integrate).

## Physics

- **Jolt Physics** — best-in-class performance, well-suited to destructible voxel bodies, ragdolls, vehicles, character controllers.
- Built as a C++ library via `build.zig`; exposed to Zig through an `extern "C"` wrapper.

## Windowing & Input

- **SDL3** via the `libs/zig-cpp-platform-stack-adapter/` adapter (decision 2026-05-26 — see [`specs/platform.md`](specs/platform.md) and project memory `project-platform-backend-sdl3`). SDL3 is built through the [`castholm/SDL`](https://github.com/castholm/SDL) `build.zig.zon` dependency (pinned; HIDAPI elected as BSD-3-Clause to stay GPL-free), not a vendored submodule — see [`external-libs-catalog.md` § Building SDL3](external-libs-catalog.md).
- Backend chosen for Android + Steam Deck + future Switch coverage. GLFW was the earlier choice; dropped because no Android support and weaker Wayland / Steam-Input integration.
- The pure-Zig native v1.x migration is **withdrawn**. Maintaining native X11 / Wayland / Win32 / Android backends in pure Zig was always aspirational for a solo team; SDL3 already covers all those platforms with shipped reliability.
- The platform-stack adapter scope expands to include: window + events + input + time + file I/O + native handle getters (as before) **plus** SDL3-provided gamepad (Steam Input mapping), sensor (Steam Deck gyro / mobile IMU), haptic (rumble), clipboard, filesystem paths (XDG / FOLDERID / NSDirectory / Android internal storage), power info, IME / text input, **2D rendering primitives (`SDL_Renderer`)** for the widget kit, and **basic audio (`SDL_AudioStream`)** as the default audio backend. These fold in for free; they would have been per-OS code otherwise.

## Subsystem swap pattern — SDL3 default + dedicated-lib opt-in

For subsystems where SDL3 covers the common-case need but a dedicated library offers richer capabilities, the engine ships **SDL3 as the default + the dedicated library as an opt-in per-project upgrade**. Per project memory `project-subsystem-swap-pattern` (2026-05-26):

| Subsystem | SDL3 default (always available) | Opt-in dedicated lib | Trigger |
| --- | --- | --- | --- |
| **Audio** | SDL3 audio via platform-stack: playback, 2D pan, format conversion | miniaudio via `libs/zig-cpp-audio-stack-adapter/` | `project.toml [audio] backend = "miniaudio"` — for 3D spatial / doppler / reverb |
| **In-game UI** | Engine widget kit at `src/ui/widgets/` (~20 reusable widgets on `SDL_Renderer` + `SDL_ttf` primitives — ALWAYS ships) | RmlUi document layer via `libs/zig-cpp-ui-stack-adapter/` for bespoke RML/RCSS-authored screens | `project.toml [ui] document = true` — links the RmlUi adapter |
| **Other SDL3 features** (gamepad, sensor, haptic, clipboard, paths, IME, power) | SDL3 via platform-stack | n/a — SDL3 is the canonical layer; no richer alternative worth swapping to | always SDL3 |

**The UI case is NOT a single-API swap** — widgets (immediate-mode primitives) and document UI (retained-mode document tree) are different abstractions, not interchangeable backends of one. See [`specs/ui.md`](specs/ui.md) § Two-layer architecture.

**The audio case IS a clean swap** — both backends model the same operations; `src/audio/` exposes one Zig API with capability flags.

## Math

- Hand-written, SIMD-friendly Zig math (preferred).
- `zig-gamedev/math` as a reference / fallback.
- Re-evaluate `mach-glfw` / `mach-vulkan` if they mature.

## ECS

- Custom **archetype-based ECS in pure Zig**.
- No third-party ECS dependency — we want cache layout control and predictable allocations.

## Data Layer (Data-Driven Design)

Four hand-authored formats + binary at runtime. Each format earns its keep for a specific shape of content; no YAML.

- **TOML** — tree / graph / config content: quests, scenes, dialog trees, mod manifests, behavior trees, `project.toml`, scene definitions, editability policies, UI layouts. Comments + nested structure + named keys. Spec: [toml.io](https://toml.io/en/).
- **CSV** — tabular game data: items, weapons, armor, recipes, voxel atlas, spell effects, loot tables, NPC stat blocks. Spreadsheet-native (Excel / LibreOffice); bulk-edit + autofilter + formula-based generation are huge solo-dev productivity wins. Line-oriented git diffs. Established indie convention (Stardew Valley, RimWorld, FTL). See [`specs/content-authoring.md`](specs/content-authoring.md) for the rationale + § "Why CSV not SQLite".
- **`.po`** (gettext) — translations. Industry standard for FOSS games + GNOME / KDE / Mozilla. Mature tooling: [Poedit](https://poedit.net/), [Weblate](https://weblate.org/), Crowdin, Transifex. Plurals + `msgctxt` + comments + fuzzy markers are first-class. See [`specs/localization.md`](specs/localization.md).
- **Binary** — everything machine-authored or runtime-loaded: save states, replay logs, voxel chunk deltas, baked meshes, baked textures (KTX2), audio. Custom packed format for chunk deltas; **FlatBuffers** ([flatbuffers.dev](https://flatbuffers.dev/)) or **Cap'n Proto** ([capnproto.org](https://capnproto.org/)) for structured save data where zero-copy reads + schema evolution earn their cost.
- **JSON** — only where an external standard or third-party tool requires it:
  - **glTF 2.0** model files (glTF *is* JSON + binary buffers — [khronos.org/gltf](https://www.khronos.org/gltf/))
  - **Asset import metadata** (per-asset `.import` sidecars; JSON Schema gives us free validation in editors — [json-schema.org](https://json-schema.org/))
  - **Build manifests / CI artifacts** where tooling expects JSON

Rules of thumb — match format to content shape:

- **Tabular** (every row has the same columns) → CSV
- **Tree / nested / heterogeneous** (recipes referencing items referencing materials) → TOML
- **Translations** → `.po`
- **Engine writes + reads at runtime** → binary
- **External-standard interchange** → whatever the standard says (glTF = JSON, KTX2 = binary, etc.)
- Hot-reload of CSV + TOML + `.po` during dev is a hard requirement
- Dev-only `--dump-save` command emits TOML from binary saves for inspection

**SQLite is intentionally not used** — neither as runtime nor as authoring source-of-truth. Editor-time use deferred to v1.x if content volume forces it; CSV/TOML stay canonical. See [`specs/content-authoring.md`](specs/content-authoring.md) § "Why CSV not SQLite".

## Asset Pipeline (Godot/Unreal-style)

Source files are author-friendly. Runtime files are engine-friendly. The pipeline bakes one into the other.

**Flow:**

1. Author drops a source file into `assets/` (e.g. `tree.png`, `sword.gltf`, `spell_fire.toml`).
2. File watcher detects it, runs the matching importer.
3. Importer writes a baked file to `.import/<guid>.<ext>` plus a sidecar `.import/<guid>.json` with import settings.
4. **Stable GUID** is assigned at first import and stored in the **Asset Database** (`assets/.assetdb.toml`, mapping GUID → source path + content hash).
   References from scenes/recipes/mods use the GUID, never the path. Renaming or moving a source file does not break references.
5. Runtime loads only `.import/` files. Source files are not shipped.
6. Editing the source re-bakes and hot-reloads the running game/editor.

**Per-asset-type importers:**

| Source | Importer output | Notes |
| --- | --- | --- |
| `.png`, `.tga` | **KTX2** with BC7 (desktop) / ASTC (Android), mipmaps baked | [Khronos KTX](https://www.khronos.org/ktx/) |
| `.gltf`, `.glb` | Engine mesh binary (deduplicated verts, GPU-ready index/vertex buffers) | [glTF 2.0](https://www.khronos.org/gltf/) |
| `.vox` (MagicaVoxel) | Engine voxel format | [ephtracy.github.io](https://ephtracy.github.io/) |
| `.wav` | OGG Vorbis or Opus | TBD with audio choice |
| `.glsl` / `.slang` | SPIR-V | Compiled via `glslang` or `slangc` |
| `.toml` (game data) | Validated + cached binary table | Schema validation at import; runtime loads cached form |

**Why a GUID layer:** Godot, Unity, and Unreal all do this for the same reason — paths are unstable, references break on rename, and merge conflicts in scene files become unreadable. A 64- or 128-bit GUID is forever, the path is just a hint.

**Performance note:** the runtime never touches source files. Everything expensive (compression, mipmaps, deduplication, schema validation) happens at import time on the dev machine, not at game startup. This is the lever that gets Godot/Unreal-class UX without their runtime cost.

## UI

- **TOML layout + SCSS styling** for in-game UI (per `myGoals.md` §2.3).
- **ImGui** (via C++ wrapper) for editor and dev tooling — fast to iterate, not shipped to players.

## Networking

- **Authoritative client-server**, written in Zig.
- UDP-based transport (custom or via a vetted C library such as ENet / GameNetworkingSockets — decision deferred to Phase 10; see [`external-libs-catalog.md`](external-libs-catalog.md)).
- Client prediction + server reconciliation, interest management, LAN discovery.

## Modding

- **Layered loader** (Core → Mods → Player Overrides) in Zig.
- **Native plugin ABI** via stable `extern "C"` interface — versioned, so engine refactors don't break shipped mods.
- **Steam Workshop** via Steamworks SDK (C++) — only on Steam builds; gated behind a build flag so non-Steam builds don't pull it in.

## Audio

- TBD. Likely **miniaudio** (C, single-header) for v1 unless a Zig-native option proves out.

## Build & Tooling

- **`build.zig`** as the single source of truth.
- **RenderDoc** for graphics debugging.
- **NSight** / **PIX** for GPU profiling.
- Custom voxel editor lives inside the engine binary (dev builds).

## Observability

Three separate telemetry concerns, three separate tools. Don't conflate them — each is right for its problem and wrong for the others.

| Concern | Tool | Backend ships in | Phase |
| --- | --- | --- | --- |
| Dev-time profiling (sub-µs spans, every frame) | **Tracy** | Editor + dev builds only (`-Dtracy=true`); never in shipped runtime | Stubs Phase 2; real backend Phase 5 |
| Runtime debug logs (text streams in editor playtest) | **libghostty** + engine log multiplexer | Editor only | Stubs Phase 2; real backend Phase 12 |
| Production server telemetry (metrics / structured logs / occasional traces from dedicated servers) | **OpenTelemetry** → Grafana stack (Prometheus / Loki / Tempo) | `libzvox-runtime` only when project enables `[modules.telemetry]`; runtime-flag gated | Stubs Phase 2; real backend Phase 10 |

All three flow through stable abstract sinks defined in Phase 2:

- `src/core/profile.zig` — `zone(name)` / `frameMark()` / `plot(name, value)`
- `src/core/log_sink.zig` — `log.info / warn / error(channel, fmt, args)`
- `src/core/metrics.zig` — `counter(name).inc()` / `gauge(name).set(v)` / `histogram(name).observe(v)`

Backends are picked at startup; call sites never move when real backends land. See [`engine-vs-game.md` § Observability](engine-vs-game.md#observability) for the engine-vs-runtime split.

**Sources:**

- Tracy: [github.com/wolfpld/tracy](https://github.com/wolfpld/tracy)
- libghostty: [github.com/ghostty-org/ghostty](https://github.com/ghostty-org/ghostty)
- OpenTelemetry: [opentelemetry.io/docs](https://opentelemetry.io/docs/)
- OTLP protocol: [opentelemetry.io/docs/specs/otlp](https://opentelemetry.io/docs/specs/otlp/)
- Grafana OTLP endpoint: [grafana.com OTLP docs](https://grafana.com/docs/grafana-cloud/send-data/otlp/)

**OTel performance discipline (Phase 10):**

- Sample aggressively — 1% of game ticks for metrics, all errors, all rare events
- Batch span processor only, never the simple/sync processor
- No auto-instrumentation — hand-pick instrumented sites
- Following these rules: <1% server overhead. Ignoring them: 20%.

## Scripting (Game Logic)

Game logic is written in **Zig (preferred) or C++** — both AOT-compiled. No interpreted scripting. See [`engine-vs-game.md` § 5](engine-vs-game.md#5-scripting) for the full design.

- Compiled per-project via `zig build-lib -dynamic` (Zig) or `zig c++ -shared` (C++) into `<project>/.import/scripts/libgame.{so,dll}`
- Uses the same stable `extern "C"` ABI as native mods — game code IS a mod structurally
- Hot-reload: `dlclose` + recompile + `dlopen`
- No external toolchain on the dev's machine — Zig ships bundled LLVM/Clang for the C++ path
- ABI surface is versioned; engine refactors must not break it or shipped games break

## In-Engine Code Editor

Embedded **Neovim**, bundled with the engine. Chosen for speed and modal-editing snappiness.

- Neovim ships at `bundled/nvim/{bin,share,lib}/` in the engine release archive — always available, no "is nvim installed?" UX papercut
- `src/editor/code_editor/` spawns `bundled/nvim/bin/nvim --embed` via `std.process.Child`, communicates over msgpack-RPC
- **Rendering**: GUI grid embed via `redraw` notifications, no terminal in the path. Reference: [neovide](https://github.com/neovide/neovide), [goneovim](https://github.com/akiyosi/goneovim)
- User's `~/.config/nvim/` is respected automatically by `nvim --embed`; default config shipped at `bundled/nvim/share/nvim/sysinit.vim` for first-time users
- Last-resort fallback: ImGui text edit if the bundled binary is missing or subprocess spawning is blocked
- License: Neovim is Apache 2.0 — bundling allowed, attribution required in engine `LICENSES.md`
- CI fetches platform-specific Neovim release tarballs (Linux/Windows/macOS) and bundles them into the engine release
- Editor-only — `src/editor/code_editor/` is gated by `tools_enabled` and never ships with games

## Runtime Debug Output (Playtest Logs)

Embedded **libghostty** as the text-stream rendering surface, with an engine-side log multiplexer on top. Plays the Godot Output / Unity Console / Unreal Output Log role during playtest.

- `src/editor/playtest_log/` panel: libghostty pane + ImGui chrome (level filter, channel tabs, pause/clear/save)
- Engine writes structured logs (level + channel + timestamp + payload) to a sink; sink renders into Ghostty as ANSI-colored text
- Click `file:line` in stream → routes to `src/editor/code_editor/` to open the file at that line
- Crash dumps pipe to both Ghostty and `<project>/.import/crash_<timestamp>.log`
- Wrapped behind `src/core/log_sink.zig`; libghostty's embedding API is post-1.0 so the wrapper means swapping backend later costs nothing
- Source: [github.com/ghostty-org/ghostty](https://github.com/ghostty-org/ghostty)
- Editor-only; shipped games log to plain stdout/stderr and log files

## Platforms

- **v1**: Windows + Linux desktop.
- **Later**: Android (mobile target from `myGoals.md`).
- **Not pursuing**: macOS, consoles, web.

## Future Considerations

- ~~Pure Zig platform layer (replace GLFW).~~ *Withdrawn 2026-05-26: SDL3 adopted instead — see § Windowing & Input.*
- Custom voxel format + editor pipeline.
- Voxel GI (VXGI-style) once Phase 6 streaming is solid.
- WebGPU fallback (low priority).

Update this document as decisions are made.
