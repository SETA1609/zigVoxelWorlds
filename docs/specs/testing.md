# Test Strategy Spec

> The test pyramid for zVoxRealms — what kinds of tests exist, where they live, what they verify, when they run. Closes [`gaps.md`](../gaps.md) §2.2.G + #32. Should be in place from Phase 2 so test scaffolding is built in alongside the module system.

## Scope

A practical test strategy for a solo dev shipping a learning project that needs to ship a real game. Heavy on tests that catch regressions cheaply; light on ceremonial coverage targets. Per the engine's [performance budget rule](../mission.md) (50–60 FPS on iGPU), perf regression is a first-class test type alongside correctness.

## The pyramid

```text
                        ┌──────────────────┐
                        │   Manual play    │   ← Phase 11+: vertical-slice sessions, exploratory
                        ├──────────────────┤
                        │ End-to-end / golden│   ← scene → save → reload, screenshot diffs
                        ├──────────────────┤
                        │ Integration tests │   ← module-level: voxel-edit + save + load, etc.
                        ├──────────────────┤
                        │ Property + fuzz   │   ← serialization round-trips, parser fuzzing
                        ├────────────────────┤
                        │     Unit tests     │   ← Zig `test "..." {}` co-located with source
                        └────────────────────┘
```

Bottom = fastest, most numerous, run on every save. Top = slowest, fewer, run on CI / pre-release.

## Layer 1 — unit tests (`test "..." {}` blocks)

Co-located with source. Every meaningful function gets at least one test block.

```zig
// src/core/handle.zig
pub const Handle = packed struct(u64) { ... };

pub fn isValid(h: Handle) bool { ... }

test "Handle.isValid returns true for in-range generation" {
    const h = Handle{ .index = 5, .generation = 1 };
    try std.testing.expect(isValid(h));
}

test "Handle.isValid returns false for INVALID_HANDLE" {
    try std.testing.expect(!isValid(INVALID_HANDLE));
}
```

### Rules

- Co-located in the same file as the production code
- Run via `zig build test`
- **No I/O, no allocations beyond test-arena, no global state mutation**
- < 1 ms per test (typical: < 100 µs)
- Coverage target: the **happy path + each documented error case**. Don't chase 100%.

### What to unit-test

✅ Pure functions (math, parsing, validation, serialization primitives)
✅ Data-structure operations (handle table, archetype query, etc.)
✅ Algorithm correctness (greedy meshing on a 2-voxel-type chunk produces expected mesh)
✅ Edge cases documented in the function's contract

❌ Don't unit-test plumbing (struct field access, simple getter/setter)
❌ Don't unit-test code that just calls into Vulkan / Jolt / OS — that's integration territory

## Layer 2 — property + fuzz tests

Best-bang-for-buck layer for systems with many possible inputs.

### Serialization round-trip

For every save-format section, every CSV/TOML schema, every network packet type:

```zig
test "ChunkDelta round-trips through serialize + deserialize" {
    var arena = std.heap.ArenaAllocator.init(std.testing.allocator);
    defer arena.deinit();
    
    var rng = std.Random.DefaultPrng.init(42);
    for (0..100) |_| {
        const original = ChunkDelta.random(arena.allocator(), &rng);
        const bytes = try original.serialize(arena.allocator());
        const restored = try ChunkDelta.deserialize(arena.allocator(), bytes);
        try std.testing.expectEqual(original, restored);
    }
}
```

Property: for any random input, `deserialize(serialize(x)) == x`. Catches off-by-ones, missing fields, alignment bugs.

### Parser fuzzing

For TOML / CSV / `.po` parsers and the C ABI input validators:

```zig
test "fuzz: project.toml parser doesn't crash on random bytes" {
    var rng = std.Random.DefaultPrng.init(0xDEADBEEF);
    var buf: [4096]u8 = undefined;
    for (0..10_000) |_| {
        const len = rng.random().uintLessThan(usize, buf.len);
        rng.random().bytes(buf[0..len]);
        _ = parseProjectToml(buf[0..len]) catch {};   // ok to error; don't crash
    }
}
```

Run as part of `zig build test`. For deeper fuzzing, hook AFL/libfuzzer at Phase 12+.

## Layer 3 — integration tests (`tests/` directory)

Multi-module scenarios. Not co-located — these need their own files since they exercise multiple subsystems.

### Test categories

- **Save-load round-trip** — spawn world, place voxels, save, restart-load, verify same state. One per save-section version
- **Module init order** — verify Core → Servers → Scene → Editor ordering doesn't have hidden circular deps
- **Mod load + ABI compat** — load a test mod, verify it can call every public ABI function without crash
- **Voxel-edit → mesh-update → render** — full pipeline test, verify the right chunks re-mesh
- **Network round-trip simulation** — client sends voxel edit; server applies; broadcasts; second client receives same state (with injected 100ms lag / 5% loss to verify reconciliation)

### Example

```zig
// tests/save_roundtrip_test.zig
test "world with voxel edits + entities round-trips" {
    var harness = try TestHarness.init();
    defer harness.deinit();
    
    const world = try harness.engine.spawnWorld(.{ .seed = 12345 });
    
    // Make some edits
    try harness.engine.setVoxel(world, .{ 5, 64, 5 }, voxel_iron);
    try harness.engine.spawnEntity(world, "torch", .{ 5, 65, 5 });
    
    // Save
    const save = try harness.engine.save(world);
    
    // Tear down + restart
    harness.engine.destroyWorld(world);
    
    // Load
    const restored = try harness.engine.loadSave(save);
    
    // Verify
    try std.testing.expectEqual(@as(u32, voxel_iron), harness.engine.getVoxel(restored, .{ 5, 64, 5 }).id);
    const entity = try harness.engine.findEntityNear(restored, .{ 5, 65, 5 });
    try std.testing.expect(entity != INVALID_HANDLE);
}
```

