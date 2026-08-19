# 02 — Project Architecture

This document explains the **overall design of ZeroClaw**: why it is
structured the way it is, what every major subsystem does, and the key
engineering decisions made along the way.

---

## Table of Contents

1. [One-Line Pitch](#1-one-line-pitch)
2. [The Microkernel Architecture](#2-the-microkernel-architecture)
3. [Dependency Direction Rule](#3-dependency-direction-rule)
4. [The Single Source of Truth Rule](#4-the-single-source-of-truth-rule)
5. [Binary Layout — What Gets Compiled](#5-binary-layout--what-gets-compiled)
6. [The Stability Tier System](#6-the-stability-tier-system)
7. [Data Flow: A Message Through the System](#7-data-flow-a-message-through-the-system)
8. [Concurrency Model](#8-concurrency-model)
9. [Security Model](#9-security-model)
10. [`ScopedToolRegistry` — Unified Tool Assembly](#10-the-scopedtoolregistry--unified-tool-assembly)
11. [Extensibility Points](#11-extensibility-points)
12. [Ingress Policy — Universal Turn Classification](#12-ingress-policy--universal-turn-classification)
13. [Desktop Companion — Tauri App](#13-desktop-companion--tauri-app)
14. [What Tradeoffs Were Made](#14-what-tradeoffs-were-made)

---

## 1. One-Line Pitch

ZeroClaw is a **single Rust binary** that:
- Listens on any number of messaging channels simultaneously
- Routes each message to an AI agent that uses the configured LLM
- Executes tools (shell, file system, web, memory) under a security policy
- Persists memory between conversations
- Exposes a WebSocket / webhook gateway for external integrations

---

## 2. The Microkernel Architecture

ZeroClaw is built around a **microkernel** pattern borrowed from OS design:

```
┌─────────────────────────────────────────┐
│               zeroclaw binary           │
│                                         │
│  ┌───────────┐   ┌───────────────────┐  │
│  │  CLI /    │   │  Agent Runtime    │  │
│  │  Gateway  │   │  (microkernel)    │  │
│  └─────┬─────┘   └────────┬──────────┘  │
│        │                  │             │
│  ┌─────▼──────────────────▼──────────┐  │
│  │         zeroclaw-api               │  │
│  │   (trait definitions only)         │  │
│  └────────────────────────────────────┘  │
│                                         │
│  Plugins (crates):                      │
│  zeroclaw-providers  zeroclaw-channels  │
│  zeroclaw-tools      zeroclaw-memory    │
│  zeroclaw-hardware   zeroclaw-gateway   │
└─────────────────────────────────────────┘
```

The **microkernel** (`zeroclaw-runtime`) only knows about the *interfaces*
(traits) defined in `zeroclaw-api`. It does not know which LLM or which
messaging platform is in use. The concrete implementations are registered
at startup via factory functions.

### Why microkernel?

In a monolithic design, the runtime would import OpenAI's HTTP client,
Telegram's bot API, and SQLite directly. Adding a new LLM would require
modifying the runtime. In a microkernel design:

- Adding a new LLM = adding a new file in `zeroclaw-providers`
- Adding a new channel = adding a new file in `zeroclaw-channels`
- The runtime never changes

---

## 3. Dependency Direction Rule

All crates have a strict one-way dependency flow:

```
zeroclaw-api        (no deps on other workspace crates)
     ↑
zeroclaw-config     (depends on api)
zeroclaw-log        (depends on api)
     ↑
zeroclaw-providers  (depends on api, config, log)
zeroclaw-memory     (depends on api, config, log)
zeroclaw-channels   (depends on api, config, log)
zeroclaw-tools      (depends on api, config, log)
     ↑
zeroclaw-runtime    (depends on all the above)
     ↑
zeroclaw binary     (depends on runtime + direct crates for CLI)
```

**No crate ever imports another at the same level or a lower level.**
`zeroclaw-providers` never imports `zeroclaw-channels`. This prevents
circular dependencies and keeps each crate testable in isolation.

---

## 4. The Single Source of Truth Rule

This rule is described in `AGENTS.md` and is the most important architectural
constraint in the project:

> **No piece of state lives in two places. Ever.**

### What this prevents

The pre-microkernel version of the project had a bug where:
1. Channel handles stored a cached `Vec<String>` of `allowed_users`
2. The truth lived in `Config`
3. When the operator reloaded config, the cached Vec was not updated
4. Users who were de-authorised could still send messages until restart

### How it is enforced today

Instead of caching config values in channel structs, channels receive a
*resolver closure*:

```rust
// Pattern — channel receives a closure, not a value
pub struct TelegramChannel {
    bot: Bot,
    // NOT: allowed_users: Vec<String>  ← forbidden duplicate
    allowed_users: Arc<dyn Fn() -> Vec<String> + Send + Sync>,
    //             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //             closure that reads live config on every call
}

// At startup:
let config = Arc::new(RwLock::new(loaded_config));
let channel = TelegramChannel {
    bot: Bot::new(token),
    allowed_users: Arc::new({
        let cfg = config.clone();
        move || cfg.read().channels.telegram.allowed_users.clone()
    }),
};
```

The runtime can reload config at any time by calling
`config.write().reload()`. Every subsequent call to `allowed_users()` will
return the new list — with no cache invalidation needed.

---

## 5. Binary Layout — What Gets Compiled

The project produces two binaries from `Cargo.toml`:

```toml
[[bin]]
name = "zeroclaw"
path = "src/main.rs"

[[bin]]
name = "zeroclaw-acp-bridge"
path = "src/bin/zeroclaw-acp-bridge.rs"
required-features = ["acp-bridge"]
```

Plus a library:

```toml
[lib]
name = "zeroclaw"
path = "src/lib.rs"
```

The library exposes the public API so that integration tests and the gateway
can import it without re-parsing `main.rs`.

### Feature flags control what is compiled

```toml
[features]
default = ["agent-runtime"]

agent-runtime = [
    "dep:zeroclaw-channels",
    "dep:zeroclaw-tools",
    "dep:zeroclaw-runtime",
    "dep:zeroclaw-tui",
    "dep:zeroclaw-hardware",
    ...
]
gateway = ["dep:zeroclaw-gateway"]
```

A minimal headless deployment can be compiled with:
```bash
cargo build --no-default-features --features gateway
```
This produces a smaller binary without the TUI, hardware, and agent-runtime
code — useful for server deployments that only serve webhook requests.

---

## 6. The Stability Tier System

ZeroClaw assigns each crate a *stability tier*:

| Tier | Breaking change policy |
|------|----------------------|
| **Stable** | Covered by semver — no breaking changes without major bump |
| **Beta** | Breaking changes allowed in MINOR versions with changelog note |
| **Experimental** | No stability guarantee |

Current tiers:

| Crate | Tier | Why |
|-------|------|-----|
| `zeroclaw-api` | Experimental → Stable at v1.0 | Trait changes affect all implementations |
| `zeroclaw-config` | Beta → Stable at v0.8 | Config schema changes affect operators |
| `zeroclaw-log` | Beta | Stable log format needed for tooling |
| `zeroclaw-runtime` | Experimental | Agent loop evolves rapidly |
| `zeroclaw-channels` | Experimental | 30+ channels, all evolving |
| `zeroclaw-gateway` | Experimental | Separate binary at v0.9 |
| `zeroclaw-macros` | Beta | Tightly coupled to config schema |

### Why this matters

When you read a crate and wonder "can I depend on this API?", the stability
tier gives you the answer. Experimental APIs can change in any patch release.

---

## 7. Data Flow: A Message Through the System

Here is the complete path from a Telegram message to an AI response:

```
User sends: "What's the weather in Paris?"
       │
       ▼
[1] TelegramChannel::listen()
       │  receives ChannelMessage { content: "...", sender: "alice" }
       ▼
[2] Channel Orchestrator
       │  checks allowed_users, deduplicates, routes to agent
       ▼
[3] AgentLoop::process_message()
       │  builds system prompt + conversation history
       │  calls ModelProvider::chat()
       ▼
[4] OpenAI / Anthropic / Ollama provider
       │  HTTP call to LLM API
       │  LLM responds: "I'll check the weather"
       │  + tool_call: weather_tool({ "city": "Paris" })
       ▼
[5] AgentLoop — tool execution
       │  SecurityPolicy::check() — is weather_tool allowed?
       │  Tool::execute({ "city": "Paris" })
       │  returns: "Paris: 22°C, partly cloudy"
       ▼
[6] AgentLoop — second LLM call
       │  conversation now includes tool result
       │  LLM responds: "The weather in Paris is 22°C..."
       ▼
[7] TelegramChannel::send()
       │  sends reply to the user's chat
       ▼
User receives: "The weather in Paris is 22°C, partly cloudy."
```

Each numbered step is a module boundary. Steps 3–6 repeat in a loop until
the LLM produces a response with no more tool calls (or a limit is hit).

---

## 8. Concurrency Model

ZeroClaw is fully async. The Tokio runtime provides a thread pool (one thread
per CPU core by default).

```
Thread pool: [T1] [T2] [T3] [T4]  (e.g. 8 threads on 8-core CPU)
                  │
         Tokio task scheduler
        /         |         \
[Telegram  [Slack      [Cron
 listener]  listener]   runner]
    │           │           │
 Agent      Agent       Agent
 turn 1     turn 2      turn 3
```

Each channel listener is a long-running Tokio task. Each incoming message
spawns a new task for the agent turn. Many agent turns can run concurrently
on the thread pool.

### Why this is safe without data races

The compiler enforces `Send + Sync` bounds on everything that crosses task
boundaries. A type that is not `Send` (e.g., one containing a raw pointer or
a non-thread-safe reference) cannot be sent to another task — this is caught
at compile time, not at runtime.

---

## 9. Security Model

ZeroClaw's security model has five layers:

### Layer 1 — Autonomy Level

Configured by the operator in `config.toml`:

```toml
[security]
autonomy = "supervised"  # or "full" or "manual"
```

- `manual` — every tool call requires operator approval
- `supervised` — tools run automatically, but certain actions are restricted
- `full` — the agent can do anything (dangerous — for trusted environments only)

### Layer 2 — Workspace Boundary

The agent can only read/write files within a configured workspace directory.
Any attempt to access `/etc/passwd` or `~/.ssh` is blocked by policy.

### Layer 3 — Allowlist / Denylist

Operators can restrict which tools are available to the agent, and which
shell commands can be run.

### Layer 4 — OS-Level Sandboxing

On Linux, ZeroClaw can run tool execution in a sandbox:
- **Firejail** — filesystem isolation via namespaces
- **Bubblewrap** — low-privilege container
- **Landlock** — kernel-enforced path restrictions (Linux 5.13+)

On macOS:
- **Seatbelt (sandbox-exec)** — Apple's sandbox profiles

### Layer 5 — Prompt Injection Defence

```rust
// crates/zeroclaw-runtime/src/security/prompt_guard.rs
pub struct PromptGuard { ... }
pub enum GuardAction { Allow, Block, Sanitize }
```

Every message and tool result is scanned for prompt injection attempts
before it enters the LLM context. The `DomainMatcher` checks whether
content claims to be from a different source than it actually came from.

---

## 10. The `ScopedToolRegistry` — Unified Tool Assembly

Before this seam existed, the per-agent tool set was assembled by hand at six
different call sites (channels orchestrator, `run`, `process_message`,
`Agent::from_config`, the gateway, and the delegate builder). Each site
re-applied security filtering itself — a recipe for sites drifting out of sync
and for the gateway's `/api/tools` listing to disagree with what a real turn
actually saw.

`ScopedToolRegistry::assemble` is the single seam that now mints every
production tool set. It applies, in order:

1. Peripheral tools (hardware boards, if any)
2. Built-in `allowed_tools`/`excluded_tools` policy filter
3. ACP memory-strip (removes memory tools on ACP fast-boot paths)
4. Per-agent MCP server scoping (`mcp_bundles` — omission is not a grant)
5. MCP pinned-resources section injection
6. Skill tool registration under the same `SecurityPolicy`

The newtype `ScopedToolRegistry` has a private inner field — production code
can only obtain one through `assemble`. Handing the engine an unfiltered tool
list is a compile error once all call sites have been migrated (P1 cut-over).

```rust
// crates/zeroclaw-runtime/src/tools/scoped.rs
pub struct ScopedToolRegistry(Vec<Box<dyn Tool>>);

impl ScopedToolRegistry {
    pub fn assemble(assembly: ScopedAssembly) -> Self { ... }
    pub fn into_inner(self) -> Vec<Box<dyn Tool>> { ... }

    // Test-only escape hatch — no other way to build one in production
    #[cfg(test)]
    pub fn from_raw_for_test(tools: Vec<Box<dyn Tool>>) -> Self { ... }
}
```

---

## 11. Extensibility Points

The eight official extension points and what each one is:

| Trait | Crate | Add a new… |
|-------|-------|-----------|
| `ModelProvider` | `zeroclaw-api` | LLM backend |
| `Channel` | `zeroclaw-api` | Messaging platform |
| `Tool` | `zeroclaw-api` | Agent capability |
| `Memory` | `zeroclaw-api` | Persistence backend |
| `Observer` | `zeroclaw-api` | Metrics / tracing sink |
| `RuntimeAdapter` | `zeroclaw-api` | Execution environment |
| `Peripheral` | `zeroclaw-api` | Hardware board |
| WASM plugin | `zeroclaw-plugins` | Sandboxed extension |

At v1.0, channels and tools will migrate to WASM plugins (via `zeroclaw-plugins`),
allowing third-party extensions without recompiling the binary.

---

## 12. Ingress Policy — Universal Turn Classification

The `ingress.rs` module in `zeroclaw-api` provides a typed contract for
classifying where every agent turn originates. Before this existed, origin
knowledge was scattered as ad-hoc booleans (`is_cron: bool`,
`headless: bool`). Now every turn is stamped with:

| Type | Purpose |
|------|---------|
| `TurnOrigin` | `Interactive`, `Channel`, `Cron`, `Daemon`, `AgentDirect`, `SubTurn` |
| `SourceClass` | `External` (channel peer) vs `Internal` (cron, SOP, subagent) |
| `Transport` | `Channel { kind, alias }`, `Gateway`, `Acp`, `Rpc`, `Internal` |
| `TrustClass` | `Trusted` vs `Untrusted` |

```rust
// crates/zeroclaw-api/src/ingress.rs
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TurnOrigin {
    Interactive,
    Channel,
    Cron,
    Daemon,
    AgentDirect,
    #[default]
    SubTurn,  // fail-closed default — sub-turns never get origin-gated behaviour
}
```

The `#[default]` on `SubTurn` is a deliberate security choice: an unstamped
or legacy envelope behaves like a nested sub-turn (least-privileged), so
new surfaces cannot accidentally inherit behaviours gated on `Interactive`
or `Channel` origins.

This is "Phase 1" — stamping happens at ingest but enforcement is per-feature.
Full per-transport peer resolution is planned for Phase 2.

---

## 13. Desktop Companion — Tauri App

`apps/tauri/` is a **Tauri desktop application** that wraps the ZeroClaw web
dashboard in a native window on macOS, Windows, and Linux.

Architecture:

```
apps/tauri/
├── src/daemon.rs        ← finds / spawns the zeroclaw daemon
├── src/gateway_client.rs← typed HTTP/WS client for the gateway API
├── src/health.rs        ← background health poller
├── src/tray/            ← system tray icon + menu
├── src/state.rs         ← shared Tauri app state (gateway URL, status)
└── splash/index.html    ← loading splash while daemon starts
```

The Tauri app does **not** embed the agent runtime. It is a thin native
shell that:
1. Checks whether a daemon is already running on port 42617
2. If not, finds the `zeroclaw` binary and spawns `zeroclaw daemon`
3. Shows a splash screen while the daemon starts
4. Opens the web dashboard (`http://localhost:42617/`) in a WebView
5. Keeps a system tray icon with connection status

This keeps the desktop app lightweight and allows it to share the same
daemon as a CLI session running in the same terminal.

---

## 14. What Tradeoffs Were Made

### Trait objects vs generics

The runtime uses `Box<dyn Tool>` (dynamic dispatch) instead of monomorphised
generics. This means:
- **Pro**: Tools can be loaded at runtime from a config file
- **Pro**: The runtime does not need to know about every Tool type at compile
  time
- **Con**: Slight overhead from vtable dispatch (~1–5 ns per call, irrelevant
  compared to LLM API latency of 100–2000 ms)

### Single binary vs microservices

ZeroClaw compiles everything into one binary instead of separate services for
the gateway, agent, and channels.
- **Pro**: Zero network overhead between subsystems
- **Pro**: Single deployment artefact — `scp zeroclaw server:` and run
- **Con**: One crash takes down all channels (mitigated by supervisor scripts
  and health checks)

### Tokio vs other async runtimes

Tokio was chosen because it is the most mature, well-documented async runtime
for Rust, with the best ecosystem compatibility (reqwest, sqlx, and almost
every async Rust library targets Tokio).
- **Alternative**: `async-std` — smaller but less ecosystem support
- **Alternative**: `smol` — minimal but less documentation

### `anyhow` vs typed errors

Most public functions return `anyhow::Result<T>` (an erased error type)
rather than custom error enums. This prioritises developer velocity over
machine-parseable error codes. For errors that cross API boundaries (e.g.,
the gateway's HTTP API), typed error codes are used via `ConfigApiError`.
