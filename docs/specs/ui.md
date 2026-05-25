# In-Game UI Engine Spec

> The runtime UI engine — HUDs, menus, dialog, inventory. ImGui handles editor UI; this spec is for shipped-game UI. Gap: [`gaps.md` § 2.1.D](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.D](../gap-references.md).

## Scope

A declarative UI engine for shipped games — HUDs, menus, dialog, inventory. Renders through `RenderServer` like everything else. Editor-time editing via a dedicated UI panel in `src/editor/panels/ui_designer/` (Phase 12).

## v1 implementation: RmlUi behind the UI-stack adapter

**Decision (Phase 0):** v1 ships [RmlUi](https://github.com/mikke89/RmlUi) (MIT) as the runtime UI engine. The wrapper that hides RmlUi specifics lives in `libs/zig-cpp-ui-stack-adapter/` (planned submodule), **not** in `src/ui/`. This is the same decoupling pattern the project uses for every other C++ dependency (Vulkan, Platform, …) — see [`external-libs-catalog.md` § 3](../external-libs-catalog.md).

### Two-layer split

```text
┌───────────────────────────────────────────────────────────────┐
│ src/ui/                                                       │
│ ───────                                                       │
│ Thin Zig API the engine + gameplay code calls.                │
│ Just typed bindings to the adapter's C ABI + engine glue      │
│ (i18n mustache, asset-DB hooks, gamepad input routing).       │
│ Knows the C ABI. Does NOT know about RmlUi.                   │
└─────────────────────┬─────────────────────────────────────────┘
                      │ stable C ABI: zvui_document_load, …
                      ▼
┌───────────────────────────────────────────────────────────────┐
│ libs/zig-cpp-ui-stack-adapter/                                │
│ ──────────────────────────────                                │
│ The actual wrapper.                                           │
│   - RmlUi vendored as the backend lib                         │
│   - FreeType bundled (RmlUi's glyph backend)                  │
│   - C++ binding layer translating RmlUi types ↔ C ABI         │
│   - The only code that #includes <RmlUi/...>                  │
│ Distributed as its own repo, owns its build.zig + LICENSE.    │
└───────────────────────────────────────────────────────────────┘
```

### Why this split

- **Swapping to a future native impl is a libs adapter change, not an engine change.** Replace `libs/zig-cpp-ui-stack-adapter/`'s implementation with native code (or another lib like Yoga, RmlUi successor, etc.); the C ABI stays; `src/ui/` doesn't recompile.
- **No `<RmlUi/*>` headers anywhere in `zigVoxelWorlds/`** — they live exclusively inside the adapter sub-repo. The engine tree never sees them. Same discipline as Vulkan + Jolt.
- **The adapter's `LICENSE` is its own** — RmlUi MIT obligations are tracked in `libs/zig-cpp-ui-stack-adapter/LICENSE`, not at the engine root. Standard per [`external-libs-catalog.md` § 3](../external-libs-catalog.md).

### Why RmlUi over custom TOML+SCSS

- Style cascade + transitions + animations + flex layout = ~5–7k LoC of engine code that RmlUi already ships solid.
- Mod authors get HTML/RCSS (a well-known mental model) instead of a bespoke TOML/SCSS dialect with no IDE support.
- Off-ramp stays open: replace the adapter's contents, not the engine's.

**Out of scope for v1:** writing a custom TOML+SCSS parser, custom layout engine, custom style resolver. Re-evaluate at end of Phase 12 — see [§ Off-ramp](#off-ramp-to-a-native-implementation).

The sections below describe the **engine-visible UI contract** — anchor layout, widget set, focus graph, quickbar behavior. They're written in implementation-neutral terms so the contract survives a future adapter swap.

## Layout model

Anchor-based (Godot Control style), not constraint-based (Auto Layout style):

- Each widget has an anchor (parent-relative `[0.0, 1.0]` per axis) + offset (pixels from anchor)
- Anchor `(0.5, 0.5)` with `(0, 0)` offset = centered on parent
- Anchor `(1.0, 0.0)` = top-right corner
- Min/preferred/max size for flexible layouts
- Containers (HBox, VBox, Grid) distribute children automatically

## Widget set (v1.0)

| Widget | Purpose |
| --- | --- |
| `Label` | text display |
| `Button` | clickable, with hover/pressed/disabled states |
| `Panel` | background container |
| `Image` | texture display |
| `List` | scrollable item list |
| `Scroll` | scrollable container |
| `Slider` | value selector |
| `TextEdit` | single-line text input |
| `TextArea` | multi-line text input |
| `Dropdown` | selection from list |
| `Tabs` | tabbed container |
| `Modal` | overlay dialog |
| `ProgressBar` | filled portion of width |
| `Tooltip` | hover-shown info |

Custom widgets register via the C ABI for mods. Internally these map to RmlUi `Core::ElementInstancer` factories registered through the wrapper.

## RML markup (example)

Source file: `<project>/assets/ui/hud.rml`. Loaded via `Ui.Document.load("hud.rml")` from the Zig wrapper.

```rml
<rml>
  <head>
    <link type="text/rcss" href="hud.rcss"/>
  </head>
  <body id="hud">
    <progressbar id="health_bar"
                 class="hud-bar health"
                 data-attr-value="player.health_pct"/>

    <div id="start_menu">
      <button id="start_button"
              focus-up="load_button" focus-down="settings_button"
              onclick="menu.start_game()">
        {{tr:ui.menu.start}}
      </button>
    </div>
  </body>
</rml>
```

**i18n note.** `{{tr:ui.menu.start}}` is the RML call-site syntax — the markup equivalent of `t("ui.menu.start")` in Zig code (see [`localization.md`](localization.md) § Code idiom). The UI-stack adapter intercepts the mustache during document load, looks the key up in the compiled gettext catalog, and substitutes the localized string before RmlUi sees the element text. **The translation format is gettext PO**, not i18next or i18next-style JSON — see [`localization.md`](localization.md), which explicitly rejects a TOML middle layer.

The `{{tr:…}}` syntax exists only so RML authors don't have to wire data-bindings for every translatable label; it does not introduce a parallel i18n system. RmlUi's own `data-bind`/`data-attr-*` is exposed through the wrapper's `Element.bindAttr()` for runtime-changing values (health %, score, etc.).

## RCSS styling

Standard RCSS — CSS 2.1 plus most of CSS3, parsed by RmlUi. Source file: `<project>/assets/ui/hud.rcss`.

```rcss
.hud-bar {
    background-color: rgba(0, 0, 0, 0.5);
    border-radius: 4px;
    width: 200px; height: 24px;
}
.hud-bar.health { image-color: #c44; }
.hud-bar.mana   { image-color: #44c; }

.hud-bar:hover { border: 2px #fff; }

button {
    transition: background-color 120ms;
}
button:focused { background-color: rgba(255, 255, 255, 0.15); }
```

Pseudo-states supported: `:hover`, `:active` (= "pressed"), `:focus`, `:checked`, `:disabled`, `:nth-child(...)`. Transitions + keyframe animations are RCSS-native — no engine work.

**RCSS subset policy:** the wrapper does not restrict RCSS syntax — anything RmlUi parses, modders can write. The engine does not promise the full set survives a future native swap; mod authors targeting that future should stick to the subset documented in `docs/specs/ui-rcss-subset.md` (TBD before Phase 12 ships).

## Focus graph (controller nav)

Every focusable widget declares its focus neighbors via the `focus-up` / `focus-down` / `focus-left` / `focus-right` attributes (RmlUi's `tab-index` + the wrapper's added attributes). Stick / D-pad moves focus along these edges. Critical for the four target games — Stardew + Atelier are very controller-driven.

The wrapper intercepts gamepad input from `platform/` and dispatches it as RmlUi focus events; modders see a stable Zig API rather than RmlUi's raw event system.

## Font subsystem

RmlUi's bundled FreeType backend handles glyph rasterization + atlasing. The wrapper:

- Configures the per-locale fallback chain (English Latin → CJK fallback for Chinese/Japanese)
- Wires DPI scaling — design-time pixels in RCSS, runtime scaled by display DPI + user font-size setting
- Loads font assets through the engine's asset DB (`<project>/assets/.assetdb.toml` → GUID), not RmlUi's filesystem lookup, so fonts respect the mod-loader layered FS

No HarfBuzz in v1 (RmlUi's FreeType backend is enough for Latin + CJK without complex shaping). Add later if Arabic/Devanagari/etc. become first-class.

## Borrowed patterns

[`gap-references.md` § 2.1.D](../gap-references.md):

- **RmlUi** as the engine — production-tested HTML/CSS-subset UI runtime (Bohemia, Frontier shipped titles)
- Godot's `Control` anchor + offset model — *concept already implemented inside RmlUi* via `position` + `top/left/right/bottom`; we adopt the model by using RmlUi
- Godot's per-widget files (`button.cpp`, `label.cpp`) — read only for *behavior contracts* we expose in our wrapper (focus-neighbor semantics, modal stacking, list-of-tooltips rules)
- Luanti's formspec DSL — informed the *moddable-data* philosophy; RML files in the mod-loader layered FS deliver the same property
- Skip Slate (Unreal) — too heavy

## Quickbar — shared across all four target games

Every target game uses a quickbar (also called hotbar): 10 slots, indexed 0–9, holding references to favorited inventory items for fast use. Daggerfall has 0–9 keys for spells/items; Stardew has the iconic bottom hotbar; Atelier has quick-item assignment; rogue-likes use it for combat actions. **Same widget, same controls, same data model — universal across all four.**

### Layout

- Anchored bottom-center of screen (anchor `(0.5, 1.0)`, offset `(0, -16)`)
- Horizontal strip of 10 slot widgets
- Each slot: icon + stack count + binding number ("1" through "0")
- Active slot: visible ring + slight scale-up + filled background
- Slot binds to an inventory item via reference (not copy) — see [`specs/gameplay.md`](gameplay.md) § Inventory

### Visibility modes

- **Visible** (default for Stardew, rogue-like, Atelier — recommended for new players in any game)
- **Hidden** (default for Daggerfall clone — preserves immersion in the FP RPG aesthetic)
- Toggle in settings → Display → "Show quickbar"; per-save preference

### Controls

| Input | Action |
| --- | --- |
| `1`–`0` keys | Activate slot 1–10 |
| Mouse wheel up/down (when visible + cursor over quickbar OR `mouse_wheel_quickbar` toggle on) | Cycle slots |
| Gamepad **D-pad left / right** (digital, not analog) | Cycle slots |
| Drag inventory item → slot | Bind slot |
| Right-click slot (or `X` on controller) | Clear binding |

### Behavior when hidden — no transient overlay

When the quickbar is set to hidden, cycling with D-pad or mouse-wheel **does not pop a transient on-screen overlay**. The natural feedback is **the held-item visual changing in the world**: a sword appears in the player's hand, then a bow replaces it. That world-visible swap *is* the confirmation that cycling worked. Adding a transient UI overlay would be redundant noise.

This matches the FP RPG aesthetic — the player learns to interpret what they're holding by looking at their hand, not at a HUD element.

If a slot is empty when cycled to: the player's hand becomes empty. Still no overlay.

### Slot binding rules

- Slots store a **stable item reference** (per-stack identity, not a per-item handle that breaks when count changes)
- If the bound item is fully consumed/removed → slot empties (no error, no broadcast — silent)
- If the bound item is dropped → slot empties
- If the bound item is moved to a different inventory container → slot follows (still bound, by reference)
- Per-save state (persists in save format per [`specs/save-ux.md`](save-ux.md))

### Activation semantics

Activating a slot equals "select this item to use." The actual *effect* of using the item is the item's behavior:

- Weapon → equipped, ready to attack
- Potion → consumed (after a brief "drink" animation)
- Tool → wielded for interaction
- Spell → readied for casting
- Food → eaten

Activation fires `event quickbar.activated { slot, item_id }` (per [`specs/events.md`](events.md)) — game code (or mods) hook this to drive the specific effect.

### Quickbar open decisions

- Multiple quickbar pages (Stardew has 12 visible + alt-paging vs Daggerfall's flat 10) — start with flat 10; revisit if a target game needs paging
- Mouse-wheel default: bind to quickbar cycle vs camera-zoom — start with off; opt-in
- Drag-binding from chest UI directly (vs requiring item to be in player inventory first) — start with player-inventory-only; revisit

## Animation / transitions

Resolves [`gaps.md` § 1.12 front-end transitions](../gaps.md) — scene fade-to-black, modal slide-in, menu cross-fade. Folded into the UI engine rather than its own spec because the surface is small and the implementation reuses the widget tree.

### Per-property tween

Animate any widget property (position, scale, opacity, color, anchor) over time:

```toml
[tween.menu_fade_in]
target = "menu.root"
property = "modulate.a"      # alpha channel of the modulate color
from = 0.0
to = 1.0
duration = 0.3               # seconds
easing = "ease_out_cubic"
```

### Easing curves (v1.0 set)

`linear` · `ease_in_quad` · `ease_out_quad` · `ease_in_out_quad` · `ease_in_cubic` · `ease_out_cubic` · `ease_in_out_cubic` · `ease_out_back` (subtle overshoot for snap-in) · `spring` (deferred to Phase 7.5 polish — needs mass/stiffness/damping params).

### Scene transitions

Built on the per-property tween:

| Transition | Implementation |
| --- | --- |
| **Fade-to-black** | Full-screen black `Panel` widget; opacity tween 0 → 1; load new scene under cover; opacity tween 1 → 0 |
| **Cross-fade** | Two scene root nodes alive simultaneously; old `modulate.a` 1 → 0 while new 0 → 1 |
| **Slide** (for in-game scene change without world break) | Translate offset tween on the world camera target while a hold-frame screenshot is composited on top |

Scene transitions are orchestrated by a `SceneTransitionManager` (one instance per running game) that owns the in-flight tween + the load coroutine.

### Modal transitions

Dialog / trade / inventory open + close per [`dialog.md`](dialog.md). Default style: **slide-up from bottom** with a 0.2 s `ease_out_cubic`. Configurable per-modal in TOML (`enter_anim = "fade"` / `"slide_left"` / `"slide_up"` / etc.).

### Menu transitions

Main menu → settings → back. Cross-fade between menu screens at 0.15 s. Focus restores to the originating button on back.

### Triggers

- State-change driven: `.hover`, `.focused`, `.pressed`, `.disabled` automatically tween declared style deltas
- Scripted: game code or BT action fires `ui.tween_start { tween_id }` per [`events.md`](events.md)

### Cost budget

UI tweens are CPU-cheap (single-property lerps); total UI tween cost target < 0.05 ms at 60 FPS even with 50 active tweens. No GPU dispatch — UI rendering reads the tweened values directly.

### Reference patterns

| Engine | Where | What to adapt |
| --- | --- | --- |
| **Godot — Tween** ✅ best fit | `$REFS/godot/scene/animation/tween.cpp` | Per-property tween model, easing-curve enum, the `interpolate_property` API surface |
| **Unreal — UMG animations** | `$REFS/UnrealEngine/Engine/Source/Runtime/UMG/Public/Animation/` | Track-based animation per widget. Heavier than needed; study the data model only |
| Stardew Valley (game) | n/a — game-side | Fade-to-black between locations is the canonical "indie-correct" simplicity |

Adaptation rule per [`engine-references.md` § Legal](../engine-references.md): study Godot's Tween API surface, close the source, implement in Zig.

## Off-ramp to a native implementation

The wrapper is the contract; RmlUi is an implementation detail. A future native impl (`backends/ui/native/`) could replace RmlUi if:

- RmlUi upstream stalls or relicenses
- Shipped-binary size or per-frame cost becomes a problem on the target hardware (i3 + iGPU)
- The C ABI boundary at the wrapper proves too narrow for an ambition we want to add (custom shaders per element, GPU-driven layout, voxel-aware widget composition, …)

**Evaluation gate:** end of Phase 12. If RmlUi is meeting needs, leave it. If two or more of the triggers above are firing, schedule a native impl as a Phase-13+ R&D track.

**Discipline to keep the off-ramp viable:**

- Engine and gameplay code call only the Zig wrapper, never RmlUi types directly
- The wrapper does not leak `Rml::Element*` or `Rml::Context*` across its API
- RML/RCSS files live under `<project>/assets/ui/` and are loaded via the wrapper's `Document.load(guid)` — never via RmlUi's filesystem
- Mod-author docs reference the wrapper Zig API, not RmlUi types

If those four rules hold, swapping the backend is a wrapper-implementation change, not a content-author break.

## Open decisions

- Renderer integration: in-game UI as its own render pass, or composited with scene? (RmlUi exposes a `RenderInterface` we implement against `render_server`.)
- Localization-aware text wrapping (CJK, RTL — see [`specs/localization.md`](localization.md))
- Touch support (Android) — defer until Android port phase
- Theme switching at runtime (light/dark/high-contrast for accessibility — see [`specs/accessibility.md`](accessibility.md))
- Mod-author RCSS subset documentation — produce `docs/specs/ui-rcss-subset.md` before Phase 12 ships

## Milestone

Phase 7.5 (Presentation Layer) + Phase 12 (editor designer panel). Build a main-menu screen + an in-game HUD with health bar, hot-bar, and minimap. Controller-navigable. Localized to two languages.
