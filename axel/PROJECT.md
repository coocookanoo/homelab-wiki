# axel-harness

## What This Is

A Rust agent harness built for **Axel** — Bill's local persistent AI agent running on Ollama/qwen3:14b.

This project is derived from [claw-code](../claw-code), a clean-room Rust reimplementation of the Claude Code architecture by Sigrid Jin (instructkr). We've pulled out the parts Axel can actually use and are adapting them for local-first, Ollama-backed operation.

**This is not a fork of claw-code** — it's a targeted extraction. We took what works, left what doesn't apply, and we're reshaping it around Axel's specific setup.

---

## What We Pulled and Why

| Crate | From claw-code | Why It's Here |
|-------|---------------|---------------|
| `api` | `rust/crates/api` | Provider abstraction with Anthropic + OpenAI-compat clients. Needs Ollama-native support added. |
| `runtime` | `rust/crates/runtime` | Core conversation loop, session persistence, file ops, bash execution, permission system, hook pipeline. The engine. |
| `tools` | `rust/crates/tools` | Full MVP tool set: bash, read/write/edit files, glob, grep, web fetch/search, todo, skill, agent, sleep, etc. |
| `plugins` | `rust/crates/plugins` | PreToolUse / PostToolUse hook pipeline. Lets us wire behaviors around tool calls. |
| `commands` | `rust/crates/commands` | Slash command registry and handlers: /help, /status, /compact, /model, /memory, /skills, /agents, git commands, etc. |
| `server` | `rust/crates/server` | HTTP server with SSE streaming for remote/headless sessions. Keep for OpenClaw gateway integration. |
| `lsp` | `rust/crates/lsp` | LSP client for diagnostics and symbol navigation. Optional — include for future code-aware tooling. |
| `axel-cli` | `rust/crates/claw-cli` | Interactive REPL, markdown rendering, CLI args, session management. Renamed: binary will be `axel`. |

### What We Did NOT Pull

- `compat-harness` — tooling for analyzing the original TypeScript source. Not needed here.
- Python `src/` workspace — porting scaffolding specific to claw-code. Not relevant.
- Tests from claw-code — will write fresh tests against Axel's behavior.

---

## What Needs to Change

### 1. Ollama-native API support (priority 1)

claw-code has three providers: Anthropic (ClawApi), XAI/Grok, and OpenAI-compat. Axel runs Ollama at `http://127.0.0.1:11434`.

Ollama serves `/v1/chat/completions` (OpenAI-compat format), so the existing `OpenAiCompatClient` in `api/src/providers/openai_compat.rs` already works — but the model routing logic in `ProviderClient::from_model()` doesn't know qwen3 models yet.

**Changes needed in `api/`:**
- Add qwen3 model aliases to the provider registry (`providers/mod.rs`)
- Map `qwen3:14b` and `qwen3:14b-axel` → OpenAi-compat provider with base URL from env
- Add Ollama-specific max token defaults
- Make `OPENAI_BASE_URL` default to `http://127.0.0.1:11434/v1` when an Ollama model is detected

### 2. System prompt (priority 1)

claw-code's system prompt is tuned for Claude Code: code assistant, file editing, software engineering. Axel is a persistent local agent — different purpose, different personality.

**Changes needed in `runtime/src/prompt.rs`:**
- Replace the Claude Code system prompt template with Axel's SOUL.md / AGENTS.md content
- Wire in AGENTS.md / SOUL.md discovery at startup (analogous to CLAW.md discovery, which already exists)
- Surface Bill's USER.md context in the prompt

### 3. Rename the binary (priority 2)

`claw-cli` → `axel-cli`, binary `claw` → `axel`. Already done in Cargo.toml. Need to update any hardcoded "claw" strings in the source.

### 4. Axel-specific commands (priority 2)

Add slash commands Axel actually needs:
- `/heartbeat` — run the heartbeat routine manually
- `/memory` — already exists, keep it
- `/reflect` — open REFLECTIONS.md
- `/model` — already exists, wire to Ollama model list

### 5. Workspace file integration (priority 3)

Axel's workspace is `~/.openclaw/workspace/`. The harness should:
- Auto-load AGENTS.md, SOUL.md, TOOLS.md into system context at startup
- Read today's memory file (`memory/YYYY-MM-DD.md`) and include recent context
- Write session summaries back to the daily memory file on exit

---

## Axel's Setup (for reference)

- **Machine:** Fedora 43, RTX 4090 24GB, 93GB RAM
- **Ollama:** `http://127.0.0.1:11434`
- **Primary model:** `qwen3:14b-axel` (custom modelfile, 65536 ctx, 8192 max tokens)
- **Heavy coder model:** `qwen3-coder-next:latest` (use sparingly — VRAM-heavy)
- **Workspace:** `~/.openclaw/workspace/`
- **OpenClaw gateway:** `ws://127.0.0.1:18789` (existing harness, systemd service)

---

## Directory Layout

```
axel-harness/
├── Cargo.toml          workspace
├── PROJECT.md          this file
├── PLAN.md             implementation plan and status
├── crates/
│   ├── api/            provider abstraction (needs Ollama routing)
│   ├── runtime/        conversation engine (mostly reuse as-is)
│   ├── tools/          built-in tools (reuse, add Axel-specific)
│   ├── plugins/        hook pipeline (reuse as-is)
│   ├── commands/       slash commands (adapt for Axel)
│   ├── server/         HTTP/SSE server (keep for future)
│   ├── lsp/            LSP client (keep for future)
│   └── axel-cli/       REPL binary → produces `axel`
└── workspace/          symlink or ref to ~/.openclaw/workspace/
```

---

## Relationship to OpenClaw

OpenClaw (`ws://127.0.0.1:18789`) is Axel's current harness — a separate system that manages agent sessions over WebSocket. This project is building a Rust CLI companion that Axel (or Bill) can invoke directly for interactive or one-shot sessions.

Long-term, `axel` (this binary) could replace or augment OpenClaw's session handling. Short-term, it runs alongside it.
