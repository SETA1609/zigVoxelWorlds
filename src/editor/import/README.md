# `src/editor/import/`

> Asset import pipeline. Reads source files from `<project>/assets/`, writes baked binary outputs into `<project>/.import/<guid>.<ext>`. Runtime loads from `.import/`, never from source. Editor-only.

Planned importers:

| File | Source format | Baked output | Notes |
| --- | --- | --- | --- |
| `texture.zig` | PNG, JPEG, TGA | KTX2 (BC7/ASTC) | via `basis_universal` + `libktx` from the asset-stack adapter |
| `mesh.zig` | glTF 2.0 | engine mesh binary | via `cgltf` from the asset-stack adapter; meshoptimizer post-pass |
| `voxel_model.zig` | MagicaVoxel `.vox` | engine voxel binary | small custom parser; pattern from Luanti `mapblock.h` |
| `audio.zig` | WAV, OGG, FLAC | decoded buffer or streaming chunk | via miniaudio decoders from the audio-stack adapter |
| `shader.zig` | GLSL, HLSL | SPIR-V | via glslang/shaderc from the vulkan-stack adapter |
| `data.zig` | TOML game data | validated cached binary | hot-path data never re-parses TOML (per [`project-structure.md`](../../../docs/project-structure.md) Conventions) |

## Pipeline shape

1. `assets/` change watcher (or manual "Reimport" UI action) enqueues a job
2. Importer reads source + `<project>/assets/.assetdb.toml` (GUID lookup, content hash)
3. If hash matches existing `.import/<guid>.<ext>` → skip
4. Otherwise, run importer, write output, update `.assetdb.toml` with the new hash
5. Notify the editor + hot-reload host about the new baked asset

## Why bake to binary

Hot paths (chunk meshing, draw, audio mixing) must not parse text. Bake once at import time, store the parsed/optimized form, load O(1) at runtime. See [`project-structure.md` § Conventions](../../../docs/project-structure.md) ("Hot paths don't parse text").

## References

- Godot's `editor/import/` directory — the layout pattern, particularly `core/io/resource_importer.cpp` (the import-job registry idea)
- Unreal `Editor/AssetTools/` — the asset-type-to-factory mapping (worth a read; don't port)

## Layering rule

Editor-only. May import `core/`, `platform/`, and the appropriate libs adapters for the source format. MUST NOT be imported from runtime code.
