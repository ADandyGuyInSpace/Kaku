# Steklo AI Chat Panel — Implementation Plan
## Project Overview
We are forking [Steklo](https://github.com/tw93/Steklo) (a WezTerm fork built for AI coding) to add a native side panel that functions as an LLM chat interface. The panel translates natural language into bash commands and can inject them directly into the active terminal pane.
Steklo is written in Rust, uses wgpu for terminal rendering, and already has LLM infrastructure (API key management, model config, base URL config via `steklo ai`).
### Goal
Add a toggleable right-side panel containing a webview-based chat UI that:
- Accepts natural language input from the user
- Sends it to a configurable LLM (OpenAI, Anthropic, local Ollama, etc.)
- Streams the response back with markdown/code rendering
- Provides "Run in terminal" and "Copy" buttons on code blocks
- Supports rich interactions: clickable buttons, dropdown model selector, right-click context menus, scrollable chat history
- Reads context from the active terminal pane (current directory, recent output, last failed command)
### Architecture Decision
We use an **embedded webview** (`wry` crate / native `WKWebView` on macOS) as a child view of Steklo's application window rather than:
- Building custom widgets in the wgpu rendering pipeline (too complex, months of work)
- Running a TUI in a split pane (can't support rich click/dropdown/button interactions well)
The webview renders HTML/CSS/JS for the chat UI. Communication between the Rust backend and the webview frontend happens via IPC message passing.
---
## Codebase Orientation
Before making changes, understand the key directories in the Steklo repo:
```
steklo/                  # CLI binary (steklo command, subcommands like steklo ai)
steklo-gui/              # GUI application binary (main window, event loop, rendering)
mux/                   # Multiplexer: pane management, tabs, splits, input routing
  mux/src/pane.rs      # Pane trait and pane types
  mux/src/tab.rs       # Tab layout, split logic
  mux/src/domain.rs    # Pane creation and lifecycle
termwiz/               # Terminal capabilities, escape sequences, surface abstraction
window/                # Window management, native platform abstractions (NSWindow, etc.)
config/                # Configuration schema, Lua config loading, defaults
  config/src/lib.rs    # Main config struct (already has AI-related fields)
crates/                # Supporting crates (likely includes AI/LLM client code)
lua-api-crates/        # Lua API bindings for WezTerm config compatibility
assets/                # Icons, default configs, bundled resources
```
Key files to study first:
1. `config/src/lib.rs` — Find the existing AI config fields (api_key, model, base_url)
2. `steklo-gui/src/main.rs` or `steklo-gui/src/termwindow/` — How the main window is created and managed
3. `mux/src/pane.rs` — The Pane trait, how panes send/receive text
4. `window/src/` — Platform-specific window code (look for macOS/cocoa abstractions)
5. `crates/` — Look for existing HTTP/LLM client code used by Steklo Assistant
---
## Phase 1: Webview Integration into the Steklo Window
### Objective
Embed a `WKWebView` as a child NSView of Steklo's main application window, positioned on the right side. The terminal rendering area shrinks to accommodate it.
### Steps
#### 1.1 Add dependencies to the workspace
In the root `Cargo.toml`, add:
```toml
[workspace.dependencies]
wry = "0.47"           # Webview abstraction (uses WKWebView on macOS)
serde_json = "1.0"     # IPC message serialization (may already exist)
```
Alternatively, if `wry` proves too heavy or conflicts with Steklo's window management, use raw `objc` / `cocoa` crates to create a `WKWebView` directly. Steklo's `window/` crate likely already uses `objc` for macOS platform code. Check `window/Cargo.toml` for existing platform dependencies.
#### 1.2 Create a new crate for the panel
```bash
mkdir -p crates/ai-panel/src
```
Create `crates/ai-panel/Cargo.toml`:
```toml
[package]
name = "ai-panel"
version = "0.1.0"
edition = "2021"
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```
Create `crates/ai-panel/src/lib.rs` — this crate provides:
- `AiPanel` struct: manages the webview lifecycle
- `AiPanelMessage` enum: typed IPC messages (Rust -> JS and JS -> Rust)
- `AiPanelConfig` struct: panel-specific config (width, position, theme)
```rust
pub struct AiPanel {
    visible: bool,
    width: u32,             // panel width in pixels
    webview: Option</* platform webview handle */>,
}
/// Messages from Rust to the webview
#[derive(Serialize)]
#[serde(tag = "type")]
pub enum ToWebview {
    LlmChunk { content: String, done: bool },
    TerminalContext { cwd: String, last_output: String },
    ConfigUpdate { model: String, available_models: Vec<String> },
    CommandResult { success: bool, output: String },
}
/// Messages from the webview to Rust
#[derive(Deserialize)]
#[serde(tag = "type")]
pub enum FromWebview {
    SendMessage { content: String },
    RunCommand { command: String },
    CopyToClipboard { text: String },
    ChangeModel { model: String },
    TogglePanel,
    RequestContext,
}
```
#### 1.3 Embed the webview into Steklo's window
Find where Steklo creates the main application window. This is likely in:
- `steklo-gui/src/termwindow/` — the TermWindow struct
- `window/src/os/macos/` — platform-specific window creation
The approach:
1. After the main `NSWindow` is created, obtain its `contentView`
2. Create a `WKWebView` instance as a child `NSView`
3. Position it on the right side of the content view
4. Resize the terminal's rendering surface to `window_width - panel_width`
5. Load the chat UI HTML into the webview
The terminal surface resize is the critical part. Look for where the wgpu surface size is calculated — it likely reads the window's content rect. You need to subtract the panel width from the available width when the panel is visible.
**Important:** The webview must not intercept keyboard events meant for the terminal. Implement focus management:
- Clicking in the terminal area focuses the terminal (webview stops receiving key events)
- Clicking in the webview focuses it (terminal stops receiving key events)
- `Escape` in the webview returns focus to the terminal
- The toggle keybinding works regardless of focus
#### 1.4 Set up IPC bridge
The webview and Rust communicate via string messages:
**Rust -> Webview:**
```rust
// Evaluate JavaScript in the webview
webview.evaluate_script(&format!(
    "window.steklo.receive({})",
    serde_json::to_string(&message)?
));
```
**Webview -> Rust:**
```javascript
// In the webview JS, call the IPC handler
window.ipc.postMessage(JSON.stringify({
    type: "SendMessage",
    content: "list all files in the current directory"
}));
```
On the Rust side, register an IPC handler when creating the webview that deserializes `FromWebview` messages and dispatches them.
#### 1.5 Add toggle keybinding
In the keybinding configuration (look for where `Cmd+Shift+A` and `Cmd+Shift+E` are defined), add:
- `Cmd+Shift+L` (or `Cmd+Shift+C`) — toggle the AI chat panel
This keybinding should:
1. If panel is hidden: create/show the webview, resize terminal surface
2. If panel is visible: hide the webview, expand terminal surface back to full width
#### 1.6 Verification
At the end of Phase 1, you should be able to:
- Press `Cmd+Shift+L` and see a blank webview panel appear on the right
- The terminal shrinks to accommodate it
- Press the keybinding again to hide it
- Send a test message from Rust to JS and see it logged in the webview console
- Send a test message from JS to Rust and see it logged in Rust
---
## Phase 2: Chat UI (HTML/CSS/JS)
### Objective
Build the complete chat interface as a single-page HTML application that runs inside the webview.
### 2.1 HTML structure
Create the HTML file at `assets/ai-panel/index.html` (bundled into the app):
```
assets/ai-panel/
├── index.html        # Main HTML file
├── styles.css        # All styles
├── panel.js          # Main application logic
├── markdown.js       # Markdown rendering (bundled marked.js)
└── highlight.js      # Syntax highlighting (bundled highlight.js)
```
Or, bundle everything into a single HTML file with inline CSS/JS to simplify loading.
### 2.2 UI components to implement
#### Chat message list
- Scrollable container showing conversation history
- User messages styled differently from assistant messages
- Assistant messages render markdown with:
  - Inline code with monospace font
  - Code blocks with syntax highlighting and a "Copy" button + "Run in Terminal" button
  - Lists, bold, italic, headers
- Auto-scroll to bottom on new messages
- Manual scroll-up pauses auto-scroll (resume when user scrolls back to bottom)
#### Input area
- Multiline text input at the bottom of the panel
- `Enter` sends the message, `Shift+Enter` for newline
- "Send" button next to the input
- Input disabled while waiting for response (show loading indicator)
#### Header bar
- Model selector dropdown (populated from config)
- "Clear conversation" button
- "Close panel" button (sends TogglePanel message to Rust)
- Connection status indicator (green dot = API configured, red = missing key)
#### Code block actions
Every code block in assistant responses should have:
- **Copy** button — copies the code to clipboard
- **Run in Terminal** button — sends the command to the active terminal pane via IPC -> Rust -> mux
- **Insert in Terminal** button — pastes the command into the terminal without executing (no trailing newline)
#### Context menu (right-click)
Right-click on a message should show:
- Copy message
- Copy as markdown
- Re-run this prompt
- Delete message
Implement with a custom context menu div (prevent default browser context menu).
#### Dropdown selector
The model selector should be a styled `<select>` or custom dropdown with:
- Currently selected model displayed
- List of available models from config
- "Custom..." option that opens an input for arbitrary model names
### 2.3 Styling
The panel should visually integrate with Steklo's terminal aesthetic:
```css
:root {
    --bg-primary: #1a1b26;        /* Match Steklo's default dark theme */
    --bg-secondary: #24283b;
    --bg-input: #1f2335;
    --text-primary: #c0caf5;
    --text-secondary: #565f89;
    --accent: #7aa2f7;
    --border: #292e42;
    --code-bg: #1f2335;
    --user-msg-bg: #292e42;
    --assistant-msg-bg: transparent;
    --button-bg: #3b4261;
    --button-hover: #444b6a;
    font-family: 'JetBrains Mono', 'SF Mono', 'Fira Code', monospace;
    font-size: 13px;
}
```
Use `prefers-color-scheme` or receive theme info from Rust to support light themes.
The panel width should be resizable by dragging the left border (implement a drag handle div).
### 2.4 JavaScript application structure
```javascript
// panel.js
const steklo = {
    messages: [],           // conversation history
    currentModel: '',       // active model name
    isStreaming: false,      // currently receiving LLM response
    config: {},             // received from Rust
    // Initialize
    init() {
        window.steklo = { receive: this.handleFromRust.bind(this) };
        this.bindEvents();
        this.requestContext();
    },
    // Handle messages from Rust
    handleFromRust(msg) {
        switch (msg.type) {
            case 'LlmChunk':
                this.appendChunk(msg.content, msg.done);
                break;
            case 'TerminalContext':
                this.updateContext(msg);
                break;
            case 'ConfigUpdate':
                this.updateConfig(msg);
                break;
        }
    },
    // Send message to Rust
    sendToRust(msg) {
        window.ipc.postMessage(JSON.stringify(msg));
    },
    // User sends a chat message
    sendMessage(content) {
        this.addMessage('user', content);
        this.sendToRust({ type: 'SendMessage', content });
        this.isStreaming = true;
        this.addMessage('assistant', ''); // placeholder for streaming
    },
    // Append streaming chunk to last assistant message
    appendChunk(content, done) {
        // append to last message, re-render markdown
        if (done) this.isStreaming = false;
    },
    // Run a command in the terminal
    runCommand(command) {
        this.sendToRust({ type: 'RunCommand', command });
    },
    // Copy text to clipboard
    copyText(text) {
        this.sendToRust({ type: 'CopyToClipboard', text });
    }
};
steklo.init();
```
### 2.5 Verification
At the end of Phase 2:
- The panel shows a styled chat interface
- You can type messages (they appear in the chat as user messages)
- Code blocks render with syntax highlighting
- Copy/Run buttons exist on code blocks and send IPC messages to Rust
- Model dropdown opens and sends selection changes to Rust
- Right-click context menu works
- Panel is resizable by dragging the left edge
---
## Phase 3: LLM Integration
### Objective
Wire the chat UI to actual LLM API calls, using Steklo's existing configuration.
### 3.1 Read Steklo's existing AI config
Steklo already stores AI configuration. Find the config struct (likely in `config/src/lib.rs` or a dedicated AI config module) and read:
- `api_key` — the user's API key
- `base_url` — API endpoint (for OpenAI-compatible APIs, Ollama, etc.)
- `model` — default model name
If needed, extend the config to add panel-specific fields:
```rust
// In config
pub struct AiPanelConfig {
    pub enabled: bool,
    pub default_width: u32,
    pub system_prompt: String,
    pub max_context_lines: usize,   // how many lines of terminal output to include
    pub available_models: Vec<String>,
}
```
### 3.2 Implement streaming LLM client
Create or extend the LLM client in `crates/ai-panel/src/llm.rs`:
```rust
pub struct LlmClient {
    client: reqwest::Client,
    base_url: String,
    api_key: String,
    model: String,
}
impl LlmClient {
    /// Send a chat completion request and stream the response.
    /// Calls `on_chunk` for each token and `on_done` when complete.
    pub async fn stream_chat(
        &self,
        messages: Vec<ChatMessage>,
        system_prompt: &str,
        on_chunk: impl Fn(String),
        on_done: impl Fn(),
    ) -> Result<(), Error> {
        // Build the request based on the base_url format:
        // - OpenAI-compatible: POST {base_url}/v1/chat/completions
        // - Anthropic: POST {base_url}/v1/messages
        // - Ollama: POST {base_url}/api/chat
        //
        // Use reqwest with streaming response body.
        // Parse SSE events, extract content deltas, call on_chunk.
    }
}
```
Support at minimum:
- OpenAI-compatible API (covers OpenAI, Together, Groq, local vLLM, LMStudio)
- Anthropic API (different request/response format)
- Ollama API (different endpoint structure)
Auto-detect the API format from the base_url or add an explicit `api_format` config field.
### 3.3 System prompt
The default system prompt should instruct the LLM to:
```
You are a terminal assistant embedded in Steklo terminal. The user will ask you
to perform tasks in their terminal. Respond with the exact commands they should
run.
Rules:
- When suggesting commands, put each command in its own code block with the
  language set to `bash` (or `sh`, `zsh` as appropriate).
- Explain briefly what each command does.
- If a task requires multiple steps, number them.
- If you're unsure about the user's OS or environment, ask.
- Be concise. Terminal users prefer brief, actionable responses.
- When the user provides terminal context (current directory, recent output),
  use it to give more accurate suggestions.
The user's current working directory is: {cwd}
Their shell is: {shell}
Their OS is: {os}
```
### 3.4 Wire IPC to LLM calls
When the Rust IPC handler receives a `SendMessage` from the webview:
1. Build the message array (conversation history + new user message)
2. Inject terminal context into the system prompt (cwd, last N lines of output)
3. Call `LlmClient::stream_chat`
4. For each chunk, send `ToWebview::LlmChunk { content, done: false }` to the webview
5. On completion, send `ToWebview::LlmChunk { content: "", done: true }`
### 3.5 Conversation management
Store conversation history in memory (reset on panel close or explicit clear). Optionally persist to `~/.config/steklo/chat_history.json` for session continuity.
Implement a maximum context window: if the conversation exceeds the model's token limit, truncate older messages (keep the system prompt and last N messages).
### 3.6 Verification
At the end of Phase 3:
- Type a natural language request in the panel
- See a streamed response from the configured LLM
- Code blocks in the response have working Copy/Run buttons
- Switching models via the dropdown changes the active model
- The LLM receives terminal context (cwd at minimum)
---
## Phase 4: Terminal Integration
### Objective
Connect the chat panel to the active terminal pane for bidirectional interaction.
### 4.1 Send commands to the terminal
When the user clicks "Run in Terminal" on a code block:
1. Webview sends `FromWebview::RunCommand { command }` via IPC
2. Rust handler gets the active pane from the mux: `mux.get_active_pane()`
3. Sends the command as input to the pane: `pane.writer().write_all(command.as_bytes())`
4. Sends a newline to execute: `pane.writer().write_all(b"\n")`
For "Insert in Terminal" (paste without executing):
- Same as above but omit the trailing newline
Look at how Steklo Assistant's `Cmd+Shift+E` "apply suggestion" works — it already does exactly this. Reuse that code path.
### 4.2 Read terminal context
To give the LLM useful context about what the user is doing:
**Current working directory:**
- Option A: Parse the terminal's OSC 7 escape sequence (terminals send this to report cwd)
- Option B: Read `/proc/{pid}/cwd` for the shell process (Linux) or use `lsof` (macOS)
- Option C: Send `pwd` to the terminal and capture output (hacky, avoid)
Look at how WezTerm/Steklo already tracks cwd — there's likely existing infrastructure for this in the mux layer or the Pane trait.
**Recent terminal output:**
- Access the pane's scrollback buffer: the `Pane` trait should have methods to read lines from the terminal surface/scrollback
- Extract the last N lines (configurable, default 50)
- Send this to the LLM as context when the user asks a question
**Last failed command (Steklo Assistant integration):**
- Steklo Assistant already detects failed commands. Hook into this detection:
- When a command fails (non-zero exit code), automatically populate the chat panel with context about the failure
- Optionally auto-open the panel and pre-fill a "Why did this fail?" prompt
### 4.3 Terminal output streaming to panel
For "explain this output" workflows:
1. User selects text in the terminal (existing selection mechanism)
2. User presses a keybinding (e.g., `Cmd+Shift+Q`) or right-clicks -> "Ask AI about this"
3. Selected text is sent to the chat panel as context
4. Panel auto-opens if hidden and pre-fills a prompt like: "Explain this terminal output: ..."
### 4.4 Error recovery flow
Integrate with Steklo Assistant's error detection:
1. Steklo detects a command failure (non-zero exit code)
2. Instead of (or in addition to) the inline suggestion, send the failed command + error output to the chat panel
3. The panel shows: "Command `xyz` failed. Would you like me to help?" with a pre-filled prompt
4. The LLM can provide a more detailed explanation than the inline assistant
### 4.5 Verification
At the end of Phase 4:
- Click "Run in Terminal" on a code block and see the command execute in the terminal
- Click "Insert in Terminal" and see the command appear in the terminal prompt without executing
- The LLM knows the user's current working directory
- Selecting text in the terminal and pressing a keybinding sends it to the chat panel
- Failed commands optionally trigger the chat panel with context
---
## Phase 5: Polish and Configuration
### 5.1 Resizable panel
- Drag handle on the left edge of the panel
- Panel width persists across sessions (save in config)
- Minimum width: 280px, maximum: 50% of window width
- Double-click the drag handle to reset to default width
### 5.2 Keyboard shortcuts within the panel
| Action              | Shortcut          |
|---------------------|-------------------|
| Send message        | `Enter`           |
| Newline             | `Shift+Enter`     |
| Clear conversation  | `Cmd+K` (in panel focus) |
| Focus input         | `/`               |
| Close panel         | `Escape`          |
| Toggle panel        | `Cmd+Shift+L`     |
### 5.3 Configuration options
Extend `~/.config/steklo/steklo.lua`:
```lua
config.ai_panel = {
    enabled = true,
    width = 400,
    position = "right",             -- "right" or "left"
    font_size = 13,
    system_prompt = "...",          -- override default system prompt
    max_context_lines = 50,         -- terminal lines sent as context
    auto_open_on_error = false,     -- auto-open panel on command failure
    models = {                      -- model presets for the dropdown
        "claude-sonnet-4-20250514",
        "gpt-4o",
        "deepseek-chat",
        "ollama/llama3",
    },
}
```
### 5.4 Theme integration
- Read Steklo's color scheme from config
- Pass theme colors to the webview on load and on config change
- The webview CSS uses CSS variables that are set from the Steklo theme
- Dark/light mode support
### 5.5 Performance considerations
- Webview is created once and hidden/shown (not destroyed/recreated on toggle)
- LLM streaming uses proper async I/O, never blocks the main thread
- Terminal context collection is debounced (don't read scrollback on every keystroke)
- Message history is bounded (default 100 messages, configurable)
- The webview HTML/CSS/JS is bundled into the binary (not loaded from disk at runtime)
---
## File Change Summary
### New files
```
crates/ai-panel/Cargo.toml           -- New crate manifest
crates/ai-panel/src/lib.rs           -- Panel struct, IPC message types, public API
crates/ai-panel/src/llm.rs           -- LLM client (streaming HTTP, multi-provider)
crates/ai-panel/src/ipc.rs           -- IPC message serialization/dispatch
crates/ai-panel/src/context.rs       -- Terminal context extraction
assets/ai-panel/index.html           -- Chat UI (HTML)
assets/ai-panel/styles.css           -- Chat UI styles
assets/ai-panel/panel.js             -- Chat UI logic
assets/ai-panel/markdown.js          -- Bundled marked.js for markdown rendering
assets/ai-panel/highlight.js         -- Bundled highlight.js for syntax highlighting
```
### Modified files
```
Cargo.toml                           -- Add ai-panel to workspace members
steklo-gui/Cargo.toml                  -- Add ai-panel dependency
steklo-gui/src/termwindow/mod.rs       -- Add panel toggle logic, resize terminal on toggle
steklo-gui/src/termwindow/render.rs    -- Adjust render surface width when panel is visible
window/src/os/macos/window.rs        -- Embed WKWebView as child NSView (or equivalent)
config/src/lib.rs                    -- Add AiPanelConfig fields
mux/src/pane.rs                      -- Add methods to read scrollback for context (if not existing)
```
### Files to study (do not modify initially)
```
steklo-gui/src/termwindow/keyevent.rs  -- Understand keybinding dispatch (to add toggle)
mux/src/tab.rs                       -- Understand split/layout (for terminal resize)
crates/*/                            -- Find existing LLM/HTTP client code to reuse
steklo/src/                             -- Understand steklo CLI (steklo ai subcommand) for config
```
---
## Build and Test
### Build
```bash
cargo build --release -p steklo-gui
```
### Test the panel in isolation
Before integrating the webview, test the chat UI by opening `assets/ai-panel/index.html` in a browser. Mock the `window.ipc.postMessage` function to log messages to the console. Mock `window.steklo.receive` to inject test responses.
### Integration test checklist
- [ ] Panel toggles on/off with keybinding
- [ ] Terminal resizes correctly when panel opens/closes
- [ ] Typing in the panel does not send keystrokes to the terminal
- [ ] Typing in the terminal does not affect the panel input
- [ ] LLM streaming works with OpenAI-compatible API
- [ ] LLM streaming works with Anthropic API
- [ ] LLM streaming works with Ollama
- [ ] "Run in Terminal" executes the command in the active pane
- [ ] "Insert in Terminal" pastes without executing
- [ ] "Copy" copies to system clipboard
- [ ] Model dropdown changes the active model
- [ ] Panel width is resizable by dragging
- [ ] Panel width persists across sessions
- [ ] Terminal context (cwd) is sent to the LLM
- [ ] Panel respects Steklo's color theme
- [ ] Panel works correctly with multiple tabs
- [ ] Panel works correctly with split panes (sends to the active/focused pane)
- [ ] No memory leaks from the webview over long sessions
- [ ] App startup time is not noticeably affected when panel is hidden
---
## Implementation Order
If implementing incrementally (recommended), follow this dependency chain:
```
1. Study codebase (read key files listed above)
   │
2. Create crates/ai-panel with types and IPC messages
   │
3. Embed a minimal webview (blank white panel) into the window
   │
4. Add toggle keybinding (show/hide + terminal resize)
   │
5. Build the chat HTML/CSS/JS with mock data
   │
6. Wire IPC: messages flow between webview <-> Rust
   │
7. Implement LLM client with streaming
   │
8. Connect LLM responses to webview via IPC
   │
9. Implement "Run in Terminal" (Rust -> mux -> active pane)
   │
10. Add terminal context reading (cwd, scrollback)
    │
11. Add config integration (read from steklo.lua)
    │
12. Polish: theming, resize handle, persistence, error states
```
Each step should be independently testable before moving to the next.
Written to ~/steklo-ai-panel-plan.md. It covers:
- Codebase orientation — which files to read, what each directory does
- Phase 1 — Embedding a webview into the Steklo window via wry/WKWebView, IPC bridge, toggle keybinding
- Phase 2 — Full chat UI spec (HTML/CSS/JS) with components: message list, input area, code block actions, dropdown selectors, context menus, styling variables matching Steklo's theme
- Phase 3 — LLM client (streaming, multi-provider: OpenAI/Anthropic/Ollama), system prompt, conversation management
- Phase 4 — Terminal integration: run/insert commands, read cwd + scrollback, error recovery hooks, selection-to-panel flow
- Phase 5 — Polish: resize handle, keyboard shortcuts, Lua config schema, theme integration, perf considerations
- File change summary — Exact new and modified files
- Dependency-ordered implementation chain — 12 steps, each independently testable
The plan is structured so an LLM can work through it step-by-step. Each phase has a verification section defining what "done" looks like before moving on. Feed it in alongside the cloned repo and it should be able to start from step 1.
