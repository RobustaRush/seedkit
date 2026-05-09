# django-extensions

Optional dev-only toolbox. Adds management commands and `runserver_plus` (Werkzeug-based debugger). Useful when the team actually uses these; pure noise otherwise.

Default **no**. Only apply when the user explicitly opts in.

## Install

```sh
uv add --dev django-extensions
```

`--dev` is intentional — these tools shouldn't ship to production. The dependency is dev-only in `pyproject.toml`, and `INSTALLED_APPS` adds the app conditionally.

## `local.py` only — never `base.py`

```python
from .base import *  # noqa: F401,F403
from .base import INSTALLED_APPS

INSTALLED_APPS = INSTALLED_APPS + ['django_extensions']
```

Don't add to `base.py` — production images would import a dev-only dep at boot and crash.

For single-settings layouts, gate it on `DEBUG`:

```python
if DEBUG:
    INSTALLED_APPS += ['django_extensions']
```

## What it adds

The commands worth knowing about:

- `shell_plus` — opens an IPython / bpython shell with every model, `timezone`, and common utils auto-imported. With `--print-sql`, also echoes every ORM-generated query.
- `runserver_plus` — Werkzeug-backed dev server with an interactive in-browser traceback debugger. `tailwind runserver` (`references/tailwind.md`) forwards to this if installed.
- `show_urls` — flat list of every URL pattern in the project; great for "where is this route defined".
- `graph_models` — dumps an ER diagram of the model graph (requires graphviz).
- `clean_pyc` / `reset_db` / `clear_cache` — rare, but occasionally useful.

## Pitfalls

- `runserver_plus`'s in-browser traceback can execute arbitrary Python from the URL bar. Never expose it on a network anyone else can reach. `DEBUG=True` plus `runserver_plus` plus `0.0.0.0` is a remote-code-execution invitation.
- `graph_models` needs `graphviz` installed at the OS level (`brew install graphviz` / `apt install graphviz`). Document that or skip the command.
- Don't reference `django_extensions` in template tags / model code. The whole package needs to remain a strip-out-able dev tool.
