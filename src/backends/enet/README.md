# `src/backends/enet/`

> `net_server` implementation, ENet variant. Wraps [ENet](http://enet.bespin.org/) through `libs/zig-cpp-net-stack-adapter/` (planned).

Holds:

- Host create / connect / disconnect
- Reliable + unreliable channels
- Packet (de)serialization hooks
- Replication frame integration (per `modules/multiplayer/`)

A second backend, `gns/`, may land later for GameNetworkingSockets. The chosen backend is selected at compile time by a `build.zig` flag (e.g. `-Dnet=enet`). The server-level API in `src/servers/net_server.zig` is identical regardless.

**Reference patterns:** Luanti `src/servermap.cpp` + `src/clientmap.cpp` for `emergeBlock` + interest management; Unreal Iris `ReplicationSystem.h:69` for the handle-based delta-replication shape. See [`engine-references.md`](../../docs/engine-references.md) § Luanti and § Unreal · Iris.
