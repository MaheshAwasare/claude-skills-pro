---
name: fastapi-production
description: Production FastAPI patterns — async DB with SQLAlchemy 2.x, dependency injection, JWT auth, structured logging, OpenAPI hardening, error handling, rate limiting, and deployment via uvicorn+gunicorn. Use when building a production FastAPI service, not when prototyping a single-file demo.
---

# FastAPI in Production

FastAPI gets you to "it works" quickly. Getting to "it survives" takes a different layout. This skill is that layout — async-first, dependency-injection-clean, OpenAPI-hardened.

## When to use

- New Python HTTP service.
- ML/AI inference API (FastAPI is the de facto standard).
- Replacing Flask for async + auto-OpenAPI benefits.
- Internal API where Python is the team's language.

## When NOT to use

- High-throughput JSON service (>20k RPS) — Go or Rust will be cheaper at scale.
- Background-job-only workload — use Celery/Arq directly without HTTP layer.

## Project layout

```
my-service/
  app/
    main.py                    # FastAPI() instance, middleware
    config.py                  # pydantic-settings
    deps.py                    # FastAPI dependencies
    db.py                      # async session factory
    models/                    # SQLAlchemy ORM
    schemas/                   # pydantic request/response models
    routers/
      health.py
      users.py
      payments.py
    services/                  # business logic
      payments.py
    middleware/
      logging.py
      tracing.py
  alembic/
    versions/
    env.py
  tests/
  pyproject.toml
  Dockerfile
  docker-compose.yml
```

`models/` (DB) and `schemas/` (API) are **separate**. Reusing one for both makes refactors painful and leaks DB structure into your API contract.

## Async DB (SQLAlchemy 2.x + asyncpg)

```python
# app/db.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(
    settings.database_url,                 # postgresql+asyncpg://...
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    echo=False,
)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)
```

```python
# app/deps.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db import SessionLocal

async def get_db() -> AsyncSession:
    async with SessionLocal() as session:
        yield session

DbDep = Annotated[AsyncSession, Depends(get_db)]
```

`Annotated[..., Depends(...)]` (Python 3.9+) is cleaner than `db: AsyncSession = Depends(get_db)` and reusable as `DbDep` everywhere.

## Routers + dependency-injected services

```python
# app/routers/payments.py
from fastapi import APIRouter
from app.deps import DbDep, CurrentUser
from app.schemas.payments import ChargeIn, ChargeOut
from app.services import payments as svc

router = APIRouter(prefix="/api/v1/payments", tags=["payments"])

@router.post("/charges", response_model=ChargeOut, status_code=201)
async def create_charge(payload: ChargeIn, db: DbDep, user: CurrentUser):
    return await svc.create_charge(db, user, payload)
```

```python
# app/services/payments.py
async def create_charge(db: AsyncSession, user: User, payload: ChargeIn) -> Charge:
    if payload.amount_paise <= 0:
        raise HTTPException(400, "amount must be positive")
    charge = Charge(user_id=user.id, amount_paise=payload.amount_paise, ...)
    db.add(charge)
    await db.commit()
    await db.refresh(charge)
    return charge
```

Routers are thin (parse, dispatch, serialize). Services hold logic and are unit-testable without a TestClient.

## Auth (JWT or Clerk/Auth0/Cognito)

```python
# app/deps.py
from fastapi import HTTPException, Depends
from fastapi.security import HTTPBearer

bearer = HTTPBearer()

async def current_user(creds = Depends(bearer), db: DbDep) -> User:
    try:
        payload = jwt.decode(creds.credentials, settings.jwt_public_key, algorithms=["RS256"])
    except jwt.PyJWTError:
        raise HTTPException(401, "invalid token")
    user = await db.get(User, payload["sub"])
    if not user:
        raise HTTPException(401, "user not found")
    return user

CurrentUser = Annotated[User, Depends(current_user)]
```

`HTTPBearer` auto-shows up in OpenAPI as a security scheme; Swagger UI gets a "Authorize" button.

## Settings via pydantic-settings

