# zVoxRealms — Vision

> Where this engine is going. The long-horizon picture, separate from the immediate work plan in [`mission.md`](mission.md) and the phased roadmap in [`ROADMAP.md`](ROADMAP.md).

## The vision in one sentence

**zVoxRealms is the engine where a solo developer can ship a Daggerfall-class voxel RPG, a voxel Stardew Valley, an Atelier-style crafting game, or a rogue-like dungeon crawler — co-op-ready by default, running well on the cheapest PC the player owns, modded as easily as Minecraft, and exported as a single small binary that contains only what that game needs.**

That sentence is the vision. Everything below is what each piece of it means.

---

## The four target games

Not example genres — the four concrete game shapes the engine is designed to support out of the box. **All four are voxel games** — voxel-3D rendering, first-person camera primary, no 2D fallback, no mesh-only worlds. Triangle meshes appear only as additive content (character bodies, props, decorative meshes) on top of the voxel world. Every architectural decision is evaluated against "does this serve at least one of these?"

1. **Voxel Daggerfall-style open-world RPG** with Morrowind-style spellmaking
   - Massive seamless voxel world (10 km × 10 km × 2 km vertical, scalable to 4 km vertical), procedurally generated from seed
   - Classless skill/perk progression (Fallout/Morrowind/Daggerfall lineage)
   - Spellmaking: combine effects, custom magic schools
   - Mostly read-only world (towns, dungeons, terrain) with quest-state and player-progress saved
2. **Voxel Stardew Valley** — life / farming sim
   - Same engine, same module set + a different scene policy
   - Voxel world, mostly read-only (town, paths, decoration)
   - **Editable subsets** — designated farm plots, mines, owned house interiors — through the scene edit-policy system
   - Crafting + relationships + day cycle + seasonal events
