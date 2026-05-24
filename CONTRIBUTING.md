# Contributing to zVoxRealms

Thanks for your interest. zVoxRealms is in **Phase 0 (Foundation)** and isn't yet ready for outside contributions — the architecture is still being validated by the maintainer. This document captures what contributions will look like once the project opens up (likely once Phase 2 lands and the module system is real).

## Current status

- **Phase:** 0 — Foundation. See [`docs/ROADMAP.md`](docs/ROADMAP.md).
- **Code volume on disk:** minimal stub (hello-world Zig + C + C++ build).
- **Accepting:** documentation PRs, typo fixes, link-rot reports. **Not yet:** feature PRs, engine subsystem rewrites.
- **Issue tracker:** to be enabled when the repo opens publicly.

## What this project is and isn't

Before opening a PR, read:

- [`docs/vision.md`](docs/vision.md) — what zVoxRealms aspires to be (Daggerfall-class voxel RPG + Stardew/Atelier/rogue-like targets, engine-as-app, per-project tree-shaken export)
- [`docs/mission.md`](docs/mission.md) — current operating posture
- [`docs/guard.md`](docs/guard.md) — **AI assistance is intentionally constrained**; this is a learning project for the maintainer. PRs that are clearly AI-generated boilerplate will be declined.

Out-of-scope features (will be closed without discussion):

- 2D / racing / RTS / non-voxel game shapes
- AAA-photorealism rendering
- MMO-scale multiplayer (40–50 player ceiling is the cap)
- macOS / console ports beyond what `ROADMAP.md` describes
- Visual scripting / no-code editor
- Scripting languages other than Zig + C++

## Licensing

zVoxRealms is **Apache License 2.0** — see [`LICENSE`](LICENSE) and [`docs/licensing.md`](docs/licensing.md).

**No CLA required.** Apache 2.0 §5 ("inbound = outbound") means contributions are licensed under the same Apache 2.0 terms automatically. By submitting a PR you affirm:

- You have the right to submit the code (it is yours, or you have permission)
- The code is licensed under Apache 2.0
- You have the right to grant the patent license in Apache 2.0 §3

Optional but appreciated: `Signed-off-by:` trailers in commits (DCO-style). Not yet enforced.

**Adapter sub-repos** (e.g. `libs/zig-cpp-vulkan-stack-adapter/`) use their own license — typically MIT. See each sub-repo's `LICENSE` and [`docs/licensing.md`](docs/licensing.md) § Adapter sub-repos.

**Forbidden dependencies:** GPL, LGPL, AGPL, SSPL, Commons Clause, "non-commercial only" licenses. PRs introducing these will be declined. See [`docs/external-libs-catalog.md`](docs/external-libs-catalog.md) § 6.

**Reference engines — no verbatim ports.** zVoxRealms studies Hazel, Luanti, Godot, and UnrealEngine as patterns + inspiration only. The rule applies to all four regardless of license:

- ❌ **Unreal source-available EULA** — direct copies would violate Epic's license and trigger lawsuit risk
- ❌ **Luanti LGPLv2.1+** — verbatim copies would force the engine GPL, killing the Apache 2.0 strategy
- ❌ **Godot MIT** + **Hazel** — license-compatible but verbatim copies create attribution debt + are bad form

The acceptable workflow: read the reference, understand the *pattern*, close the file, write your own Zig implementation. PRs containing translated reference code will be declined regardless of license. See [`docs/engine-references.md` § Legal](docs/engine-references.md) and [`docs/licensing.md` § Reference-engine policy](docs/licensing.md).

## Code style

### Zig

- `zig fmt` is enforced. Run before commit.
- No global mutable state; pass allocators + contexts explicitly.
- Tests co-located with source in `test "..." {}` blocks where reasonable.
- Comments: WHY-only. Don't restate what the code says.

### C / C++

- Follow [`docs/cpp-style.md`](docs/cpp-style.md) — Google C++ Style Guide as baseline with documented deviations.
- `clang-format` runs on every C/C++ file. The root `.clang-format` is `BasedOnStyle: Google` + project tweaks.
- **Every `extern "C"` function must be `noexcept` and catch all exceptions.** This is non-negotiable for the mod ABI.

### Commit rules

**Atomic commits — one concern per commit.**

A commit should be the smallest unit of meaningful change. Split when it grows. Specifically:

- ✅ One feature, one fix, one refactor, one doc update — per commit
- ✅ The repo compiles + tests pass at every commit (`git bisect` stays useful)
- ✅ Each commit is revertable without breaking unrelated functionality
- ❌ Don't mix a refactor with a feature in one commit — split them
- ❌ Don't mix unrelated doc updates with code changes
- ❌ Don't bundle "fixed typo + added module + reorganized 5 docs" — that's three commits

When in doubt: would future-you, hitting `git log` after six months, understand each commit at a glance? If no, split it.

