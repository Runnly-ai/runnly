# Codex CLI — Architecture & Code Logic

This document provides a deep-dive into the architecture of the Codex CLI Rust implementation (`codex-rs`). It is intended for developers who want to understand how the system works internally.

---

## 1. Workspace Overview

The `codex-rs` Cargo workspace contains **80+ crates** organized into functional layers. The main execution path is:

```
CLI → TUI / App-Server → Core (orchestration) → Model Client → Provider API
```

### Crate Map by Layer

| Layer | Crates | Purpose |
|---|---|---|
| **Entrypoint** | `cli/` | Binary entrypoint, argument parsing, subcommand dispatch |
| **User Interface** | `tui/` | Ratatui-based interactive TUI |
| **Server** | `app-server/`, `app-server-daemon/`, `app-server-client/`, `app-server-transport/` | JSON-RPC 2.0 server bridging clients to core |
| **Protocol** | `app-server-protocol/`, `protocol/` | Wire types for session ops/events and JSON-RPC schema |
| **Orchestration** | `core/` | Config, session, turn lifecycle, tool execution, rollout persistence |
| **Model API** | `codex-api/`, `codex-client/` | HTTP SSE and WebSocket transport for model provider APIs |
| **Auth** | `login/` | OAuth, API key, and agent-identity credential management |
| **MCP** | `codex-mcp/`, `mcp-server/`, `rmcp-client/` | Model Context Protocol client and server |
| **Execution** | `exec/`, `exec-server/`, `shell-command/` | Headless mode, background terminals, command execution |
| **Sandboxing** | `sandboxing/`, `execpolicy/`, `bwrap/`, `linux-sandbox/`, `windows-sandbox-rs/` | Platform-specific sandbox enforcement |
| **Config** | `config/` | Config loading, layering, schema, constraints |
| **Persistence** | `state/`, `thread-store/`, `rollout-trace/` | SQLite metadata DB, thread store abstraction, trace data |
| **Skills** | `skills/`, `core-skills/` | Skill definitions, loading, and prompt injection |
| **Features** | `features/` | Feature flag system with stage gating |
| **Tooling** | `core-plugins/`, `tools/`, `connectors/` | Plugin system, tool implementations, service connectors |
| **Utils** | `utils/*` (20+ crates) | Shared small utilities: paths, caching, PTY, terminal detection, etc. |

---

## 2. Entrypoint: `cli/`

The `clap`-based `MultitoolCli` in `cli/src/main.rs` parses all arguments and dispatches to subcommands:

- **No subcommand**: Launches the interactive TUI (main consumer path)
- `codex exec` / `codex review`: Non-interactive headless execution
- `codex login` / `codex logout`: Authentication flows
- `codex mcp`: Manage MCP server launchers
- `codex mcp-server`: Run Codex as an MCP server (for other agents)
- `codex app-server`: Experimental JSON-RPC daemon
- `codex sandbox`: Test sandbox configurations
- `codex doctor`: Diagnostics and troubleshooting
- `codex update`: Self-update

The `arg0` dispatch system (`codex-arg0`) detects symlink-based invocation names so the binary can support subcommand-based dispatch via argv[0] inspection.

---

## 3. Protocol: `codex-protocol`

The protocol layer defines the **SQ/EQ (Submission Queue / Event Queue)** pattern that governs all communication between a client (TUI, IDE) and the agent (core session loop).

### `Op` (Operations — Client → Agent)

Key variants:

- `UserInput` / `UserTurn` — send user messages with full turn context
- `Interrupt` / `CleanBackgroundTerminals` — lifecycle control
- `RealtimeConversation{Start,Audio,Text,Close}` — WebRTC audio/text
- `ReviewStart` / `ReviewRespond` — code review flow
- `Compact` — history compaction
- `ApplyPatchApproval` / `ExecApproval` — user approval responses
- `Steer` — redirect the active turn

### `Event` (Events — Agent → Client)

Key variants:

- `SessionConfiguredEvent` — emitted once on thread start with all active settings
- `ItemStartedEvent` / `ItemCompletedEvent` — lifecycle for response items
- `ContentItemEvent` — streaming text content
- `FunctionCallEvent` — tool invocation
- `FunctionCallOutputEvent` — tool result
- `TurnAbortedEvent` — turn interrupted or errored
- `StatusEvent` — agent status changes
- `TokenUsageInfo` — model usage statistics
- `ErrorEvent` — error reporting

### Foundation Types

- `TurnItem`: structured items within a turn (messages, tool calls/outputs, file changes)
- `UserInput`: user input variants (text, file attachment, slash command)
- `Submission`: wraps an `Op` with a unique ID for correlation
- `RolloutItem`: fully serialized turn events for persistence

