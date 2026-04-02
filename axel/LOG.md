# Project Log — axel-harness

Running log of work sessions, decisions, and context. Newest entries at the top.

---

# Claude Code Log

_Claude Code writes here. Bill uses this for context when resuming work with Claude._

## 2026-04-02 — Session 1

**Who:** Bill + Claude Code

### What we did

**Explored claw-code** (`/home/adminbill/projects/claw-code`) — a clean-room Rust reimpl of Claude Code by Sigrid Jin (instructkr), published 2026-03-31. Bill had started this work the day before with Axel but the session log wasn't saved.

**Identified what Axel could use** from claw-code:
- `api` — provider abstraction (Anthropic, XAI, OpenAI-compat)
- `runtime` — conversation loop, session, file ops, bash, permissions, hooks
- `tools` — full MVP tool set (bash, file r/w/edit, glob, grep, web, todo, skill, agent, etc.)
- `plugins` — PreToolUse/PostToolUse hook pipeline
- `commands` — slash command registry (/help, /status, /compact, /memory, /skills, /agents, git commands, etc.)
- `server` — HTTP/SSE server for remote sessions
- `lsp` — LSP client for diagnostics/symbol nav
- `claw-cli` → renamed `axel-cli`, binary `claw` → `axel`

**Did NOT pull:**
- `compat-harness` — TypeScript source analysis tooling, not relevant
- Python `src/` workspace — porting scaffolding only

**Created the project** at `/home/adminbill/projects/axel-harness`:
- Workspace compiles clean
- `.gitignore` in place
- `PROJECT.md` — what this is and what needs changing
- `PLAN.md` — 5-phase implementation plan

**Completed Phase 2 — Ollama routing** in `crates/api/`:
- Added `ProviderKind::Ollama` to the provider enum
- Added `DEFAULT_OLLAMA_BASE_URL = "http://127.0.0.1:11434/v1"` to `openai_compat.rs`
- Added `OpenAiCompatConfig::ollama()` and `OpenAiCompatClient::for_ollama()` (no required API key)
- Registered qwen3 model aliases in `MODEL_REGISTRY`:
  - `qwen3` → `qwen3:14b`, `qwen3-axel` → `qwen3:14b-axel`, `qwen3-coder` → `qwen3-coder-next:latest`
  - Full names also registered directly
