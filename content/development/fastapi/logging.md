---
title: "Logging"
date: 2026-03-18
description: "Logging with loguru: setup, structured JSON output, and streaming to CloudWatch."
weight: 60
draft: false
params:
  neso:
    show_toc: true
---

Setting up logging for a FastAPI application (or any other Python application, for that matter).

<!--more-->

## Why loguru

Python's stdlib `logging` works, but the configuration is verbose: handlers, formatters, log levels per logger, `basicConfig` vs `dictConfig`. `loguru` replaces all of that with a single `logger` object that's ready to use on import.

```bash
uv add loguru
```

```python
from loguru import logger

logger.info("Server started")
logger.warning("Slow query | elapsed={elapsed:.1f}s", elapsed=1.2)
logger.error("Failed to connect to DB")
```
Here, `loguru` interpolates the `elapsed` variable into the message string. This also lays the groundwork for structured logging, where the message is a template and the variables are the data.

That's it --- no `getLogger`, no handler setup, no formatter strings. The default output is colorized, includes timestamps, levels, module/function/line, and looks good out of the box.

## Log Level via Environment

loguru reads the `LOGURU_LEVEL` environment variable to set the minimum log level. No code changes needed --- just set it in your `.env` or deployment config:

```bash
# .env
LOGURU_LEVEL=DEBUG    # dev
```

```bash
# production
LOGURU_LEVEL=INFO
```

This means the log level lives alongside your other config (see [Configuration](../configuration)) without needing to wire it through `Settings`.


## Exception Tracebacks

`logger.exception()` catches the current exception and prints a full traceback alongside the log message --- useful during development:

```python
try:
    result = await fetch_user(user_id)
except Exception:
    logger.exception("Failed to fetch user {user_id}", user_id=user_id)
```

loguru's tracebacks are more readable than stdlib's: they include variable values at each frame, colorized output, and the full chain for re-raised exceptions. You can also use `logger.opt(exception=True).error(...)` for the same effect when you want a level other than `ERROR`.

In production, full tracebacks add noise and can leak internals. Disable them by setting `diagnose=False` on the handler:

```python
def setup_logging(json_logs: bool = False) -> None:
    logger.remove()

    logger.add(
        sys.stderr,
        serialize=json_logs,
        backtrace=True,   # show full call chain
        diagnose=False,    # show variable values in frames only in dev
    )
```

With `diagnose=False`, tracebacks are still logged (via `logger.exception`), but they look like standard Python tracebacks without the extra variable introspection. With `backtrace=False`, only the frames from the exception would be shown, not the full call chain leading up to the `try` block --- personally, I prefer to see the full call chain: for my own sanity, it's either no traceback at all, or the full traceback.

## Structured JSON Output

The default formatter is great for local development. For production, switch to JSON so your log aggregator (CloudWatch, Datadog, etc.) can parse fields automatically:

```python
import sys

from loguru import logger


def setup_logging(json_logs: bool = False) -> None:
    logger.remove()  # remove the default stderr handler

    if json_logs:
        logger.add(sys.stderr, serialize=True)  # JSON lines
    else:
        logger.add(sys.stderr)                   # pretty-printed default
```

`serialize=True` outputs each log entry as a JSON object with `text`, `record.level`, `record.time`, `record.message`, `record.extra`, etc.

Toggle via your settings:

```python
# src/main.py
setup_logging(json_logs=not settings.debug)
```

## Streaming to CloudWatch

With JSON logging enabled, CloudWatch (or any log aggregator that reads stdout/stderr) picks up the structured output automatically --- no special integration needed. The container runtime streams stderr to the log driver.

For ECS / Fargate, the default `awslogs` log driver captures everything written to stdout/stderr and sends it to CloudWatch Logs. Just make sure:

1. JSON logging is on (`serialize=True`)
2. The task role has `logs:CreateLogStream` and `logs:PutLogEvents` permissions
3. The log group exists (create it in your IaC, don't rely on auto-creation)

No loguru plugin or CloudWatch SDK is needed --- the log driver handles transport. I'll speak more about this in the [Deployment](../deployment) page.

<!-- TODO: Request-ID middleware
     Assign a unique ID to each incoming request (via middleware),
     store it in a contextvars.ContextVar, and attach it to every log
     entry via loguru's `logger.bind(request_id=...)` or `logger.contextualize()`.
     This lets you trace all log lines for a single request across services. -->

<!-- TODO: Correlation IDs
     Similar to request IDs but propagated across service boundaries.
     The upstream caller sends a correlation ID in a header (e.g. X-Correlation-ID),
     the middleware reads it (or generates one if missing), and attaches it to logs
     and outgoing requests. This enables distributed tracing without a full
     tracing system like OpenTelemetry. -->
