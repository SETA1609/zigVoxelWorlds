# `src/editor/code_editor/`

> Embedded Neovim host for editing project scripts (Zig + C++) without leaving the editor. ImGui fallback when `nvim` is not on the user's PATH.

## Architecture

- Spawn `nvim --embed` as a child process via `platform/`'s `std.process.Child` helpers
- Bidirectional msgpack-RPC over the child's stdin/stdout
- Render Neovim's redraw notifications into an ImGui surface (text grid, cursor, mode indicators)
- Forward ImGui keyboard input back to nvim as `nvim_input` calls

The editor never re-implements an editor — Neovim handles syntax, LSP, completion, plugins. The host just bridges its redraw protocol to ImGui.

## Fallback

If `nvim` is missing or msgpack-RPC handshake fails: a minimal ImGui text editor with no syntax highlighting. Acceptable for "view this file" but not for serious editing — surfaces a "install Neovim for full editing" hint.

## References

- Neovim API: <https://neovim.io/doc/user/api.html>
- UI redraw protocol: <https://neovim.io/doc/user/ui.html>
- Reference clients: `goneovim` (Go), `neovide` (Rust), `firenvim` (TypeScript)

## Out of scope

Reformatting, building, linting — those are the script-builder's job (see [`../script_builder/`](../script_builder/README.md)).

## User-customisable

Honors the user's existing `~/.config/nvim/init.lua` — same Neovim, same plugins, same keybinds. No project-specific overrides unless `<project>/.zvoxrealms/nvim/` exists.
