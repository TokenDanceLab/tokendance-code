# TokenDanceCode

> Local command-line coding agent · Rust development preview

![TokenDanceCode CLI](docs/images/tokendance-cli-hero.svg)

[中文](README.md) · [AgentHub SDK](docs/agenthub-sdk.md) · [Release readiness](docs/release-readiness.md)

---

## What Is TokenDanceCode

TokenDanceCode is a local command-line coding agent for personal repositories, currently in Rust development preview. The CLI single-turn command, interactive REPL and Rust AgentHub SDK adapter use `MockProvider` to return simulated responses. Setting an API key does not yet make these entry points call a model or carry out coding tasks from prompts.

The core library contains the following modules. See [current status](docs/rust-rewrite-status.md) for integration limits:

- 📖 **Read code** — search files, regex matching, glob patterns
- ✏️ **Edit files** — exact-string replacement, full-file write
- ⚡ **Run commands** — PowerShell tool with permission-gated execution
- 🔄 **Manage sessions** — JSONL transcript, session resume, context compaction
- 🤖 **Extend capabilities** — MCP tool protocol, subagent orchestration

The command tool primarily targets Windows / PowerShell. The Rust SDK provides AgentHub event mapping and approval interfaces; model-backed execution integration remains unfinished.

## Current implementation

`crates/` contains the active Rust implementation. `packages/core`, `packages/sdk` and the legacy TypeScript CLI source remain as interface references; their presence does not establish feature parity between two runtimes.

`packages/cli/bin/tokendance.js` is an npm launcher for the Rust binary. It looks for local `target/release/tokendance` and `target/debug/tokendance`, then tries a platform-native package. It exits if no binary is found; there is no TypeScript fallback. Native-package build and distribution work remains open, so run from source for now.

## Quick Start

### Run from source

A Rust toolchain is required. The `run` command below demonstrates a simulated session and needs no API key.

```bash
git clone https://github.com/TokenDanceLab/tokendance-code.git
cd tokendance-code
cargo run -p tokendance-cli -- --version     # tokendance 0.3.0-rs.0
cargo run -p tokendance-cli -- doctor --json  # health check
cargo run -p tokendance-cli -- run "hello"    # simulated response
cargo test --workspace
```

### Interactive REPL

```bash
cargo run -p tokendance-cli                   # enter interactive mode
tokendance> read the main.rs file
tokendance> summarize this repo
tokendance> /help
tokendance> /exit
```

## Core library modules

The following sections describe library implementations. Mock runs through the CLI/REPL or SDK adapter do not verify end-to-end model, MCP, subagent or OS sandbox integration.

### 🔧 Built-in Tools (7)

| Tool | Risk | Description |
|------|------|-------------|
| `read_file` | Read | Read workspace files with path safety checks |
| `write_file` | Write | Write files, permission-gated |
| `edit_file` | Write | Exact-string replace (`old_string` → `new_string`), `replace_all` support |
| `glob` | Read | File pattern matching, sorted by modification time |
| `grep` | Read | Regex search with content/files_with_matches/count modes |
| `run_powershell` | Shell | Execute PowerShell commands, destructive commands hard-denied |
| `echo` | Read | Test echo tool |

### 🌐 Provider Transports (3)

These HTTP transports live in `crates/tokendance-core/src/providers/` and are not yet wired into the CLI/REPL or Rust SDK adapter execution entry points. The variables below configure the transports; they do not replace those entry points' `MockProvider`.

| Provider | Auth | Transport Gate |
|----------|------|----------------|
| OpenAI Chat Completions | `TOKENDANCE_GATEWAY_API_KEY` → `OPENAI_API_KEY` | `TOKENDANCE_GATEWAY_HTTP_TRANSPORT=1` |
| OpenAI Responses | `TOKENDANCE_OPENAI_API_KEY` → `OPENAI_API_KEY` | `TOKENDANCE_OPENAI_TRANSPORT=1` |
| Anthropic Messages | `TOKENDANCE_ANTHROPIC_API_KEY` → `ANTHROPIC_API_KEY` | `TOKENDANCE_ANTHROPIC_TRANSPORT=1` |

All providers automatically redact API keys from error messages.

### 🛡️ Security

- **4 permission modes**: Default → Safe → Auto → Yolo
- **Tool risk classification**: Read / Write / Shell / Network / Dangerous
- **Path safety**: Workspace path normalization, blocks directory traversal and symlink escape
- **Destructive command interception**: PowerShell `Remove-Item`, `format-volume` hard-denied
- **Sandboxing abstraction**: Policy types and argument generation for Windows restricted tokens / macOS Seatbelt / Linux bwrap. Windows remains a stub and Linux availability reports unsupported; this is not enforced OS process isolation.

### 🔌 Extensibility

