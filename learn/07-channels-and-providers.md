# 07 — Channels and Providers

This document explains how ZeroClaw connects to messaging platforms (channels)
and LLM APIs (providers). Both are pluggable systems built on the traits
defined in `zeroclaw-api`.

---

## Table of Contents

1. [The Two-Sided Architecture](#1-the-two-sided-architecture)
2. [Channels — The 30+ Integrations](#2-channels--the-30-integrations)
3. [The Orchestrator — Channel Lifecycle](#3-the-orchestrator--channel-lifecycle)
4. [Message Routing — Handler Chain](#4-message-routing--handler-chain)
5. [Session History per Sender](#5-session-history-per-sender)
6. [Providers — LLM API Integrations](#6-providers--llm-api-integrations)
7. [The Factory Pattern](#7-the-factory-pattern)
8. [The `CompatFamilySpec` Blanket Trait](#8-the-compatfamilyspec-blanket-trait)
9. [The `ReliableModelProvider` Wrapper](#9-the-reliablemodelprovider-wrapper)
10. [Error Classification and Retry Logic](#10-error-classification-and-retry-logic)
11. [Provider Fallback Notification](#11-provider-fallback-notification)
12. [Adding a New Channel](#12-adding-a-new-channel)
13. [Adding a New Provider](#13-adding-a-new-provider)

---

## 1. The Two-Sided Architecture

ZeroClaw sits in the middle:

```
          INBOUND                         OUTBOUND
  ┌───────────────────┐              ┌──────────────────┐
  │  Telegram          │              │  OpenAI / GPT    │
  │  Discord           │              │  Anthropic       │
  │  Slack             │   ZeroClaw   │  Gemini          │
  │  WhatsApp          │◄────────────►│  Ollama (local)  │
  │  Email             │              │  OpenRouter      │
  │  CLI               │              │  Azure OpenAI    │
  │  30+ more          │              │  10+ more        │
  └───────────────────┘              └──────────────────┘
         channels                         providers
```

Channels bring messages *in*. Providers process them and generate replies.
The runtime translates between the two without either knowing about the other.

---

## 2. Channels — The 30+ Integrations

```
crates/zeroclaw-channels/src/
├── telegram.rs          # Telegram Bot API
├── discord.rs           # Discord Gateway WebSocket
├── slack.rs             # Slack Bolt (Events API + RTM)
├── whatsapp.rs          # WhatsApp Cloud API (Meta)
├── whatsapp_web.rs      # WhatsApp Web (unofficial, no Meta key)
├── email_channel.rs     # IMAP/SMTP email
├── gmail_push.rs        # Gmail Push Notifications (PubSub)
├── signal.rs            # Signal messenger
├── irc.rs               # IRC protocol
├── matrix.rs            # Matrix (Element)
├── bluesky.rs           # Bluesky / AT Protocol
├── nostr.rs             # Nostr decentralised protocol
├── reddit.rs            # Reddit bot
├── twitter.rs           # Twitter/X API
├── notion.rs            # Notion AI assistant
├── cli.rs               # Interactive terminal (default)
├── webhook.rs           # Generic HTTP webhook receiver
├── voice_call.rs        # VoIP/WebRTC calls
├── voice_wake.rs        # Always-on voice wake word detection
├── transcription.rs     # Speech-to-text pipeline
└── ...
```

Every channel is behind a Cargo feature gate:

```toml
# crates/zeroclaw-channels/Cargo.toml
[features]
default = []
channel-telegram = ["teloxide"]       # adds the telegram.rs file
channel-discord = ["serenity"]
channel-slack = ["slack-morphism"]
channel-matrix = ["matrix-sdk"]
channel-email = ["mail_parser", "lettre"]
channel-nostr = ["nostr-sdk"]
channel-voice-call = ["webrtc"]
```

The root `Cargo.toml` bundles these into higher-level features:

```toml
# Root Cargo.toml
[features]
default = ["agent-runtime"]
agent-runtime = [
    "zeroclaw-channels/channel-telegram",
    "zeroclaw-channels/channel-discord",
    "zeroclaw-channels/channel-slack",
    # ... bundled defaults
]
```

**Why feature gates?** The compiled binary only includes code for channels
the operator enables. An embedded deployment that only needs Telegram doesn't
pay the binary size or compilation time of the Slack SDK.

---

## 3. The Orchestrator — Channel Lifecycle

```rust
// crates/zeroclaw-channels/src/orchestrator/mod.rs

/// Start all configured channels and wire them to the agent dispatcher.
pub async fn start_channels(
    config: Arc<RwLock<Config>>,
    dispatcher: Arc<MessageDispatcher>,
) -> Result<()> {
    let cfg = config.read();

    // Start each enabled channel type
    #[cfg(feature = "channel-telegram")]
    if let Some(telegram_configs) = &cfg.channels.telegram {
        for (alias, channel_cfg) in telegram_configs {
            if !channel_cfg.enabled { continue; }

            let token = channel_cfg.bot_token.clone();
            let dispatcher = Arc::clone(&dispatcher);

            tokio::spawn(async move {
                // Each channel runs in its own Tokio task
                let channel = TelegramChannel::new(token, alias.clone());
                run_channel_with_reconnect(channel, dispatcher).await;
            });
        }
    }
    // ... similar for Discord, Slack, etc.

    Ok(())
}
```

`run_channel_with_reconnect` wraps the channel's `listen()` call with
exponential backoff:

```rust
async fn run_channel_with_reconnect(
    channel: impl Channel + 'static,
    dispatcher: Arc<MessageDispatcher>,
) {
    let mut backoff = Duration::from_secs(1);

    loop {
        let result = channel.listen(dispatcher.handler_fn()).await;

        match result {
            Ok(()) => {
                // Channel stopped cleanly (shutdown signal)
                break;
            }
            Err(e) => {
                // Connection dropped — log and retry
                tracing::warn!("Channel {} disconnected: {e}", channel.name());
                tokio::time::sleep(backoff).await;
                backoff = (backoff * 2).min(Duration::from_secs(300)); // cap at 5 min
            }
        }
    }
}
```

---

## 4. Message Routing — Handler Chain

When a message arrives from any channel, it flows through this chain:

```
1. ChannelMessage received
   │
2. AllowlistChecker.check(message)
   → Only senders in `allowed_users` pass through
   → Block and log unauthorised senders
   │
3. MessageDebouncer.submit(message)
   → Coalesces rapid successive messages (e.g. user typing fast)
   → Holds for 300ms, then releases the last version
   │
4. SessionBackend.load(sender_id)
   → Loads conversation history from SQLite
   │
5. CancellationToken management
   → If sender has in-progress turn: cancel it first
   → Create new token for this turn
   │
6. run_tool_call_loop(...)
   → THE AGENT LOOP (see document 06)
   │
7. Channel.send(response)
   → Sends reply back to the original platform
   │
8. SessionBackend.save(session)
   → Persists updated conversation history
```

---

## 5. Session History per Sender

Each (channel, sender) pair has its own conversation history:

```rust
// The session key includes channel type, alias, and sender ID
let session_key = format!("{channel_type}:{channel_alias}:{sender_id}");
// e.g. "telegram:my-bot:alice123"
//       "discord:my-server:#general:alice#0001"
```

```rust
// crates/zeroclaw-infra/src/session_sqlite.rs
pub struct SqliteSessionBackend {
    pool: SqlitePool,
}

impl SessionBackend for SqliteSessionBackend {
    async fn load(&self, key: &str) -> Vec<ChatMessage> {
        sqlx::query_as!(SessionRow, "SELECT * FROM sessions WHERE key = ?", key)
            .fetch_all(&self.pool)
            .await
            .map(|rows| rows.into_iter().map(|r| r.into()).collect())
            .unwrap_or_default()
    }

    async fn save(&self, key: &str, messages: &[ChatMessage]) {
        // Delete old rows for this session, insert new ones
        // (SQLite is fast enough for this at conversation scale)
    }
}
```

Session histories are bounded. `trim_history` is called before each turn:

```rust
// crates/zeroclaw-runtime/src/agent/history.rs
pub fn trim_history(
    messages: &mut Vec<ChatMessage>,
    max_messages: Option<usize>,
) {
    let max = max_messages.unwrap_or(100);
    if messages.len() > max {
        // Always keep system message (index 0)
        // Remove oldest user/assistant pairs from the middle
        let keep_recent = max / 2;
        messages.drain(1..messages.len() - keep_recent);
    }
}
```

---

## 6. Providers — LLM API Integrations

```
crates/zeroclaw-providers/src/
├── openai.rs         # OpenAI API (gpt-4o, gpt-4o-mini, o1, …)
├── anthropic.rs      # Anthropic API (claude-opus-4-5, …)
├── gemini.rs         # Google Gemini API
├── gemini_cli.rs     # NEW: Gemini CLI provider (uses installed gemini binary)
├── azure_openai.rs   # Azure OpenAI (same API, different auth)
├── openrouter.rs     # OpenRouter (aggregates 100+ providers)
├── openrouter_catalog.rs  # NEW: OpenRouter model catalog helpers
├── bedrock.rs        # AWS Bedrock (Claude + others)
├── ollama.rs         # Local models via Ollama
├── compatible.rs     # Generic OpenAI-compatible endpoints
├── copilot.rs        # GitHub Copilot provider
├── glm.rs            # NEW: Zhipu AI (GLM series) provider
├── kilocli.rs        # NEW: KiloCLI provider
├── openai_codex.rs   # NEW: OpenAI Codex provider (codex CLI path)
├── telnyx.rs         # NEW: Telnyx AI provider
├── multimodal.rs     # NEW: multimodal content helpers shared across providers
├── catalog.rs        # NEW: per-family catalog source lookup table
├── dispatch.rs       # Provider dispatch helpers
├── model_pin.rs      # Model pinning support
├── models_dev.rs     # models.dev integration
├── pricing.rs        # Token pricing database
├── request_payload.rs # Shared request payload helpers
├── stream_guard.rs   # Streaming token guard
├── vision_override.rs # Vision/image override helpers
├── auth/             # Provider authentication helpers
├── reliable.rs       # Wrapper: retry + fallback
└── factory.rs        # Construction dispatch
```

---

## 7. The Factory Pattern

Adding a provider to ZeroClaw involves one slot in a macro and one trait
implementation. The macro ensures the compilation fails if any slot is missing
an implementation — you cannot forget to wire a new provider.

```rust
// zeroclaw-config/src/providers.rs
macro_rules! for_each_model_provider_slot {
    ($macro:ident) => {
        $macro!(
            openai: OpenAiModelProviderConfig,
            anthropic: AnthropicModelProviderConfig,
            gemini: GeminiModelProviderConfig,
            azure_openai: AzureOpenAiModelProviderConfig,
            openrouter: OpenRouterModelProviderConfig,
            bedrock: BedrockModelProviderConfig,
            ollama: OllamaModelProviderConfig,
            // ... one row per provider
        );
    };
}
```

The `factory.rs` uses this macro to generate exhaustive dispatch:

```rust
// crates/zeroclaw-providers/src/factory.rs
pub fn create_provider(
    family: &str,
    alias: &str,
    config: &Config,
) -> Result<Box<dyn ModelProvider>> {
    // Macro expands to a match arm for every row in the slot list
    for_each_model_provider_slot!(build_from_slot)
    // If `family` matches no arm → compile error (not runtime error)
}
```

---

## 8. The `CompatFamilySpec` Blanket Trait

Most providers are "OpenAI compatible" — they speak the same HTTP API format
as OpenAI but with different base URLs and authentication styles. Instead of
writing the same provider code 10 times, there is a blanket trait:

```rust
// crates/zeroclaw-providers/src/factory.rs

/// Spec trait for OpenAI-compatible providers.
/// Implementing this gives a full FamilyProviderFactory for free.
pub trait CompatFamilySpec {
    const DISPLAY: &'static str;          // e.g. "Groq"
    const DEFAULT_URL: &'static str;      // e.g. "https://api.groq.com/openai/v1"
    const AUTH: AuthStyle;                 // Bearer vs API-Key header
    const MODELS_DEV_KEY: Option<&'static str> = None;    // for model listing
    const OPENROUTER_VENDOR_PREFIX: Option<&'static str> = None;

    fn build_compat_base(&self, alias, key, api_url) -> OpenAiCompatibleModelProvider { ... }
    fn build_compat(&self, alias, key, api_url) -> OpenAiCompatibleModelProvider {
        self.build_compat_base(alias, key, api_url)  // default: no modifiers
    }
}

// Blanket impl: if you implement CompatFamilySpec, you get FamilyProviderFactory
impl<T: CompatFamilySpec> FamilyProviderFactory for T {
    fn create_provider(&self, alias, key, api_url, opts) -> Result<Box<dyn ModelProvider>> {
        Ok(Box::new(self.build_compat(alias, key, api_url).build()))
    }
}
```

So adding a new OpenAI-compatible provider is just:

```rust
// New provider: Groq
impl CompatFamilySpec for GroqModelProviderConfig {
    const DISPLAY: &'static str = "Groq";
    const DEFAULT_URL: &'static str = "https://api.groq.com/openai/v1";
    const AUTH: AuthStyle = AuthStyle::Bearer;
    const MODELS_DEV_KEY: Option<&'static str> = Some("groq");
}
```

Five lines. No HTTP code. No retry logic. The blanket impl generates everything.

---

## 9. The `ReliableModelProvider` Wrapper

All providers are wrapped in `ReliableModelProvider` before being used by
the agent loop. This wrapper adds:

- **Retries** with exponential backoff for transient errors
- **Fallback** to a secondary provider/model if primary fails permanently
- **Timeout** per request
- **Budget enforcement** — abort if cost exceeds limit

```rust
// crates/zeroclaw-providers/src/reliable.rs
pub struct ReliableModelProvider {
    primary: Box<dyn ModelProvider>,
    fallback: Option<Box<dyn ModelProvider>>,
    retry_config: RetryConfig,
    timeout: Duration,
}

#[async_trait]
impl ModelProvider for ReliableModelProvider {
    async fn chat(&self, req: ChatRequest<'_>) -> Result<ChatResponse> {
        let mut attempt = 0;
        let mut last_error;

        loop {
            attempt += 1;

            match tokio::time::timeout(self.timeout, self.primary.chat(req)).await {
                Ok(Ok(resp)) => return Ok(resp),
                Ok(Err(err)) => {
                    last_error = err;
                    if is_non_retryable(&last_error) { break; }
                    if attempt >= self.retry_config.max_attempts { break; }

                    let wait = exponential_backoff(attempt);
                    tokio::time::sleep(wait).await;
                }
                Err(_timeout) => {
                    last_error = anyhow!("request timed out");
                    break;
                }
            }
        }

        // Primary exhausted → try fallback
        if let Some(fallback) = &self.fallback {
            record_provider_fallback(primary_name, primary_model, fallback_name, fallback_model);
            return fallback.chat(req).await;
        }

        Err(last_error)
    }
}
```

---

## 10. Error Classification and Retry Logic

```rust
// crates/zeroclaw-providers/src/reliable.rs

/// Errors that will NOT get better with retries.
pub fn is_non_retryable(err: &anyhow::Error) -> bool {
    let msg = err.to_string().to_lowercase();

    // Authentication errors — retrying won't fix a bad API key
    if msg.contains("401") || msg.contains("unauthorized") || msg.contains("invalid api key") {
        return true;
    }
    // Rate limit is retryable (wait and retry will eventually work)
    // Context window exceeded is NOT non-retryable (history trimming can fix it)
    if is_context_window_exceeded(err) {
        return false;  // special case: let retry loop handle with trimming
    }
    // 4xx client errors (except rate limit) are non-retryable
    if msg.contains("400") || msg.contains("404") || msg.contains("422") {
        return true;
    }
    false  // 5xx server errors, network errors → retry
}

fn exponential_backoff(attempt: u32) -> Duration {
    let base = Duration::from_millis(500);
    let max = Duration::from_secs(30);
    (base * 2u32.pow(attempt - 1)).min(max)
    // attempt 1: 500ms, 2: 1s, 3: 2s, 4: 4s, ... cap at 30s
}
```

---

## 11. Provider Fallback Notification

When a fallback occurs (primary provider failed, switched to secondary),
the user should be told. The challenge: the failure happens deep inside
`ReliableModelProvider`; the notification must be sent by the channel code
which is many call frames up the stack.

The solution: task-local storage (same pattern as session keys):

```rust
// Write: inside ReliableModelProvider (deep in call stack)
fn record_provider_fallback(req_prov, req_model, actual_prov, actual_model) {
    PROVIDER_FALLBACK.try_with(|cell| {
        *cell.borrow_mut() = Some(ProviderFallbackInfo { ... });
    });
}

// Read: in channel code (top of call stack)
// AFTER the agent turn completes:
if let Some(fallback_info) = take_last_provider_fallback() {
    channel.send(SendMessage::new(
        format!("⚠️ Switched from {} to {} due to an error.",
            fallback_info.requested_provider,
            fallback_info.actual_provider),
        reply_target,
    )).await.ok();
}
```

No function argument threading needed across 10+ call frames.

---

## 12. Adding a New Channel

**Step 1**: Create `crates/zeroclaw-channels/src/myplatform.rs`:

```rust
use async_trait::async_trait;
use zeroclaw_api::channel::{Channel, ChannelMessage, SendMessage};

pub struct MyPlatformChannel {
    api_token: String,
    alias: String,
}

impl MyPlatformChannel {
    pub fn new(api_token: String, alias: String) -> Self {
        Self { api_token, alias }
    }
}

#[async_trait]
impl Channel for MyPlatformChannel {
    fn name(&self) -> &str {
        &self.alias
    }

    async fn listen(
        &self,
        handler: Arc<dyn Fn(ChannelMessage) -> BoxFuture<'static, ()> + Send + Sync>,
    ) -> anyhow::Result<()> {
        // Connect to the platform's WebSocket/polling endpoint
        // For each message:
        let msg = ChannelMessage {
            id: event.message_id.to_string(),
            sender: event.user_id.to_string(),
            reply_target: event.chat_id.to_string(),
            content: event.text.clone(),
            channel: "myplatform".to_string(),
            channel_alias: Some(self.alias.clone()),
            timestamp: event.timestamp,
            thread_ts: None,
            interruption_scope_id: Some(event.chat_id.to_string()),
            attachments: vec![],
        };
        handler(msg).await;
        // Loop forever until shutdown signal
        Ok(())
    }

    async fn send(&self, msg: SendMessage) -> anyhow::Result<()> {
        // POST to the platform's send message API
        reqwest::Client::new()
            .post("https://api.myplatform.com/send")
            .bearer_auth(&self.api_token)
            .json(&json!({
                "chat_id": msg.recipient,
                "text": msg.content,
            }))
            .send()
            .await?
            .error_for_status()?;
        Ok(())
    }
}
```

**Step 2**: Add a feature gate and register in the orchestrator.

**Step 3**: Add config schema (`MyPlatformConfig`) with the `#[secret]` attribute
on sensitive fields.

---

## 13. Adding a New Provider

**Option A** — OpenAI-compatible (preferred, 5 lines):

```rust
// In zeroclaw-config: add slot
myai: MyAiModelProviderConfig,

// In zeroclaw-providers:
impl CompatFamilySpec for MyAiModelProviderConfig {
    const DISPLAY: &'static str = "MyAI";
    const DEFAULT_URL: &'static str = "https://api.myai.com/v1";
    const AUTH: AuthStyle = AuthStyle::Bearer;
}
```

**Option B** — Custom API (full trait implementation):

```rust
pub struct MyAiProvider {
    api_key: String,
    client: reqwest::Client,
}

#[async_trait]
impl ModelProvider for MyAiProvider {
    async fn chat(&self, req: ChatRequest<'_>) -> anyhow::Result<ChatResponse> {
        // Convert req.messages to your API's format
        // POST to your endpoint
        // Parse response into ChatResponse
    }
}
```

Then wrap it in `ReliableModelProvider::new(Box::new(MyAiProvider { ... }), ...)`.
