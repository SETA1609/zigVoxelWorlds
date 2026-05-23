# In-Game UI Engine Spec

> The runtime UI engine — HUDs, menus, dialog, inventory. ImGui handles editor UI; this spec is for shipped-game UI. Gap: [`gaps.md` § 2.1.D](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.D](../gap-references.md).

## Scope

A declarative UI engine driven by TOML layout + SCSS styling (per [`tech-stack.md`](../tech-stack.md) § UI). Renders through `RenderServer` like everything else. Editor-time editing via a dedicated UI panel in `src/editor/panels/ui_designer/` (Phase 12).

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

Custom widgets register via the C ABI for mods.

## TOML schema (example)

```toml
[hud.health_bar]
type = "ProgressBar"
anchor = [0.0, 0.0]
offset = [16, 16]
size = [200, 24]
style_class = "hud-bar health"
value_bind = "player.health_pct"

[menu.start_button]
type = "Button"
anchor = [0.5, 0.5]
text = "tr:ui.menu.start"   # localized string
on_click = "menu.start_game"
focus_neighbors = { up = "load_button", down = "settings_button" }
```

## SCSS styling

Standard SCSS subset, compiled to a flat style-rule table at import time:

```scss
.hud-bar {
  background: rgba(0, 0, 0, 0.5);
  border-radius: 4px;
  &.health { fill: $color-health; }
  &.mana   { fill: $color-mana; }
}

.hud-bar:hover { border: 2px solid #fff; }
```

Pseudo-states: `:hover`, `:pressed`, `:focused`, `:disabled`.

## Focus graph (controller nav)

Every focusable widget declares its `focus_neighbors = { up, down, left, right }`. Stick / D-pad moves focus along these edges. Critical for the four target games — Stardew + Atelier are very controller-driven.

## Font subsystem

- FreeType (via adapter sub-repo) for glyph rasterization
- Atlas-based — text rendered as quads with UV into a generated atlas
- Per-locale fallback chain (English Latin → CJK glyph fallback for Chinese/Japanese chars)
- DPI scaling: design-time pixels, runtime scaled by display DPI + user font-size setting

## Borrowed patterns

[`gap-references.md` § 2.1.D](../gap-references.md):

- Godot's `Control` anchor + offset system — directly applicable
- Godot's per-widget files (`button.cpp`, `label.cpp`) — read for widget behavior contracts
- Luanti's formspec DSL — TOML schema philosophy: declarative, parseable, hot-reloadable
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

## Open decisions

- Renderer integration: in-game UI as its own render pass, or composited with scene?
- Localization-aware text wrapping (CJK, RTL — see [`specs/localization.md`](localization.md))
- Touch support (Android) — defer until Android port phase
- Theme switching at runtime (light/dark/high-contrast for accessibility — see [`specs/accessibility.md`](accessibility.md))

## Milestone

Phase 7.5 (Presentation Layer) + Phase 12 (editor designer panel). Build a main-menu screen + an in-game HUD with health bar, hot-bar, and minimap. Controller-navigable. Localized to two languages.
