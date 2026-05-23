# AI Subsystem Spec

> Pathfinding, behavior trees, perception, and AI level-of-detail tiers for Daggerfall-scale NPC populations. Gap: [`gaps.md` § 1.9](../gaps.md) + [`gaps.md` § 3 #18 NPC AI architecture](../gaps.md). Reference patterns: this spec (no dedicated section in [`gap-references.md`](../gap-references.md) yet — extends Tier 1 references).

## Scope

A data-driven AI system that supports:

- Thousands of NPCs in the voxel Daggerfall slice (a single town has 200–500 residents; the world holds thousands more across other settlements + wandering)
- Daily schedules (Stardew villagers, Daggerfall townspeople)
- Combat AI (rogue-like monsters, Daggerfall hostiles)
- Static encounters (Atelier NPCs at workstations)
- All driven by data (TOML behavior trees) — designers + modders never touch engine code to add a new NPC behavior

Out of scope for v1.0: full goal-oriented action planning (GOAP), neural / learned AI, swarm/flocking (deferred to v1.x if needed).

## Architecture choice — Behavior Trees, with NavMesh pathfinding

The three industry-standard NPC AI paradigms:

| Paradigm | Pros | Cons | Fits us? |
| --- | --- | --- | --- |
| **Behavior Trees (BT)** | Data-driven, debuggable, composable subtrees, industry-proven (Halo, Bioshock, every Bethesda title, many indies). Authorable in TOML. | Can become "spaghetti trees" without discipline. | ✅ best fit |
| **GOAP (Goal-Oriented Action Planning)** | Emergent behavior, NPCs find unscripted solutions (FEAR, Shadow of Mordor) | Planner cost grows with action count. Hard to debug. Harder to author. | ❌ v1.x maybe |
| **Utility AI** | NPCs weigh competing scores per action; smooth gradient behavior (The Sims, RimWorld) | Tuning the scoring curves is its own discipline. Less authorable. | ❌ defer |

**Decision: Behavior Trees + utility-style score nodes embedded as a BT node type when needed** (hybrid). This is the same pattern Unreal's BTService + decorator system uses. Gives BT authorability with utility-AI smoothness for the cases that need it (e.g. "which villager do I greet first" = score-based; "what do I do at 8pm" = scheduled BT branch).

## Behavior Tree primitives

Standard BT node taxonomy:

| Node type | Behavior |
| --- | --- |
| **Selector** (`?`) | Run children left-to-right; succeed on first success; fail if all fail |
| **Sequence** (`→`) | Run children left-to-right; fail on first failure; succeed if all succeed |
| **Parallel** | Run all children simultaneously; success/fail policy declared |
| **Inverter** | Invert child's success/failure |
| **Repeater** | Re-run child N times or forever |
| **Until-success** / **Until-failure** | Loop child until result matches |
| **Decorator (condition)** | Gate child on a predicate (e.g. "only if time > 8pm") |
| **Decorator (utility-score)** | Run only if this branch's score is highest among siblings |
| **Action (leaf)** | Game-side function: `move_to(target)`, `attack(entity)`, `say(line_id)`, etc. |
| **Subtree reference** | Include another BT by name (reuse "go-to-bed" across villager types) |

### Authoring — TOML

```toml
[bt.villager_daily_schedule]
root = "selector_time_of_day"

[[bt.villager_daily_schedule.nodes]]
id = "selector_time_of_day"
type = "selector"
children = ["seq_night", "seq_morning", "seq_workday", "seq_evening"]

[[bt.villager_daily_schedule.nodes]]
id = "seq_night"
type = "sequence"
children = ["cond_is_night", "subtree_go_to_bed"]

[[bt.villager_daily_schedule.nodes]]
id = "cond_is_night"
type = "decorator_condition"
condition = "game.time_of_day in [22, 6]"

[[bt.villager_daily_schedule.nodes]]
id = "subtree_go_to_bed"
type = "subtree"
ref = "bt.common_go_home_and_sleep"
```

This trades visual-editor convenience for git-diffability + modder-friendliness + scriptability. The Phase 12 editor adds a graph-view rendering on top of the TOML (TOML stays source of truth).

## Blackboard

Per-NPC scratch state shared across BT ticks. Standard pattern.

```zig
const Blackboard = struct {
    entity: Handle,
    target: ?Handle = null,         // current combat target
    home: ?WorldPos = null,         // bed location
    workplace: ?WorldPos = null,    // workstation
    last_seen_player: ?WorldPos = null,
    perception_events: RingBuffer(PerceptionEvent, 32),
    // ... per-NPC data
};
```

Blackboard read/write keys exposed to BT nodes as named fields. Modders extend via the C ABI (register new blackboard fields at module-init time).

## Pathfinding — NavMesh on voxel surfaces

NavMesh is the industry-standard approach for open-world pathfinding. We generate a 2.5D navmesh from walkable voxel surfaces.

**Generation:**

1. For each chunk, find voxels whose top face is exposed (the floor a humanoid can stand on)
2. Filter by walkability — voxel type allows stepping (not water, not lava unless flagged)
3. Filter by clearance — at least N voxels of air above (humanoid ≈ 2 voxels)
4. Connect adjacent walkable voxels → navmesh polygon
5. Mark edge connectivity (stair-step within 1 voxel height = traversable; jump within 2 voxels = jump edge; gap > N voxels = no edge)

A* over the navmesh polygons gives a path. Path smoothing (string-pulling) makes it look natural.

**Async rebuild:** when a chunk is edited (player mines a tunnel, scene-edit policy permits build), the chunk's navmesh tile is rebuilt off-thread. Tile-based navmesh means edits don't invalidate the whole world.

**Reference:** [Recast / Detour](https://github.com/recastnavigation/recastnavigation) — industry-standard open-source library, **zlib license** (compatible with Apache 2.0). Used by Unity, parts of Unreal, many indies. **Decision:** ship a Recast adapter sub-repo (`libs/zig-recast-adapter/`, MIT — matching other adapters per [`licensing.md`](../licensing.md)) rather than reimplement A* + funnel algorithm from scratch.

Alternative considered: hand-rolled A* on the voxel grid directly (no navmesh). **Rejected:** A* on raw voxel cells is O(W × D × H) and stalls on long-range paths (NPC walking across a town). NavMesh's polygon abstraction collapses thousands of voxels into ~50 polys; A* over 50 polys is <1 ms.

## Perception

Two perception channels per NPC:

### Sight

- FOV cone (default 90° horizontal, 30° vertical) + max distance (default 30 m)
- Per-frame check against entities in cone: raycast to test occlusion against voxels + meshified geometry
- Hit → emit `perception.saw { observer, target, world_pos }` into the NPC's blackboard ring

### Hearing

- Radius (default 15 m for footsteps; spell sounds carry further per [`audio.md`](audio.md))
- Sound events emitted from gameplay (`sound.played { pos, loudness, source }`) propagate to nearby NPCs
- Walls dampen — hearing raycast against voxels (with attenuation factor per voxel material)

Perception events feed the BT via decorator nodes (`cond_saw_player_recently`).

## AI Level of Detail (LOD) — critical for Daggerfall-scale

You cannot tick all NPC BTs every frame at Daggerfall scale. The pattern Skyrim, Dragon Age, and most large-RPG engines use is **distance-tiered AI**:

| Tier | Distance to nearest player | Tick rate | BT depth |
| --- | --- | --- | --- |
| **Hot** | < 50 m | Every frame (60 Hz) | Full BT — combat, dialogue, full perception |
| **Warm** | 50–200 m | Every 8 frames (~7.5 Hz) | Full BT, but raycasts dropped to once/sec |
| **Cold** | 200 m+ | Once per second (1 Hz) | Simplified BT — schedule-only branch |
| **Frozen** | Off-loaded chunks / very far | Once per game-minute | Position + schedule advance, no BT eval; pure state machine |

NPC migrates between tiers as the player moves. The transition is lossless — the BT is paused (its current node stored on the blackboard), not reset.

**Schedule-only fallback:** every NPC has a daily schedule (`8am-12pm @ workplace`, `12pm-1pm @ tavern`, etc.). At the **Frozen** tier, the engine just teleports the NPC to the schedule's current location at each game-minute tick. No BT runs. Player gets close → tier upgrades, BT resumes.

This pattern is what makes "thousands of NPCs simulating across the world" affordable. Without it, Daggerfall scale is impossible on the target iGPU.

## Reference patterns

| Engine | Where to study | What to adapt |
| --- | --- | --- |
| **Unreal — BehaviorTree** | `$REFS/UnrealEngine/Engine/Source/Runtime/AIModule/Public/BehaviorTree/` (BTNode, BTCompositeNode, BTDecorator, BTService, BTTaskNode) + `$REFS/UnrealEngine/Engine/Source/Editor/BehaviorTreeEditor/` for the visual editor | The BT **node taxonomy** (Composite / Decorator / Service / Task) and tick semantics. **Don't port the UObject machinery.** Visual editor concept influences our Phase 12 graph view |
| **Unreal — MassAI / MassEntity** | `$REFS/UnrealEngine/Engine/Source/Runtime/MassEntity/` + `$REFS/UnrealEngine/Engine/Plugins/AI/MassAI/` | Large-population AI built on data-oriented ECS. Study the tier/processor model for how to batch AI work. Heavy framework — extract concepts only |
| **Godot — NavigationAgent3D** | `$REFS/godot/scene/3d/navigation_agent_3d.cpp` + `$REFS/godot/modules/navigation/` | Navmesh agent pattern: agent component + navigation server + per-region cost. Closer to what we want than Unreal's machinery |
| **Godot — no built-in BT** | Community plugins (`LimboAI`) — not in `$REFS/` | Skip; use Unreal taxonomy + our own implementation |
| **Luanti — entity AI** | `$REFS/luanti-custom/src/server/mods.cpp` + `$REFS/luanti-custom/builtin/game/` (mob entity scripts in Lua) | Per-mob hand-coded scripts. **Not** a good pattern for our scale; useful only to see "what's the minimum viable" |
| **Hazel** | No AI subsystem | skip |

Adaptation rule per [`engine-references.md` § Legal](../engine-references.md): read Unreal's BT node taxonomy, **close the source**, implement from scratch in Zig. Unreal-derived code = lawsuit risk per [`feedback_reference_engine_no_verbatim.md`](../../memory/feedback_reference_engine_no_verbatim.md).

## Integration with other systems

| System | Touch point |
| --- | --- |
| [`specs/ecs.md`](ecs.md) | NPC = entity with `AIController` component holding (BT ref, blackboard, current tier) |
| [`specs/events.md`](events.md) | Perception events on the bus; gameplay events (`combat.hit { attacker, target }`) feed BTs |
| [`specs/scene.md`](scene.md) | NPC schedules reference scene-region names ("work at `blacksmith_forge`"); ownership transitions can pause schedules |
| [`specs/dialog.md`](dialog.md) | BT action node `start_dialog(target, dialog_id)` opens the modal dialog |
| [`specs/multiplayer.md`](multiplayer.md) | Server-authoritative AI — BTs tick on the host only; clients receive position/animation updates. AI LOD tiers computed against nearest player |
| [`specs/voxel.md`](voxel.md) | NavMesh rebuilds on voxel edits within editable regions |
| [`specs/gameplay.md`](gameplay.md) | Combat AI consumes the skill/perk system to pick abilities |

## Per target game — how the same AI serves all four

| Game | Dominant AI work |
| --- | --- |
| **Voxel Daggerfall** | Thousands of NPCs with daily schedules, faction-driven combat, dialog-state-aware BTs |
| **Voxel Stardew** | ~30–50 villagers with rich schedules + relationships; light combat in mines |
| **Voxel Atelier** | Static NPCs at workstations + light wandering + combat in gathering zones |
| **Voxel rogue-like** | Hostile monsters with combat BTs; no schedules; per-run spawn |

Same BT engine, different content. The rogue-like skips schedules entirely; Daggerfall pushes the LOD tiers hard.

## Open decisions

- **NavMesh agent radius scaling** — single agent type for v1.0 (humanoid). Multi-agent (humanoid + crawler + flyer) deferred to v1.x. Flying creatures use 3D pathfinding on a different graph
- **Off-thread BT ticking** — for the Hot/Warm tiers. Cold/Frozen run on a worker thread already. Decide during Phase 8 implementation by profiling
- **Squad / formation AI** — Daggerfall has guard patrols. Defer to v1.x; v1.0 ships individual-AI only
- **Stealth / detection meter** — full stealth detection mechanic (Skyrim-style) is gameplay design, not engine. Engine provides the perception primitives; gameplay layer builds the meter
- **AI debugging UI** — visualize BTs in editor (current node highlighted per NPC), perception cones, navmesh overlay. Phase 12 work

## Milestone

Phase 8 (Gameplay Modules — AI is part of this phase, sitting alongside skills/magic/crafting). Daggerfall slice runs with:

- 200+ NPCs in the town, each on a daily schedule
- Combat AI for 5+ enemy types in 3+ dungeons
- LOD tiers active — town simulates from inside the town walls and "lives" when re-entered
- Hot-tier NPC count keeps the AI frame cost under 1 ms on the target iGPU
- BT TOML hot-reload — editing a behavior in editor takes effect in playtest within a tick
