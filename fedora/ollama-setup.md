# Ollama + Open WebUI Setup (Fedora Desktop)

LMStudio has been replaced with Ollama for all local LLM inference.
Ollama provides a full CLI, a stable REST API, and remote access via Tailscale.

## Installed Components

| Component | URL | Notes |
|---|---|---|
| Ollama | `http://localhost:11434` | systemd service, auto-starts |
| Open WebUI | `http://localhost:3000` | Docker container, auto-starts |

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
Environment="OLLAMA_CONTEXT_LENGTH=65536"
```

- `OLLAMA_HOST=0.0.0.0` — exposes API on all interfaces (required for Docker containers and Tailscale)
- `OLLAMA_KEEP_ALIVE=-1` — model stays loaded in VRAM indefinitely (instant response)
- `OLLAMA_KV_CACHE_TYPE=q4_0` — quantized KV cache, frees ~3GB VRAM at 64K context
- `OLLAMA_CONTEXT_LENGTH=65536` — 64K context window (OpenClaw requires at least 36K)
- `OLLAMA_MAX_LOADED_MODELS=1` — only one model in VRAM at a time (RTX 4090 is 22.5GB)
- `OLLAMA_FLASH_ATTENTION=1` — memory-efficient attention

### Common Commands

```bash
ollama list                        # list downloaded models
ollama ps                          # show what's loaded in memory
ollama pull qwen3-coder-next:latest  # download a model
ollama stop qwen3-coder-next:latest  # unload from memory
ollama run qwen3:8b                # interactive chat
sudo systemctl restart ollama      # restart the service
```

## Open WebUI

Runs as a Docker container on port 3000.

```bash
docker ps --filter name=open-webui     # check status
docker logs open-webui --tail 50       # view logs
docker restart open-webui              # restart
```

Models are auto-discovered from Ollama — no manual config needed.
To pull a new model from the UI: **Admin Panel → Settings → Models → Pull from Ollama**.

**Note:** Open WebUI must be started with `--add-host=host.docker.internal:host-gateway` and `OLLAMA_BASE_URL=http://host.docker.internal:11434`. Ollama binds to `0.0.0.0` but Docker containers cannot reach `127.0.0.1` on the host directly.

```bash
docker run -d \
  --name open-webui \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

## Current Models

| Model | Size | Use |
|---|---|---|
| `qwen3-coder-next:latest` | 51 GB | OpenClaw/Axel agent (default), coding tasks |
| `qwen3:8b` | 5.2 GB | Fast inference, light tasks |
| `qwen3-vl:8b` | 6.1 GB | Vision + language |
| `qwen3-vl:32b` | 20 GB | Vision + language (higher quality) |
| `qwen2.5:32b` | 19 GB | General purpose |
| `nomic-embed-text:latest` | 274 MB | Embeddings |

## OpenClaw Integration

Axel (`main` agent) runs `ollama/qwen3:14b-axel` (custom modelfile, 65536 ctx).

Config: `~/.openclaw/openclaw.json` → `agents.defaults.model.primary`

Key config:
- `contextWindow: 65536` (was 131072 — caused crashes)
- `contextTokens: 49152` — compaction triggers at 75% context
- `compaction.mode: safeguard`

Cron jobs:
- `axel-heartbeat` — every **1 hour** (was 30min — caused session pile-up)
- `axel-daily-report` — **disabled**
- `Daily LLM Trending Report` — **disabled**

Service file: `~/.config/systemd/user/openclaw-gateway.service`
- Version: `2026.4.1`
- Binary: `/usr/lib/node_modules/openclaw/dist/index.js`
- After updates run `openclaw doctor` to verify entrypoint still matches

Gateway restart:
```bash
systemctl --user restart openclaw-gateway
```

## Moltbook

Axel has a verified Moltbook account: `axel-localai`
- API key in `~/.openclaw/workspace/TOOLS.md`
- Uses curl API only (site is Next.js SPA — web_fetch/agent-browser don't work)
- Every post/comment requires a math verification challenge solved within 5 minutes
- Heartbeat checks Moltbook feed every hour automatically

## Remote Access (Tailscale)

Ollama listens on `0.0.0.0:11434`. Connect any device on your Tailscale network using:

```
http://<tailscale-ip>:11434
```

## Continue Extension (VS Code)

Config: `~/.continue/config.yaml`

```yaml
models:
- apiBase: http://localhost:11434
  model: qwen3-coder-next:latest
  provider: ollama
tabAutocompleteModel:
  apiBase: http://localhost:11434
  model: qwen3-coder-next:latest
  provider: ollama
```

**Note:** Use `http://localhost:11434` NOT `http://localhost:11434/v1` — the `/v1` suffix breaks the native Ollama provider.

## See Also

- [[fedora/PLAN]] — Fedora desktop plan
