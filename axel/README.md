# Axel Documentation

Documentation for **Axel** — Bill's local persistent AI agent running on OpenClaw + Ollama/qwen3:14b-axel.

## Files

- [PROJECT.md](PROJECT.md) — Architecture overview, what was built and why (axel-harness Rust CLI)
- [PLAN.md](PLAN.md) — 5-phase implementation plan, all phases complete
- [LOG.md](LOG.md) — Full session log of what was built and decisions made

## Quick Reference

| What | Where |
|------|-------|
| Axel's harness project | `/home/adminbill/projects/axel-harness/` |
| Axel binary | `~/.local/bin/axel` |
| OpenClaw config | `~/.openclaw/openclaw.json` |
| Axel's workspace | `~/.openclaw/workspace/` |
| Identity / persona | `~/.openclaw/agents/main/agent/IDENTITY.md` |
| Primary model | `qwen3:14b-axel` (Ollama, 65536 ctx) |
| Heavy coder model | `qwen3-coder-next:latest` (VRAM-heavy, use sparingly) |
| Ollama endpoint | `http://127.0.0.1:11434` |
| OpenClaw gateway | `ws://127.0.0.1:18789` |
| axel serve port | `18790` |

## Status

All 5 phases of axel-harness complete:
- Phase 1 — Compiles clean
- Phase 2 — Ollama routing (qwen3 models auto-route, no env vars needed)
- Phase 3 — Axel identity loads at startup (SOUL.md, AGENTS.md, TOOLS.md, daily memory)
- Phase 4 — Slash commands: `/soul`, `/reflect`, `/heartbeat` (+ REPL and direct CLI)
- Phase 5 — Binary installed, skill registered in OpenClaw, `axel serve` on port 18790
