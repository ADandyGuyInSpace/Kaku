# Build System and Development

## Makefile Targets

| Target | Command | Description |
|--------|---------|-------------|
| `all` (default) | alias for `build` | |
| `build` | `cargo build -p steklo -p steklo-gui -p wezterm-mux-server-impl` | Build all main packages |
| `test` | `cargo nextest run` (2 invocations) | Tests excluding ligatures; also tests wezterm-escape-parser |
| `check` | `cargo check` (4 invocations) | Check steklo, steklo-gui, escape-parser, cell, surface, ssh |
| `app` | `PROFILE=debug ./scripts/build.sh --app-only` | Build debug macOS app bundle |
| `dev` | `cargo watch -x "run -p steklo-gui"` | Auto-reload dev mode (watches steklo-gui, window, term, mux, config, steklo, lua-api-crates) |
| `fmt` | `cargo +nightly fmt -p steklo -p steklo-gui -p mux -p wezterm-term -p termwiz -p config -p wezterm-font` | Format 7 core packages |
| `fmt-check` | same with `--check` | Verify formatting |
| `install-tools` | installs cargo-nextest, cargo-watch, nightly rustfmt | One-time dev setup |
| `test-webgpu-fallback` | `./scripts/test_webgpu_fallback.sh --strict` | Test GPU fallback |

## Recommended Dev Workflow

```bash
make install-tools    # One-time setup
make fmt              # Format first
make check            # Verify compilation
make test             # Run tests
make dev              # Fast local run (auto-rebuild)
RUST_LOG=debug make dev  # With debug logging
```

## Build Scripts (`scripts/`)

### `build.sh` (398 lines) -- Main Build Script

macOS-only. 7-step pipeline:
1. Build binaries (release universal by default: aarch64 + x86_64, `lipo` merge)
2. Prepare app bundle from `assets/macos/Steklo.app` template
3. Stamp version from `steklo/Cargo.toml` into `Info.plist`
4. Download vendor plugins (`download_vendor.sh`)
5. Copy resources (shell-integration, fonts, vendor plugins, terminfo)
6. Sign app bundle (ad-hoc for dev, `STEKLO_SIGNING_IDENTITY` for release)
7. Create update archive (`steklo_for_update.zip` + SHA256) and DMG

Environment variables:
- `PROFILE` -- `debug` (default for `make app`), `release`, `release-opt`
- `BUILD_ARCH` -- `universal` (default), `native`, `arm64`, `x86_64`
- `STEKLO_SIGNING_IDENTITY` -- Code signing identity for release
- `--native-arch` -- Build for current arch only (faster)
- `--app-only` -- Skip DMG creation
- `--open` -- Open app after build

### `notarize.sh` (167 lines) -- Apple Notarization

Submits to Apple notarization service. Environment variables:
- `STEKLO_NOTARIZE_APPLE_ID`
- `STEKLO_NOTARIZE_TEAM_ID`
- `STEKLO_NOTARIZE_PASSWORD`

### `download_vendor.sh` (51 lines) -- Vendor Dependencies

Git clones zsh-autosuggestions, zsh-syntax-highlighting, zsh-completions, zsh-z into `assets/vendor/`.

### `bench_startup.sh` (112 lines) -- Startup Benchmark

Cold-start comparison vs Ghostty/Alacritty using `hyperfine` + AppleScript.

### `reset_first_run.sh` (33 lines) -- Reset First-Run State

Removes `state.json` and legacy sentinel files.

### `test_webgpu_fallback.sh` (173 lines) -- WebGPU Fallback Test

Forces fallback adapter, verifies OpenGL marker in `vmmap` output.

## Cargo Profiles

```toml
[profile.release]
opt-level = 3, debug = false, strip = true, lto = "thin", codegen-units = 16, panic = "abort"

[profile.release-opt]  # For smallest binary
inherits = "release", opt-level = "z", lto = "fat", codegen-units = 1, strip = "symbols"

[profile.dev]
split-debuginfo = "unpacked"  # Faster incremental on macOS
```

## CI/CD (`.github/workflows/`)

### ci.yml (push to main, PRs)

Pipeline: `fmt` -> (`test` + `check`) -> `universal-build`

| Job | Runner | Description |
|-----|--------|-------------|
| `fmt` | macos-latest | Nightly fmt, auto-commit, verify |
| `test` | macos-latest | cargo-nextest (depends on fmt) |
| `check` | macos-latest | cargo check steklo + steklo-gui (depends on fmt) |
| `universal-build` | macos-latest | Build arm64+x86_64, lipo, verify universal (depends on check+test) |

### release.yml (tag-triggered)

1. Import signing certificate from secrets (P12 + temp keychain)
2. `PROFILE=release-opt BUILD_ARCH=universal ./scripts/build.sh`
3. `./scripts/notarize.sh` (if credentials available)
4. Upload to GitHub Release: DMG, update zip, SHA256

## .cargo/config.toml

```toml
[build]
rustflags = ["-A", "unexpected_cfgs"]  # Suppress cfg warnings globally

[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"

[target.x86_64-pc-windows-msvc]
rustflags = ["-C", "target-feature=+crt-static", "-A", "unexpected_cfgs"]
```

## Git Submodules

| Submodule | Path |
|-----------|------|
| harfbuzz | `deps/harfbuzz/harfbuzz` |
| freetype2 | `deps/freetype/freetype2` |
| libpng | `deps/freetype/libpng` |
| zlib | `deps/freetype/zlib` |
