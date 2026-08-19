# 03 — Cargo Workspace and Every Crate

This document explains how Rust's workspace system works, then tours every
crate in this project — what it does, what it depends on, and why it exists
as a separate crate rather than a module.

---

## Table of Contents

1. [What is a Cargo Workspace?](#1-what-is-a-cargo-workspace)
2. [The workspace Cargo.toml](#2-the-workspace-cargotoml)
3. [Why Split into Many Crates?](#3-why-split-into-many-crates)
4. [Crate Map — Every Crate Explained](#4-crate-map--every-crate-explained)
5. [The Dependency Graph](#5-the-dependency-graph)
6. [How Optional Crates Work](#6-how-optional-crates-work)
7. [The Root Crate — `zeroclawlabs`](#7-the-root-crate--zeroclawlabs)
8. [Build Scripts (`build.rs`)](#8-build-scripts-buildrs)
9. [The `xtask` Pattern](#9-the-xtask-pattern)

---

## 1. What is a Cargo Workspace?

A **workspace** is a collection of Rust crates that:
- Share a single `Cargo.lock` file (deterministic dependency versions)
- Can reference each other via `path = "..."` dependencies
- Declare common version, edition, and metadata in one place

Without a workspace, each crate would have its own lockfile and might resolve
the same dependency to different versions — causing duplicate code in the
final binary.

```
zeroclaw/          ← workspace root
├── Cargo.toml     ← declares [workspace] members
├── Cargo.lock     ← shared lockfile for ALL members
├── src/           ← root crate (zeroclawlabs)
└── crates/
    ├── zeroclaw-api/
    │   └── Cargo.toml   ← member crate
    ├── zeroclaw-config/
    │   └── Cargo.toml   ← member crate
    └── ...
```

---

## 2. The workspace Cargo.toml

```toml
[workspace]
members = [
    ".",                            # root crate (zeroclawlabs)
    "crates/zeroclaw-api",
    "crates/zeroclaw-infra",
    "crates/zeroclaw-config",
    "crates/zeroclaw-log",
    "crates/zeroclaw-spawn",        # NEW: sanctioned tokio::spawn wrapper
    "crates/zeroclaw-commands",     # NEW: shared slash-command catalogue
    "crates/zeroclaw-providers",
    "crates/zeroclaw-memory",
    "crates/zeroclaw-channels",
    "crates/zeroclaw-tools",
    "crates/zeroclaw-sop-graph",    # NEW: SOP Blueprint graph types
    "crates/zeroclaw-runtime",
    "crates/zeroclaw-eval",         # NEW: agent evaluation harness
    "apps/zerocode",                # NEW: interactive TUI client (replaces zeroclaw-tui)
    "crates/zeroclaw-plugins",
    "crates/zeroclaw-gateway",
    "crates/zeroclaw-hardware",
    "crates/zeroclaw-tool-call-parser",
    "crates/robot-kit",
    "crates/aardvark-sys",
    "crates/zeroclaw-macros",
    "apps/tauri",
    "tools/fill-translations",
    "xtask",
]
resolver = "2"   # use the v2 feature resolver (required for workspace feature unification)
```

### `[workspace.package]` — shared metadata

```toml
[workspace.package]
version = "0.8.4"
edition = "2024"        # Rust 2024 edition features
license = "MIT OR Apache-2.0"
repository = "https://github.com/zeroclaw-labs/zeroclaw"
rust-version = "1.96.0"  # minimum supported Rust version (MSRV)
```

Every member crate can inherit these with `version.workspace = true` instead
of repeating the version string.

### `[workspace.dependencies]` — version pinning

```toml
[workspace.dependencies]
zeroclaw-api = { path = "crates/zeroclaw-api", version = "0.8.4" }
tokio = { version = "1.50", default-features = false, ... }
serde = { version = "1.0", default-features = false, features = ["derive"] }
```

Member crates reference these with `dep.workspace = true`:

```toml
# crates/zeroclaw-providers/Cargo.toml
[dependencies]
zeroclaw-api.workspace = true   # inherits path + version from workspace
tokio.workspace = true          # inherits version + features
```

This ensures every crate uses the **same version** of every shared dependency.
Without workspace dependencies, two crates might pull in `serde 1.0.200` and
`serde 1.0.195` — two different compiled versions in the same binary.

---

## 3. Why Split into Many Crates?

### Reason 1 — Incremental compilation

Rust compiles crates in parallel. If you change `zeroclaw-channels/src/telegram.rs`,
only `zeroclaw-channels` and the root crate need to recompile. All other
crates are untouched. In a monolithic crate, any change recompiles everything.

### Reason 2 — Enforced boundaries

If `zeroclaw-providers` and `zeroclaw-channels` are separate crates, it is
*impossible* for a channel to import a provider directly. The dependency graph
is the enforcer. In a single crate with modules, you could accidentally add
`use crate::providers::openai::...` in a channel file.

### Reason 3 — Optional compilation

Crates can be optional dependencies. When building without the gateway,
`zeroclaw-gateway` is never compiled — its code does not exist in the binary.
This is harder to achieve with modules.

### Reason 4 — Independent versioning

`zeroclaw-config` is `Beta` stability; `zeroclaw-runtime` is `Experimental`.
Different version bump policies can be applied per crate.

---

## 4. Crate Map — Every Crate Explained

### `zeroclaw-api` — The Contract Layer

**Path**: `crates/zeroclaw-api/`
**Stability**: Experimental → Stable at v1.0
**Dependencies**: only standard library + a few tiny crates (async-trait, serde, tokio task-locals)

This is the most important crate. It defines:
- `ModelProvider` trait — how the runtime talks to LLMs
- `Channel` trait — how the runtime talks to messaging platforms
- `Tool` trait — what a callable agent tool looks like
- `Memory` trait — how conversation memory is stored
- `Observer` trait — how metrics/traces are emitted
- `RuntimeAdapter` trait — execution environment abstraction
- `Peripheral` trait — hardware board abstraction
- Shared data types: `ChatMessage`, `ChannelMessage`, `ToolResult`, etc.

**Why separate?** Every other crate depends on this one. If implementations
(`zeroclaw-providers`, `zeroclaw-channels`) imported each other, you would get
circular dependencies. `zeroclaw-api` has no implementation crate imports —
it is the dependency graph's root.

---

### `zeroclaw-config` — Configuration and Secrets

**Path**: `crates/zeroclaw-config/`
**Stability**: Beta → Stable at v0.8
**Key files**:
- `schema.rs` — the full `Config` struct (every TOML key)
- `secrets.rs` — `SecretStore` (encrypted credential storage)
- `policy.rs` — `SecurityPolicy` type
- `paths.rs` — where config files live on each OS
- `validation_warnings.rs` — config validation logic

This crate knows the exact shape of `config.toml`. It loads, merges, and
validates configuration, handles environment variable overrides, and provides
runtime config patching (the `set_prop` API used by the gateway).

**Why separate?** Config is needed by almost every crate. By isolating it,
changes to the TOML schema do not require touching the runtime or providers.

---

### `zeroclaw-log` — Unified Log Emission

**Path**: `crates/zeroclaw-log/`
**Stability**: Beta
**Key files**:
- `event.rs` — `LogEvent` schema (OTel/ECS hybrid)
- `writer.rs` — JSONL persistence
- `broadcast.rs` — process-wide broadcast channel for the dashboard
- `subscriber.rs` — global `tracing` subscriber installation

Every module in the project logs through the `record!` macro from this crate.
One `record!` call does three things simultaneously:
1. Emits a `tracing` event (visible in `RUST_LOG` output)
2. Appends to `state/runtime-trace.jsonl` (persistent log)
3. Broadcasts to the WebSocket dashboard

**Why separate?** Logging is a cross-cutting concern. Isolating it prevents
`tracing` from leaking into every crate's public API.

---

### `zeroclaw-macros` — Compile-Time Code Generation

**Path**: `crates/zeroclaw-macros/`
**Stability**: Beta
**Type**: `proc-macro` crate

Provides the `Configurable` derive macro. See document 05 for full details.

**Why separate?** Proc-macro crates must be compiled for the *host* (the
machine running `cargo build`), not the target. They cannot be mixed with
regular code. Cargo enforces this separation.

---

### `zeroclaw-providers` — LLM Backends

**Path**: `crates/zeroclaw-providers/`
**Stability**: Beta
**Channel implementations**:
- `anthropic.rs` — Claude API
- `openai.rs` — OpenAI API (GPT-4, o1, etc.)
- `gemini.rs` — Google Gemini
- `ollama.rs` — local Ollama models
- `azure_openai.rs` — Azure-hosted OpenAI
- `bedrock.rs` — AWS Bedrock
- `compatible.rs` — any OpenAI-compatible endpoint
- `reliable.rs` — retry + rate-limit wrapper around any provider
- `router.rs` — routes across multiple providers

Every file exports a struct that implements `ModelProvider` from `zeroclaw-api`.

---

### `zeroclaw-channels` — Messaging Integrations

**Path**: `crates/zeroclaw-channels/`
**Stability**: Experimental
**30+ integrations**: Telegram, Slack, Discord, WhatsApp, WeChat, Mattermost,
Matrix, IRC, email, Nostr, Bluesky, Reddit, Twitter/X, LINE, Signal, ...

Also contains:
- `orchestrator/` — channel lifecycle manager, routes messages to agents,
  manages concurrency

---

### `zeroclaw-tools` — Agent Capabilities

**Path**: `crates/zeroclaw-tools/`
**Stability**: Experimental

Built-in tools the agent can call:
- Shell command execution
- File read/write/list
- Web fetch and browser automation
- Memory store/retrieve
- Cron scheduling
- Image generation

Each tool implements `Tool` from `zeroclaw-api`.

---

### `zeroclaw-runtime` — The Agent Brain

**Path**: `crates/zeroclaw-runtime/`
**Stability**: Experimental
**Subdirectories**:

| Dir | Purpose |
|-----|---------|
| `agent/` | Agent struct, builder, turn lifecycle |
| `agent/loop_.rs` | The main agent loop (message → tools → response) |
| `agent/turn/execution.rs` | `ResolvedAgentExecution` — stable per-agent execution bundle |
| `agent/personality.rs` | Personality file loading (SOUL.md, IDENTITY.md, USER.md…) |
| `agent/personality_templates/` | Dashboard starter templates for personality files |
| `agent/memory_strategy.rs` | Memory loading strategy selector |
| `security/` | Policy enforcement, sandboxing, audit logging |
| `cron/` | Scheduled task runner |
| `skills/` | Agent skill loading and execution |
| `sop/` | Standard Operating Procedures |
| `sop/procedural_memory.rs` | Epic F: safe SOP self-improvement via proposals |
| `tools/scoped.rs` | `ScopedToolRegistry` — unified policy-gated tool assembly |
| `observability/otel.rs` | OpenTelemetry OTLP export with content policy |
| `hooks/` | Lifecycle hooks (before/after turn) |
| `onboard/` | First-run setup wizard |
| `rag/` | Retrieval-augmented generation pipeline |

---

### `zeroclaw-memory` — Persistence Backends

**Path**: `crates/zeroclaw-memory/`
**Stability**: Beta
**Backends**:
- `markdown.rs` — human-readable `.md` files
- `sqlite.rs` — SQLite database
- `qdrant.rs` — vector database for semantic search
- `postgres.rs` — PostgreSQL (feature-gated)
- `none.rs` — no-op (for testing / memory-free deployments)

Also includes:
- `embeddings.rs` — generate text embeddings for vector search
- `consolidation.rs` — merge and summarise old memories
- `decay.rs` — time-based importance decay
- `retrieval.rs` — configurable retrieval pipeline

Memory Epic A additions (durable store seam):
- `budget.rs` — eviction engine (count + byte caps, pinned entries skipped)
- `dedup.rs` — deduplication gate (Jaccard threshold, reject/merge actions)
- `policy_gate.rs` — write-time policy validation (namespace + category limits)
- `merge.rs` — conflict-free memory merge for multi-agent shared stores

New `MemoryEntry` fields: `superseded_by` (soft-delete via CAS chain), `pinned`
(exempt from budget eviction).

---

### `zeroclaw-gateway` — HTTP / WebSocket Gateway

**Path**: `crates/zeroclaw-gateway/`
**Stability**: Experimental

Provides:
- REST API for config management, memory access, session listing
- WebSocket endpoint for live log streaming
- Webhook receiver for external platforms (Slack events API, etc.)
- Pairing mechanism for first-time device setup

---

### `zeroclaw-infra` — Shared Infrastructure

**Path**: `crates/zeroclaw-infra/`
**Stability**: Beta

Utilities used by multiple crates that don't belong in `zeroclaw-api`:
- `debounce.rs` — rate-limited event emission
- `session.rs` — session key construction
- Stall watchdog — detects hung agent turns

---

### `apps/zerocode` — Interactive TUI Client

**Path**: `apps/zerocode/`
**Stability**: Experimental

A standalone interactive terminal client built with `ratatui` + `crossterm`.
`zerocode` speaks JSON-RPC to a running ZeroClaw daemon — it does **not**
embed the agent runtime. It provides:

- Chat interface for agent turns
- Config management panes
- Log streaming
- SOP pane (experimental, behind `sop-authoring` feature)

Because it is a pure RPC client, `zeroclaw_spawn::spawn!` (the daemon-path
sanctioned interface) is intentionally **not** used here — the pane noted
in `src/main.rs` is a standalone TUI process.

This replaces the former `crates/zeroclaw-tui` onboarding wizard.

---

### `zeroclaw-spawn` — Sanctioned Task Spawner

**Path**: `crates/zeroclaw-spawn/`
**Stability**: Beta

Provides the `spawn!` macro — the only sanctioned way to spawn Tokio tasks on
daemon-path code. Every `spawn!` call:
- Emits a structured `runtime.task.spawn` log record at the call site
- Propagates the current tracing span into the child task
- Instruments the task body with a child span

This ensures every background task carries attribution (which module spawned
it, on which file + line) in the runtime trace log.

```rust
// crates/zeroclaw-spawn/src/lib.rs
zeroclaw_spawn::spawn!(async move {
    // all spawns emit a lifecycle record and carry the parent's trace span
    channel.listen(handler).await;
});
```

> **Rule**: Production daemon code MUST use `zeroclaw_spawn::spawn!` instead
> of bare `tokio::spawn`. The standalone `apps/zerocode` TUI client is the
> documented exception.

---

### `zeroclaw-commands` — Slash Command Catalogue

**Path**: `crates/zeroclaw-commands/`
**Stability**: Beta

Defines the canonical set of built-in slash commands (`/help`, `/clear`,
`/new`, `/stop`, …) as typed `BuiltinCommandId` variants and the
`CommandSurface` enum (CLI, Web, TUI, Channel). This is the single source of
truth so every surface (CLI, gateway, zerocode TUI, channels) advertises the
same command identities.

---

### `zeroclaw-sop-graph` — SOP Blueprint Graph Types

**Path**: `crates/zeroclaw-sop-graph/`
**Stability**: Beta

Pure serde projection types for the SOP Blueprint graph wire shape. The
runtime builds these, the gateway exports their JSON Schema, and the zerocode
TUI deserialises them off the RPC stream. By living in a tiny shared crate
with no heavy deps, all three surfaces read one wire shape without pulling
in each other's dependencies.

---

### `zeroclaw-eval` — Agent Evaluation Harness

**Path**: `crates/zeroclaw-eval/`
**Stability**: Experimental

Phase 0 of the evaluation framework: deterministic replay of captured LLM
trace fixtures through the real agent loop. Key modules:
- `case.rs` — test case definition
- `replay.rs` — fixture playback engine
- `grader.rs` — correctness checks
- `runner.rs` — batch runner
- `report.rs` — structured result reports

The harness lets CI verify that the agent loop handles specific LLM response
patterns correctly without making live API calls.

---

### `zeroclaw-hardware` — USB and GPIO

**Path**: `crates/zeroclaw-hardware/`
**Stability**: Experimental

USB device discovery, serial port communication, and GPIO abstractions for
embedded peripherals (sensors, actuators connected to a ZeroClaw agent).

---

### `zeroclaw-plugins` — WASM Plugin System

**Path**: `crates/zeroclaw-plugins/`
**Stability**: Experimental

Foundation for loading third-party tools and channels as WebAssembly modules.
Planned to replace the compiled-in channel and tool crates at v1.0.

Recent additions:
- `wasm_channel.rs` \u2014 WASM channel host bindings (inbound queue, config jail)
- `wasm_memory.rs` \u2014 WASM memory host bindings
- `host.rs` \u2014 registration API for plugin channel/tool discovery

---

### `zeroclaw-tool-call-parser` — LLM Output Parsing

**Path**: `crates/zeroclaw-tool-call-parser/`
**Stability**: Beta → Stable at v0.8

Parses the raw text output from LLMs that use XML/JSON function-calling
formats. Handles partial tool calls (streaming), malformed JSON, and
provider-specific quirks.

---

### `robot-kit` — Robotics Abstractions

**Path**: `crates/robot-kit/`
**Stability**: Experimental

High-level abstractions for robot control (motor controllers, sensors)
built on top of `zeroclaw-hardware` and `zeroclaw-api`'s `Peripheral` trait.

---

### `aardvark-sys` — Total Phase Hardware SDK

**Path**: `crates/aardvark-sys/`
**Stability**: Experimental

Low-level FFI bindings to the Aardvark I2C/SPI/GPIO USB adapter SDK from
Total Phase. Provides raw `unsafe` C interop; higher-level Rust abstractions
live in `zeroclaw-hardware`.

---

### `apps/tauri` — Desktop App

**Path**: `apps/tauri/`
**Stability**: Experimental

A Tauri desktop companion that wraps the web dashboard in a native window.
Key modules:
- `daemon.rs` — finds or spawns the `zeroclaw daemon` process
- `gateway_client.rs` — typed HTTP+WebSocket client for the gateway API
- `health.rs` — background health poller driving the tray icon state
- `tray/` — system tray icon, menu, and status icons (idle/working/error)
- `state.rs` — shared `AppState` (gateway URL, connection status)
- `capabilities/` — macOS-specific permissions (AppleScript, Screenshot)
- `splash/index.html` — loading screen shown while daemon starts

The app is a thin native shell: it does **not** embed the agent runtime.
It reuses the gateway running on port 42617 whether started by it or by
the CLI session running in the same terminal.

---

### `tools/fill-translations` — Dev Tooling

**Path**: `tools/fill-translations/`

A CLI binary that scans source files for `fl!()` Fluent translation calls and
auto-fills missing keys in translation files.

---

### `xtask` — Build Automation

**Path**: `xtask/`

A Cargo workspace member that acts as a build scripting tool. Running
`cargo xtask <command>` executes Rust code instead of shell scripts for
complex build automation tasks.

---

## 5. The Dependency Graph

```
aardvark-sys     (no workspace deps)
robot-kit        (depends on: zeroclaw-api, zeroclaw-hardware)
zeroclaw-api     (no workspace deps)
zeroclaw-macros  (no workspace deps — proc-macro)
zeroclaw-commands (no workspace deps)
zeroclaw-sop-graph (no workspace deps)
              ↑
zeroclaw-infra      (api)
zeroclaw-config     (api, macros)
zeroclaw-log        (api, config)
zeroclaw-spawn      (log)          # NEW
zeroclaw-tool-call-parser  (api)
              ↑
zeroclaw-providers  (api, config, log)
zeroclaw-memory     (api, config, log)
zeroclaw-channels   (api, config, log, infra, tool-call-parser)
zeroclaw-tools      (api, config, log, memory)
zeroclaw-hardware   (api, config, log)
              ↑
zeroclaw-runtime    (api, config, log, spawn, providers, memory, channels, tools, infra, hardware, tool-call-parser, sop-graph)
zeroclaw-eval       (api, runtime)  # NEW
zeroclaw-gateway    (api, config, log, runtime)
apps/zerocode       (standalone TUI — RPC client, no runtime dep)
apps/tauri          (standalone desktop shell, no runtime dep)
              ↑
zeroclawlabs (root) (all of the above)
```

---

## 6. How Optional Crates Work

Some crates are `optional = true` in the root `Cargo.toml`:

```toml
zeroclaw-channels = { workspace = true, optional = true }
zeroclaw-runtime  = { workspace = true, optional = true }
zeroclaw-eval     = { workspace = true, optional = true }
zeroclaw-gateway  = { workspace = true, optional = true }
```

They are activated by feature flags:

```toml
[features]
default = ["agent-runtime"]
agent-runtime = [
    "dep:zeroclaw-channels",
    "dep:zeroclaw-runtime",
    ...
]
gateway = ["dep:zeroclaw-gateway"]
```

> **Note**: The former `zeroclaw-tui` crate has been replaced by
> `apps/zerocode`. The TUI is now a standalone binary that connects
> to the daemon via JSON-RPC rather than being an optional compiled-in
> feature.

In source code, optional crates are guarded:

```rust
// src/lib.rs
#[cfg(feature = "agent-runtime")]
pub mod agent;

#[cfg(feature = "gateway")]
pub mod gateway;
```

If you compile without the `agent-runtime` feature, the `agent` module does
not exist in the binary. The compiler verifies that no code outside the
`#[cfg]` block references it.

---

## 7. The Root Crate — `zeroclawlabs`

The root crate (`src/`) is the *integration layer*. It:
- Defines the CLI using `clap` (commands, subcommands, flags)
- Wires together all the crates at startup
- Routes CLI commands to the appropriate subsystem
- Handles the `main` function

It deliberately contains minimal logic. The actual work is in the library
crates.

---

## 8. Build Scripts (`build.rs`)

```rust
// build.rs (root)
fn main() {
    println!("cargo:rerun-if-changed=build.rs");
}
```

This is a no-op build script. The single `rerun-if-changed` instruction tells
Cargo to only re-run this script when `build.rs` itself changes — without it,
Cargo would re-run on every build.

More complex crates use `build.rs` for:
- Generating code from `.proto` files (protobuf)
- Embedding compile-time constants (git hash, build date)
- Linking native C libraries

---

## 9. The `xtask` Pattern

```toml
# Cargo.toml
[[bin]]
# xtask is a workspace member with a [[bin]]
```

`cargo xtask <cmd>` is a community convention for writing build scripts in
Rust instead of bash. Because `xtask` is a workspace member, it can import
other workspace crates. For example, `cargo xtask generate-schema` could
import `zeroclaw-config` and generate a JSON Schema file at build time.

This is better than shell scripts because:
- Cross-platform (works on macOS, Linux, Windows)
- Fully type-checked
- Access to the same libraries as the main project