| System | Description |
|--------|-------------|
| **MCP Client** | stdio JSON-RPC protocol, dynamic tool discovery, `mcp__{server}__{tool}` namespacing |
| **Subagent** | Isolated session + restricted tool set + recursion prevention, multi-agent orchestration |
| **Hooks** | PreToolUse / PostToolUse / TurnCompleted / TurnFailed lifecycle hooks |
| **Memory** | Markdown file persistent memory, cross-session context |
| **Instruction Discovery** | Auto-discover AGENTS.md / CLAUDE.md, Global → Project → Local scope hierarchy |

### 📡 Sessions & Streaming

- **JSONL Transcript** — append-only, crash-recoverable, UUID parent chain
- **SSE Streaming** — incremental buffering, multi-line data, comment skipping
- **StreamEvent** — ContentDelta / ToolStarted / TurnCompleted real-time events
- **Session Resume** — replay transcript to restore full message history
- **Context Compaction** — auto-compress old messages above threshold

## Project Structure

```
tokendance-code/
├── crates/                          # Rust implementation
│   ├── tokendance-core/             #   Runtime, Provider, Tools, Permissions
│   ├── tokendance-sdk/              #   AgentHub SDK facade
│   └── tokendance-cli/              #   CLI binary
├── packages/                        # npm packages and legacy TypeScript source
│   ├── core/                        #   Legacy runtime interface reference
│   ├── sdk/                         #   Legacy AgentHub SDK interface reference
│   ├── cli/                         #   Rust npm wrapper + legacy CLI source
│   └── agenthub-example/            #   Integration example (private)
├── docs/                            # Shared documentation
├── scripts/                         # Verification & release scripts
├── Cargo.toml                       # Rust workspace
├── package.json                     # npm workspace
└── README.md
```

## CLI Commands

These commands use a built Rust binary; `run` and the REPL return simulated responses. To run from source, prefix the arguments with `cargo run -p tokendance-cli --`.

```bash
tokendance                              # Interactive REPL
tokendance run "summarize this repo"    # Single turn
tokendance run --json "hello"           # Structured JSON output
tokendance run --stream-json "hello"    # Streaming JSONL output
tokendance doctor --json                # Health check
tokendance config validate --json       # Config validation
tokendance sessions list                # List sessions
tokendance transcript search "needle"   # Search transcripts
tokendance quality                      # Quality overview
```

REPL slash commands: `/help` `/status` `/exit` `/compact`

## AgentHub SDK interface

The current Rust facade lives in `crates/tokendance-sdk`; its adapter execution entry point still uses `MockProvider`. The [AgentHub SDK docs](docs/agenthub-sdk.md) retain the TypeScript API and integration contract. Those examples do not establish Rust model-backed execution or native-package publication.

## Rust modules

| Module | Function |
|--------|----------|
| `config` | Settings loading / validation / merge |
| `permissions` | 4-mode permission engine |
| `provider` | ModelProvider trait + MockProvider |
| `providers/*` | OpenAI Chat / Responses / Anthropic HTTP transports |
| `runtime` | Agent loop + streaming + hooks integration |
| `tools` | 7 built-in tools + ToolExposure + MCP registration |
| `transcript` | JSONL append / session resume |
| `streaming` | SSE incremental parser |
| `context` | Instruction discovery (AGENTS.md / CLAUDE.md) |
| `memory` | Persistent memory CRUD |
| `hooks` | Lifecycle hooks |
| `mcp` | MCP client (stdio JSON-RPC) |
| `subagent` | Subagent spawning / isolation |
| `compact` | Context compaction |
| `sandbox` | Cross-platform sandboxing abstraction |
| `worktree` | Git worktree management |
| `types` | Core types + RuntimeEvent |

## Verification

The full test suite executes PowerShell. Windows uses `powershell.exe`; Linux/macOS require PowerShell 7's `pwsh` on `PATH`.

```bash
# Rust gates
cargo fmt --all -- --check
cargo test --workspace
cargo run -p tokendance-cli -- doctor

# Root script also runs Cargo fmt + workspace tests
pnpm verify

# Release readiness
node scripts/check-rust-release-plan.mjs
node scripts/smoke-rust-release.mjs
```

## Documentation

| Document | Content |
|----------|---------|
| [Architecture](docs/rust-rewrite-architecture.md) | Rust rewrite architecture — 20 module design decisions |
| [Status](docs/rust-rewrite-status.md) | Module implementation, entry-point limits and release work |
| [Tool Reference](docs/rust-tool-reference.md) | 7 built-in tools input/output schemas |
| [Release Checklist](docs/rust-release-checklist.md) | Release owner pre-publish checklist |
| [AgentHub SDK](docs/agenthub-sdk.md) | SDK integration, event sink, approval bridge |
| [Release Readiness](docs/release-readiness.md) | npm publish gates and registry status |
| [Architecture Comparison](docs/架构对标评估.md) | Claude Code / Codex / OpenCode comparison |

## License

MIT
