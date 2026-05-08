# Custom User Model

Set `AUTH_USER_MODEL` **before the first `migrate`**. Adding it later requires data migrations and breaks foreign keys to `auth.User`.

A `pk=BigAutoField` empty subclass is enough — extending it later (extra fields, email-as-username, etc.) doesn't require schema changes to existing rows.

## Create the app

```sh
mkdir users
uv run django-admin startapp users users
```

(`startapp <name> <path>` lets you place the app at the repo root. If you prefer `apps/users/` or similar, adjust the path and the dotted reference accordingly.)

## users/models.py

```python
from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
    pass
```

## users/admin.py

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin

from .models import User

admin.site.register(User, UserAdmin)
```

## Settings

In `config/settings.py` (or `config/settings/base.py` for split):

```python
INSTALLED_APPS = [
    ...
    "users",
]

AUTH_USER_MODEL = "users.User"
```

## Migrate

Now run the boot check (`migrate`, `createsuperuser`). The `users_user` table replaces `auth_user`; `createsuperuser` will use the new model.