- `metadata_for_model`: any model starting with `qwen`, `llama`, or `mistral` auto-routes to Ollama
- `max_tokens_for_model`: qwen3 models → 8192 (matches Axel's modelfile)
- `ProviderClient::Ollama` variant wired through send/stream/kind

### Current state

`cargo check --workspace` passes clean. Binary `axel` will route `--model qwen3:14b-axel` to local Ollama automatically — no env vars needed.

### Completed in this session: Phase 3 also done (see below)

### Next session: pick up at Phase 4 (Axel-specific slash commands)

Update `runtime/src/prompt.rs` to load Axel's workspace files at startup:
- `~/.openclaw/workspace/SOUL.md`
- `~/.openclaw/workspace/AGENTS.md`
- `~/.openclaw/workspace/TOOLS.md`
- Today's `~/.openclaw/workspace/memory/YYYY-MM-DD.md`

So any `axel` REPL session starts as Axel — same identity, same context.

---

## 2026-04-02 — Session 1 continued (Phase 3)

**Phase 3 — Axel identity / workspace prompt loading — COMPLETE**

Added `AxelWorkspace` to `runtime/src/prompt.rs`:
- Loads `SOUL.md`, `AGENTS.md`, `TOOLS.md` from `~/.openclaw/workspace/` (or `$AXEL_WORKSPACE`)
- Loads today's `memory/YYYY-MM-DD.md`, capped at 80 lines
- Per-file char limit: 8000 chars
- `AxelWorkspace::is_empty()` guards against missing workspace gracefully

Updated `SystemPromptBuilder`:
- New `with_axel_workspace()` builder method
- When workspace is present: replaces generic Claude Code intro with "You are Axel..." and injects `# Identity`, `# Agent workspace`, `# Tools reference`, `# Recent memory` sections
- Project CLAW.md and config still load beneath as before

New `load_axel_system_prompt()` function — drop-in replacement for `load_system_prompt` that also loads the workspace.

Updated `axel-cli/src/main.rs` — `build_system_prompt()` and `print_system_prompt()` both call `load_axel_system_prompt`.

**Verified:** `axel system-prompt` outputs SOUL.md → AGENTS.md → TOOLS.md → today's memory → then environment/config sections. Working.

### Handed off to Axel

Phase 4 (slash commands) handed to Axel to implement. Task brief written to:
- `~/.openclaw/workspace/TASK_axel_harness_phase4.md` — full instructions
- `~/.openclaw/workspace/memory/2026-04-02.md` — flagged so he sees it at session start

Axel's job: add `/soul`, `/reflect`, `/heartbeat` commands to `crates/commands/src/lib.rs`, wire handlers in `axel-cli/src/main.rs`, write his LOG.md entry, commit, delete task file.

---

## 2026-04-02 — Session 1 continued (Phase 4)

**Phase 4 — Axel slash commands — COMPLETE** (done by Claude Code; Axel attempted but couldn't execute file edits)

Added to `crates/commands/src/lib.rs`:
- `SlashCommandSpec` entries for `soul`, `reflect`, `heartbeat` (with aliases `identity` and `hb`)
- `SlashCommand` enum variants: `Soul`, `Reflect`, `Heartbeat`
- Parse arms in `SlashCommand::parse()`
- `resume_supported_slash_commands()` exhaustiveness updated
- Handler functions: `handle_soul_slash_command()`, `handle_reflect_slash_command()`, `handle_heartbeat_slash_command()`
- Shared `axel_workspace_dir()` and `read_workspace_file()` helpers

Wired into `crates/axel-cli/src/main.rs`:
- Both `handle_repl_command()` and `run_resume_command()` dispatch sites

**Note on Axel:** He claimed to have done the work twice without actually executing any edits. Likely a tool-call limitation in the OpenClaw TUI — he may not have file write tools enabled or available in that session context. Worth investigating separately.

---

## 2026-04-02 — Session 1 continued (Phase 4 fix)

Bill caught that `Soul`, `Reflect`, `Heartbeat` were missing from the direct CLI path — the `parse_direct_slash_cli_action()` function and `CliAction` enum in `axel-cli/src/main.rs`.

Fixed:
- Added `CliAction::Soul`, `CliAction::Reflect`, `CliAction::Heartbeat` variants to the enum
- Wired them in `parse_direct_slash_cli_action()` so `axel /soul` etc. work from the terminal
- Wired the dispatch at the top-level action handler

All three commands now work in both the REPL (`/soul` typed interactively) and direct CLI (`axel /soul` from terminal).

---

## 2026-04-02 — Session 1 continued (Phase 5)

**Phase 5 — OpenClaw integration — COMPLETE**

**Binary installed:** `~/.local/bin/axel` (13MB release build)

**Skill registered:** `~/.openclaw/workspace/skills/axel-harness/`
- `SKILL.md` — documents one-shot prompt, REPL, server mode, model selection guide
- `_meta.json` — local skill marker

**`axel serve` command added:**
- `CliAction::Serve { port }` + `run_serve()` in axel-cli
- Starts HTTP/SSE server from the `server` crate on port 18790 (avoids OpenClaw's 18789)
- Usage: `axel serve`, `axel serve 8080`, `axel serve --port 8080`
- Added `pub async fn serve()` wrapper to server crate

**Smoke test:** `axel /soul` → prints SOUL.md ✅

**All 5 phases complete.** Project is functional end-to-end.

### What's left / future work

- Axel needs to actually USE the skill — test `axel --model qwen3:14b-axel prompt "..."` from his OpenClaw session
- Investigate why Axel couldn't execute file edits in Phase 4 (tool allowlist issue)
- Consider adding `axel serve` as a systemd service alongside OpenClaw
- Phase 6 ideas: session memory write-back on exit, `/model` wired to `ollama list`, better streaming output

---

# Axel Log

_Axel writes here. This is his running record of what he's built, why, and what he thinks about it._

## Instructions for Axel

When you work on this project (or Bill asks you about it), add an entry here:
- Date and a short title
- What you did or figured out
- Any decisions you made and why
- Anything worth remembering for next time

This is yours — write it the way you think, not like a commit message.

---

_No entries yet. Axel — when you pick this up, start here._

---

# Key Reference

| What | Where |
|------|-------|
| This project | `/home/adminbill/projects/axel-harness/` |
| Source (claw-code) | `/home/adminbill/projects/claw-code/` |
| Axel's workspace | `~/.openclaw/workspace/` |
| Ollama endpoint | `http://127.0.0.1:11434` |
| Axel's primary model | `qwen3:14b-axel` |
| Heavy coder model | `qwen3-coder-next:latest` |
| Plan | `PLAN.md` |
| Architecture | `PROJECT.md` |
