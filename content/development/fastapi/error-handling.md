---
title: "Error Handling"
date: 2026-03-18
description: "Custom exception classes, exception handlers, problem-detail responses (RFC 9457), and validation error reshaping."
weight: 40
draft: false
params:
  neso:
    show_toc: true
---

Turning exceptions into consistent, machine-readable API responses.

<!--more-->

## The Defaults

FastAPI ships with `HTTPException` for quick error responses:

```python
from fastapi import HTTPException


@router.get("/users/{user_id}")
async def get_user(user_id: int, db: DbSession) -> UserResponse:
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse.model_validate(user)
```

This returns `{"detail": "User not found"}` with a 404 status code.

> [!NOTE]
> `detail` accepts any JSON-serializable value, not just strings. You can pass a dict or list for structured errors:
> ```python
> raise HTTPException(
>     status_code=400,
>     detail={"code": "INVALID_STATE", "message": "Booking is already confirmed"},
> )
> ```

`HTTPException` works fine for simple APIs. But as your app grows, you'll want:

- A consistent error shape across all endpoints
- Error types that carry domain semantics (`NotFoundError`, not `HTTPException(404)`)
- Machine-readable error codes, not just human messages

## Custom Exceptions

Define a base exception that carries everything needed to build an error response:

```python
# src/exceptions.py
class AppError(Exception):
    """Base exception for all application errors."""

    def __init__(
        self,
        status_code: int = 500,
        title: str = "Internal Server Error",
        detail: str = "",
    ) -> None:
        self.status_code = status_code
        self.title = title
        self.detail = detail


class NotFoundError(AppError):
    def __init__(self, resource: str, resource_id: object) -> None:
        super().__init__(
            status_code=404,
            title="Not Found",
            detail=f"{resource} {resource_id} not found",
        )


class ConflictError(AppError):
    def __init__(self, detail: str = "Resource already exists") -> None:
        super().__init__(status_code=409, title="Conflict", detail=detail)


class ForbiddenError(AppError):
    def __init__(self, detail: str = "Insufficient permissions") -> None:
        super().__init__(status_code=403, title="Forbidden", detail=detail)
```

Now route handlers read like domain code:

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: DbSession) -> UserResponse:
    user = await db.get(User, user_id)
    if not user:
        raise NotFoundError("User", user_id)
    return UserResponse.model_validate(user)
```

## Exception Handlers

FastAPI needs a handler to convert `AppError` into an HTTP response. Register it on the app:

```python
# src/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from src.exceptions import AppError


def create_app() -> FastAPI:
    app = FastAPI(...)

    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
        return JSONResponse(
            status_code=exc.status_code,
            content={"title": exc.title, "detail": exc.detail},
        )

    # ... routers, middleware ...
    return app
```

Any `AppError` (or subclass) raised anywhere --- in route handlers, dependencies, or services --- is now caught and serialized consistently.

## Problem-Detail Responses (RFC 9457)

[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) defines a standard JSON shape for API errors. Adopting it means every client, SDK generator, and monitoring tool can parse your errors without custom logic.

A problem-detail response looks like this:

```json
{
    "type": "https://my-api.example.com/errors/not-found",
    "title": "Not Found",
    "status": 404,
    "detail": "User 42 not found",
    "instance": "/users/42"
}
```

You don't need a library for this --- it's a Pydantic model and an exception handler:

```python
# src/schemas/errors.py
from pydantic import BaseModel


class ProblemDetail(BaseModel):
    type: str = "about:blank"
    title: str
    status: int
    detail: str = ""
    instance: str = ""
```

```python
# src/main.py
from src.exceptions import AppError
from src.schemas.errors import ProblemDetail


ERROR_TYPE_BASE = "https://my-api.example.com/errors"


def create_app() -> FastAPI:
    app = FastAPI(...)

    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
        slug = exc.title.lower().replace(" ", "-")
        body = ProblemDetail(
            type=f"{ERROR_TYPE_BASE}/{slug}",
            title=exc.title,
            status=exc.status_code,
            detail=exc.detail,
            instance=str(request.url.path),
        )
        return JSONResponse(
            status_code=exc.status_code,
            content=body.model_dump(exclude_none=True),
            media_type="application/problem+json",
        )

    return app
```

> [!NOTE]
> Setting `media_type="application/problem+json"` is part of the RFC spec. It tells clients the body follows the problem-detail format. Most JSON parsers handle it transparently.

This replaces the simpler handler from the previous section --- the `AppError` hierarchy stays exactly the same.

## Reshaping Validation Errors

FastAPI's default response for invalid request data (`RequestValidationError`) looks like this:

```json
{
    "detail": [
        {
            "type": "missing",
            "loc": ["body", "email"],
            "msg": "Field required",
            "input": {},
            "url": "https://errors.pydantic.dev/2.11/v/missing"
        }
    ]
}
```

The `url` field (added by Pydantic v2) is noisy for API consumers, and the shape doesn't match the problem-detail format. Override the handler to clean it up:

```python
# src/main.py
from fastapi.exceptions import RequestValidationError

from src.schemas.errors import ProblemDetail


def create_app() -> FastAPI:
    app = FastAPI(...)

    @app.exception_handler(RequestValidationError)
    async def validation_error_handler(
        request: Request, exc: RequestValidationError
    ) -> JSONResponse:
        errors = []
        for err in exc.errors():
            errors.append({
                "field": ".".join(str(loc) for loc in err["loc"]),
                "message": err["msg"],
                "type": err["type"],
            })

        body = ProblemDetail(
            type=f"{ERROR_TYPE_BASE}/validation-error",
            title="Validation Error",
            status=422,
            detail=f"{len(errors)} validation error(s)",
            instance=str(request.url.path),
        )
        payload = body.model_dump()
        payload["errors"] = errors

        return JSONResponse(
            status_code=422,
            content=payload,
            media_type="application/problem+json",
        )

    return app
```

The response now looks like this:

```json
{
    "type": "https://my-api.example.com/errors/validation-error",
    "title": "Validation Error",
    "status": 422,
    "detail": "2 validation error(s)",
    "instance": "/users/",
    "errors": [
        {"field": "body.email", "message": "Field required", "type": "missing"},
        {"field": "body.password", "message": "Field required", "type": "missing"}
    ]
}
```

> [!WARNING]
> The `errors` extension field is allowed by RFC 9457 (the spec is extensible), but not all tools will know about it. The top-level `title`, `status`, and `detail` fields are always enough for a human or a generic error handler to understand what went wrong.
