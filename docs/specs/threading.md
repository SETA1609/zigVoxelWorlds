# Threading Model Spec

> The thread topology and inter-thread communication that all servers + modules must agree on. Blocks Phase 2. Gap: [`gaps.md` § 2.1.I + § 3 #12](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.I](../gap-references.md).

## Scope

Define which threads exist in zVoxRealms, what each owns, and how they communicate. Single decision; affects every Server / Backend / module surface. Must be locked before Phase 2 (Module System + Server Pattern) starts coding.

## Thread topology

| Thread | Owns | Communication |
| --- | --- | --- |
| **Main** | Input, event dispatch, scene tree mutation, gameplay tick, UI | Drives all other threads via per-server command queues |
| **Render** | Vulkan command buffer recording, GPU submission | Receives draw commands via `RenderServer` queue; never touches scene state directly |
| **Voxel-gen pool** (N workers) | Chunk generation from seed, meshing | Pulls generation requests; pushes finished chunks back to main |
| **Physics** | Jolt world step, collision callbacks | Game-thread ↔ physics-thread marshalling (CommandQueueMT-style); double-buffered state |
| **Audio** | miniaudio mix thread, stream decoding | One-way: main thread enqueues "play sound X at position Y"; audio drains |
| **Network** | UDP socket I/O, packet (de)serialization | Pushes received packets to main; pulls outbound packets from a queue |
| **AI** (optional pool) | Pathfinding queries, BT tick (deferred-results) | Async query model; main thread fires query + gets handle, polls or callback |

## Communication primitive

A single shared abstraction: **SPSC ring buffer** (single-producer, single-consumer) per server. Main thread is producer; the server's owning thread is consumer.

- Lock-free where possible (Zig's `std.atomic`)
- Variable-size commands (header with type + length, payload follows)
- One ring per server (render, physics, voxel-gen-results, audio, network-out)
- Sync barriers: when main thread needs a return value, it submits + spins on a per-command done-flag

For shared state that *both* threads read (rare — try to avoid), use double-buffering or copy-on-write.

## Borrowed pattern

Godot's `CommandQueueMT` ([`gap-references.md` § 2.1.I](../gap-references.md)) — main thread enqueues; render thread drains; variable-size commands. Same shape; pure Zig implementation.

## Rules

1. **Scene state is main-thread-only.** Other threads never directly read/write `World`, `Chunk`, `Entity`. They send commands referencing `Handle`s.
2. **Server APIs are thread-safe by design.** Calling `RenderServer.drawMesh(handle)` from main thread → enqueues to render thread's ring. Documented per function: "main thread only," "server-owning thread only," "any thread."
3. **Server-internal threading is the server's business.** `VoxelServer`'s gen-pool is internal; callers don't see it.
4. **No global mutexes.** If you need to coordinate, use a queue + a server.
5. **Backends are single-threaded.** The Vulkan backend runs on the render thread. The Jolt backend runs on the physics thread. The complexity stops at the server boundary.

## Open decisions

- Exact ring-buffer size per server (start at 64 KB, tune)
- Render thread is mandatory or optional? (default mandatory; `-Dsingle-thread=true` for debugging)
- Voxel-gen worker count: hardcoded vs CPU-detection?
- Job system or per-server-thread? (Lean toward per-server — simpler; revisit if it becomes a bottleneck)

## Milestone (validates this spec)

Phase 2 milestone — `RenderServer.drawMesh(handle)` called from main thread results in a cube rendered without main thread ever touching Vulkan. Verified by setting a breakpoint on a Vulkan call and confirming the stack trace shows the render thread, not main.
