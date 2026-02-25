# Steklo AI System

Steklo has a deeply integrated AI assistant for command-line error recovery and a unified settings UI for external AI coding tools.

## Architecture Overview

The AI system spans three layers:
1. **Shell hooks** (zsh) -- capture failed commands
2. **Lua handler** (steklo.lua) -- orchestrate API calls and display results
3. **Rust TUI** (steklo binary) -- configure AI tools

## Steklo Assistant (Error Recovery)

### How It Works

1. Shell command fails (exit code != 0)
2. zsh hooks (`_steklo_ai_preexec` / `_steklo_ai_precmd`) send command text + exit code as terminal user variables via OSC 1337 SetUserVar
3. Lua `user-var-changed` handler intercepts these
4. Reads `~/.config/steklo/assistant.toml` for API key, model, base URL
5. Builds prompt with: command, exit code, CWD, git branch context
6. Launches background `curl` to AI API (chat completions endpoint, OpenAI-compatible)
7. Polls response via file-based IPC (`~/.config/steklo/ai_jobs/` directory)
8. Parses JSON response for: `summary`, `command`, `why`, `confidence`
9. Injects styled notice into terminal with fix suggestion
10. User applies fix with **Cmd+Shift+E**

### Configuration -- `assistant_config.rs`

File: `~/.config/steklo/assistant.toml`

```toml
model = "DeepSeek-V3.2"
base_url = "https://api.vivgrid.com/v1"
api_key = ""
enabled = true
```

Source: `steklo/src/assistant_config.rs` (226 lines)
- `ensure_assistant_config()` -- creates config if missing, ensures required keys
- `AssistantConfig` struct with `model`, `base_url`, `api_key`, `enabled` fields

### Safety Features

- Dangerous commands (`rm -rf`, `mkfs`, `git reset --hard`, etc.) are detected and pasted without auto-execution
- Non-actionable commands (diagnostics only) are filtered out
- Confidence scoring in AI responses

### Key Bindings

| Shortcut | Action |
|----------|--------|
| `Cmd+Shift+A` | Open AI settings TUI (`run-steklo-ai-config` event) |
| `Cmd+Shift+E` | Apply latest AI fix suggestion (`steklo-ai-apply-last-fix` event) |

## AI Settings TUI

### Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `steklo/src/ai_config/mod.rs` | 88 | Entry point + OpenCode theme JSON |
| `steklo/src/ai_config/tui.rs` | 2659 | Full ratatui-based TUI |
| `steklo/src/ai_config/tui/ui.rs` | 327 | UI rendering |
| `steklo/src/ai_config/theme.rs` | 72 | Theme colors |

### Supported AI Tools (8 total)

Each tool has its own config path and field editor:

| Tool | Config Location |
|------|----------------|
| Steklo Assistant | `~/.config/steklo/assistant.toml` |
| Claude Code | `~/.claude.json` or `~/.claude/settings.json` |
| Codex | env-based (OPENAI_API_KEY) |
| Gemini CLI | `~/.gemini/settings.json` |
| Copilot CLI | GitHub auth |
| Factory Droid | custom config |
| OpenCode | `~/.config/opencode/opencode.json` |
| OpenClaw | custom config |

### Invocation

- CLI: `steklo ai`
- GUI: `Cmd+Shift+A` (triggers `run-steklo-ai-config` custom event)
- Interactive menu: select "ai" from `steklo` with no args

## Lua-Side AI Implementation

Located in `assets/macos/Steklo.app/Contents/Resources/steklo.lua` (~500 lines of AI logic):

### Key Functions

- **User var handler**: Watches for `STEKLO_AI_CMD` and `STEKLO_AI_EXIT_CODE` user variables
- **API call**: Constructs OpenAI-compatible chat completion request
- **Response parsing**: Expects JSON with `summary`, `command`, `why`, `confidence`
- **Display**: Renders styled inline notification in terminal
- **Apply**: `steklo-ai-apply-last-fix` event sends command to active pane

### File-Based IPC

The Lua side uses `~/.config/steklo/ai_jobs/` for async communication:
- Request triggers background `curl` that writes response to a job file
- Lua polls for job completion via `wezterm.time.call_after()` timer
- Completed jobs are parsed and displayed
