# Celery

Docs: <https://docs.celeryq.dev/> · <https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html>

Django has no background-task system. Celery is the established Python distributed queue: workers process jobs off a broker (Redis here), with retries, rate limits, schedules (Beat), priority queues, chained workflows. Use it for non-trivial background work — emails, image processing, periodic syncs.

Requires Redis (`references/redis.md`).

Ask the user about **periodic tasks** (Beat) — apply that section if yes.

## Install

```sh
uv add 'celery[redis]'
```

## config/celery.py

Default the settings module to match the layout — split: `"config.settings.production"` (mirrors wsgi/asgi); single-file: `"config.settings"`. A worker booted without `DJANGO_SETTINGS_MODULE` set should run with prod hardening; the host dev shell sets `DJANGO_SETTINGS_MODULE=config.settings.local` to override.

```python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")  # split layout

app = Celery("config")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

## config/\_\_init\_\_.py

```python
from .celery import app as celery_app

__all__ = ("celery_app",)
```

## Settings

In `config/settings.py` (or `config/settings/base.py`). If `redis.md` set `REDIS_URL`, reuse it.

```python
REDIS_URL = env("REDIS_URL", default="redis://127.0.0.1:6379")

CELERY_BROKER_URL = f"{REDIS_URL}/1"
CELERY_RESULT_BACKEND = f"{REDIS_URL}/2"
CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True  # silence Celery 5+ deprecation
```

## Define a task

Add `@shared_task` functions to `<app>/tasks.py` inside a registered Django app — `app.autodiscover_tasks()` only scans `INSTALLED_APPS`, **not** `config/`. Use `from celery import shared_task` on each function. If the project has no domain app yet, the worker boots and idles; the user creates an app and adds `tasks.py` when they have real work to schedule.

## Local — run on the host

```sh
uv run celery -A config worker -l info
```

Open a second terminal alongside `uv run manage.py runserver`. The worker shares the project venv and the `.env` file. `docker compose up -d redis` (and `db` if Postgres-in-Docker) must already be running.

## VPS — docker-compose.prod.yml

Production image has `/opt/venv/bin` on `PATH` — call `celery` directly:

```yaml
services:
  celery:
    image: ghcr.io/{owner}/{project_slug}:latest
    restart: unless-stopped
    command: celery -A config worker -l info
    env_file: .env.prod
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
```

## Managed platforms

- **Fly.io** — `fly.toml`:
  ```toml
  [processes]
    web = "gunicorn config.wsgi --bind 0.0.0.0:8000"
    worker = "celery -A config worker -l info"
  ```
- **Railway / Render** — second service on the same image with command `celery -A config worker -l info`.

---

## Periodic tasks — Celery Beat

Beat's default `PersistentScheduler` writes `celerybeat-schedule` (+ `-shm`, `-wal` SQLite sidecars) at runtime. Add `celerybeat-schedule*` to `.gitignore` — the `*.sqlite3` rule misses them (no extension).

### Settings

```python
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "example-task": {
        "task": "{project_slug}.tasks.example_task",
        "schedule": crontab(hour=8, minute=0),
    },
}
```

### Local — run on the host

```sh
uv run celery -A config beat -l info
```

Third terminal, alongside the worker and runserver.

### VPS — docker-compose.prod.yml

```yaml
services:
  celery-beat:
    image: ghcr.io/{owner}/{project_slug}:latest
    restart: unless-stopped
    command: celery -A config beat -l info
    env_file: .env.prod
    depends_on:
      - celery
```

### Managed platforms

Add a third process for beat alongside web and worker.
