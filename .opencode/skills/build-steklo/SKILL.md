---
name: build-steklo
description: Build, test, and run Steklo terminal emulator locally
---

## Build and Run Steklo

### Prerequisites

- macOS (Steklo is macOS-only)
- Rust stable toolchain
- Rust nightly toolchain (for rustfmt only)
- cargo-nextest, cargo-watch (install via `make install-tools`)
- Git submodules initialized (`git submodule update --init --recursive`)

### Quick Development Cycle

```bash
make fmt          # Format code (requires nightly rustfmt)
make check        # Fast compile check
make test         # Run tests with cargo-nextest
make dev          # Auto-rebuild + run steklo-gui on changes
```

Override log level: `RUST_LOG=debug make dev`

### Build App Bundle

```bash
make app                              # Debug app bundle -> dist/Steklo.app
./scripts/build.sh --native-arch      # Release, current arch only
./scripts/build.sh                    # Release, universal binary (arm64+x86_64)
./scripts/build.sh --native-arch --open  # Build and open
```

### Test Specific Crates

```bash
cargo nextest run -p wezterm-escape-parser  # Escape parser (no_std)
cargo nextest run -p wezterm-cell           # Cell types
cargo check -p steklo                         # CLI binary
cargo check -p steklo-gui                     # GUI binary
```

### Formatting

Only 7 core packages are formatted (not deps or vendored code):
`steklo`, `steklo-gui`, `mux`, `wezterm-term`, `termwiz`, `config`, `wezterm-font`

Uses nightly rustfmt with: edition 2018, `imports_granularity = "Module"`, 4-space indent.

### CI Pipeline

Push to main or PR triggers: fmt -> (test + check) -> universal-build

### Common Issues

- **Submodules not initialized**: Run `git submodule update --init --recursive` for freetype, harfbuzz, libpng, zlib
- **Nightly rustfmt missing**: Run `rustup toolchain install nightly --component rustfmt`
- **Ligature test failures**: The `shapecache::test::ligatures_jetbrains` test is excluded in Makefile (font-dependent)
