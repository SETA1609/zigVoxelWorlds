# Planning Gaps — Decide Before You Code

> 38 open decisions that aren't yet documented anywhere. Most are not blocking Phase 0–1, but knowing they exist prevents nasty mid-phase pivots. Items marked **🔥 urgent** unblock Phase 0; **⏰ before-phase** must be settled before its named phase starts; **📅 later** can wait until Phase 12+.

Cross-reference: [`ROADMAP.md`](ROADMAP.md), [`ARCHITECTURE.md`](ARCHITECTURE.md), [`vision.md`](vision.md), [`mission.md`](mission.md), [`guard.md`](guard.md).

The numbering is for citation, not priority. Use the tags for sequencing.

---

## §1. Data schemas — the most urgent gap

You've referenced `project.toml`, `mod.toml`, `scene.toml`, `orchestrator.toml`, and `assetdb.toml` throughout the docs without specifying their fields. Without these, Phase 0 cannot finish.

1. **🔥 `project.toml` schema** — exact fields per section. `[project]` name/version/engine_compat, `[modules]` enable-disable table, `[scripts]` language+entry, `[export.<target>]` per-platform options. Field-by-field spec doc.
2. **🔥 `mod.toml` schema** — manifest fields: name, version, engine ABI compat range, dependencies, load order, plugin path. Stable across engine versions.
3. **⏰ before Phase 9** **Scene definition format** — what lives in a `scene.toml`? Entity spawn list, region AABBs, edit-policy reference, lighting, weather, time-of-day overrides?
4. **⏰ before Phase 9** **`orchestrator.toml`** — how does it link scenes to world coordinates and triggers? Loading/unloading model.
5. **🔥 `assetdb.toml`** — exact GUID format (UUIDv7 recommended for time-orderable IDs), content-hash algorithm (Blake3 vs SHA-256), per-entry fields (guid, source_path, content_hash, importer_version, settings_ref).
6. **⏰ before Phase 13** **Save file binary format** — exact layout: magic header bytes, version per section (chunk delta, quest state, player state, faction state, mod-registered sections), endianness, alignment.

## §2. Architecture — type/layout decisions

7. **🔥 `Handle` layout** — u64 with `(generation: u32, index: u32)`? Or `(server_id: u8, generation: u24, index: u32)` for per-server handle spaces? Decide once; affects every Server API.
8. **🔥 Coordinate system** — Y-up vs Z-up; left-handed vs right-handed (Vulkan is right-handed by default); units (meters); block size in world units (1m default).
9. **🔥 Voxel data layout** — bits per voxel (8 limits to 256 types, 16 gives 65k, 32 is overkill); palette per chunk vs global; voxel-ID semantics (material? biome? both? — biome separate channel).
10. **🔥 Chunk size** — 16³ (= 4096 voxels) vs 32³ (= 32768) vs 64³ (= 262144). Affects memory, meshing batch size, network packet size, streaming granularity. Most engines pick 16³ or 32³; 64³ usually too big.
11. **⏰ before Phase 6** **Origin rebasing strategy** — at 10 km² float32 precision degrades. Either use float64 throughout (cost) or rebase the world origin when player crosses a threshold (complexity). Pick one + define the trigger.
12. **🔥 Threading model** — main thread + render thread + chunk-gen pool + audio thread + network thread + AI thread? Which can talk to which? Lock-free queues, actor model, channels? Designed before Phase 2 because servers + handles touch this.
13. **🔥 C ABI surface design** — concrete `extern "C"` function set the engine exposes to mods/scripts. Versioned. Designed once, retrofitted = pain. Even a skeleton with 10 functions is enough for Phase 0.

## §3. Gameplay shapes — what your four games actually look like at the engine level

14. **⏰ before Phase 8** **Camera model(s)** — first-person (Daggerfall), top-down isometric (Stardew), third-person orbit (Atelier). Each needs different camera math + FOV + cull frustum + UI placement. Will you ship multiple modes or pick one?
15. **⏰ before Phase 1** (input layer) **Input mapping system** — keyboard + mouse + gamepad + rebinding. Action-based vs raw scancode. TOML-driven mappings?
16. **⏰ before Phase 12** **UI scaling** — DPI awareness, controller-friendly navigation, font subsystem (FreeType? stb_truetype? bitmap fonts?).
17. **⏰ before Phase 13** **Save slot UX** — multiple saves per profile, autosave cadence, manual saves, save metadata (screenshot, playtime, level).
18. **⏰ before Phase 8** **Quest system data model** — boolean flags only? state machines? branching dialog trees? scripted quests? How does "Daggerfall has 700+ quests" actually get expressed?
19. **⏰ before Phase 8** **NPC AI architecture** — utility AI, behavior trees, GOAP, hand-coded states. Stardew needs NPC schedules; Daggerfall needs roaming + dialog; rogue-like needs combat tactics. Different needs.
20. **⏰ before Phase 8** **Magic system data model** — effect composition rules, mana/cost, cooldowns, target acquisition, in-data spell DSL. Per the Morrowind spellmaking goal.
21. **⏰ before Phase 8** **Crafting math** — quality formula (skill × ingredient quality + random), time-cost, success curves, station types, Atelier-style synthesis sub-rules.
22. **⏰ before Phase 8** **Inventory model** — slot count, weight, stackability, container hierarchy, equipped vs carried, hot-bar / wheel UX.

