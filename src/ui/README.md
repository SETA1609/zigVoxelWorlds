# `src/ui/`

> Runtime UI engine for shipped games (HUDs, menus, dialog, inventory). Editor UI is ImGui inside `src/editor/`.

## What this directory is

Thin Zig API that the engine + gameplay code calls. **It does not know about RmlUi.** The actual RmlUi-aware wrapper lives in `libs/zig-cpp-ui-stack-adapter/` (planned submodule) and exposes a stable C ABI; `src/ui/` contains just the typed Zig bindings to that C ABI plus engine glue (i18n call-site mustache, asset-DB lookup, gamepad input routing).

This is the same decoupling pattern used for Vulkan + Platform + (planned) Physics + Audio + Net. The C++ dependency lives in `libs/`; the Zig API lives here. See [`specs/ui.md`](../../docs/specs/ui.md) for the two-layer split diagram and off-ramp criteria.

## What lives here

Planned files:

- `document.zig` — `Document.load(guid)`, `Document.show/hide()`, hot-reload entry point
- `element.zig` — `Element` handle (opaque), `bindAttr`, `bindList`, `onEvent`
- `i18n.zig` — `{{tr:…}}` mustache substitution; routes to gettext-PO `t("...")` per [`specs/localization.md`](../../docs/specs/localization.md)
- `input_router.zig` — gamepad / D-pad / mouse-wheel → focus events
- `font_loader.zig` — feed font assets through the engine asset DB instead of the adapter's FS lookup

## Discipline rules (preserves the off-ramp)

1. Gameplay code calls **only** the `src/ui/` Zig API.
2. The C ABI surface lives in the libs adapter; `src/ui/` never imports anything `RmlUi`-shaped.
3. RML/RCSS files load through `Document.load(guid)`, never via filesystem hooks of the underlying lib.
4. Mod-author docs reference the Zig wrapper, not the underlying lib types.

If the four rules hold, swapping `libs/zig-cpp-ui-stack-adapter/` to a different backend (a future native impl, a different lib) is a libs-only change. `src/ui/` doesn't recompile.

## Layering rule

May import `core/`, `servers/render_server`, `platform/`, and the libs adapter's C ABI. MUST NOT import `editor/` or `modules/`.

## References

- [`specs/ui.md`](../../docs/specs/ui.md) — the contract
- [`specs/localization.md`](../../docs/specs/localization.md) — gettext-PO i18n
- [`external-libs-catalog.md`](../../docs/external-libs-catalog.md) — RmlUi entry in § 3
- RmlUi upstream: <https://github.com/mikke89/RmlUi> (MIT) — note: never `@import`ed from this directory
