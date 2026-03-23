# Changelog & Decisions

## 2026-03-17 — Asset Hierarchy Feature

- Added `AssetCategory` model with parent/child hierarchy support
- Created `init_asset_hierarchy.py` seed script (14 categories, 84 total with subcategories)
- Integrated asset categories into Work Order form
- Added edit templates for all 7 entities
- Files: `app/models/asset_category.py`, `app/routers/asset_categories.py`, `app/schemas/asset_category.py`

## 2026-03-16 — Authentication System (Phase 1-3)

- Implemented JWT auth with python-jose + bcrypt
- HTTP-only cookie sessions (XSS-resistant)
- Protected all routes with `Depends(get_current_user)`
- Login/logout UI at `/login` and `/logout`

## 2026-03-16 — Project Bootstrap

- FastAPI + SQLAlchemy + Jinja2 stack
- Entities: Zone, Cell, Door, Camera, Staff, WorkOrder, User
- SQLite for dev, PostgreSQL for production
- Docker Compose setup (simple + full production stack)
- AI work order suggestions via Ollama (local LLM)
