---
title: "Configuration"
date: 2026-03-18
description: "Type-safe configuration with pydantic-settings: .env files, nested settings, and secrets."
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

## Nested Settings

As the settings class grows, group related config into nested `BaseModel` sub-models. Set `env_nested_delimiter` on the root `Settings` so pydantic-settings can map environment variables like `DB__POOL_SIZE` to `settings.db.pool_size`:

```python
# src/config.py
from pydantic import BaseModel, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class DatabaseSettings(BaseModel):
    url: str = "sqlite+aiosqlite:///dev.db"
    password: SecretStr = SecretStr("")
    pool_size: int = 5
    echo: bool = False


class RedisSettings(BaseModel):
    url: str = "redis://localhost:6379/0"
    ttl: int = 3600


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_nested_delimiter="__",
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
DB__URL=postgresql+asyncpg://user:pass@localhost/mydb
DB__PASSWORD=s3cret
DB__POOL_SIZE=20
REDIS__URL=redis://cache:6379/0
```

The field name becomes the prefix automatically --- `db: DatabaseSettings` maps to `DB__*` env vars. Access in code reads naturally: `settings.db.url`, `settings.redis.ttl`.

> [!NOTE]
> Nested sub-models inherit from `BaseModel`, **not** `BaseSettings`. Only the root class should be `BaseSettings` --- this is the pattern the [pydantic-settings docs](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) document for nested configuration. `env_prefix` on sub-models has no effect; the delimiter handles nesting.


## Why use `pydantic-settings` when we have...

### `dotenv`
`python-dotenv` loads `.env` into `os.environ` and gives you strings back. You still have to cast types, set defaults, and validate manually. `pydantic-settings` does all of that declaratively --- types, defaults, validation, and `.env` loading in one class.

### `os.getenv`
`os.getenv("DATABASE_URL", "sqlite:///dev.db")` works for one or two values. At ten settings it becomes a scattered mess of string casts, missing-key bugs, and no single place to see what the app expects. A `Settings` class is that single place, and you get type errors at startup instead of runtime `KeyError`s in production.

Also, with a `Settings` object you get two ways to control config in tests: swap the entire object via `dependency_overrides` (see [Dependency Injection](../dependency-injection#1-settings-read-only-config-from-env--env)), or patch individual env vars and let `Settings()` re-read them. Neither option exists when config is scattered across bare `os.getenv` calls.


## Practical tips

Use `SecretStr` for any field that holds a secret (passwords, API keys, tokens). It prevents the value from leaking into logs, tracebacks, and `model_dump()` output:

```python
from pydantic import SecretStr

password: SecretStr = SecretStr("")

settings.db.password.get_secret_value()  # access the actual value
str(settings.db.password)                # → '**********'
```
