# `modules/multiplayer/`

> Network protocols + sync + replication + interest management. Host-authoritative; 4-player co-op + 40–50p dedicated server target. Phase 10.

## What it provides

- Replication frame builder — handle-based delta replication; same on-wire format as the save delta ([`specs/save-ux.md`](../../docs/specs/save-ux.md))
- Interest management — per-client active-block + active-entity sets; only relevant deltas are sent
- Host-authoritative state machine — clients send input intents; host runs the sim and broadcasts results
- Lockstep voxel edits — chunk modifications are ordered by the host to avoid client divergence
- Connection lifecycle — join / rejoin / handoff / migration (the latter only if/when feasible)

## Reference patterns

- **Unreal Iris** `ReplicationSystem.h:69` + `ObjectReplicationBridge.h` + `ReplicationFragment.h` — fragment-based delta replication. Adapt: `NetRefHandle`-equivalent on top of our `Handle`; per-fragment delta replication aligns with save deltas.
- **Luanti** `src/servermap.cpp` + `src/clientmap.cpp` — `emergeBlock` + interest management + network serialization for chunks
- NOT Unreal's legacy `Net*` property replication — Iris is the right reference

See [`engine-references.md`](../../docs/engine-references.md) § Unreal · Iris and § Luanti.

## Layering

Depends on `core/`, `scene/`, `servers/net_server` + the chosen backend (ENet or GNS). MUST NOT import `editor/`.

## Spec

[`specs/multiplayer.md`](../../docs/specs/multiplayer.md).
