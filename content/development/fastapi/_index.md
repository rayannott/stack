---
title: "FastAPI"
description: "Project structure, error handling, testing, and deployment guides for FastAPI."
summary: "DI, testing, validation, logging, deployment, auth, and more."
weight: 1
---
This is how I set up clean and maintainable **FastAPI** projects: DI, testing, validation, logging, deployment, auth, and more.
<!--more-->

> [!TIP]
> I encourage you to feed the contents of this cookbook into your AI agent of choice. Append `raw.md` to any page URL to get the raw markdown (e.g. [this page's source](raw.md)).

## What's inside

Sections are ordered from foundational to advanced:

1. [Project Structure](project-structure) -- directory layout, `pyproject.toml` with `uv`, app factory pattern
2. [Dependency Injection](dependency-injection) -- idiomatic DI with `Depends`, lifespan, `app.state`, and when to reach for `dependency-injector`
3. [Validation & Serialization](validation) -- Pydantic models, custom validators, `Annotated` field metadata
4. [Error Handling](error-handling) -- custom exceptions, problem-detail responses (RFC 9457), validation error reshaping
5. [Testing](testing) -- `pytest` + `httpx.AsyncClient`, `dependency_overrides`, fixtures
6. [Logging](logging) -- structured logging, request-id middleware, correlation IDs
7. [Auth](auth) -- OAuth2 with JWT, security schemes via `Depends`, role-based access
8. [Deployment](deployment) -- multi-stage Dockerfile, `uvicorn` production settings, health checks
