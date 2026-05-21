# Changelog

All notable changes to zVoxRealms are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project will adhere to [Semantic Versioning](https://semver.org/spec/v2.0.0.html) once a 1.0 release exists.

Until v1.0, expect breaking changes between **every** minor version. The engine is in Phase 0 (Foundation). Pre-1.0 versions track planning milestones, not stability.

## [Unreleased]

### Planning phase complete (Phase 0 — In Progress)

#### Removed

- `docs/myGoals.md` — original scope brain-dump; substance fully migrated to `vision.md`, `mission.md`, `ARCHITECTURE.md`, `tech-stack.md`, `engine-vs-game.md`, `engine-references.md`, and the per-phase specs under `docs/specs/`. Git history preserves the original.

#### Added

- Apache License 2.0 (`LICENSE`, `NOTICE`)
- Third-party attribution catalog (`LICENSES.md`)
- Full doc suite under `docs/`:
  - `vision.md` — long-horizon vision (four target games, differentiators, 3-year horizon)
  - `mission.md` — current operating statement + six principles + decision process
  - `ARCHITECTURE.md` — engine layers, distribution model, module system, server pattern, scripting, editor panels, save model, world editability policy
  - `ROADMAP.md` — 15 phases, vertical-slice-driven, MVP scoped to Phase 1+2
  - `tech-stack.md` — Zig-first principle, data layer (TOML + binary + JSON-where-needed), asset pipeline, observability three-tier, scripting, in-engine code editor (bundled Neovim), runtime debug output (libghostty)
  - `project-structure.md` — target on-disk layout
  - `engine-references.md` — concrete patterns to borrow from Hazel, Luanti, Godot, UnrealEngine (paths verified)
  - `external-libs.md` — three-tier integration catalog (Zig-native / direct cImport / adapter sub-project)
  - `engine-vs-game.md` — engine binary vs exported game responsibilities + library categorization
  - `licensing.md` — strategy: Apache 2.0 dev + Steam stable; permissive deps only; adapter sub-repos MIT (Apache for codec-adjacent wrappers); trademark separate; no CLA
  - `cpp-style.md` — Google C++ Style Guide baseline with project deviations (exceptions inside / forbidden at C boundary; RTTI per wrapped lib; C++23)
  - `guard.md` — rules for AI collaboration in this learning project
  - `gaps.md` — consolidated gap analysis: canonical engine-cores alignment + missing systems (tier-ranked) + 38 open decisions
  - `mvp.md` — MVP definition (Phase 1 + Phase 2)
- Repo-root meta files: `CONTRIBUTING.md`, `CHANGELOG.md`, `SECURITY.md`
- `.clang-format` at repo root + adapter sub-repo
- MIT `LICENSE` for `libs/zig-cpp-vulkan-adapter/`

#### Decisions made (no code yet)

- **License:** Apache 2.0 for engine, MIT default for adapter sub-repos, no GPL deps
- **Scripting:** Zig + C++ via stable `extern "C"` ABI; same surface as mods
- **Editor:** bundled Neovim for code editing, libghostty for playtest debug log
- **Observability:** Tracy (dev) + libghostty (editor logs) + OpenTelemetry/Grafana (dedicated servers)
- **Save model:** seed-deterministic baseline + voxel deltas + structured game state (binary, versioned)
- **World editability:** per-scene TOML policy (`full` / `none` / `voxel_type_allowlist` / `coord_range_allowlist` / `tag_allowlist` / `script`)
- **MVP scope:** Phase 1 + Phase 2 only (Vulkan window + module system + RenderServer + hello module + rotating cube)
- **Distribution model:** engine-as-app (Project Manager + editor in one binary, re-exec on project open); export tree-shakes a per-project `libzvox-runtime.{so,dll}` via `zig build-lib -dynamic`

#### Not yet started

- On-disk source layout matching `project-structure.md`
- Build artifact rename from `demo` to `zvoxrealms`
- Vulkan triangle (Phase 1)
- Module system implementation (Phase 2)

---

## How to update this file

When a change lands:

1. Add an entry under `[Unreleased]` in the appropriate section: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`
2. One line per change. Reference the PR or issue if applicable
3. When cutting a release, rename `[Unreleased]` → `[X.Y.Z] - YYYY-MM-DD` and start a fresh `[Unreleased]`

Pre-1.0 entries can be terse. Once 1.0 ships, every entry should explain the user-visible impact.

## Sources

- [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/)
- [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html)
