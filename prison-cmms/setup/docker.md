# Docker Deployment

## Development (SQLite)

```bash
docker compose -f docker-compose.simple.yml up -d
```

## Production (PostgreSQL + full stack)

```bash
cp .env.example .env
# Edit .env with your values
docker compose up -d
```

## Production Stack

| Service | Purpose |
|---|---|
| FastAPI | Web app |
| PostgreSQL | Database |
| Redis | Task queue broker |
| Celery worker | Async task processing |
| Celery Beat | Scheduled tasks |
| Ollama | Local LLM for AI suggestions |
| Nginx | Frontend reverse proxy |

## Environment Variables

Key vars in `.env`:

```env
DATABASE_URL=postgresql://user:pass@db/cmms
SECRET_KEY=your-secret-key
LLM_ENDPOINT=http://ollama:11434
```

See `.env.example` for full reference.

## Viewing Logs

```bash
docker compose logs -f app
```

## See Also

- [[setup/local-dev]] — Local development
- [[features/work-orders]] — AI work order suggestions (requires Ollama)