---

## 4. Core Orchestration: `codex-core`

This is the largest and most central crate (~80+ source files). It owns config, session state, the turn loop, tool execution, rollout persistence, and most runtime orchestration.

### 4.1 Config System (`core/src/config/`)

The config system is a **layered stack** architecture:

1. Config is loaded by `ConfigBuilder` from multiple sources (CLI args → env vars → `~/.codex/config.toml` → project `.codex/config.toml` → requirements)
2. Layers are merged by `ConfigLayerStack` with increasing priority
3. Config is **constrained** — `.codex/requirements.toml` can restrict what settings users can change

Key types:

- `Config`: runtime-ready resolved config
- `ConfigToml`: the full TOML schema definition
- `ConfigOverrides`: fine-grained CLI overrides via `-c 'key=value'`
- `ManagedFeatures`: wrapped feature flags available at runtime
- `ConfigLockfileToml`: optional lock file enforcing exact config values

### 4.2 Session & Turn Loop (`core/src/session/`)

#### `Session` struct

Holds all mutable state for an active thread:

- `conversation_id` (`ThreadId`)
- `tx_event`: channel for broadcasting events to the client
- `agent_status`: watch channel for agent lifecycle
- `state` (`Mutex<SessionState>`): the full mutable session state
- `mailbox`: async channel for inter-component messaging
- `active_turn`: the currently executing turn (if any)
- `services`: all runtime service references (network proxy, MCP manager, exec server, telemetry, extensions)
- `features`: immutable feature flags for session lifetime

#### `SessionConfiguration`

Immutable-ish snapshot of settings for the session:

- Provider info, collaboration mode, model, reasoning config
- Permission profile and sandbox policy
- cwd, codex_home, thread name
- Environment selections, feature toggles

#### Turn lifecycle

The `turn.rs` module orchestrates each turn:

1. **Context building**: `ContextManager` assembles instructions from personality, skills, connectors, apps, AGENTS.md, and user context
2. **Model session creation**: `ModelClientSession` is created per turn (lazy WebSocket connection)
3. **Submission dispatch**: `submission_loop()` processes `Op`s one at a time
4. **Streaming**: Response is streamed via SSE or WebSocket
5. **Tool processing**: Each tool call (shell, file, MCP, etc.) is intercepted, approved if needed, executed, and results are fed back
6. **Persistence**: Turn data is written to rollout JSONL files

#### `CodexThread`

The public-facing handle for a thread:

- `submit(Op)`: enqueue operations
- `shutdown_and_wait()`: graceful shutdown
- Exposes config snapshots, session telemetry, and thread metadata

#### `ThreadManager`

Manages thread creation and lifecycle:

- `start_thread()`: creates a new thread with config and initial history
- `fork_snapshot()`: creates a forked thread from a prior snapshot
- Handles thread store integration, environment selection, and shell snapshot inheritance

### 4.3 Model Client (`core/src/client.rs`)

`ModelClient` handles all outbound communication with model providers:

- **Lifetime**: Lives for the entire session, holds auth state and transport configuration
- **Per-turn sessions**: `ModelClientSession` is created per turn, caches WebSocket connections
- **Transport**: Supports both HTTP SSE streaming and WebSocket
- **WebSocket prewarm**: Best-effort pre-connection before the first stream request, enabling `previous_response_id` reuse

### 4.4 Rollout Persistence (`core/src/rollout.rs`)

Session history is persisted as **JSONL files** (one JSON object per line):

- `RolloutRecorder`: writes events to JSONL incrementally
- `Cursor`: cursor-based pagination for listing threads
- Supports compaction (history summarization to reduce token usage)

### 4.5 Key Modules

| Module | Purpose |
|---|---|
| `agent/` | Agent status tracking, mailbox, interrupt handling |
| `compact.rs` | History compaction/summarization |
| `exec.rs` | Shell command execution with sandboxing |
| `exec_policy.rs` | Execution policy enforcement |
| `shell.rs` | OS shell interaction |
| `tools/` | Tool implementations (shell, file, network approval, etc.) |
| `unified_exec/` | Modern execution with background terminal support |
| `skills.rs` | Skill loading and prompt injection |
| `web_search.rs` | Web search tool integration |
| `connectors.rs` | External app/service connectors |
| `guardian/` | Safety and approval review system |
| `mcp.rs` / `mcp_tool_call.rs` | MCP tool call processing |
| `state/` | In-memory session state types |
| `turn_diff_tracker.rs` | Tracks file diffs for reversing/undo |

---

