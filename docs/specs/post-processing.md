# Post-Processing Pipeline Spec

> Tonemapping, antialiasing, bloom, vignette, LUT color grading — the final screen-space pass before swapchain present. Gap: [`gaps.md` § 1.4 (post-processing)](../gaps.md). Reference patterns: [`gap-references.md` § 1.4 post-processing](../gap-references.md). Sits in Phase 7.5 (Presentation Layer) alongside [`materials.md`](materials.md) and [`lighting.md`](lighting.md).

## Scope

A lean, iGPU-friendly post-process chain that runs as a single full-screen compute (or fragment) pass at end of frame, reading the HDR scene color target + depth target and writing the final LDR swapchain image.

**Design constraint:** the entire chain must fit a frame budget of **~0.5 ms** on the target low-end iGPU (i3 / Ryzen 3 with integrated graphics, per [`mission.md`](mission.md)). Effects that violate that are out.

## Effects shipped in v1.0

| Effect | Order | Cost target (iGPU) | Why include |
| --- | --- | --- | --- |
| **Bloom** | 1 (reads HDR before tonemap) | ~0.3 ms | Critical for emissive voxels — torches, magic, lava, sun reflections. Without it, emissive looks flat. |
| **ACES tonemap** | 2 | ~0.05 ms | Industry-standard HDR → SDR mapping. Makes voxel colors look "right." Filmic curve preserves highlights better than Reinhard. |
| **LUT color grading** | 3 | ~0.05 ms | Per-scene mood (cold dungeons, warm towns, sickly swamps). 32³ texture lookup; authored externally. |
| **FXAA** | 4 | ~0.1 ms | Cheap antialiasing; voxel edges benefit a lot. Spatial-only (no temporal data needed). |
| **Vignette** | 5 (last) | ~0.02 ms | Subtle edge darkening; cheap cinematic polish; optional per-scene. |

Total: **~0.5 ms** within the budget. Single fused compute pass (one dispatch reads HDR, applies all enabled effects, writes LDR) preferred over five chained passes — saves bandwidth on iGPU.

### Why these and not others

| Effect | Status | Why excluded from v1.0 |
| --- | --- | --- |
| Motion blur | ❌ | Off by default per [`accessibility.md`](accessibility.md) (motion sickness). Not worth the budget for an opt-in effect. |
| Depth of field | ❌ | Expensive on iGPU; niche use case; voxel aesthetic doesn't benefit. Revisit post-v1. |
| SSAO / SSGI | ❌ | Subtle on blocky voxel geometry; cost prohibitive on iGPU. Voxel ambient occlusion already baked per-vertex per [`specs/lighting.md`](lighting.md). |
| Chromatic aberration | ❌ | Overdone in modern games; user-noise. |
| Lens flare | ❌ | Per-scene rare effect; can be added later as an optional module. |
| TAA | ❌ | Requires per-frame jitter + history buffer + motion vectors → significant complexity. FXAA is enough at the target resolution. |

## Data model — per-scene authoring

Post-process settings declared per scene (or inherited from project default). Same pattern as Godot's `WorldEnvironment` resource ([`gap-references.md` § 2.1.F](../gap-references.md) → Godot lighting) — engine reads the config, the renderer applies enabled effects from a list.

```toml
[scene.royal_archive.post_process]
# Bloom
bloom.enabled = true
bloom.threshold = 1.0          # luminance threshold for what bleeds
bloom.intensity = 0.8
bloom.radius = 4               # blur kernel radius

# Tonemap
tonemap.mode = "aces"          # "aces" | "reinhard" | "none"
tonemap.exposure = 1.0
tonemap.white_point = 11.2

# Color grading
lut.enabled = true
lut.texture = "luts/cold_dungeon.ktx2"
lut.strength = 1.0             # 0.0 = bypass, 1.0 = full

# Antialiasing
fxaa.enabled = true
fxaa.quality = "high"          # "low" | "medium" | "high"

# Vignette
vignette.enabled = true
vignette.intensity = 0.35
vignette.smoothness = 0.5
vignette.color = [0, 0, 0]
```

Scene declarations override project-level defaults in `project.toml`:

