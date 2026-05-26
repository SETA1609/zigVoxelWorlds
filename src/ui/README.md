# `src/ui/`

> Runtime UI for shipped games. **Two orthogonal layers** per [`specs/ui.md`](../../docs/specs/ui.md). Editor UI is ImGui inside `src/editor/`.

## Two layers

| Layer | Location | Backend | Ships in every project? |
| --- | --- | --- | --- |
| **Widget kit** (immediate-mode HUD + simple menus) | `src/ui/widgets/` | SDL3 primitives via `libs/zig-cpp-platform-stack-adapter/` | ✅ Yes — every target needs at least a HUD |
| **Document UI** (custom-designed bespoke screens) | `src/ui/document.zig` | RmlUi via `libs/zig-cpp-ui-stack-adapter/` | ❌ Opt-in via `project.toml [ui] document = true` |

Per project memory `project-subsystem-swap-pattern` (decision 2026-05-26).

## Widget kit — what lives here

`src/ui/widgets/` provides ~20 pre-themed, controller-navigable widgets built on SDL3 primitives (`SDL_Renderer`, `SDL_ttf`):

- Primitives: `Label`, `Button`, `Image`, `Panel`
- Layout: `HBox`, `VBox`, `Grid`
- Game HUD: `ProgressBar`, `HotbarSlot`, `InventoryCell`
- Containers: `ListView`, `Modal`, `Tabs`, `Dropdown`, `Tooltip`, `Toast`
- Input: `TextInput`, `Checkbox`, `Slider`

Properties: retained-mode (you create a widget, it persists, events propagate), pre-themed (one theme struct, no style cascade), controller-navigable via `focus_neighbors`. Pure Zig. ~2–3k LoC engine code.

Use for: HUDs, simple menus, pause overlay, settings screens, prototyping.

## Document UI — when to use

`src/ui/document.zig` wraps the RmlUi document tree from `libs/zig-cpp-ui-stack-adapter/`. Use when the project needs:

- Real layout (flex, anchors, complex grids)
- Style cascade (CSS-like selectors)
- Transitions + animations
- RML/RCSS authored by content designers / modders

Daggerfall, Stardew, Atelier ship this. arena_modes does not.

## What lives in this directory

Planned files:

- `widgets/` (sub-dir) — the widget kit implementations (Label.zig, Button.zig, …)
- `widgets/theme.zig` — palette + typography defaults
- `widgets/layout.zig` — anchor + offset + HBox/VBox/Grid resolver
- `widgets/focus.zig` — controller focus-neighbor graph
- `document.zig` — opt-in RmlUi wrapper entry point (compile-time-gated by `build_options.has_document_ui`)
- `i18n.zig` — `{{tr:…}}` mustache substitution (routes to gettext-PO per [`specs/localization.md`](../../docs/specs/localization.md))
- `input_router.zig` — gamepad / D-pad / mouse-wheel → focus events
- `quickbar.zig` — universal 10-slot quickbar (used by all 5 targets — see [`specs/ui.md`](../../docs/specs/ui.md) § Quickbar)

## Discipline rules

1. Gameplay code calls **only** the `src/ui/` Zig API. Never reaches into the libs adapter directly.
2. Widget code uses SDL3 through platform-stack, not directly.
3. Document UI code is gated by `if (comptime build_options.has_document_ui)`; absent code doesn't tree-shake automatically without this gate.
4. Mod-author docs reference the Zig API, not RmlUi or SDL3 types.

## Layering rule

May import `core/`, `servers/render_server`, `platform/`, and (conditionally) `libs/zig-cpp-ui-stack-adapter/`. MUST NOT import `editor/` or `modules/`.

## References

- [`specs/ui.md`](../../docs/specs/ui.md) — full two-layer contract
- [`specs/localization.md`](../../docs/specs/localization.md) — gettext-PO i18n
- [`external-libs-catalog.md`](../../docs/external-libs-catalog.md) — RmlUi entry (§3 opt-in)
- RmlUi upstream: <https://github.com/mikke89/RmlUi> (MIT) — never `@import`ed from `src/ui/` directly
