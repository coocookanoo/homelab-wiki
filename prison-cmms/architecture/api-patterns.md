# API Patterns & Conventions

## URL Structure

Each entity follows a consistent pattern:

| Method | Path | Purpose |
|---|---|---|
| GET | `/entities/` | List all |
| GET | `/entities/{id}` | Get one |
| POST | `/entities/` | Create |
| PUT | `/entities/{id}` | Update |
| DELETE | `/entities/{id}` | Delete |

HTML views are served at the same paths with Jinja2 templates.

## Entities

- `/zones/`
- `/cells/`
- `/doors/`
- `/cameras/`
- `/staff/`
- `/work-orders/`
- `/asset-categories/`

## Delete Pattern

Deletes use `fetch()` with the DELETE method — **not** a form POST:

```javascript
fetch(`/entities/${id}`, { method: 'DELETE' })
  .then(() => window.location.href = '/entities/')
```

## Database Access

All routes use SQLAlchemy with dependency injection:

```python
def get_entity(id: int, db: Session = Depends(get_db)):
    ...
```

## Authentication

- JWT tokens stored in HTTP-only cookies
- Protected routes use `current_user = Depends(get_current_user)`
- Login at `/login`, logout at `/logout`

## See Also

- [[features/authentication]] — Auth system details
- [[architecture/entities]] — Entity relationships
