# ZeroClaw — Learning Guide

This folder is a complete, line-by-line walkthrough of the ZeroClaw project
for someone learning Rust at the same time. Every document explains **what**
the code does, **why** a specific decision was made, and **how** you could
build the same thing from scratch.

---

## Reading Order

| # | Document | What you will learn |
|---|----------|---------------------|
| 1 | [01-rust-fundamentals.md](./01-rust-fundamentals.md) | Core Rust concepts — ownership, traits, async, macros — as they appear in this project |
| 2 | [02-project-architecture.md](./02-project-architecture.md) | High-level design: microkernel architecture, stability tiers, ingress policy, key decisions |
| 3 | [03-workspace-and-crates.md](./03-workspace-and-crates.md) | Cargo workspaces, every crate and its role (incl. zeroclaw-spawn, zeroclaw-eval, zerocode), dependency graph |
| 4 | [04-core-traits-and-api.md](./04-core-traits-and-api.md) | `zeroclaw-api` crate deep-dive: traits, task-locals, attribution, new ingress/principal types |
| 5 | [05-config-system.md](./05-config-system.md) | TOML config, secrets, the `Configurable` derive macro |
| 6 | [06-runtime-and-agent-loop.md](./06-runtime-and-agent-loop.md) | The agent loop, tool execution, security policy |
| 7 | [07-channels-and-providers.md](./07-channels-and-providers.md) | Model providers (OpenAI, Anthropic, Gemini CLI, Telnyx, …) and messaging channels |
| 8 | [08-tools-memory-security.md](./08-tools-memory-security.md) | Built-in tools, memory backends, sandbox, prompt-injection defence |
| 9 | [09-build-your-own-guide.md](./09-build-your-own-guide.md) | End-to-end guide: build a similar AI agent runtime from zero |
| 10 | [10-frameworks-and-dependencies.md](./10-frameworks-and-dependencies.md) | Every external crate: what it does, why it was chosen, alternatives |

---

## How to use these documents

- If you are **brand new to Rust**, start with document 1 and read every
  section before touching the source code.
- If you **know Rust but not the project**, start with document 2.
- If you want to **add a feature** (new channel, new tool, new provider),
  jump to document 4 which explains the trait contracts.
- If you want to **build something similar from scratch**, document 9 is a
  complete playbook.

---

## Project in one sentence

ZeroClaw is a fully async, trait-driven, multi-platform AI agent runtime
written in Rust that connects to any LLM backend (OpenAI, Anthropic, Gemini,
Ollama, Telnyx, …) and any messaging channel (Telegram, Slack, Discord, CLI, …),
executes tools securely, persists memory, and enforces a configurable security
policy — all from a single compiled binary, with a companion TUI client
(`zerocode`) that connects via JSON-RPC.
