# C++ Style & Conventions

> What conventions our C++ code follows, what deviations we accept from the baseline, and what rules to enforce when reviewing C++ in this repo. Used by humans, AI reviewers, and tooling.

## Baseline

**[Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)** — the entire guide, except where this document explicitly says otherwise.

We picked Google because:

- It's well-known, well-documented, and most reviewers (human or AI) know it
- It encodes years of large-codebase wisdom on naming, headers, ownership, and readability
- It comes with first-class tooling: `clang-format --style=Google`, `clang-tidy` Google checks, `cpplint`

## Where we deviate

C++ in zVoxRealms is almost entirely **C++ adapter code** wrapping vendor libraries (Jolt, VMA, ImGui, glslang, KTX, FlatBuffers, Tracy, libghostty, GameNetworkingSockets, Steamworks). Wrapped libs make assumptions Google's guide rejects. Where the vendor lib's design demands it, we deviate:

### 1. Exceptions: allowed inside C++ adapter code; forbidden across the C ABI

Google says "do not use C++ exceptions." We do, because Jolt, glslang, KTX, FlatBuffers, libghostty and most C++ libs throw. The rule:

- **Inside a C++ adapter** (`adapter.cpp`): exceptions are allowed because the wrapped library uses them
- **At the `extern "C"` boundary**: every exception **must** be caught and translated to a return code or error enum. An exception escaping through the C ABI is undefined behavior. Pattern:

  ```cpp
  extern "C" int adapter_create_world(WorldHandle* out) noexcept {
    try {
      auto* world = new JoltWorld();
      *out = reinterpret_cast<WorldHandle>(world);
      return ADAPTER_OK;
    } catch (const std::exception& e) {
      adapter_set_last_error(e.what());
      return ADAPTER_ERROR;
    } catch (...) {
      adapter_set_last_error("unknown C++ exception");
      return ADAPTER_ERROR;
    }
  }
  ```

- **Every `extern "C"` function** must be marked `noexcept` so the compiler enforces this at the boundary
- Zig consumers see a stable error code, never a C++ exception

### 2. RTTI: enabled where vendor libs need it; disabled where they don't

Google says no RTTI. Jolt uses RTTI internally. The adapter `build.zig` for each wrapped library decides:

- `adapters/jolt/build.zig` — RTTI on (`-frtti`)
- `adapters/imgui/build.zig` — RTTI off (`-fno-rtti`)
- Default for new adapters: off, enable only if the wrapped lib requires it

### 3. C++ standard: C++23

Google tracks the language version most of Google uses (currently C++20 with cautious C++23 adoption). We pin **C++23** explicitly (`-std=c++23`) because Zig ships a Clang frontend that supports it and we want modern facilities (`<expected>`, `std::print`, modules-light if we want them later).

### 4. Naming exception: `extern "C"` boundary uses `snake_case`

Google says PascalCase for types and functions in C++. For the C ABI boundary functions specifically, use **`snake_case` prefixed by the adapter name** to mirror what Zig consumers see and what the library is doing at the C layer:

```cpp
// Wrong (Google C++ default — looks weird across the C boundary):
extern "C" WorldHandle JoltCreateWorld() noexcept;

// Right (matches C convention + matches Zig usage on the other side):
extern "C" jolt_world_handle_t jolt_create_world() noexcept;
```

Inside C++ implementation files (not at the C ABI), follow Google: `PascalCase` for types, `snake_case_` for member variables, etc.

### 5. Smart pointers: yes; raw `new`/`delete` only inside adapters

Google's guidance applies: prefer `std::unique_ptr` for ownership transfer; `std::shared_ptr` only when shared ownership is real (rarely). Inside adapter C++ code, raw `new` is acceptable at the C ABI boundary to hand a raw pointer back as an opaque handle — but the C++ side that destroys it must use `delete`, matching the boundary.

### 6. Includes order: per Google, with one addition

Google's order:

1. Related header
2. C system headers
3. C++ standard library
4. Other libraries' headers
5. Your project's headers

Our addition: **wrapped vendor library headers come immediately before your project's headers** (group 4b), so the adapter's `#include "Jolt/Jolt.h"` is visually separate from generic third-party headers like `<spdlog/...>`.

### 7. Forward declarations

Google's stance has softened on this; ours is: **forward-declare aggressively** in headers, include the full type only in `.cpp` files. Adapters frequently have wrapped-library types that are large templates (Jolt's PBD solver, ImGui's contexts) — forward-declaring keeps adapter headers light and the Zig consumer's `cImport` fast.

