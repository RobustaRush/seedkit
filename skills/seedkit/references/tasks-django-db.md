# Django Tasks — Database backend

Docs: <https://github.com/RealOrangeOne/django-tasks-database>

`django-tasks-db` stores tasks in your existing DB. No broker, no extra service.

## Install

```sh
uv add django-tasks-db
```

Django 6.0+ ships `django.tasks` in stdlib — don't add the standalone `django-tasks` package, it shadows the stdlib module.

## INSTALLED_APPS

`django.tasks` is built into Django 6.0+. Only the backend's app registers:

```python
INSTALLED_APPS = [
    ...
    "django_tasks_db",   # ships migrations for the task table
]
```

## Settings

```python
TASKS = {"default": {"BACKEND": "django_tasks_db.DatabaseBackend"}}
```

## Migrate

```sh
uv run manage.py migrate
```

## Run worker (host)

```sh
uv run manage.py db_worker
```

## Local — run on the host

```sh
uv run manage.py db_worker
```

Open a second terminal alongside `uv run manage.py runserver`. The worker shares the project venv and reads from the same DB.

## VPS — docker-compose.prod.yml

Production image has `/opt/venv/bin` on `PATH`:

```yaml
services:
  worker:
    image: ghcr.io/{owner}/{project_slug}:latest
    restart: unless-stopped
    command: python manage.py db_worker
    env_file: .env.prod
    depends_on:
      db:
        condition: service_healthy
```

## Managed platforms

- **Fly.io** — `fly.toml`:
  ```toml
  [processes]
    web = "gunicorn config.wsgi --bind 0.0.0.0:8000"
    worker = "python manage.py db_worker"
  ```
- **Railway / Render** — second service on the same image, command `python manage.py db_worker`.
