# Camera System Spec

> First-person primary across all four target games; optional third-person added later. Gap: [`gaps.md` § 2.2.K + § 3 #14](../gaps.md). Reference patterns: [`gap-references.md` § 2.2.K](../gap-references.md).

## Scope

**All four target games (Daggerfall-clone / voxel Stardew / Atelier / rogue-like) are first-person primary.** Third-person (over-the-shoulder or orbit) is a v1.x feature added later. This scope decision dramatically simplifies the camera system relative to a multi-mode engine.

This was previously assumed differently (different camera per game). The corrected design:

- v1.0: first-person only
- v1.x: first-person + optional third-person via toggle (TAB / F4)
- Top-down / RTS / orthographic: not in scope

## v1.0 — first-person camera

Single camera entity attached to the player's head bone:

- Position = player head joint position (from skeleton, if rigged) or `player.position + head_offset`
- Orientation = controlled by mouse delta (yaw + pitch); roll = 0 (no Quake-style roll for normal play)
- Pitch clamped to ±89° (prevents gimbal flip)
- FoV configurable (default 75°, slider 60°–110° in settings — important for motion-sickness reduction, see [`specs/accessibility.md`](accessibility.md))
- View bobbing on walk (toggleable; off-by-default for accessibility)

## Camera entity model

```zig
const Camera = struct {
    handle: Handle,
    mode: enum { first_person, third_person_orbit },
    attached_to: Handle,           // player or NPC entity
    yaw: f32,
    pitch: f32,
    fov_degrees: f32,
    head_offset: Vec3,             // first-person eye-position relative to entity
    
    // third-person fields (v1.x)
    distance: f32,                 // distance from target
    target_offset: Vec3,           // shoulder offset
};
```

Single camera struct supports both modes; mode switch is a flag, not a class hierarchy.

## Input handling

- Mouse delta → yaw/pitch (sensitivity configurable)
- Right-stick (gamepad) → yaw/pitch (sensitivity configurable; aim-assist deferred)
- Scroll wheel → FoV adjust (rebindable; off by default)
- Modifier key + mouse → temporary look-around without affecting player heading (rare; for menus/inspection)

## Frustum culling

- Camera computes view frustum each frame
- Voxel chunk culling: cull chunks fully outside frustum (per-chunk AABB test)
- Entity culling: cull entities whose AABB is outside frustum
- Frustum data exposed to `RenderServer` for cull-aware draw submission

## Third-person mode (v1.x)

Activate via toggle key:

- Camera position = `target_position + target_offset + (-camera_forward * distance)`
- Spring constraint — distance shrinks if voxels block the line-of-sight (no clipping into walls)
- Right-stick orbits around target instead of rotating in-place
- Player visible (skeletal-animated voxel mesh — see [`specs/animation.md`](animation.md))
- HUD adjustments (crosshair may move to over-the-shoulder offset)

## Borrowed patterns

[`gap-references.md` § 2.2.K](../gap-references.md):

- Godot `scene/3d/camera_3d.cpp` — projection setup, frustum extraction
- Unreal `CameraComponent.h` — component-on-actor pattern (for the attach-to-entity model)

## What this simplifies

The first-person-primary decision removes complexity from many other systems:

- **UI** ([`specs/ui.md`](ui.md)): HUD layout assumes first-person screen-anchor positions
- **Input**: mouse-look is the primary; touch (Android) needs virtual stick — but that's v1.x
- **Animation** ([`specs/animation.md`](animation.md)): player body animation is hidden in first-person; just hand/weapon animation matters until third-person ships
- **Audio** ([`specs/audio.md`](audio.md)): HRTF / 3D spatialization tracks camera (which = player head)
- **Networking** ([`specs/multiplayer.md`](multiplayer.md)): only player position + look-direction needs syncing; no third-person rig state

## Open decisions

- Head-bob — opt-in or opt-out default? (Accessibility says opt-out — disable by default)
- Mouse acceleration — off by default (FPS convention)
- Smoothing on look — none by default (raw input feels responsive)
- Per-game default FoV — same default across all four games; user override per save

## Milestone

Phase 1 (Vulkan basics) + Phase 12 (editor for camera controls) + Phase 8 (gameplay tied to camera). Player walks through a generated world in first-person; FoV slider in pause menu works; v1.x adds the third-person toggle.
