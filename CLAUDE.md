# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Grok Build?

**Grok Build** (the `grok` CLI) is SpaceXAI's terminal-based AI coding agent — a Rust TUI that understands codebases, edits files, executes shell commands, searches the web, and manages long-running tasks. This repo is a periodic sync from the SpaceXAI monorepo; `SOURCE_REV` records the monorepo commit SHA.

## Build & Test Commands

```sh
# Prerequisites: Rust toolchain (pinned in rust-toolchain.toml), DotSlash on PATH, protoc

# Build + launch the TUI (fast dev iteration)
cargo run -p xai-grok-pager-bin

# Release binary (output: target/release/xai-grok-pager)
cargo build -p xai-grok-pager-bin --release

# Fast validation (always target specific crates, not --workspace)
cargo check -p xai-grok-pager-bin
cargo check -p xai-grok-shell

# Per-crate tests
cargo test -p xai-grok-config
cargo test -p xai-grok-workspace
cargo test -p xai-grok-tools
cargo test -p xai-grok-sampler

# Linting (config in clippy.toml)
cargo clippy -p <crate>

# Formatting (config in rustfmt.toml)
cargo fmt --all
```

> ⚠️ Full `--workspace` builds are slow (~85 crates). Always target specific packages with `-p`.

## Key Architecture

### Workspace layout (~85 crates)

```
crates/
├── codegen/                          # Core crate closure (~60 crates)
│   ├── xai-grok-pager-bin/           # Binary entrypoint (main.rs only — composition root)
│   ├── xai-grok-pager/               # TUI: scrollback, prompt, modals, ratatui rendering
│   ├── xai-grok-pager-minimal/       # Minimal pager (auth, plan, overlay, todo panels)
│   ├── xai-grok-pager-render/        # TUI rendering primitives
│   ├── xai-grok-shell/               # Agent runtime + leader/stdio/headless entry points
│   ├── xai-grok-shell-base/          # Shell abstractions, prompt, terminal bridge
│   ├── xai-grok-tools/               # Tool implementations (file edit, search, bash, MCP, ...)
│   ├── xai-grok-tools-api/           # Tool API types and schemas
│   ├── xai-grok-agent/               # Agent loop, prompt building, system reminders, compaction
│   ├── xai-grok-workspace/           # Host filesystem, VCS, execution, checkpoints, permissions
│   ├── xai-grok-workspace-types/     # Workspace type definitions
│   ├── xai-grok-workspace-client/    # Workspace client SDK
│   ├── xai-grok-mcp/                 # MCP client (OAuth, server management, HTTP transport)
│   ├── xai-grok-config/              # Configuration loading, validation, signed policy, overrides
│   ├── xai-grok-config-types/        # Config type definitions
│   ├── xai-grok-sampler/             # LLM sampling: streaming, retries, HTTP client, metrics
│   ├── xai-grok-sampling-types/      # Sampling request/response types
│   ├── xai-grok-models/              # Model definitions and registry
│   ├── xai-grok-memory/              # Memory backend: embedding, chunking, storage, search
│   ├── xai-grok-auth/                # Authentication flow (OAuth, tokens)
│   ├── xai-codebase-graph/           # Code analysis knowledge graph
│   ├── xai-grok-markdown/            # Markdown rendering (user messages, docs)
│   ├── xai-grok-markdown-core/       # Markdown core parsing
│   ├── xai-grok-mermaid/             # Mermaid diagram rendering
│   ├── xai-grok-hooks/               # Hook system (PreToolUse, PostToolUse, Stop)
│   ├── xai-grok-shared/              # Shared utilities across crates
│   ├── xai-grok-env/                 # Environment variable management
│   ├── xai-grok-http/                # HTTP client utilities
│   ├── xai-grok-telemetry/           # Telemetry and tracing
│   ├── xai-grok-plugin-marketplace/  # Plugin marketplace
│   ├── xai-grok-sandbox/             # Sandbox execution environment
│   ├── xai-grok-subagent-resolution/ # Subagent resolution
│   ├── xai-grok-announcements/       # Feature announcements
│   ├── xai-grok-version/             # Version info
│   ├── xai-grok-update/              # Auto-update mechanism
│   ├── xai-grok-voice/               # Voice input support
│   ├── xai-chat-state/               # Chat state management
│   ├── xai-acp-lib/                  # Agent Client Protocol (ACP) library
│   ├── xai-agent-lifecycle/          # Agent lifecycle management
│   ├── xai-hooks-plugins-types/      # Hooks/plugins type definitions
│   ├── xai-hunk-tracker/             # Diff hunk tracking
│   ├── xai-fast-worktree/            # Fast git worktree operations
│   ├── xai-fsnotify/                 # Filesystem notification
│   ├── xai-file-utils/               # File utility functions
│   ├── xai-gix-status/               # Git status (gitoxide-based)
│   ├── xai-tty-utils/                # TTY utilities
│   ├── xai-token-estimation/         # Token counting/estimation
│   ├── xai-crash-handler/            # Crash handler & panic reporting
│   ├── xai-system-power/             # System power management
│   ├── xai-mixpanel/                 # Mixpanel analytics client
│   ├── xai-prompt-queue/             # Prompt queuing
│   ├── xai-ratatui-inline/           # Ratatui inline patches
│   ├── xai-ratatui-textarea/         # Ratatui textarea widget
│   ├── xai-sqlite-journal/           # SQLite journal implementation
│   ├── xai-tracing-macros/           # Tracing proc macros
│   ├── xai-test-support/             # Test utilities and mocks
│   ├── ptyctl/                       # PTY control library
│   └── ptyctl-cli/                   # PTY control CLI
├── common/                           # Shared leaf crates
│   ├── xai-tool-types/               # Tool type definitions (task, schema, metadata)
│   ├── xai-tool-runtime/             # Tool execution runtime
│   ├── xai-tool-protocol/            # Tool protocol definitions
│   ├── xai-computer-hub-core/        # Computer use hub core
│   ├── xai-computer-hub-sdk/         # Computer use SDK
│   ├── xai-computer-hub-mcp-adapter/ # Computer use MCP adapter
│   ├── xai-circuit-breaker/          # Circuit breaker pattern
│   ├── xai-interjection-core/        # Interjection system
│   ├── xai-grok-compaction/          # Message compaction
│   ├── xai-tracing/                  # Tracing infrastructure
│   └── xai-test-utils/               # Common test utilities
├── build/
│   └── xai-proto-build/              # Protobuf codegen build helpers
prod/mc/                               # Production microservices
└── third_party/                       # Vendored dependencies
    ├── mermaid-to-svg/               # Mermaid → SVG rendering (Rust port)
    ├── dagre_rust/                   # Dagre graph layout (Rust port)
    ├── graphlib_rust/                # Graph library (Rust port)
    └── ordered_hashmap/              # Ordered hashmap implementation
```

