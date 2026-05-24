# Licensing Strategy

> What license the engine uses, what license games made with it use, and why. Authoritative reference for anyone deciding "can I use this commercially?" The actual legal text lives in [`LICENSE`](../LICENSE) (Apache 2.0). Third-party attributions live in [`LICENSES.md`](../LICENSES.md).

## TL;DR

- **The engine is open source under Apache License 2.0.** Anyone can use it to build games — including commercial games — without paying royalties or asking permission.
- **Your games made with the engine are your own IP**, under your own commercial EULA. You ship them on Steam (or anywhere) as proprietary products.
- **No copyleft surprises:** all engine dependencies are permissive (MIT / Apache / BSD / zlib / public-domain). GPL / AGPL / SSPL dependencies are forbidden by policy.

## Why Apache 2.0 (not MIT, not GPL, not custom)

| License | Why considered | Why rejected (or chosen) |
| --- | --- | --- |
| **Apache 2.0** ✅ chosen | Permissive + explicit patent grant | Best balance for a multi-dependency engine that wraps C++ vendor libs. Patent §3 protects against troll claims from contributors. NOTICE convention makes attribution mechanics standard. |
| MIT | Most permissive | No explicit patent grant. Acceptable but Apache 2.0 is strictly safer with the same commercial-friendly outcome. |
| BSD-3 | Permissive, simple | Same as MIT — no patent grant. No advantage over Apache 2.0 here. |
| GPL-3 / AGPL-3 | Copyleft prevents competitors from selling closed forks | Forces every game made with the engine to be open source. Kills the ecosystem we want (devs shipping their commercial games on Steam). |
| Custom source-available (Unreal-style) | Royalty model, source visible but not open | Solo-dev legal enforcement is impractical; "not really open source" cuts off community contributions and adoption. |
| Dual-license MIT + commercial | Sell commercial licenses for use-cases where MIT terms aren't enough | Solves a problem we don't have — there's no scenario where MIT terms aren't enough for Scenario A (selling our own games). |

Reference: <https://www.apache.org/licenses/LICENSE-2.0>

## Two distribution channels, one license

zVoxRealms ships in two forms with **the same Apache 2.0 source**:

1. **Open source / dev** — `git clone` and `zig build`. Free. Includes everything.
2. **Steam stable build** — curated, tested, auto-updating commercial product on Steam. Costs money. Includes everything plus convenience (no toolchain setup, no manual builds, official support, signed binaries).

The license is the same. What differs is **distribution and packaging**, not legal terms. This is the same model as:

- Red Hat selling RHEL on top of permissive Linux
- Docker Inc. selling Docker Desktop on top of containerd
- The id Software releases (Doom source open + Doom BFG on Steam)
- Most "commercial open source" companies

A buyer of the Steam build can technically also clone the dev branch and use it for free. Most won't bother — they're paying for **the polished product** (curation, support, updates, signing, no compile step), not the source code. That is a legitimate business.

## The "stable branch" on Steam is a release tag, not a separate license

`main` and the `stable-vX.Y` tags are all Apache 2.0. The Steam product is built from a specific tag. There is no proprietary branch.

Implications:

- A contributor can read the same source code as a paying Steam customer
- A community fork can exist (under Apache 2.0) — but cannot use the **zVoxRealms** trademark (see § Trademarks)
- The Steam build can include build-time integrations (Steam Workshop, achievements, cloud saves) that the open dev build doesn't ship by default — but those are gated by `-Dsteam=true` and use the proprietary [Steamworks SDK](https://partner.steamgames.com/) under Valve's terms

## Adapter sub-repos

The C++ libraries zVoxRealms wraps (Jolt, VMA, ImGui, glslang, KTX, FlatBuffers, Tracy, libghostty, GameNetworkingSockets, Steamworks) are not vendored in-tree. Each adapter is a **standalone sub-repo with its own `LICENSE`**, consumed by zVoxRealms via `build.zig.zon` or a git submodule. Existing precedent: [`libs/zig-cpp-vulkan-stack-adapter/`](../libs/zig-cpp-vulkan-stack-adapter/).

**License default for adapter sub-repos: MIT.** Reasons:

- The wrappers are thin glue code — `extern "C"` shims + idiomatic Zig API on top. No algorithmic IP, no novel implementations
- MIT maximizes reuse across the wider Zig ecosystem (vulkan-zig, mach, zig-clap, most zig-gamedev libs are MIT)
- The Apache 2.0 patent grant doesn't earn its complexity for trivial glue with no patentable surface area
- Smaller LICENSE file, fewer terms for downstream readers
- MIT is in Apache 2.0's [Category A](https://www.apache.org/legal/resolved.html#category-a) — fully compatible as inbound to zVoxRealms

