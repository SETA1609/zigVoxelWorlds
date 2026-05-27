# C ABI Spec

> The stable `extern "C"` surface that mods, DLCs, and game scripts all use. Versioned. Once shipped, breaking changes require migration paths. Closes [`gaps.md`](../gaps.md) #13, #23. **Phase-0-blocking** — locks the contract before any code depends on it.

## Scope

A single C ABI is the load-bearing surface for:

- **Native-tier mods & DLCs** — `.so`/`.dll` plugins `dlopen`'d at runtime (only for content Ed25519-signed by the project publisher's key; per [`engine-vs-game.md`](../engine-vs-game.md) and [`specs/mod-manager.md`](mod-manager.md) § Security)
- **Game scripts** — `libgame.so` compiled per-project (per [`engine-vs-game.md`](../engine-vs-game.md) § 5)
- **WASM-sandboxed mods** — use a curated Host API (strict subset of the above C ABI) via WAMR; never raw `dlopen`

The stable C ABI (and generated `zvox_abi.h`) is the contract for native-tier content and game scripts. WASM mods are limited to the documented Host API surface. All must be versioned and stable for the lifetime of the engine.

## Design principles

1. **Pure C at the boundary** — no C++ types, no templates, no `std::*`, no exceptions, no RTTI. Every boundary function `noexcept` (enforced per [`cpp-style.md`](../cpp-style.md))
2. **Opaque handles only** — Vulkan / Jolt / internal types never cross. Mods see `uint64_t` `Handle`s only (per [`specs/core-types.md`](core-types.md))
3. **Versioned** — every release ships an ABI version. Mods declare compat range in `mod.toml`. Engine refuses incompatible mods at load
4. **Explicit ownership** — every pointer that crosses has documented allocator + freer + lifetime (per [`cpp-style.md`](../cpp-style.md))
5. **Stable layouts** — struct layouts at the boundary are `#pragma pack`'d or use only fixed-width integers. No platform-dependent sizing

## ABI surface — skeleton

The actual headers live in `src/core/abi.zig` (Zig source) generated as `zvox_abi.h` (C header) at build time. This skeleton sketches the structure.

### Versioning

```c
#define ZVOX_ABI_VERSION_MAJOR  0
#define ZVOX_ABI_VERSION_MINOR  1
#define ZVOX_ABI_VERSION_PATCH  0

// Returned by zvox_engine_init() — mods compare against their compiled-against version
typedef struct {
    uint16_t major;
    uint16_t minor;
    uint16_t patch;
    uint16_t flags;     // reserved
} ZvoxAbiVersion;

ZvoxAbiVersion zvox_abi_version(void);
```

### Mod entry / lifecycle

Every mod's plugin exposes these four symbols (Zig codegen produces stubs from `mod.toml`):

```c
// Called once at mod load. Mod registers its content + listeners here.
// Returns 0 on success, non-zero on error (engine logs + refuses load).
int32_t zvox_mod_init(ZvoxEngineApi* api, ZvoxModContext* ctx);

// Called for each engine init level. Mods slot their work into the right level.
// (Core / Servers / Scene / Editor — per Godot pattern in engine-references.md)
void zvox_mod_register(ZvoxEngineApi* api, ZvoxModContext* ctx, uint32_t level);

// Called once per tick. Most mods don't need this (use events instead).
void zvox_mod_tick(ZvoxEngineApi* api, ZvoxModContext* ctx, double dt);

// Called at mod unload — cleanup.
void zvox_mod_shutdown(ZvoxEngineApi* api, ZvoxModContext* ctx);
```

### `ZvoxEngineApi` — the function table

A struct of function pointers that the engine fills in at mod load. Mods call functions through this table — never through direct linkage to engine symbols.

