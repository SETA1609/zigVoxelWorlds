# `src/editor/export/launcher_template/`

> The launcher stub binary copied next to the built `libzvox-runtime` + `.pck` during export.

Source for a tiny executable that:

1. Locates `libzvox-runtime.{so,dll}` next to itself (`dlopen`/`LoadLibrary`)
2. Locates the matching `.pck` (same-name or appended footer; the three-location search lives in the runtime lib, not here)
3. Calls the runtime's `zvox_runtime_main(int argc, char** argv)` entry point
4. Exits with the runtime's return code

The launcher itself contains **no engine code, no module code, no assets** — it's the smallest possible bootstrap. The runtime DLL holds everything else, which keeps the binary footprint per-platform-target minimal and lets one launcher template work for every project.

## Naming at export time

`pack.zig` copies this template binary, renames it to `<project.name>` (or `<project.name>.exe` on Windows), and writes it next to the built lib + PCK.

## Why not bake the launcher into the runtime DLL

The launcher needs to be a separate executable because:

- Operating systems launch `.exe`/ELF binaries, not `.dll`/`.so`
- A separate launcher lets the user double-click a per-game name (`my-rpg.exe`) rather than always launching `zvox-runtime.exe`
- Mod managers and Steam launch options expect an `.exe` to point at
