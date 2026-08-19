# 10 — Frameworks and Dependencies

A complete catalogue of every significant external dependency in ZeroClaw,
why each was chosen, what it does, and what the alternatives are.

---

## Table of Contents

1. [Async Runtime](#1-async-runtime)
2. [HTTP Client](#2-http-client)
3. [HTTP Server (Gateway)](#3-http-server-gateway)
4. [Serialization](#4-serialization)
5. [CLI Framework](#5-cli-framework)
6. [Config and Schema](#6-config-and-schema)
7. [Error Handling](#7-error-handling)
8. [Cryptography and Security](#8-cryptography-and-security)
9. [Database and Persistence](#9-database-and-persistence)
10. [Concurrency Utilities](#10-concurrency-utilities)
11. [Memory and Vector Search](#11-memory-and-vector-search)
12. [Channels and Protocols](#12-channels-and-protocols)
13. [UI, TUI, and Desktop](#13-ui-tui-and-desktop)
14. [Tooling and Observability](#14-tooling-and-observability)
15. [Hardware and Embedded](#15-hardware-and-embedded)
16. [Build and Dev Tools](#16-build-and-dev-tools)
17. [Dependency Philosophy](#17-dependency-philosophy)

---

## 1. Async Runtime

### `tokio` — v1.50

The foundation of ZeroClaw's concurrency model.

```toml
tokio = { version = "1.50", default-features = false, features = [
    "rt-multi-thread",  # thread pool for parallelism
    "macros",           # #[tokio::main], #[tokio::test]
    "time",             # tokio::time::sleep, Interval
    "net",              # TCP/UDP sockets
    "io-util",          # AsyncReadExt, AsyncWriteExt
    "sync",             # Mutex, RwLock, channel
    "process",          # Child process spawning (shell tool)
    "io-std",           # stdin/stdout async
    "fs",               # async file I/O
    "signal",           # SIGINT/SIGTERM handling
] }
```

**Why Tokio?**
- The de-facto standard Rust async runtime — 95%+ of async Rust crates
  target Tokio
- `rt-multi-thread` schedules tasks across CPU cores automatically
- `CancellationToken` (from `tokio-util`) is exactly what ZeroClaw needs for
  per-turn cancellation

**Alternatives**: `async-std` (less ecosystem support), `smol` (lightweight but
less mature), bare `futures` executor (minimal but no scheduler).

**Why `default-features = false`?** Binary size. Tokio includes many features
(tracing integration, statistics, test utilities) that are not needed in
production. Explicit feature selection shaves ~200KB from the binary.

### `tokio-util` — v0.7

Provides `CancellationToken`, `codec` (framed streams), and utilities not
in the main `tokio` crate.

```rust
use tokio_util::sync::CancellationToken;
let token = CancellationToken::new();
// Clone and pass to subtasks — cancelling parent cancels all children
```

### `tokio-stream` — v0.1

Bridges Tokio's async I/O with the `Stream` trait, used for processing
LLM streaming responses token by token.

### `futures-util` — v0.3

Utilities for `Future` and `Stream` composition. Used for `sink` support
in WebSocket channels.

---

## 2. HTTP Client

### `reqwest` — v0.12

```toml
reqwest = { version = "0.12", default-features = false, features = [
    "json",         # .json() body serialization/deserialization
    "rustls-tls",   # TLS via rustls (pure Rust, no system OpenSSL)
    "blocking",     # Synchronous API (used in a few places)
    "multipart",    # File uploads
    "stream",       # Streaming response bodies
    "socks",        # SOCKS5 proxy support
] }
```

**Why Reqwest?**
- Ergonomic builder API
- Async by default, backed by hyper
- `rustls-tls` avoids the `openssl` system dependency (no more "linking
  against system OpenSSL" problems on Alpine Linux / embedded targets)

**Why `rustls-tls` not `native-tls`?**
- `native-tls` uses the OS TLS library (SecureTransport on macOS,
  SChannel on Windows, OpenSSL on Linux) — creates a system dependency
- `rustls` is pure Rust — compiles everywhere, reproducible builds,
  no FIPS/licensing concerns

**Alternatives**: `hyper` (lower-level, more control), `ureq` (sync-only,
minimal), `surf` (async-std based).

---

## 3. HTTP Server (Gateway)

### `axum` — v0.8

```toml
axum = { version = "0.8", features = [
    "http1",   # HTTP/1.1
    "json",    # Json extractor/response
    "tokio",   # Tokio integration
    "query",   # Query string extractors
    "ws",      # WebSocket upgrade
    "macros",  # #[debug_handler]
] }
```

**Why Axum?**
- Tower-native: all middleware is a `Tower::Service`, which is the standard
  Rust web middleware interface
- Compile-time type safety for route handlers (unlike Express.js)
- Zero-cost abstraction — no runtime dispatch in hot paths
- Developed by the Tokio team alongside Tokio itself

**Alternatives**: `actix-web` (actor model, heavier), `warp` (combinators,
less ergonomic), `rocket` (synchronous, different model).

### `hyper` + `hyper-util` + `tower` + `tower-http`

The lower-level HTTP stack that axum builds on:
- `hyper`: raw HTTP/1.1 protocol
- `hyper-util`: `TcpIncoming`, graceful shutdown
- `tower`: `Service` trait, middleware chaining
- `tower-http`: `RequestBodyLimitLayer`, `TimeoutLayer`

---

## 4. Serialization

### `serde` — v1.0

```toml
serde = { version = "1.0", features = ["derive"] }
```

The backbone of Rust serialization. The `#[derive(Serialize, Deserialize)]`
proc macro generates all marshalling code at compile time — zero runtime
overhead.

**Used for**: Config files (TOML), API request/response bodies (JSON),
database rows, memory entries, log events, everything.

### `serde_json` — v1.0

JSON support for serde. Used for OpenAI API calls, tool schemas, gateway
request/response bodies.

```rust
let body = serde_json::json!({  // macro creates Value at runtime
    "model": "gpt-4o",
    "messages": messages
});
let response: MyStruct = serde_json::from_str(&json_string)?;  // type-safe parse
```

### `toml` — v1.0 / `toml_edit` — v0.25

- `toml`: Rust standard TOML deserialization (used for config loading)
- `toml_edit`: **Lossless** TOML manipulation — reads a TOML file, edits it,
  writes it back **preserving all comments and formatting**

`toml_edit` is used for the `config set-key` CLI command and the live patch
endpoint. Without it, a round-trip through `toml::to_string` would strip all
comments from the user's config file.

### `schemars` — v1.2

Derives JSON Schema from Rust structs:

```rust
#[derive(schemars::JsonSchema, Deserialize)]
pub struct Config {
    pub agent: AgentConfig,
}

// → generates { "type": "object", "properties": { "agent": { ... } } }
```

Used for the `zeroclaw schema` command and IDE config file validation.

---

## 5. CLI Framework

### `clap` — v4.5

```toml
clap = { version = "4.5", features = ["derive"] }
```

Derives a fully-featured CLI argument parser from a struct:

```rust
#[derive(Parser)]
#[command(name = "zeroclaw", about = "Zero overhead AI agent")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
    #[arg(long, global = true, help = "Path to config file")]
    config: Option<PathBuf>,
}
```

Generated automatically: help text, error messages, shell completions,
version flag.

### `clap_complete` — v4.5

Generates shell completion scripts (bash, zsh, fish, PowerShell) from the
Clap `Command` definition. No manual completion script maintenance.

### `clap-markdown` — v0.1

Generates Markdown documentation for the CLI from the Clap definition,
used to keep `docs/book/src/reference/cli.md` in sync automatically.

---

## 6. Config and Schema

### `directories` — v6.0

Cross-platform OS config directory resolution:

```rust
use directories::ProjectDirs;
let dirs = ProjectDirs::from("com", "zeroclaw", "zeroclaw").unwrap();
// macOS: ~/Library/Application Support/zeroclaw/
// Linux: ~/.config/zeroclaw/
// Windows: C:\Users\Alice\AppData\Roaming\zeroclaw\zeroclaw\
```

**Why not hardcode `~/.config`?** Portability. macOS doesn't use `.config`.
Windows uses a different path entirely.

### `shellexpand` — v3.1

Expands shell variables and `~` in config paths:

```rust
let expanded = shellexpand::tilde("~/work/project").to_string();
// → "/Users/alice/work/project"
```

Allows users to write `workspace = "~/work"` in config.

---

## 7. Error Handling

### `anyhow` — v1.0

Ergonomic error type for application code:

```rust
use anyhow::{Context, Result, anyhow, bail};

fn load_config(path: &Path) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("reading config from {path:?}"))?;
    toml::from_str(&content).context("parsing config TOML")
}
```

`anyhow::Error` is a boxed `dyn Error + Send + Sync + 'static` with context
chain support. `?` on any `Result<T, E: Error>` converts it to `anyhow::Error`.

**Why**: Application code doesn't need typed errors (the caller can't handle
each variant differently). `anyhow` avoids the boilerplate of `Box<dyn Error>`
and provides context chaining for debugging.

### `thiserror` — v2.0

Derives `std::error::Error` for library error types:

```rust
#[derive(thiserror::Error, Debug)]
pub enum SecurityError {
    #[error("command not in allowed list: {0}")]
    CommandBlocked(String),
    #[error("path traversal blocked: {0}")]
    PathTraversal(String),
}
```

Used in `zeroclaw-api` and `zeroclaw-config` where callers need to match on
error variants. `thiserror` is the library companion; `anyhow` is the
application companion.

---

## 8. Cryptography and Security

### `chacha20poly1305` — v0.10

Authenticated Encryption with Associated Data (AEAD):

```rust
use chacha20poly1305::{ChaCha20Poly1305, Key, Nonce, aead::Aead};

// Encrypt API key before storing in config
let cipher = ChaCha20Poly1305::new(Key::from_slice(&key_bytes));
let ciphertext = cipher.encrypt(&nonce, plaintext.as_bytes())?;
```

**Why ChaCha20-Poly1305 over AES-GCM?**
- Constant-time on all hardware — AES-GCM is fast on CPUs with AES-NI
  but slow (and potentially timing-vulnerable) on ARM without it
- Chosen by WireGuard, TLS 1.3, Signal — proven in production
- Pure Rust, no FFI

### `hmac` + `sha2` + `hex` — v0.12, v0.10, v0.4

HMAC-SHA256 for webhook signature verification:

```rust
// Verify Telegram webhook signature, Stripe webhook, etc.
let mut mac = Hmac::<Sha256>::new_from_slice(secret)?;
mac.update(payload);
mac.verify_slice(expected)?;
```

### `rand` — v0.10

Cryptographically secure random number generation. Used for:
- One-time pairing codes (gateway onboarding)
- Nonce generation for AEAD
- Session token generation

### `ring` — v0.17 (optional)

HMAC-SHA256 for Zhipu/GLM JWT authentication. Used only when the
`channel-zhipuai` feature is enabled. Heavy C dependency — kept optional.

### `regex` — v1.10

Regular expressions for:
- Credential scrubbing (`SENSITIVE_KV_REGEX`)
- Leak detection patterns
- Tool call parsing

ZeroClaw uses `LazyLock<Regex>` to compile patterns once:

```rust
static SENSITIVE_KV_REGEX: LazyLock<Regex> = LazyLock::new(|| {
    Regex::new(r"(?i)(api[_-]?key|token|password|secret)\s*[:=]\s*[^\s,]+").unwrap()
});
```

---

## 9. Database and Persistence

### `sqlx` — v0.8

Async SQL with compile-time query checking:

```toml
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio", "macros"] }
```

```rust
// Query checked at compile time against a live database file
let row = sqlx::query!("SELECT key, value FROM memories WHERE key = ?", key)
    .fetch_one(&pool)
    .await?;
```

If the SQL is wrong (wrong column name, wrong type), the build fails.
No runtime surprises.

### `rusqlite` — v0.37 (optional, behind `agent-runtime`)

Synchronous SQLite driver with `bundled` feature (compiles SQLite from
source — no system SQLite dependency). Used for the legacy migration path
and some internal session backends.

### `chrono` — v0.4

Date/time handling:
- Timestamps on memory entries
- Cron schedule parsing
- Log event timestamps

### `chrono-tz` — v0.10 (optional)

Timezone-aware scheduling (user's local timezone for cron jobs).

### `cron` — v0.15 (optional)

CRON expression parsing for the scheduler:

```rust
use cron::Schedule;
let schedule = "0 30 9 * * Mon-Fri".parse::<Schedule>()?;
// "9:30 AM every weekday"
```

---

## 10. Concurrency Utilities

### `parking_lot` — v0.12

Faster, non-poisoning mutexes:

```rust
use parking_lot::RwLock;
let state: RwLock<Config> = RwLock::new(config);
// Never panics from a "poisoned lock" if a thread panics while holding it
// 10-40% faster than std::sync::Mutex on uncontended paths
```

**Why not `std::sync::Mutex`?** `parking_lot` doesn't poison (a panicking
thread doesn't permanently break the lock), and it's measurably faster
in benchmarks.

### `portable-atomic` — v1

Atomic operations on targets that don't have native 64-bit atomics
(e.g., some ARM7 embedded targets). Fallback implementation that uses
locks transparently.

### `lru` — v0.16 (optional)

LRU cache for bounded sender state (caching per-user metadata without
unbounded growth).

---

## 11. Memory and Vector Search

### `qdrant-client` (in `zeroclaw-memory`)

Rust client for Qdrant vector database. Used by `QdrantMemory` backend
for semantic (embedding-based) memory search.

### `fastembed` / OpenAI embedding API

Embedding generation for the semantic memory backend. Can run locally
(via `fastembed` with ONNX models) or via OpenAI's embedding endpoint.

---

## 12. Channels and Protocols

### `tokio-tungstenite` — v0.29

Async WebSocket client, used for:
- Discord Gateway connection
- Lark/DingTalk WebSocket API
- Nostr relay connections

```toml
tokio-tungstenite = { version = "0.29", features = ["rustls-tls-webpki-roots"] }
# Again: rustls over native TLS for portability
```

### `lettre` — v0.11 (optional, `channel-email`)

Email sending via SMTP with TLS. The `mail-parser` crate handles IMAP
message parsing on the receiving side.

### `nostr-sdk` — v0.44 (optional, `channel-nostr`)

Nostr protocol client. Used for the Nostr channel integration and also
for validating Nostr key format in the TUI onboarding wizard.

### `rumqttc` — v0.25

MQTT client for IoT/hardware integrations. Used when ZeroClaw runs on
embedded hardware and needs to communicate via MQTT broker.

### `tokio-socks` — v0.5 (optional)

SOCKS5 proxy support for channels that need to operate behind a proxy.

---

## 13. UI, TUI, and Desktop

### `ratatui` — v0.30 (optional, `tui-onboarding`)

Terminal User Interface framework. Used for the interactive first-run
onboarding wizard (`zeroclaw-tui`).

```rust
// ratatui renders widgets into a terminal buffer using crossterm
terminal.draw(|frame| {
    frame.render_widget(input_field, layout[0]);
    frame.render_widget(help_text, layout[1]);
})?;
```

### `crossterm` — v0.29 (optional, `tui-onboarding`)

Cross-platform terminal control (cursor movement, raw mode, event
reading). The backend driver for ratatui.

### `dialoguer` — v0.12

Interactive CLI prompts (text input, selections, confirmations):

```rust
let name = dialoguer::Input::<String>::new()
    .with_prompt("Your name")
    .interact()?;

let selected = dialoguer::FuzzySelect::new()
    .with_prompt("Choose a model")
    .items(&models)
    .interact()?;
```

Used unconditionally (non-optional) in the memory CLI.

### `console` — v0.16

Terminal colour, style, and width utilities. Used with `dialoguer` for
styled prompts and with `indicatif` for progress bar rendering.

### `indicatif` — v0.18

Progress bars and spinners for the update pipeline:

```rust
let bar = indicatif::ProgressBar::new(total_bytes);
// Shows: [████████░░░░] 8.2 MB / 12.1 MB (5.1 MB/s, 0.8s)
```

### `tauri` — v2 (`apps/tauri`)

The Tauri framework provides a native desktop window host for the ZeroClaw
web dashboard on macOS, Windows, and Linux.

```toml
# apps/tauri/Cargo.toml
tauri = { version = "2", features = ["tray-icon", "protocol-asset"] }
tauri-build = "2"  # build-time codegen for capabilities
```

Tauri's architecture:
- **Rust core** (`apps/tauri/src/`) — process management, native APIs
- **WebView** — renders the existing web dashboard HTML/CSS/JS
- **Capabilities** — declarative JSON files that grant specific permissions
  (AppleScript access, screenshot capture, filesystem paths)

**Why Tauri over Electron?**
- Uses the OS's built-in WebView (WKWebView on macOS, WebView2 on Windows,
  WebKitGTK on Linux) — no bundled Chromium, 5-50x smaller binary
- Core logic in Rust — same language as the rest of the project
- Built-in updater, tray icon, and IPC from day one

**Why a desktop app at all?**
The web dashboard requires the daemon to be running. The Tauri app manages
the daemon lifecycle (start on launch, stop on quit, restart on crash) so
non-technical users get a one-click experience without a terminal.

---

## 14. Tooling and Observability

### `tracing` — v0.1

Structured logging and distributed tracing:

```rust
tracing::info!(session = %key, model = %model, "starting agent turn");
tracing::warn!(tool = %name, error = %e, "tool call failed");
```

`tracing` captures span context (which tool call, which session) and
emits structured events. Subscribers format and route these events.

### `tracing-subscriber` — v0.3

The subscriber that renders tracing events. Configured with `EnvFilter`
to respect `RUST_LOG=zeroclaw=debug,reqwest=warn`.

### OpenTelemetry SDK suite (`opentelemetry`, `opentelemetry-otlp`, `opentelemetry-sdk`)

ZeroClaw's `OtelObserver` exports traces and metrics via OTLP to any
compatible backend (Jaeger, Grafana Tempo, Datadog, New Relic).

```toml
[observability]
otel_endpoint = "http://localhost:4318"  # OTLP/HTTP
otel_genai_content = "scrubbed"          # LLM I/O content policy
otel_tool_io = "scrubbed"               # tool I/O content policy
```

The OTel suite is used for:
- **Traces**: one span per agent turn, child spans per LLM call and tool call
- **Metrics**: counters + histograms for LLM calls, tool calls, memory ops,
  token usage, session counts, queue depth
- **Content policy**: each `OtelObserver` holds an immutable
  `OtelContentConfig` (Off/Scrubbed/Full) derived once at construction so
  concurrent observers with different privacy settings cannot interfere

**Why not just use `tracing` for this?**
The `tracing` crate emits text-centric logs. OTel emits structured spans and
numeric metrics in a standardised wire format that APM tools can aggregate,
query, and alert on. Both coexist: `tracing` for developer logs, OTel for
production observability.

### `uuid` — v1.22

UUIDs for:
- Session IDs
- Tool call IDs
- Webhook pairing codes

```toml
uuid = { version = "1", features = ["v4", "std"] }
# v4 = random UUIDs (using the `rand` crate)
```

---

## 15. Hardware and Embedded

### `aardvark-sys` (internal, `crates/aardvark-sys`)

FFI bindings to the Total Phase Aardvark USB adapter SDK
(I2C/SPI/GPIO hardware access). The `crates/aardvark-sys` crate is a
stub when the SDK is absent — so it compiles everywhere but only
functions on hardware with the physical adapter.

### `glob` — v0.3 (optional)

File path pattern matching used for USB device discovery
(`/dev/ttyUSB*`, `/dev/ttyACM*`).

### `libc` — v0.2 (Unix only)

Low-level POSIX bindings. Used for:
- Root/privilege detection (`getuid()`)
- Process signal handling
- Platform capability checks

---

## 16. Build and Dev Tools

### `build.rs`

ZeroClaw has a `build.rs` that:
- Embeds the git commit hash into the binary
- Validates feature flag combinations at compile time
- Generates the locales module path

> **MSRV**: The workspace minimum supported Rust version is `1.96.0`
> (updated from 1.87 in v0.8.4). The `zeroclaw-spawn` crate description
> notes that `spawn!` is the sanctioned spawn interface for daemon-path code,
> keeping task attribution complete in production traces.

### `cargo nextest` (external, CI)

A faster test runner that runs each test in parallel in its own process.
Used in the CI pipeline (`./dev/ci.sh test`).

### `cargo llvm-cov` (external, CI)

Code coverage using LLVM instrumentation. Produces an lcov report
for the CI coverage gate.

### `cargo audit` (external, CI)

Scans `Cargo.lock` against the RustSec advisory database for known
vulnerabilities.

### `cargo deny` (external, CI)

Enforces:
- License allowlist (no GPL dependencies)
- Source allowlist (no non-crates.io sources in production)
- Bans known problematic crates

### `taplo` — v0.8

TOML formatter and linter for `Cargo.toml`, `config.toml`, and all other
TOML files in the repository.

### `flake.nix` / `nix/`

Nix flake for a reproducible development environment. `nix develop` gives
a shell with the exact Rust toolchain, `cargo-nextest`, `taplo`, and all
dev tools pinned to exact versions.

---

## 17. Dependency Philosophy

ZeroClaw follows these rules when evaluating new dependencies:

| Rule | Detail |
|------|--------|
| **No OpenSSL** | All TLS via `rustls` — portable, no system dependency |
| **Pure Rust preferred** | Avoid FFI crates unless there is no Rust alternative |
| **Feature gates** | New integrations are optional features, not default |
| **No duplication** | If Tokio already provides it (e.g. `Mutex`), don't add `parking_lot` for the same purpose without a measured reason |
| **Minimal features** | Every crate uses `default-features = false` and explicitly lists needed features |
| **Version pinning** | Major versions pinned; minor/patch follow caret ranges automatically |
| **Security audit** | Every direct dependency is checked by `cargo audit` in CI |
| **License compatibility** | Only MIT and Apache-2.0 dependencies (enforced by `cargo deny`) |

### Why this matters for binary size

A typical "default everything" Rust project pulls in 200+ transitive
dependencies. ZeroClaw's feature-gated approach means:

- **Minimal install** (CLI + one provider, no channels): ~6MB binary
- **Default install** (all bundled channels): ~18MB binary
- **Full install** (all optional features): ~35MB binary

The same codebase, three very different deployment footprints.

---

## Quick Reference

| Crate | Version | Category | License |
|-------|---------|----------|---------|
| `tokio` | 1.50 | Async runtime | MIT |
| `reqwest` | 0.12 | HTTP client | MIT/Apache-2 |
| `axum` | 0.8 | HTTP server | MIT |
| `serde` | 1.0 | Serialization | MIT/Apache-2 |
| `serde_json` | 1.0 | JSON | MIT/Apache-2 |
| `toml` | 1.0 | TOML | MIT/Apache-2 |
| `toml_edit` | 0.25 | Lossless TOML | MIT/Apache-2 |
| `clap` | 4.5 | CLI | MIT/Apache-2 |
| `anyhow` | 1.0 | Errors | MIT/Apache-2 |
| `thiserror` | 2.0 | Typed errors | MIT/Apache-2 |
| `chacha20poly1305` | 0.10 | Encryption | MIT/Apache-2 |
| `parking_lot` | 0.12 | Concurrency | MIT/Apache-2 |
| `sqlx` | 0.8 | SQL/SQLite | MIT/Apache-2 |
| `chrono` | 0.4 | Date/time | MIT/Apache-2 |
| `regex` | 1.10 | Regex | MIT/Apache-2 |
| `tracing` | 0.1 | Observability | MIT |
| `uuid` | 1.22 | IDs | MIT/Apache-2 |
| `schemars` | 1.2 | JSON Schema | MIT |
| `directories` | 6.0 | OS paths | MIT/Apache-2 |
| `ratatui` | 0.30 | TUI | MIT |
| `tauri` | 2 | Desktop | MIT/Apache-2 |
| `opentelemetry` | 0.x | Observability | Apache-2 |
| `async-trait` | 0.1 | Async traits | MIT/Apache-2 |
| `tokio-tungstenite` | 0.29 | WebSocket | MIT |
| `rand` | 0.10 | CSPRNG | MIT/Apache-2 |
| `indicatif` | 0.18 | Progress bars | MIT |
