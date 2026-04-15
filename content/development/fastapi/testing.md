---
title: "Testing"
date: 2026-03-18
description: "httpx.AsyncClient, dependency_overrides, and fixtures for testing FastAPI."
weight: 50
draft: false
params:
  neso:
    show_toc: true
---

Testing FastAPI apps with `pytest`, `httpx.AsyncClient`, and `pytest-asyncio`. For project setup (`pyproject.toml`, test directory layout, `asyncio_mode`), see [Project Structure](../project-structure#pyprojecttoml-with-uv).

<!--more-->

## The Test Client

Use `httpx.AsyncClient` with `ASGITransport` to test your app in-process --- no real network, native async, and the same API you'd use in production code:

```python
@pytest.fixture
async def client(app: FastAPI) -> AsyncIterator[AsyncClient]:
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
```

`base_url` can be anything --- requests never leave the process. The `app` parameter is a fixture that builds a fresh `FastAPI` instance per test (shown in the next section).

> [!NOTE]
> Starlette's sync `TestClient` works, but `AsyncClient` is the better default for FastAPI: it supports `async def` endpoints natively and mirrors the `httpx` API you'd use to call external services.

## `dependency_overrides`

FastAPI's DI and its testing story are two sides of the same coin. The [Dependency Injection](../dependency-injection) page shows the production wiring --- `get_settings`, `get_booking_service`, `get_session` --- each used in `Depends()`. In tests, you swap them via `app.dependency_overrides`.

It's a dict mapping a `Depends` callable to its replacement. FastAPI checks it **before** resolving any dependency, short-circuiting the entire chain:

```
Production:  handler → get_booking_service → get_booking_repo → get_session → app.state.engine

Test:        handler → dependency_overrides[get_booking_service] → lambda: test_service
```

You don't need to mock `app.state`, the lifespan, or any intermediate step --- override the `get_*` function and FastAPI does the rest. The entire engine → session → repo → service chain is bypassed. This is why the [DI patterns](../dependency-injection) use thin `get_*` wrappers rather than inline logic: each wrapper is a clean seam you can swap in tests.

### The fixture pattern

Wire overrides in the `app` fixture. Each mock (settings, repo) comes in as its own fixture parameter, so pieces stay independently swappable. The `app` fixture calls `create_app()`, applies overrides, `yield`s, then calls `app.dependency_overrides.clear()` in teardown to prevent leakage between tests.

You can override at any level of the [dependency chain](../dependency-injection) --- `get_settings` to swap config, `get_booking_repo` to replace just the repository, `get_session` to point at a test database, or `get_booking_service` to bypass the entire chain. The complete `conftest.py` is in the [next section](#fixtures).

> [!WARNING]
> The override key must be the **exact callable** used in `Depends()`. If the route uses `Depends(get_booking_service)`, the key is `get_booking_service` --- not the class, not a string, not a different wrapper.

## Fixtures

Complete `conftest.py` tying everything together:

```python
# tests/conftest.py
from collections.abc import AsyncIterator
from unittest.mock import MagicMock

import pytest
from fastapi import FastAPI
from httpx import ASGITransport, AsyncClient

from src.app import create_app
from src.config import Settings
from src.dependencies import get_settings, get_booking_service
from src.repositories.bookings import BookingRepository
from src.services.bookings import BookingService


@pytest.fixture
def test_settings() -> Settings:
    return Settings(debug=True)


@pytest.fixture
def mock_repo() -> MagicMock:
    return MagicMock(spec=BookingRepository)


@pytest.fixture
async def app(
    test_settings: Settings,
    mock_repo: MagicMock,
) -> AsyncIterator[FastAPI]:
    app = create_app()
    app.dependency_overrides[get_settings] = lambda: test_settings
    app.dependency_overrides[get_booking_service] = lambda: BookingService(mock_repo)
    yield app
    app.dependency_overrides.clear()


@pytest.fixture
async def client(app: FastAPI) -> AsyncIterator[AsyncClient]:
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
```

All fixtures default to `function` scope --- each test gets a fresh app, fresh overrides, and a fresh client. Widen scope only when test speed demands it and you've verified there's no shared mutable state.

> [!NOTE]
> `AsyncClient` with `ASGITransport` does **not** trigger [lifespan events](https://fastapi.tiangolo.com/advanced/async-tests/#httpx). That's fine here --- `dependency_overrides` bypasses everything the lifespan would set up, so `app.state` never needs to be populated. For integration tests that use real dependencies (no overrides), wrap the app in [`LifespanManager`](https://github.com/florimondmanca/asgi-lifespan) so the lifespan runs:
>
> ```python
> from asgi_lifespan import LifespanManager
>
> @pytest.fixture
> async def client(app: FastAPI) -> AsyncIterator[AsyncClient]:
>     async with LifespanManager(app):
>         async with AsyncClient(
>             transport=ASGITransport(app=app),
>             base_url="http://test",
>         ) as ac:
>             yield ac
> ```

## Integration tests (real database)

Unit tests with mocked services are fast but don't catch SQL bugs, schema mismatches, or driver-specific behaviour. Integration tests hit a real database --- **the same engine as production**. Never swap Postgres for SQLite --- query semantics, types, and constraints differ.

Use [testcontainers-python](https://testcontainers-python.readthedocs.io/) to spin up a disposable Postgres container. The container starts once per session; each test gets its own transaction that rolls back on teardown:

```python
# tests/integration/conftest.py
from collections.abc import AsyncIterator, Iterator

import pytest
from fastapi import FastAPI
from httpx import ASGITransport, AsyncClient
from sqlmodel import Session, SQLModel, create_engine
from testcontainers.postgres import PostgresContainer

from src.app import create_app
from src.dependencies import get_session


@pytest.fixture(scope="session")
def engine():
    with PostgresContainer("postgres:16") as pg:
        engine = create_engine(pg.get_connection_url())
        SQLModel.metadata.create_all(engine)
        yield engine
        engine.dispose()


@pytest.fixture
def session(engine) -> Iterator[Session]:
    with Session(engine) as session:
        yield session
        session.rollback()


@pytest.fixture
async def app(session: Session) -> AsyncIterator[FastAPI]:
    app = create_app()
    app.dependency_overrides[get_session] = lambda: session
    yield app
    app.dependency_overrides.clear()


@pytest.fixture
async def client(app: FastAPI) -> AsyncIterator[AsyncClient]:
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
```

Only `get_session` is overridden --- the real repo and service are constructed per-request as normal, exercising the full [DI chain](../dependency-injection#3-per-request-di-chain-sql-layer) against a live Postgres instance. See [Project Structure](../project-structure) for the `tests/integration/` directory layout.

> [!NOTE]
> **Testcontainers trade-offs:** Docker must be available --- both locally and in CI. The first container start adds a few seconds of overhead (session-scoping amortizes this across all tests). Testcontainers also starts a [Ryuk](https://github.com/testcontainers/moby-ryuk) sidecar to garbage-collect orphaned containers; some locked-down CI environments block this (set `TESTCONTAINERS_RYUK_DISABLED=true` as a workaround, but you risk leaked containers on crashes). On GitHub Actions with `ubuntu-latest`, Docker is pre-installed natively (the runner is a full VM, not a container) --- no extra configuration needed. However, if your CI runner is itself a container (e.g. GitHub Actions with `container:`, GitLab CI, or Kubernetes-based runners), you need [Docker-in-Docker](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/): either mount the host Docker socket or run a DinD sidecar service.

### Coverage

Add [`pytest-cov`](https://pytest-cov.readthedocs.io/) to your test dependencies and run:

```bash
uv run pytest --cov=src --cov-report=term-missing
```

### Example

```python
# tests/test_rooms.py
from httpx import AsyncClient


async def test_list_rooms(client: AsyncClient) -> None:
    response = await client.get("/rooms")

    assert response.status_code == 200
    assert isinstance(response.json(), list)


async def test_health_returns_debug_flag(client: AsyncClient) -> None:
    response = await client.get("/health")

    assert response.status_code == 200
    assert response.json()["debug"] is True
```

## Further Reading

- [FastAPI --- Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [httpx --- Async Client](https://www.python-httpx.org/async/)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)
- [testcontainers-python](https://testcontainers-python.readthedocs.io/) --- disposable Docker containers for integration tests (Postgres, Redis, etc.)
- [polyfactory](https://polyfactory.litestar.dev/) --- auto-generate test data from Pydantic models
- [hypothesis](https://hypothesis.readthedocs.io/) --- property-based testing
- [pytest-cov](https://pytest-cov.readthedocs.io/)