```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str
    jwt_public_key: str
    log_level: str = "INFO"
    sentry_dsn: str | None = None
    cors_origins: list[str] = []
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

settings = Settings()
```

Single source of truth for all env vars; type-checked at startup. App fails to boot if a required var is missing.

## Logging (structured)

```python
# app/middleware/logging.py
import structlog
from starlette.middleware.base import BaseHTTPMiddleware

logger = structlog.get_logger()

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        if request.url.path in {"/healthz", "/readyz"}:
            return await call_next(request)
        log = logger.bind(method=request.method, path=request.url.path)
        log.info("request_start")
        response = await call_next(request)
        log.info("request_end", status=response.status_code)
        return response
```

## Error handling (consistent shape)

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"error": "validation_error", "detail": exc.errors()},
    )

@app.exception_handler(Exception)
async def unhandled_handler(request: Request, exc: Exception):
    logger.exception("unhandled", path=request.url.path)
    return JSONResponse(status_code=500, content={"error": "internal_error"})
```

Every error response has `{"error": "<code>", ...}` — clients can switch on a stable code, not a free-form string.

## Health + readiness

```python
@router.get("/healthz")
async def healthz():
    return {"status": "ok"}

@router.get("/readyz")
async def readyz(db: DbDep):
    try:
        await db.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(503, "db unavailable")
    return {"status": "ready"}
```

Same `/healthz` vs `/readyz` discipline as in `scaffold-go-microservice`.

## Deployment (uvicorn + gunicorn)

```dockerfile
FROM python:3.12-slim AS deps
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=deps /app/.venv /app/.venv
COPY app /app/app
ENV PATH=/app/.venv/bin:$PATH
EXPOSE 8000
CMD ["gunicorn", "app.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

`gunicorn -w 4 -k uvicorn.workers.UvicornWorker` is the prod recipe — one master, 4 worker processes (each is a uvicorn event loop). Worker count = `2 * cores + 1` typically.

## Anti-patterns

- **Sync DB in an async app** — blocks the event loop. Use `asyncpg`/`asyncmy`/`aiosqlite`.
- **Reusing SQLAlchemy ORM as Pydantic schema** — leaks DB columns to API. Separate `models/` and `schemas/`.
- **`Depends()` chains 4 deep with side effects** — DI is for wiring, not for "now also charge the customer." Keep DI pure.
- **Putting JWT verification in a middleware** — middleware can't return typed `User`. Use `Depends(current_user)`.
- **`os.environ` reads scattered through code** — single `Settings` class with pydantic-settings.
- **Single `/health`** — same trap as Go scaffold; split liveness and readiness.
- **`async def` everywhere even when work is sync** — fine, but don't call sync DB libs inside; the event loop blocks.
- **`uvicorn --reload` in production** — dev only. Use gunicorn + uvicorn workers.
- **Returning Pydantic models that include private fields** — define explicit `*Out` schemas; never serialize the DB row directly.
- **No `response_model` on routes** — OpenAPI loses contract; clients can't generate types reliably.
- **No background queue at all** — first webhook with a slow downstream takes down the API. Use Arq or Celery.

## Verify it worked

- [ ] `pytest` passes; tests run against an in-process app via `httpx.AsyncClient(app=app)`.
- [ ] `/docs` renders Swagger UI with all routes, `Authorize` button, and matching schemas.
- [ ] DB connection pool returns to pool after each request (observe `pool.checkedout()`).
- [ ] Killing the DB → `/readyz` returns 503; `/healthz` stays 200.
- [ ] Settings missing required env var → app fails to boot with a clear error.
- [ ] All error responses have shape `{"error": "<code>", ...}`.
- [ ] Sentry catches an unhandled exception with stack trace + request context.
- [ ] gunicorn runs N workers in container; workers don't share state in memory (verify via global counter test).
- [ ] OpenAPI schema (`/openapi.json`) is committed-to-repo or generated in CI; client codegen works.
- [ ] Handlers don't call sync I/O libraries (no `requests.get`, no `psycopg2`).
