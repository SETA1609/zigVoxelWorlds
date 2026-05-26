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

## WASM was considered for hot-reload and rejected (2026-05-26)

A WASM-based hot-reload path (compile scripts to WASM in dev, native dylib for export) was evaluated. **Rejected** in favor of the native-dylib pipeline above.

Reasons:

| Concern | Native dylib (chosen) | WASM (rejected) |
| --- | --- | --- |
| **Dev/prod parity** | Same runtime in dev + export — what you tested IS what ships | Different runtime — perf, allocator behavior, memory layout diverge |
| **Editor playtest perf** | Same as exported game | ~1.5–3× slower (WASM bytecode + sandbox overhead) |
| **Debugging** | `gdb` / `lldb` / native debuggers work; breakpoints in source | WASM debugging is improving (DWARF for WASM) but not at parity for non-browser hosts |
| **Toolchain** | One: `zig build-lib -dynamic` / `zig c++ -shared` | Two — native for export + WASM for dev. Doubles CI matrix. |
| **C ABI calls** | Direct function calls | Marshaled through WAMR host-call interface; per-call overhead |
| **dlopen-cache issue** | Real on Linux; solved by versioned filenames (Handmade Hero pattern) | Avoided naturally — small win |

The **dev/prod divergence is the deciding cost**. If editor playtest behaves differently from the exported game, every bug becomes a guess at which side reproduced it.

WASM IS used elsewhere in the engine — for **sandboxed third-party mods only** (see `libs/zig-cpp-wasm-stack-adapter/` + the WAMR row in [`external-libs-catalog.md`](../../../docs/external-libs-catalog.md) § 3 and `specs/mod-manager.md`). That's a different problem (untrusted code, cross-platform mod packaging, sandboxing) with the opposite trade-offs.

## References

- Unreal hot-reload mechanics — read for the *idea*, do not port (see [`engine-references.md`](../../../docs/engine-references.md) § Unreal · Plugin descriptors — "Skip UE's hot-reload mechanics for v1" turned out to apply to engine modules; for *project scripts* we do implement it, but the C ABI boundary makes it simpler)
- Casey Muratori's "Handmade Hero" episodes on Windows DLL hot-reload — practical primer for the versioned-filename + state-snapshot pattern
