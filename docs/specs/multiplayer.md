# Multiplayer Spec

> Host-authoritative 4-player co-op default, optional 40–50 dedicated servers. Roadmap: [`ROADMAP.md` § Phase 10](../ROADMAP.md). Open transport + authority questions: [`gaps.md` § 3](../gaps.md). Save-model interaction: [`ARCHITECTURE.md` cross-cutting](../ARCHITECTURE.md).

## Scope

Built as `modules/multiplayer/` behind `NetServer`. The transport library is one of ENet (C, direct cImport) or GameNetworkingSockets (C++, adapter sub-repo) — decided in Phase 10.

## Components

- **Authoritative client-server** — server owns state; clients submit intent + display prediction
- **Client prediction + server reconciliation** — players see immediate response to local input, corrected when authoritative state arrives
- **Interest management** — sync only nearby entities (port Luanti's active-block pattern)
- **LAN discovery** — UDP broadcast for local games; manual server key for online
- **4-player co-op target** — default; the engine's design center
- **Dedicated servers up to 40–50 players** — optional; lean runtime path

## Reference patterns

- Luanti `src/servermap.cpp` + `src/clientmap.cpp` — chunk streaming, `emergeBlock`, interest management, MapBlock network serialization
- Unreal Iris (`Engine/Source/Runtime/Net/Iris/Public/Iris/ReplicationSystem/`) — handle-based delta replication; `FNetRefHandle` + `ReplicationFragment`

## Save-model alignment

The save format's voxel-delta + game-state-section structure ([`ARCHITECTURE.md`](../ARCHITECTURE.md)) is the same format the network protocol ships over the wire. Server sends delta blobs, not chunks. Seed-deterministic regen + delta layering makes this tractable.

## OTel telemetry hook

Phase 10 also wires the OpenTelemetry backend behind `src/core/log_sink.zig` and `src/core/metrics.zig` for dedicated-server observability — see [`tech-stack.md` § Observability](../tech-stack.md#observability). Gated by `--otlp-endpoint <url>`; clients never enable it.

## Open decisions

- Transport choice (ENet vs GameNetworkingSockets)
- Edit authority model (host-decides vs client-prediction-with-rollback)
- Server discovery (LAN broadcast vs lobby service)
- NAT traversal (direct-connect-only vs hole-punching)

## Milestone (from ROADMAP)

4 players co-op the dungeon-clear loop with acceptable latency; a dedicated-server instance ships tick-time p50/p95/p99 + player count to a local Grafana dashboard.
