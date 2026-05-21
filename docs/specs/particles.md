# Particle / VFX System Spec

> GPU-driven particles for magic effects, weather, footstep dust, sparks, blood. Gap: [`gaps.md` § 2.1.B](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.B](../gap-references.md).

## Scope

A GPU compute-driven particle system. Emitters spawn particles; particles update on the GPU per-frame; rendered as quads / point sprites / ribbons with material support. Voxel-world interaction (sparks lighting things, particles raycasting against chunks) is the novel piece.

## Architecture

### Emitter

A scene entity with:

- Position + orientation (or attached to a parent transform / handle)
- Spawn rate (particles per second)
- Burst mode (N particles at once on trigger)
- Lifetime (per-particle, with random variation)
- Initial velocity + spread cone
- Material reference

### Particle update (compute shader)

Per-frame compute pass:

- Decrement lifetime; kill expired
- Integrate position + velocity + acceleration (gravity, drag, force fields)
- Color over life (gradient)
- Scale over life (curve)
- Rotation over life

### Particle render

- One indirect draw per emitter (or instanced quads)
- Sorted back-to-front if alpha-blended
- Material from [`specs/materials.md`](materials.md) (shader, blend mode, texture)

## Voxel-world interaction

- **Collision via voxel raycast** — each particle raycasts against the chunk grid; on hit either bounces, sticks, or dies (per-emitter config)
- **Light emission** — emissive particles contribute to nearby voxel lighting via the [voxel lighting](voxel.md) propagation step (deferred — expensive)
- **Snow piling** — long-lived emitter that, on particle death, writes a voxel delta (only in regions with `voxel_type_allowlist` policy permitting it — see [editability policy](../ARCHITECTURE.md))

## TOML emitter authoring

```toml
[emitter.torch_flame]
shape = "cone"
direction = [0, 1, 0]
spread_radians = 0.2
spawn_rate = 80                # per second
lifetime = { min = 0.5, max = 1.2 }
velocity = { initial = 2.5, drag = 0.4 }
acceleration = [0, 1.5, 0]     # upward (buoyancy)
color_over_life = ["#ffeaa0", "#ff8030", "#603020"]   # gradient stops
scale_over_life = [0.5, 1.0, 0.2]
material = "particle_additive"
collide_voxels = false
```

## Performance budget

- Hard cap per scene: 50,000 active particles (configurable)
- Per-emitter cap: 5,000
- Update compute pass: budget 0.5 ms on iGPU
- Cull emitters outside camera frustum (don't simulate offscreen)

## Borrowed patterns

[`gap-references.md` § 2.1.B](../gap-references.md):

- Godot `gpu_particles_3d.cpp` + `particle_process_material.cpp` — emitter + GPU compute pattern
- Godot `servers/rendering/renderer_rd/effects/` — for the compute pipeline shape
- Niagara (Unreal) — module/emitter/system layering is gold-standard; reimplement at 5% complexity

## Open decisions

- GPU vs CPU emit (GPU spawn requires indirect draw; CPU spawn is simpler)
- Particle data layout — SoA vs AoS (SoA wins for GPU)
- Mesh particles (ribbons, trails) vs quad-only — defer trails to post-1.0
- LOD: simulate fewer particles when distant

## Milestone

Phase 7.5 (Presentation Layer). Cast a fireball spell → emitter spawns at hit point → flame + smoke particles for 1.5s → sparks bounce off nearby voxels. Walk in rain → 5000 rain particles tracking the player; reasonable perf on iGPU.
