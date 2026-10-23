# TokenDanceCode

> 本地命令行 Coding Agent · Rust 开发预览
> Local command-line coding agent · Rust development preview

![TokenDanceCode CLI](docs/images/tokendance-cli-hero.svg)

[English](README.en.md) · [AgentHub SDK](docs/agenthub-sdk.md) · [发布准备](docs/release-readiness.md)

---

## 什么是 TokenDanceCode

TokenDanceCode 是面向个人代码仓库的本地命令行 Coding Agent，目前处于 Rust 开发预览阶段。CLI 的单次运行、交互式 REPL 和 Rust AgentHub SDK adapter 使用 `MockProvider` 返回模拟响应；配置 API key 尚不能让这些入口调用模型或按提示完成代码任务。

核心库已有以下模块，接入进度见[当前状态](docs/rust-rewrite-status.md)：

- 📖 **阅读代码** — 搜索文件、正则匹配、Glob 模式
- ✏️ **修改文件** — 精确字符串替换、整文件写入
- ⚡ **执行命令** — PowerShell 工具，受权限系统保护
- 🔄 **管理会话** — JSONL transcript、session resume、上下文压缩
- 🤖 **扩展能力** — MCP 工具协议、Subagent 编排

命令执行工具以 Windows / PowerShell 为主要目标；Rust SDK 提供 AgentHub 事件映射与审批接口，真实模型执行接入仍待完成。

## 当前实现

`crates/` 是当前 Rust 实现。`packages/core`、`packages/sdk` 和旧 TypeScript CLI 源码保留作接口参考，不代表两套运行时已经功能对齐。

`packages/cli/bin/tokendance.js` 是 Rust binary 的 npm 启动包装：查找本地 `target/release/tokendance`、`target/debug/tokendance`，再尝试平台原生包；找不到时退出，不会回退到 TypeScript。平台原生包的构建与分发仍在准备，当前从源码运行。

## 快速开始

### 从源码运行

需要 Rust 工具链；以下 `run` 命令演示模拟会话，无需 API key。

```bash
git clone https://github.com/TokenDanceLab/tokendance-code.git
cd tokendance-code
cargo run -p tokendance-cli -- --version     # tokendance 0.3.0-rs.0
cargo run -p tokendance-cli -- doctor --json  # 健康检查
cargo run -p tokendance-cli -- run "hello"    # 模拟响应
cargo test --workspace
```

### 交互式 REPL

```bash
cargo run -p tokendance-cli                   # 进入交互模式
tokendance> read the main.rs file
tokendance> summarize this repo
tokendance> /help
tokendance> /exit
```

## 核心库模块

以下清单描述库内实现。CLI/REPL 与 SDK adapter 的模拟运行不能作为真实模型、MCP、Subagent 或系统沙箱已端到端接通的验收。

### 🔧 内置工具 (7)

| 工具 | 风险 | 说明 |
|------|------|------|
| `read_file` | Read | 读取工作区文件，自动路径安全检查 |
| `write_file` | Write | 写入文件，受权限门控 |
| `edit_file` | Write | 精确字符串替换（`old_string` → `new_string`），支持 `replace_all` |
| `glob` | Read | 文件模式匹配，按修改时间排序 |
| `grep` | Read | 正则搜索，支持 content/files_with_matches/count 三种模式 |
| `run_powershell` | Shell | 执行 PowerShell 命令，破坏性命令硬拦截 |
| `echo` | Read | 测试用回显工具 |

### 🌐 Provider 传输 (3)

这些 HTTP transport 位于 `crates/tokendance-core/src/providers/`，尚未接入 CLI/REPL 和 Rust SDK adapter 的运行入口。下表环境变量用于对应 transport，不会切换入口的 `MockProvider`。

| Provider | 认证 | 传输门控 |
|----------|------|----------|
| OpenAI Chat Completions | `TOKENDANCE_GATEWAY_API_KEY` → `OPENAI_API_KEY` | `TOKENDANCE_GATEWAY_HTTP_TRANSPORT=1` |
| OpenAI Responses | `TOKENDANCE_OPENAI_API_KEY` → `OPENAI_API_KEY` | `TOKENDANCE_OPENAI_TRANSPORT=1` |
| Anthropic Messages | `TOKENDANCE_ANTHROPIC_API_KEY` → `ANTHROPIC_API_KEY` | `TOKENDANCE_ANTHROPIC_TRANSPORT=1` |

所有 Provider 自动遮盖 API key，错误信息中不泄露凭证。

### 🛡️ 安全体系

- **4 级权限模式**: Default → Safe → Auto → Yolo
- **工具风险分类**: Read / Write / Shell / Network / Dangerous
- **路径安全**: 工作区路径归一化，阻止目录穿越和 symlink 逃逸
- **破坏性命令拦截**: PowerShell `Remove-Item`、`format-volume` 等硬拒绝
- **Sandboxing 抽象**: Windows restricted token / macOS Seatbelt / Linux bwrap 的策略类型与参数生成；Windows 仍为 stub，Linux availability 返回不支持，不能视为已生效的系统进程隔离

### 🔌 扩展能力

