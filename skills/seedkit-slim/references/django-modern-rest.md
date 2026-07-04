# django-modern-rest

Import name is `dmr` — not `modern_rest` / `django_modern_rest`. Not a Django app: nothing goes in `INSTALLED_APPS`, no migrations.

```sh
uv add 'django-modern-rest[msgspec,openapi]' pyjwt  # `import dmr` reaches dmr.security.jwt unconditionally — without pyjwt even `manage.py check` breaks
```

Controllers are class-based views; there is no `router.register()`:

```python
# api/controllers.py
from dmr import Body, Controller
from dmr.plugins.msgspec import MsgspecSerializer

class UserController(Controller[MsgspecSerializer]):
    async def post(self, parsed_body: Body[UserCreate]) -> User: ...
```

```python
# config/urls.py
from django.urls import include, path
from dmr.routing import Router
from api.controllers import UserController

router = Router("api/", [path("users/", UserController.as_view(), name="users")])

urlpatterns = [
    # ...
    path(router.prefix, include((router.urls, "api"), namespace="api")),
]
```
