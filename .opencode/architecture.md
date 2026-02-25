# Steklo Architecture Deep-Dive

## Binary Targets

### steklo (CLI) -- `steklo/src/main.rs`

Entry point: `main()` at line 248, calls `run()` then `Mux::shutdown()`.

CLI parser (clap) defined at `main.rs:22-52` with subcommands at `main.rs:88-149`:

| Command | File | Description |
|---------|------|-------------|
| `start` (hidden) | delegates to steklo-gui | Start GUI via exec() |
| `ai` | `steklo/src/ai_config/mod.rs` | Manage AI tool configs (ratatui TUI) |
| `config` | `steklo/src/config_cmd.rs` | Open/create `~/.config/steklo/steklo.lua` |
| `init` | `steklo/src/init.rs` | Shell integration setup (zsh only) |
| `update` | `steklo/src/update.rs` | Self-update from GitHub releases |
| `reset` | `steklo/src/reset.rs` | Remove shell integration & managed defaults |
| `cli` (hidden) | `steklo/src/cli/mod.rs` | Interact with mux server (pane/tab ops) |
| `set-working-directory` (hidden) | inline | Emit OSC 7 for CWD |
| `shell-completion` (hidden) | inline | Generate completions |

When invoked with no subcommand from a terminal, shows an interactive TUI menu (lines 310-511) with ASCII art logo and 5 choices navigable by arrow keys, j/k, or number keys.

GUI delegation (`main.rs:513`): resolves `steklo-gui` binary in same dir, canonical path parent, `/Applications/Steklo.app/Contents/MacOS/steklo-gui`, or `~/Applications/...`.

### steklo-gui (GUI) -- `steklo-gui/src/main.rs`

Entry point: `main()` at line 763. Bootstrap flow:
1. Designate main thread, set error callback
2. Parse CLI opts, ensure user config exists
3. Bootstrap environment (env-bootstrap crate)
4. Register Lua context setup functions
5. Initialize config system (triggers Lua config load)
6. Create Mux singleton
7. Resolve publish mode (single-instance on non-macOS)
8. Create `GuiFrontEnd`
9. Spawn async terminal GUI (`spawn_terminal_gui_async`)
10. On exit: `Mux::shutdown()`, `frontend::shutdown()`

Publish mechanism (lines 384-549): On macOS, always starts fresh. On Linux, tries Unix socket connection to existing instance.

## Core Module Architecture

### Mux (Multiplexer) -- `mux/src/lib.rs`

Central singleton (`Mux` struct, lines 104-118):
```
tabs:        RwLock<HashMap<TabId, Arc<Tab>>>
panes:       RwLock<HashMap<PaneId, Arc<dyn Pane>>>
windows:     RwLock<HashMap<WindowId, Window>>
domains:     RwLock<HashMap<DomainId, Arc<dyn Domain>>>
subscribers: RwLock<HashMap<usize, Box<dyn Fn(MuxNotification)>>>
clients:     RwLock<HashMap<ClientId, ClientInfo>>
```

Notification system (`MuxNotification` enum, lines 58-100):
`PaneOutput`, `PaneAdded`, `PaneRemoved`, `WindowCreated`, `WindowRemoved`, `Alert`, `Empty`, `AssignClipboard`, `SaveToDownloads`, `TabAddedToWindow`, `PaneFocused`

PTY reading pipeline (`lib.rs:281-373`):
1. `read_from_pane_pty()` -- blocking reads in dedicated thread
2. Data via `socketpair()` to `parse_buffered_data()`
3. `termwiz::escape::parser::Parser` parses escape sequences
4. Actions coalesced with configurable delay
5. `send_actions_to_mux()` applies to pane, notifies subscribers

### Tab -- `mux/src/tab.rs`

Uses `bintree::Tree<Arc<dyn Pane>, SplitDirectionAndSize>` for pane layout. Supports zoom, recency tracking, split operations.

### Domain -- `mux/src/domain.rs`

Async trait: `spawn()`, `split_pane()`, `attach()`, `detach()`. Implementations: `LocalDomain`, SSH domains, WSL domains, exec domains.

### Terminal -- `term/src/terminal.rs`

`Terminal` struct wrapping `TerminalState` which implements the full VT terminal state machine. Key sub-modules:
- `terminalstate/performer.rs` -- CSI/OSC action execution
- `terminalstate/keyboard.rs` -- keyboard encoding (xterm, kitty protocol)
- `terminalstate/mouse.rs` -- mouse protocol handling
- `terminalstate/image.rs` -- inline image protocols (sixel, iTerm2, kitty)
- `screen.rs` -- screen model with scrollback buffer

### Config -- `config/src/lib.rs`

- Lazy-loading color scheme registry (lines 178-262)
- Config path: `~/.config/steklo/steklo.lua` (line 477)
- File watching via `notify` crate with Lua config pipe
- Default template generation (lines 601-684)
- Atomic file writes via temp + rename (lines 525-599)

### Window -- `window/src/lib.rs`

Key traits:
- `WindowOps` (line 261): `show()`, `hide()`, `close()`, `set_cursor()`, `invalidate()`, `set_title()`, `toggle_fullscreen()`
- `WindowEvent` (line 168): `CloseRequested`, `Destroyed`, `Resized`, `NeedRepaint`, `FocusChanged`, `KeyEvent`, `MouseEvent`

macOS backend in `window/src/os/macos/` (7 files) using Cocoa/CoreGraphics/CoreText.

### TermWindow -- `steklo-gui/src/termwindow/mod.rs`

The 4045-line core struct managing the terminal window. Owns:
- Render state (OpenGL/WebGPU)
- Glyph cache and shape cache
- Tab bar state
- Input mapping
- Overlay management
- Clipboard, selection, mouse/key event handling

## Data Flow: Keystroke to Screen

1. macOS event -> `window/os/macos/window.rs` -> `WindowEvent::KeyEvent`
2. `TermWindow::key_event()` in `termwindow/keyevent.rs`
3. Key mapping lookup in `inputmap.rs`
4. If terminal input: encode via `term/src/input.rs` -> write to PTY
5. PTY output -> `mux/lib.rs::read_from_pane_pty()` -> parse escape sequences
6. `MuxNotification::PaneOutput` -> `TermWindow` subscriber
7. `TermWindow::invalidate()` -> schedule repaint
8. `render/paint.rs` -> `render/pane.rs` -> `render/screen_line.rs`
9. Glyph shaping (harfbuzz) -> rasterization (freetype) -> GPU texture upload
10. wgpu/Metal or glium/OpenGL draw calls
