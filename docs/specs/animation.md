# Animation System Spec

> Skeletal animation, state machines, blending, IK. The biggest single gap. Gap: [`gaps.md` § 2.1.A](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.A](../gap-references.md).

## Scope

Animate NPCs, players (visible in third-person mode later — first-person hides the body), creatures, doors, machinery. Voxel-character animation is novel territory; no reference engine has it solved well.

## Components

### Skeleton + skinning

- Skeleton = tree of `Joint` (transform + parent index)
- Per-vertex skin weights (max 4 influences per vertex)
- Pose = array of per-joint transforms
- Skinning math runs on GPU (compute shader or vertex shader)

### Animation clip

- Time-keyed channel data per joint (position, rotation, scale)
- Loop or one-shot
- Imported from glTF (Phase 4 asset pipeline)

### Animation state machine

- Nodes = animation clips or sub-state-machines
- Transitions with conditions (variable comparisons, time-elapsed, event triggers)
- Blend weight per node when in transition
- Inspired by Godot `AnimationTree` ([`gap-references.md` § 2.1.A](../gap-references.md))

### Blend operations

- **1D blend space** — e.g. walk/run blended by speed parameter
- **2D blend space** — e.g. directional walk (forward / strafe vectors)
- **Layer blending** — upper body (attack) over lower body (walk)

### Inverse kinematics

- Foot IK on uneven voxel terrain — critical for character believability
- Two-bone IK (knee, elbow)
- Look-at IK for head/neck tracking targets
- Deferred until Phase 7.5 polish if it blocks the schedule

## Voxel-character animation

Most reference engines animate triangle-mesh skeletons. Voxel characters need their own approach:

**Option A — Skinned voxel mesh.** Treat the voxel character as a mesh with per-vertex skin weights to a skeleton. Reuses standard skeletal animation; the voxel "look" is just blocky geometry. Best for v1.0.

**Option B — Rigged voxel parts.** Each voxel cluster is a child of a joint; transform applied to the cluster as a whole. Cheaper but more limited (no per-voxel deformation).

**Option C — Bone-attached chunks.** Hybrid — main body is skinned (A); detail parts (hat, weapon) are attached to bone tips (B).

v1.0 = Option A; revisit if perf or art demands.

## Animation events

Clips embed event keys (`{ time = 0.45, name = "footstep" }`). Player module subscribes to these via [`specs/events.md`](events.md) — fires footstep SFX, particle puff, etc.

## Borrowed patterns

[`gap-references.md` § 2.1.A](../gap-references.md):

- Godot `scene/animation/animation_tree.cpp` + `animation_blend_space_{1d,2d}.cpp` — the state-machine + blending model
- Godot `scene/3d/skeleton_3d.cpp` — joint hierarchy + pose interpolation
- Skip Unreal AnimGraph's UObject machinery; conceptual borrow only
- No reference for voxel-skinning — original engineering

## Open decisions

- IK solver — built-in two-bone vs FABRIK
- Animation compression (curve-fit reduction, quantization)
- Per-animation instance state vs shared (shared = lots of instances of one walk cycle)
- glTF skinning import path — straightforward via cgltf, but adapt for voxel-mesh skinning

## Milestone

Phase 7.5 (Presentation Layer). NPC walks across uneven terrain with foot IK; transitions from idle → walk → run smoothly via speed parameter; plays attack animation triggered by gameplay event; voxel mesh deforms correctly with skeleton.
