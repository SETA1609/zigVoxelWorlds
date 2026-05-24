# Platform Adapter Spec

> The stable Zig API for window, events, action-mapped input, time, file I/O, and native window handle exposure. Lives as a sub-repo at `libs/zig-cpp-platform-stack-adapter/`. **Single Zig package, multiple backends as source files** — backend selected at build time per target.
>
> This adapter has **no Vulkan dependency.** Surface creation lives in the [`zig-cpp-vulkan-stack-adapter`](https://github.com/SETA1609/zig-cpp-vulkan-stack-adapter) and consumes the `NativeWindowHandle` type defined here. The two adapters are decoupled — `vulkan-stack → platform-stack` is a one-way data dep, not a cycle.
>
> Closes [`gaps.md` § 3 #14 input mapping](../gaps.md). Catalog row: [`external-libs-catalog.md` § 3](../external-libs-catalog.md) (Platform-stack meta-package). Migration design from [`tech-stack.md` § Windowing & Input](../tech-stack.md#windowing--input). Pattern precedent: SDL, Godot's `DisplayServer`, Unreal's `IPlatformApplication`.

## Scope

A platform abstraction library exposing one stable Zig API surface. The implementation under the hood can swap between major versions of the sub-repo **without engine source changes**:

| Sub-repo version | Backend | Why |
| --- | --- | --- |
| **v0.x — v1.0** | GLFW (Zlib) | Rapid iteration; cross-platform coverage out of the box; well-tested; Hazel-precedent |
| **v1.x onward** | Pure-Zig native: X11, Wayland, Win32, Android | No C deps; smaller export per-target; matches `tech-stack.md` § Windowing long-term goal; no GLFW thread-affinity quirks |

Engine code looks identical across the migration:

```zig
const platform = @import("platform");
const vk_stack = @import("vulkan_stack");

const window = try platform.Window.create(.{
    .title = "zVoxRealms",
    .size = .{ .w = 1280, .h = 720 },
    .vulkan_compatible = true,
});

while (platform.nextEvent()) |ev| switch (ev) {
    .key      => |k| input.dispatch(k),
    .resize   => |r| renderer.handleResize(r),
    .close    => running = false,
    .gamepad  => |g| input.dispatchGamepad(g),
}

if (input.actionPressed(.jump)) player.jump();

// Surface creation: platform exposes the raw handle; vulkan-stack creates the surface.
// Platform layer has no Vulkan dep; engine bridges in one line.
const handle  = platform.nativeHandle(window);
const surface = try vk_stack.createSurface(vk_instance, handle);
```

## Sub-repo layout — single library, backends as files

**Not** lib-in-lib. One Zig package, one `build.zig.zon`, one stable version across all backends. Backends are source files; the build system picks one per target.

```
libs/zig-cpp-platform-stack-adapter/
├── LICENSE                          # MIT
├── README.md
├── build.zig                        # per-target backend selection (see below)
├── build.zig.zon                    # zero Vulkan deps; GLFW vendored under vendor/glfw/
├── src/
│   ├── root.zig                     # public API — re-exports from `backend` module
│   ├── common.zig                   # shared types: Event, KeyCode, WindowOptions, ActionId, NativeWindowHandle
│   ├── action_input.zig             # action-mapping layer (platform-agnostic)
│   ├── native_handle.zig            # per-backend native handle extraction
│   ├── backend/
│   │   ├── glfw.zig                 # v0 backend — single file; GLFW handles per-OS internally
│   │   └── native/                  # v1.x backend — file per OS
│   │       ├── linux.zig            # runtime-dispatches X11 vs Wayland (compiled together)
│   │       ├── linux_x11.zig        # imported only by linux.zig
│   │       ├── linux_wayland.zig    # imported only by linux.zig
│   │       ├── windows.zig
│   │       ├── macos.zig
│   │       └── android.zig
│   └── tests/                       # integration tests against the public API
└── vendor/
    └── glfw/                        # external lib as git submodule
                                     # compiled only when backend=glfw
```

`vendor/glfw/` is a **vendored dependency** of the adapter, not a sub-library of it. Structurally identical to how the Vulkan-stack adapter vendors VMA.

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
    ) orelse .glfw;

    // Pick the backend source file by (choice, target OS)
    const backend_root = switch (backend_choice) {
        .glfw => "src/backend/glfw.zig",
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

    // GLFW vendored source compiled only when backend=glfw
    if (backend_choice == .glfw) {
        backend_mod.addCSourceFiles(.{
            .files = &glfw_sources_for_target(target.result.os.tag),
            // ...
        });
    }
}
```

The compiler walks the import graph from the chosen backend root. Files for other OSes / other backends are **never referenced** → never parsed, type-checked, or codegen'd. Same as how SDL's CMake excludes `src/video/win32/` when building for Linux, but cleaner because Zig's `@import` is graph-resolved per build rather than preprocessor-gated.

### Per-target tree-shake guarantee

| Export target | Files compiled into the export | Files NOT touched |
| --- | --- | --- |
| `--target x86_64-linux-gnu -Dplatform_backend=native` | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (linux branch) + `backend/native/linux.zig` + `linux_x11.zig` + `linux_wayland.zig` | All Win32/macOS/Android backend files; all GLFW vendor source |
| `--target x86_64-windows-gnu -Dplatform_backend=native` | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (win32 branch) + `backend/native/windows.zig` | All Linux/macOS/Android backend files; all GLFW vendor source |
| `--target x86_64-linux-gnu -Dplatform_backend=glfw` | `root.zig` + `common.zig` + `action_input.zig` + `native_handle.zig` (glfw branch) + `backend/glfw.zig` + `vendor/glfw/` (Linux subset only) | All native backend files; GLFW's Windows/macOS sources (GLFW's own CMake gates them) |

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

These rules are what keep the GLFW → native migration cheap. Violating any of them turns the v1.x swap into a rewrite.

### Rule 1 — Design the API to the engine's needs, not GLFW's idioms

The public API in `src/root.zig` exposes **engine concepts**, not GLFW concepts. No `glfwSetKeyCallback`-style C function-pointer registration. No `GLFWwindowhint`. No GLFW handle types.

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

// BAD — would poison the API for the native backend
pub fn setKeyCallback(window: *Window, cb: fn (...) callconv(.c) void) void;
pub const WindowHint = enum { glfw_resizable, ... };
```

The native backend reads `Event` from a queue; the GLFW backend reads `Event` from a queue. Both populate `Event` from their own sources internally.

### Rule 2 — Expose native window handles; the renderer creates surfaces

The platform adapter has **no Vulkan dependency.** It exposes raw OS window handles as a typed `NativeWindowHandle` union; the renderer (which already depends on the Vulkan-stack adapter) is the one that calls `vk*Surface*KHR` to convert that handle into a `vk.SurfaceKHR`.

This keeps the two adapters genuinely independent: a headless server, a config-editor tool, or any non-rendering binary can pull in platform-stack without dragging vulkan-zig along.

Per-backend, the adapter implements `nativeHandle(window) → NativeWindowHandle`:

| Backend | What `nativeHandle()` returns |
| --- | --- |
| GLFW (Linux X11) | `.x11 { display = glfwGetX11Display(), window = glfwGetX11Window(w) }` |
| GLFW (Linux Wayland) | `.wayland { display = glfwGetWaylandDisplay(), surface = glfwGetWaylandWindow(w) }` |
| GLFW (Windows) | `.win32 { hinstance = GetModuleHandleW(null), hwnd = glfwGetWin32Window(w) }` |
| Native X11 | `.x11 { display, window }` directly from XCB/Xlib state |
| Native Wayland | `.wayland { display, surface }` directly from wl_* state |
| Native Win32 | `.win32 { hinstance, hwnd }` directly from CreateWindowExW state |
| Native Android | `.android { window = ANativeWindow* }` |
| Native macOS | `.cocoa { layer = CAMetalLayer* }` (deferred, not v1.0) |

The renderer side then has one function (in `zig-cpp-vulkan-stack-adapter`):

```zig
pub fn createSurface(
    instance: vk.Instance,
    handle: NativeWindowHandle,    // imported from platform-stack
) !vk.SurfaceKHR {
    return switch (handle) {
        .x11     => |h| vk.createXlibSurfaceKHR(instance, ...),
        .wayland => |h| vk.createWaylandSurfaceKHR(instance, ...),
        .win32   => |h| vk.createWin32SurfaceKHR(instance, ...),
        .android => |h| vk.createAndroidSurfaceKHR(instance, ...),
        .cocoa   => |h| vk.createMetalSurfaceEXT(instance, ...),
    };
}
```

Engine bridges in one line:

```zig
const surface = try vk_stack.createSurface(vk_instance, platform.nativeHandle(window));
```

The `NativeWindowHandle` type lives in **platform-stack** (it's the OS's concept of a window handle). Vulkan-stack imports the type — that's a one-way data dependency (no behavior, just structs of raw pointers). The reverse direction (platform-stack depending on vulkan-zig) is what we deliberately avoid.

Reference precedent: SDL exposes `SDL_GetWindowWMInfo` returning a per-platform native handle struct; the renderer (Vulkan, OpenGL, D3D11, Metal) consumes it. SDL itself has no Vulkan dep. Godot's `DisplayServer::window_get_native_handle` is the same pattern. Unreal's `IPlatformApplication` exposes window handles; `FVulkanRHI` consumes them.

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

### Rule 4 — Integration tests run against both backends

Because the API surface is stable, every integration test must work against `glfw` AND `native` backends. CI runs the test suite twice:

```yaml
# (sketch — actual CI lives elsewhere)
strategy:
  matrix:
    backend: [glfw, native]
    target: [x86_64-linux-gnu, x86_64-windows-gnu]
```

When the native backend lands, divergence is caught immediately, not at engine integration time. This is the insurance policy for the migration.

## Public API surface (v1.0)

### Window

```zig
pub const Window = opaque {};

pub const WindowOptions = struct {
    title: []const u8,
    size: Size = .{ .w = 1280, .h = 720 },
    position: ?Position = null,         // null → OS-default
    fullscreen: bool = false,
    resizable: bool = true,
    borderless: bool = false,
    vulkan_compatible: bool = true,
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

### Native window handle (for renderer surface creation)

```zig
pub const NativeWindowHandle = union(enum) {
    x11:     struct { display: *anyopaque, window: u64 },
    wayland: struct { display: *anyopaque, surface: *anyopaque },
    win32:   struct { hinstance: *anyopaque, hwnd: *anyopaque },
    android: struct { window: *anyopaque },
    cocoa:   struct { layer: *anyopaque },     // CAMetalLayer* — deferred post-v1.0
};

pub fn nativeHandle(window: *Window) NativeWindowHandle;
pub fn requiredVulkanInstanceExtensions() []const [*:0]const u8;
                                         // ↑ just C-string array — no Vulkan TYPES involved
```

`requiredVulkanInstanceExtensions()` returns the names of `VK_KHR_*_surface` extensions the renderer must enable on `vk.Instance` creation (e.g. `VK_KHR_xlib_surface` on X11). It returns C strings — no Vulkan type dependency.

`nativeHandle()` exposes raw OS window handles as a typed union. The renderer (`zig-cpp-vulkan-stack-adapter`) imports the `NativeWindowHandle` type and consumes it via its own `createSurface(instance, handle)` function.

**The platform adapter has zero Vulkan dependency.** A headless server, a config-editor tool, or any non-rendering binary can use this adapter without dragging vulkan-zig along.

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
| Renderer (Vulkan-stack adapter) | Engine calls `platform.nativeHandle(window)` and passes the result to `vk_stack.createSurface(instance, handle)`. Platform adapter has no Vulkan dep. `NativeWindowHandle` type is defined here and imported by vulkan-stack (one-way data dep) |

## Reference patterns

| Engine | Where | What to adapt |
| --- | --- | --- |
| **SDL3** ✅ best fit | `$REFS/godot/thirdparty/sdl/` (Godot's vendored copy) · upstream: <https://github.com/libsdl-org/SDL> · `src/video/{x11,wayland,cocoa,windows}/` · `src/events/SDL_events.c` | THE reference impl of "one stable API, many backends." Study the event-queue pattern + the per-platform `SDL_VideoDevice` vtable. Don't port C++; absorb the design |
| **Godot — DisplayServer** | `$REFS/godot/servers/display_server.h` + `platform/{linuxbsd,windows,macos,android}/display_server_*.cpp` | Abstract base + per-platform subclasses. Same idea, C++ machinery. Read the API surface; implement via Zig comptime dispatch instead of vtables |
| **Unreal — IPlatformApplication** | `$REFS/UnrealEngine/Engine/Source/Runtime/ApplicationCore/Public/GenericPlatform/` + per-platform under `Linux/`, `Windows/`, `Apple/`, `Android/` | Heavier than we want, but worth reading for the breadth of platform concerns it covers (drag-drop, IME, accessibility hooks) |
| **GLFW** | <https://github.com/glfw/glfw> · `src/{x11,wayland,cocoa,win32}_*.c` | Our v0 backend. Study `src/internal.h` for the `_GLFWplatform` vtable — that's the shape the v1.x native backend should match in spirit |
| **Hazel** | `$REFS/Hazel/Hazel/src/Platform/` | GLFW-coupled, no abstraction. **Anti-reference** — this is what NOT to do |

Adaptation rule per [`engine-references.md` § Legal](../engine-references.md): read, understand, reimplement in Zig. SDL is the closest license-compatible (zlib) reference but **do not copy verbatim** — design your API to engine needs (rule 1), use SDL only as pattern validation.

## Migration plan — GLFW v0 → native v1.x

### Pre-flight checks (before native backend work starts)

- All engine code compiles and runs against the GLFW backend
- All four design rules audited — no GLFW idioms in the public API
- Integration test suite passes against the GLFW backend
- Capability flag set is committed; engine code that branches on capabilities works

### Build the native backend incrementally

| Step | Scope | Verification |
| --- | --- | --- |
| 1 | `linux_x11.zig` only — no Wayland yet | Same integration tests pass against `-Dplatform_backend=native -Dlinux_backend=x11` |
| 2 | `linux_wayland.zig` + runtime dispatch in `linux.zig` | Tests pass against both X11 and Wayland in CI |
| 3 | `windows.zig` | Tests pass against Windows in CI |
| 4 | `android.zig` (the v1.x Android port lands here) | Tests pass against Android emulator |
| 5 | macOS deferred per `mission.md` | n/a v1.x |

Each step is its own atomic sub-repo commit. Engine repo doesn't touch this.

### Release the native backend as `libs/zig-cpp-platform-stack-adapter@v2.0`

- Engine bumps the dep version in `build.zig.zon` from `v1.x` (GLFW) to `v2.0` (native)
- One engine commit: `chore(deps): bump platform-adapter v1.x → v2.0 (native backend)`
- Roll-back is reverting the version pin; same engine source on either side

### Cleanup (optional v2.x)

- Remove `backend/glfw.zig` and `vendor/glfw/` from the sub-repo, OR keep them as an opt-in for users who prefer GLFW (e.g. for embedded distros with poor Wayland support)
- Decision deferred until v2.0 ships; user-feedback-driven

## Open decisions

- **macOS backend timing** — `mission.md` defers macOS post-v1.0. If a community user contributes a macOS backend earlier, integrate; otherwise wait
- **Android backend touchscreen events** — `.touch` event type vs treating touch as mouse — decide during Android port (Phase post-v1.0)
- **Multiple-window support** — v1.0 ships single primary window only; multi-window (editor + playtest in separate OS windows) deferred to Phase 12
- **Game controller force feedback** — defer to v1.x; expose capability flag now so we can light up later without ABI break

## Milestone

Phase 1 — adopt the Platform-stack adapter alongside the Vulkan-stack adapter. Verifications:

- Window opens, events received, key + mouse + gamepad input working end-to-end against the GLFW v0 backend
- Action bindings load from TOML; per-save rebindings persist
- Context stack: push `dialog` → gameplay's `attack_primary` is masked → pop → gameplay's binding is live again
- Synthetic injection: `injectAction(.jump, true, 1.0)` triggers the same code path as a real spacebar press; verified by integration test
- Axis-binding modifiers: deadzone + smooth + invert applied correctly on gamepad sticks
- Native handle exposure: `platform.nativeHandle(window)` returns a typed `NativeWindowHandle`; the renderer's `vk_stack.createSurface(instance, handle)` consumes it without the platform adapter linking vulkan-zig
- Build verification: `nm` on a Linux export shows no Win32/macOS/Android symbols, and no `vk*` symbols emitted from the platform adapter itself (only from vulkan-stack)

Native backend (v1.x) ships after v1.0 voxel-Daggerfall release per [`vision.md` § Shipping strategy](../vision.md). Same verifications must pass against `-Dplatform_backend=native` in CI before the v1.x → v2.0 sub-repo bump lands in the engine.