```toml
[project.post_process_defaults]
tonemap.mode = "aces"
tonemap.exposure = 1.0
fxaa.enabled = true
bloom.enabled = true
bloom.intensity = 0.6
lut.enabled = false
vignette.enabled = false
```

Resolution order: scene > region (if region has its own override block) > project default.

## LUT pipeline

LUTs (Look-Up Tables) are 32³ RGB textures sampled per pixel to remap color.

**Authoring flow:**

1. Artist takes a reference screenshot
2. Opens in any tool (DaVinci Resolve, Photoshop, free Resolve build) with the neutral 32³ LUT strip baked into the bottom of the image
3. Color-grades the screenshot to taste
4. Extracts the modified LUT strip
5. Imports `.png` strip into engine → asset pipeline bakes to `.ktx2` 3D texture

This is the same workflow every modern engine uses — Godot, Unreal, Unity all read the same strip format. Standard size: 1024×32 strip = unrolled 32×32×32 cube.

## Per-scene transitions

When the player moves between scenes with different post-process settings (e.g. wilderness → dungeon), settings cross-fade over **0.5 s** by default. Same tween system as [`specs/ui.md`](ui.md) § Animation / transitions.

Per-axis crossfade — `bloom.intensity` lerps independently from `lut.strength`. LUT texture swap is handled by sampling **both** old + new LUTs during the transition window, weighted by progress, then dropping the old reference.

## Reference patterns

Per [`gap-references.md`](../gap-references.md):

- **Godot `WorldEnvironment`** (`$REFS/godot/scene/3d/world_environment.cpp` + `scene/resources/environment.cpp`) — data-driven config; renderer reads the resource and applies effects from a list. Strong pattern for our per-scene TOML model
- **Unreal `PostProcessVolume`** (`$REFS/UnrealEngine/Engine/Source/Runtime/Engine/Classes/Engine/PostProcessVolume.h`) — spatial per-region post-process with blending. **More than v1.0 needs**; defer spatial-volume blending to v1.x. Note the `Settings` struct pattern: every effect parameter as a typed field with an explicit `bOverride_*` flag

Adaptation rule per [`engine-references.md` § Legal](../engine-references.md): study the data layout + ordering of effects, then implement in Zig. No verbatim ports — Unreal especially.

## Implementation sketch

| Pass | Resource | Output |
| --- | --- | --- |
| 0 (existing scene render) | HDR scene color (RGBA16F) + depth (D32F) | input to post-process |
| 1 (bloom downsample chain) | HDR scene → ½ → ¼ → ⅛ → 1/16 (5 mips) | bloom blur pyramid |
| 2 (fused post-process) | reads HDR + bloom pyramid + LUT 3D + depth | LDR swapchain (BGRA8 / RGB10A2) |

Single fused compute is the goal. Bloom needs its own downsample chain because it's spatial — that's a separate set of dispatches before the main fused pass.

Pipeline-state cache (per [`materials.md`](materials.md)) keys by enabled-effect bitmask — `(bloom | tonemap_aces | lut | fxaa | vignette)` → one specialized pipeline. Avoids `if`s in the shader for disabled effects.

## Hot-reload

Editing post-process TOML in the editor hot-reloads in playtest — same hot-reload bus the materials use. Lets a level designer dial in a dungeon's bloom + LUT live.

## Open decisions

- **Fused single-pass vs chained passes** — depends on shader-compiler quality on the target iGPU. Decide during Phase 7.5 implementation by benchmarking both
- **LUT 3D texture vs 2D strip lookup at runtime** — 3D is fewer ALU ops; 2D is simpler bind setup. Default to 3D
- **Per-region (not just per-scene) post-process** — Unreal's `PostProcessVolume` pattern. Probably wait for v1.x; per-scene is enough for v1.0's four target games
- **Auto-exposure** — a feedback loop where tonemap exposure tracks scene luminance. Useful for dynamic outdoor → indoor transitions but adds GPU complexity. v1.x

## Milestone

Phase 7.5 (Presentation Layer). Walk a torch-lit cave → torch flame visibly **blooms**; emerge into sunlight → ACES tonemap holds highlights without clipping; dungeon LUT swaps to "cold" tint over 0.5 s as you enter the door; FXAA softens voxel edges. All within 0.5 ms on the target iGPU.
