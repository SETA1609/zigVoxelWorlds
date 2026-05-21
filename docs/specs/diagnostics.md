# Diagnostics + Crash Dump Pipeline Spec

> Crash dumps, symbolication, player upload, save recovery. [`specs/editor.md`](editor.md) covers the editor's playtest log (libghostty); this spec covers *shipped game* diagnostics. Gap: [`gaps.md` § 2.2.E + § 3 #37](../gaps.md). Reference patterns: [`gap-references.md` § 2.2.E](../gap-references.md).

## Scope

Three players are involved in a crash:

1. **The player whose game crashed** — sees a recoverable failure path, can submit a report
2. **The developer (you)** — gets a symbolicated stack trace + game state context
3. **The community (other players)** — benefit from issue triage based on aggregated reports

This spec is what shipped games do; the editor's `libghostty` log panel ([`specs/editor.md`](editor.md)) is a separate concern.

## Crash signal handling

Out-of-process crash reporter pattern (Unreal Crash Report Client model):

- Engine spawns a tiny watchdog process at game startup via `std.process.Child`
- Watchdog opens a pipe to the game process
- On game-process signal (SIGSEGV, SIGABRT, SIGFPE, etc.) → game writes a minidump + state snapshot to a file → watchdog detects death → watchdog presents the crash UI

Why out-of-process: in-process handlers are fragile (signal context corruption, memory state unreliable). The watchdog runs in clean memory.

## Crash dump contents

| Section | Content |
| --- | --- |
| Header | Engine version, build hash, OS + version, GPU info, CPU info, RAM |
| Stack trace | Native stack (with addresses; symbols resolved later) |
| Active modules | List of loaded modules + versions |
| Active mods | List of enabled mods + versions |
| Game state snapshot | Current scene, player position, save slot reference |
| Recent log | Last N lines from the engine log |
| Recent input | Last few seconds of player input (for repro) |
| User note | Optional text from the crash dialog |

Output: `<save_dir>/crashes/crash_<timestamp>.dmp` + `.json` metadata.

## Symbolication

Stack-trace addresses → file:line names. Requires symbol files (`.pdb` on Windows, debug info on Linux).

CI builds upload symbols to a server (Sentry, BugSplat, or self-hosted) keyed by build hash. The crash report client uploads the dump; server symbolicates server-side.

For dev builds (with debug symbols in the binary), symbolication happens locally.

## Player-facing crash dialog (watchdog UI)

```text
┌────────────────────────────────────────────────┐
│  The game has crashed                          │
│                                                │
│  We're sorry. zVoxRealms hit an unexpected     │
│  error. The good news:                         │
│                                                │
│  • Your last autosave is intact                │
│  • Submitting a report helps us fix this       │
│                                                │
│  Crash ID: crash_2026-05-21_1432_abc123        │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │ What were you doing? (optional)          │  │
│  │ [text area]                              │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  [ ] Include game-state snapshot               │
│  [ ] Include recent log                        │
│                                                │
│  [ Submit Report ]  [ Recover Save ]  [ Quit ] │
└────────────────────────────────────────────────┘
```

## Save recovery

On crash detection, before the dialog:

- Check timestamp of last autosave vs crash time
- If autosave < 5 minutes old → "Recover Save" button restores it
- If older → suggest manual save selection

## Privacy / consent

- Crash reports are **opt-in** by default per [`gaps.md` § 3 #36](../gaps.md) (GDPR-friendly default)
- Settings have a "submit crash reports automatically" toggle
- Game-state + log inclusion are separate toggles (some users won't share game state)
- Reports include no PII unless the player wrote text in the dialog
- Anonymous machine fingerprint (hashed) only — never IP, email, name

## In-game console (cheats / debug)

Backtick `~` opens a developer console (only in non-Steam builds, or behind a "developer mode" toggle):

- Type commands ("teleport 0 64 0", "give iron_sword", "spawn_npc bear")
- Output stream via [`specs/events.md`](events.md) → console listener
- History + autocomplete
- Per-game commands registered via the mod ABI

## Log levels + categories

Standard log categories per system:

- `core`, `render`, `voxel`, `physics`, `audio`, `net`, `ui`, `modding`, `scene`, `gameplay`, `editor`

Per-category log levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal`.

Logs route through `src/core/log_sink.zig` (per [`tech-stack.md` § Observability](../tech-stack.md#observability)) — stdout + log file + (editor only) libghostty pane + (dedicated server) OTLP.

## Borrowed patterns

[`gap-references.md` § 2.2.E](../gap-references.md):

- Godot `core/os/crash_handler*.cpp` + `editor/editor_crash_handler.cpp` — signal handler + crash log writer
- Unreal `CrashReportClient/` — out-of-process pattern, separate program for upload UI

## Open decisions

- Crash report backend — Sentry-compatible self-hosted vs commercial vs simple S3 bucket
- Symbol storage — separate per-platform vs unified
- Whether the watchdog ships in development builds too (probably yes, helpful)
- Log file rotation — by-size, by-day, by-session

## Milestone

Phase 12 (editor) + Phase 13 (exported game). Crash a dev build via a deliberate `unreachable` → watchdog shows the crash dialog → submit a report → see it appear (with symbolicated stack) in the crash-tracking dashboard. Recover the last autosave from the same dialog.
