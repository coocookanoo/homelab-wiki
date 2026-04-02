# Local Development Setup

## Prerequisites

- Python 3.10+
- pip

## Quick Start

```bash
cd prison_cmms
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

App runs at `http://localhost:8000`

## Seed Asset Hierarchy

Run once after first setup to populate the 14 CMMS asset categories:

```bash
python init_asset_hierarchy.py
```

## Database

SQLite (`cmms.db`) is used for local development. Created automatically on first run.

## Default Credentials

Check `.env.example` for default admin credentials.

## Local LLM (AI Suggestions)

The AI work order suggestion feature requires Ollama running locally.

```bash
# Ollama should already be running as a systemd service
curl http://localhost:11434/api/tags   # confirm it's up

# Set env var to point at local Ollama
export LM_API_URL=http://127.0.0.1:11434
export LM_MODEL=qwen3-coder-next:latest
```

Without Ollama, the app falls back to `app/ai_local_stub.py` automatically — no errors,
just canned stub responses.

## See Also

- [[setup/docker]] — Docker deployment
- [[architecture/entities]] — Data model overview
