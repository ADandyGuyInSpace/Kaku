# Crates Reference

## Workspace Members (Cargo.toml)

Workspace members explicitly listed:
`crates/bidi`, `deps/cairo`, `steklo`, `crates/wezterm-blob-leases`, `crates/wezterm-cell`, `crates/wezterm-escape-parser`, `crates/wezterm-dynamic`, `steklo-gui`, `crates/wezterm-open-url`, `crates/wezterm-ssh`, `crates/wezterm-surface`, `crates/wezterm-uds`

## Internal Crates (`crates/`)

| Crate | Path | Purpose |
|-------|------|---------|
| `async_ossl` | `crates/async_ossl` | Async OpenSSL integration |
| `base91` | `crates/base91` | Base91 encoding/decoding |
| `bidi` | `crates/bidi` | Unicode Bidirectional text algorithm |
| `bintree` | `crates/bintree` | Binary tree for pane layout splitting |
| `codec` | `crates/codec` | Mux server wire protocol codec |
| `color-types` | `crates/color-types` | Color type definitions (SRGBA, etc.) |
| `env-bootstrap` | `crates/env-bootstrap` | Environment bootstrapping (logging, Lua setup) |
| `filedescriptor` | `crates/filedescriptor` | Cross-platform file descriptor abstraction |
| `frecency` | `crates/frecency` | Frecency-based ranking (frequency + recency) |
| `lfucache` | `crates/lfucache` | Least-Frequently-Used cache |
| `luahelper` | `crates/luahelper` | Lua <-> Rust value conversion |
| `procinfo` | `crates/procinfo` | Process info queries (foreground process, CWD) |
| `promise` | `crates/promise` | Async promise/future implementation |
| `pty` | `crates/pty` | Cross-platform PTY abstraction (`portable-pty`) |
| `rangeset` | `crates/rangeset` | Range set data structure |
| `ratelim` | `crates/ratelim` | Rate limiting utilities |
| `tabout` | `crates/tabout` | Table output formatting |
| `umask` | `crates/umask` | Umask save/restore |
| `vtparse` | `crates/vtparse` | Low-level VT escape sequence parser |
| `wezterm-blob-leases` | `crates/wezterm-blob-leases` | Blob lease management for images |
| `wezterm-cell` | `crates/wezterm-cell` | Terminal cell types (Cell, CellAttributes) |
| `wezterm-char-props` | `crates/wezterm-char-props` | Unicode character properties |
| `wezterm-client` | `crates/wezterm-client` | Client for mux server connection |
| `wezterm-dynamic` | `crates/wezterm-dynamic` | Dynamic value type system (FromDynamic/ToDynamic) |
| `wezterm-escape-parser` | `crates/wezterm-escape-parser` | High-level semantic escape sequence parser |
| `wezterm-font` | `crates/wezterm-font` | Font loading, HarfBuzz shaping, FreeType rasterization |
| `wezterm-gui-subcommands` | `crates/wezterm-gui-subcommands` | Shared GUI subcommand defs (StartCommand) |
| `wezterm-input-types` | `crates/wezterm-input-types` | Input event types (KeyCode, Modifiers, MouseEvent) |
| `wezterm-mux-server-impl` | `crates/wezterm-mux-server-impl` | Mux server implementation (local listener) |
| `wezterm-open-url` | `crates/wezterm-open-url` | Cross-platform URL opening |
| `wezterm-ssh` | `crates/wezterm-ssh` | SSH client implementation |
| `wezterm-surface` | `crates/wezterm-surface` | Terminal surface/line model (SequenceNo, Line, Change) |
| `wezterm-toast-notification` | `crates/wezterm-toast-notification` | Cross-platform toast notifications |
| `wezterm-uds` | `crates/wezterm-uds` | Unix domain socket utilities |
| `wezterm-version` | `crates/wezterm-version` | Version string management |

## Top-Level Crates

| Crate | Path | Purpose |
|-------|------|---------|
| `config` | `config/` | Configuration system (Lua-based, 24 source files) |
| `mux` | `mux/` | Multiplexer core (tabs, panes, PTY, domains, 17 files) |
| `wezterm-term` | `term/` | Terminal emulation (state machine, screen, 19 files) |
| `termwiz` | `termwiz/` | Terminal widget library (caps, input, surface, 21 files) |
| `window` | `window/` | Windowing abstraction (macOS Cocoa, 18 files) |

## Lua API Crates (`lua-api-crates/`)

| Crate | Purpose |
|-------|---------|
| `battery` | `wezterm.battery_info` |
| `color-funcs` | Color manipulation + scheme loaders (iTerm2, base16, gogh, sexy) |
| `filesystem` | File system operations (read_dir, glob) |
| `logging` | `wezterm.log_info`, `wezterm.log_error`, etc. |
| `mux` | Mux Lua bindings: MuxDomain, MuxPane, MuxTab, MuxWindow |
| `plugin` | Plugin system |
| `procinfo-funcs` | Process info for Lua |
| `serde-funcs` | JSON/TOML serialization |
| `share-data` | Shared data between Lua callbacks |
| `spawn-funcs` | Process spawning from Lua |
| `ssh-funcs` | SSH connection config |
| `termwiz-funcs` | Termwiz surface/cell functions |
| `time-funcs` | Time/date functions |
| `url-funcs` | URL parsing/manipulation |
| `window-funcs` | Window/GUI functions (`wezterm.gui.get_appearance()`) |

## Vendored Dependencies (`deps/`)

| Dep | Path | Purpose |
|-----|------|---------|
| `freetype` | `deps/freetype/` | Font rasterization (FreeType2 + libpng + zlib submodules) |
| `harfbuzz` | `deps/harfbuzz/` | Text shaping engine (harfbuzz submodule) |
| `cairo-sys-rs` | `deps/cairo/` | 2D graphics FFI (patched in workspace Cargo.toml) |
| `fontconfig` | `deps/fontconfig/` | Font discovery (Linux, uses pkg-config) |

## Key External Dependencies

- `wgpu` 25.0.2 -- GPU abstraction (Metal backend only on macOS)
- `glium` 0.35 -- OpenGL fallback renderer
- `mlua` 0.9 -- Lua 5.4 bindings
- `ratatui` 0.29 -- TUI framework (for AI config UI)
- `clap` 4.0 -- CLI argument parsing
- `git2` 0.20 -- Git operations (vendored libgit2 + OpenSSL)
- `image` 0.25 -- Image handling (png, jpeg, gif, webp only)
- `reqwest` 0.12 -- HTTP client
- `tokio` 1.43 -- Async runtime
