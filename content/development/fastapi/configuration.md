---
title: "Configuration"
date: 2026-03-18
description: "Type-safe configuration with pydantic-settings: .env files and grouped settings with prefixes."
weight: 15
draft: false
params:
  neso:
    show_toc: true
---

Managing application settings with `pydantic-settings` --- type-safe, validated, and environment-aware.

<!--more-->

## The Settings Class
```python
# src/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",          # don't crash on unknown env vars
    )

    app_name: str = "My API"
    debug: bool = False
    database_url: str = "sqlite+aiosqlite:///dev.db"
    cors_origins: list[str] = ["*"]
```

Values are resolved in this order (first match wins):

1. Environment variables (case-insensitive)
2. `.env` file
3. Field defaults

```bash
# .env
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/mydb
```

> [!NOTE]
> `extra="ignore"` is important in practice. Without it, any unknown environment variable that happens to match a prefix will cause a validation error at startup.

For how to inject `Settings` into route handlers via `Depends`, see the [Dependency Injection](../dependency-injection#1-settings-read-only-config-from-env--env) page.

## Grouped Settings with Prefixes

As the settings class grows, group related config into sub-settings with their own `env_prefix`:

```python
# src/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_")

    url: str = "sqlite+aiosqlite:///dev.db"
    username: str = "user"
    password: str = "pass"
    pool_size: int = 5
    echo: bool = False


class RedisSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="REDIS_")

    url: str = "redis://localhost:6379/0"
    ttl: int = 3600


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        extra="ignore",
    )

    app_name: str = "My API"
    debug: bool = False
    cors_origins: list[str] = ["*"]
    db: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()
```

```bash
# .env
DB_URL=postgresql+asyncpg://user:pass@localhost/mydb
DB_POOL_SIZE=20
REDIS_URL=redis://cache:6379/0
```

Each sub-settings class loads its own env vars independently via its prefix. Access in code reads naturally: `settings.db.url`, `settings.redis.ttl`.


## Why use `pydantic-settings` when we have...

### `dotenv`
`python-dotenv` loads `.env` into `os.environ` and gives you strings back. You still have to cast types, set defaults, and validate manually. `pydantic-settings` does all of that declaratively --- types, defaults, validation, and `.env` loading in one class.

### `os.getenv`
`os.getenv("DATABASE_URL", "sqlite:///dev.db")` works for one or two values. At ten settings it becomes a scattered mess of string casts, missing-key bugs, and no single place to see what the app expects. A `Settings` class is that single place, and you get type errors at startup instead of runtime `KeyError`s in production.

Also, with a `Settings` object you get two ways to control config in tests: swap the entire object via `dependency_overrides` (see [Dependency Injection](../dependency-injection#1-settings-read-only-config-from-env--env)), or patch individual env vars and let `Settings()` re-read them. Neither option exists when config is scattered across bare `os.getenv` calls.


## Practical tips
> [!WARNING]
> Please don't commit the `.env` files to version control 😊
<!-- TODO(airat) add more -->
