# LMStudio → Ollama Migration

## Context
LMStudio has no CLI for model management — models can only be loaded/unloaded via the GUI, which blocks programmatic control by OpenClaw, Axel heartbeat jobs, and other tools. Switching to Ollama gives full CLI control (`ollama run`, `ollama stop`, `ollama ps`), a stable API at `localhost:11434`, and remote access via Tailscale without any GUI dependency.

**Ollama location:** Fedora desktop (local GPU), exposed via Tailscale for remote reach.
**LM Model Manager:** Deprecate — Ollama's built-in CLI replaces it.
**Model:** `qwen3-coder-next:latest`

---

## Phase 1 — Install Ollama + Pull Model

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Configure Ollama to listen on all interfaces (for Tailscale remote access):
```bash
sudo systemctl edit ollama
# Add:
# [Service]
# Environment="OLLAMA_HOST=0.0.0.0"
# Environment="OLLAMA_KEEP_ALIVE=35m"
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

Pull model:
```bash
ollama pull qwen3-coder-next:latest
ollama ps   # confirm loaded
```

> `OLLAMA_KEEP_ALIVE=35m` keeps the model warm across the 30-min heartbeat ticks.

---

## Phase 2 — Frontend: Open WebUI

Deploy via Docker on Fedora desktop:
```bash
docker run -d \
  -p 3000:3000 \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Access at `http://localhost:3000`. Open WebUI auto-discovers Ollama models.

---

## Phase 3 — OpenClaw

**Files to change:**
- `~/.openclaw/openclaw.json` — via `openclaw config set` (do NOT hand-edit)
- `~/.openclaw/agents/main/agent/agent.json`
- `~/.openclaw/agents/main/agent/models.json`
- `~/.openclaw/workspace/SOUL.md`

**Changes:**
1. Add `ollama` provider at correct port (`http://127.0.0.1:11434`) in openclaw.json with `qwen3-coder-next:latest` model entry — use `openclaw config set`
2. Update global default model: `agents.defaults.model.primary` → `ollama/qwen3-coder-next:latest`
3. Update `agents/main/agent/agent.json` model field
4. Update `agents/main/agent/models.json` — remove lmstudio provider, fix ollama provider port (currently wrong: 1234 → 11434)
5. Update SOUL.md model line: `lmstudio/qwen_qwen3-coder-next` → `ollama/qwen3-coder-next:latest`

---

## Phase 4 — Continue IDE

**File:** `/home/adminbill/.continue/config.yaml`

Change:
```yaml
# Before
apiBase: http://localhost:1234/v1
provider: lmstudio

# After
apiBase: http://localhost:11434/v1
provider: ollama
model: qwen3-coder-next:latest
```

---

## Phase 5 — Prison CMMS

**Files to change:**
- `/home/adminbill/projects/prison_cmms/.env`
- `/home/adminbill/projects/prison_cmms/.env.example`
- `/home/adminbill/projects/prison_cmms/.env.docker`
- `/home/adminbill/projects/prison_cmms/docker-compose.simple.yml`
- `/home/adminbill/projects/prison_cmms/app/ai/manager.py`

**Changes:**
1. All env files: `LM_API_URL=http://127.0.0.1:1234` → `LM_API_URL=http://127.0.0.1:11434`
2. `.env.docker` + `docker-compose.simple.yml`: use `http://host.docker.internal:11434`
3. `app/ai/manager.py` line 17: update default fallback URL to `http://127.0.0.1:11434`
4. Update default model env var `LM_MODEL` default from `qwen/qwen3.5-35b-a3b` to `qwen3-coder-next:latest`

Note: `docker-compose.yml` (full stack) already has an Ollama service defined — verify the internal service URL is `http://ollama:11434` and that the Ollama container is not commented out.

---

## Phase 6 — ToolBridge

**Files to change:**
- `/home/adminbill/projects/toolbridge/.env`
- `/home/adminbill/projects/toolbridge/app/config.py`

**Changes:**
1. `.env`: rename `LM_STUDIO_URL` → `OLLAMA_URL`, update value to `http://localhost:11434/v1`
2. `app/config.py` line 6: rename var and update default URL to match

---

## Phase 7 — Roomba AI Controller

**Files to change:**
- `/home/adminbill/projects/roomba-ai-controller/config/settings.py`
- `/home/adminbill/projects/roomba-ai-controller/.env.example`

**Changes:**
1. `settings.py`: change `ai_backend` default from `"lmstudio"` to `"ollama"`, update `lm_studio_host` default to `http://localhost:11434/v1`
2. `.env.example`: `AI_BACKEND=ollama`, `LM_STUDIO_HOST=http://localhost:11434/v1`

Note: `agent_loop.py` already has the ollama branch — just the default needs flipping.

---

## Phase 8 — ChatDev

**Files to change:**
- `/home/adminbill/projects/ChatDev/yaml_instance/` — all YAML files with `provider: lmstudio`
- `/home/adminbill/projects/ChatDev/runtime/node/agent/providers/lmstudio_provider.py`
- `/home/adminbill/projects/ChatDev/.env.lmstudio` → create `.env.ollama`

**Changes:**
1. All YAML files: `provider: lmstudio` → `provider: openai` (Ollama is OpenAI-compatible), `base_url: http://localhost:11434/v1`
2. Create `.env.ollama`: `BASE_URL=http://localhost:11434/v1`, `API_KEY=ollama`
3. `builtin_providers.py`: verify the openai/generic provider handles no-auth or dummy key correctly with Ollama

---

## Phase 9 — NeoVim Cockpit

**Files to change:**
- `/home/adminbill/projects/nvim-cockpit/lmstudio-query.sh`
- `/home/adminbill/projects/nvim-cockpit/lmstudio-context.sh`
- `/home/adminbill/projects/nvim-cockpit/nvim-config/lua/core/lm.lua`

**Changes:**
1. `lmstudio-query.sh`: update endpoint from `localhost:1234/v1/chat/completions` → `localhost:11434/v1/chat/completions`, remove LMStudio-specific auth header
2. `lmstudio-context.sh`: same endpoint update
3. `nvim-config/lua/core/lm.lua`: update URL, rename internal references from `lmstudio` → `ollama`

Optionally rename the scripts to `ollama-query.sh` / `ollama-context.sh` and update the Lua references accordingly.

---

## Phase 10 — LM Model Manager (Deprecate)

Add a `DEPRECATED.md` to `/home/adminbill/projects/lm-model-manager/` noting:
- Replaced by Ollama CLI (`ollama pull`, `ollama list`, `ollama rm`)
- Scripts in this project are LMStudio-specific and no longer maintained

Do NOT delete — leave scripts intact in case of rollback need.

---

## Verification

After each phase, run the relevant smoke test:

| Component | Test |
|-----------|------|
| Ollama | `ollama ps` + `curl localhost:11434/api/tags` |
| Open WebUI | Open `localhost:3000`, confirm model appears |
| OpenClaw | `openclaw agent --to main --message "ping"` |
| Continue | Trigger inline completion in VS Code / NeoVim |
| CMMS | `curl http://localhost:11434/v1/models` from app container |
| ToolBridge | Run existing test suite or health endpoint |
| Roomba | `python benchmark_model.py --backend ollama` |
| NeoVim | Trigger LM query from editor, confirm response |

Full end-to-end: run `openclaw cron run axel-heartbeat` and confirm HEARTBEAT_OK with no LMStudio errors in logs.
