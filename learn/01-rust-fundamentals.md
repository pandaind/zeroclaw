# 01 — Rust Fundamentals (as seen in ZeroClaw)

This document teaches every Rust concept you need to understand ZeroClaw.
For each concept there is a short explanation, then the exact line(s) from
this project that use it, and finally a plain-English "why" section.

---

## Table of Contents

1. [Ownership and Borrowing](#1-ownership-and-borrowing)
2. [The `Result` and `Option` Types](#2-the-result-and-option-types)
3. [Structs and Enums](#3-structs-and-enums)
4. [Traits](#4-traits)
5. [Generics and Trait Bounds](#5-generics-and-trait-bounds)
6. [Async / Await](#6-async--await)
7. [Closures and Function Pointers](#7-closures-and-function-pointers)
8. [Arc and Mutex — Shared State Across Threads](#8-arc-and-mutex--shared-state-across-threads)
9. [Feature Flags (`#[cfg(feature = ...)]`)](#9-feature-flags-cfgfeature--)
10. [Derive Macros](#10-derive-macros)
11. [Procedural Macros](#11-procedural-macros)
12. [The `?` Operator and Error Propagation](#12-the--operator-and-error-propagation)
13. [Module System and Visibility](#13-module-system-and-visibility)
14. [Lifetimes](#14-lifetimes)
15. [Pattern Matching](#15-pattern-matching)
16. [Iterators and Closures](#16-iterators-and-closures)
17. [Tokio Task-Locals](#17-tokio-task-locals)
18. [Clippy Lint Configuration](#18-clippy-lint-configuration)

---

## 1. Ownership and Borrowing

### What it is

Rust has no garbage collector. Instead, every value has exactly one *owner*,
and when the owner goes out of scope the value is dropped (freed). You can
*borrow* a value — either as an immutable reference `&T` or a mutable
reference `&mut T` — but the compiler enforces that you cannot have both at
the same time.

### Where ZeroClaw uses it

```rust
// crates/zeroclaw-api/src/channel.rs
pub struct ChannelMessage {
    pub id: String,          // owned String
    pub sender: String,      // owned String
    pub content: String,     // owned String
    pub attachments: Vec<MediaAttachment>,  // owned Vec
}
```

Every field is **owned**. When a `ChannelMessage` is passed from the channel
to the agent loop, the compiler ensures no other code can mutate it at the
same time.

```rust
// crates/zeroclaw-api/src/model_provider.rs
pub struct ChatRequest<'a> {
    pub messages: &'a [ChatMessage],  // borrowed slice — zero copy
    pub tools: Option<&'a [ToolSpec]>,
}
```

`ChatRequest` borrows the message slice with lifetime `'a`. This means the
model provider call does not need to clone the entire conversation history.

### Why this matters

Ownership eliminates use-after-free bugs, double-frees, and data races at
compile time — no runtime overhead.

---

## 2. The `Result` and `Option` Types

### What they are

```rust
enum Option<T> { Some(T), None }
enum Result<T, E> { Ok(T), Err(E) }
```

Rust has no `null` and no exceptions. Every fallible function returns
`Result<T, E>`.

### Where ZeroClaw uses them

```rust
// crates/zeroclaw-api/src/tool.rs
async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
```

`anyhow::Result<T>` is a shorthand for `Result<T, anyhow::Error>` where
`anyhow::Error` can wrap any error type. Every tool call either succeeds with
a `ToolResult` or fails with a structured error that carries a full context
chain.

```rust
// crates/zeroclaw-api/src/channel.rs
pub thread_ts: Option<String>,       // None means top-level message
pub channel_alias: Option<String>,   // None for channels without aliasing
```

`Option<String>` signals "this field may not exist" without a special
sentinel value like `""` or `"null"`.

### Why this matters

Callers are forced to handle both cases. The compiler will not let you
accidentally ignore an error or dereference a null.

---

## 3. Structs and Enums

### Structs — named collections of fields

```rust
// crates/zeroclaw-api/src/model_provider.rs
pub struct ChatMessage {
    pub role: String,
    pub content: String,
}

impl ChatMessage {
    pub fn system(content: impl Into<String>) -> Self { ... }
    pub fn user(content: impl Into<String>) -> Self { ... }
    pub fn assistant(content: impl Into<String>) -> Self { ... }
}
```

Constructor methods on `impl` blocks replace multiple struct literals.
`impl Into<String>` means you can pass a `&str`, `String`, or anything
that converts to a `String` — the method accepts all of them.

### Enums — types with multiple shapes

```rust
// crates/zeroclaw-api/src/channel.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum ChannelApprovalResponse {
    Approve,
    Deny,
    #[serde(rename = "always")]
    AlwaysApprove,
}
```

Enums in Rust are *algebraic data types*. Each variant can carry different
data. Here the variants are unit variants (no payload), but Rust enums can
hold tuples or structs:

```rust
// src/lib.rs
pub enum GatewayCommands {
    Start,           // no payload
    Stop { port: u16 },  // struct variant
}
```

### Why this matters

Enums make illegal states unrepresentable. You cannot pass an integer
`3` where `ChannelApprovalResponse` is expected — the type system catches it.

---

## 4. Traits

### What they are

Traits are Rust's interfaces. They define a set of methods a type must
implement.

### The core traits in zeroclaw-api

```rust
// crates/zeroclaw-api/src/tool.rs
#[async_trait]
pub trait Tool: Send + Sync + crate::attribution::Attributable {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> serde_json::Value;
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;

    // Provided (default) method — implementors get this for free
    fn spec(&self) -> ToolSpec {
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }
}
```

Key observations:

- `: Send + Sync` — the trait requires implementors to be safe to move across
  threads (`Send`) and safe to share references across threads (`Sync`). This
  is mandatory because the async runtime (Tokio) runs on a thread pool.
- `#[async_trait]` — the `async-trait` crate is a workaround for Rust's
  current limitation that traits cannot directly have `async fn` in their
  definition (stable `async fn in traits` now exists in Rust 1.75+ but this
  project still uses the crate for compatibility).
- Default method `spec()` — implementors do not need to write this; the
  trait assembles it from the three required methods.

### Trait objects — `dyn Tool`

```rust
// The runtime stores tools as dynamic trait objects
let tools: Vec<Box<dyn Tool>> = vec![
    Box::new(ShellTool::new()),
    Box::new(MemoryStoreTool::new()),
];
```

`Box<dyn Tool>` is a *fat pointer*: a pointer to the data and a pointer to a
vtable of method implementations. This lets you mix different concrete tool
types in the same `Vec`.

### Why this matters

ZeroClaw can accept *any* type that implements `Tool` — built-in tools, WASM
plugins, tools from channels — without changing the runtime. This is the
foundation of the extensibility model.

---

## 5. Generics and Trait Bounds

### What they are

Generics let you write code that works for multiple types. Trait bounds
restrict which types are allowed.

### In ZeroClaw

```rust
// crates/zeroclaw-api/src/channel.rs
pub fn new(content: impl Into<String>, recipient: impl Into<String>) -> Self
```

`impl Into<String>` is a shorthand for `<T: Into<String>>`. The compiler
generates a specific version of `new` for each concrete type used at call
sites — this is *monomorphisation* and is zero-cost at runtime.

```rust
// crates/zeroclaw-api/src/model_provider.rs
pub trait ModelProvider: Send + Sync {
    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse>;
}
```

Any type that implements `ModelProvider + Send + Sync` can be used anywhere
the runtime expects a provider.

---

## 6. Async / Await

### What it is

Rust's async system is based on *futures*. A `Future` is a value that
represents a computation that may not have completed yet. `async fn` returns
a `Future`. `.await` suspends the current task and yields to the async
runtime until the future resolves.

ZeroClaw uses **Tokio** as its async runtime — a multi-threaded executor
that efficiently schedules thousands of tasks.

### How it appears

```rust
// Cargo.toml
tokio = { version = "1.50", default-features = false, features = [
    "rt-multi-thread",   // thread pool executor
    "macros",            // #[tokio::main], tokio::select!
    "time",              // sleep, interval
    "net",               // TCP, UDP
    "io-util",           // AsyncRead, AsyncWrite helpers
    "sync",              // Mutex, RwLock, broadcast, mpsc
    "process",           // spawn child processes
    "io-std",            // stdin/stdout async
    "fs",                // async file I/O
    "signal",            // SIGINT, SIGTERM handling
]}
```

Only features actually used are listed — this keeps compile time and binary
size down.

```rust
// src/main.rs (conceptually)
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Everything inside here is async
    run_agent().await?;
    Ok(())
}
```

`#[tokio::main]` wraps the `main` function in a Tokio runtime setup.

### Why this matters

AI agents spend most of their time waiting for HTTP responses from LLM APIs,
reading files, and listening for messages. Async lets the single binary handle
many concurrent channels and conversations without spawning an OS thread per
connection.

---

## 7. Closures and Function Pointers

### What they are

A closure is an anonymous function that can capture variables from its
surrounding scope.

### In ZeroClaw

From `AGENTS.md` (the project design document):

> **Resolver closures** (`Arc<dyn Fn() -> T + Send + Sync>`) that close over
> `Arc<RwLock<Config>>` and resolve on call.

This pattern avoids caching config by storing a closure that reads the
*live* config whenever called:

```rust
// Pattern used throughout the codebase
let config = Arc::new(RwLock::new(load_config()));
let allowed_users = Arc::new({
    let config = config.clone();  // increment ref count
    move || config.read().unwrap().allowed_users.clone()
    // ^^^^ closure captures `config`
});

// Later, at use-time:
let users = allowed_users();  // always reads current config
```

`move` transfers ownership of `config` into the closure. `Arc::clone`
increments the reference count so both the outer scope and the closure
share the same config.

---

## 8. Arc and Mutex — Shared State Across Threads

### What they are

- `Arc<T>` — *Atomically Reference Counted*. Multiple owners across threads.
  Cloning an `Arc` increments a counter; dropping it decrements. The value
  is freed when the counter hits zero.
- `Mutex<T>` / `RwLock<T>` — mutual exclusion. `Mutex` allows one writer
  at a time. `RwLock` allows many concurrent readers or one writer.

### Why ZeroClaw uses `parking_lot`

```toml
# Cargo.toml
parking_lot = "0.12"  # Fast mutexes that don't poison on panic
```

The standard library's `Mutex::lock()` returns a `PoisonError` if a thread
panicked while holding the lock. `parking_lot`'s mutex does not poison,
which avoids `.unwrap()` on every lock acquisition.

### Typical usage pattern

```rust
use std::sync::Arc;
use parking_lot::RwLock;

struct AgentState {
    history: Vec<ChatMessage>,
}

// In the channel orchestrator:
let state: Arc<RwLock<AgentState>> = Arc::new(RwLock::new(AgentState {
    history: vec![],
}));

// Reading (many simultaneous readers allowed):
let history_len = state.read().history.len();

// Writing (exclusive):
state.write().history.push(new_message);
```

---

## 9. Feature Flags (`#[cfg(feature = ...)]`)

### What they are

Feature flags let you compile different code depending on which features were
requested at build time. This is declared in `Cargo.toml`:

```toml
[features]
default = ["agent-runtime"]
agent-runtime = ["zeroclaw-channels", "zeroclaw-tools", "zeroclaw-runtime", ...]
gateway = ["zeroclaw-gateway"]
```

In Rust source:

```rust
// src/lib.rs
#[cfg(feature = "agent-runtime")]
pub mod agent;

#[cfg(feature = "gateway")]
pub mod gateway;
```

### Why ZeroClaw uses this heavily

A minimal deployment (e.g., embedded hardware) might not need the full agent
runtime. A headless server might not need TUI code. Feature flags mean one
codebase produces multiple binaries of varying sizes.

```toml
# Cargo.toml — optional crates only compiled when the feature is enabled
zeroclaw-channels = { workspace = true, optional = true }
zeroclaw-runtime  = { workspace = true, optional = true }
zeroclaw-tui      = { workspace = true, optional = true }
```

---

## 10. Derive Macros

### What they are

`#[derive(...)]` automatically generates trait implementations for a type.

### Common derives in ZeroClaw

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct ChannelApprovalRequest {
    pub tool_name: String,
    pub arguments_summary: String,
    pub raw_arguments: Option<serde_json::Value>,
}
```

| Derive | What it generates |
|--------|------------------|
| `Debug` | `fmt::Debug` impl — lets you print the value with `{:?}` |
| `Clone` | `.clone()` method — creates a deep copy |
| `Serialize` | `serde::Serialize` — converts to JSON/TOML/etc. |
| `Deserialize` | `serde::Deserialize` — parses from JSON/TOML/etc. |
| `PartialEq` | `==` and `!=` operators |
| `Eq` | Marker that `==` is reflexive/transitive (required for HashMap keys) |

### Custom derive — `Configurable`

ZeroClaw has its own derive macro in `crates/zeroclaw-macros`:

```rust
#[derive(Debug, Clone, Default, Serialize, Deserialize, Configurable)]
#[prefix = "channels.telegram"]
pub struct TelegramConfig {
    #[serde(default)]
    pub enabled: bool,
    #[secret]
    pub bot_token: String,
    pub allowed_users: Vec<String>,
}
```

The `Configurable` macro reads the `#[prefix]` and `#[secret]` annotations
and generates:
- `secret_fields()` — lists which fields are secrets
- `set_secret()` / `encrypt_secrets()` / `decrypt_secrets()` — key management
- `prop_fields()` / `get_prop()` / `set_prop()` — runtime config patching

This eliminates hundreds of lines of repetitive boilerplate.

---

## 11. Procedural Macros

### What they are

Procedural macros operate on Rust's *token stream* at compile time. They are
Rust programs that produce Rust code.

ZeroClaw's `zeroclaw-macros` crate is a `proc-macro` crate:

```toml
# crates/zeroclaw-macros/Cargo.toml
[lib]
proc-macro = true

[dependencies]
proc-macro2 = "1"
quote = "1"          # generate code with quasi-quotation
syn = { version = "2", features = ["full"] }  # parse Rust AST
```

### How the `Configurable` macro works (simplified)

```rust
// crates/zeroclaw-macros/src/lib.rs (simplified)
#[proc_macro_derive(Configurable, attributes(prefix, secret, nested))]
pub fn derive_configurable(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    // 1. Read #[prefix = "..."] attribute
    // 2. Walk struct fields
    // 3. For #[secret] fields, generate secret_fields() entries
    // 4. For #[nested] fields, generate delegation to child struct
    // 5. Emit generated Rust code using `quote!`
    quote! {
        impl Configurable for #name {
            fn secret_fields(&self) -> Vec<SecretFieldInfo> { ... }
            fn set_prop(&mut self, name: &str, val: &str) -> Result<()> { ... }
        }
    }.into()
}
```

---

## 12. The `?` Operator and Error Propagation

### What it does

`expr?` is shorthand for:

```rust
match expr {
    Ok(val) => val,
    Err(e) => return Err(e.into()),
}
```

It early-returns on error, converting the error type if needed.

### In ZeroClaw

```rust
// src/main.rs
async fn apply_comment_inline(
    config_path: &std::path::Path,
    path: &str,
    comment: &str,
) -> Result<()> {
    zeroclaw_config::comment_writer::apply_comments(
        config_path,
        &[(path.to_string(), comment.to_string())],
    )
    .await
    .context("failed to write comment annotation")   // anyhow: add context
    // If apply_comments fails, return early with a descriptive error
}
```

`anyhow::Context::context()` wraps the original error with an extra message,
building a chain like:
```
failed to write comment annotation
  caused by: permission denied (os error 13)
```

### Why this matters

Error chains give users and developers precise diagnostics without writing
elaborate error-handling code at every call site.

---

## 13. Module System and Visibility

### What it is

Rust organises code into *modules*. A module can be inline (`mod foo { }`) or
a separate file (`mod foo;` loads `foo.rs` or `foo/mod.rs`).

Visibility rules:
- `pub` — visible everywhere
- `pub(crate)` — visible within this crate only
- `pub(super)` — visible to the parent module only
- (no keyword) — private, only visible within this module

### In ZeroClaw

```rust
// src/lib.rs
pub mod commands;        // part of the public API
pub(crate) mod daemon;   // internal plumbing, not exported
pub mod agent;           // public — channel orchestrator calls this
```

The distinction between `pub` and `pub(crate)` is important: it documents
*intent*. `pub(crate)` says "this is implementation detail, not a contract".

---

## 14. Lifetimes

### What they are

Lifetimes track how long references are valid. The compiler uses them to
prevent dangling references.

### In ZeroClaw

```rust
// crates/zeroclaw-api/src/model_provider.rs
pub struct ChatRequest<'a> {
    pub messages: &'a [ChatMessage],
    pub tools: Option<&'a [ToolSpec]>,
}
```

`'a` says "the `ChatRequest` cannot outlive the slice it borrows". This is
enforced at compile time — you cannot accidentally hold a `ChatRequest` after
the `Vec<ChatMessage>` it points into has been dropped.

```rust
// crates/zeroclaw-runtime/src/agent/mod.rs
pub struct AgentAttribution<'a>(pub &'a str);
```

`AgentAttribution` borrows a string slice for the agent alias. It exists only
for the duration of a single function call — enough to open a tracing span.

---

## 15. Pattern Matching

### What it is

`match` in Rust is exhaustive — the compiler forces you to handle every
possible case.

### In ZeroClaw

```rust
// crates/zeroclaw-runtime/src/agent/mod.rs
pub(crate) fn is_runtime_approved_arg_tool(tool_name: &str) -> bool {
    matches!(
        tool_name,
        "shell" | "schedule" | "cron_add" | "cron_update" | "cron_run"
    )
}
```

`matches!` is a macro that expands to a `match` expression returning `bool`.
The `|` separates patterns — it matches any of the listed tool names.

```rust
// crates/zeroclaw-api/src/channel.rs
pub enum ChannelApprovalResponse {
    Approve,
    Deny,
    AlwaysApprove,
}

// Usage
match response {
    ChannelApprovalResponse::Approve => execute_once(),
    ChannelApprovalResponse::Deny => skip(),
    ChannelApprovalResponse::AlwaysApprove => {
        allowlist.insert(tool_name);
        execute_once();
    }
}
// The compiler errors if you forget any variant.
```

---

## 16. Iterators and Closures

### In ZeroClaw

```rust
// src/main.rs
fn config_patch_prop_kind(config: &Config, path: &str) -> Option<crate::config::PropKind> {
    config
        .prop_fields()       // returns Vec<PropFieldInfo>
        .into_iter()         // consume into an iterator
        .find(|f| f.name == path)  // closure: find first match
        .map(|f| f.kind)     // closure: extract the kind field
}
```

This is idiomatic Rust: chain iterator adapters instead of writing explicit
loops. The compiler often optimises this to the same assembly as a `for` loop.

---

## 17. Tokio Task-Locals

### What they are

`tokio::task_local!` declares variables that are local to a single async
*task* (like thread-locals, but per-task instead of per-thread).

### In ZeroClaw

```rust
// crates/zeroclaw-api/src/lib.rs
tokio::task_local! {
    /// Current thread/sender ID for per-sender rate limiting.
    pub static TOOL_LOOP_THREAD_ID: Option<String>;

    /// Override for tool choice mode.
    pub static TOOL_CHOICE_OVERRIDE: Option<String>;

    /// Session key for the currently active session.
    pub static TOOL_LOOP_SESSION_KEY: Option<String>;

    /// Extended-thinking budget for reasoning models (Anthropic, DeepSeek-R1).
    pub static NATIVE_THINKING_OVERRIDE: Option<NativeThinkingParams>;
}
```

These are set at the start of each agent turn and read deep inside the call
stack without passing them as function arguments. This avoids "prop drilling"
(threading a value through many layers of function calls).

### Why task-locals instead of globals

Multiple agents can run concurrently in the same process. A global `static`
would be shared between all tasks. A task-local is isolated to the agent turn
that set it.

---

## 18. Clippy Lint Configuration

### What it is

Clippy is Rust's official linter. It catches common mistakes and style issues
beyond what the compiler enforces.

### How ZeroClaw configures it

```rust
// src/main.rs (top of file)
#![warn(clippy::all, clippy::pedantic)]
#![allow(
    clippy::module_name_repetitions,  // ZeroClaw structs repeat module name by convention
    clippy::too_many_lines,           // some functions are long by necessity
    clippy::missing_errors_doc,       // docs are in AGENTS.md, not every fn
    dead_code,                        // WIP features may have unused stubs
    unused_imports,
    unused_variables,
)]
```

`#![warn(...)]` — enable these lints at warning level.
`#![allow(...)]` — silence these lints project-wide where they would be noisy
without adding safety value.

```toml
# clippy.toml (workspace root)
# Per-workspace clippy settings apply to every crate.
```

---

## Summary

| Rust concept | Where you see it in ZeroClaw |
|---|---|
| Ownership | `ChannelMessage` fields — owned Strings |
| Borrowing | `ChatRequest<'a>` — borrows message slice |
| `Result`/`Option` | Every fallible function, every optional field |
| Traits | `Tool`, `Channel`, `ModelProvider`, `Memory` |
| Async/Await | Every I/O-bound operation |
| `Arc<RwLock<T>>` | Shared config and agent state |
| Feature flags | Optional crates: channels, runtime, gateway |
| Derive macros | `Debug`, `Serialize`, `Deserialize`, `Configurable` |
| Proc macros | `zeroclaw-macros` — generates config boilerplate |
| `?` operator | Every function that calls another fallible function |
| Modules | `pub`, `pub(crate)`, private — intentional API surface |
| Task-locals | `TOOL_LOOP_THREAD_ID`, `NATIVE_THINKING_OVERRIDE` — per-agent-turn context |
