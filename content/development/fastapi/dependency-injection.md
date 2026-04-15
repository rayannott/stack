---
title: "Dependency Injection"
date: 2026-03-18
description: "Idiomatic dependency injection in FastAPI: settings, singletons, and per-request objects."
weight: 20
draft: false
params:
  neso:
    show_toc: true
---

Patterns for wiring settings, repositories, services, DB pools, and per-request objects into FastAPI route handlers --- using only what the framework gives you.

<!--more-->

> [!NOTE]
> This cookbook uses `Annotated` type aliases throughout (available since FastAPI 0.95+). Older tutorials show `param: Settings = Depends(get_settings)` --- that still works, but the `Annotated` form is cleaner and reusable across multiple endpoints.

## The Problem

You have objects (settings, repositories, services, DB pools) that route handlers need. You need to control **how many** instances exist, **when** they're created and destroyed, and make them **swappable in tests**.

FastAPI's native DI covers two lifetimes out of the box:

- **Per-process** singletons via the lifespan context manager
- **Per-request** objects via `Depends` (optionally with `yield` for teardown)

That handles the vast majority of real-world apps. The examples below use [SQLModel](https://sqlmodel.tiangolo.com/) (sync) for the SQL layer and [motor](https://motor.readthedocs.io/) for MongoDB. [Section 5](#5-with-async-sqlalchemy) shows how the same patterns look with async SQLAlchemy.

## 1. Settings (read-only config from env / `.env`)

**Pattern: `lru_cache` + `Depends`**

`Settings()` reads from disk (`.env`) or the environment, so you memoize it. Since settings are immutable after creation, `lru_cache` is safe. (For the full `Settings` class setup --- nested settings, prefixes, env-specific overrides --- see the [Configuration](../configuration) page.)

```python
# src/dependencies.py
from functools import lru_cache
from typing import Annotated

from fastapi import Depends

from src.config import Settings


@lru_cache
def get_settings() -> Settings:
    return Settings()


# Reusable type alias --- use this in any handler signature
SettingsDep = Annotated[Settings, Depends(get_settings)]
```

```python
# src/api.py
@router.get("/health")
def health(settings: SettingsDep) -> dict:
    return {"status": "ok", "debug": settings.debug}
```

**Testing:** override the dependency --- `lru_cache` is bypassed entirely. Point it at a disposable Postgres container so your tests run against the same engine as production (never swap Postgres for SQLite --- query behaviour, types, and constraints differ). [testcontainers-python](https://testcontainers-python.readthedocs.io/) spins one up in a fixture. See [Testing](../testing#dependency_overrides) for the full fixture pattern.

```python
app.dependency_overrides[get_settings] = lambda: Settings(
    db=DatabaseSettings(url="postgresql://test:test@localhost:5433/test_db")
)
```

If you call `get_settings()` directly outside the FastAPI request cycle (e.g. in a CLI command or a test helper), `dependency_overrides` has no effect. Reset the cache instead: `get_settings.cache_clear()`.

## 2. Engine and connection pool (process-scoped)

**Pattern: Lifespan + `app.state`**

The database engine (and its underlying connection pool) is the one true process-scoped singleton in the SQL path. Create it in the lifespan context manager from settings and store it on `app.state` --- no module-level globals. Everything else (sessions, repos, services) is per-request (next section).

```python
# src/app.py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI
from sqlmodel import create_engine

from src.dependencies import get_settings
from src.routers import bookings


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    engine = create_engine(
        str(settings.db.url),        # ← nested settings (see Configuration)
        pool_size=settings.db.pool_size,
        echo=settings.db.echo,
    )
    app.state.engine = engine
    yield
    engine.dispose()


def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(bookings.router)
    return app
```

Schema creation and migrations are handled by [Alembic](https://alembic.sqlalchemy.org/), not in the lifespan --- the lifespan is only for wiring runtime resources.

> [!NOTE]
> The [official full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) uses a module-level global `engine = create_engine(str(settings.SQLALCHEMY_DATABASE_URI))`. That works fine --- the engine is a thread-safe connection pool --- but `app.state` integrates better with the app factory pattern and `dependency_overrides` in tests: you can create multiple app instances with different engines without monkey-patching module-level globals.

> [!WARNING]
> Starlette's `app.state` is dynamically typed --- `request.app.state.engine` will cause Mypy / Pyright errors because the attribute doesn't exist in the `State` class definition. Suppress with `# type: ignore[attr-defined]`, or use `cast` to keep strict linters happy:
>
> ```python
> from typing import cast
> from sqlalchemy import Engine
>
> engine = cast(Engine, request.app.state.engine)
> ```

## 3. Per-request DI chain (SQL layer)

**Pattern: `Depends` sub-dependencies**

When the ORM requires a session scoped to a unit of work (SQLAlchemy / SQLModel), the session, repository, and service should all be **per-request**. FastAPI's [sub-dependency](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/) system wires them into a chain that is resolved automatically on each request:

```
Endpoint ← ServiceDep ← RepoDep ← SessionDep ← app.state.engine
```

Each link in the chain is a `Depends` function. FastAPI calls them right-to-left, caches each result within the request, and tears down `yield`-based dependencies after the response is sent.

```python
# src/dependencies.py
from collections.abc import Iterator
from typing import Annotated

from fastapi import Depends, Request
from sqlmodel import Session

from src.repositories.bookings import BookingRepository
from src.services.bookings import BookingService


def get_session(request: Request) -> Iterator[Session]:
    with Session(request.app.state.engine) as session:
        yield session


SessionDep = Annotated[Session, Depends(get_session)]


def get_booking_repo(session: SessionDep) -> BookingRepository:
    return BookingRepository(session)


BookingRepoDep = Annotated[BookingRepository, Depends(get_booking_repo)]


def get_booking_service(repo: BookingRepoDep) -> BookingService:
    return BookingService(repo)


BookingServiceDep = Annotated[BookingService, Depends(get_booking_service)]
```

The repo receives the session directly --- it never opens or closes sessions itself:

```python
# src/repositories/bookings.py
from collections.abc import Sequence

from sqlmodel import Session, select

from src.models.bookings import Booking


class BookingRepository:
    def __init__(self, session: Session) -> None:
        self.session = session

    def list_all(self) -> Sequence[Booking]:
        return self.session.exec(select(Booking)).all()

    def get_by_id(self, booking_id: int) -> Booking | None:
        return self.session.get(Booking, booking_id)
```

The service receives the repo --- it has no knowledge of sessions or the database:

```python
# src/services/bookings.py
from src.repositories.bookings import BookingRepository
from src.schemas.bookings import BookingResponse


class BookingService:
    def __init__(self, repo: BookingRepository) -> None:
        self.repo = repo

    def list_bookings(self) -> list[BookingResponse]:
        return [BookingResponse.model_validate(b) for b in self.repo.list_all()]
```

The endpoint declares only `BookingServiceDep`. FastAPI resolves the entire chain --- `get_booking_service` → `get_booking_repo` → `get_session` --- and tears down the session after the response:

```python
# src/routers/bookings.py
from fastapi import APIRouter

from src.dependencies import BookingServiceDep
from src.schemas.bookings import BookingResponse

router = APIRouter(prefix="/bookings", tags=["bookings"])


@router.get("/")
def list_bookings(service: BookingServiceDep) -> list[BookingResponse]:
    return service.list_bookings()
```

**Testing:** override at whatever level you need. Each override short-circuits everything below it in the chain. See [Testing](../testing#dependency_overrides) for the full fixture pattern.

```python
# swap the whole service (unit-test the endpoint in isolation)
app.dependency_overrides[get_booking_service] = lambda: BookingService(mock_repo)
# swap only the session (integration-test with a real repo against a test DB)
app.dependency_overrides[get_session] = lambda: test_session
```

**Key properties:**

- Each request gets its own session, repo, and service --- no shared mutable state between requests.
- `Depends` results are cached within a request: if multiple dependencies need the same `SessionDep`, the session is created only once.
- Repos and services are lightweight objects --- constructing them per-request costs virtually nothing compared to the I/O they perform.

> [!TIP]
> **Why per-request, not singletons?** You could store the repo and service on `app.state` as singletons (passing the session factory to the repo and letting it manage sessions internally). That works, but:
>
> - The repo now manages session lifecycle (open/close per method), which is harder to test and reason about.
> - `get_repo` as a `Depends` becomes dead boilerplate --- endpoints only use services, so nobody calls it.
> - It contradicts the [FastAPI SQL tutorial](https://fastapi.tiangolo.com/tutorial/sql-databases/) and [SQLModel docs](https://sqlmodel.tiangolo.com/tutorial/fastapi/session-with-dependency/), which both inject sessions per-request.
>
> The singleton approach **is** correct for clients that manage their own connection pool and have no per-request session concept --- see the [next section](#4-singleton-services-non-sql-clients).

## 4. Singleton services (non-SQL clients)

**Pattern: Lifespan + `app.state` + `Depends`**

Not every data store has a per-request session concept. MongoDB (via `motor`), Redis, HTTP API clients, or compute-heavy objects like ML models all manage their own connection pool or have no teardown-per-request need. For these, the entire client → repo → service chain is process-scoped: create everything in the lifespan, attach **only the service** to `app.state`.

```python
# src/app.py (lifespan --- showing both SQL and MongoDB side by side)
from motor.motor_asyncio import AsyncIOMotorClient
from sqlmodel import create_engine

from src.repositories.audit import AuditRepository
from src.services.audit import AuditService


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()

    # SQL --- only the engine is process-scoped; sessions/repos/services are per-request
    engine = create_engine(str(settings.db.url))
    app.state.engine = engine

    # MongoDB --- full singleton chain (motor manages its own pool)
    mongo_client = AsyncIOMotorClient(str(settings.mongo.url))
    mongo_db = mongo_client[settings.mongo.database]
    audit_repo = AuditRepository(mongo_db)
    app.state.audit_service = AuditService(audit_repo)

    yield

    engine.dispose()
    mongo_client.close()
```

Only the service goes on `app.state`. The client and repo are internal wiring details --- endpoints never touch them, and exposing `get_audit_repo` as a `Depends` would be unnecessary boilerplate.

```python
# src/dependencies.py
def get_audit_service(request: Request) -> AuditService:
    return request.app.state.audit_service


AuditServiceDep = Annotated[AuditService, Depends(get_audit_service)]
```

**Testing:** override the one dependency:

```python
app.dependency_overrides[get_audit_service] = lambda: AuditService(mock_audit_repo)
```

**Mixed dependencies** --- a SQL-backed service can depend on both a per-request repo and a singleton:

```python
def get_booking_service(
    repo: BookingRepoDep, audit: AuditServiceDep
) -> BookingService:
    return BookingService(repo, audit)
```

FastAPI resolves both: `repo` is per-request (via the DI chain from [section 3](#3-per-request-di-chain-sql-layer)), `audit` is a singleton pointer-read from `app.state`.

**The rule of thumb:** if the client manages its own connection pool and has no request-scoped session, use singletons. If the ORM requires a session scoped to a unit of work (SQLAlchemy / SQLModel), use the per-request DI chain.

## 5. With async SQLAlchemy

The DI chain structure from [section 3](#3-per-request-di-chain-sql-layer) is identical when using async SQLAlchemy --- only the engine and session setup change. SQLModel table models work unchanged with `AsyncSession`.

```python
# src/app.py (lifespan)
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    engine = create_async_engine(str(settings.db.url))   # async driver, e.g. asyncpg
    app.state.session_factory = async_sessionmaker(engine, expire_on_commit=False)
    yield
    await engine.dispose()
```

```python
# src/dependencies.py
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession


async def get_session(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.app.state.session_factory() as session:
        yield session


SessionDep = Annotated[AsyncSession, Depends(get_session)]
```

The rest of the chain (`get_repo` → `get_service` → endpoint) stays the same --- swap `Session` for `AsyncSession` in the repo's type hints.

> [!NOTE]
> With async, you store a `session_factory` (an `async_sessionmaker`) on `app.state` rather than the raw engine, because `AsyncSession(engine)` is not the intended API --- `async_sessionmaker` configures session options (like `expire_on_commit`) in one place. With sync SQLModel, `Session(engine)` is the standard API, so storing the engine directly is fine.

## Decision Matrix

| What you're injecting | Pattern | Why |
|---|---|---|
| Read-only config (`Settings`) | `@lru_cache` + `Depends` | Docs-blessed; memoizes `.env` and environment read; no teardown needed |
| SQL engine / connection pool | Lifespan + `app.state` | Process-scoped; teardown via `engine.dispose()` after `yield` |
| SQL session, repo, service | `Depends` chain with sub-dependencies | ORM session must be scoped to a unit of work; per-request creation and teardown |
| Non-SQL client + repo + service (MongoDB, Redis, HTTP) | Lifespan + `app.state` + `Depends` | Client manages its own pool; no per-request session; only the service goes on `app.state` |

## Anti-Patterns

> [!WARNING]
> These are common mistakes that look convenient at first but cause real pain in testing, lifecycle management, or type safety.

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Module-level `engine = create_engine(...)` | Couples to import-time settings; can't vary per test without monkey-patching | Create engine in lifespan from settings, store on `app.state` |
| Module-level `service = None  # type: ignore` set at startup | Not type-safe, not testable without monkey-patching, invisible lifecycle | Lifespan + `app.state` + `Depends` |
| `@lru_cache` on stateful dependencies (repos, services) | No teardown, outside FastAPI's lifecycle, awkward with `dependency_overrides` | Lifespan + `app.state` for singletons, `Depends` chain for per-request |
| Singleton repos that open/close sessions internally | Hides session scope; harder to test; risk of leaking sessions across requests | Per-request DI chain --- inject the session into the repo |
| Importing singletons directly (`from src.repo import repo`) | Untestable, hidden coupling, no lifecycle management | Always inject via `Depends` |

## Thread Safety

FastAPI runs **sync** route handlers in a threadpool. Shared objects may be accessed from multiple threads concurrently.

- **`Settings`** is safe --- it's read-only after creation.
- **Engine / connection pool** is safe --- SQLAlchemy's engine is designed for concurrent access.
- **Per-request repos and services** are safe --- each request gets its own instances, so there is no shared mutable state.
- **Singleton services** (non-SQL, on `app.state`) with mutable state need protection:
  - Sync handlers: use `threading.Lock`.
  - Async handlers: use `asyncio.Lock` --- concurrent coroutines interleave at `await` points.
  - Or delegate to a database / cache that handles concurrency internally.
- **Per-worker-process**: with `uvicorn --workers N` (or Gunicorn), each worker is a separate OS process with its own copy of `app.state`. Singletons are not shared across workers --- keep this in mind for in-memory caches, warmup data, and startup side effects.

## Further Reading

- [FastAPI - Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/) --- how to use `Depends` to inject dependencies into route handlers.
- [FastAPI - Sub-dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/) --- how to build dependency chains where one dependency depends on another.
- [FastAPI - SQL Databases](https://fastapi.tiangolo.com/tutorial/sql-databases/) --- official tutorial using SQLModel with `Depends` for session injection.
- [SQLModel - Session with FastAPI Dependency](https://sqlmodel.tiangolo.com/tutorial/fastapi/session-with-dependency/) --- SQLModel's recommended session-per-request pattern.
- [FastAPI - Lifespan Events](https://fastapi.tiangolo.com/advanced/events/) --- how to use the lifespan context manager to create and manage resources.
- [full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) --- official project template showing SQLModel + FastAPI in production.
- [Starlette - State](https://www.starlette.io/applications/#storing-state-on-the-app-instance) --- how to store state on the app instance.
- [dependency-injector](https://python-dependency-injector.ets-labs.org/) --- third-party DI container; useful when you need DI outside route handlers (Celery tasks, CLI commands) or have deep dependency graphs. Check out how I use it here in my [secret santa telegram bot](https://github.com/rayannott/ded-moroz/blob/main/src/dependencies.py).
