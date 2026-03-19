---
title: "FastAPI"
description: "Project structure, error handling, testing, and deployment guides for FastAPI."
summary: "DI, testing, validation, logging, deployment, auth, and more."
weight: 1
---
This is how I set up clean and maintainable **FastAPI** projects: DI, testing, validation, logging, deployment, auth, and more.
<!--more-->

> [!TIP]
> I encourage you to feed the contents of this cookbook into your AI agent of choice. Append `raw.md` to any page URL to get the raw markdown (e.g. [this page's source](raw.md) would give you the raw markdown of all the pages in this section (concatenated), or the "Source" icon next to any individual page's title).

## What's inside

Sections are ordered from foundational to advanced:

1. [Project Structure](project-structure) -- directory layout, `pyproject.toml` with `uv`, app factory pattern
2. [Configuration](configuration) -- `pydantic-settings`, nested settings with prefixes, environment-specific overrides
3. [Dependency Injection](dependency-injection) -- idiomatic DI with `Depends`, lifespan, `app.state`, and when to reach for `dependency-injector`
4. [Validation & Serialization](validation) -- Pydantic models, custom validators, `Annotated` field metadata
5. [Error Handling](error-handling) -- custom exceptions, problem-detail responses (RFC 9457), validation error reshaping
6. [Testing](testing) -- `pytest` + `httpx.AsyncClient`, `dependency_overrides`, fixtures
7. [Logging](logging) -- loguru setup, structured JSON output, streaming to CloudWatch
8. [Auth](auth) -- OAuth2 with JWT, security schemes via `Depends`, role-based access
9. [Deployment](deployment) -- multi-stage Dockerfile, `uvicorn` production settings, health checks
