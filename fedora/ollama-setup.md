# Ollama + OpenClaw Setup (Fedora Desktop)

**Last updated:** 2026-04-03 — Full wipe and fresh rebuild. LM Studio removed entirely.

## Current Stack

| Tool | Purpose | Status |
|---|---|---|
| Ollama 0.19.0 | Local LLM inference server | Running (systemd) |
| OpenClaw 2026.4.2 | AI agent TUI (Axel) | Running (systemd user service) |
| Continue (VS Code) | Code assistant + tab autocomplete | Wiped — needs fresh setup |
| Open WebUI | Web chat UI | Removed |

## Active Models

| Model | Size | Use |
|---|---|---|
| `qwen3:32b` | 20GB | Primary — OpenClaw/Axel, general use |
| `nomic-embed-text:latest` | 274MB | Embeddings |

**Why qwen3:32b:** Smaller models (14B and below) fail at OpenClaw agentic tool-use — they hallucinate tool calls and loop forever. 32B fits fully on RTX 4090, runs at ~50-70 tok/s.

## Ollama Service

Service file: `/etc/systemd/system/ollama.service`
Override: `/etc/systemd/system/ollama.service.d/override.conf`

```ini
[Service]
Environment="CUDA_VISIBLE_DEVICES=0"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_KEEP_ALIVE=-1"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_KV_CACHE_TYPE=q4_0"
Environment="OLLAMA_CONTEXT_LENGTH=32768"
```

- `OLLAMA_CONTEXT_LENGTH=32768` — safe for qwen3:32b (was 65536, reduced to ensure full GPU fit)
- `OLLAMA_KV_CACHE_TYPE=q4_0` — quantized KV cache, saves ~1-2GB VRAM
- `OLLAMA_KEEP_ALIVE=-1` — model stays loaded indefinitely
- `OLLAMA_HOST=0.0.0.0` — exposes API on all interfaces (Tailscale access)

### Common Commands

```bash
ollama list                        # list downloaded models
ollama ps                          # show what's loaded in memory
ollama pull qwen3:32b              # download a model
ollama stop qwen3:32b              # unload from memory
sudo systemctl restart ollama      # restart the service
nvidia-smi --query-compute-apps=pid,used_memory,name --format=csv,noheader  # check VRAM usage
```

## OpenClaw Integration

**Config:** `~/.openclaw/openclaw.json`
**Gateway:** `~/.config/systemd/user/openclaw-gateway.service`

### Critical: native Ollama API only

**Never use `/v1`** with OpenClaw + Ollama. The OpenAI-compat endpoint silently breaks tool calling — the model outputs raw JSON, agent loops forever.

- **Wrong:** `baseUrl: "http://127.0.0.1:11434/v1"`, `api: "openai-completions"`
- **Correct:** `baseUrl: "http://127.0.0.1:11434"`, `api: "ollama"`

### Key config

```json
{
  "agents": { "defaults": { "model": { "primary": "ollama/qwen3:32b" } } },
  "gateway": { "mode": "local", "auth": { "mode": "token" } },
  "models": {
    "mode": "merge",
    "providers": {
      "ollama": {
        "apiKey": "ollama-local",
        "baseUrl": "http://127.0.0.1:11434",
        "api": "ollama",
        "models": [
          { "id": "qwen3:32b", "contextWindow": 32768, "maxTokens": 8192 },
          { "id": "nomic-embed-text:latest", "contextWindow": 8192, "maxTokens": 8192 }
        ]
      }
    }
  }
}
```

### Auth token

- Stored in `~/.openclaw/gateway.token`
- `~/.bashrc` exports: `export OPENCLAW_GATEWAY_TOKEN=$(cat ~/.openclaw/gateway.token)`
- Existing terminals: `export OPENCLAW_GATEWAY_TOKEN=$(cat ~/.openclaw/gateway.token)`

### Gateway commands

```bash
systemctl --user status openclaw-gateway.service
systemctl --user restart openclaw-gateway.service
openclaw tui
```

### Session reset (if TUI gets stuck or wrong model)

```bash
systemctl --user stop openclaw-gateway.service
rm ~/.openclaw/agents/main/sessions/sessions.json
rm ~/.openclaw/agents/main/sessions/*.jsonl
systemctl --user start openclaw-gateway.service
```

## VRAM Notes (RTX 4090, 24GB)

- `qwen3:32b` @ Q4_K_M = ~20GB model weight
- KV cache @ 32768 ctx + q4_0 compression ≈ 1-2GB
- Total: ~22GB — fits cleanly, full GPU inference (~50-70 tok/s)

## Remote Access (Tailscale)

Ollama listens on `0.0.0.0:11434`. Connect from any Tailscale device:
```
http://<tailscale-ip>:11434
```

## What Was Removed (2026-04-03)

- LM Studio + all GGUF models (~336GB freed)
- All 29 old Ollama models
- llama.cpp build
- lm-agent Python agent + venv
- Open WebUI Docker container (not currently reinstalled)

## See Also

- [[fedora/PLAN]] — Fedora desktop plan
- [[fedora/lmstudio-to-ollama-migration]] — migration notes (historical)
