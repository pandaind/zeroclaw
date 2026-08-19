# 09 — Build Your Own AI Agent Runtime

This is the hands-on guide for building a similar project from scratch.
You will end up with a minimal but fully functional AI agent runtime in Rust:
a provider, a tool, a channel, persistent memory, config loading, and a
reliable provider wrapper.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Project Setup](#2-project-setup)
3. [Step 1 — The API Crate (Traits)](#3-step-1--the-api-crate-traits)
4. [Step 2 — The Config Crate](#4-step-2--the-config-crate)
5. [Step 3 — First Provider (OpenAI)](#5-step-3--first-provider-openai)
6. [Step 4 — First Tool (Shell)](#6-step-4--first-tool-shell)
7. [Step 5 — Memory Crate (SQLite)](#7-step-5--memory-crate-sqlite)
8. [Step 6 — First Channel (CLI)](#8-step-6--first-channel-cli)
9. [Step 7 — The Agent Loop](#9-step-7--the-agent-loop)
10. [Step 8 — Wire Everything in `main.rs`](#10-step-8--wire-everything-in-mainrs)
11. [Step 9 — Reliable Provider Wrapper](#11-step-9--reliable-provider-wrapper)
12. [Step 10 — Security Basics](#12-step-10--security-basics)
13. [Running and Testing](#13-running-and-testing)
14. [What to Add Next](#14-what-to-add-next)
15. [Key Design Decisions Recap](#15-key-design-decisions-recap)

---

## 1. Prerequisites

Install Rust:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup update stable
```

We will use:
- Rust edition 2024 (Rust 1.96+)
- Tokio for async
- Clap for CLI
- Serde + toml for config
- Reqwest for HTTP (calling OpenAI)
- sqlx for SQLite memory
- async-trait for trait objects with `async fn`
- anyhow for error handling

---

## 2. Project Setup

```bash
# Create a new Cargo workspace
mkdir myagent && cd myagent
cat > Cargo.toml << 'EOF'
[workspace]
members = [
    "crates/myagent-api",
    "crates/myagent-config",
    "crates/myagent-providers",
    "crates/myagent-tools",
    "crates/myagent-memory",
    "crates/myagent-channels",
    "crates/myagent-runtime",
]
resolver = "2"

[workspace.dependencies]
tokio       = { version = "1", features = ["rt-multi-thread", "macros", "time", "net"] }
async-trait = "0.1"
anyhow      = "1"
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
reqwest     = { version = "0.12", features = ["json", "rustls-tls"], default-features = false }
sqlx        = { version = "0.8", features = ["sqlite", "runtime-tokio", "macros"] }
clap        = { version = "4", features = ["derive"] }
toml        = "0.8"
tracing     = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
EOF

# Create each crate directory
for crate in api config providers tools memory channels runtime; do
    mkdir -p crates/myagent-${crate}/src
    cat > crates/myagent-${crate}/Cargo.toml << EOF
[package]
name = "myagent-${crate}"
version = "0.1.0"
edition = "2024"
EOF
    echo "pub mod lib;" > crates/myagent-${crate}/src/lib.rs
done

# Create the main binary
mkdir -p src
touch src/main.rs
cat >> Cargo.toml << 'EOF'

[[bin]]
name = "myagent"
path = "src/main.rs"

[dependencies]
myagent-config   = { path = "crates/myagent-config" }
myagent-runtime  = { path = "crates/myagent-runtime" }
tokio.workspace  = true
clap.workspace   = true
EOF
```

---

## 3. Step 1 — The API Crate (Traits)

This crate defines all shared traits. Nothing imports concrete types from it.
All other crates depend on this; it depends on nothing.

```toml
# crates/myagent-api/Cargo.toml
[package]
name = "myagent-api"
version = "0.1.0"
edition = "2024"

[dependencies]
async-trait.workspace = true
anyhow.workspace = true
serde.workspace = true
serde_json.workspace = true
```

```rust
// crates/myagent-api/src/lib.rs
pub mod model_provider;
pub mod tool;
pub mod channel;
pub mod memory;
```

```rust
// crates/myagent-api/src/model_provider.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "role", content = "content", rename_all = "lowercase")]
pub enum ChatMessage {
    System(String),
    User(String),
    Assistant(String),
    Tool { tool_call_id: String, content: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolCall {
    pub id: String,
    pub name: String,
    pub arguments: serde_json::Value,
}

#[derive(Debug)]
pub struct ChatResponse {
    pub content: Option<String>,
    pub tool_calls: Vec<ToolCall>,
    pub input_tokens: u32,
    pub output_tokens: u32,
}

#[derive(Debug)]
pub struct ChatRequest<'a> {
    pub messages: &'a [ChatMessage],
    pub system: Option<&'a str>,
    pub tools: &'a [ToolSpec],   // tool schemas the model may call
    pub model: &'a str,
    pub temperature: Option<f64>,
    pub max_tokens: Option<u32>,
}

// Forward declaration — defined in tool.rs
pub use crate::tool::ToolSpec;

/// A model provider can process chat messages and return responses.
#[async_trait]
pub trait ModelProvider: Send + Sync {
    fn name(&self) -> &str;
    fn default_model(&self) -> &str;
    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse>;
}
```

```rust
// crates/myagent-api/src/tool.rs
use async_trait::async_trait;
use serde_json::Value;

#[derive(Debug, Clone)]
pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub parameters_schema: Value,
}

#[derive(Debug)]
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}

/// A tool the agent can call during its turn.
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn parameters_schema(&self) -> Value;

    async fn execute(&self, args: Value) -> anyhow::Result<ToolResult>;

    fn spec(&self) -> ToolSpec {
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters_schema: self.parameters_schema(),
        }
    }
}
```

```rust
// crates/myagent-api/src/channel.rs
use async_trait::async_trait;
use std::sync::Arc;

#[derive(Debug, Clone)]
pub struct ChannelMessage {
    pub sender: String,
    pub content: String,
    pub reply_target: String,   // used to route the reply back
}

#[async_trait]
pub trait Channel: Send + Sync {
    fn name(&self) -> &str;
    async fn send(&self, target: &str, content: &str) -> anyhow::Result<()>;
}
```

```rust
// crates/myagent-api/src/memory.rs
use async_trait::async_trait;

#[derive(Debug, Clone)]
pub struct MemoryEntry {
    pub key: String,
    pub value: String,
    pub score: Option<f64>,
}

#[async_trait]
pub trait Memory: Send + Sync {
    async fn store(&self, key: &str, value: &str) -> anyhow::Result<()>;
    async fn recall(&self, query: &str, limit: usize) -> anyhow::Result<Vec<MemoryEntry>>;
    async fn forget(&self, key: &str) -> anyhow::Result<()>;
}
```

---

## 4. Step 2 — The Config Crate

```toml
# crates/myagent-config/Cargo.toml
[dependencies]
myagent-api  = { path = "../myagent-api" }
serde.workspace = true
toml.workspace = true
anyhow.workspace = true
```

```rust
// crates/myagent-config/src/lib.rs
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Config {
    #[serde(default)]
    pub agent: AgentConfig,
    pub openai: Option<OpenAiConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct AgentConfig {
    pub system_prompt: Option<String>,
    pub max_tool_rounds: Option<u32>,    // default: 10
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OpenAiConfig {
    pub api_key: String,
    pub model: Option<String>,           // default: "gpt-4o-mini"
}

impl Config {
    pub fn load(path: &PathBuf) -> anyhow::Result<Self> {
        let content = std::fs::read_to_string(path)?;
        let config: Self = toml::from_str(&content)?;
        Ok(config)
    }

    pub fn default_path() -> PathBuf {
        let mut p = dirs::config_dir().unwrap_or_else(|| PathBuf::from("."));
        p.push("myagent");
        p.push("config.toml");
        p
    }
}
```

Create a default config file:

```toml
# ~/.config/myagent/config.toml
[agent]
system_prompt = "You are a helpful assistant."
max_tool_rounds = 10

[openai]
api_key = "sk-..."
model = "gpt-4o-mini"
```

---

## 5. Step 3 — First Provider (OpenAI)

```toml
# crates/myagent-providers/Cargo.toml
[dependencies]
myagent-api  = { path = "../myagent-api" }
async-trait.workspace = true
anyhow.workspace = true
reqwest.workspace = true
serde.workspace = true
serde_json.workspace = true
tokio.workspace = true
```

```rust
// crates/myagent-providers/src/openai.rs
use anyhow::{Context, anyhow};
use async_trait::async_trait;
use myagent_api::model_provider::*;
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};

pub struct OpenAiProvider {
    api_key: String,
    model: String,
    client: reqwest::Client,
}

impl OpenAiProvider {
    pub fn new(api_key: String, model: String) -> Self {
        Self {
            api_key,
            model,
            client: reqwest::Client::new(),
        }
    }
}

// Internal types for deserialization
#[derive(Deserialize)]
struct OaiResponse {
    choices: Vec<OaiChoice>,
    usage: OaiUsage,
}
#[derive(Deserialize)]
struct OaiChoice {
    message: OaiMessage,
}
#[derive(Deserialize)]
struct OaiMessage {
    content: Option<String>,
    tool_calls: Option<Vec<OaiToolCall>>,
}
#[derive(Deserialize)]
struct OaiToolCall {
    id: String,
    function: OaiFunction,
}
#[derive(Deserialize)]
struct OaiFunction {
    name: String,
    arguments: String,
}
#[derive(Deserialize)]
struct OaiUsage {
    prompt_tokens: u32,
    completion_tokens: u32,
}

#[async_trait]
impl ModelProvider for OpenAiProvider {
    fn name(&self) -> &str { "openai" }
    fn default_model(&self) -> &str { &self.model }

    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse> {
        // Build the messages array
        let messages: Vec<Value> = req.messages.iter().map(|m| match m {
            ChatMessage::System(text)    => json!({ "role": "system", "content": text }),
            ChatMessage::User(text)      => json!({ "role": "user", "content": text }),
            ChatMessage::Assistant(text) => json!({ "role": "assistant", "content": text }),
            ChatMessage::Tool { tool_call_id, content } => json!({
                "role": "tool",
                "tool_call_id": tool_call_id,
                "content": content,
            }),
        }).collect();

        // Build tool schemas for the API
        let tools: Vec<Value> = req.tools.iter().map(|t| json!({
            "type": "function",
            "function": {
                "name": t.name,
                "description": t.description,
                "parameters": t.parameters_schema,
            }
        })).collect();

        let mut body = json!({
            "model": req.model,
            "messages": messages,
        });
        if !tools.is_empty() {
            body["tools"] = json!(tools);
        }
        if let Some(t) = req.temperature {
            body["temperature"] = json!(t);
        }
        if let Some(m) = req.max_tokens {
            body["max_tokens"] = json!(m);
        }

        let resp = self.client
            .post("https://api.openai.com/v1/chat/completions")
            .bearer_auth(&self.api_key)
            .json(&body)
            .send()
            .await
            .context("HTTP request to OpenAI failed")?
            .error_for_status()
            .context("OpenAI returned error status")?
            .json::<OaiResponse>()
            .await
            .context("Failed to parse OpenAI response")?;

        let message = &resp.choices[0].message;

        // Parse tool calls if present
        let tool_calls: Vec<ToolCall> = message.tool_calls.as_deref()
            .unwrap_or_default()
            .iter()
            .map(|tc| {
                let args: Value = serde_json::from_str(&tc.function.arguments)
                    .unwrap_or(Value::Object(Default::default()));
                ToolCall { id: tc.id.clone(), name: tc.function.name.clone(), arguments: args }
            })
            .collect();

        Ok(ChatResponse {
            content: message.content.clone(),
            tool_calls,
            input_tokens: resp.usage.prompt_tokens,
            output_tokens: resp.usage.completion_tokens,
        })
    }
}
```

---

## 6. Step 4 — First Tool (Shell)

```toml
# crates/myagent-tools/Cargo.toml
[dependencies]
myagent-api  = { path = "../myagent-api" }
async-trait.workspace = true
anyhow.workspace = true
serde_json.workspace = true
tokio.workspace = true
```

```rust
// crates/myagent-tools/src/shell.rs
use anyhow::anyhow;
use async_trait::async_trait;
use myagent_api::tool::{Tool, ToolResult};
use serde_json::{Value, json};
use tokio::process::Command;

pub struct ShellTool;

#[async_trait]
impl Tool for ShellTool {
    fn name(&self) -> &str { "shell" }

    fn description(&self) -> &str {
        "Execute a shell command and return its output."
    }

    fn parameters_schema(&self) -> Value {
        json!({
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The shell command to run"
                }
            },
            "required": ["command"]
        })
    }

    async fn execute(&self, args: Value) -> anyhow::Result<ToolResult> {
        let command = args["command"]
            .as_str()
            .ok_or_else(|| anyhow!("missing 'command' argument"))?;

        let output = Command::new("sh")
            .arg("-c")
            .arg(command)
            .output()
            .await?;

        let stdout = String::from_utf8_lossy(&output.stdout).to_string();
        let stderr = String::from_utf8_lossy(&output.stderr).to_string();
        let exit_code = output.status.code().unwrap_or(-1);

        if exit_code == 0 {
            Ok(ToolResult {
                success: true,
                output: stdout,
                error: None,
            })
        } else {
            Ok(ToolResult {
                success: false,
                output: stdout,
                error: Some(format!("exit code {exit_code}: {stderr}")),
            })
        }
    }
}
```

---

## 7. Step 5 — Memory Crate (SQLite)

```toml
# crates/myagent-memory/Cargo.toml
[dependencies]
myagent-api  = { path = "../myagent-api" }
async-trait.workspace = true
anyhow.workspace = true
sqlx.workspace = true
tokio.workspace = true
```

```rust
// crates/myagent-memory/src/sqlite.rs
use anyhow::Context;
use async_trait::async_trait;
use myagent_api::memory::{Memory, MemoryEntry};
use sqlx::{SqlitePool, sqlite::SqliteConnectOptions};
use std::str::FromStr;

pub struct SqliteMemory {
    pool: SqlitePool,
}

impl SqliteMemory {
    pub async fn new(db_path: &str) -> anyhow::Result<Self> {
        let opts = SqliteConnectOptions::from_str(db_path)?
            .create_if_missing(true);
        let pool = SqlitePool::connect_with(opts).await?;

        // Create table
        sqlx::query(
            "CREATE TABLE IF NOT EXISTS memories (
                key       TEXT PRIMARY KEY,
                value     TEXT NOT NULL,
                timestamp INTEGER NOT NULL
            )"
        )
        .execute(&pool)
        .await
        .context("creating memories table")?;

        // Create FTS5 virtual table for full-text search
        sqlx::query(
            "CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts
             USING fts5(key, value, content=memories, content_rowid=rowid)"
        )
        .execute(&pool)
        .await
        .ok(); // ignore if already exists or FTS5 not available

        Ok(Self { pool })
    }
}

#[async_trait]
impl Memory for SqliteMemory {
    async fn store(&self, key: &str, value: &str) -> anyhow::Result<()> {
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        sqlx::query!(
            "INSERT OR REPLACE INTO memories (key, value, timestamp) VALUES (?, ?, ?)",
            key, value, now
        )
        .execute(&self.pool)
        .await?;
        Ok(())
    }

    async fn recall(&self, query: &str, limit: usize) -> anyhow::Result<Vec<MemoryEntry>> {
        // Simple LIKE search (upgrade to FTS5 for production)
        let pattern = format!("%{query}%");
        let limit = limit as i64;

        let rows = sqlx::query!(
            "SELECT key, value FROM memories
             WHERE value LIKE ? OR key LIKE ?
             ORDER BY timestamp DESC LIMIT ?",
            pattern, pattern, limit
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(rows.into_iter().map(|r| MemoryEntry {
            key: r.key,
            value: r.value,
            score: None,
        }).collect())
    }

    async fn forget(&self, key: &str) -> anyhow::Result<()> {
        sqlx::query!("DELETE FROM memories WHERE key = ?", key)
            .execute(&self.pool)
            .await?;
        Ok(())
    }
}
```

---

## 8. Step 6 — First Channel (CLI)

```toml
# crates/myagent-channels/Cargo.toml
[dependencies]
myagent-api  = { path = "../myagent-api" }
async-trait.workspace = true
anyhow.workspace = true
tokio.workspace = true
```

```rust
// crates/myagent-channels/src/cli.rs
use async_trait::async_trait;
use myagent_api::channel::Channel;

pub struct CliChannel;

#[async_trait]
impl Channel for CliChannel {
    fn name(&self) -> &str { "cli" }

    async fn send(&self, _target: &str, content: &str) -> anyhow::Result<()> {
        println!("\nAssistant: {content}\n");
        Ok(())
    }
}
```

---

## 9. Step 7 — The Agent Loop

```toml
# crates/myagent-runtime/Cargo.toml
[dependencies]
myagent-api      = { path = "../myagent-api" }
async-trait.workspace = true
anyhow.workspace = true
serde_json.workspace = true
tokio.workspace = true
tracing.workspace = true
```

```rust
// crates/myagent-runtime/src/agent_loop.rs
use anyhow::{anyhow, Result};
use myagent_api::{
    channel::Channel,
    memory::Memory,
    model_provider::{ChatMessage, ChatRequest, ChatResponse, ModelProvider},
    tool::{Tool, ToolSpec},
};
use std::sync::Arc;

pub struct AgentLoop {
    pub provider: Box<dyn ModelProvider>,
    pub tools: Vec<Box<dyn Tool>>,
    pub memory: Arc<dyn Memory>,
    pub channel: Arc<dyn Channel>,
    pub system_prompt: String,
    pub model: String,
    pub max_tool_rounds: u32,
}

impl AgentLoop {
    /// Run one complete agent turn for the given user message.
    pub async fn run_turn(
        &self,
        user_message: &str,
        history: &mut Vec<ChatMessage>,
        reply_target: &str,
    ) -> Result<()> {
        // 1. Recall relevant memories
        let memories = self.memory.recall(user_message, 5).await?;
        let memory_context = if memories.is_empty() {
            String::new()
        } else {
            let lines: Vec<_> = memories.iter()
                .map(|e| format!("- [{}] {}", e.key, e.value))
                .collect();
            format!("\n\nRelevant memories:\n{}", lines.join("\n"))
        };

        // 2. Build tool specs
        let tool_specs: Vec<ToolSpec> = self.tools.iter().map(|t| t.spec()).collect();

        // 3. Append user message to history
        history.push(ChatMessage::User(user_message.to_string()));

        // 4. Tool call loop
        let mut round = 0;
        loop {
            if round >= self.max_tool_rounds {
                return Err(anyhow!("max tool rounds ({}) exceeded", self.max_tool_rounds));
            }
            round += 1;

            // 5. Inject system prompt with memory context
            let system = format!("{}{}", self.system_prompt, memory_context);

            let req = ChatRequest {
                messages: history,
                system: Some(&system),
                tools: &tool_specs,
                model: &self.model,
                temperature: Some(0.7),
                max_tokens: Some(4096),
            };

            // 6. Call the model
            tracing::debug!("Sending request to provider (round {round})");
            let response = self.provider.chat(req).await?;

            // 7. If model returned text and no tool calls → done
            if response.tool_calls.is_empty() {
                let content = response.content.unwrap_or_default();
                history.push(ChatMessage::Assistant(content.clone()));
                self.channel.send(reply_target, &content).await?;
                return Ok(());
            }

            // 8. Record the assistant turn (with tool calls)
            if let Some(text) = &response.content {
                history.push(ChatMessage::Assistant(text.clone()));
            }

            // 9. Execute all tool calls
            for tc in &response.tool_calls {
                let result = self.execute_tool(&tc.name, tc.arguments.clone()).await;
                let output = match result {
                    Ok(r) => r.output,
                    Err(e) => format!("Tool error: {e}"),
                };
                tracing::debug!("Tool {} returned {} bytes", tc.name, output.len());
                history.push(ChatMessage::Tool {
                    tool_call_id: tc.id.clone(),
                    content: output,
                });
            }
        }
    }

    async fn execute_tool(
        &self,
        name: &str,
        args: serde_json::Value,
    ) -> anyhow::Result<myagent_api::tool::ToolResult> {
        let tool = self.tools.iter()
            .find(|t| t.name() == name)
            .ok_or_else(|| anyhow!("unknown tool: {name}"))?;
        tool.execute(args).await
    }
}
```

---

## 10. Step 8 — Wire Everything in `main.rs`

```rust
// src/main.rs
use anyhow::Result;
use clap::{Parser, Subcommand};
use myagent_config::Config;
use std::sync::Arc;

#[derive(Parser)]
#[command(name = "myagent", about = "A minimal Rust AI agent")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
    #[arg(long, global = true)]
    config: Option<std::path::PathBuf>,
}

#[derive(Subcommand)]
enum Commands {
    /// Start interactive chat
    Chat,
}

#[tokio::main]
async fn main() -> Result<()> {
    // Initialise logging
    tracing_subscriber::fmt::init();

    let cli = Cli::parse();
    let config_path = cli.config.unwrap_or_else(Config::default_path);
    let config = Config::load(&config_path)?;

    match cli.command {
        Commands::Chat => run_chat(config).await,
    }
}

async fn run_chat(config: Config) -> Result<()> {
    use myagent_channels::cli::CliChannel;
    use myagent_memory::sqlite::SqliteMemory;
    use myagent_providers::openai::OpenAiProvider;
    use myagent_runtime::agent_loop::AgentLoop;
    use myagent_tools::shell::ShellTool;

    // Validate required config
    let openai_cfg = config.openai
        .as_ref()
        .ok_or_else(|| anyhow::anyhow!("openai config not set"))?;

    // Build components
    let provider = Box::new(OpenAiProvider::new(
        openai_cfg.api_key.clone(),
        openai_cfg.model.clone().unwrap_or_else(|| "gpt-4o-mini".to_string()),
    ));

    let memory_path = "sqlite:///tmp/myagent_memory.db";
    let memory = Arc::new(SqliteMemory::new(memory_path).await?);

    let channel = Arc::new(CliChannel);

    let agent = AgentLoop {
        provider,
        tools: vec![Box::new(ShellTool)],
        memory: Arc::clone(&memory),
        channel: Arc::clone(&channel),
        system_prompt: config.agent.system_prompt
            .unwrap_or_else(|| "You are a helpful assistant.".to_string()),
        model: openai_cfg.model.clone().unwrap_or_else(|| "gpt-4o-mini".to_string()),
        max_tool_rounds: config.agent.max_tool_rounds.unwrap_or(10),
    };

    // Interactive loop
    let mut history = Vec::new();
    println!("myagent ready. Type your message (Ctrl+D to quit).\n");

    loop {
        print!("You: ");
        use std::io::Write;
        std::io::stdout().flush()?;

        let mut line = String::new();
        if std::io::stdin().read_line(&mut line)? == 0 {
            break;  // EOF
        }
        let message = line.trim();
        if message.is_empty() { continue; }

        if let Err(e) = agent.run_turn(message, &mut history, "stdout").await {
            eprintln!("Error: {e}");
        }
    }

    println!("\nGoodbye!");
    Ok(())
}
```

---

## 11. Step 9 — Reliable Provider Wrapper

```rust
// crates/myagent-providers/src/reliable.rs
use async_trait::async_trait;
use myagent_api::model_provider::{ChatRequest, ChatResponse, ModelProvider};
use std::time::Duration;

pub struct ReliableModelProvider {
    primary: Box<dyn ModelProvider>,
    fallback: Option<Box<dyn ModelProvider>>,
    max_retries: u32,
    timeout: Duration,
}

impl ReliableModelProvider {
    pub fn new(primary: Box<dyn ModelProvider>) -> Self {
        Self {
            primary,
            fallback: None,
            max_retries: 3,
            timeout: Duration::from_secs(60),
        }
    }

    pub fn with_fallback(mut self, fallback: Box<dyn ModelProvider>) -> Self {
        self.fallback = Some(fallback);
        self
    }
}

#[async_trait]
impl ModelProvider for ReliableModelProvider {
    fn name(&self) -> &str { self.primary.name() }
    fn default_model(&self) -> &str { self.primary.default_model() }

    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse> {
        let mut last_err = anyhow::anyhow!("no attempts made");

        for attempt in 1..=self.max_retries {
            let result = tokio::time::timeout(
                self.timeout,
                self.primary.chat(req),
            ).await;

            match result {
                Ok(Ok(resp)) => return Ok(resp),
                Ok(Err(err)) => {
                    last_err = err;
                    // Don't retry auth errors
                    let msg = last_err.to_string().to_lowercase();
                    if msg.contains("401") || msg.contains("invalid api key") {
                        break;
                    }
                    // Exponential backoff between retries
                    let wait = Duration::from_millis(500 * 2u64.pow(attempt - 1));
                    tokio::time::sleep(wait).await;
                }
                Err(_timeout) => {
                    last_err = anyhow::anyhow!("request timed out after {:?}", self.timeout);
                }
            }
        }

        // Try fallback
        if let Some(fb) = &self.fallback {
            tracing::warn!("Primary provider failed, using fallback: {}", fb.name());
            return fb.chat(req).await;
        }

        Err(last_err)
    }
}
```

Use it in `main.rs`:

```rust
let provider = Box::new(ReliableModelProvider::new(
    Box::new(OpenAiProvider::new(api_key, model))
).with_fallback(
    Box::new(OpenAiProvider::new(api_key2, "gpt-4o-mini".into()))
));
```

---

## 12. Step 10 — Security Basics

```rust
// crates/myagent-runtime/src/security.rs

/// Scrub well-known secret patterns from a string before logging or displaying.
pub fn scrub_credentials(text: &str) -> String {
    // API keys: sk-..., gh_..., pk_...
    let sk_pattern = regex::Regex::new(r"\b(sk-[A-Za-z0-9]{20,}|gh_[A-Za-z0-9]{36})\b").unwrap();
    sk_pattern.replace_all(text, "[REDACTED]").to_string()
}

/// Validate that a path does not escape the workspace root (path traversal).
pub fn validate_path(workspace: &std::path::Path, user_path: &str) -> anyhow::Result<std::path::PathBuf> {
    let full = workspace.join(user_path);
    let canonical = full.canonicalize()
        .map_err(|_| anyhow::anyhow!("path does not exist: {user_path}"))?;

    if !canonical.starts_with(workspace) {
        return Err(anyhow::anyhow!("path traversal blocked: {user_path}"));
    }
    Ok(canonical)
}
```

Apply scrubbing in the agent loop:

```rust
// In execute_tool:
let raw_output = tool.execute(args).await?;
let safe_output = myagent_runtime::security::scrub_credentials(&raw_output.output);
```

---

## 13. Running and Testing

```bash
# Build and run
cargo run -- chat

# Run tests
cargo test --workspace

# Check for compile errors without linking
cargo check --all-targets
```

Sample session:

```
myagent ready. Type your message (Ctrl+D to quit).

You: What files are in the current directory?
Assistant: Let me check that for you.
[Calling shell: ls -la]
total 48
drwxr-xr-x  5 alice staff  160 Jan 15 09:30 .
drwxr-xr-x 12 alice staff  384 Jan 15 09:00 ..
-rw-r--r--  1 alice staff 8192 Jan 15 09:30 Cargo.toml
...

You: Remember that I prefer dark mode in all tools.
[Calling memory_store: key=preference_dark_mode value="Prefers dark mode in all tools"]
Assistant: Got it! I'll remember that you prefer dark mode in all tools.
```

---

## 14. What to Add Next

Now that you have the minimal skeleton, here are the natural next steps
in order of value:

| Step | What | Why |
|------|------|-----|
| 1 | `FileReadTool`, `FileWriteTool` | Core utility tools |
| 2 | Config secrets (encrypt API key at rest) | Security |
| 3 | Session persistence (SQLite) | History survives restart |
| 4 | Second channel (Telegram) | Real-world utility |
| 5 | Streaming responses | Better UX for long answers |
| 6 | Cost tracking | Budget awareness |
| 7 | Cron scheduler | Proactive tasks |
| 8 | Web search tool | Knowledge beyond training |
| 9 | Embeddings + Qdrant | Semantic memory at scale |
| 10 | Admin approval loop | Safe autonomy for risky tools |

---

## 15. Key Design Decisions Recap

These are the decisions ZeroClaw made that you should replicate in your own project:

| Decision | What | Why |
|----------|------|-----|
| Microkernel traits | All extension points are traits in one crate | New providers/tools require zero runtime changes |
| `async-trait` | `async fn` in traits | Trait objects work with async code |
| `Arc<dyn Trait>` | Reference-counted trait objects | Multiple owners, no lifetime battles |
| `anyhow::Result` | Single error type everywhere | Ergonomic `?` propagation |
| `Box<dyn Tool>` for tool list | Runtime-assembled tool list | No generics explosion |
| Workspace resolver = 2 | Separate feature resolution per crate | Feature flags don't leak between crates |
| `reqwest` with `rustls-tls` | Pure-Rust TLS | No system OpenSSL dependency |
| `sqlx` with compile-time checked queries | Type-safe SQL | SQL errors at compile time, not runtime |
| Scrub credentials before logging | Security | Prevents accidental secret leakage |
| Exponential backoff in reliable wrapper | Resilience | Handles transient API failures gracefully |