## §4. Modding & scripting

23. **⏰ before Phase 14** **Modding security model** — can mods access disk? network? arbitrary syscalls? File-system sandbox? Threat model: pranks-only or "don't trust Workshop downloads"? Affects design fundamentally.
24. **🔥 ABI versioning policy** — semver? Compatibility window across engine releases? When does ABI break vs extend? Affects what you commit to in the engine version 1.0.
25. **⏰ before Phase 14** **Script package contract** — beyond the C ABI: entry-function signature, lifecycle hooks (`onLoad`, `onTick`, `onUnload`), error-handling expectations.

## §5. Multiplayer

26. **⏰ before Phase 10** **Network transport choice** — ENet vs GameNetworkingSockets vs custom UDP. Tickrate. Reliable channels. Currently parked.
27. **⏰ before Phase 10** **Edit authority model** — host decides, client requests, optimistic + rollback? Most co-op uses host-decides for simplicity; some games need client-side prediction for build placement.
28. **⏰ before Phase 10** **Server discovery** — LAN broadcast, manual server key, lobby service? Steam Networking integration?
29. **⏰ before Phase 10** **NAT traversal** — direct connect only? Steam Networking? hole-punching service? Affects how non-LAN co-op works.

## §6. Process — how you actually work

30. **⏰ before commit #2** **Branching strategy** — trunk-based with feature flags? gitflow? Affects how dev/stable channels in [`licensing.md`](licensing.md) actually operate.
31. **📅 when it goes public** **Issue tracker** — GitHub Issues + labels? Linear?
32. **📅 post Phase 11** **Release cadence** — weekly dev, monthly stable, yearly major? Tag scheme. Steam build automation.
33. **🔥 Test strategy** — unit (`test "..." {}` co-located) vs integration (`tests/`) vs golden-image / visual regression. Decide before Phase 1 so test scaffolding is in place from day one.
34. **⏰ before Phase 5** **Performance regression detection** — benchmark suite, per-PR perf budget, CI hardware specs. Critical for the 50–60 FPS target.

## §7. Business / publishing

35. **📅 before any Steam page** **Trademark filing** — defensive "zVoxRealms" trademark filing (USPTO + EUIPO, ~$300/jurisdiction). [`licensing.md`](licensing.md) calls trademark out as separate from copyright; it has no legal force until filed.
36. **📅 before Phase 15** **Steam page / wishlist plan** — early Steam page = wishlist accumulation = launch boost. Plan content + assets.
37. **⏰ before Phase 10 OTel rollout** **Telemetry consent UX** — opt-in or opt-out (varies by jurisdiction; GDPR matters in EU). Affects data-collection ABI.
38. **⏰ before Phase 15** **Crash reporting** — local-only logs, opt-in upload, player-controlled? What data is included (stack trace + which? user-supplied note? game state snapshot?).

---

## Suggested resolution order

If you tackle these in the order below, each phase is unblocked when it starts:

### Week 1–2 — write data-schema docs
1. `project.toml` (#1) — most-referenced thing
2. `mod.toml` (#2)
3. `assetdb.toml` (#5)
4. Save format header + section versioning (#6, partial — just enough for chunk deltas)

### Week 2–3 — architecture micro-decisions (mostly written, no code yet)
5. `Handle` layout (#7)
6. Coordinate system (#8)
7. Voxel data layout (#9)
8. Chunk size (#10)
9. Threading model (#12)
10. C ABI surface skeleton (#13)
11. Test strategy (#33)
12. Branching strategy (#30)
13. ABI versioning policy (#24)

### Pre-Phase coding
14. Camera model (before Phase 1 viewport)
15. Input mapping (before Phase 1)
16. Origin rebasing strategy (before Phase 6)
17. Quest / AI / magic / crafting / inventory data models (before Phase 8)
18. Network transport + authority (before Phase 10)
19. Performance regression CI (before Phase 5)
20. Save UX + slot management (before Phase 13)
21. Crash reporting + telemetry consent (before Phase 13–14)

### Eventual
22. Trademark filing (before public Steam page)
23. Steam page assets (before Phase 15)
24. Release cadence + automation (before Phase 11)

---

## How this doc gets updated

When a planning item is decided:

1. The decision goes into the relevant doc (`ARCHITECTURE.md`, `tech-stack.md`, `engine-vs-game.md`, etc.) or its own new spec doc (e.g. `docs/schemas/project-toml.md`)
2. Cross out the corresponding item here, or replace it with a one-line "→ decided in [link]"
3. When all items in a category are decided, collapse the section

This file should shrink over time. When it's empty, you're done planning. (You won't be — new questions will appear — but the trend should be downward.)
