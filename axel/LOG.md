# Project Log — axel-harness

---

## 2026-08-14 — LM Studio removed; axel + Alpaca both on Ollama/qwen

**Who:** Bill + Claude Code

### What we did

**LM Studio deleted from the box entirely** (42GB: `~/.lmstudio`, PATH entries in 4 shell
files, desktop entry). Its two models were imported into Ollama first and verified
responding before anything was deleted:
- `google/gemma-4-26b-a4b` → `bench-gemma26:latest`
- `nvidia/nemotron-3-nano` → `nemotron-3-nano:latest`

Conversations + server logs preserved at `~/backups/lmstudio-archive-20260814/` (97MB).

**Why:** two engines contending for one 24GB 4090. LM Studio sat on 17.9GB of VRAM for a
model `lms ps` reported as *not loaded* (an orphaned allocation), which starved Chrome's
GPU process into repeated SIGILL crashes. Consolidating onto Ollama removes the contention
and the duplicate model library.

### The breakage this caused, and the fix

`axel` with no flags **failed outright** after the removal — `crates/axel-cli/src/main.rs:46`
had `const DEFAULT_MODEL: &str = "google/gemma-4-26b-a4b"`, which routes to LM Studio's
`:1234`. Neither `~/.claw/settings.json`'s `model` key nor its `env` block overrides this
(verified empirically — settings `env` is *not* applied to provider resolution), and there
is **no `AXEL_MODEL` env var**. Only `--model` / `--provider` / `AXEL_PROVIDER` work at runtime.

Fixed by changing the constant to `qwen3.6:27b-q4_K_M` and rebuilding (`cargo build --release
-p axel-cli`, 3.4s incremental). Old binary kept at `~/.local/bin/axel.bak-20260814`.

`qwen3.6:27b-q4_K_M` is **unlisted** in the provider registry, and `detect_provider_kind`
resolves unlisted models to `ProviderKind::Ollama` — so the routing is correct without a
registry entry.

### Benchmark (same GGUF, same GPU, one engine at a time)

`gemma-4-26B-A4B-it-Q4_K_M`, 3 runs, median, streaming:

| engine | TTFT | throughput |
|--------|------|------------|
| Ollama | 0.33s | **130 tok/s** |
| LM Studio | **0.19s** | 97 tok/s |

**Caveat:** not an engine verdict. Both are llama.cpp; our Ollama has real tuning LM Studio
didn't (`OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_KV_CACHE_TYPE=q4_0`, `NUM_BATCH=512`). Read it as
"Ollama as configured here beats LM Studio as configured here."

**Reasoning-model streaming gotcha:** gemma-4-26B streams into `delta.reasoning` (Ollama) and
`delta.reasoning_content` (LM Studio) — **never** `delta.content` while thinking. Any
OpenAI-compat client that only reads `content` sees an empty response from both.

### `/v1` tool-calling concern — retested, no longer applies

The 2026-04-03 entry warns that the `/v1` OpenAI-compat endpoint silently breaks Ollama tool
calling. Retested 2026-08-14 with axel's Ollama provider (whose `default_base_url` *is*
`http://127.0.0.1:11434/v1`): a prompt requiring a file read produced a correct `read_file`
tool call and the right answer. Tool calling works over `/v1` with the current stack.

### Alpaca (new GUI, replaces LM Studio's interface)

Flatpak `com.jeffser.Alpaca` 9.2.5. Two gotchas worth knowing:
1. It installs and runs its **own bundled Ollama** on `:11435` with an empty model library —
   it will not see `/var/lib/ollama/models`. Change the instance from `ollama:managed` to
   type `ollama` (Ollama External) pointed at `http://127.0.0.1:11434`. Config lives in
   `~/.var/app/com.jeffser.Alpaca/data/alpaca.db`, table `instance`.
2. An **empty `api` property throws `Illegal header value b'Bearer '`** — the code does
   `'Bearer {}'.format(properties.get('api'))`, so an empty string yields a trailing space,
   which httpx rejects. Set any non-empty dummy token; Ollama ignores it.

Default + title model set to `qwen3.6:27b-q4_K_M`, same as axel.

### Current state

- One engine (Ollama), one model store (`/var/lib/ollama/models`), one GUI (Alpaca)
- `qwen3.6:27b-q4_K_M` resident, `UNTIL: Forever` (`OLLAMA_KEEP_ALIVE=-1`)
- `~/.config/axel/models.toml` rewritten — all entries are Ollama tags now

### Open tradeoff

Keeping a 17GB model resident forever recreates the exact VRAM squeeze LM Studio caused —
Chrome is back to ~4GB headroom and its GPU process will likely keep crashing. Switching
`PREFERRED_MODEL` in `~/.mempalace/kg_extract.py` to `gemma4:12b` (7.6GB) would leave ~14GB
free. Not done — deferred to Bill.

---

## 2026-04-03 — LLM wipe + fresh rebuild

**Who:** Bill + Claude Code

### What we did

**Full LLM stack wipe** — everything outside `~/projects/` deleted:
- `~/.lmstudio/` (336GB of GGUFs), `~/.ollama/`, `~/.openclaw/`, `~/.continue/`, `~/llama.cpp/`, `~/lm-agent/`, `~/agent-env/`, both LM Studio config dirs
- All 29 Ollama models removed
- TTS models moved to `~/projects/tts/` (kept)

**Root cause found** — all previous OpenClaw failures traced to one config mistake:
- Old setup used `baseUrl: "http://127.0.0.1:11434/v1"` + `api: "openai-completions"`
- The `/v1` OpenAI-compat endpoint **silently breaks Ollama tool calling** — model outputs raw JSON instead of proper tool calls, agent loops forever
- Fix: `baseUrl: "http://127.0.0.1:11434"` (no `/v1`) + `api: "ollama"` (native API)

**Model choice** — switched from qwen3:14b-axel to `qwen3:32b`:
- 14B and smaller models proved too small for OpenClaw agentic tool-use (hallucinate calls, loop)
- 32B fits fully on RTX 4090 (20GB @ Q4_K_M), fast at ~50-70 tok/s
- 32768 context window with q4_0 KV cache compression

**Fresh OpenClaw setup:**
- `openclaw setup` → fresh `~/.openclaw/openclaw.json`
- Gateway auth token generated, stored in `~/.openclaw/gateway.token`
- `~/.bashrc` exports `OPENCLAW_GATEWAY_TOKEN` on login
- Native Ollama provider configured, gateway running as systemd user service

**Verified working:** `openclaw tui` connected, `ollama/qwen3:32b` confirmed in status bar.

### Current state

OpenClaw + qwen3:32b working. axel-harness binary still at `~/.local/bin/axel` — will need model config updated from qwen3:14b-axel to qwen3:32b if used.

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
