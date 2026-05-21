# Audio Architecture Spec

> Spatial audio + bus mixing + music streaming on top of miniaudio. Gap: [`gaps.md` § 2.1.C](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.C](../gap-references.md).

## Scope

[`tech-stack.md`](../tech-stack.md) picks miniaudio as the platform / decode / mix backend. This spec is the engine layer on top: bus tree, 3D spatializer, reverb zones, music streaming.

## Bus tree

A static tree of named buses; per-bus volume + effect chain. Game code routes sounds to bus names; players can adjust per-bus volume in settings.

```text
Master
├── SFX
│   ├── Player        (footsteps, attacks, voice)
│   ├── World         (ambient, voxel-edit sounds)
│   ├── Enemies       (creature SFX)
│   └── UI            (button clicks, menu transitions)
├── Music
│   └── (single bus; crossfaded between tracks)
├── Dialog
│   └── (separate bus so subtitles + voice volume control independently)
└── Ambient
    └── (looping environment beds — wind, rain, cave drips)
```

## 3D positional audio (first-person-primary games)

- All four target games are **first-person primary** (per project decision; see [memory](../README.md) + `vision.md` §"How players experience it"). HRTF spatialization is high-value because the listener orientation is the camera.
- Source position + listener (camera) position + listener forward + up → per-channel gain
- Attenuation curve per source (linear / exponential / log; per-sound configurable)
- Doppler off by default; enable per-source for moving things (projectiles)

## Reverb zones

Defined as AABBs in `scene.toml`:

```toml
[[scene.reverb_zones]]
aabb = [[0, 0, 0], [50, 20, 50]]
preset = "cave"        # references a reverb preset
```

Presets: `cave`, `hall`, `forest`, `interior_small`, `interior_large`, `underwater`, etc.

When the listener enters a zone, the master bus gets the zone's reverb effect; smooth crossfade on transition.

## Occlusion

For first-person games, occlusion is high-impact. Implementation:

- Raycast from listener to source through voxel chunks
- If hit → low-pass filter + attenuate the source
- Cheaper alternative: pre-baked "audio occlusion" per chunk (defer to optimization phase)

## Music streaming

- One music bus, two slots (current + next) for crossfade
- Tracks declared in TOML with loop points (`intro_end`, `loop_start`, `loop_end`)
- Triggered by scene events (`scene.entered`, `combat.started`, etc. — see [`specs/events.md`](events.md))
- Streaming from disk via miniaudio's decoder; small RAM footprint

## Voice / dialog

- Dialog system separate (see Phase 8 gameplay)
- Plays on Dialog bus
- Subtitle sync: dialog file ships with timing TOML (`[[lines]] start=0.0 end=2.3 text="..."`)
- Optional TTS fallback for unvoiced lines (system TTS on each platform; deferred)

## Borrowed patterns

[`gap-references.md` § 2.1.C](../gap-references.md):

- Godot `AudioServer` + `AudioStreamPlayer3D` for the bus + spatial source pattern
- Godot's reverb / effect chain implementation as reference for bus-effect insertion
- Skip Unreal Audio Mixer's complexity; the simple version is enough

## Open decisions

- HRTF library (miniaudio has basic; consider Steam Audio / IPL later)
- Voice/dialog file format (Opus most likely; per-game decision)
- Max simultaneous voices (defaults to 64; adjustable per platform)

## Milestone

Phase 7.5 (Presentation Layer). Walk near a torch; hear it crackle louder; turn away; volume drops based on head orientation. Walk into a cave; reverb engages. Switch BGM track on scene transition with crossfade. All controllable via in-game settings menu (per-bus sliders).
