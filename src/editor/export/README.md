# `src/editor/export/`

> Per-project export pipeline. Tree-shakes enabled modules from `project.toml`, builds a project-specific `libzvox-runtime.{so,dll}`, packs assets into a PCK, and pairs it with a small launcher executable. Editor-only.

## Subtree

- `tree_shake.zig` — read `project.toml`, walk `modules/<name>/config.zig` to resolve transitive deps + the comptime-enabled set
- `link.zig` — invoke `zig build-lib -dynamic` over the tree-shaken module set + the engine runtime
- `pack.zig` — PCK writer (Godot-style magic `0x43504447` + flat directory + blobs); see [`engine-references.md`](../../../docs/engine-references.md) § Godot · Export — PCK three-location loader
- [`launcher_template/`](launcher_template/README.md) — the tiny stub binary copied next to the built lib + PCK

## Output layout

```text
<project>/export/<platform>/
├── my-rpg               # launcher (copied from launcher_template/, renamed)
├── libzvox-runtime.so   # tree-shaken: only project-enabled modules linked
└── my-rpg.pck           # asset bundle
```

The runtime finds `my-rpg.pck` via Godot's three-location search: separate file, embedded section, or appended footer (`core/io/file_access_pack.cpp:218-285`).

## Divergence from Godot

Godot ships a fat pre-built template per platform and copies it (`editor/export/editor_export_platform_pc.cpp:157-208` literal `da->copy(template_path, p_path)`). zVoxRealms instead **builds the per-project library from source** at export time — possible because Zig ships LLVM bundled, so no host toolchain is required. Smaller exports, only the modules used. See [`engine-references.md` § Godot · Export](../../../docs/engine-references.md).

## Cross-compilation

Zig is a cross-compiler. The exporter can build a Windows `.dll` from Linux, an Android `.so` from macOS, etc., without separate toolchains. `tree_shake.zig` selects targets from the project's allowed platform list in `project.toml`.

## Layering rule

Editor-only. May import `core/`, `platform/`, all `modules/`, and the libs adapters. MUST NOT be imported from runtime code.
