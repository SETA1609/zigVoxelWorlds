# Accessibility Spec

> The v1.0 accessibility baseline — colorblind modes, key/controller rebinding, font scaling, subtitle controls, motion-sickness reduction, high-contrast UI. Gap: [`gaps.md` § 2.2.J](../gaps.md). Reference patterns: [`gap-references.md` § 2.2.J](../gap-references.md).

## Scope

Accessibility is **table stakes** for v1.0 — required for a credible Steam release in 2026. Not an afterthought. Baked into the UI engine ([`specs/ui.md`](ui.md)) and gameplay defaults from day one.

## v1.0 accessibility baseline

| Feature | Default | Settings UI |
| --- | --- | --- |
| **Colorblind modes** | Off | Toggle for protanopia / deuteranopia / tritanopia. UI palette shifts via SCSS theme switch. Combat-critical color cues (enemy health bars) use shape + color, not color-only |
| **Key rebinding** | All bindable | Settings → Controls → list of actions; click an action; press new key. Including modifiers. Save per-profile |
| **Controller rebinding** | All bindable | Same as key rebinding, for gamepad |
| **Font scaling** | 100% | Settings → Display → Font Size slider 75%–200%. UI engine respects this; reflows widgets |
| **Subtitle controls** | On + medium | Settings → Audio → Subtitle on/off, size (small/med/large/xl), background opacity, speaker name color |
| **Field-of-view slider** | 75° | First-person games — slider 60°–110° in Settings → Display. Critical for motion-sickness reduction |
| **Head-bob** | Off | Off by default (accessibility-first). Toggleable in Settings → Display |
| **Motion blur** | Off | Off by default. Toggle in Settings → Display |
| **Camera shake** | On (mild) | Toggle + intensity slider |
| **High-contrast UI mode** | Off | Settings → Display → High Contrast toggle. SCSS theme switches to high-contrast palette + thicker borders + bolder text |
| **Reduced flashing** | Off | Reduces particle/light flicker — important for photosensitive epilepsy |
| **Persistent UI hints** | Off | Show button prompts always-on instead of fading after first use |

## Per-system implementations

### UI engine ([`specs/ui.md`](ui.md))

- Theme switching: `accessibility.colorblind_mode` and `accessibility.high_contrast` drive which SCSS theme compiles
- Font scaling: all `font-size` values multiplied by user's font-size setting
- Focus visualization: always-visible focus indicator (not just on controller)

### Camera ([`specs/camera.md`](camera.md))

- FoV slider feeds `Camera.fov_degrees`
- Head-bob toggle gates the view-bob animation
- Motion-blur toggle gates the post-process pass

### Audio ([`specs/audio.md`](audio.md))

- Subtitle stream is independent of voice playback — even if voice is muted, subtitles can display
- Per-bus volume controls (already in audio spec) — including a Dialog bus separate from SFX
- Reduced-flash toggle dampens audio cues for events flagged as "flashy" (lightning, explosions)

### Visuals (lighting + particles)

- Reduced-flashing toggle: lightning strikes dimmed; high-intensity particle flashes throttled; rapidly strobing lights cap their max delta

### Input

- Modifier key support (Shift / Ctrl / Alt) in rebinds — players with limited mobility may need alternative chord combinations
- "Hold" vs "Toggle" for actions (some players can't hold buttons)
- Repeat-rate adjustment for menu navigation

## Subtitle system specifics

- Speaker name + dialog text on separate visual lines
- Background panel (configurable opacity) to ensure readability over bright scenes
- Sound-effect subtitles (configurable: off / important-only / all) — "[door creaks]", "[footsteps approaching]"

## Borrowed patterns

[`gap-references.md` § 2.2.J](../gap-references.md):

- Godot has limited engine-level accessibility; the relevant code is the focus-graph in `scene/gui/control.cpp`
- Unreal: limited engine-level support; mostly game-side
- The real "reference" is industry-standard accessibility frameworks: [Game Accessibility Guidelines](https://gameaccessibilityguidelines.com/) (CVAA-compliant baseline)

## What's not in v1.0 (deferred)

- **Screen reader integration** — meaningful only for menu-only games; combat/exploration screen-reader is hard. Targeted research for v1.1+
- **Voice control** — defer
- **Eye tracking** — defer
- **Single-button play modes** — game-design-specific; per-game decision
- **Cognitive accessibility** (simplified text, slower pacing) — partially addressed by difficulty settings; full treatment deferred

## Marketing value

A meaningful accessibility baseline isn't just ethical — it's a Steam-page selling point. The "accessibility features" tag pulls in players who actively seek inclusive games and benefits SEO. List the baseline features in the Steam description.

## Open decisions

- Default settings: opt-in or opt-out for sensitive features? (Following industry guidance: motion-blur off, head-bob off, photosensitive-safe on by default — features that exclude players are opt-in, not opt-out)
- Per-save vs global accessibility settings (global; players want consistency)
- Color-blind simulation tooling for the editor (so devs can test their UI)

## Milestone

Phase 12 (editor settings UI) + Phase 13 (exported game). Run the game with protanopia + 150% font + high contrast + subtitles + 90° FoV + head-bob off. All gameplay-critical info still readable + recognizable. Independent audit (using e.g. [Can I Play That?](https://caniplaythat.com/) review framework) passes the baseline checklist.