## 5. TUI: `codex-tui`

The interactive terminal UI built with **Ratatui**.

### `App` struct (`tui/src/app.rs`)

Central orchestrator:

- Owns all UI state: ChatWidget, HistoryCell, BottomPane, keymap, session state
- Runs the event loop processing key events, app-server notifications, and timers
- Delegates rendering to `render/` modules
- Connects to an in-process or remote app-server

### Key UI Components

| Component | Module | Description |
|---|---|---|
| **ChatWidget** | `chatwidget/` | Main conversation: message bubbles, streaming text, markdown, diffs |
| **BottomPane** | `bottom_pane/` | Input composer, approval popups, status bar, MCP forms |
| **HistoryCell** | `history_cell/` | Session picker for resuming past threads |
| **StatusIndicator** | `status/` | Agent status/activity indicator |
| **DiffRender** | `diff_render.rs` | Inline diff visualization |
| **MarkdownRender** | `markdown_render.rs` | Markdown → ratatui with syntax highlighting |
| **Keymap** | `keymap.rs` | Configurable keybindings (vim, emacs, default presets) |
| **ResumePicker** | `resume_picker.rs` | Thread selection for resumption |
| **ExecCell** | `exec_cell/` | Embedded terminal execution widget |
| **Streaming** | `streaming/` | Real-time text streaming display |

### Connection to Core

The TUI communicates with core **through the app-server protocol**:

- **In-process mode**: `InProcessAppServerClient` communicates via channels
- **Remote mode**: `RemoteAppServerClient` communicates over WebSocket / Unix domain socket

---

## 6. App Server: `codex-app-server`

A JSON-RPC 2.0 server providing a structured API surface.

### Transport

- Unix domain socket (default)
- WebSocket (for remote connections)
- TCP (for debugging)
- In-process channels (for embedded use)

### Request Flow

1. `transport.rs` receives raw JSON-RPC messages
2. `message_processor.rs` deserializes and dispatches by method
3. `request_processors/` contains handlers for each RPC method (thread start/resume/list, turn start/submit, config read/write, mcp management, review, fs, process, account)
4. `thread_state.rs` manages in-memory lifecycle for active threads
5. `config_manager.rs` handles config reads/writes with layer merging

### App Server Protocol (v2)

Organized by domain (`protocol/v2/`):

- Each file corresponds to a resource area (thread, turn, config, account, mcp, fs, realtime, etc.)
- All payloads use camelCase serialization
- Pagination uses cursor-based patterns
- Experimental APIs use `#[experimental(...)]` attribute macros

---

## 7. Authentication: `codex-login`

### Auth Modes

- `CodexAuth::ApiKey`: Direct API key (from file, env var, or inline)
- `CodexAuth::Chatgpt`: OAuth flow via chatgpt.com backend (refresh tokens)
- `CodexAuth::ChatgptAuthTokens`: Token-based auth without refresh
- `CodexAuth::AgentIdentity`: JWT-based identity for automated agents

### AuthManager

- Central coordinator for credential provisioning
- Multiple storage backends (JSON file, OS keychain)
- Automatic token refresh with exponential backoff
- Detects unauthorized responses and triggers re-authentication

### Login Flows

- **Browser OAuth**: Device code flow + local server for callback
- **Access token**: Direct token from stdin or file
- **API key**: OpenAI API key authentication

---

## 8. MCP Integration: `codex-mcp`

### MCP Client

- `McpConnectionManager` manages connections to MCP servers
- Supports stdio and SSE transport
- Handles tool registration, listing, and execution
- Supports OAuth-based MCP authentication

### Codex Apps

Special MCP server for first-party app integrations:

- Connector-based tool discovery
- Auth elicitation flow for app authentication

### Tool Processing

- `McpToolCall` transformer converts MCP tool schemas to internal tool format
- Tool exposure/approval workflows gate which tools are visible
- `McpConnectionManager` handles concurrent tool calls, error propagation, and reconnection

---

## 9. Execution & Sandboxing

### Headless Mode (`codex-exec`)

- `codex exec PROMPT`: run a single prompt, print output, exit
- `codex review`: run a code review on a PR/diff
- Uses in-process app-server client
- Supports JSONL output mode for programmatic consumption

### Sandboxing (`codex-sandboxing`)

Platform-specific sandbox implementations:

| Platform | Mechanism | Implementation |
|---|---|---|
| **macOS** | Seatbelt | `.sbpl` sandbox profiles with App Sandbox-compatible rules |
| **Linux** | Bubblewrap + Landlock | `bwrap` for namespace isolation, `landlock` LSM for FS restrictions |
| **Windows** | Windows Sandbox | Hyper-V-based lightweight VM sandbox |

