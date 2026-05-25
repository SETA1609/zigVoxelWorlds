# `src/editor/hot_reload/`

> Dynamic-library reload host — swap a freshly built script `.so/.dll` into the running game without restarting the editor.

## Pipeline

1. `script_builder/` finishes a build and signals: "new lib at `<path>`"
2. Hot-reload host snapshots persistent state owned by the old lib (per the stable C ABI's `serialize_state`/`deserialize_state` hooks)
3. `dlclose` (Linux/macOS) / `FreeLibrary` (Windows) the old lib
4. `dlopen` / `LoadLibrary` the new one — Linux requires a unique path each reload to defeat the dlopen cache, so the script builder writes versioned filenames (`foo.v17.so`)
5. Call the new lib's `register_types()` against the engine's type registry
6. Restore state via the new lib's `deserialize_state`

If any step fails, the old lib stays loaded; the editor surfaces the error.

## C ABI requirements

Reloadable libs must:

- Have **no globals with non-trivial destructors** (would crash on `dlclose`)
- Export `register_types` and `unregister_types`
- Export `serialize_state` + `deserialize_state` if they hold persistent runtime state
- Hold zero raw pointers into engine memory across reload boundaries — only `Handle`s

See [`specs/c-abi.md`](../../../docs/specs/c-abi.md) for the full contract.

## Editor-only

Hot reload is a development feature. Exported games do not include this code path. Scripts in exported games are statically linked into `libzvox-runtime.so` (per `editor/export/tree_shake.zig`).

## References

- Unreal hot-reload mechanics — read for the *idea*, do not port (see [`engine-references.md`](../../../docs/engine-references.md) § Unreal · Plugin descriptors — "Skip UE's hot-reload mechanics for v1" turned out to apply to engine modules; for *project scripts* we do implement it, but the C ABI boundary makes it simpler)
- Casey Muratori's "Handmade Hero" episodes on Windows DLL hot-reload — practical primer
