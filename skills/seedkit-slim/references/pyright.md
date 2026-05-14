# pyright + django-stubs

`djangoSettingsModule` is a `django-stubs` plugin option, not a pyright option — putting it under `[tool.pyright]` triggers "unknown config" warnings.

```toml
[tool.pyright]
include = ["."]
exclude = [".venv", "**/migrations"]
venvPath = "."
venv = ".venv"

[tool.django-stubs]
django_settings_module = "config.settings.local"
```

Install: `uv add --dev pyright django-stubs django-stubs-ext`. In `settings/base.py`:

```python
try:
    import django_stubs_ext
    django_stubs_ext.monkeypatch()
except ImportError:  # dev-only; prod runs `uv sync --no-dev`
    pass
```

`env(...)` with a non-NOTSET default trips the `django-environ` stubs (the `default` param is typed against a `NoValue` sentinel). Tag those sites:

```python
DEBUG = env.bool("DJANGO_DEBUG", default=False)  # type: ignore[call-arg]
```

When `path()` complains about `Consumer.as_asgi()`:

```python
path("ws/echo/", EchoConsumer.as_asgi()),  # type: ignore[arg-type]  # channels returns ASGIApp, path() expects view callable
```

`django-stubs` exposes `User.pk`, not `User.id`. Use `user.pk` in serializers / API handlers. `request.user` types as `User | AnonymousUser`; narrow with `cast(User, request.user)` when a view is auth-gated.
