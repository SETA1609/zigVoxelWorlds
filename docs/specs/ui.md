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

## Animation / transitions

- Per-property tweens (position, scale, opacity, color) with duration + easing
- Triggered by state changes (`.hover`, `.focused`) or scripted
- Spring physics for "bouncy" feedback (deferred to Phase 7.5 polish)

## Open decisions

- Renderer integration: in-game UI as its own render pass, or composited with scene?
- Localization-aware text wrapping (CJK, RTL — see [`specs/localization.md`](localization.md))
- Touch support (Android) — defer until Android port phase
- Theme switching at runtime (light/dark/high-contrast for accessibility — see [`specs/accessibility.md`](accessibility.md))

## Milestone

Phase 7.5 (Presentation Layer) + Phase 12 (editor designer panel). Build a main-menu screen + an in-game HUD with health bar, hot-bar, and minimap. Controller-navigable. Localized to two languages.
