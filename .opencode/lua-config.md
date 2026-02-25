# Lua Configuration System

## Overview

Steklo uses WezTerm's Lua configuration system with full API compatibility. Config is loaded from `~/.config/steklo/steklo.lua` with bundled defaults at `Steklo.app/Contents/Resources/steklo.lua`.

## Config Loading

Source: `config/src/lib.rs` and `config/src/lua.rs`

1. Config path resolved at `config/src/lib.rs:477`
2. Lua file evaluated by mlua (Lua 5.4)
3. Returned table mapped to `Config` struct via `wezterm-dynamic` (FromDynamic/ToDynamic)
4. File watching via `notify` crate triggers reload
5. Changes transferred to main thread via Lua config pipe

## Config Structure

Source: `config/src/config.rs` -- massive `Config` struct

Key sub-modules:
| File | Content |
|------|---------|
| `config/src/keyassignment.rs` | `KeyAssignment` enum -- all possible key binding actions |
| `config/src/keys.rs` | Key binding definitions |
| `config/src/font.rs` | Font configuration |
| `config/src/color.rs` | Color scheme and palette |
| `config/src/window.rs` | Window appearance (padding, opacity) |
| `config/src/background.rs` | Background image/gradient config |
| `config/src/bell.rs` | Bell behavior |
| `config/src/terminal.rs` | Terminal behavior |
| `config/src/ssh.rs` | SSH configuration |
| `config/src/frontend.rs` | Frontend selection (WebGpu/OpenGL) |

## Lazy Color Scheme Registry

Source: `config/src/lib.rs:178-262`

Instead of eagerly loading all ~1001 color schemes at startup, schemes are loaded on-demand. This significantly improves startup time.

## Default User Config Template

Source: `config/src/lib.rs:601-684`

Generated config loads bundled defaults and provides commented-out overrides:
```lua
-- Load bundled Steklo defaults
local ok, defaults = pcall(dofile, '/Applications/Steklo.app/Contents/Resources/steklo.lua')
if ok and type(defaults) == 'table' then
  for k, v in pairs(defaults) do config[k] = v end
end
-- Override examples below...
```

## Bundled steklo.lua

Source: `assets/macos/Steklo.app/Contents/Resources/steklo.lua` (2372 lines)

Major sections:
1. **Window config** (lines 1-150): padding, fullscreen handling, resize debounce
2. **Theme** (lines ~150-400): Steklo dark theme colors, cursor, tab bar
3. **Font** (lines ~400-500): JetBrains Mono, fallback chain, sizing
4. **Key bindings** (lines ~500-800): macOS-native shortcuts (Cmd+T, Cmd+D, etc.)
5. **Tab title** (lines ~800-1000): dynamic tab titles with process name/CWD
6. **AI fix system** (lines ~1000-1600): command error detection, API calls, display
7. **Events** (lines ~1600-2372): window-resized, user-var-changed, custom events

## WezTerm Lua API

Full API compatibility. Key namespaces:
- `wezterm` -- core functions (log, time, action, etc.)
- `wezterm.gui` -- GUI functions (get_appearance, etc.)
- `wezterm.mux` -- multiplexer functions
- `wezterm.time` -- time functions (call_after, etc.)
- `wezterm.action` -- key binding actions
- `wezterm.color` -- color manipulation
- `wezterm.font` -- font resolution

## Custom Events Used by Steklo

| Event | Purpose |
|-------|---------|
| `run-steklo-ai-config` | Launch AI settings TUI (Cmd+Shift+A) |
| `steklo-ai-apply-last-fix` | Apply AI fix suggestion (Cmd+Shift+E) |
| `toggle-split-direction` | Toggle pane split direction (Cmd+Shift+S) |
| `user-var-changed` | Intercept shell user variables (AI hooks) |
| `window-resized` | Handle fullscreen padding changes |

## Config Derive Macro

Source: `config/derive/` -- `wezterm-config-derive` proc macro

Provides `#[derive(ConfigMeta)]` for automatic config field documentation and metadata generation.