### Data flow (high level)

1. **`xai-grok-pager-bin`** starts the binary and composes all subsystems
2. **`xai-grok-pager`** renders the TUI and handles keyboard/shell input
3. Input feeds into **`xai-grok-shell`** (agent runtime) which manages sessions
4. **`xai-grok-agent`** builds prompts from context, chat state, system reminders, and user messages
5. **`xai-grok-sampler`** streams LLM responses via HTTP
6. Tool calls are dispatched through **`xai-grok-tools`** (file edits, bash, search, MCP servers)
7. **`xai-grok-workspace`** handles filesystem operations, VCS, checkpoints, and sandboxing
8. **`xai-grok-mcp`** provides MCP server connectivity for external tools
9. **`xai-grok-memory`** manages long-term memory with embedding, chunking, and search

### Key patterns

- **Composition root pattern**: `xai-grok-pager-bin` is the sole binary package that wires everything together
- **Per-crate focus**: Most crates are single-purpose and small (~85 crates for high modularity)
- **Generated root Cargo.toml**: The workspace root is auto-generated — edit per-crate `Cargo.toml` files instead
- **Third-party ports**: `third_party/` contains Rust ports of the Mermaid diagram stack (mermaid-to-svg, dagre_rust, graphlib_rust)

### Important constraints

- Toolchain is pinned in `rust-toolchain.toml` (currently 1.92.0)
- Root `Cargo.toml` is **generated** — treat as read-only
- DotSlash (`bin/protoc` and other hermetics) must be on `PATH` before building
- Windows builds are best-effort (not CI-tested from this tree)
- `std::fs::canonicalize` is banned — use `dunce::canonicalize` instead
