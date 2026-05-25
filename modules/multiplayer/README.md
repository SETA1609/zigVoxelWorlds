# `modules/multiplayer/`

> Network protocols + sync + replication + interest management. **Host-authoritative for co-op / PvE; server-authoritative for PvP.** Player count is configurable per project (`project.toml` `[multiplayer] max_players`) — no hard cap baked into the engine. Phase 10.

## Player count

The previous spec said "4-player co-op + 40–50p dedicated". Both numbers were assumptions inherited from the Daggerfall co-op design. Revised 2026-05-25 to support the Megabonk-Survivors + Hunger Games target (5th target — see `modules/arena_modes/`):

- Daggerfall / Stardew / Atelier / traditional rogue-like: default 4-player co-op
- Arena modes PvE co-op: configurable squad size
- Arena modes PvP FFA / teams: scales as high as the netcode + Iris-style delta replication can sustain. Engine ceiling will be set by Phase 10 benchmarking against commodity server hardware.

## Authority model

| Mode | Authority | Notes |
| --- | --- | --- |
| Co-op (Daggerfall, Stardew, Atelier, rogue_tower) | Host-authoritative | Clients predict + reconcile against host frames |
| PvE arena (`modules/arena_modes/` co-op) | Host-authoritative | Same model — host runs hordes + validates state |
| **PvP arena** (`modules/arena_modes/` FFA / teams) | **Server-authoritative** | First target needing validated movement + combat resolution. Anti-cheat-grade validation added in whichever phase ships the PvP arena. |

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
