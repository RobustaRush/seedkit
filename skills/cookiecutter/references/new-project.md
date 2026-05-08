# New Django Project

## Create

```sh
uv init --bare {project_slug}     # --bare skips main.py / README.md / .python-version
cd {project_slug}
uv add django django-environ
uv run django-admin startproject config .
```

After `startproject`, only **edit** `config/settings.py` — do not strip lines `django-admin` wrote (keep `DEFAULT_AUTO_FIELD`, `STATIC_URL`, etc.). The instructions below replace `SECRET_KEY` / `DEBUG` / `ALLOWED_HOSTS` / `DATABASES`; everything else stays.

Don't create an app named after the project (e.g. `shop/`) unless the user explicitly asked for one. The Django package is `config/`; apps come later when there's something to put in them.

## Settings — ask the user which structure they prefer

Use the snippets below verbatim. Don't keep `import os` from the generated `settings.py`, don't add a redundant `env("DEBUG", ...)` after the cast, don't reimplement `env.list` with `.split(",")`. The shape is intentional.

---

### Option A: Single settings file (simpler)

Keep `config/settings.py` in place. Remove `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` and `DATABASES` — add at the top:

```python
from pathlib import Path
import environ

BASE_DIR = Path(__file__).resolve().parent.parent

env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")

DEBUG = env.bool("DJANGO_DEBUG", default=False)
SECRET_KEY = env("DJANGO_SECRET_KEY", default="django-insecure-build-only" if DEBUG else None)
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=[])  # DEBUG already accepts localhost / 127.0.0.1 / [::1]
DATABASES = {"default": env.db("DATABASE_URL", default="sqlite:///db.sqlite3" if DEBUG else None)}
```

The `if DEBUG else None` defaults let dev / build steps boot with no `.env`. Production has `DJANGO_DEBUG` unset, so the defaults disappear and missing values raise `ImproperlyConfigured`.

`manage.py`, `config/wsgi.py`, and `config/asgi.py` keep the default `DJANGO_SETTINGS_MODULE = "config.settings"`.

---

### Option B: Split settings (base / local / production)

```sh
mkdir config/settings
mv config/settings.py config/settings/base.py
touch config/settings/__init__.py
```

**config/settings/base.py** — remove `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` and `DATABASES` — add at the top:

```python
from pathlib import Path
import environ

BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")

DEBUG = env.bool("DJANGO_DEBUG", default=False)
SECRET_KEY = env("DJANGO_SECRET_KEY", default="django-insecure-build-only" if DEBUG else None)
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=[])  # DEBUG already accepts localhost / 127.0.0.1 / [::1]
DATABASES = {"default": env.db("DATABASE_URL", default="sqlite:///db.sqlite3" if DEBUG else None)}
```

Dev-only defaults are gated by `DEBUG`. Production has `DJANGO_DEBUG` unset → defaults vanish → missing values raise `ImproperlyConfigured`.

**config/settings/local.py**

```python
from .base import *
```

**config/settings/production.py**

```python
from .base import *
```

**manage.py** — change `DJANGO_SETTINGS_MODULE` to `"config.settings.local"`.

**config/wsgi.py and config/asgi.py** — change `DJANGO_SETTINGS_MODULE` to `"config.settings.production"`.

---

## STATIC_ROOT

Append to `config/settings.py` (or `config/settings/base.py` for split layout) so `collectstatic` has a destination — `startproject` only writes `STATIC_URL`:

```python
STATIC_ROOT = BASE_DIR / "staticfiles"
```

## .env

```sh
DJANGO_SECRET_KEY=local-dev-secret-key-change-in-production
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
# DATABASE_URL — set per references/database.md (SQLite or PostgreSQL)
```

## .gitignore

Write a `.gitignore` for a Django + uv project. Must include `.venv/`, `.env`, `*.sqlite3`, `staticfiles/`, `media/`. Add the other standard Python / Django / editor / tooling entries you know belong. Do this before the first commit.

## Boot check

```sh
uv run manage.py migrate
uv run manage.py createsuperuser
uv run manage.py runserver
```

Confirm `/admin/` login works before continuing.

## Scripts

Add [Poe the Poet](https://poethepoet.natn.io/) for cross-platform task shortcuts:

```sh
uv add --dev poethepoet
```

```toml
[tool.poe.tasks]
dev     = "python manage.py runserver"
migrate = "python manage.py migrate"
test    = "pytest"
lint    = "ruff check ."
```

Run any task with `uv run poe <name>`:

```sh
uv run poe dev
uv run poe migrate
uv run poe test
uv run poe lint
```
