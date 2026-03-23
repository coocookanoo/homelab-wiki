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

## See Also

- [[setup/docker]] — Docker deployment
- [[architecture/entities]] — Data model overview