3. **Voxel Atelier-style crafting / alchemy RPG**
   - Multi-stage synthesis: gather → process → synthesize
   - Quality system derived from skill + ingredients
   - Recipe discovery
   - Voxel gathering zones with controlled editability (you can harvest, you can't reshape the cliff)
4. **Voxel rogue-like dungeon crawler** — endless tower
   - Procedurally generated voxel runs, each seed a fresh dungeon
   - Per-run state lost on death; meta-progression persists
   - Scene policy: dungeons fully destructible during the run, world hub read-only

The engine is the *intersection* of what these four voxel games need. If a feature serves none of them, it isn't in scope. If a feature serves exactly one, it ships as an optional module.

---

## How players experience it

- **Plays well on a 10-year-old laptop.** 50–60 FPS on i3 / Ryzen 3 with integrated graphics, in 4-player co-op. < 3.5 GB RAM. 8-chunk default view distance. Mobile (Android) later. AAA-photorealism is **not** in scope.
- **Co-op without ceremony.** Default 4 players, LAN-discoverable or with a manual server key. No required accounts, no required cloud. Optional dedicated servers up to ~40–50 players when someone wants to host one.
- **The game survives a patch.** Worlds are seed-deterministic; only deltas are saved. A patched generator produces compatible worlds for unmodified areas; modified deltas layer on top. Quest state, faction reputation, NPC state, player inventory all persist across patches via versioned binary schemas.
- **Modding is first-class, not a hack.** Players install mods from a folder or from Steam Workshop with one click. Mods can add new spells, recipes, biomes, and entire systems — the same C ABI shipped games use for their own scripts. No game ever has to choose between "supporting mods" and "shipping smoothly".

---

## How developers experience it

- **One binary, opens like Godot.** Launch the engine → Project Manager appears. Create a project, open it, the editor swaps in. Open another project, the engine re-execs itself. Crashes in one editor session don't kill another.
- **Export to a tiny game.** A Stardew-style farming game without physics or networking exports to a launcher + a `libzvox-runtime.so` containing only the modules that game uses. No fat templates. A 4-player co-op rogue-like includes physics + networking; a singleplayer crafting demo doesn't. Per-project tree-shaking, every export.
- **Code in Zig, code in C++, no Lua tax.** Game logic compiles via the bundled Zig toolchain (`zig build-lib` for Zig, `zig c++` for C++). Hot-reload is `dlclose` → recompile → `dlopen`. No external toolchain installed on the developer's machine. No interpreted-script GC pause. Game code runs at engine speed.
- **Edit code inside the engine.** A bundled Neovim ships with the engine; the editor embeds it via msgpack-RPC grid rendering. The developer's existing `~/.config/nvim/` is respected. Runtime errors during playtest show in a libghostty-backed log panel; clicking a stack-trace `file:line` jumps the Neovim cursor to that line.
- **Profile what matters.** Tracy for dev frame profiling (Phase 5+), libghostty for editor playtest logs (Phase 12), OpenTelemetry → Grafana for dedicated-server production telemetry (Phase 10+). Three concerns, three tools, abstract sinks from day one.

---

## The differentiators

Things zVoxRealms does that no current engine does *together*:

| Differentiator | Why it matters | Why it's possible |
| --- | --- | --- |
| **Engine-as-app + per-project tree-shaken dynamic-lib export** | Player downloads contain only what the game uses. No 200 MB Stardew clone with physics engines stubbed out. | Zig ships LLVM bundled. Per-project relinking from the editor is trivial. C++ engines can't do this without shipping a toolchain to end users. |
| **Voxel-first + RPG-first** | Most voxel engines optimize for Minecraft-like building. Most RPG engines (Unity/Unreal/Godot) are weak on voxels. zVoxRealms is built for voxel RPGs specifically. | Borrowed: Luanti's voxel core, Hazel's clean ECS, Unreal's MassEntity scheduler concepts, all in Zig. |
| **Modding ABI = scripting ABI** | Mods and game scripts are the same thing structurally. One stable `extern "C"` surface to maintain. Players can mod every shipped game. | Native plugins via `dlopen` (`std.DynLib`). Zig's comptime exposes a clean ABI without runtime reflection. |
| **Authoritative co-op + delta saves + deterministic regen** | Saves stay small over time. Patches don't break worlds. Multiplayer sync is delta-based, not chunk-based. Server runs lean. | Seed-deterministic generator ships in every game runtime. Same delta format used for save and net protocol. |
| **Editor UX = Godot, runtime cost = native** | First-class voxel brush, biome painter, scene browser, skill/perk editor, recipe editor — at AAA editor quality, with engine-native performance because there's no script-VM tax. | Editor is just `tools_enabled` comptime code on top of the runtime; ships zero overhead to the game. |
| **Per-scene editability policy** | Same engine supports Minecraft-mode (full edit), Stardew-mode (only mines), Atelier-mode (only gathering zones), Daggerfall-mode (mostly read-only). | Policy is TOML data, not code. `VoxelServer` evaluates per-region before applying any delta. |

---

## What zVoxRealms is NOT

The vision is sharpened by what's explicitly out of scope.

- **Not a photorealistic engine.** Stylized voxel only.
- **Not an MMO platform.** 40–50 player ceiling. WoW-scale is not a goal.
- **Not console-first.** Linux + Windows desktop. Android later. No PlayStation/Xbox/Switch parity.
- **Not a general-purpose engine.** Voxel + the four genres. Not 2D platformers, not racing, not RTS. Build with Godot or Unity for those.
- **Not a no-code engine.** Designers and modders get rich data-driven tools (TOML for skills/recipes/spells/scenes), but gameplay logic is in Zig or C++. Visual scripting is not a goal pre-1.0.
- **Not a competitor to Unity/Unreal.** zVoxRealms doesn't try to support every game shape. It tries to be *unfairly good* at four specific shapes.

---

## 3-year horizon

At the end of the three-year arc the engine should be:

- **v1.0 shipped** — covers Phases 0–15 of [`ROADMAP.md`](ROADMAP.md), including a playable Daggerfall-clone vertical slice
- **One published target game** — at least one of the four target shapes shipped commercially (Steam) using the engine, demonstrating the export pipeline end-to-end
- **A modding scene** — Steam Workshop integration live, with at least one third-party mod available for the published game
- **Documented stable C ABI** — versioned, with deprecation policy. Mods shipped at v1 still work at v1.x.
- **A small but real community** — developers using the engine to build their own games. Not "Godot scale" — closer to early Defold or LÖVE community size is success.

---

## What this vision is grounded in

This vision is not aspirational fluff — it's the synthesis of concrete decisions made in:

- [`vision.md`](vision.md) + [`mission.md`](mission.md) — original scope authority
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — how the layers serve this vision
- [`tech-stack.md`](tech-stack.md) — what makes each piece possible
- [`engine-vs-game.md`](engine-vs-game.md) — the engine/game split that enables tree-shaken export
- [`engine-references.md`](engine-references.md) — concrete patterns borrowed from Hazel, Luanti, Godot, Unreal
- [`ROADMAP.md`](ROADMAP.md) — the 15 phases that get us there

The mission ([`mission.md`](mission.md)) describes the current-state operating posture for hitting this vision.
