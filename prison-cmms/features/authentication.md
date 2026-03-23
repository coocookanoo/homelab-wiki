# Authentication System

## Overview

JWT-based authentication with HTTP-only cookie sessions.

## Stack

- **python-jose** — JWT token generation/validation
- **bcrypt** — Password hashing
- **HTTP-only cookies** — Token storage (XSS-resistant)

## Flow

1. User submits credentials to `POST /login`
2. Server validates password with bcrypt
3. Server issues JWT token in HTTP-only cookie
4. Protected routes check cookie via `Depends(get_current_user)`
5. `GET /logout` clears the cookie

## Protected Route Pattern

```python
@router.get("/zones/")
def list_zones(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    ...
```

## Token Config

Set in `.env`:

```env
SECRET_KEY=your-secret-key
ACCESS_TOKEN_EXPIRE_MINUTES=60
```

## User Management

Users are separate from Staff records. Managed at `/users/`.

## See Also

- [[architecture/api-patterns]] — Route-level auth usage
