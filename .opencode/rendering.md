# GPU Rendering Pipeline

## Overview

Steklo uses a dual GPU backend: **wgpu (Metal)** as primary and **OpenGL (glium)** as fallback. The rendering pipeline is in `steklo-gui/src/`.

## Render Architecture

### Core Files

| File | Lines | Purpose |
|------|-------|---------|
| `termwindow/mod.rs` | 4045 | TermWindow -- core window struct |
| `termwindow/render/mod.rs` | | Render module root |
| `termwindow/render/paint.rs` | | Main paint entry point |
| `termwindow/render/draw.rs` | | Draw primitives |
| `termwindow/render/pane.rs` | | Individual pane rendering |
| `termwindow/render/screen_line.rs` | | Screen line rendering with glyph shaping |
| `termwindow/render/tab_bar.rs` | | Standard tab bar |
| `termwindow/render/fancy_tab_bar.rs` | | Enhanced tab bar |
| `termwindow/render/split.rs` | | Split divider rendering |
| `termwindow/render/borders.rs` | | Window border rendering |
| `termwindow/render/corners.rs` | | Corner rendering |
| `termwindow/render/window_buttons.rs` | | Window button rendering |
| `renderstate.rs` | | OpenGL/WebGPU render state management |
| `glyphcache.rs` | | Texture atlas for glyphs |
| `shapecache.rs` | | Text shaping result cache |
| `termwindow/webgpu.rs` | | WebGPU state and fallback logic |
| `quad.rs` | | Quad rendering primitives |
| `uniforms.rs` | | Shader uniforms |
| `customglyph.rs` | | Custom glyph rendering (box drawing, etc.) |
| `colorease.rs` | | Color easing animations |
| `utilsprites.rs` | | Utility sprites and render metrics |

### Render Flow

1. `TermWindow` receives `NeedRepaint` event
2. `render/paint.rs` -- decides what needs redrawing
3. For each visible pane: `render/pane.rs`
4. For each screen line: `render/screen_line.rs`
5. Text shaping via HarfBuzz (cached in `shapecache.rs`)
6. Glyph rasterization via FreeType (cached in `glyphcache.rs` texture atlas)
7. Quads built for each glyph cell
8. GPU draw call via wgpu (Metal) or glium (OpenGL)

### GPU Backends

**Primary: wgpu + Metal**
- Configured in workspace Cargo.toml: `wgpu = { features = ["metal", "wgsl"] }`
- macOS-only: Vulkan, DX12, GL backends removed
- State managed in `termwindow/webgpu.rs`

**Fallback: OpenGL (glium)**
- Activated when wgpu/Metal is unavailable
- Uses `glium` crate (v0.35)
- EGL setup in `window/src/egl.rs`
- Fallback tested via `scripts/test_webgpu_fallback.sh`

### Text Rendering Pipeline

1. **Font loading**: `wezterm-font` crate
   - System font discovery (CoreText on macOS)
   - Bundled JetBrains Mono + Nerd Font symbols fallback
   - Font configured in `config/src/font.rs`

2. **Text shaping**: HarfBuzz (vendored at `deps/harfbuzz/`)
   - Converts Unicode + font -> positioned glyph sequences
   - Results cached in `shapecache.rs`
   - Handles complex scripts, ligatures, etc.

3. **Glyph rasterization**: FreeType (vendored at `deps/freetype/`)
   - Renders glyph bitmaps at specified sizes
   - macOS font rendering tuned for Retina displays
   - Results cached in texture atlas (`glyphcache.rs`)

4. **GPU upload and draw**:
   - Glyphs uploaded to texture atlas
   - Quads rendered with proper UV coordinates
   - Cell backgrounds, cursor, selection rendered separately

### Custom Glyph Rendering

`customglyph.rs` handles:
- Box drawing characters (U+2500-U+257F)
- Block elements (U+2580-U+259F)
- Braille patterns (U+2800-U+28FF)
- Powerline symbols
- These are rendered procedurally, not from font

### Box Model

`termwindow/box_model.rs` implements a CSS-like box model for UI layout:
- Used for tab bar, modal dialogs, command palette
- Supports padding, margin, border concepts
- Flexible layout calculation

## Performance Optimizations

- Lazy color scheme loading (~1001 schemes loaded on demand)
- Text shaping cache (avoids re-shaping unchanged lines)
- Glyph texture atlas (avoids re-rasterizing known glyphs)
- Configurable parser coalesce delay (batches escape sequence processing)
- Stripped binary with aggressive LTO in release
- Feature-pruned wgpu (Metal only, no Vulkan/DX12/GL)
