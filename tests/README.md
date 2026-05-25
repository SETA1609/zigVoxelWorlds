# `tests/`

> Integration tests. **Unit tests live next to source** (`foo.zig` keeps its tests inside the same file with `test "name" { ... }`).

## What goes here

- Multi-module end-to-end flows (project load → spawn → tick → save → reload)
- Importer round-trips (PNG → KTX2 → load → blit)
- Replication frame coherence (host frame N + delta = client frame N+1)
- Save-delta + seed-deterministic regeneration round-trips
- Mod-loader layered override resolution
- Build-graph snapshots: assert that `tools_enabled = false` builds elide `editor/`

## What does NOT go here

- Per-function unit tests (live next to source, in the same `.zig` file)
- Manual playtest checklists (live in [`docs/guard.md`](../docs/guard.md) § Sanity checks or per-spec test plans)
- Performance / FPS regression tests — those land here later under `tests/perf/` once Phase 5+ benchmarks exist

## Running

```sh
zig build test            # all tests (in-source + integration)
zig build test-integration  # just this directory (planned alias; TBD wired in build.zig)
```

## References

- Zig's testing model: <https://ziglang.org/documentation/master/#Zig-Test>
- [`specs/testing.md`](../docs/specs/testing.md) — the project's testing strategy
