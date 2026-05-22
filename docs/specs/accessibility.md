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

## Post-v1: TTS Screen Reader (planned polish)

A post-v1 accessibility upgrade — particularly impactful because all four target games are **text-only / mute** (no recorded voice acting). For players using a screen reader, TTS is the *only* path to spoken dialog. Higher value than in voice-acted games.

### Choice — Piper TTS

Use **[Piper](https://github.com/rhasspy/piper)** (Rhasspy). Specifically chosen:

| Constraint | Why Piper fits |
| --- | --- |
| **Apache 2.0** | Compatible with the engine's Apache 2.0 licensing strategy (per [`docs/licensing.md`](../licensing.md)) — unlike eSpeak NG (GPLv3) or commercial TTS |
| **Offline-only** | ONNX-based; no network call after voice model is loaded |
| **No LLM** | Neural TTS, not language model. Small runtime (~5 MB), voice models ~50–200 MB |
| **30+ languages, multi-voice per language** | One runtime, many voice options |
| **CPU-only inference** | Real-time on modern CPUs (~10–50 ms per sentence); doesn't touch the iGPU frame budget |
| **Word-level timings** | Inference returns per-word timings — enables karaoke-style subtitle highlighting |

Reject list (and why):

- **eSpeak NG** — GPLv3 would force the engine GPL. Hard no
- **Festival / Flite** — older, lower quality, English-heavy
- **Cloud TTS** (Google / AWS / Azure / ElevenLabs) — per-call cost + online + privacy concerns
- **Coqui TTS** — MPL 2.0 OK but ~500 MB+ runtime too heavy
- **OS-native screen readers** (Narrator / VoiceOver / Orca) — require UI accessibility tree integration that our TOML UI doesn't expose. Could be added as a "use OS voice" fallback later, but Piper is primary
- **Embedded LLM** — overkill; LLMs are 10-100× larger than Piper voice models

Voice samples + project page: <https://rhasspy.github.io/piper-samples/>

### Distribution — voice packs as free DLC

Voice packs are distributed as **free DLC**, not bundled with the base game and not delivered via Workshop. Reasons:

- Base game stays small (no 50 languages of voice data bundled)
- Players install only the voices they need
- DLC mechanism gives storefronts a native acquisition UX
- Activates the same Steam DLC integration hooks already planned ([`gaps.md`](../gaps.md) #39, [`engine-vs-game.md`](../engine-vs-game.md) § 3b TODO)

Per-vendor support for free DLC:

| Vendor | Free DLC? | Notes |
| --- | --- | --- |
| **Steam** | ✅ | Each voice pack = own free AppID; `ISteamApps::BIsDlcInstalled` detects |
| **GOG** | ✅ | DLC supported, including free |
| **Epic Games Store** | ✅ | Free DLC supported |
| **Microsoft Store** | ✅ | Free DLC supported |
| **itch.io** | ⚠ | No native DLC; ship as separate free downloads in the same project page |
| **Humble Bundle** | ✅ | Tied to Steam keys generally |
| **DRM-free / self-hosted** | n/a | Ship as separate downloads from your GitHub releases or a simple HTTP endpoint |

The engine's voice-pack detection abstracts the platform: on Steam it queries `BIsDlcInstalled`; on itch.io / DRM-free it scans the `voices/` directory; on others it uses the vendor's equivalent ownership check.

### Architecture

```text
Engine (libzvox-runtime, only with -Daccessibility-tts=true at build time):
  └── tts/
        ├── piper runtime (ONNX inference, static lib, ~5 MB)
        └── voice loader / sentence-queue / playback (hooks into audio bus)

Voice DLCs (installed per platform's DLC mechanism):
  <game install>/voices/<locale>/<voice_id>.onnx + .onnx.json     (~50-200 MB each)

In-game UI (Settings → Accessibility → Screen Reader):
  ├── Enabled: off / on
  ├── Voice: dropdown of installed voices, filtered by current locale
  ├── Rate: slider 0.5× – 2.0×
  ├── Volume: own audio bus (per specs/audio.md)
  ├── Read scope: [✓] UI [✓] Dialog [ ] Item descriptions [ ] World narration
  └── Download more voices... → opens platform's DLC store page for available voices
```

### Refinements / polish features

- **Karaoke-style subtitle highlighting**: when TTS speaks a line, the corresponding word in the subtitle is highlighted in real time using Piper's per-word timings. Big readability win for dyslexic players, low-vision players, and language learners.
- **Sound-effect captioning**: extend the subtitle system (`[sword swing]`, `[door creak]`, `[footsteps approaching]`) so TTS reads world-sound cues too. Same authoring pipeline as accessibility subtitles already planned.
- **Per-character voices in dialog**: each NPC can declare a preferred voice profile in their dialog TOML (e.g. `voice = "tts:en-amy-medium"` or `voice = "tts:en-male-deep"`). Same voice across all NPCs is the default; per-NPC override is polish.
- **Reader pacing controls**: pause / skip / repeat-line buttons during dialog (per [`specs/dialog.md`](dialog.md) controller grammar).
- **Curated default voice per language** — rather than dump Piper's full voice list on players, pick one high-quality voice per language as the recommended default; expose the full list under "More voices."

### Out of scope even post-v1

- **Voice cloning** for player-supplied voices (legal + ethical minefield)
- **Real-time translation** (TTS reading translated text on the fly) — interesting but separate feature
- **Whispering / shouting variants** — Piper doesn't expose prosody control at that level for v1
- **Lip sync** to TTS audio — requires animation system integration; defer

### Why this fits post-v1, not v1.0

- 2-3 weeks of integration work (Piper bindings + audio bus integration + voice-pack download UX + per-platform DLC detection)
- DLC distribution pipeline needs Steam DLC integration to already exist (Phase 14 work — see [`gaps.md`](../gaps.md) #39)
- Voice-pack curation (picking + testing the recommended voice per language) takes designer time
- Ship the core game with subtitles + the rest of the v1.0 accessibility baseline first; add TTS as a "Accessibility Update" patch a few months post-launch — gives a fresh Steam-page beat

This deserves a follow-up commitment in [`gaps.md`](../gaps.md) Tier 3 (post-v1 polish) so it stays tracked.

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
