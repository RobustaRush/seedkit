# Celery

Redis broker + django-celery-beat for scheduling.

## Install

```sh
uv add 'celery[redis]' django-celery-beat django-redis  # django-redis is the Django cache backend; required if any cache subsystem points at Redis
```

## Settings

```python
# config/settings.py
INSTALLED_APPS += ["django_celery_beat"]

REDIS_URL = env("REDIS_URL")  # no trailing /<db>; settings append per subsystem so broker / results / cache stay segmented

CELERY_BROKER_URL = f"{REDIS_URL}/0"
CELERY_RESULT_BACKEND = f"{REDIS_URL}/1"
CELERY_BEAT_SCHEDULER = "django_celery_beat.schedulers:DatabaseScheduler"
CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True  # silences Celery 6 deprecation; old default flips in 7.x

CELERY_BEAT_SCHEDULE = {
    "daily-digest": {
        "task": "jobs.tasks.send_daily_digest",
        "schedule": 60 * 60 * 24,
    },
}
```

`.env` / `.env.example`:

```sh
REDIS_URL=redis://localhost:6379
```

## Compose service

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data  # shadow the image's `VOLUME /data` — otherwise Compose creates a fresh anonymous volume on every `up`
    ports:
      - "6379:6379"

volumes:
  redis_data:
```

## App wiring

```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")  # mirror wsgi.py — split layouts use config.settings.production
app = Celery("config")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# config/__init__.py
from .celery import app as celery_app

__all__ = ("celery_app",)
```

## Sample task

```python
# jobs/tasks.py
from celery import shared_task

@shared_task
def send_daily_digest():
    return "sent"
```

Verify autodiscovery without holding a worker open:

```sh
uv run python -c "from config import celery_app; celery_app.loader.import_default_modules(); print(sorted(t for t in celery_app.tasks if not t.startswith('celery.')))"
```
