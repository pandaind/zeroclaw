# 08 — Tools, Memory, and Built-in Capabilities

This document covers the three capabilities that make the agent useful beyond
pure conversation: the tool library, the memory system, and the security
subsystem's leak prevention.

---

## Table of Contents

1. [The Tool Library — Overview](#1-the-tool-library--overview)
2. [Shell Tool](#2-shell-tool)
3. [File Tools](#3-file-tools)
4. [Web Tools](#4-web-tools)
5. [Memory Tools (agent-facing)](#5-memory-tools-agent-facing)
6. [`send_via` — Per-Turn Output Routing Tool](#6-send_via--per-turn-output-routing-tool)
7. [MCP Tool Support](#7-mcp-tool-support)
8. [The Memory System — Architecture](#8-the-memory-system--architecture)
9. [Memory Backends](#9-memory-backends)
10. [Memory Epic A — Durable Store Seam](#10-memory-epic-a--durable-store-seam)
11. [Memory Decay and Scoring](#11-memory-decay-and-scoring)
12. [Memory Key Conventions](#12-memory-key-conventions)
13. [Knowledge Graph](#13-knowledge-graph)
14. [Embeddings and Semantic Search](#14-embeddings-and-semantic-search)
15. [OTel Content Policy — Privacy-Aware Observability](#15-otel-content-policy--privacy-aware-observability)
16. [The Leak Detector](#16-the-leak-detector)
17. [The Audit Logger](#17-the-audit-logger)

---

## 1. The Tool Library — Overview

`crates/zeroclaw-tools/src/` contains ~60 tool modules. Each implements the
`Tool` trait from `zeroclaw-api`. Here is a functional grouping:

| Category | Tools |
|----------|-------|
| Shell/Process | `shell` (runtime), `pipeline` |
| Files | `file_edit`, `file_write`, `glob_search`, `content_search`, `pdf_read` |
| Web | `web_fetch`, `web_search_tool`, `browser`, `text_browser`, `http_request` |
| Memory | `memory_store`, `memory_recall`, `memory_forget`, `memory_export`, `memory_purge` || Routing | `send_via` (per-turn output routing) || Git | `git_operations` |
| Images | `image_gen`, `image_info`, `screenshot` |
| LLM sub-tasks | `llm_task`, `claude_code`, `codex_cli`, `gemini_cli` |
| External APIs | `google_workspace`, `jira_tool`, `notion_tool`, `weather_tool`, `pushover` |
| Hardware | `hardware_board_info`, `hardware_memory_map`, `hardware_memory_read` |
| MCP | `mcp_tool`, `mcp_client` |
| Utility | `calculator`, `ask_user`, `escalate`, `sessions`, `knowledge_tool` |

The tools are registered in `crates/zeroclaw-tools/src/lib.rs` and assembled
by the runtime's tool factory before each agent session.

---

## 2. Shell Tool

The shell tool is the most powerful — and most dangerous — built-in tool.

```rust
// In crates/zeroclaw-runtime/src/tools/ (shell.rs)

pub struct ShellTool {
    security_policy: Arc<SecurityPolicy>,
    sandbox: Arc<dyn Sandbox>,
    workspace: PathBuf,
}

#[async_trait]
impl Tool for ShellTool {
    fn name(&self) -> &str { "shell" }

    fn description(&self) -> &str {
        "Execute a shell command. Use for running programs, scripts, \
         file management, and system operations."
    }

    fn parameters_schema(&self) -> serde_json::Value {
        json!({
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The shell command to execute"
                },
                "working_dir": {
                    "type": "string",
                    "description": "Working directory (optional, defaults to workspace)"
                }
            },
            "required": ["command"]
        })
    }

    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
        let command = args["command"].as_str()
            .ok_or_else(|| anyhow!("missing 'command' argument"))?;

        // Security: validate the command against the policy
        self.security_policy.validate_command_execution(command, false)?;

        // Execute in the sandbox (or directly if no sandbox)
        let output = self.sandbox.run("sh", &["-c", command], &[]).await?;

        // Scrub credentials from output before returning
        let stdout = scrub_credentials(&output.stdout);
        let stderr = scrub_credentials(&output.stderr);

        Ok(ToolResult {
            success: output.exit_code == 0,
            output: format!("stdout:\n{stdout}\nstderr:\n{stderr}"),
            error: if output.exit_code != 0 {
                Some(format!("exit code: {}", output.exit_code))
            } else {
                None
            },
        })
    }
}
```

**Security constraints on the shell tool:**

```toml
# config.toml — shell tool policy
[security]
autonomy = "supervised"         # requires approval for risky commands
workspace = "/home/alice/work"  # chroot to this directory
allowed_commands = [
    "ls", "cat", "grep", "find",  # read-only commands
    "git *",                       # git commands
    "cargo *",                     # cargo commands
]
# excluded_tools = ["shell"]     # disable completely if desired
```

---

## 3. File Tools

`file_edit` and `file_write` are separated for security:
- `file_write` creates new files (lower risk)
- `file_edit` modifies existing files (higher risk — can overwrite)

```rust
// crates/zeroclaw-tools/src/file_edit.rs (simplified)
#[async_trait]
impl Tool for FileEditTool {
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
        let path = args["path"].as_str()?;
        let old_text = args["old_text"].as_str()?;
        let new_text = args["new_text"].as_str()?;

        // Security: path must be inside the workspace
        let full_path = self.workspace.join(path).canonicalize()?;
        if !full_path.starts_with(&self.workspace) {
            return Ok(ToolResult {
                success: false,
                output: String::new(),
                error: Some("path traversal blocked".into()),
            });
        }

        let content = tokio::fs::read_to_string(&full_path).await?;

        // Exact string replacement — avoids accidental partial matches
        if !content.contains(old_text) {
            return Ok(ToolResult {
                success: false,
                output: String::new(),
                error: Some("old_text not found in file".into()),
            });
        }

        let new_content = content.replacen(old_text, new_text, 1);
        tokio::fs::write(&full_path, new_content).await?;

        Ok(ToolResult { success: true, output: "File updated".into(), error: None })
    }
}
```

`glob_search` and `content_search` expose the filesystem search tools:

```rust
// Glob search: find files matching a pattern
// e.g. args = {"pattern": "**/*.rs", "max_results": 50}
impl Tool for GlobSearchTool {
    async fn execute(&self, args: Value) -> anyhow::Result<ToolResult> {
        let pattern = args["pattern"].as_str()?;
        let matches = glob::glob(&self.workspace.join(pattern).to_string_lossy())?
            .take(max_results)
            .filter_map(|e| e.ok())
            .map(|p| p.to_string_lossy().to_string())
            .collect::<Vec<_>>();
        Ok(ToolResult { success: true, output: matches.join("\n"), error: None })
    }
}
```

---

## 4. Web Tools

`web_fetch` downloads a URL and returns its text content:

```rust
// crates/zeroclaw-tools/src/web_fetch.rs
async fn execute(&self, args: Value) -> anyhow::Result<ToolResult> {
    let url = args["url"].as_str()?;

    // Security: check URL against domain allowlist/denylist
    self.prompt_guard.inspect("web_fetch", &args)?;

    let response = self.http_client
        .get(url)
        .timeout(Duration::from_secs(30))
        .send()
        .await?;

    let content = match response.headers()["content-type"].to_str() {
        Ok(ct) if ct.contains("text/html") => {
            // Strip HTML tags → readable text
            html2text::from_read(response.bytes().await?.as_ref(), 80)
        }
        _ => response.text().await?,
    };

    // Truncate to prevent context overflow
    let truncated = truncate_with_ellipsis(&content, 8_000);
    Ok(ToolResult { success: true, output: truncated, error: None })
}
```

`web_search_tool` abstracts over multiple search providers:

```toml
# config.toml
[tools.web_search]
provider = "brave"    # or "google", "serpapi", "tavily"
api_key = ""          # set via `zeroclaw config set-secret`
```

---

## 5. Memory Tools (agent-facing)

The agent can directly manipulate its own memory:

```
memory_store    → Save a key-value fact
memory_recall   → Semantic or keyword search
memory_forget   → Delete a key
memory_purge    → Delete all memories matching a pattern
memory_export   → Export memories to a file (GDPR)
```

These are thin wrappers around the `Memory` trait that the runtime already
uses internally. The agent calling `memory_store` writes to the same backend
as the runtime's auto-save system.

```rust
// crates/zeroclaw-tools/src/memory_store.rs
#[async_trait]
impl Tool for MemoryStoreTool {
    async fn execute(&self, args: Value) -> anyhow::Result<ToolResult> {
        let key = args["key"].as_str()?.to_string();
        let value = args["value"].as_str()?.to_string();
        let category = args["category"]
            .as_str()
            .and_then(|s| MemoryCategory::from_str(s).ok())
            .unwrap_or(MemoryCategory::General);

        self.memory.store(&key, &value, category, None).await?;
        Ok(ToolResult {
            success: true,
            output: format!("Stored: {key}"),
            error: None,
        })
    }
}
```

---

## 6. `send_via` — Per-Turn Output Routing Tool

The `send_via` tool lets the agent control **where and how** its reply is
delivered for the current turn. It is the only tool that manipulates routing
rather than data.

```
# Routing-only (no body) — redirects the main reply
send_via(target: "discord.main")                  # send reply to different channel
send_via(modality: "voice")                       # force text→voice conversion
send_via(target: "telegram.personal", modality: "text")  # both

# Fanout (with body) — sends a separate message; main reply still goes to origin
send_via(target: "email.default", body: "Detailed digest...")
```

Authorisation: targets are constrained to channels in peer groups that include
the active agent. The agent cannot route to an arbitrary channel.

Modality resolution priority:
1. Explicit `modality` parameter
2. Peer group `output_modality`
3. Text (channel default)

Implementation detail: `send_via` writes into a `TURN_ROUTING` task-local
variable scoped by the orchestrator around each `run_tool_call_loop` call.
This ensures concurrent turns cannot cross-contaminate each other's routing
instructions even though they share the same `SendViaTool` instance.

---

## 7. MCP Tool Support

ZeroClaw supports the **Model Context Protocol** (MCP) — an open standard
for connecting agents to external tool servers:

```toml
# config.toml
[[tools.mcp_servers]]
name = "filesystem"
command = "uvx mcp-server-filesystem /tmp/sandbox"

[[tools.mcp_servers]]
name = "github"
command = "uvx mcp-server-github"
env = { GITHUB_TOKEN = "" }   # set via set-secret
```

At startup, the MCP client:
1. Launches each server as a subprocess
2. Connects via stdin/stdout JSON-RPC
3. Fetches the server's tool manifest
4. Wraps each tool as a `Box<dyn Tool>` with name `mcp_<server>_<tool>`

```rust
// crates/zeroclaw-tools/src/mcp_tool.rs
pub struct McpTool {
    server_name: String,
    tool_name: String,
    spec: ToolSpec,
    client: Arc<McpClient>,
}

#[async_trait]
impl Tool for McpTool {
    fn name(&self) -> &str { &self.spec.name }  // "mcp_filesystem_read_file"
    async fn execute(&self, args: Value) -> anyhow::Result<ToolResult> {
        self.client.call_tool(&self.tool_name, args).await
    }
}
```

The `filter_tool_specs_for_turn` function from document 06 applies
specifically to MCP tools — operators can configure which MCP tools the
agent sees in which contexts, keeping the tool list manageable.

---

## 8. The Memory System — Architecture

```
crates/zeroclaw-memory/src/
├── traits.rs          ← Memory trait + MemoryEntry + MemoryCategory
├── sqlite.rs          ← SQLite backend (default)
├── markdown.rs        ← Plain markdown files backend
├── qdrant.rs          ← Qdrant vector database backend
├── postgres.rs        ← PostgreSQL backend
├── embeddings.rs      ← Text embedding generation
├── decay.rs           ← Time-based relevance decay
├── consolidation.rs   ← Merging/summarising old memories
├── retrieval.rs       ← Hybrid search (keyword + semantic)
├── snapshot.rs        ← Point-in-time export/restore
└── knowledge_graph.rs ← Neo4j/Postgres knowledge graph
```

### Memory categories

```rust
pub enum MemoryCategory {
    Core,          // Critical facts — highest recall priority, no decay
    Facts,         // Factual knowledge about the user/domain
    General,       // General notes
    Daily,         // Day-specific entries (auto-expire after 30 days)
    Conversation,  // Auto-saved conversation turns (excluded from cron context)
    Procedural,    // How-to instructions for recurring tasks
    Knowledge,     // Domain knowledge base entries
}
```

Categories affect:
- **Recall priority**: `Core` entries always appear; `Conversation` entries
  have lower weight
- **Decay rate**: `Daily` entries decay faster; `Core` entries don't decay
- **Context filtering**: `Conversation` excluded from headless (cron) runs

---

## 9. Memory Backends

### SQLite (default)

```rust
// crates/zeroclaw-memory/src/sqlite.rs
pub struct SqliteMemory {
    pool: SqlitePool,
}

impl Memory for SqliteMemory {
    async fn store(&self, key, value, category, session_id) -> Result<()> {
        sqlx::query!(
            "INSERT OR REPLACE INTO memories (key, value, category, session_id, timestamp)
             VALUES (?, ?, ?, ?, ?)",
            key, value, category.to_string(), session_id, now()
        ).execute(&self.pool).await?;
        Ok(())
    }

    async fn recall(&self, query, limit, session_id, ...) -> Result<Vec<MemoryEntry>> {
        // FTS5 full-text search
        sqlx::query_as!(MemoryRow,
            "SELECT *, rank FROM memories WHERE memories MATCH ?1
             ORDER BY rank LIMIT ?2",
            query, limit
        ).fetch_all(&self.pool).await
        .map(|rows| rows.into_iter().map(MemoryEntry::from).collect())
    }
}
```

SQLite's FTS5 module provides keyword search with relevance ranking. This
is the default backend — no external dependencies.

### Markdown (human-readable)

```rust
// crates/zeroclaw-memory/src/markdown.rs
// Stores each memory as a file: ~/.config/zeroclaw/memory/key.md
// with YAML frontmatter for metadata

// File format:
// ---
// category: facts
// timestamp: 2024-01-15T09:30:00Z
// ---
// Alice prefers TypeScript over JavaScript for all web projects.
```

Good for inspecting and editing memories manually. Limited search capability
(grep-based).

### Qdrant (semantic vector search)

```rust
// crates/zeroclaw-memory/src/qdrant.rs
pub struct QdrantMemory {
    client: QdrantClient,
    embedder: Arc<dyn Embedder>,
    collection: String,
}

impl Memory for QdrantMemory {
    async fn store(&self, key, value, ...) -> Result<()> {
        // Generate embedding vector from text
        let embedding = self.embedder.embed(value).await?;
        // Store vector + payload in Qdrant
        self.client.upsert_points(&self.collection, vec![
            PointStruct {
                id: hash_key(key),
                vector: embedding,
                payload: json!({ "key": key, "value": value, "category": category })
            }
        ]).await?;
        Ok(())
    }

    async fn recall(&self, query, limit, ...) -> Result<Vec<MemoryEntry>> {
        let query_vec = self.embedder.embed(query).await?;
        // Cosine similarity search
        let results = self.client.search_points(&self.collection, query_vec, limit).await?;
        Ok(results.into_iter().map(MemoryEntry::from).collect())
    }
}
```

Qdrant is the premium backend for large memory stores where keyword search
is insufficient. It requires running a Qdrant server (Docker or cloud).

---

## 10. Memory Epic A — Durable Store Seam

Epic A introduced a policy-aware, CAS-safe write pipeline with four new
modules:

### `dedup.rs` — Deduplication Gate

Before a memory entry is written, it is checked against existing entries for
the same category. The gate uses Jaccard similarity for fuzzy matching:

```rust
// crates/zeroclaw-memory/src/dedup.rs
pub enum DedupAction {
    Insert,                          // no duplicate found
    Reject { dup_of: String },       // duplicate exists, reject write
    Merge { into: String },          // merge new content into existing
}

pub fn dedup_gate(
    candidates: &[MemoryEntry],
    incoming: &str,
    cfg: &MemoryConfig,
) -> DedupAction {
    if !cfg.dedup_on_write { return DedupAction::Insert; }

    // Exact-match check
    if let Some(existing) = candidates.iter().find(|e| e.content == incoming) {
        return match cfg.dedup_action {
            MemoryDedupAction::Reject => DedupAction::Reject { dup_of: existing.id.clone() },
            MemoryDedupAction::Merge  => DedupAction::Merge  { into:   existing.id.clone() },
        };
    }

    // Fuzzy match via Jaccard coefficient
    if let Some(dup_of) = conflict::find_text_conflicts(
        candidates, incoming, cfg.dedup_jaccard_threshold
    ).into_iter().next() {
        return match cfg.dedup_action {
            MemoryDedupAction::Reject => DedupAction::Reject { dup_of },
            MemoryDedupAction::Merge  => DedupAction::Merge  { into: dup_of },
        };
    }

    DedupAction::Insert
}
```

### `budget.rs` — Eviction Engine

Each memory category can have a row count cap and a byte cap. When exceeded,
the eviction engine removes the least important non-pinned entries:

```rust
// crates/zeroclaw-memory/src/budget.rs
pub struct EvictionReport {
    pub evicted_by_count: u64,
    pub evicted_by_bytes: u64,
    pub pinned_skipped: u64,      // entries with pinned=true are immune
}

// Called after each write to enforce budget
pub fn compact_category_to_budget(
    conn: &Connection,
    category: &str,
    cfg: &MemoryConfig,
) -> Result<EvictionReport> {
    // Order by importance ASC or created_at ASC (configurable via evict_order)
    // DELETE the excess rows, skipping pinned=1 entries
    ...
}
```

Config:
```toml
[memory]
dedup_on_write = true
dedup_jaccard_threshold = 0.85   # 85% word overlap = duplicate
dedup_action = "reject"          # or "merge"
core_max_rows = 500              # evict oldest core entries after 500
core_max_bytes = 2097152         # 2 MB hard cap for core category
evict_order = "value"            # evict lowest-importance first (vs "oldest")
```

### `policy_gate.rs` — Write-Time Policy Validation

Before any write, validates namespace and category limits from `PolicyEnforcer`:

```rust
// crates/zeroclaw-memory/src/policy_gate.rs
pub async fn validate_store(
    memory: &dyn Memory,
    policy: &PolicyEnforcer,
    namespace: &str,
    category: &MemoryCategory,
) -> Result<(), PolicyViolation> {
    policy.validate_store(namespace, category)?;  // category-level check
    // Count existing entries, reject if over namespace limit
    let ns_count = memory.count_in_scope(Some(namespace), None).await?;
    policy.check_namespace_limit(ns_count)?;
    Ok(())
}
```

### `superseded_by` and `pinned` Fields

Two new fields on `MemoryEntry`:

- **`superseded_by: Option<String>`** — soft-delete via CAS (Compare-And-Swap)
  chain. When a memory is updated, the old version gets
  `superseded_by = new_id` rather than being deleted. Queries filter
  `WHERE superseded_by IS NULL` to see only current entries. This allows
  auditing the full update history.

- **`pinned: bool`** — if `true`, the entry is exempt from budget eviction.
  Operators can pin critical facts (e.g. `core/user-name`) to ensure they
  are never automatically removed regardless of category caps.

---

## 11. Memory Decay and Scoring

Older memories should have less influence than recent ones. The decay system
applies a time-based score penalty:

```rust
// crates/zeroclaw-memory/src/decay.rs
pub const DEFAULT_HALF_LIFE_DAYS: f64 = 30.0;

/// Penalise entries by age. Core entries are exempt.
pub fn apply_time_decay(entries: &mut Vec<MemoryEntry>, half_life_days: f64) {
    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs_f64();

    for entry in entries.iter_mut() {
        if entry.category == MemoryCategory::Core { continue; }  // exempt

        let age_days = (now - entry.timestamp as f64) / 86_400.0;
        let decay_factor = 0.5_f64.powf(age_days / half_life_days);
        // At half_life_days=30: 30-day-old entry → 50% score, 60-day → 25%

        if let Some(score) = &mut entry.score {
            *score *= decay_factor;
        }
    }
}
```

The formula $\text{score} \times 0.5^{t / h}$ where $t$ is age in days and
$h$ is the half-life gives exponential decay:
- 30 days old, half-life 30 → 50% weight
- 60 days old → 25% weight
- 90 days old → 12.5% weight

---

## 12. Memory Key Conventions

Reserved key prefixes (filtered from context injection):

| Prefix | Purpose | Never fed back into context |
|--------|---------|----------------------------|
| `assistant_resp_*` | Auto-saved assistant summaries | Yes |
| `user_msg_*` | Raw per-turn user messages | Yes |

Safe key patterns (use these for semantic memories):

```
alice_preference_language    → "Prefers TypeScript"
project_stack_myapp          → "Next.js + Prisma + PostgreSQL"
fact_birthday                → "Birthday is March 15"
daily_2024_01_15_summary     → "Worked on issue #42 today"
```

The `is_assistant_autosave_key` and `is_user_autosave_key` functions check
these prefixes to avoid echoing auto-saved raw text back into the prompt
(which would grow exponentially).

---

## 13. Knowledge Graph

For structured domain knowledge, ZeroClaw supports a knowledge graph backend:

```rust
// crates/zeroclaw-memory/src/knowledge_graph.rs
pub struct KnowledgeGraph {
    // Nodes and edges stored in SQLite or PostgreSQL
}

// Queried via:
// Entity → relations → related entities
// e.g. "Alice" → "works on" → "Project X"
//      "Project X" → "uses" → "React"
//      "React" → "documentation at" → "https://react.dev"
```

The knowledge graph is used by the `knowledge_tool` to answer structured
queries like "What does Alice work on?" without consuming LLM context.

---

## 14. Embeddings and Semantic Search

Text embedding converts a string into a dense vector of numbers (e.g. 1536
dimensions for `text-embedding-3-small`). Similar texts have similar vectors.

```rust
// crates/zeroclaw-memory/src/embeddings.rs
#[async_trait]
pub trait Embedder: Send + Sync {
    async fn embed(&self, text: &str) -> anyhow::Result<Vec<f32>>;
}

pub struct OpenAiEmbedder {
    client: reqwest::Client,
    api_key: String,
    model: String,  // "text-embedding-3-small"
}

#[async_trait]
impl Embedder for OpenAiEmbedder {
    async fn embed(&self, text: &str) -> anyhow::Result<Vec<f32>> {
        let resp = self.client
            .post("https://api.openai.com/v1/embeddings")
            .bearer_auth(&self.api_key)
            .json(&json!({ "input": text, "model": &self.model }))
            .send()
            .await?
            .json::<EmbeddingResponse>()
            .await?;
        Ok(resp.data[0].embedding.clone())
    }
}
```

The `RetrievalPipeline` combines keyword search (BM25/FTS5) and semantic
search (cosine similarity) using a hybrid score:

```rust
// crates/zeroclaw-memory/src/retrieval.rs
pub struct RetrievalPipeline {
    keyword_weight: f64,    // 0.4 default
    semantic_weight: f64,   // 0.6 default
}

pub fn hybrid_score(keyword_score: f64, semantic_score: f64, weights: &RetrievalPipeline) -> f64 {
    weights.keyword_weight * keyword_score + weights.semantic_weight * semantic_score
}
```

---

## 15. OTel Content Policy — Privacy-Aware Observability

ZeroClaw exports traces and metrics to any OpenTelemetry OTLP endpoint
(Jaeger, Grafana Tempo, Datadog, etc.). Because traces may include LLM
prompts and tool arguments, the content policy controls what is exported.

```toml
# config.toml
[observability]
otel_endpoint = "http://localhost:4318"  # OTLP HTTP endpoint

# GenAI content: what goes in LLM input/output span attributes
otel_genai_content = "scrubbed"    # "off" | "scrubbed" | "full"
otel_genai_content_max_chars = 500 # truncate at 500 chars (0 = off)

# Tool I/O content: what goes in tool call span attributes
otel_tool_io = "scrubbed"          # "off" | "scrubbed" | "full"
otel_tool_io_max_chars = 200
```

Policy levels:
- **`off`**: no content in spans. Only metadata (model name, tool name, token counts).
- **`scrubbed`**: content included but run through `scrub_for_export` (same credential-scrubbing regex as the loop). Default and recommended for production.
- **`full`**: raw content included. Only for local dev/debugging environments.

Each `OtelObserver` instance holds an immutable `OtelContentConfig` snapshot
derived from `ObservabilityConfig` at construction time. This prevents
last-writer-wins drift when multiple observers with different policies are
active concurrently.

```rust
// crates/zeroclaw-runtime/src/observability/otel_config.rs
pub(crate) struct OtelContentConfig {
    pub(crate) genai_policy:       OtelContentPolicy,  // Off | Scrubbed | Full
    pub(crate) genai_max_chars:    usize,
    pub(crate) tool_io_policy:     OtelContentPolicy,
    pub(crate) tool_io_max_chars:  usize,
}
```

The OTel observer tracks metrics instruments covering:
- `agent.starts`, `agent.duration`
- `llm.calls`, `llm.duration`, `tokens.used`
- `tool.calls`, `tool.duration`
- `memory.recall.count`, `memory.store.count`
- `rag.retrieve.count`, `rag.retrieve.duration`
- `active.sessions`, `queue.depth`

---

## 16. The Leak Detector

The `LeakDetector` scans tool output and LLM responses for accidentally
included sensitive data before it is logged or sent to external services:

```rust
// crates/zeroclaw-runtime/src/security/leak_detector.rs
pub struct LeakDetector {
    patterns: Vec<LeakPattern>,
}

pub struct LeakPattern {
    name: &'static str,
    regex: Regex,
    sensitivity: Sensitivity,  // Low, Medium, High, Critical
}

// Built-in patterns:
// - Email addresses
// - Credit card numbers (Luhn-validated)
// - Social Security Numbers
// - API keys (common prefixes: sk-, gh-, pk_)
// - Private keys (BEGIN RSA/EC/OPENSSH PRIVATE KEY)
// - AWS credentials (AKIA...)
// - IP addresses + port combinations

pub struct LeakResult {
    pub has_leaks: bool,
    pub redacted: String,
    pub detections: Vec<Detection>,
}

impl LeakDetector {
    pub fn redact(&self, text: &str) -> LeakResult {
        let mut result = text.to_string();
        let mut detections = vec![];

        for pattern in &self.patterns {
            result = pattern.regex.replace_all(&result, |caps: &regex::Captures| {
                detections.push(Detection { name: pattern.name, ... });
                format!("[REDACTED:{}]", pattern.name)
            }).to_string();
        }

        LeakResult {
            has_leaks: !detections.is_empty(),
            redacted: result,
            detections,
        }
    }
}
```

The leak detector runs:
1. On all tool output before it is appended to chat history
2. On all LLM responses before they are logged
3. On all outbound channel messages (as a last-line defence)

---

## 17. The Audit Logger

All security-relevant events are written to an append-only audit log:

```rust
// crates/zeroclaw-runtime/src/security/audit.rs
pub struct AuditLogger {
    writer: Arc<Mutex<File>>,
    path: PathBuf,
}

pub enum AuditEventType {
    ToolCall { tool: String, args_hash: String },
    ToolBlocked { tool: String, reason: String },
    CommandRejected { command: String, reason: String },
    ApprovalRequested { tool: String },
    ApprovalGranted { tool: String },
    ApprovalDenied { tool: String },
    PromptInjectionDetected { source: String },
    LeakDetected { pattern: String },
    UnauthorisedAccess { sender: String, channel: String },
}

pub struct AuditEvent {
    pub timestamp: SystemTime,
    pub session_key: String,
    pub event: AuditEventType,
}
```

The audit log uses JSONL format (one JSON object per line) for easy parsing:

```jsonl
{"timestamp":"2024-01-15T09:30:00Z","session":"telegram:bot:alice","event":"ToolCall","tool":"shell","args_hash":"sha256:abc123"}
{"timestamp":"2024-01-15T09:30:01Z","session":"telegram:bot:alice","event":"ToolBlocked","tool":"shell","reason":"command not in allowed_commands"}
```

This log is forensically useful — it is append-only (never truncated), so even
if the agent is compromised, past events cannot be erased.