**Exception — use Apache 2.0 for adapter sub-repos that wrap patent-prone tech:**

| Adapter | Wrapped library | License | Why Apache 2.0 |
| --- | --- | --- | --- |
| `zig-ktx-adapter` | KTX-Software / Basis Universal | Apache-2.0 | BCn / ASTC codec patents — defensive patent grant matters |
| `zig-steamworks-adapter` | Steamworks SDK | (sub-repo stays private / proprietary) | Steamworks SDK itself is proprietary; the adapter can't be open source anyway |

For everything else (Jolt, VMA, ImGui, glslang, GLFW, FlatBuffers, Tracy, libghostty, GameNetworkingSockets, ENet, miniaudio, cgltf, zstd wrappers): **MIT**.

**Hard rule — don't fragment licenses within a single repo.** When an adapter exists as a standalone repo, license it however you want (MIT or Apache 2.0 per the table above). When code lives inside the zVoxRealms repo, it's Apache 2.0 — period. Don't mix licenses across subdirectories of one repository; it confuses contributors, breaks SBOM tools, and creates maintenance traps.

**Consume as dependencies, not vendored copies.** zVoxRealms pulls adapter sub-repos via `build.zig.zon`. Inside the engine repo, there is no `adapters/<name>/` directory containing copied adapter source code. The integration catalog in [`external-libs-catalog.md`](external-libs-catalog.md) lists each adapter sub-repo's location.

**Adapter C++ style:** every adapter sub-repo follows [`cpp-style.md`](cpp-style.md) (Google C++ Style Guide as baseline + project deviations). Each sub-repo ships its own `.clang-format` mirroring the zVoxRealms root.

**License chain when zVoxRealms ships:**

- Engine code (Apache 2.0) + MIT adapter sub-repos + their underlying permissive libs
- SPDX expression for the combined work: `Apache-2.0 AND MIT` (with BSD-3, Zlib, MIT-0, Apache-2.0 also appearing for specific sub-deps)
- All attribution flows to [`LICENSES.md`](../LICENSES.md) at the repo root and into each shipped game's release archive (via the Phase 13 export pipeline)
- No copyleft anywhere — the engine and every game made with it can ship commercially

## Reference-engine policy — adaptations only, never verbatim ports

zVoxRealms studies four reference engines (Hazel, Luanti, Godot, UnrealEngine — see [`engine-references.md`](engine-references.md)) to understand proven patterns. **None of their source code is copied into this repo, ever.** The rule applies even to permissive-licensed references:

