# 04 — Core Traits and the API Crate

This document is a complete walkthrough of `crates/zeroclaw-api/` — the most
fundamental crate in the project. Every other crate depends on it. Understanding
this crate is the key to understanding how ZeroClaw can support any LLM, any
channel, and any tool without modifying the runtime.

---

## Table of Contents

1. [Why a Separate API Crate?](#1-why-a-separate-api-crate)
2. [File Map](#2-file-map)
3. [The `Tool` Trait — Line by Line](#3-the-tool-trait--line-by-line)
4. [The `ModelProvider` Trait — Line by Line](#4-the-modelprovider-trait--line-by-line)
5. [The `Channel` Trait — Line by Line](#5-the-channel-trait--line-by-line)
6. [The `Memory` Trait](#6-the-memory-trait)
7. [Attribution — Logging with Identity](#7-attribution--logging-with-identity)
8. [Task-Local Variables](#8-task-local-variables)
9. [The `#[async_trait]` Workaround](#9-the-async_trait-workaround)
10. [How to Implement a New Tool](#10-how-to-implement-a-new-tool)
11. [How to Implement a New Provider](#11-how-to-implement-a-new-provider)
12. [Ingress, Principal, and Auth Types](#12-ingress-principal-and-auth-types)

---

## 1. Why a Separate API Crate?

Imagine the alternative: the runtime directly imports OpenAI's HTTP client.
To swap to Anthropic, you modify the runtime. To add a new channel, you
add `if channel == "telegram" { ... }` to the message dispatcher.

The API crate breaks this coupling. The runtime only knows about traits —
abstract contracts. Concrete implementations are registered separately.

```
Without zeroclaw-api:                With zeroclaw-api:
  runtime → openai.rs                runtime → ModelProvider trait
  runtime → anthropic.rs                    ↗ openai.rs implements it
  runtime → telegram.rs                    ↗ anthropic.rs implements it
  runtime → slack.rs               Channel trait
                                          ↗ telegram.rs implements it
                                          ↗ slack.rs implements it
```

The runtime compiles once. New providers and channels are added without
touching it.

---

## 2. File Map

```
crates/zeroclaw-api/src/
├── lib.rs               # crate root — re-exports + task-locals
├── agent.rs             # AgentConfig type (config for one agent instance)
├── attribution.rs       # Attributable trait, Role enum
├── channel.rs           # Channel trait, ChannelMessage, SendMessage
├── elicitation.rs       # NEW: structured clarification-request protocol
├── hook.rs              # NEW: lifecycle hook contracts
├── ingress.rs           # NEW: SourceClass, TurnOrigin, Transport, TrustClass
├── jsonrpc.rs           # NEW: JSON-RPC types shared by gateway and zerocode
├── media.rs             # MediaAttachment type (images, audio, video)
├── memory_traits.rs     # Memory trait, MemoryEntry, MemoryCategory
├── model_provider.rs    # ModelProvider trait, ChatMessage, ChatResponse, ToolCall
├── observability_traits.rs  # Observer trait
├── peripherals_traits.rs    # Peripheral trait (hardware boards)
├── plan.rs              # NEW: plan/approval protocol types
├── platform.rs          # NEW: platform capability descriptor types
├── principal.rs         # NEW: PrincipalId, AgentAlias, AuthMethod — auth subject
├── runtime_status.rs    # NEW: daemon runtime status contract types
├── runtime_traits.rs    # RuntimeAdapter trait
├── schema.rs            # JSON schema helpers
├── session_keys.rs      # Session key construction
├── tool.rs              # Tool trait, ToolResult, ToolSpec
└── vad.rs               # Voice Activity Detection traits
```

---

## 3. The `Tool` Trait — Line by Line

```rust
// crates/zeroclaw-api/src/tool.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
```

`async_trait` — because traits with `async fn` need this macro until Rust's
native `async fn in trait` support stabilises universally.
`serde` — because `ToolSpec` and `ToolResult` are serialised to JSON when
sent to LLM APIs and stored in logs.

```rust
/// Boilerplate-collapsing macro: pair a concrete `Tool` impl with a
/// matching `Attributable` impl...
#[macro_export]
macro_rules! tool_attribution {
    ($ty:ty, $kind:expr) => {
        impl $crate::attribution::Attributable for $ty {
            fn role(&self) -> $crate::attribution::Role {
                $crate::attribution::Role::Tool($kind)
            }
            fn alias(&self) -> &str {
                <Self as $crate::tool::Tool>::name(self)
            }
        }
    };
}
```

This is a *declarative macro* (`macro_rules!`). It generates an `Attributable`
implementation for any Tool. Without this macro, every new tool would need to
write the same 6-line boilerplate. Usage:

```rust
// In the ShellTool module:
crate::tool_attribution!(ShellTool, ::zeroclaw_api::attribution::ToolKind::Shell);
```

The macro expands to:

```rust
impl ::zeroclaw_api::attribution::Attributable for ShellTool {
    fn role(&self) -> ::zeroclaw_api::attribution::Role {
        ::zeroclaw_api::attribution::Role::Tool(::zeroclaw_api::attribution::ToolKind::Shell)
    }
    fn alias(&self) -> &str {
        <ShellTool as ::zeroclaw_api::tool::Tool>::name(self)
    }
}
```

```rust
/// Result of a tool execution
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}
```

Every tool execution returns this. `success: false` with `error: Some(...)` is
how a tool signals failure without panicking. The agent loop logs the error
and may present it to the LLM as context.

```rust
/// Description of a tool for the LLM
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,  // JSON Schema object
}
```

This is the structure sent to the LLM in the API request. The LLM reads
`description` to decide when to call the tool, and `parameters` to know what
arguments to provide.

```rust
#[async_trait]
pub trait Tool: Send + Sync + crate::attribution::Attributable {
```

- `Send` — the tool can be moved to another thread (required for Tokio tasks)
- `Sync` — the tool can be shared (referenced) from multiple threads simultaneously
- `crate::attribution::Attributable` — supertrait: every Tool also implements
  `Attributable`, so log entries from a tool carry the tool's identity

```rust
    fn name(&self) -> &str;
```

Returns a borrowed string (no allocation). This is the "function name" as the
LLM sees it — must be stable (e.g., `"shell"`, `"memory_store"`).

```rust
    fn description(&self) -> &str;
```

Human and LLM readable description. Good descriptions dramatically improve the
LLM's ability to use the tool correctly.

```rust
    fn parameters_schema(&self) -> serde_json::Value;
```

Returns a JSON Schema object. Example for a shell tool:

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The shell command to execute"
    }
  },
  "required": ["command"]
}
```

```rust
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;
```

The main method. Takes parsed JSON arguments (the LLM's tool call arguments),
runs the operation, returns the result. `async` because tool execution often
involves I/O (running processes, fetching URLs, reading files).

```rust
    // Provided default method — implementors get this for free
    fn spec(&self) -> ToolSpec {
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }
}
```

This is a *default method*. Implementors can override it, but most don't need
to — the trait assembles the spec from the three required methods.

---

## 4. The `ModelProvider` Trait — Line by Line

```rust
// crates/zeroclaw-api/src/model_provider.rs

/// A single message in a conversation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatMessage {
    pub role: String,   // "system", "user", "assistant", "tool"
    pub content: String,
}
```

This is the universal message format. All LLM providers speak different wire
formats (OpenAI uses `{"role": "user", "content": "..."}`, Anthropic uses
`{"role": "user", "content": [{"type": "text", "text": "..."}]}`). The
provider converts between this canonical struct and the API's wire format.

```rust
impl ChatMessage {
    pub fn system(content: impl Into<String>) -> Self {
        Self { role: "system".into(), content: content.into() }
    }
    pub fn user(content: impl Into<String>) -> Self { ... }
    pub fn assistant(content: impl Into<String>) -> Self { ... }
    pub fn tool(content: impl Into<String>) -> Self { ... }
}
```

Constructor methods prevent invalid role strings. You cannot accidentally
write `role: "System"` (wrong capitalisation).

```rust
/// A tool call requested by the LLM.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolCall {
    pub id: String,         // LLM-assigned call ID (for multi-call correlation)
    pub name: String,       // tool name
    pub arguments: String,  // raw JSON arguments string
    pub extra_content: Option<serde_json::Value>,
    // ^^^^ provider-specific opaque data that must round-trip (e.g. Gemini thought_signature)
}
```

The `extra_content` field is an example of pragmatic design: some providers
attach metadata to tool calls that must be echoed back in follow-up requests.
Rather than adding provider-specific fields, a generic JSON blob is used.

```rust
pub struct TokenUsage {
    pub input_tokens: Option<u64>,
    pub output_tokens: Option<u64>,
    pub cached_input_tokens: Option<u64>,  // prompt cache hits
}
```

Used for cost calculation. `Option<u64>` because some providers don't report
token counts.

```rust
pub struct ChatResponse {
    pub text: Option<String>,           // text response from LLM
    pub tool_calls: Vec<ToolCall>,      // tool calls requested
    pub usage: Option<TokenUsage>,
    pub reasoning_content: Option<String>,
    // ^^^^ raw "thinking" output from reasoning models (DeepSeek-R1, etc.)
    // preserved as pass-through for next API call
}
```

`text` is `Option` because some LLM responses are *only* tool calls with no
text. `reasoning_content` is kept separate from `text` because it must be
sent back to the model in a specific position in the message history.

```rust
pub struct ChatRequest<'a> {
    pub messages: &'a [ChatMessage],
    pub tools: Option<&'a [ToolSpec]>,
    pub model: &'a str,
    pub system_prompt: Option<&'a str>,
    ...
}
```

`'a` lifetime — borrows everything from the caller. No allocation for the
request itself. The provider builds its own API request body from this.

```rust
#[async_trait]
pub trait ModelProvider: Send + Sync {
    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse>;

    // Optional streaming method — not all providers support it
    async fn chat_stream(
        &self,
        req: ChatRequest<'_>,
    ) -> anyhow::Result<Pin<Box<dyn Stream<Item = anyhow::Result<ChatResponse>> + Send>>> {
        // Default: call chat() and return a single-item stream
        let resp = self.chat(req).await?;
        Ok(Box::pin(stream::once(async move { Ok(resp) })))
    }
}
```

`chat_stream` has a default implementation that falls back to non-streaming.
Providers that support streaming (OpenAI, Anthropic) can override it for
real-time token delivery.

`Pin<Box<dyn Stream<...>>>` is the standard Rust pattern for returning an
async stream from a trait method:
- `Box<dyn ...>` — heap-allocated type-erased stream
- `Pin<...>` — required by the `Stream` trait (stream cannot be moved once created)

---

## 5. The `Channel` Trait — Line by Line

```rust
// crates/zeroclaw-api/src/channel.rs

pub struct ChannelMessage {
    pub id: String,
    pub sender: String,
    pub reply_target: String,   // where to send the reply (e.g. chat_id for Telegram)
    pub content: String,
    pub channel: String,        // channel type name, e.g. "telegram"
    pub channel_alias: Option<String>,   // bot instance alias
    pub timestamp: u64,
    pub thread_ts: Option<String>,       // platform thread ID for threaded replies
    pub interruption_scope_id: Option<String>,  // for cancellation grouping
    pub attachments: Vec<MediaAttachment>,
}
```

`reply_target` vs `sender`: the sender is *who* sent the message (user ID).
`reply_target` is *where* to send the reply (which could be a group chat ID,
not the user's DM).

`interruption_scope_id` is used by the agent loop's cancellation system. If
a user sends a new message in the same thread as an in-progress agent turn,
the old turn is cancelled and a new one starts.

```rust
pub struct SendMessage {
    pub content: String,
    pub recipient: String,
    pub subject: Option<String>,        // for email channels
    pub thread_ts: Option<String>,
    pub cancellation_token: Option<CancellationToken>,
    pub attachments: Vec<MediaAttachment>,
}

impl SendMessage {
    pub fn new(content: impl Into<String>, recipient: impl Into<String>) -> Self { ... }
    pub fn with_subject(...) -> Self { ... }

    // Builder-pattern methods — return `Self` so they can be chained
    pub fn in_thread(mut self, thread_ts: Option<String>) -> Self {
        self.thread_ts = thread_ts;
        self
    }
    pub fn with_cancellation(mut self, token: CancellationToken) -> Self { ... }
    pub fn with_attachments(mut self, attachments: Vec<MediaAttachment>) -> Self { ... }
}
```

The *builder pattern*: methods return `Self` (by value, not reference) so
they can be chained:

```rust
let msg = SendMessage::new("Hello!", "chat_123")
    .in_thread(Some("ts_456".into()))
    .with_cancellation(token);
```

Each method takes ownership of `self`, modifies it, and returns it.

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    /// Human-readable name for logging
    fn name(&self) -> &str;

    /// Start listening for messages, call `handler` for each one
    async fn listen(
        &self,
        handler: Arc<dyn Fn(ChannelMessage) -> BoxFuture<'static, ()> + Send + Sync>,
    ) -> anyhow::Result<()>;

    /// Send a message
    async fn send(&self, msg: SendMessage) -> anyhow::Result<()>;

    /// Send typing indicator (optional — default no-op)
    async fn send_typing(&self, recipient: &str) -> anyhow::Result<()> {
        Ok(())
    }
}
```

`handler: Arc<dyn Fn(ChannelMessage) -> BoxFuture<'static, ()> + Send + Sync>`
is a callback:
- `Arc<dyn Fn(...)>` — shared reference to any function
- `-> BoxFuture<'static, ()>` — the function returns an async future
- `+ Send + Sync` — safe to call from multiple threads

The channel calls `handler(message).await` for each received message.
The `handler` is provided by the orchestrator — it routes messages to agents.

---

## 6. The `Memory` Trait

```rust
// crates/zeroclaw-api/src/memory_traits.rs

#[async_trait]
pub trait Memory: Send + Sync {
    async fn store(&self, key: &str, value: &str, ...) -> anyhow::Result<()>;
    async fn retrieve(&self, query: &str, ...) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn delete(&self, key: &str) -> anyhow::Result<()>;
    async fn list(&self, ...) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn export(&self, filter: &ExportFilter) -> anyhow::Result<Vec<MemoryEntry>>;
}
```

The five operations map to the five things an agent needs to do with memory:
- `store` — save a fact
- `retrieve` — semantic or keyword search
- `delete` — forget
- `list` — enumerate entries
- `export` — GDPR data export

Different backends implement these differently:
- `MarkdownMemory` — writes key=value files to disk
- `SqliteMemory` — stores rows in a SQLite table, uses FTS5 for keyword search
- `QdrantMemory` — stores embeddings in a vector database for semantic search

---

## 7. Attribution — Logging with Identity

Every log entry in ZeroClaw carries an attribution: who produced it, and what
kind of entity it was.

```rust
// crates/zeroclaw-api/src/attribution.rs

pub trait Attributable {
    fn role(&self) -> Role;
    fn alias(&self) -> &str;
}

pub enum Role {
    Agent,
    Tool(ToolKind),
    Channel(ChannelKind),
    Provider(ProviderKind),
    Memory(MemoryKind),
}
```

A `ShellTool` has `Role::Tool(ToolKind::Shell)` and `alias = "shell"`.
A Telegram channel has `Role::Channel(ChannelKind::Telegram)` and
`alias = "my-telegram-bot"` (the operator's config alias).

This lets the log reader show entries like:
```
[agent:clamps] Starting turn for user alice
[channel:telegram/clamps-bot] Received message from alice
[tool:shell/shell] Executing: ls -la
[provider:anthropic/claude-3-5] Sending 2847 tokens
```

---

## 8. Task-Local Variables

```rust
// crates/zeroclaw-api/src/lib.rs

tokio::task_local! {
    pub static TOOL_LOOP_THREAD_ID: Option<String>;
    pub static TOOL_CHOICE_OVERRIDE: Option<String>;
    pub static TOOL_LOOP_SESSION_KEY: Option<String>;
    pub static NATIVE_THINKING_OVERRIDE: Option<NativeThinkingParams>;  // NEW
}
```

`NATIVE_THINKING_OVERRIDE` carries extended-thinking budget parameters
(token budget for reasoning models such as Claude 3.7 Sonnet or DeepSeek-R1).
It is set at the outer orchestration boundary and read by `run_tool_call_loop`
when building `ChatRequest`. The minimum budget is `1_024` tokens; values
below this are clamped at resolution time (Anthropic rejects lower budgets
with HTTP 400).

These are set at the start of each agent turn:

```rust
// Pseudocode — the actual call in the runtime
TOOL_LOOP_THREAD_ID.scope(Some(sender_id.clone()), async {
    TOOL_LOOP_SESSION_KEY.scope(Some(session_key.clone()), async {
        // Everything inside here can read the task-local
        let tool_result = shell_tool.execute(args).await?;
        // shell_tool reads TOOL_LOOP_SESSION_KEY to scope its audit log
    }).await
}).await
```

Task-locals are like a "request context" in web frameworks. They allow deep
callees (like a tool deep in the call stack) to access request-scoped data
without it being passed as a function argument through every intermediate layer.

---

## 9. The `#[async_trait]` Workaround

Before Rust 1.75, traits could not directly have `async fn` methods. The
`async-trait` crate provides a proc-macro workaround.

What `#[async_trait]` does:

```rust
// You write:
#[async_trait]
trait Tool: Send + Sync {
    async fn execute(&self, args: Value) -> Result<ToolResult>;
}

// The macro desugars this to:
trait Tool: Send + Sync {
    fn execute<'life0, 'async_trait>(
        &'life0 self,
        args: Value,
    ) -> Pin<Box<dyn Future<Output = Result<ToolResult>> + Send + 'async_trait>>
    where
        'life0: 'async_trait,
        Self: 'async_trait;
}
```

The macro hides this complexity. You write natural async code; the macro
produces the low-level future boxing that the trait system needs.

**Note**: Rust 1.75 introduced `async fn` in stable traits. ZeroClaw still
uses `async-trait` for compatibility with older MSRV and for the `?Sized` and
object-safety guarantees the macro provides.

---

## 10. How to Implement a New Tool

Minimal example — a tool that returns the current UTC time:

```rust
// crates/zeroclaw-tools/src/time_tool.rs
use async_trait::async_trait;
use serde_json::json;
use zeroclaw_api::tool::{Tool, ToolResult};

pub struct TimeTool;

#[async_trait]
impl Tool for TimeTool {
    fn name(&self) -> &str {
        "current_time"
    }

    fn description(&self) -> &str {
        "Returns the current UTC date and time."
    }

    fn parameters_schema(&self) -> serde_json::Value {
        json!({
            "type": "object",
            "properties": {},
            "required": []
        })
    }

    async fn execute(&self, _args: serde_json::Value) -> anyhow::Result<ToolResult> {
        let now = chrono::Utc::now().to_rfc3339();
        Ok(ToolResult {
            success: true,
            output: now,
            error: None,
        })
    }
}

// Generate the Attributable impl (required by Tool's supertrait bound)
zeroclaw_api::tool_attribution!(TimeTool, zeroclaw_api::attribution::ToolKind::Plugin);
```

Then register it in the tool factory:

```rust
// crates/zeroclaw-tools/src/factory.rs
pub fn create_tools(config: &Config) -> Vec<Box<dyn Tool>> {
    vec![
        Box::new(ShellTool::new()),
        Box::new(TimeTool),          // ← add here
        Box::new(MemoryStoreTool::new()),
        // ...
    ]
}
```

That's it. The runtime and agent loop pick it up automatically.

---

## 11. How to Implement a New Provider

Minimal example — a provider that echoes the last user message (for testing):

```rust
// crates/zeroclaw-providers/src/echo.rs
use async_trait::async_trait;
use zeroclaw_api::model_provider::{ChatMessage, ChatRequest, ChatResponse, ModelProvider};

pub struct EchoProvider;

#[async_trait]
impl ModelProvider for EchoProvider {
    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse> {
        // Find the last user message
        let last_user = req.messages
            .iter()
            .rev()
            .find(|m| m.role == "user")
            .map(|m| m.content.clone())
            .unwrap_or_else(|| "No message".to_string());

        Ok(ChatResponse {
            text: Some(format!("Echo: {last_user}")),
            tool_calls: vec![],
            usage: None,
            reasoning_content: None,
        })
    }
}
```

Register in the factory:

```rust
// crates/zeroclaw-providers/src/factory.rs
pub fn create_model_provider(name: &str, config: &Config) -> Box<dyn ModelProvider> {
    match name {
        "openai" => Box::new(OpenAiProvider::new(config)),
        "echo" => Box::new(EchoProvider),   // ← add here
        _ => panic!("Unknown provider: {name}"),
    }
}
```

Operators then use `provider = "echo"` in their `config.toml`.

---

## 12. Ingress, Principal, and Auth Types

Three new modules in `zeroclaw-api` cover the authentication and turn-origin
contract that was previously implicit:

### `principal.rs` — Authenticated Subject

`PrincipalId` is a stable, opaque subject identifier that every gateway, RPC,
and ACP connection carries once an auth provider verifies a credential:

```rust
// crates/zeroclaw-api/src/principal.rs
pub struct PrincipalId(pub String);

impl PrincipalId {
    /// Sentinel for the single-operator / trusted-local path.
    /// An unbound connection that has no distinct IdP principal.
    pub const SHARED_OPERATOR: &'static str = "shared-operator";
}
```

The `SHARED_OPERATOR` sentinel lets callers treat "trusted but anonymous
operator" as a real `Principal` instead of branching on `Option`, without
special-casing throughout the codebase.

`AgentAlias` (a newtype for the agent name a principal may bind) keeps the
alias from being confused with arbitrary strings in grant checks.

### `ingress.rs` — Turn Classification

See [Architecture § 12](./02-project-architecture.md#12-ingress-policy--universal-turn-classification)
for the full `TurnOrigin` / `Transport` / `TrustClass` breakdown.

### `plan.rs` and `elicitation.rs` — LLM Interaction Protocols

- **`plan.rs`** — types for structured plan/approval exchanges (the agent
  proposes a plan, the operator approves or revises it before execution).
- **`elicitation.rs`** — typed clarification-request protocol: the agent
  asks a structured question; the channel renders it as a form or select list
  rather than free text.

Both are "Phase 1" type-only APIs; surface rendering is per-channel.
