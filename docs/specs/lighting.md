# Scene Lighting, Decals, Weather, Time-of-Day Spec

> Scene-level lighting (sun, point, spot), projected decals, weather state machine, day-night cycle. Gap: [`gaps.md` § 2.1.F](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.F](../gap-references.md).

## Scope

Voxel lighting (per-chunk propagation) is in [`specs/voxel.md`](voxel.md). This spec covers everything above the chunk: directional sun, dynamic lights (torches, fire, spells), decals projected on voxel surfaces, weather, and the day-night cycle that drives them all.

## Lights

| Type | Properties |
| --- | --- |
| **DirectionalLight** | direction (vec3), color, intensity, shadow-casting, shadow-bias. Used for sun + moon. |
| **PointLight** | position, radius, color, intensity, shadow-casting (optional). Torches, fire, spells. |
| **SpotLight** | position, direction, cone angle, range, color, intensity. Flashlights, focused beams. |

Each light is a scene entity (or component on an entity) with a `Handle`. `RenderServer` queries the scene for active lights affecting the current view and submits them to the shader.

### Lighting integration with voxel lighting

Two pipelines merge per-pixel:

- **Baked voxel light** (per-chunk, low-frequency): propagated via [`specs/voxel.md`](voxel.md) lighting; sampled per vertex / per voxel face
- **Dynamic scene light** (high-frequency): directional + point + spot; computed per-pixel in the shader

The two are added in the fragment shader. Voxel light handles distant ambient + sun-baked terrain; dynamic light handles torches and spells that move/flicker.

## Decals

- AABB projector with texture + albedo/normal/orm masks
- Projected onto voxel surfaces (raymarched within the AABB; intersect with voxel grid)
- Examples: gunshot marks, blood splatter, footprints, magic circle glyphs
- Lifetime — eternal (player-placed art), timed (footprints fade), or until-overwritten

### Voxel-decal interaction

Decals "burn into" voxel face textures via a per-chunk decal buffer:

- Each chunk has a small decal-list (max ~32 decals affecting it)
- Fragment shader samples the decal buffer + blends with the underlying voxel texture
- LRU evict if a chunk exceeds the cap

## Time of day

- Single float `time_of_day` in [0.0, 1.0)
- Server-authoritative; replicated to clients (see Luanti pattern in [`gap-references.md`](../gap-references.md))
- Drives sun rotation: `sun_direction = f(time_of_day)`
- Drives sky color: gradient lookup over time
- Drives ambient light intensity
- Game speed parameter — 1 minute real-time = N seconds game-time (per-game configurable)

## Weather

State machine per scene region:

```text
Clear ──> Cloudy ──> LightRain ──> HeavyRain ──> Thunderstorm
   ↑          ↓           ↓             ↓             ↓
   └──────────┴───────────┴─────────────┴─────────────┘
```

Transitions:

- Random per-region (weighted by biome)
- Scripted by quests (always rains in this dungeon area)
- Sync across clients (server pushes weather state)

Each state drives:

- **Particle emitters** ([`specs/particles.md`](particles.md)): rain particles, snow particles
- **Ambient audio** ([`specs/audio.md`](audio.md)): rain ambience bed
- **Lighting**: sun dimmed during heavy weather
- **Visibility**: fog density increases

## Sky

- Skybox (cubemap) for clear weather — different per biome / planet
- Procedural sky tint based on time-of-day (cheap; no atmosphere sim)
- Stars at night
- Optional clouds (volumetric — defer to post-1.0)

## Borrowed patterns

[`gap-references.md` § 2.1.F](../gap-references.md):

- Godot `scene/3d/light_3d.cpp` — light entity model
- Godot `scene/3d/decal.cpp` — decal AABB projector
- Godot `scene/3d/world_environment.cpp` — sky, ambient, fog
- Luanti `server.cpp` + `serverenvironment.cpp` `time_of_day` — multiplayer-safe day-night
- Unreal `SkyAtmosphereComponent.h` — volumetric sky concept (skip implementation; too heavy)

## Open decisions

- Shadow technique — cascaded shadow maps (CSM) vs raytraced shadows (probably CSM for iGPU)
- Max dynamic lights per scene (start: 8 shadow-casters + 64 non-shadow)
- Decal storage format (per-chunk fixed-size array vs page table)
- Weather transition smoothness (instant vs N-second crossfade)

## Milestone

Phase 7.5 (Presentation Layer). Watch the sun rise; sky shifts from black + stars → orange dawn → blue day; ambient light follows. Walk into a cave; voxel-light dominates; light a torch (point light source) — torchlight illuminates nearby walls. Cast a fire spell; gunshot decal appears on the wall where it hits. Weather transitions from clear → rain → thunderstorm with particle + audio + lighting response.