## Layer 4 — golden-image / visual regression

Catches rendering regressions. Compare rendered output to a known-good reference image.

### How it works

1. Test sets up a scene (specific world seed + camera pose + light config)
2. Renders one frame to an offscreen PNG
3. Compares to `tests/golden/<test_name>.png` pixel-by-pixel (with small tolerance for float jitter)
4. On mismatch: writes the diff image + new image to a CI artifact for human review

### What to golden-test

- Voxel-chunk meshing on a representative chunk
- Lighting on a torch scene
- Particle effects (deterministic with seed)
- UI panels at default + max font scale + colorblind-mode-1

Skip golden-testing rapidly-changing pre-1.0 systems — burns more time on artifact updates than catches bugs. Add per-system as that system stabilizes.

### Tolerance

- ~0.5% per-pixel delta allowed (float-precision jitter is unavoidable)
- Structural similarity index (SSIM) > 0.98 (catches subtle issues color diff misses)
- Cite: <https://en.wikipedia.org/wiki/Structural_similarity_index_measure>

## Layer 5 — performance regression

Per [`mission.md`](../mission.md) Operating Principle 5: 50–60 FPS on iGPU is non-negotiable. Without regression detection, slow code creeps in unnoticed.

### Benchmark suite

`tests/bench/` contains scenarios that exercise hot paths:

- 10k entities + ECS scheduler tick
- Chunk mesh-gen for a representative chunk
- Voxel-set on a chunk with neighbor relight
- Save serialization for a 1 km² populated world
- Path-find through a 50-room dungeon (per [`specs/ai.md` — TBD](../gaps.md))

### Measurement

- Each benchmark: 1000 iterations, report median + p95 + p99
- Compare against a baseline checked into `tests/bench/baseline.json` (per-platform — Linux + Windows + iGPU profile)
- Fail CI if any benchmark regresses by > 10% vs baseline
- Manual baseline-bump via `zig build update-baseline` when intentional perf changes ship

### Target hardware in CI

Per the performance target: at minimum, run benchmarks against a synthetic "low-end" target via CPU + GPU constraints. Ideal long-term: GitHub Actions with self-hosted iGPU runners. For pre-v1.0: rely on relative regression (% from baseline) on whatever CI runs, accept absolute numbers won't match production iGPU until self-hosting.

## Layer 6 — manual / exploratory

Not automatable. Done by you + (later) playtesters.

- Vertical-slice play sessions per phase milestone
- Bug-bashing before each release tag
- Mod compatibility testing with a corpus of third-party mods (post Phase 14)

Captured as Issue templates once the repo opens.

## CI integration

Every PR runs:

```yaml
# .github/workflows/ci.yml (target shape)
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: goto-bus-stop/setup-zig@v2
        with: { version: '0.16.0' }
      - run: zig build test               # unit + property + integration
      - run: zig build test-golden        # golden-image regression
      - run: zig build bench-regression   # perf regression vs baseline
      - run: zig fmt --check src/
      - run: clang-format --dry-run -Werror $(find src -name '*.cpp' -o -name '*.h')
```

Times to keep honest:

- Unit tests: < 30 seconds
- Integration tests: < 2 minutes
- Golden tests: < 1 minute (or skipped if no changes to renderer)
- Perf benchmarks: < 5 minutes

Total < 8 minutes per PR.

## What we don't test

Some things look testable but the test cost > the bug cost:

- Vulkan validation layer warnings — handled by the validation layer itself when `-Dvalidation=true`; CI shouldn't gate
- Specific GPU vendor quirks — too many; rely on Vulkan validation + bug reports
- Cross-mod conflicts — combinatorial explosion; rely on the in-engine conflict warnings (per [`specs/mod-manager.md`](mod-manager.md))
- Audio-bus mix correctness — humans hear it; no good automated test

## Per-phase test obligations

| Phase | Tests required to "land" |
| --- | --- |
| Phase 2 (Module System) | Module init-order test; ABI version-check test |
| Phase 3 (Voxel Core) | Mesh-gen unit + property tests; chunk-delta round-trip |
| Phase 4 (Asset Pipeline) | Importer round-trip per asset type; assetdb GUID stability test |
| Phase 5 (Physics) | Body create/destroy + handle reuse; bench: 1000-body simulation |
| Phase 7 (ECS) | Archetype query correctness; processor dep-graph cycle detection; bench: 10k-entity tick |
| Phase 10 (Multiplayer) | Network round-trip with lag/loss; client prediction reconciliation |
| Phase 13 (Export) | Exported game runs on clean machine; PCK three-location load |

## Closes

- [`gaps.md`](../gaps.md) #32 — Test strategy
- [`gaps.md`](../gaps.md) §2.2.G — Test strategy beyond unit tests

Sources:

- Zig's built-in test runner: <https://ziglang.org/documentation/master/#Zig-Test>
- Property-based testing rationale: <https://hypothesis.works/articles/what-is-property-based-testing/>
- Golden-image SSIM: <https://en.wikipedia.org/wiki/Structural_similarity_index_measure>
- Game-engine perf-regression precedent: id Software's Tech blog, Unreal's Stat-system telemetry