| Engine | License | Why no verbatim copy |
| --- | --- | --- |
| **Unreal** | Epic [Unreal EULA](https://www.unrealengine.com/eula/source) — source-available, not open source | Direct copies violate the EULA. Building a "competing engine" with Unreal source is forbidden. Epic has the legal team to enforce. Triggers DMCA / cease-and-desist / lawsuit risk. |
| **Luanti** | [LGPLv2.1+](https://github.com/luanti-org/luanti/blob/master/LICENSE.txt) — copyleft | Mixing LGPL source into an Apache 2.0 binary forces the entire engine GPL. Kills the licensing strategy. |
| **Godot** | MIT | License-compatible but creates per-file attribution debt + signals poor engineering. Adapt the pattern. |
| **Hazel** | varies (educational origin) | Same as Godot — adapt the pattern. |

**Acceptable workflow:** read reference file → understand the *pattern* (data layout, algorithm shape, API design) → close the file → write a Zig equivalent from your understanding.

**Forbidden:** verbatim copy, line-by-line translation with cosmetic changes, copy-paste-then-modify, including reference-engine headers, vendoring reference-engine source under `vendor/` or `libs/`.

PRs that look like translated reference code are declined regardless of license. See [`engine-references.md` § Legal](engine-references.md) for the full rule and contributor expectations.

## Dependency policy

The engine's third-party dependencies must be:

- **Permissive** — MIT, Apache 2.0, BSD-2/3, zlib, ISC, MIT-0, public domain
- **License-compatible with Apache 2.0 commercial distribution**
- **Listed in [`LICENSES.md`](../LICENSES.md)** with upstream URL, SPDX identifier, and role

Forbidden by policy:

- **GPL / LGPL / AGPL / SSPL / Commons Clause**. These either force the engine to relicense or create commercial-distribution risk
- **Custom "non-commercial use only" licenses** — would block selling games made with the engine
- **License-unknown / unattributed code** — copying snippets from Stack Overflow into the engine without checking the license

When a dependency is dual-licensed (e.g. zstd is BSD-3 or GPL-2), pin the permissive option explicitly in `build.zig.zon` and note the chosen license in `LICENSES.md`.

## Game IP

Games developed with zVoxRealms — including the project owner's commercial titles (Daggerfall clone, voxel Stardew, Atelier-style game, rogue-likes) and any third-party games made with the engine — are **not derivative works of the engine** under Apache 2.0's terms. The Apache 2.0 grants distribution rights for the engine code, not for game-specific code, assets, scripts, or data.

Your shipped game contains:

| Component | License |
| --- | --- |
| `libzvox-runtime.{so,dll}` (the engine code, statically linked from your project's enabled modules) | Apache 2.0 — attribution required in the game's distribution |
| Bundled third-party libs inside the runtime (Jolt, Vulkan, Tracy, etc.) | Their original permissive licenses — listed in `LICENSES.md` |
| `libgame.{so,dll}` (your project's scripts) | **Your choice.** Recommend: proprietary, all rights reserved |
| `game.pck` (your project's assets + data) | **Your choice.** Recommend: proprietary |
| The launcher executable | Apache 2.0 (it's engine code) |
| Project-shipped mods under `mods/` (if any) | **Your choice per mod** — declare in each `mod.toml` |

For your own Steam releases: ship a custom commercial EULA that grants players a license to play your game. The engine's Apache 2.0 license sits underneath, attribution-only.

## Trademarks

The name **"zVoxRealms"**, the logo, and any associated branding are **trademarks of SETA1609**, separate from the Apache 2.0 software license. Apache 2.0 §6 explicitly does not grant trademark rights.

This means:

- A community fork can exist under Apache 2.0 (legal use of the code), but **cannot call itself "zVoxRealms"** (no trademark license)
- Forks must rename — the Firefox/Iceweasel and Redis/Valkey precedent
- This protects the brand on Steam from competing impostor releases while leaving the code open

## Contributions

For a permissive-only Apache 2.0 project with no commercial relicensing, **no CLA is required**. The Apache 2.0 license's `§5 Submission of Contributions` automatically licenses contributions under the same terms ("inbound = outbound"). This is the convention in most permissive Apache projects (Kubernetes, Rust crates ecosystem, most CNCF projects).

Optional future considerations:

- **DCO (Developer Certificate of Origin)** via `Signed-off-by:` git commit trailers can be added later if contribution volume warrants attestation that the contributor has rights to submit
- A formal **CLA (Contributor License Agreement)** would only be needed if the project ever pivots to dual-licensing (sell commercial licenses separately). Not the current plan; revisit if the model changes

Source: <https://www.apache.org/foundation/license-faq.html#CLA-and-CCLA>

## Steam-specific considerations

When shipping the Steam stable build:

1. **Steam Direct just requires distribution rights** — you have them under Apache 2.0
2. **Include the engine's `LICENSE` + filtered `LICENSES.md`** in the release archive (the export pipeline handles this)
3. **Steamworks SDK is proprietary** — only ship in builds with `-Dsteam=true`; never in the open-source binary downloads
4. **Cloud saves use Valve's infrastructure** — Valve's TOS applies separately from the engine license
5. **Workshop content** uploaded by players is under the [Steam Subscriber Agreement](https://store.steampowered.com/subscriber_agreement/), independent of the engine license
6. **Your game's EULA** is what governs the player's right to play — it sits on top of the engine's Apache 2.0

## What this strategy gives up

Honest about the tradeoff:

- **Anyone can fork the engine, polish it, and sell their own competing version on Steam.** Apache 2.0 permits this. The defense is the trademark, the polish of your own product, and the network effect of community contributions flowing back into your repo.
- **No license-fee revenue stream from the engine itself.** Revenue comes from your games + (optionally) paid support contracts for the Steam stable build.
- **No "commercial license required" deterrent** to competitors who'd rather not contribute upstream. They legally don't have to.

These are accepted costs. The alternative (AGPL or source-available) costs more in adoption and community than it gains in commercial-license fees for a solo developer.

## Adding this attribution to source files

For new files (Zig, C, C++), the standard header is:

```text
// Copyright 2026 SETA1609
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
```

For TOML / Markdown / config files, no header needed — the repo-level `LICENSE` covers them.

## Sources

- Apache License 2.0: <https://www.apache.org/licenses/LICENSE-2.0>
- Apache Foundation License FAQ: <https://www.apache.org/foundation/license-faq.html>
- OSI license list: <https://opensource.org/licenses>
- Multi-licensing patterns: <https://en.wikipedia.org/wiki/Multi-licensing>
- Trademark separation in open source: <https://opensource.com/article/17/9/open-source-licensing>
- Steam Direct onboarding: <https://partner.steamgames.com/doc/gettingstarted/onboarding>
