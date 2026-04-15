---
title: "Project Structure"
date: 2026-03-18
description: "Directory layout, pyproject.toml with uv, and app factory pattern for FastAPI projects."
weight: 10
draft: false
params:
  neso:
    show_toc: true
---

How to lay out a FastAPI project so it stays maintainable as it grows.

<!--more-->

## Directory Layout

### Small projects (flat)

For a handful of endpoints with no database, a flat layout is fine:

```
my-api/
├── src/
│   ├── __init__.py
│   ├── app.py           # create_app factory, lifespan
│   ├── main.py          # entry point: app = create_app()
│   ├── config.py         # Settings (pydantic-settings)
│   ├── dependencies.py   # Depends helpers + type aliases
│   ├── schemas.py        # Pydantic request/response models
│   └── routes.py         # all route handlers
├── tests/
│   ├── __init__.py
│   └── test_routes.py
├── pyproject.toml
├── .env
└── .env.example
```

### Production projects (layered)

Once you have multiple resources, background tasks, or a database, split by layer:

```
my-api/
├── src/
│   ├── __init__.py
│   ├── app.py             # create_app factory, lifespan
│   ├── main.py            # entry point: app = create_app()
│   ├── config.py           # Settings
│   ├── dependencies.py     # shared Depends (settings, db session)
│   ├── exceptions.py       # custom exception classes
│   ├── middleware.py        # CORS, request-id, logging middleware (optional?)
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── users.py        # UserCreate, UserResponse, ...
│   │   └── bookings.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── users.py        # SQLAlchemy / ORM models
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── users.py        # router = APIRouter(prefix="/users")
│   │   └── bookings.py
│   ├── services/
│   │   ├── __init__.py
│   │   └── users.py        # business logic, orchestration
│   └── repositories/
│       ├── __init__.py
│       └── users.py        # raw DB queries
├── tests/
│   ├── __init__.py
│   ├── conftest.py           # shared fixtures (app, client, db)
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── test_schemas.py   # pure validation, no I/O
│   │   └── test_services.py  # business logic with mocked repos
│   └── integration/
│       ├── __init__.py
│       ├── conftest.py       # real DB, test containers
│       ├── test_users.py     # full request → response
│       └── test_bookings.py
├── pyproject.toml
├── .env
└── Dockerfile
```

**What goes where:**

| Layer | Responsibility |
|---|---|
| `routers/` | HTTP concerns only: parse request, call service, return response |
| `services/` | Business logic, orchestration, validation that depends on state |
| `repositories/` | Data access --- SQL queries, external API calls |
| `schemas/` | Pydantic models for request/response (no DB coupling) |
| `models/` | ORM models (SQLAlchemy, etc.) |
| `dependencies.py` | `Depends` helpers and `Annotated` type aliases |
| `exceptions.py` | Custom exception classes (see [Error Handling](../error-handling)) |
| `tests/unit/` | Fast, no I/O --- schema validation, service logic with mocked deps |
| `tests/integration/` | Slower, real DB or test containers --- full request/response cycles |

> [!NOTE]
> Some teams prefer domain-driven grouping (everything for `users` in one folder). Both work --- the layered approach shown here is easier to follow when the team is small and the domain boundaries are still fuzzy.

## `pyproject.toml` with `uv`

```toml
[project]
name = "my-api"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi[standard]",     # includes uvicorn + the fastapi CLI
    "pydantic-settings",
    "sqlalchemy[asyncio]",
]

[dependency-groups]
dev = [
    "ruff",
    "pre-commit",
]
test = [
    "pytest",
    "pytest-asyncio",
    "httpx",                 # for AsyncClient in tests
]

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.ruff]
target-version = "py313"
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
```

Install everything with:

```bash
uv sync --all-groups
```

> [!NOTE]
> `fastapi[standard]` pulls in `uvicorn[standard]` (with `uvloop` + `httptools`), the `fastapi` CLI, and other production essentials. Always use this extra unless you have a reason to pin a specific `uvicorn` version.

## Running the App

The `fastapi` CLI (bundled with `fastapi[standard]`) provides two commands:

```bash
# Development: auto-reload on file changes, debug mode
uv run fastapi dev

# Production-like: no reload, single worker
uv run fastapi run
```

**Auto-discovery:** the CLI looks for a variable named `app` in `main.py` or `app.py` (in that order). If your app lives elsewhere, pass the path explicitly:

```bash
uv run fastapi dev src/main.py
```

| Command | Reload | Debug | Use case |
|---|---|---|---|
| `uv run fastapi dev` | yes | yes | Local development |
| `uv run fastapi run` | no | no | Containers, staging, CI |
| `uv run uvicorn src.main:app` | manual flags | manual flags | Full control over workers, host, port |

For production deployments behind a reverse proxy, you'll typically use `uvicorn` directly or a process manager like `gunicorn` with `uvicorn` workers --- see the [Deployment](../deployment) page.

## The Entry Point

For anything beyond a toy project, use a `create_app()` factory. It keeps initialization explicit and makes testing straightforward (each test can get a fresh app).

Keep the factory in `src/app.py` --- separate from `src/main.py` --- so that tests can `from src.app import create_app` without triggering the module-level `app = create_app()` as a side effect.

```python
# src/app.py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from src.dependencies import get_settings
from src.exception_handlers import register_exception_handlers
from src.routers import bookings, users


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    # set up DB pools, caches, etc. on app.state
    # (see the Dependency Injection page for the full pattern)
    yield
    # teardown


def create_app() -> FastAPI:
    app = FastAPI(title="My API", lifespan=lifespan)

    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],  # ⚠ dev only — see warning below
        allow_methods=["*"],
        allow_headers=["*"],
    )

    register_exception_handlers(app)

    app.include_router(users.router)
    app.include_router(bookings.router)

    return app
```

```python
# src/main.py
from src.app import create_app

app = create_app()
```

`main.py` is the file the FastAPI CLI discovers (`fastapi dev` looks for an `app` variable in `main.py`). Tests import `create_app` from `src.app` and never touch `main.py`.

> [!WARNING]
> `allow_origins=["*"]` is fine for local development but **must not** be used in production --- it allows any website to make credentialed requests to your API. In production, list your specific origins (e.g. `["https://app.example.com"]`). See the [FastAPI CORS docs](https://fastapi.tiangolo.com/tutorial/cors/).

> [!WARNING]
> Don't put heavy initialization (DB connections, HTTP clients) inside `create_app()` directly --- use the lifespan context manager instead. `create_app()` should be fast and side-effect-free so tests can call it freely. See [Dependency Injection](../dependency-injection) for the lifespan pattern.

A typical router file looks like this:

```python
# src/routers/users.py
from fastapi import APIRouter

from src.dependencies import DbSession, ServiceDep
from src.schemas.users import UserCreate, UserResponse

router = APIRouter(prefix="/users", tags=["users"])


@router.post("/", status_code=201)
async def create_user(body: UserCreate, db: DbSession) -> UserResponse:
    ...

@router.get("/")
async def get_users(db: DbSession) -> List[UserResponse]:
    ...

@router.get("/{user_id}")
async def get_user(user_id: int, db: DbSession) -> UserResponse:
    ...
```

## Configuration

See the [Configuration](../configuration) page for the full `Settings` class setup: `.env` loading, nested settings with prefixes, and environment-specific overrides. For how to inject `Settings` into route handlers via `Depends`, see [Dependency Injection](../dependency-injection#1-settings-read-only-config-from-env--env).
