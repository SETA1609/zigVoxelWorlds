# Multiplayer Spec

> Host-authoritative 4-player co-op default, optional 40–50 dedicated servers. Roadmap: [`ROADMAP.md` § Phase 10](../ROADMAP.md). Open transport + authority questions: [`gaps.md` § 3](../gaps.md). Save-model interaction: [`ARCHITECTURE.md` cross-cutting](../ARCHITECTURE.md).

## Scope

Built as `modules/multiplayer/` behind `NetServer`. Transport is **GameNetworkingSockets (GNS)** via `libs/zig-cpp-net-stack-adapter/` (C++ adapter sub-repo). Decision recorded 2026-05-29 — supersedes the earlier ENet-vs-GNS open question.

**Two build modes, one protocol:**

- **Steam build** (`-Dsteam=true` links Steamworks SDK): unlocks Steam Datagram Relay (SDR) for NAT-less connectivity, Steam auth tickets, Steam lobbies + friend invites. Players on Steam never deal with NAT.
- **Standalone build** (default — for itch.io, GOG, direct downloads): GNS standalone with libsodium-backed encryption + ICE-style NAT punching (covers ~70-80% of consumer NATs). Symmetric-NAT fallback is "LAN-only or supply a TURN relay."

Same wire protocol on both builds; the build flag selects which transport features are active. See [`external-libs-catalog.md` § 3](../external-libs-catalog.md) for the adapter and [`engine-vs-game.md`](../engine-vs-game.md) for the C++ wrapping rationale.

**Why GNS over ENet** (recorded for future-me; full argument in `project_multiplayer_transport_gns` memory):

- Encryption, connection state machine, lane priorities, and NAT-punching come for free → ~1-2 weeks less engine work than building those on top of ENet.
- Steam Datagram Relay materializes the "co-op without ceremony" promise (`vision.md:50`) for Steam users; ENet would require us to operate a STUN/TURN service.
- Same library serves Steam + itch + GOG via the build flag; ENet would require parallel Steamworks-on-top integration.
- Cost: bigger dep (~150 KB libsodium + protobuf headers), C++ adapter (vs ENet's direct `@cImport`), and the Luanti-fork validation pathway no longer applies (Luanti uses ENet natively).

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

- Edit authority model (host-decides vs client-prediction-with-rollback)
- Server discovery for standalone builds (LAN broadcast vs minimal lobby service we host)
- TURN relay for symmetric-NAT fallback on standalone builds (skip and require port-forwarding vs self-host vs paid service)
- Whether to expose Steam lobbies to non-Steam builds via the same `NetServer` API surface (transparent vs build-mode-aware)

## Milestone (from ROADMAP)

4 players co-op the dungeon-clear loop with acceptable latency; a dedicated-server instance ships tick-time p50/p95/p99 + player count to a local Grafana dashboard.
