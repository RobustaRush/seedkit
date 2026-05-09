# Debug — django-orbit

Dashboard at `/orbit/`. Per-request event correlation via `family_hash`. Dev-only.

## Install

```sh
uv add --dev django-orbit
```

With MCP support (so AI assistants can query live telemetry):

```sh
uv add --dev "django-orbit[mcp]"
```

## Settings

In `config/settings.py` (or `config/settings/local.py` for split):

```python
if DEBUG:
    INSTALLED_APPS += ["orbit"]
    # Insert AFTER SecurityMiddleware (index 0) so SSL redirect / HSTS / host
    # validation still run first. Putting Orbit at index 0 lets it observe
    # requests that SecurityMiddleware would have rejected — the data is
    # noisy and the security stack should be honoured even in dev.
    MIDDLEWARE.insert(1, "orbit.middleware.OrbitMiddleware")

    ORBIT_CONFIG = {
        "IGNORE_PATHS": ["/orbit/", "/static/", "/media/"],
        "HIDE_REQUEST_HEADERS": ["Authorization", "Cookie", "X-API-Key"],
        "HIDE_REQUEST_BODY_KEYS": ["password", "token", "api_key", "secret"],
        "SLOW_QUERY_THRESHOLD_MS": 100,
    }
```

`OrbitMiddleware` sits right after `SecurityMiddleware` — high enough to wrap everything else, low enough to respect the security stack.

## URLs

```python
from django.conf import settings

if settings.DEBUG:
    from django.urls import include, path
    urlpatterns += [path("orbit/", include("orbit.urls"))]
```

## Migrate

```sh
uv run manage.py migrate
```

## Logging (optional)

`django-orbit` is a dev dep, so the orbit handler lives in `config/settings/local.py` for the split layout. **Don't gate `LOGGING` itself on `if DEBUG:`** — that locks production out of any logging config and leaves it on Django's bare defaults. Instead, define a baseline `LOGGING` at module scope and **append** the orbit handler in dev:

`base.py` (always loaded):

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "handlers": {
        "console": {"class": "logging.StreamHandler"},
    },
    "root": {"handlers": ["console"], "level": "INFO"},
}
```

`local.py` (dev-only — adds the orbit handler on top):

```python
LOGGING["handlers"]["orbit"] = {"()": "orbit.handlers.OrbitLogHandler"}
LOGGING["root"]["handlers"].append("orbit")
LOGGING["root"]["level"] = "DEBUG"
```

Single-file layout: keep the baseline at module scope, then guard only the orbit-handler append with `if DEBUG:`.

## MCP — AI assistant integration (optional)

`claude_desktop_config.json` (macOS: `~/Library/Application Support/Claude/`):

```json
{
  "mcpServers": {
    "django-orbit": {
      "command": "uv",
      "args": ["run", "manage.py", "orbit_mcp"],
      "cwd": "/path/to/project",
      "env": {"DJANGO_SETTINGS_MODULE": "config.settings"}
    }
  }
}
```

Tools: recent requests, slow queries, exceptions, N+1 patterns, keyword search, performance stats.

## Dashboard

- `http://localhost:8000/orbit/` — live event feed.
- `http://localhost:8000/orbit/stats/` — Apdex, P50–P99, error rate, cache hit rate.
