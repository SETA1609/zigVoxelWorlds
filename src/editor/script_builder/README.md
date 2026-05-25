# `src/editor/script_builder/`

> Watches `<project>/scripts/` for changes, invokes the Zig toolchain to compile gameplay scripts into hot-reloadable dynamic libraries.

## What gets built

Per [scripting memory](../../../docs/specs/c-abi.md) and the engine-as-app model:

- `<project>/scripts/zig/*.zig` → `<project>/.cache/scripts/zig/<name>.{so,dll}` via `zig build-lib -dynamic`
- `<project>/scripts/cpp/*.cpp` → `<project>/.cache/scripts/cpp/<name>.{so,dll}` via `zig c++ -shared` against the stable C ABI

Both compile against the stable C ABI exposed by `core/` + `servers/` (no leaked Zig types, no leaked C++ types). See [`specs/c-abi.md`](../../../docs/specs/c-abi.md).

## Pipeline

1. File-system watcher fires (uses `platform/`'s watcher; OS-specific underneath)
2. Build job enqueued with debounce window (~200 ms)
3. Toolchain invoked in a sub-process (Zig bundles its own LLVM — no host toolchain dependency)
4. On success, `hot_reload/` is notified with the new `.so/.dll` path
5. On failure, errors are surfaced in the editor's build-output panel

## Why Zig handles both languages

Zig ships LLVM bundled, so `zig c++` is a self-contained C++ compiler. The engine doesn't need the user to install a separate Clang/MSVC. Single toolchain for both script flavors.

## References

- Hot-reload pattern: [`engine-references.md`](../../../docs/engine-references.md) § Godot · Module dispatch + Unreal § plugin descriptors (file-watch + reload — but our v1 skips UE's hot-reload mechanics for engine code; only project scripts hot-reload)

## Out of scope

Editing the source code (that's `code_editor/`), and reloading the freshly-built lib into the running game (that's `hot_reload/`).