### 8. Comments: WHY not WHAT

Repo-wide rule from [`AGENTS.md`](../AGENTS.md) / `CLAUDE.md` philosophy: **default to no comments**. Only add a comment when the *why* is non-obvious — a hidden constraint, a subtle invariant, a workaround for a vendor-lib bug, behavior that would surprise a reader.

Do not write:

- "// Loop over bodies" — the code says that
- "// Increment counter" — the code says that
- "// Added by Sebastian on 2026-05" — git blame says that

Do write:

- `// Jolt asserts on Step() if dt is zero — clamp to FLT_EPSILON`
- `// MUST be called before BroadPhase::Update; see Jolt issue #743`

## Adapter-specific rules (over and above Google + deviations above)

The C ABI boundary is load-bearing for mod compatibility ([`engine-vs-game.md`](engine-vs-game.md), [`licensing.md`](licensing.md)). Treat it as a versioned contract.

1. **Every `extern "C"` function is `noexcept`.** No exception escapes. Compiler-enforced.
2. **Opaque handles, not C++ pointers.** Across the C boundary, types are `void*` or typedef'd `uintptr_t`. C++ pointers (`JoltWorld*`) never appear in the C header.
3. **No C++ types in the C header.** No `std::string`, no `std::vector`, no templates. Pass length-prefixed buffers (`const char* data, size_t len`) or out-pointers.
4. **Ownership is explicit per function.** Document in the header comment: who allocates, who frees, what lifetime the pointer has. `// Caller frees with adapter_destroy_world(handle).`
5. **No global state.** Adapter context lives in the handle. Multiple `World` instances must be possible simultaneously.
6. **ABI is versioned.** When the C ABI changes, bump `ADAPTER_ABI_VERSION` in the adapter's header. The engine + mod loader check this at load time.
7. **Thread safety is explicit.** Document which functions are safe to call from multiple threads. Default: not thread-safe; engine code calls from one thread or wraps in a mutex.

## Tooling

A `.clang-format` lives at the root of each C++-bearing repo (zVoxRealms root + each adapter sub-repo). Baseline: `Google` preset with C++23 standard.

To format C++ before committing:

```bash
clang-format -i $(find src/cpp src/c -name "*.cpp" -o -name "*.c" -o -name "*.h" -o -name "*.hpp")
```

Recommended `clang-tidy` checks: `google-*`, `cppcoreguidelines-*` (selectively — they conflict with our exception/RTTI deviations), `modernize-*`, `readability-*`, `performance-*`, `bugprone-*`.

A `.clang-tidy` file with the curated checks should ship with each adapter sub-repo.

## For AI reviewers / LLM code review

When asked to review C++ in this repo, apply these rules in order:

1. **First, check the C ABI boundary.** Every `extern "C"` function must be `noexcept` and catch all exceptions. This is the most common bug class in adapter code.
2. **Check ownership** — every pointer that crosses the C boundary must have documented allocator + freer + lifetime in the header.
3. **Check the adapter doesn't leak C++ types into the C header** (no `std::*`, no templates, no C++ classes as types).
4. **Apply Google C++ Style Guide** for everything else: naming (per § 4 deviations), header order (per § 6), comments (per § 8), smart pointers (per § 5).
5. **Flag inconsistencies with this document.** If a rule here contradicts Google, this document wins.

When reviewing a new adapter for the first time, also check:

- `.clang-format` exists and uses `BasedOnStyle: Google` + `Standard: c++23`
- `LICENSE` file exists (MIT default; see [`licensing.md`](licensing.md) § Adapter sub-repos)
- The C ABI header documents ownership semantics per function
- The `build.zig` decides RTTI/exceptions per the wrapped lib's needs

## Sources

- Google C++ Style Guide: <https://google.github.io/styleguide/cppguide.html>
- C++ Core Guidelines (selective): <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>
- `clang-format` style options: <https://clang.llvm.org/docs/ClangFormatStyleOptions.html>
- `clang-tidy` checks: <https://clang.llvm.org/extra/clang-tidy/checks/list.html>
- Boundary safety pattern reference: SEI CERT C++ ERR62-CPP (catch exceptions before crossing the C boundary): <https://wiki.sei.cmu.edu/confluence/x/Hh4xBQ>
