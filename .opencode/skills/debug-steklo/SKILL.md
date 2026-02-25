---
name: debug-steklo
description: Debug Steklo terminal issues including rendering, shell integration, and AI assistant
---

## Debugging Steklo

### Enable Debug Logging

```bash
RUST_LOG=debug make dev                    # Full debug logging
RUST_LOG=config=debug make dev             # Config loading only
RUST_LOG=mux=debug make dev                # Mux/PTY operations only
RUST_LOG=wezterm_font=debug make dev       # Font loading/shaping
STEKLO_STRICT_CONFIG=1 make dev              # Strict Lua config validation
```

### Debug GPU Rendering

- WebGPU (Metal) is primary, OpenGL is fallback
- Test fallback: `make test-webgpu-fallback` or `./scripts/test_webgpu_fallback.sh --strict`
- Frontend selection in `config/src/frontend.rs`
- WebGPU state in `steklo-gui/src/termwindow/webgpu.rs`
- Render state in `steklo-gui/src/renderstate.rs`

### Debug Shell Integration

- Generated init: `~/.config/steklo/zsh/steklo.zsh`
- Source line in `~/.zshrc`: `source "$HOME/.config/steklo/zsh/steklo.zsh"`
- Plugin dir: `~/.config/steklo/zsh/plugins/`
- State file: `~/.config/steklo/state.json` (config version tracking)
- Reset everything: `steklo reset`
- Reset first-run: `./scripts/reset_first_run.sh`

### Debug AI Assistant

- Config: `~/.config/steklo/assistant.toml`
- Job files: `~/.config/steklo/ai_jobs/` (request/response IPC)
- Lua handler in `assets/macos/Steklo.app/Contents/Resources/steklo.lua` (search for `user-var-changed`)
- Shell hooks: `_steklo_ai_preexec` and `_steklo_ai_precmd` in generated steklo.zsh
- User vars: `STEKLO_AI_CMD` (command text), `STEKLO_AI_EXIT_CODE` (exit code)

### Debug Config Loading

- User config: `~/.config/steklo/steklo.lua`
- Bundled defaults: `Steklo.app/Contents/Resources/steklo.lua`
- Config system: `config/src/lib.rs` (lazy schemes at line 178, path at line 477)
- Lua evaluation: `config/src/lua.rs`

### Key Source Locations for Common Issues

| Issue | Look At |
|-------|---------|
| Keyboard input | `steklo-gui/src/termwindow/keyevent.rs`, `steklo-gui/src/inputmap.rs` |
| Mouse handling | `steklo-gui/src/termwindow/mouseevent.rs` |
| Tab bar | `steklo-gui/src/tabbar.rs`, `steklo-gui/src/termwindow/render/tab_bar.rs` |
| Clipboard | `steklo-gui/src/termwindow/clipboard.rs` |
| Selection | `steklo-gui/src/termwindow/selection.rs` |
| Window resize | `steklo-gui/src/termwindow/resize.rs` |
| Process spawning | `steklo-gui/src/termwindow/spawn.rs`, `steklo-gui/src/spawn.rs` |
| PTY I/O | `mux/src/lib.rs:281-373` (read_from_pane_pty) |
| Escape sequences | `term/src/terminalstate/performer.rs` |
| Font rendering | `crates/wezterm-font/`, `steklo-gui/src/glyphcache.rs` |
| macOS window | `window/src/os/macos/window.rs` |
| Self-update | `steklo/src/update.rs` |
