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

That handles the vast majority of real-world apps.

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

**Testing:** override the dependency --- `lru_cache` is bypassed entirely. See [Testing](../testing#dependency_overrides) for the full fixture pattern.

```python
app.dependency_overrides[get_settings] = lambda: Settings(database_url="sqlite+aiosqlite:///test.db")
```

## 2. Stateful singletons (repos, services, DB pools)

**Pattern: Lifespan + `app.state` + `Depends`**

Resources with mutable state or teardown needs should be created once in the lifespan context manager and stored on `app.state`. Do **not** use `lru_cache` for these --- it has no shutdown hook, operates outside FastAPI's lifecycle, and interacts awkwardly with `dependency_overrides`.

```python
# src/app.py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from src.api import router
from src.dependencies import get_settings
from src.repository import BookingRepository
from src.service import BookingService
from src.solver import Solver


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    # Depends() is not available here --- call get_settings() directly.
    settings = get_settings()                            # ← Settings (see Configuration)
    engine = create_async_engine(settings.database_url)  # ← from settings
    app.state.session_factory = async_sessionmaker(engine, expire_on_commit=False)
    app.state.repo = BookingRepository(app.state.session_factory)
    app.state.service = BookingService(app.state.repo, Solver())
    yield
    await engine.dispose()


def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(router)
    return app
```

```python
# src/dependencies.py
from typing import Annotated

from fastapi import Depends, Request

from src.repository import BookingRepository
from src.service import BookingService


def get_repo(request: Request) -> BookingRepository:
    return request.app.state.repo


def get_service(request: Request) -> BookingService:
    return request.app.state.service


RepoDep = Annotated[BookingRepository, Depends(get_repo)]
ServiceDep = Annotated[BookingService, Depends(get_service)]
```

```python
# src/api.py
from src.dependencies import ServiceDep
from src.schemas import RoomResponse


@router.get("/rooms")
def list_rooms(service: ServiceDep) -> list[RoomResponse]:
    return [RoomResponse(room_id=rid) for rid in service.list_rooms()]
```

**Testing:** override at the dependency level, no monkey-patching needed. The override short-circuits the entire lifespan chain (engine → session_factory → repo → service). See [Testing](../testing#dependency_overrides) for the full fixture pattern.

```python
app.dependency_overrides[get_service] = lambda: BookingService(mock_repo, Solver())
```

**Key properties:**

- Objects are created exactly once (per process), before any request is served.
- `request.app.state` is just a pointer read --- zero cost per request.
- Teardown code runs after `yield` on shutdown.
- Type-safe --- no `= None  # type: ignore` anywhere.

## 3. Per-request objects (DB sessions, auth context)

**Pattern: `Depends` with `yield`**

Objects that must be created fresh for each request and cleaned up afterward --- database sessions are the canonical example.

```python
# src/dependencies.py
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession


async def get_db(request: Request) -> AsyncIterator[AsyncSession]:
    # session_factory was stored on app.state during lifespan
    async with request.app.state.session_factory() as session:
        yield session
        # session is closed automatically by the context manager


DbSession = Annotated[AsyncSession, Depends(get_db)]
```

In a layered architecture, `DbSession` is consumed by the repository layer --- route handlers interact with services only (see [section 2](#2-stateful-singletons-repos-services-db-pools)). The `session_factory` is already on `app.state` --- it was set up in the lifespan alongside the engine and connection pool (created from [`settings.database_url`](../configuration)).

**Testing:** override `get_db` to return an in-memory SQLite session or a transaction-rolled-back session. See [Testing](../testing#dependency_overrides) for the full fixture pattern.

```python
app.dependency_overrides[get_db] = lambda: test_session
```

## Decision Matrix

| What you're injecting | Pattern | Why |
|---|---|---|
| Read-only config (`Settings`) | `@lru_cache` + `Depends` | Docs-blessed; memoizes `.env` read; no teardown needed |
| Stateful singletons (repo, service, pool) | Lifespan + `app.state` + `Depends` | Explicit lifecycle; teardown via `yield`; testable |
| Per-request objects (DB session, auth context) | `Depends` with `yield` | FastAPI handles per-request creation and teardown |

## Anti-Patterns

> [!WARNING]
> These are common mistakes that look convenient at first but cause real pain in testing, lifecycle management, or type safety.

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Module-level `service = None  # type: ignore` set at startup | Not type-safe, not testable without monkey-patching, invisible lifecycle | Lifespan + `app.state` + `Depends` |
| `@lru_cache` on stateful dependencies (repos, services) | No teardown, outside FastAPI's lifecycle, awkward with `dependency_overrides` | Lifespan + `app.state` |
| Constructing resources inside `Depends` without caching | New instance per request --- leaks connections, loses state | `app.state` for singletons, `yield` for per-request |
| Importing singletons directly (`from src.repo import repo`) | Untestable, hidden coupling, no lifecycle management | Always inject via `Depends` |

## Thread Safety

FastAPI runs **sync** route handlers in a threadpool. All patterns above produce shared objects accessed from multiple threads concurrently.

- **`Settings`** is safe --- it's read-only after creation.
- **Mutable objects** (in-memory repos, caches) are **not** thread-safe by default.
  - Sync handlers: use `threading.Lock` to protect shared mutable state.
  - Async handlers: use `asyncio.Lock` --- concurrent coroutines interleave at `await` points, so shared mutable state still needs protection even though there's no threadpool involved.
  - Or delegate to a database / cache that handles concurrency internally.
- **Per-worker-process**: with `uvicorn --workers N` (or Gunicorn), each worker is a separate OS process with its own copy of `app.state`. Singletons are not shared across workers --- keep this in mind for in-memory caches, warmup data, and startup side effects.

## Further Reading

- [FastAPI - Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/) --- how to use the `Depends` decorator to inject dependencies into route handlers.
- [FastAPI - Lifespan Events](https://fastapi.tiangolo.com/advanced/events/) --- how to use the lifespan context manager to create and manage resources.
- [Starlette - State](https://www.starlette.io/applications/#storing-state-on-the-app-instance) --- how to store state on the app instance.
- [dependency-injector](https://python-dependency-injector.ets-labs.org/) --- third-party DI container; useful when you need DI outside route handlers (Celery tasks, CLI commands) or have deep dependency graphs. Check out how I use it here in my [secret santa telegram bot](https://github.com/rayannott/ded-moroz/blob/main/src/dependencies.py).
