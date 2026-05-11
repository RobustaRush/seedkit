# Logging — structured (structlog + django-structlog)

Docs: <https://www.structlog.org/> · <https://django-structlog.readthedocs.io/>

Pretty console in dev, JSON lines in prod. Foreign loggers (Django, Celery, urllib3) render identically because both renderers are stdlib `logging` formatters wrapped by `ProcessorFormatter`. `django-structlog` ships the request middleware and Celery signal handlers — per-request `request_id`, `correlation_id`, `user_id` and per-task `task_id`/`task_name` flow via `contextvars` without project code.

## Install

```sh
uv add structlog django-structlog
```

## Settings

In `config/settings.py` (or `config/settings/base.py` for split):

```python
import structlog

# Shared chain. Used as `foreign_pre_chain` (stdlib records) and inside
# structlog.configure() (structlog-native records). Each record runs it once.
PRE_CHAIN = [
    structlog.contextvars.merge_contextvars,
    structlog.stdlib.add_log_level,
    structlog.stdlib.add_logger_name,
    structlog.processors.TimeStamper(fmt="iso", utc=True),
    structlog.processors.StackInfoRenderer(),
    structlog.processors.format_exc_info,
]

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {
            "()": "structlog.stdlib.ProcessorFormatter",
            "processor": structlog.processors.JSONRenderer(),
            "foreign_pre_chain": PRE_CHAIN,
        },
        "console": {
            "()": "structlog.stdlib.ProcessorFormatter",
            "processor": structlog.dev.ConsoleRenderer(colors=True),
            "foreign_pre_chain": PRE_CHAIN,
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "console" if DEBUG else "json",
        },
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django.db.backends": {"level": "WARNING"},
        "django.request": {"level": "WARNING"},
        "urllib3": {"level": "WARNING"},
        "botocore": {"level": "WARNING"},
        "celery": {"level": "INFO"},
    },
}

structlog.configure(
    processors=[*PRE_CHAIN, structlog.stdlib.ProcessorFormatter.wrap_for_formatter],
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)
```

## Per-request context

Add `django_structlog` to `INSTALLED_APPS` and insert its middleware **after** `AuthenticationMiddleware` (`request.user` must exist before the middleware binds `user_id`):

```python
auth_idx = MIDDLEWARE.index("django.contrib.auth.middleware.AuthenticationMiddleware")
MIDDLEWARE.insert(auth_idx + 1, "django_structlog.middlewares.RequestMiddleware")
```

Always-on. `LOGGING` belongs at module scope in `base.py` — never inside an `if DEBUG:` block, or production runs with bare Django defaults (no console handler at the root level).

For Celery, add `django_structlog.celery.steps.DjangoStructLogInitStep` to the app's boot steps — it binds `task_id` / `task_name` and propagates the parent request's `correlation_id`:

```python
# config/celery.py
from celery import Celery
from django_structlog.celery.steps import DjangoStructLogInitStep

app = Celery("config")
app.steps["worker"].add(DjangoStructLogInitStep)
```

## Sentry / GlitchTip / Bugsink

If `error-reporting.md` is in use, sentry-sdk's `LoggingIntegration` already routes `WARNING+` to breadcrumbs and `ERROR+` to events. Override per-logger via `LoggingIntegration(level=…, event_level=…)`.

## Pitfalls

- Call `structlog.configure()` only from settings (loaded once, before any logger).
- Keep `cache_logger_on_first_use=True`.
- No `ConsoleRenderer` in prod — ANSI codes break JSON parsers.
- `log.exception(...)` already implies `exc_info=True`; don't pass it again.