| 系统 | 说明 |
|------|------|
| **MCP Client** | stdio JSON-RPC 协议，动态工具发现，`mcp__{server}__{tool}` 命名空间 |
| **Subagent** | 独立 session + 受限工具集 + 递归防护，支持多 Agent 编排 |
| **Hooks** | PreToolUse / PostToolUse / TurnCompleted / TurnFailed 生命周期钩子 |
| **Memory** | Markdown 文件持久化记忆，跨 session 保持上下文 |
| **Instruction Discovery** | 自动发现 AGENTS.md / CLAUDE.md，Global → Project → Local 三层覆盖 |

### 📡 会话与流式

- **JSONL Transcript** — 仅追加，崩溃可恢复，UUID 父子链
- **SSE 流式解析** — 增量缓冲，多行数据，注释跳过
- **StreamEvent** — ContentDelta / ToolStarted / TurnCompleted 实时事件
- **Session Resume** — 从 transcript 重放恢复完整消息历史
- **Context Compaction** — 超阈值自动压缩旧消息

## 项目结构

```
tokendance-code/
├── crates/                          # Rust 实现
│   ├── tokendance-core/             #   Runtime, Provider, Tools, Permissions
│   ├── tokendance-sdk/              #   AgentHub SDK facade
│   └── tokendance-cli/              #   CLI binary
├── packages/                        # npm 包与旧 TypeScript 源码
│   ├── core/                        #   旧 Runtime 接口参考
│   ├── sdk/                         #   旧 AgentHub SDK 接口参考
│   ├── cli/                         #   Rust npm wrapper + 旧 CLI 源码
│   └── agenthub-example/            #   接入示例 (private)
├── docs/                            # 共享文档
├── scripts/                         # 验证 & 发布脚本
├── Cargo.toml                       # Rust workspace
├── package.json                     # npm workspace
└── README.md
```

## CLI 命令

以下命令使用已构建的 Rust binary；`run` 和 REPL 返回模拟响应。直接从源码运行时，在命令参数前加 `cargo run -p tokendance-cli --`。

```bash
tokendance                              # 交互式 REPL
tokendance run "summarize this repo"    # 单次运行
tokendance run --json "hello"           # 结构化 JSON 输出
tokendance run --stream-json "hello"    # 流式 JSONL 输出
tokendance doctor --json                # 健康检查
tokendance config validate --json       # 配置校验
tokendance sessions list                # 列出会话
tokendance transcript search "needle"   # 搜索 transcript
tokendance quality                      # 质量概览
```

REPL 内置 slash commands: `/help` `/status` `/exit` `/compact`

## SDK 接口（AgentHub）

当前 Rust facade 位于 `crates/tokendance-sdk`，adapter 的执行入口仍使用 `MockProvider`。[AgentHub SDK 文档](docs/agenthub-sdk.md)保留 TypeScript 接口与集成契约；其中示例不代表 Rust adapter 已完成真实模型接入或原生包发布。

## Rust 模块清单

| 模块 | 功能 |
|------|------|
| `config` | Settings 加载 / 验证 / 合并 |
| `permissions` | 4-mode 权限引擎 |
| `provider` | ModelProvider trait + MockProvider |
| `providers/*` | OpenAI Chat / Responses / Anthropic HTTP 传输 |
| `runtime` | Agent loop + streaming + hooks 集成 |
| `tools` | 7 内置工具 + ToolExposure + MCP 注册 |
| `transcript` | JSONL 追加 / session resume |
| `streaming` | SSE 增量解析器 |
| `context` | Instruction 发现 (AGENTS.md / CLAUDE.md) |
| `memory` | 持久化记忆 CRUD |
| `hooks` | 生命周期钩子 |
| `mcp` | MCP client (stdio JSON-RPC) |
| `subagent` | Subagent 生成 / 隔离 |
| `compact` | 上下文压缩 |
| `sandbox` | 跨平台沙箱抽象 |
| `worktree` | Git worktree 管理 |
| `types` | 核心类型 + RuntimeEvent |

## 验证

完整测试会执行 PowerShell。Windows 使用 `powershell.exe`；Linux/macOS 需要把 PowerShell 7 的 `pwsh` 加入 `PATH`。

```bash
# Rust 验证
cargo fmt --all -- --check
cargo test --workspace
cargo run -p tokendance-cli -- doctor

# 根脚本同样运行 Cargo fmt + workspace tests
pnpm verify

# 发布准备
node scripts/check-rust-release-plan.mjs
node scripts/smoke-rust-release.mjs
```

## 文档

| 文档 | 内容 |
|------|------|
| [架构设计](docs/rust-rewrite-architecture.md) | Rust 重写架构：20 个模块的设计决策 |
| [当前状态](docs/rust-rewrite-status.md) | 模块实现、入口接入限制与发布待办 |
| [工具参考](docs/rust-tool-reference.md) | 7 个内置工具的输入/输出 schema |
| [发布清单](docs/rust-release-checklist.md) | Release owner 发布前检查项 |
| [AgentHub SDK](docs/agenthub-sdk.md) | SDK 接入、event sink、approval bridge |
| [发布准备](docs/release-readiness.md) | npm 发布门禁和 registry 状态 |
| [架构对标](docs/架构对标评估.md) | Claude Code / Codex / OpenCode 对标分析 |

## License

MIT
