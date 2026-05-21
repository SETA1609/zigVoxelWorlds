# Event / Messaging Bus Spec

> Pub-sub for gameplay events. Without this, every system polls or hard-couples to every other. Gap: [`gaps.md` § 2.1.E](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.E](../gap-references.md).

## Scope

A typed, comptime-validated event bus that lets any module emit events and any module subscribe. Critical infrastructure for Phase 8 gameplay modules (quest triggers, achievements, sound effects, particle spawns) and for mods to react to engine events.

## Components

- **Event types** — comptime struct definitions in `src/core/events.zig` plus mod-defined event types registered through the C ABI
- **Listener registry** — per-event-type list of `(opaque_ctx, fn_ptr)` pairs
- **Emit API** — `events.emit(MyEvent{ .field = value })` — type-safe at compile time; Zig comptime resolves the listener list
- **Subscribe API** — `events.subscribe(MyEvent, ctx, callback)` — returns a `SubscriptionHandle` for later `unsubscribe`
- **Priority + ordering** — listeners run in registration order by default; explicit priority via `subscribeWithPriority`
- **Cross-module via C ABI** — mods register listeners through `extern "C"` shims; payloads serialized through a stable layout (FlatBuffers? plain C struct?) — decision in [`gaps.md` § 3](../gaps.md)

## Borrowed pattern

Godot's signal system ([`gap-references.md` § 2.1.E](../gap-references.md)): signal declaration, multi-cast emit, callable-based listeners. Zig comptime makes it type-safe without Godot's `Variant` tax — listeners receive the exact event struct, not a tagged-union.

## Engine events (canonical set, extensible)

Module load lifecycle:

- `engine.module_loaded { name }`
- `engine.module_unloaded { name }`

Scene events:

- `scene.entered { scene_id }`
- `scene.exited { scene_id }`
- `chunk.loaded { x, y, z }`
- `chunk.unloaded { x, y, z }`

Voxel events:

- `voxel.modified { pos, old_id, new_id, by_entity }`

Entity events:

- `entity.spawned { handle }`
- `entity.destroyed { handle }`
- `entity.damaged { handle, amount, source }`

Gameplay (fired by modules, listed here for the canonical schema):

- `skill.leveled { entity, skill_id, level }`
- `item.crafted { entity, item_id, quality }`
- `spell.cast { caster, spell_id, target_pos }`
- `quest.flag_set { quest_id, flag, value }`

Mods extend this set via their own event types registered at module init.

## Performance

- Listener invocation: direct function call (Zig comptime → no vtable)
- Emit cost: O(listeners) per event; expected listener count per event = single digits in practice
- No allocation in hot path: event payload is a stack-allocated comptime struct
- Tracy zone per `emit()` for profiling

## Open decisions

- Sync vs async dispatch — default sync (caller blocks until all listeners return); async deferred for replay-determinism
- Listener removal during emit (concurrent-modification handling)
- Mod-event serialization format (FlatBuffers vs plain C struct vs custom)

## Milestone

Phase 7 (or earlier — slot into Phase 2 if Module System uses it for `engine.module_loaded`). Test: subscribe two listeners to `voxel.modified` from two modules; mine a block; both listeners fire with the correct payload.