```c
typedef struct {
    // Version of this API struct (for forward-compatibility — see § Compatibility)
    ZvoxAbiVersion version;
    
    // ===== Logging =====
    void (*log)(ZvoxLogLevel level, const char* category, const char* msg);
    
    // ===== Voxel server =====
    uint64_t (*voxel_get)(uint64_t world, int32_t x, int32_t y, int32_t z);
    int32_t  (*voxel_set)(uint64_t world, int32_t x, int32_t y, int32_t z, uint32_t voxel_type_id);
    int32_t  (*voxel_raycast)(uint64_t world, const ZvoxRay* ray, ZvoxRaycastHit* out);
    
    // ===== Scene / entities =====
    uint64_t (*entity_spawn)(uint64_t scene, const char* template_name, const ZvoxTransform* xform);
    int32_t  (*entity_destroy)(uint64_t entity);
    int32_t  (*entity_set_component)(uint64_t entity, const char* comp_name, const void* data, size_t size);
    int32_t  (*entity_get_component)(uint64_t entity, const char* comp_name, void* out, size_t* size);
    
    // ===== Events =====
    int32_t  (*event_subscribe)(const char* event_id, void (*cb)(const void* payload, void* user), void* user);
    int32_t  (*event_unsubscribe)(int32_t subscription_id);
    int32_t  (*event_emit)(const char* event_id, const void* payload, size_t size);
    
    // ===== Content registration =====
    uint32_t (*register_voxel_type)(const ZvoxVoxelTypeDesc* desc);
    uint32_t (*register_item)(const ZvoxItemDesc* desc);
    uint32_t (*register_recipe)(const ZvoxRecipeDesc* desc);
    
    // ===== Render server (limited surface — most rendering is engine-managed) =====
    uint64_t (*mesh_load)(const char* guid);
    void     (*mesh_unload)(uint64_t mesh);
    int32_t  (*draw_mesh)(uint64_t mesh, const ZvoxTransform* xform, uint64_t material);
    
    // ===== Audio =====
    uint64_t (*audio_play)(const char* sound_guid, const ZvoxVec3* pos, float volume);
    void     (*audio_stop)(uint64_t handle);
    
    // ... (etc — full surface lives in zvox_abi.h)
} ZvoxEngineApi;
```

### `ZvoxModContext` — per-mod state

```c
typedef struct {
    const char* mod_id;             // from mod.toml
    const char* mod_version;
    void*       user_data;          // mod can stash anything here; engine doesn't touch
    
    // Paths for the mod to find its own resources
    const char* data_path;          // <project>/mods/<id>/data
    const char* asset_path;         // <project>/mods/<id>/assets
} ZvoxModContext;
```

### Common value types

All boundary types are fixed-width + plain-old-data:

```c
typedef struct { float x, y, z; }       ZvoxVec3;
typedef struct { float x, y, z, w; }    ZvoxQuat;
typedef struct {
    ZvoxVec3 position;
    ZvoxQuat rotation;
    ZvoxVec3 scale;
} ZvoxTransform;

typedef struct { ZvoxVec3 origin; ZvoxVec3 direction; float max_distance; } ZvoxRay;
typedef struct {
    int8_t   hit;
    uint64_t entity;
    int32_t  voxel_x, voxel_y, voxel_z;
    float    distance;
    ZvoxVec3 point;
    ZvoxVec3 normal;
} ZvoxRaycastHit;
```

## Versioning policy

**Semver, but applied to the ABI not the engine:**

- **Major bump** (`X.0.0`): breaks existing mods. Old mods refused to load. Migration path provided
- **Minor bump** (`0.X.0`): additive only — new functions, new event types, new struct fields appended at the end. Old mods continue to work
- **Patch bump** (`0.0.X`): bug fixes, no signature changes

### What counts as a major change

Any of:

- Removing a function from `ZvoxEngineApi`
- Changing a function's signature (parameter types, return type, argument count)
- Reordering struct fields
- Changing the size of a struct field
- Changing the semantics of an existing function (what it does, what it returns)

### What counts as a minor change

- Adding a new function at the end of `ZvoxEngineApi` (function pointer table is append-only)
- Adding a new field at the end of a struct (with old mods still seeing the unchanged prefix)
- Adding new event types, new content-descriptor types
- Adding new component types

