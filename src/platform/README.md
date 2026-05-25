# `src/platform/`

> Window, input, file I/O, threading, process spawning. The OS abstraction layer.

Planned files:

- Window creation + event loop (driven by `libs/zig-cpp-platform-stack-adapter/` GLFW C ABI)
- Input mapping (keyboard, mouse, gamepad)
- File I/O wrappers (sync + async)
- Thread pool / job system
- `std.process.Child` helpers for re-exec (PM → editor, per [`engine-references.md`](../../docs/engine-references.md) § Godot · Project Manager)

See [`specs/platform.md`](../../docs/specs/platform.md) for the planned API.

**Layering rule:** platform may import `core/` only.
