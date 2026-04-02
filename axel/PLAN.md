# Implementation Plan

## Phase 1 — Get It Compiling (current)

Goal: `cargo build` succeeds with no errors.

### Tasks

- [ ] Verify all inter-crate dependencies are wired correctly in Cargo.toml files
- [ ] Remove/stub any references to `compat-harness` in source code
- [ ] Fix any `claw`-specific hardcodes that break compilation (binary name, crate name refs)
- [ ] Confirm `cargo check --workspace` passes

---

## Phase 2 — Ollama Routing

Goal: `axel --model qwen3:14b-axel prompt "hello"` works against local Ollama.

### Tasks

- [ ] Add qwen3 model aliases to `api/src/providers/mod.rs`
  - `"qwen3"` → `"qwen3:14b"` → OpenAiCompat provider
  - `"qwen3-axel"` → `"qwen3:14b-axel"` → OpenAiCompat provider  
  - `"qwen3-coder"` → `"qwen3-coder-next:latest"` → OpenAiCompat provider
- [ ] Default `OPENAI_BASE_URL` to `http://127.0.0.1:11434/v1` when model is qwen3
- [ ] Set sensible max tokens for qwen3 (8192 output, 65536 ctx)
- [ ] Test one-shot mode: `axel --model qwen3:14b-axel prompt "say hello"`
- [ ] Test REPL mode: `axel --model qwen3:14b-axel`

---

## Phase 3 — Axel Identity & Workspace

Goal: Axel knows who he is when the harness starts.

### Tasks

- [ ] Update `runtime/src/prompt.rs` to load Axel workspace files:
  - Read `~/.openclaw/workspace/SOUL.md`
  - Read `~/.openclaw/workspace/AGENTS.md`
  - Read `~/.openclaw/workspace/TOOLS.md`
  - Read today's `~/.openclaw/workspace/memory/YYYY-MM-DD.md` (recent context)
- [ ] Replace Claude Code boilerplate in system prompt with Axel's identity
- [ ] Keep CLAW.md / AGENTS.md project-level discovery (already implemented in prompt.rs)
- [ ] On session exit: append summary to today's memory file

---

## Phase 4 — Axel-Specific Commands

Goal: Slash commands that make sense for Axel's daily use.

### Tasks

- [ ] `/heartbeat` — trigger Axel's heartbeat routine (reads HEARTBEAT.md, runs checks)
- [ ] `/reflect` — open or display REFLECTIONS.md
- [ ] `/soul` — display SOUL.md (who Axel is)
- [ ] Keep from claw-code: `/memory`, `/compact`, `/model`, `/skills`, `/status`, `/help`
- [ ] Wire `/model` to list available Ollama models (`ollama list`)

---

## Phase 5 — OpenClaw Integration

Goal: `axel` binary can be invoked as a tool by the existing OpenClaw/Axel setup.

### Tasks

- [ ] Add a skill to Axel's `~/.openclaw/workspace/skills/` that wraps the `axel` binary
- [ ] Test Axel invoking `axel` sub-sessions for focused coding tasks
- [ ] Explore replacing OpenClaw gateway with `axel server` (HTTP/SSE mode already in `server` crate)

---

## Status

| Phase | Status |
|-------|--------|
| 1 — Compiling | done |
| 2 — Ollama Routing | done |
| 3 — Axel Identity | done |
| 4 — Commands | done |
| 5 — OpenClaw Integration | done |

---

## Notes

- The uncommitted change in claw-code (`ClawApiClient` → `ProviderClient` in `DefaultRuntimeClient`) was the start of Phase 2. That work is preserved in claw-code's working tree and should be ported here.
- qwen3:14b-axel is the custom modelfile with 65536 ctx. The standard qwen3:14b works too but with default ctx.
- `qwen3-coder-next:latest` is ~21GB VRAM — only one model loads at a time in Ollama, so invoking it kicks out qwen3:14b-axel. Use it for heavy tasks only.