**Conventional-ish format — `<type>(<scope>): <subject>`:**

```text
feat(voxel): add greedy mesher for opaque blocks
fix(render): clamp view distance to 8 chunks on iGPU
docs(roadmap): shrink Phase 0 MVP definition
chore(deps): bump Jolt to 5.1.0
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`.

**Scope:** subsystem name (`voxel`, `render`, `physics`, `net`, `editor`, `core`, `modules`, `adapters`, or the doc filename for doc commits).

**Subject line — short and descriptive, ≤ 150 characters (tweet-sized max).** Imperative mood ("add X", not "added X"), lowercase, no trailing period.

> Note: GitHub's `git log --oneline` view truncates around 72 chars. Aim for **≤ 72 chars** when feasible (fits all common UIs); only stretch toward 150 when the extra precision is worth losing the truncation.

**Body — optional but recommended for non-trivial commits.** Explains the *why*, not the *what* (the diff shows what). Wrap at 80 cols. Use bullet lists for multi-step rationale.

**Author + committer:** both fields must use the GitHub noreply format `SETA1609 <123449150+SETA1609@users.noreply.github.com>` per [no-PII policy](../docs/licensing.md). Apply via `--author` flag + `GIT_COMMITTER_NAME` + `GIT_COMMITTER_EMAIL` env vars. Never commit with the global git-config default if it leaks PII.

### Branching strategy — trunk-based with build flags

**`main` is always green and is where everything lands.** No long-lived `develop` / `release-*` / per-feature integration branches. Resolves [`gaps.md` § 3 #29](docs/gaps.md).

| Branch | Lifetime | Used for |
| --- | --- | --- |
| `main` | forever | The trunk. Always builds, always tested. All work merges here |
| Feature branches (`feat/<name>`, `fix/<name>`, `docs/<name>`) | days to weeks | One concern at a time; merge to `main` when done; deleted after merge |
| Release tags (`v0.1.0`, `v0.2.0`, ..., `v1.0.0`, ...) | forever | Snapshot points. No long-lived release branches — tag and move on |

**Feature flags handle "not yet ready" code in trunk.** This project already uses `-Dsteam=true`, `-Dtracy=true`, `-Dplatform_backend=glfw|native` build options as the gating mechanism for optional / WIP features. The same pattern applies to gameplay-side WIP — gate behind a `-Dfeature_<name>` flag rather than parking on a long-lived branch. Cheaper to merge little-and-often than to maintain a branch that drifts.

**Why trunk-based + flags, not GitFlow:**

- Solo-dev pre-1.0 — no team coordination overhead that GitFlow's release-train solves
- The engine-as-app + per-project tree-shake model already implies build flags as the toggle mechanism; trunk-based + flags is the same pattern at the source level
- Adapter sub-repos already work this way (each has one `main`)
- Reference precedent: Hazel, Mach, and most solo-dev Zig engines use single-branch trunk

**Hotfixes after v1.0 ships:** branch from the release tag, cherry-pick fixes, tag the patch (`v1.0.1`), merge the hotfix forward to `main` if not already there. No long-lived release branches — branches exist only as long as the hotfix work takes. Pattern formalized when v1.0 actually approaches (post-Phase 15 concern).

**For now:** single `main` branch; feature branches as needed; tags at release points. The hotfix machinery doesn't matter until you have something to hotfix.

## Pull request expectations

- One concern per PR. Split if it grows.
- PRs that touch architecture (`src/core/`, `src/servers/`, `src/scene/`, the module dispatch, the C ABI surface) need a discussion in an issue first.
- PRs that touch performance-critical code (voxel meshing, chunk streaming, render passes) include a benchmark comparison.
- PRs that change docs follow the existing structure; if you're moving content between docs, say where it came from in the PR body.
- Adapter sub-repos (under `libs/`) maintain their own contribution rules. Defer to each sub-repo's `CONTRIBUTING.md` if one exists.

## What you can help with right now

In priority order, even pre-public:

1. **Doc review** — find inconsistencies between docs (numerous; see [`docs/gaps.md`](docs/gaps.md))
2. **Reference engine source pointers** — if a path in [`docs/engine-references.md`](docs/engine-references.md) is stale, file an issue
3. **Typos, broken links, formatting** — direct PR welcome

After Phase 2 lands (module system + Server pattern), feature contributions will open.

## Code of conduct

Be respectful. Disagree about technical decisions in technical terms. Personal attacks are not accepted.

## Contact

All contact runs through GitHub. No email channel.

- **General questions / discussions:** GitHub Discussions on the repo (once public)
- **Bug reports / feature requests:** GitHub Issues on the repo (once public)
- **Security vulnerabilities:** [GitHub Security Advisories](https://github.com/SETA1609/zigVoxelWorlds/security/advisories) — see [`SECURITY.md`](SECURITY.md)
- **Maintainer:** [@SETA1609](https://github.com/SETA1609)

Thanks for reading.
