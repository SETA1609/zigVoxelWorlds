# zVoxRealms

> A Zig-first voxel game engine for solo developers shipping Daggerfall-style RPGs, voxel Stardew sims, Atelier-style crafting games, and rogue-like dungeon crawlers — co-op-ready by default, lean on low-end hardware, modded as easily as Minecraft, exported as a single small binary that contains only what the game needs.

**Status:** Phase 0 (Foundation) — planning + scaffolding. Engine code is a hello-world stub. Real implementation starts at [Phase 1](docs/ROADMAP.md#phase-1-window--vulkan-basics-next).

**License:** [Apache 2.0](LICENSE). Games made with the engine are the developer's IP under their own EULA. See [`docs/licensing.md`](docs/licensing.md).

## Read this first

If you've just landed here:

1. [`docs/vision.md`](docs/vision.md) — what zVoxRealms aspires to be
2. [`docs/mission.md`](docs/mission.md) — current operating statement
3. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — engine layers, distribution model, server pattern
4. [`docs/ROADMAP.md`](docs/ROADMAP.md) — 15 phases from foundation to v1 vertical slice
5. [`docs/guard.md`](docs/guard.md) — **AI collaboration rules**; this is a learning project

For the full doc index, see [`docs/`](docs/) and the per-subsystem specs in [`docs/specs/`](docs/specs/).

## What this engine is (and isn't)

**Is:**

- A voxel engine designed for RPG/sim/crafting/rogue-like shapes
- Engine-as-app — one binary opens as a Project Manager or editor
- Per-project tree-shaken dynamic-lib export (Zig's bundled LLVM makes this possible; C++ engines can't do it cleanly)
- Multiplayer-by-default — host-authoritative, 4-player co-op, optional 40–50p dedicated servers
- First-class modding via stable `extern "C"` ABI (no Lua, no GDScript)
- Performance target: 50–60 FPS on i3 / Ryzen 3 + integrated graphics

**Isn't:**

- A general-purpose engine (no 2D, no racing, no RTS)
- Photorealistic — stylized voxel only
- MMO-scale (40–50 player ceiling)
- A no-code editor (game logic is Zig + C++)
- Console-first (Linux + Windows desktop; Android later)

See [`docs/vision.md`](docs/vision.md) for the full scope and [`docs/mission.md`](docs/mission.md) for what's out of scope.

## Build

Requires [Zig 0.16+](https://ziglang.org/download/). No external C/C++ toolchain needed — Zig ships LLVM/Clang.

```bash
zig build              # build the binary
zig build run          # build + run
```

Release modes:

```bash
zig build -Doptimize=ReleaseFast
zig build -Doptimize=ReleaseSafe
zig build -Doptimize=ReleaseSmall
```

Cross-compile:

```bash
zig build -Dtarget=x86_64-windows
zig build -Dtarget=x86_64-linux-gnu
```

For the Zig + C + C++ build system internals, see [`docs/cheat-sheet.md`](docs/cheat-sheet.md).

## Repo layout

```text
.
├── build.zig                 # Zig build system
├── src/                      # Engine source (currently hello-world stub)
├── libs/                     # Vendored adapter sub-repos (each MIT)
├── docs/                     # Planning + reference + specs (the bulk of work to date)
├── LICENSE                   # Apache 2.0
├── LICENSES.md               # Third-party attributions
├── NOTICE                    # Copyright notice
├── CONTRIBUTING.md           # Contribution guidelines
├── CHANGELOG.md              # Project history
├── SECURITY.md               # Vulnerability reporting
└── .clang-format             # Google C++ style + project tweaks
```

The target layout (after Phase 0 completes) is documented in [`docs/project-structure.md`](docs/project-structure.md).

## Reference engines

zVoxRealms borrows concrete patterns from Hazel, Luanti-Custom, Godot, and UnrealEngine — see [`docs/engine-references.md`](docs/engine-references.md) for the verified file-level pointers and goal-fit tags (what to copy vs. skip).

## Contact

All contact runs through GitHub:

- General + bugs: GitHub Issues / Discussions on the repo (once public)
- Security: [GitHub Security Advisories](https://github.com/SETA1609/zigVoxelWorlds/security/advisories)
- Maintainer: [@SETA1609](https://github.com/SETA1609)

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). The project isn't open to feature PRs until Phase 2 lands and the architecture is real in code — but doc PRs, typo fixes, and link-rot reports are welcome.
