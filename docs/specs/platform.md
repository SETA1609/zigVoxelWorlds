# Platform Adapter Spec

> The stable Zig API for window, events, action-mapped input, time, file I/O, per-OS native handle getters, gamepad, sensor, haptic, clipboard, filesystem paths, power, and IME. Lives as a sub-repo at `libs/zig-cpp-platform-stack-adapter/`. **SDL3 is the backend** (decision 2026-05-26), built via the [`castholm/SDL`](https://github.com/castholm/SDL) `build.zig.zon` dependency — not vendored (see [`../external-libs-catalog.md` § Building SDL3](../external-libs-catalog.md)). The "single Zig package, multiple backends as source files" architecture below is retained so a future native or alternate backend can be added without engine source changes — but no such alternate is planned for v1.0.
>
> This adapter has **no Vulkan dependency and no dependency on any other adapter.** Surface creation lives in [`zig-cpp-vulkan-stack-adapter`](https://github.com/SETA1609/zig-cpp-vulkan-stack-adapter) via its own per-OS `createX11Surface` / `createWaylandSurface` / `createWin32Surface` / `createAndroidSurface` functions. The engine bridges with a small helper (`src/render/surface.zig`) that calls a platform getter and the matching vulkan creator. **Both adapters are fully standalone** — no shared types, no cross-imports.
>
> Closes [`gaps.md` § 3 #14 input mapping](../gaps.md). Catalog row: [`external-libs-catalog.md` § 3](../external-libs-catalog.md) (Platform-stack meta-package). Backend choice rationale in [`tech-stack.md` § Windowing & Input](../tech-stack.md#windowing--input) and project memory `project-platform-backend-sdl3`. Pattern precedent: SDL3 itself, Godot's `DisplayServer`, Unreal's `IPlatformApplication`.

## Scope

A platform abstraction library exposing one stable Zig API surface backed by SDL3:

| Sub-repo version | Backend | Why |
| --- | --- | --- |
| **v0.x — v0.5** | GLFW (zlib) — hello-world only | Initial scaffolding; superseded |
| **v0.6 onward** | **SDL3** (zlib) | Android + Steam Deck + future Switch coverage; Steam Input gamepad mapping; gyro/IMU sensor; IME; haptic — all free with SDL3; GLFW has none of these. Decision 2026-05-26 — see project memory `project-platform-backend-sdl3`. |
| ~~Pure-Zig native~~ | **Withdrawn** | The earlier plan to ship pure-Zig X11/Wayland/Win32/Android backends in v1.x is withdrawn. Maintaining native backends across five platforms is solo-team-aspirational; SDL3 ships them all reliably. The multi-backend file layout below remains in case a concrete reason ever emerges to add a native backend — but doing so is no longer planned. |

Engine code looks identical regardless of which backend the adapter ships internally:

```zig
const platform = @import("platform");
const render   = @import("render");   // engine's own bridge module

const window = try platform.Window.create(.{
    .title = "zVoxRealms",
    .size = .{ .w = 1280, .h = 720 },
    .renderer = .vulkan,
});

while (platform.nextEvent()) |ev| switch (ev) {
    .key      => |k| input.dispatch(k),
    .resize   => |r| renderer.handleResize(r),
    .close    => running = false,
    .gamepad  => |g| input.dispatchGamepad(g),
}

if (input.actionPressed(.jump)) player.jump();

// Surface creation lives in the engine's small bridge helper, not in
// either adapter. The helper calls platform getters + vulkan creators
// per OS. Comptime switch — Linux builds compile only the .linux arm.
const surface = try render.createSurface(vk_instance, window);
```

## Sub-repo layout — single library, backends as files

**Not** lib-in-lib. One Zig package, one `build.zig.zon`, one stable version across all backends. Backends are source files; the build system picks one per target.

```
libs/zig-cpp-platform-stack-adapter/
├── LICENSE                          # MIT
├── README.md
├── build.zig                        # per-target backend selection (see below)
├── build.zig.zon                    # zero Vulkan deps; SDL3 via castholm/SDL dep
├── src/
│   ├── root.zig                     # public API — re-exports from `backend` module
│   ├── common.zig                   # shared types: Event, KeyCode, WindowOptions, ActionId (no native-handle type — those getters return inline anon structs)
│   ├── action_input.zig             # action-mapping layer (platform-agnostic)
│   ├── native_handle.zig            # per-backend native handle extraction
│   ├── backend/
│   │   ├── sdl3.zig                 # SDL3 backend — single file; SDL3 handles per-OS internally
│   │   └── native/                  # optional retained slot for a future native backend
│   │       ├── linux.zig            # runtime-dispatches X11 vs Wayland (compiled together)
│   │       ├── linux_x11.zig        # imported only by linux.zig
│   │       ├── linux_wayland.zig    # imported only by linux.zig
│   │       ├── windows.zig
│   │       ├── macos.zig
│   │       └── android.zig
│   └── tests/                       # integration tests against the public API
└── vendor/                          # empty for the SDL3 backend (SDL3 comes from
                                     # the castholm/SDL build.zig.zon dependency);
                                     # only used if a future native backend vendors
                                     # a C lib such as libxkbcommon
```

SDL3 is **not** vendored — it's a pinned `build.zig.zon` dependency on [`castholm/SDL`](https://github.com/castholm/SDL), which packages SDL's C sources for the Zig build system (decision 2026-05-27; see [`../external-libs-catalog.md` § Building SDL3](../external-libs-catalog.md)). The `vendor/` dir stays empty for the SDL3 backend. The `backend/native/` slot is retained as scaffolding for a possible future native backend; **it's not on the roadmap** as of 2026-05-26 but the architecture supports it without rewriting the public API.

## Build-time backend selection — per-target tree-shaking

```zig
// libs/zig-cpp-platform-stack-adapter/build.zig (sketch)
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const backend_choice = b.option(
        BackendChoice,
        "platform_backend",
        "Platform backend implementation",
    ) orelse .sdl3;

    // Pick the backend source file by (choice, target OS)
    const backend_root = switch (backend_choice) {
        .sdl3 => "src/backend/sdl3.zig",
        .native => switch (target.result.os.tag) {
            .linux   => "src/backend/native/linux.zig",
            .windows => "src/backend/native/windows.zig",
            .macos   => "src/backend/native/macos.zig",
            .android => "src/backend/native/android.zig",
            else => @panic("Unsupported target OS"),
        },
    };

    const backend_mod = b.createModule(.{
        .root_source_file = b.path(backend_root),
        .target = target,
        .optimize = optimize,
    });

    const platform_mod = b.addModule("platform", .{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    });
    platform_mod.addImport("backend", backend_mod);

    // SDL3 comes from the castholm/SDL build.zig.zon dependency — link its
    // per-target `SDL3` artifact rather than compiling SDL sources ourselves.
    if (backend_choice == .sdl3) {
        const sdl = b.dependency("sdl", .{ .target = target, .optimize = optimize });
        backend_mod.linkLibrary(sdl.artifact("SDL3"));
    }
}
```

The compiler walks the import graph from the chosen backend root. Files for other OSes / other backends are **never referenced** → never parsed, type-checked, or codegen'd. castholm/SDL's `build.zig` compiles the `SDL3` artifact for the requested target only, so per-OS SDL sources never enter the build.

### Per-target tree-shake guarantee

| Export target | Files compiled into the export | Files NOT touched |
| --- | --- | --- |
| `--target x86_64-linux-gnu -Dplatform_backend=sdl3` (default) | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (sdl3 branch) + `backend/sdl3.zig` + the castholm/SDL `SDL3` artifact (Linux target only) | SDL3's Windows/macOS/Android sources (SDL3's own CMake gates them); all `backend/native/*` files |
| `--target x86_64-windows-gnu -Dplatform_backend=sdl3` | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (sdl3 branch) + `backend/sdl3.zig` + the castholm/SDL `SDL3` artifact (Windows target only) | SDL3's Linux/macOS/Android sources; all `backend/native/*` files |
| `--target aarch64-linux-android -Dplatform_backend=sdl3` | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (sdl3 branch) + `backend/sdl3.zig` + the castholm/SDL `SDL3` artifact (Android target — includes `SDLActivity.java`) | SDL3's desktop sources |
| `--target x86_64-linux-gnu -Dplatform_backend=native` (future option, not on roadmap) | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (linux branch) + `backend/native/linux.zig` + `linux_x11.zig` + `linux_wayland.zig` | All Win32/macOS/Android backend files; the castholm/SDL `SDL3` artifact entirely |

Verification rule: `nm libzvox-runtime.so | grep -i 'win32\|wayland\|cocoa'` shows only the symbols for the target platform.

## Linux X11 vs Wayland — compiled together, dispatched at runtime

Both `linux_x11.zig` and `linux_wayland.zig` are compiled into the Linux binary. `linux.zig` decides at startup which to use:

```zig
// src/backend/native/linux.zig
pub fn init() !void {
    if (std.posix.getenv("WAYLAND_DISPLAY")) |_| {
        try wayland.init();
        active_subbackend = .wayland;
    } else if (std.posix.getenv("DISPLAY")) |_| {
        try x11.init();
        active_subbackend = .x11;
    } else {
        return error.NoDisplayServer;
    }
}
```

This matches what every Linux app does. Cost: ~50 KB of extra code per compiled-in sub-backend; reward: the same Linux binary works on both display servers.

Opt-out: `-Dlinux_backend=wayland` compiles only `linux_wayland.zig` (embedded kiosks, Wayland-only distros). Default is "both."

## The four design rules — non-negotiable

These rules are what keep a future backend swap cheap (SDL3 → native, SDL3 → SDL4, etc.). Violating any of them turns a future backend swap into a rewrite.

### Rule 1 — Design the API to the engine's needs, not SDL3's idioms

The public API in `src/root.zig` exposes **engine concepts**, not SDL3 concepts. No `SDL_AddEventWatch`-style C function-pointer registration leaking across the boundary. No `SDL_WindowFlags`. No raw `SDL_Window*` / `SDL_Event` types.

```zig
// GOOD — engine concept
pub fn nextEvent() ?Event;
pub const Event = union(enum) {
    key:      KeyEvent,
    mouse:    MouseEvent,
    resize:   ResizeEvent,
    close:    void,
    gamepad:  GamepadEvent,
    focus:    FocusEvent,
};

// BAD — would poison the API by leaking backend idioms
pub fn setKeyCallback(window: *Window, cb: fn (...) callconv(.c) void) void;
pub const WindowHint = enum { sdl_resizable, ... };
```

The SDL3 backend pumps `Event` from `SDL_PollEvent` into a queue; a future native backend would pump from its own sources. Both populate the same engine-visible `Event` shape.

### Rule 2 — Per-OS native handle getters; the renderer has matching creators; engine bridges

The platform adapter has **no Vulkan dependency and no dependency on any other adapter.** It exposes per-OS getter functions returning raw OS primitives (pointers + integers). The Vulkan-stack adapter has matching per-OS surface creators that take those same raw primitives. **No shared type crosses the boundary** — each adapter is fully standalone.

Why fully decoupled and not "shared NativeWindowHandle type": a shared union would force vulkan-stack to import a definition from platform-stack (or vice versa). Adopting per-OS function pairs eliminates even that. Each adapter is reusable in isolation — a headless tool linking platform-stack stays Vulkan-free; a Vulkan-stack consumer that uses a different windowing layer (SDL, raw X11, custom) needs no platform-stack import.

This is exactly the pattern SDL3 exposes via `SDL_GetWindowProperties` + per-backend `SDL_PROP_WINDOW_X11_*`/`WAYLAND_*`/`WIN32_*`/`ANDROID_*` keys, which Vulkan itself uses for its `VK_KHR_*_surface` extensions (separate `vkCreate*SurfaceKHR` functions, not one unified `vkCreateNativeSurfaceKHR`).

#### Per-backend, platform-stack implements per-OS getters

Each returns `null` when the current backend isn't that OS:

| Function | Returns (or `null` if not this OS) |
| --- | --- |
| `getX11Handle(window)` | `?{ display: *anyopaque, window: u64 }` |
| `getWaylandHandle(window)` | `?{ display: *anyopaque, surface: *anyopaque }` |
| `getWin32Handle(window)` | `?{ hinstance: *anyopaque, hwnd: *anyopaque }` |
| `getAndroidHandle(window)` | `?{ window: *anyopaque }` |
| `getCocoaHandle(window)` | `?{ layer: *anyopaque }` (deferred — `mission.md` defers macOS post-v1.0) |

Per-backend behavior:

| Backend | Behavior |
| --- | --- |
| SDL3 (Linux X11) | `getX11Handle` reads `SDL_GetWindowProperties(window)` for `SDL_PROP_WINDOW_X11_DISPLAY_POINTER` + `SDL_PROP_WINDOW_X11_WINDOW_NUMBER`; others return null |
| SDL3 (Linux Wayland) | `getWaylandHandle` reads `SDL_PROP_WINDOW_WAYLAND_DISPLAY_POINTER` + `SDL_PROP_WINDOW_WAYLAND_SURFACE_POINTER`; others null |
| SDL3 (Windows) | `getWin32Handle` reads `SDL_PROP_WINDOW_WIN32_INSTANCE_POINTER` + `SDL_PROP_WINDOW_WIN32_HWND_POINTER`; others null |
| SDL3 (Android) | `getAndroidHandle` reads `SDL_PROP_WINDOW_ANDROID_WINDOW_POINTER` (`ANativeWindow*`); others null |
| SDL3 (macOS, deferred) | `getCocoaHandle` reads `SDL_PROP_WINDOW_COCOA_METAL_LAYER_POINTER`; others null |
| Future native backend (if ever added) | Returns the adapter's internal raw handles directly (no SDL property indirection) |

#### Vulkan-stack implements matching per-OS surface creators

Each takes only the raw primitives — no platform-stack import:

```zig
pub fn createX11Surface(
    instance: vk.Instance,
    display: *anyopaque,
    window: u64,
) !vk.SurfaceKHR;

pub fn createWaylandSurface(
    instance: vk.Instance,
    display: *anyopaque,
    surface: *anyopaque,
) !vk.SurfaceKHR;

pub fn createWin32Surface(
    instance: vk.Instance,
    hinstance: *anyopaque,
    hwnd: *anyopaque,
) !vk.SurfaceKHR;

pub fn createAndroidSurface(
    instance: vk.Instance,
    window: *anyopaque,
) !vk.SurfaceKHR;
```

Each internally calls the matching `vkCreate*SurfaceKHR` from the corresponding `VK_KHR_*_surface` extension. None of these functions import anything from platform-stack.

#### Engine bridges in one helper

`src/render/surface.zig` is the engine's one-time bridge between the two standalone adapters. The cross-platform branch is `comptime`-resolved against the target OS, so each export contains only the matching branch.

```zig
const builtin  = @import("builtin");
const platform = @import("platform");
const vk_stack = @import("vulkan_stack");
const vk = vk_stack.vk;

pub fn createSurface(instance: vk.Instance, window: *platform.Window) !vk.SurfaceKHR {
    return switch (comptime builtin.target.os.tag) {
        .linux => blk: {
            if (platform.getWaylandHandle(window)) |h|
                break :blk try vk_stack.createWaylandSurface(instance, h.display, h.surface);
            if (platform.getX11Handle(window)) |h|
                break :blk try vk_stack.createX11Surface(instance, h.display, h.window);
            return error.NoDisplayServer;
        },
        .windows => {
            const h = platform.getWin32Handle(window) orelse return error.NoWin32Handle;
            return vk_stack.createWin32Surface(instance, h.hinstance, h.hwnd);
        },
        .android => {
            const h = platform.getAndroidHandle(window) orelse return error.NoAndroidHandle;
            return vk_stack.createAndroidSurface(instance, h.window);
        },
        else => @compileError("unsupported platform for Vulkan surface"),
    };
}
```

Engine call sites stay one line:

```zig
const surface = try render.createSurface(vk_instance, window);
```

Reference precedent: **SDL3's `SDL_GetWindowProperties()` per-backend property keys** (which replaced SDL2's `SDL_GetWindowWMInfo` tagged-struct API) + Vulkan's own `VK_KHR_*_surface` extension model. Per-platform getters on the windowing side, per-platform creators on the rendering side, caller pairs them. No shared "native handle" type — Vulkan itself doesn't have one. Our Zig API exposes typed per-OS getters (`getX11Handle`, etc.) so the engine never sees raw `SDL_PropertiesID` values; the SDL3 property layer is an implementation detail inside the adapter.

### Rule 3 — Pick engine-canonical behavior; document divergence honestly

Some things genuinely differ across OS / display server. Don't pretend they don't.

| Concern | Engine-canonical choice | Honest divergence note |
| --- | --- | --- |
| Initial window position | `setPosition(x, y)` best-effort | Wayland: compositor decides — request advisory |
| Window decorations | Default to OS-styled; toggle borderless via `WindowOptions.borderless` | Wayland: compositor may impose; X11: WM-dependent |
| High-DPI scaling | Per-window `scale_factor: f32` (1.0 = 100%) | macOS: backing-store scale; Windows: per-monitor DPI; Linux X11: round trips per monitor change; Wayland: scale per output |
| Cursor capture | `setRelativeMouseMode(bool)` | All backends support; behavior diverges on alt-tab — documented in `common.zig` doc-comment |
| Gamepad hotplug | Event-based `.gamepad_connected` / `.gamepad_disconnected` | Linux: `udev`; Windows: XInput callbacks; macOS: IOKit |
| Clipboard | UTF-8 string for v1.0 | Image/file clipboard deferred to v1.x |
| IME / dead keys | Composed text via `.text_input` event | Wayland text-input-protocol-v3; Win32 IME; X11 XIM — all surfaced as the same event |
| Window position query | Returns last-known; may lag on Wayland | Documented as "best-effort" |

A platform API that pretends differences don't exist always lies. One that exposes "best-effort + queryable capability flags" stays honest.

Capability flags exposed to engine code:

```zig
pub const Capabilities = struct {
    can_set_window_position: bool,    // false on Wayland
    can_query_window_position: bool,
    can_capture_global_input: bool,
    has_xinput_compatible_gamepad: bool,
    high_dpi_scale_per_monitor: bool,
    // ...
};
pub fn capabilities() Capabilities;
```

Engine code that needs platform-specific behavior queries the flag.

### Rule 4 — Integration tests run against every supported backend

Because the API surface is stable, every integration test must work against every backend the adapter ships. Today that's the SDL3 backend only. If a native backend is ever added, CI must run the test suite against both. CI matrix (sketch):

```yaml
# (sketch — actual CI lives elsewhere)
strategy:
  matrix:
    backend: [sdl3]                     # add `native` here if/when that backend lands
    target: [x86_64-linux-gnu, x86_64-windows-gnu, aarch64-linux-android]
```

If a native backend is ever added, divergence between backends would be caught immediately, not at engine integration time. This is the insurance policy for a future swap.

## Public API surface (v1.0)

### Window

```zig
pub const Window = opaque {};

pub const Renderer = enum { none, vulkan, opengl };

pub const WindowOptions = struct {
    title: []const u8,
    size: Size = .{ .w = 1280, .h = 720 },
    position: ?Position = null,         // null → OS-default
    fullscreen: bool = false,
    resizable: bool = true,
    borderless: bool = false,
    renderer: Renderer = .vulkan,       // GPU API bound at creation; engine always .vulkan
    parent: ?*Window = null,            // for modal / child windows
};

pub const Size = struct { w: u32, h: u32 };
pub const Position = struct { x: i32, y: i32 };

pub fn create(opts: WindowOptions) !*Window;
pub fn destroy(window: *Window) void;
pub fn setTitle(window: *Window, title: []const u8) void;
pub fn setSize(window: *Window, size: Size) void;
pub fn setPosition(window: *Window, pos: Position) void;
pub fn size(window: *Window) Size;
pub fn position(window: *Window) Position;            // best-effort
pub fn scaleFactor(window: *Window) f32;
pub fn shouldClose(window: *Window) bool;
```

> **Renderer choice — the engine uses Vulkan only.** `WindowOptions.renderer` selects the GPU API bound at window creation (SDL sets `SDL_WINDOW_VULKAN` vs `SDL_WINDOW_OPENGL` up front). The **engine always passes `.vulkan`** and never exercises the OpenGL path. `.opengl` (a managed GL context + `glSwapWindow` / `glGetProcAddress`) and `.none` are **library-level capabilities of [`zig-cpp-platform-stack-adapter`](https://github.com/SETA1609/zig-cpp-platform-stack-adapter) for other consumers** — kept so the adapter isn't Vulkan-locked, but outside the engine's usage. GL API details live in that library's `docs/mission.md`.

### Events

```zig
pub const Event = union(enum) {
    key:           KeyEvent,
    mouse_button:  MouseButtonEvent,
    mouse_motion:  MouseMotionEvent,
    mouse_scroll:  MouseScrollEvent,
    resize:        ResizeEvent,
    focus:         FocusEvent,
    close:         void,                // window-close requested
    gamepad:       GamepadEvent,
    text_input:    TextInputEvent,      // IME-composed text
    file_drop:     FileDropEvent,
};

pub fn nextEvent() ?Event;              // returns null when queue empty
pub fn pollAllEvents() void;            // drives the backend's event pump
```

### Action-mapped input — closes `gaps.md` § 3 #14

Direct key/button reads are anti-patterns for rebindable games. Action mapping is engine-first.

```zig
pub const ActionId = enum {
    move_forward, move_back, move_left, move_right,
    jump, sprint, crouch,
    attack_primary, attack_secondary,
    interact, inventory, menu_pause, ...
    // game-specific actions register via the C ABI per specs/c-abi.md
};

pub const ActionBinding = union(enum) {
    key:           KeyCode,
    mouse_button:  MouseButton,
    gamepad_button:GamepadButton,
    gamepad_axis:  struct {
        axis: GamepadAxis,
        threshold: f32 = 0.15,    // deadzone — values below this collapse to 0
        smooth: f32 = 0.0,        // 0 = off; e.g. 0.05 = light EMA smoothing
        scale: f32 = 1.0,         // analog magnitude → action value scaling
        invert: bool = false,     // negate axis (e.g. Y-axis aim invert)
    },
    composite:     []const ActionBinding,  // any-of: triggers if any member triggers
};

pub fn bindAction(action: ActionId, binding: ActionBinding) void;
pub fn unbindAction(action: ActionId, binding: ActionBinding) void;
pub fn actionPressed(action: ActionId) bool;
pub fn actionJustPressed(action: ActionId) bool;
pub fn actionJustReleased(action: ActionId) bool;
pub fn actionValue(action: ActionId) f32;    // analog (gamepad axis), modifier-chain applied
```

Bindings load from TOML at startup per [`specs/data-schemas.md`](data-schemas.md). User-rebound bindings save to per-save settings per [`specs/save-ux.md`](save-ux.md).

Mods register new actions via the public C ABI ([`specs/c-abi.md`](c-abi.md)). The platform adapter only knows about engine-core ActionIds at compile time; mod-defined IDs go through a runtime-allocated `u32` slot.

### Input Mapping Contexts — stackable binding sets

Pattern borrowed from Unreal's Enhanced Input. Without it, every gameplay/menu/dialog/vehicle system polls every action and gates with mode flags — fragile and verbose. Context stack inverts the control: the active context decides which actions are live; consumers don't gate.

```zig
pub const InputContextId = enum {
    gameplay,        // base layer — always at stack bottom
    ui_menu,         // any menu/settings panel
    dialog,          // dialog modal per specs/dialog.md
    trade,           // trade modal
    inventory,       // inventory open
    crafting,        // crafting station UI
    vehicle,         // riding / driving (v1.x)
    builder_mode,    // free-camera / no-clip editor mode
    cinematic,       // scripted cutscene (most actions disabled)
    // mod-defined contexts register via the C ABI per specs/c-abi.md
};

pub fn pushContext(ctx: InputContextId) void;
pub fn popContext() InputContextId;        // returns popped ctx
pub fn replaceTopContext(ctx: InputContextId) void;
pub fn activeContext() InputContextId;     // topmost
pub fn isContextActive(ctx: InputContextId) bool;  // anywhere in stack
```

### Resolution

Each context declares its own binding table in TOML. When evaluating an action:

1. Walk the stack top-to-bottom
2. First context with an explicit binding for that `ActionId` wins
3. If no context binds it explicitly, fall back to `gameplay` (always at the bottom)
4. If `gameplay` doesn't bind it either, the action is inert in this frame

Higher contexts can **mask** lower ones: declare `action.attack_primary = "none"` in `dialog` context, and clicking won't attack while dialog is open — even though gameplay would otherwise fire `attack_primary` on click.

### Lifecycle

Contexts integrate with existing modal/scene events:

| Trigger | Effect on stack |
| --- | --- |
| Dialog opens ([`specs/dialog.md`](dialog.md)) | Engine pushes `dialog` |
| Dialog closes | Engine pops `dialog` |
| Inventory opens | Engine pushes `inventory` |
| Trade opens | Engine pushes `trade` |
| Cutscene starts ([`specs/scene.md` § Scripted cutscenes](scene.md)) | Engine pushes `cinematic` |
| Player enters vehicle (v1.x) | Engine pushes `vehicle` |

Mods that introduce new modal interactions push their own contexts in `init_servers` per [`engine-vs-game.md`](../engine-vs-game.md).

### Multi-stack? — no, single global stack

We use ONE global stack, not per-player stacks. Justifications:

- Engine is single-machine; multiplayer per [`specs/multiplayer.md`](multiplayer.md) replicates state, not input
- A target game with split-screen (post-v1.0 scope) would need per-player stacks; revisit then
- Single stack matches Godot's `InputMap` (global), simpler than Unreal's per-PlayerController stacks

### Synthetic action injection

Pattern borrowed from Godot's `InputEventAction.parse_input_event(...)`. Lets engine code drive input downstream without a real keypress.

```zig
pub fn injectAction(action: ActionId, pressed: bool, value: f32) void;
pub fn injectActionEvent(action: ActionId, kind: enum { just_pressed, just_released, held, released }) void;
```

Uses:

| Use case | Where |
| --- | --- |
| Integration tests | Drive the player from a script — see [`specs/testing.md`](testing.md) |
| Scripted cutscenes | "release the player to walk forward at this beat" per [`specs/scene.md` § Scripted cutscenes](scene.md) |
| AI debugging | Replay recorded sessions for regression isolation |
| Demo recordings | Capture session input + replay it ([`ROADMAP.md` § Phase 15 replay milestone](../ROADMAP.md)) |
| Tutorial overlays | "press X to jump" tutorial replays the input visually |
| Bot / training | NPC AI driving the player avatar for headless training (post-v1.0) |

Injected actions are first-class: `actionPressed`/`actionJustPressed` return true; the `.action` event variant fires; downstream consumers can't distinguish injected from real input. Multiplayer note: injected actions on the client are **NOT** replicated to the server unless explicitly sent — guards against test code corrupting an authoritative session.

### What we deliberately don't include

| Pattern | Source | Why we skip for v1.0 |
| --- | --- | --- |
| **Full modifier chain** (Unreal: composable Negate → Scalar → Swizzle → Smooth) | Unreal Enhanced Input | Over-engineered for our scope. Axis-binding has the four real modifiers (threshold/smooth/scale/invert) inline. Revisit if a target game needs more |
| **Trigger taxonomy** (Hold, Tap, Chord, Combo, Pulse) | Unreal Enhanced Input | State machines on top of base actions are clearer for our four target games. Game-side combat-combo systems live in gameplay code, not in the platform layer |
| **Scene-tree event bubbling** | Godot | We don't have a scene tree the way Godot does; entities live in ECS. UI focus + gameplay arbitration handled by the engine's UI layer (per [`specs/ui.md`](ui.md)) reading from a separate `nextEvent()` consumer |
| **Per-player input stacks** | Unreal PlayerController | Engine is single-machine; multiplayer replicates state. Revisit for split-screen if a v1.x target game needs it |
| **`InputEventAction`-equivalent for raw key/mouse events** | Godot can inject any InputEvent | We expose only action-level injection. Mod code that wants raw-input injection must implement at the action layer; keeps the platform surface narrow |

### Time

```zig
pub fn now() u64;                       // monotonic ns
pub fn perfFreq() u64;                  // ticks per second
pub fn perfCounter() u64;               // raw perf counter
pub fn sleep(ns: u64) void;
```

### File I/O

Most file I/O routes through Zig stdlib (`std.fs`). The platform adapter only exposes things stdlib doesn't:

```zig
pub fn appDataDir(allocator: Allocator, app_name: []const u8) ![]u8;
                                        // %APPDATA%\app_name on Windows
                                        // ~/.local/share/app_name on Linux
                                        // ~/Library/Application Support/app_name on macOS
pub fn appCacheDir(allocator: Allocator, app_name: []const u8) ![]u8;
pub fn openWithSystemDefault(path: []const u8) !void;
                                        // "xdg-open" / "start" / "open"
```

### Native window handle getters (for renderer surface creation)

Per-OS getters returning raw OS primitives. No shared `NativeWindowHandle` type — the platform adapter and the vulkan-stack adapter never share a type definition. Each getter returns `null` when the current backend isn't that OS.

```zig
pub fn getX11Handle(window: *Window) ?struct {
    display: *anyopaque,            // Display* (Xlib) or xcb_connection_t*
    window: u64,                    // X11 window ID
};

pub fn getWaylandHandle(window: *Window) ?struct {
    display: *anyopaque,            // wl_display*
    surface: *anyopaque,            // wl_surface*
};

pub fn getWin32Handle(window: *Window) ?struct {
    hinstance: *anyopaque,          // HINSTANCE
    hwnd: *anyopaque,               // HWND
};

pub fn getAndroidHandle(window: *Window) ?struct {
    window: *anyopaque,             // ANativeWindow*
};

pub fn getCocoaHandle(window: *Window) ?struct {
    layer: *anyopaque,              // CAMetalLayer* — deferred post-v1.0
};

pub fn requiredVulkanInstanceExtensions() []const [*:0]const u8;
                                         // ↑ just C-string array — no Vulkan TYPES involved
```

`requiredVulkanInstanceExtensions()` returns the names of `VK_KHR_*_surface` extensions the renderer must enable on `vk.Instance` creation (e.g. `VK_KHR_xlib_surface` on X11). It returns C strings — no Vulkan type dependency.

The per-OS getters expose raw OS primitives. The engine's `src/render/surface.zig` helper picks the right getter at `comptime`-resolved compile time (based on `builtin.target.os.tag`) and passes the result to the corresponding `vk_stack.create*Surface(...)` function. See Rule 2 above for the helper shape.

**The platform adapter has zero Vulkan dependency and zero dependency on the vulkan-stack adapter.** A headless server, a config-editor tool, or any non-rendering binary can use this adapter without dragging vulkan-zig along.

## Integration with other subsystems

| System | Touch point |
| --- | --- |
| [`specs/c-abi.md`](c-abi.md) | Mod-defined ActionIds + mod-defined InputContextIds allocated through the public C ABI |
| [`specs/ui.md`](ui.md) | UI focus graph consumes raw key/button events; action-mapped input feeds the same UI controller-nav system. UI panels push their own context (`ui_menu`/`inventory`/etc.) on open |
| [`specs/dialog.md`](dialog.md) | Opens push `dialog` / `trade` contexts; close pops them. Engine drives this — gameplay code never branches on "is dialog open" |
| [`specs/scene.md`](scene.md) | Cutscene triggers push `cinematic` context (most actions disabled); skip-cutscene action stays bound. Synthetic action injection drives scripted player actions during cutscenes |
| [`specs/save-ux.md`](save-ux.md) | Per-save user rebindings persist via the save format |
| [`specs/accessibility.md`](accessibility.md) | Full key + controller remap is built on `bindAction`/`unbindAction`; head-bob + FoV sliders are engine-side, not platform |
| [`specs/multiplayer.md`](multiplayer.md) | Platform adapter is single-machine only; network input replication is engine-layer. Injected actions are local-only — never replicated to authoritative server |
| [`specs/testing.md`](testing.md) | Integration tests drive the player via `injectAction` — same code path as real input. Tests run against both backends per design rule 4 |
| [`specs/audio.md`](audio.md) | Audio backend is miniaudio — independent of platform adapter (see catalog § Note on Platform-stack exclusions) |
| Renderer (Vulkan-stack adapter) | Engine helper `src/render/surface.zig` picks the right per-OS getter (`platform.getX11Handle` / `getWin32Handle` / etc.) and passes the raw primitives to the matching `vk_stack.createX11Surface` / `createWin32Surface` / etc. **No shared type and no cross-adapter import** — both adapters are fully standalone |

## Reference patterns

| Engine | Where | What to adapt |
| --- | --- | --- |
| **SDL3** ✅ best fit | `$REFS/godot/thirdparty/sdl/` (Godot's vendored copy) · upstream: <https://github.com/libsdl-org/SDL> · `src/video/{x11,wayland,cocoa,windows}/` · `src/events/SDL_events.c` | THE reference impl of "one stable API, many backends." Study the event-queue pattern + the per-platform `SDL_VideoDevice` vtable. Don't port C++; absorb the design |
| **Godot — DisplayServer** | `$REFS/godot/servers/display_server.h` + `platform/{linuxbsd,windows,macos,android}/display_server_*.cpp` | Abstract base + per-platform subclasses. Same idea, C++ machinery. Read the API surface; implement via Zig comptime dispatch instead of vtables |
| **Unreal — IPlatformApplication** | `$REFS/UnrealEngine/Engine/Source/Runtime/ApplicationCore/Public/GenericPlatform/` + per-platform under `Linux/`, `Windows/`, `Apple/`, `Android/` | Heavier than we want, but worth reading for the breadth of platform concerns it covers (drag-drop, IME, accessibility hooks) |
| **SDL3** | <https://github.com/libsdl-org/SDL> · `src/video/{x11,wayland,cocoa,windows,android}/`, `src/joystick/`, `src/sensor/`, `src/haptic/` | Our backend. Study `src/video/SDL_sysvideo.h` for the `SDL_VideoDevice` vtable — that's the shape any future native backend should match in spirit |
| **GLFW** | <https://github.com/glfw/glfw> · `src/{x11,wayland,cocoa,win32}_*.c` | Earlier v0 hello-world backend, superseded. Still a useful reference for a *minimal* per-OS windowing implementation if a future native backend is ever attempted |
| **Hazel** | `$REFS/Hazel/Hazel/src/Platform/` | GLFW-coupled, no abstraction. **Anti-reference** — this is what NOT to do |

Adaptation rule per [`engine-references.md` § Legal](../engine-references.md): read, understand, reimplement in Zig. SDL is the closest license-compatible (zlib) reference but **do not copy verbatim** — design your API to engine needs (rule 1), use SDL only as pattern validation.

## Backend swap plan — GLFW → SDL3 (active 2026-05-26)

This replaces the earlier "GLFW v0 → native v1.x" migration. The sub-repo today contains a GLFW hello-world; the next sub-repo commits replace that with SDL3.

### Pre-flight checks

- Engine consumers compile against the existing GLFW hello-world surface (the API is intentionally narrow at this stage)
- All four design rules audited — no GLFW idioms leaked into the public API
- Capability flag set committed; SDL3-only features (gamepad, sensor, haptic, IME, power) get capability flags

### Swap GLFW → SDL3 incrementally

| Step | Scope | Verification |
| --- | --- | --- |
| 1 | Add [`castholm/SDL`](https://github.com/castholm/SDL) as a pinned `build.zig.zon` dependency (HIDAPI elected BSD-3-Clause); link its `SDL3` artifact | `zig build` of the adapter succeeds with SDL3 linked |
| 2 | Replace `backend/glfw.zig` with `backend/sdl3.zig` — window + event pump + Vulkan surface property reads | Hello-world opens an SDL3 window on Linux X11 and Wayland; surface query returns valid props |
| 3 | Wire native-handle getters via `SDL_GetWindowProperties` | `platform.getX11Handle` / `getWaylandHandle` / `getWin32Handle` / `getAndroidHandle` each return inline-anon-struct of raw primitives or `null` |
| 4 | Action-mapped input through `SDL_PollEvent` | Synthetic + real inputs route through the same code path; integration tests pass |
| 5 | Add gamepad (`SDL_Gamepad`), sensor (`SDL_Sensor`), haptic (`SDL_GamepadRumble`), clipboard (`SDL_SetClipboardText`), filesystem paths (`SDL_GetPrefPath`), power (`SDL_GetPowerInfo`), IME (`SDL_StartTextInput`) | Each gets its own integration test |
| 6 | Android sub-target build — pull in `SDLActivity.java` template | Android emulator integration test |

Each step is its own atomic sub-repo commit. Engine repo doesn't touch this until the SDL3 adapter version is bumped.

### Release `libs/zig-cpp-platform-stack-adapter@v0.6`

- Engine bumps the dep version in `build.zig.zon` from `v0.5` (GLFW hello-world) to `v0.6` (SDL3)
- One engine commit: `chore(deps): bump platform-adapter v0.5 → v0.6 (SDL3 backend)`
- Roll-back is reverting the version pin; engine source unchanged on either side

### Cleanup

- Remove `backend/glfw.zig` and `vendor/glfw/` from the sub-repo at v0.6
- `backend/native/` directory **retained** as scaffolding for a possible future native backend; no implementation in v0.6

## Open decisions

- **macOS backend timing** — `mission.md` defers macOS post-v1.0. If a community user contributes a macOS backend earlier, integrate; otherwise wait
- **Android backend touchscreen events** — `.touch` event type vs treating touch as mouse — decide during Android port (Phase post-v1.0)
- **Multiple-window support** — v1.0 ships single primary window only; multi-window (editor + playtest in separate OS windows) deferred to Phase 12
- **Game controller force feedback** — SDL3 provides this via `SDL_GamepadRumble`; ship in v0.6, no longer deferred

## Milestone

Phase 1 — adopt the Platform-stack adapter alongside the Vulkan-stack adapter. Verifications:

- Window opens, events received, key + mouse + gamepad input working end-to-end against the SDL3 backend (v0.6+)
- Action bindings load from TOML; per-save rebindings persist
- Context stack: push `dialog` → gameplay's `attack_primary` is masked → pop → gameplay's binding is live again
- Synthetic injection: `injectAction(.jump, true, 1.0)` triggers the same code path as a real spacebar press; verified by integration test
- Axis-binding modifiers: deadzone + smooth + invert applied correctly on gamepad sticks
- Native handle getters: `platform.getX11Handle(window)` / `getWaylandHandle` / `getWin32Handle` / `getAndroidHandle` each return either inline-anon-struct of raw primitives or `null`. The renderer's matching per-OS `createX11Surface` / `createWin32Surface` / etc. consume them. No shared type between adapters
- Build verification: `nm` on a Linux export shows no Win32/macOS/Android symbols. The platform adapter emits zero `vk*` symbols (no Vulkan dep). The vulkan-stack adapter emits zero `SDL_*` / `x11*` / `wl_*` / `win32*` symbols beyond the matching `vkCreate*SurfaceKHR` call

A future native backend (if ever undertaken) would follow the same verification matrix against `-Dplatform_backend=native` in CI. Not on the post-v1.0 roadmap as of 2026-05-26 — SDL3 is the planned backend through v1.0+.
