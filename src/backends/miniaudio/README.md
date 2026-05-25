# `src/backends/miniaudio/`

> `audio_server` implementation. Wraps [miniaudio](https://miniaud.io/) through `libs/zig-cpp-audio-stack-adapter/` (planned).

Holds:

- Device init + listener setup
- Sound source management (3D + 2D)
- Bus / mixer hierarchy
- Loading hooks (decoded buffers from `editor/import/audio.zig`)

See [`specs/audio.md`](../../../docs/specs/audio.md) for the API.
