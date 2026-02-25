# Shell Integration System

Steklo provides a comprehensive zsh-first shell integration installed via `steklo init`.

## Setup Flow

1. `steklo init` (Rust: `steklo/src/init.rs`)
   - Locates `setup_zsh.sh` in app bundle Resources
   - Runs it to install shell integration
   - Optionally installs OpenCode theme

2. `setup_zsh.sh` (872 lines: `assets/shell-integration/setup_zsh.sh`)
   - Creates `~/.config/steklo/zsh/` directory structure
   - Copies 4 bundled zsh plugins
   - Generates `steklo.zsh` init script
   - Patches `~/.zshrc` with source line
   - Configures starship, yazi, TouchID sudo

## Shell Integration Files

| File | Location | Purpose |
|------|----------|---------|
| `setup_zsh.sh` | `assets/shell-integration/` | Master zsh setup (872 lines) |
| `steklo.sh` | `assets/shell-integration/` | WezTerm-compatible integration (582 lines) |
| `first_run.sh` | `assets/shell-integration/` | First-run onboarding wizard (180 lines) |
| `install_cli_tools.sh` | `assets/shell-integration/` | Homebrew tool installer (194 lines) |
| `check_config_version.sh` | `assets/shell-integration/` | Config upgrade detector (133 lines) |
| `state_common.sh` | `assets/shell-integration/` | Shared state helpers (89 lines) |
| `install_opencode_theme.sh` | `assets/shell-integration/` | OpenCode theme installer (145 lines) |

## Bundled Zsh Plugins

Downloaded by `scripts/download_vendor.sh`, stored in `assets/vendor/`:

| Plugin | Purpose |
|--------|---------|
| zsh-z | Smart cd (learns most-used directories) |
| zsh-autosuggestions | History-based command suggestions |
| zsh-syntax-highlighting | Real-time command validation/coloring |
| zsh-completions | Extended command completions |

## Generated steklo.zsh Features

The generated `~/.config/steklo/zsh/steklo.zsh` provides:

- **PATH**: `$STEKLO_ZSH_DIR/bin` prepended
- **Starship prompt**: initialization
- **History**: 50000 lines, dedup, shared, timestamps
- **Key bindings**: prefix search Up/Down, Shift+Arrow selection, Cmd+A select-all, Tab smart-accept
- **Plugins**: zsh-z, autosuggestions, syntax-highlighting (deferred), completions
- **Aliases**: git (ga, gst, gco, etc.), ls, grep, directory navigation
- **yazi wrapper** (`y`/`yy`): with cwd sync on exit
- **SSH wrapper**: auto-sets `TERM=xterm-256color`
- **sudo wrapper**: same TERM fix
- **AI hooks**: preexec/precmd for command failure detection
- **1Password SSH agent fix**: auto-adds `IdentitiesOnly=yes`

## Optional CLI Tools

Installed via Homebrew during `steklo init` (user-prompted):

| Tool | Purpose |
|------|---------|
| Starship | Fast, customizable prompt |
| git-delta | Syntax-highlighting diff pager |
| Lazygit | Terminal Git UI (`Cmd+Shift+G`) |
| Yazi | Terminal file manager (`Cmd+Shift+Y` or `y`) |

## Shell Completions

| File | Shell |
|------|-------|
| `assets/shell-completion/_steklo` | Zsh (1072 lines, full command/option completions) |
| `assets/shell-completion/bash` | Bash (~1500 lines) |
| `assets/shell-completion/fish` | Fish |

## Reset Flow

`steklo reset` (Rust: `steklo/src/reset.rs`, 492 lines):
1. Removes Steklo source line from `.zshrc`
2. Strips legacy inline integration blocks
3. Removes `steklo.zsh` from `~/.config/steklo/zsh/`
4. Unsets Steklo-managed git defaults (delta config)
5. Strips managed theme blocks from `steklo.lua`
6. Removes state files

## Config Version System

The config upgrade system (`check_config_version.sh`) uses a version number (currently 11) stored in `~/.config/steklo/state.json`. On each terminal launch, it checks if upgrades need to be applied and prompts the user.