The `SandboxManager`:

1. Accepts a `PermissionProfile`
2. Applies `policy_transforms` to produce platform-specific sandbox commands
3. Constructs a `SandboxExecRequest` that wraps the target command

### Execution Policies (`codex-execpolicy`)

- Define allowed command prefixes and network access rules
- Project-level `.codex/requirements.toml` can mandate execution constraints
- Warnings and blocks surfaced through the approval system

---

## 10. Persistence & State

| Crate | Storage | Purpose |
|---|---|---|
| **Rollouts** (`core/src/rollout.rs`) | JSONL files | Full turn history (one event per line) |
| **State DB** (`state/`) | SQLite | Mirrored metadata for fast queries (thread listing, search) |
| **Thread Store** (`thread-store/`) | Abstraction | Local filesystem or in-memory thread persistence |
| **Config State** (`config/src/state.rs`) | File system | Config layer tracking and file watchers |
| **Log DB** (`state/src/log_db.rs`) | SQLite | Structured log querying |

---

## 11. Skills System

Skills are Markdown files that provide **contextual instructions** injected into the system prompt.

- **`skills/`**: Skill definitions (Markdown + optional code)
- **`core-skills/`**: Loading, caching, parsing, injecting skill context into prompts
- Skills are triggered via `$SkillName` syntax in user messages
- They can include instructions, environment variable requirements, and config rules

---

## 12. Feature Flags: `codex-features`

A feature flag system with three stages: **Dev**, **Experimental**, **Stable**.

Flags can be toggled via:

- CLI: `runnly --enable feature_name`
- Config: `features.feature_name = true`
- Environment: `CODEX_FEATURE_feature_name = true`

---

## 13. Data Flow: End-to-End

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER                                       │
│  (TUI / IDE Extension / `codex exec` / MCP client)                 │
└────────────────────────┬────────────────────────────────────────────┘
                         │ JSON-RPC (in-process / WebSocket / UDS)
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    App Server (codex-app-server)                     │
│  message_processor → request_processors → thread_state              │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Op/Event protocol
                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Core Session Loop                            │
│                                                                      │
│  submission_loop {                                                    │
│    ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐     │
│    │ Receive Op   │───▶│ Build Context│───▶│ Create Turn      │     │
│    │ (user input) │    │ (instructions,│    │ (ModelSession)   │     │
│    └──────────────┘    │ skills, files,│    └────────┬─────────┘     │
│                        │ permissions)  │             │               │
│                        └──────────────┘             ▼               │
│                                        ┌──────────────────┐         │
│                                        │ Stream Model      │         │
│                                        │ Response          │         │
│                                        │ (SSE/WebSocket)   │         │
│                                        └────────┬─────────┘         │
│                                                  │                   │
│                                  ┌───────────────┴───────────────┐  │
│                                  │     Process Events             │  │
│                                  │  ┌─────────┐ ┌────────────┐   │  │
│                                  │  │Content  │ │Tool Call   │   │  │
│                                  │  │Streaming│ │ (shell,     │   │  │
│                                  │  │         │ │  file, MCP) │   │  │
│                                  │  └─────────┘ └──────┬─────┘   │  │
│                                  │                      │         │  │
│                                  │             ┌────────▼──────┐ │  │
│                                  │             │ Sandbox +     │ │  │
│                                  │             │ Execute +     │ │  │
│                                  │             │ Approve       │ │  │
│                                  │             └───────────────┘ │  │
│                                  └────────────────────────────────┘  │
│                                                                      │
│  } → Event stream to client                                         │
│  → Persist to rollout (JSONL)                                       │
│  → Mirror to state DB (SQLite)                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 14. Key Architectural Patterns

### Constrained Config

Config values can be wrapped in `Constrained<T>`, enforcing project- or admin-level restrictions on what users can change.

### Layered Config Stack

Config is loaded from multiple sources in priority order: `defaults < user config < project config < project requirements < CLI overrides`.

### SQ/EQ Pattern

The Submission Queue / Event Queue pattern decouples clients from the agent's turn loop. Clients submit `Op`s and receive `Event`s asynchronously.

### In-Process / Remote Transparency

The app-server client abstraction (`InProcessAppServerClient` vs `RemoteAppServerClient`) allows the TUI, exec mode, and IDE extension to use the same protocol with transparent in-process or remote communication.

### Platform-Specific Sandboxing

The `SandboxManager` abstracts platform differences behind a unified `PermissionProfile` → `SandboxExecRequest` transformation, with platform-specific implementations in dedicated crates.