### What counts as a patch change

- Bug fixes that don't alter behavior contracts
- Performance improvements
- Documentation changes

### How mods declare compat

In `mod.toml` (per [`specs/data-schemas.md`](data-schemas.md)):

```toml
[mod.compat]
abi_version = "^0.2"        # compatible with any 0.2.x or 0.3.x ... up to 0.x; not 1.x
```

The engine compares the mod's `abi_version` range against the engine's current `ZVOX_ABI_VERSION_*`. If outside the range → load refused with a clear error.

### Pre-1.0 special rule

While `ZVOX_ABI_VERSION_MAJOR == 0`, breaking changes are allowed on minor bumps (per semver convention for 0.x). After v1.0 ships, minor bumps are strictly additive.

## Forward compatibility — struct extension

Old mods built against API v0.2 must still work when the engine ships v0.3 (same major). Two mechanisms:

1. **Append-only function table.** Old mods see indices 0..N-1; new functions live at indices N..N+M and old mods simply don't call them.
2. **Append-only struct layout.** Each struct has a `size: uint32_t` first field; engine reads only as many fields as the mod was built against. Example:

```c
typedef struct {
    uint32_t struct_size;     // = sizeof(ZvoxVoxelTypeDesc) at compile time
    uint32_t voxel_type_id;
    const char* name;
    uint32_t flags;
    float    hardness;
    // ... v0.2 fields ...
    // [v0.3 adds:]
    uint8_t  light_emission;
} ZvoxVoxelTypeDesc;
```

When the engine receives a `ZvoxVoxelTypeDesc*`, it reads `struct_size` first and ignores fields beyond what was sent. New fields default to zero.

## What's NOT in the ABI

Things that look tempting but are intentionally engine-private:

- Direct Vulkan handles / shader access — mods that want custom rendering go through `RenderServer` or `draw_mesh`
- Direct file system access — mods read through the asset DB (GUID-based), not raw paths. Sandboxes against path traversal
- Direct memory allocation in engine pools — mods use their own allocator
- Async/threading primitives — mods don't spawn threads on engine state; they post to event queues or wait their tick

Per [`engine-vs-game.md`](../engine-vs-game.md) — these are the **proprietary IP boundary**. The engine keeps internals private; mods get the data + event surface.

## C ABI evolution log

A `CHANGELOG-ABI.md` ships alongside the engine, tracking every ABI change. Sample format:

```markdown
## 0.3.0 — 2026-08-XX

### Added (minor — backward compatible)

- `entity_get_aabb()` — query an entity's world-space AABB
- `ZvoxVoxelTypeDesc.light_emission` — new field, appended

### Patched

- `voxel_raycast()` — fixed off-by-one on chunk boundaries

## 0.2.0 — 2026-06-XX

[Initial ABI release]
```

Mods can target a range; modders can plan upgrades.

## Implementation note for Zig

Zig generates the C header automatically via `zig build header-export`. The Zig source is the source of truth; the `.h` is generated. Boundary functions live in `src/core/abi.zig`; they wrap engine internals with the rules above.

```zig
// In src/core/abi.zig
pub export fn voxel_get(world: u64, x: i32, y: i32, z: i32) callconv(.C) u64 {
    const w = handle_table.get(WorldHandle, world) catch return 0;
    const result = w.voxel_at(.{ .x = x, .y = y, .z = z }) catch return 0;
    return result.id;
}
```

`callconv(.C)` enforces the C calling convention. `catch return 0` ensures no Zig error escapes the boundary (matches `noexcept` rule for C++ adapters).

## Closes

- [`gaps.md`](../gaps.md) #13 — C ABI surface design
- [`gaps.md`](../gaps.md) #23 — ABI versioning policy

Sources:

- Semver: <https://semver.org/spec/v2.0.0.html>
- C ABI struct-versioning patterns: Vulkan's `sType` + `pNext` chain pattern, Wayland protocol versioning
- The "function table" pattern: Wayland client libs, libavcodec
